---
layout: post
title: "Proxmox VM 백업 및 복원 가이드 (PBS)"
excerpt: "별도 물리 서버의 Proxmox Backup Server로 VM을 백업·복원한 작업 기록"

categories:
  - Blog

tags:
  - [proxmox, backup, infra]

toc: true
toc_sticky: true

date: 2026-07-23
---

# Proxmox VM 백업 및 복원 가이드

이 문서는 Proxmox VE의 기본 구조를 이해하고, 별도 물리 서버에 구축한 Proxmox Backup Server(PBS)로 VM을 백업·복원하기 위한 작업 기록이다.

현재 환경의 핵심 원칙은 다음과 같다.

- 운영 VM은 `VM 101`이다.
- 개발 VM은 `VM 100`이다.
- 운영 VM에는 불필요한 종료나 설정 변경을 하지 않는다.
- PBS는 운영 VM이 있는 물리 서버와 다른 물리 서버에 구성한다.
- 실제 운영 백업 전에는 별도 테스트 VM으로 백업 경로를 검증한다.

---

## 1. Proxmox 기본 동작 구조

```text
물리 서버
├─ CPU, RAM, 디스크, 네트워크 카드
└─ Proxmox VE
   ├─ KVM: CPU와 메모리 가상화
   ├─ QEMU: 가상 디스크와 가상 장치 제공
   ├─ qm: QEMU/KVM VM 관리 명령어
   └─ 사용자 VM
      ├─ VM 100: Ubuntu 개발 서버
      └─ VM 101: Ubuntu 운영 서버
```

### 1.1 Proxmox VE

Proxmox VE는 Debian Linux를 기반으로 하는 가상화 관리 플랫폼이다. 웹 UI와 CLI를 통해 VM을 만들고 CPU, RAM, 디스크, 네트워크를 할당한다.

### 1.2 KVM

KVM은 가상머신 자체가 아니라 Linux 커널의 가상화 기술이다. 물리 CPU의 가상화 기능을 사용해 여러 VM이 CPU와 메모리를 나누어 사용하게 한다.

### 1.3 QEMU

QEMU는 각 VM에 다음과 같은 가상 하드웨어를 제공한다.

- 가상 CPU
- 가상 디스크
- 가상 네트워크 카드
- 가상 BIOS 또는 UEFI

Proxmox에서는 일반적으로 QEMU와 KVM이 함께 동작한다.

### 1.4 qm

`qm`은 Proxmox에서 QEMU/KVM VM을 관리하는 CLI 명령어다.

```bash
qm list          # VM 목록
qm config 100    # VM 100 설정 조회
qm start 100     # VM 100 시작
qm shutdown 100  # VM 100 정상 종료 요청
qm stop 100      # VM 100 강제 정지
qm destroy 100   # VM 100 삭제
```

`qm destroy`와 `qm stop`은 영향이 크므로 대상 VMID를 반드시 재확인해야 한다.

---

## 2. Proxmox 스토리지 구조

```text
물리 디스크
└─ Proxmox
   ├─ local
   │  └─ 일반 파일 저장 공간(/var/lib/vz)
   │     ├─ ISO 설치 이미지
   │     ├─ 디렉터리 방식 백업 파일
   │     └─ 컨테이너 템플릿
   ├─ local-lvm
   │  └─ LVM-thin 기반 VM 디스크 저장 공간
   │     ├─ VM 100 가상 디스크
   │     └─ VM 101 가상 디스크
   └─ pbs-remote
      └─ 네트워크로 연결된 원격 PBS Datastore
```

### 2.1 local

`local`은 기본적으로 `/var/lib/vz`를 사용하는 디렉터리 저장소다. 일반 파일시스템이므로 ISO, 템플릿, 디렉터리 방식 백업 파일을 저장할 수 있다.

```text
/var/lib/vz/
├─ dump/       VM 백업 파일
├─ template/   ISO와 컨테이너 템플릿
└─ images/     스토리지 설정에 따라 일부 VM 디스크
```

### 2.2 LVM

LVM(Logical Volume Manager)은 물리 디스크 공간을 논리 볼륨으로 나누어 관리하는 Linux 기능이다.

```text
물리 디스크
└─ LVM Volume Group
   ├─ Proxmox 시스템용 Logical Volume
   └─ VM 디스크용 Logical Volume
```

### 2.3 LVM-thin

LVM-thin은 VM에 설정한 최대 디스크 크기를 처음부터 모두 점유하지 않고, 실제로 기록된 블록을 중심으로 공간을 사용하는 방식이다.

```text
VM 가상 디스크 최대 크기: 500GiB
VM 내부 파일 사용량:      107GiB
LVM-thin 실제 기록량:     삭제 블록·스냅샷·Discard 상태에 따라 달라짐
```

VM 내부 `df -h` 값과 호스트의 LVM-thin 사용량은 항상 같지 않다. 게스트에서 파일을 삭제해도 Discard/TRIM이 전달되지 않았다면 호스트에서는 해당 블록을 계속 사용 중인 것으로 볼 수 있다.

### 2.4 local-lvm

`local-lvm`은 Proxmox가 LVM-thin 영역에 붙인 스토리지 이름이다. VM 디스크, 컨테이너 디스크, 스냅샷 데이터를 저장한다.

일반 디렉터리가 아닌 블록 스토리지이므로 `cp`, `tar` 같은 명령으로 일반 파일을 직접 저장할 수 없다.

### 2.5 pbs-remote

`pbs-remote`는 운영 Proxmox에 등록한 원격 PBS 스토리지 이름이다. 실제 데이터는 다른 물리 서버의 PBS `backup-store`에 저장된다.

운영 Proxmox에는 다음 연결 정보가 저장된다.

- PBS 주소: `192.168.10.105`
- Datastore: `backup-store`
- PBS 사용자와 인증 정보
- PBS 인증서 SHA-256 지문

따라서 로컬 스토리지처럼 `pvesm` 목록에 표시되지만 실제 저장 위치는 원격 서버다.

### 2.6 df와 pvesm의 차이

```bash
df -h
```

`df -h`는 마운트된 일반 파일시스템의 사용량을 확인한다. LVM-thin 블록 저장소인 `local-lvm`은 일반 파일시스템으로 마운트되지 않으므로 이 목록에 직접 나타나지 않는다.

```bash
pvesm status
```

`pvesm status`는 Proxmox에 등록된 스토리지의 유형, 상태, 전체 용량, 사용량, 여유 공간을 보여준다.

---

## 3. 주요 CLI 명령어 구분

| 명령어 | 역할 |
|---|---|
| `qm` | VM 생성, 설정 조회, 시작, 종료, 삭제 |
| `pvesm` | Proxmox 스토리지 등록, 조회, 상태 확인 |
| `vzdump` | VM 또는 컨테이너 백업 즉시 실행 |
| `qmrestore` | QEMU/KVM VM 백업 복원 |
| `pvesh` | Proxmox API를 CLI로 호출하고 백업 일정 등의 설정 관리 |
| `lvs` | 호스트의 LVM/LVM-thin 상태 확인 |

```text
VM 관리       → qm
스토리지 관리 → pvesm
즉시 백업     → vzdump
VM 복원       → qmrestore
일정 등록     → pvesh
```

---

## 4. VM과 호스트 자원 확인

### 4.1 VM 목록과 설정

```bash
qm list
qm config 100
qm config 101
```

### 4.2 VM 디스크 할당과 LVM-thin 사용률

```bash
lvs -o lv_name,lv_size,data_percent
```

예시 해석:

```text
vm-100-disk-0   4MiB   → VM 100의 UEFI 설정용 EFI 디스크
vm-100-disk-1 500GiB   → VM 100의 Ubuntu 시스템 디스크
vm-101-disk-0   4MiB   → VM 101의 UEFI 설정용 EFI 디스크
vm-101-disk-1 800GiB   → VM 101의 Ubuntu 시스템 디스크
```

더 상세한 LVM-thin 상태:

```bash
lvs -a -o lv_name,lv_size,data_percent,metadata_percent
```

### 4.3 실행 중인 VM 자원 사용량

```bash
pvesh get /nodes/$(hostname)/qemu --output-format yaml
```

### 4.4 Proxmox 호스트 전체 상태

```bash
pvesh get /nodes/$(hostname)/status --output-format yaml
```

### 4.5 ISO 목록

```bash
pvesm list local --content iso
```

---

## 5. VM 하드웨어 설정 개념

| 항목 | 의미 |
|---|---|
| Machine: `q35` | VM의 가상 메인보드 유형으로 PCIe 장치 지원에 적합 |
| BIOS: `OVMF` | VM의 UEFI 부팅 펌웨어 |
| EFI Storage | UEFI 부팅 설정을 저장하는 작은 EFI 디스크 위치 |
| VirtIO SCSI single | 가상화 환경에 최적화된 SCSI 디스크 컨트롤러 |
| QEMU Agent | Proxmox와 게스트 OS가 IP, 종료, 파일시스템 동결 정보를 교환하는 기능 |
| TPM | 암호화 키와 Secure Boot 관련 정보를 저장하는 가상 보안 장치 |
| Discard | 게스트에서 삭제된 블록을 호스트 LVM-thin까지 전달하는 기능 |
| IO thread | 가상 디스크 입출력을 별도 스레드에서 처리하는 기능 |
| FQDN | 호스트명과 도메인을 모두 포함한 전체 도메인 이름 |

QEMU Agent를 사용하려면 VM 옵션만 켜는 것으로 끝나지 않고 게스트 OS에 `qemu-guest-agent`가 설치·실행되어야 한다.

---

## 6. PBS 백업 구조

### 6.1 디렉터리 백업과 PBS 백업의 차이

Proxmox 디렉터리 스토리지로 백업하면 일반적으로 `.vma.zst` 같은 단일 압축 파일이 생성된다.

PBS 백업은 하나의 `.zst` 파일이 아니라 다음 구조로 저장된다.

- Chunk: VM 디스크 데이터를 나눈 작은 데이터 조각
- Metadata: 어떤 Chunk를 어떤 순서로 조합해야 하는지 기록한 정보
- Manifest: 백업 스냅샷의 구성과 무결성 정보

```text
VM 디스크
→ Chunk A + Chunk B + Chunk C
→ Metadata가 조합 순서와 VM 설정 기록
→ PBS가 복원 시 원래 VM 디스크로 재구성
```

같은 Chunk는 한 번만 저장할 수 있어 증분 백업과 중복 제거에 유리하다. PBS 저장 경로의 `.chunks`나 메타데이터를 직접 수정해서는 안 된다.

### 6.2 현재 PBS 배치 구조

```text
물리 서버 1: 운영 Proxmox(pve01)
└─ VM 101: 운영 서버
       │
       │ INT → DEV / TCP 8007
       ▼
물리 서버 2: 개발 Proxmox(pve)
└─ VM 104: PBS 192.168.10.105
   ├─ OS 디스크: 32GiB
   └─ 백업 디스크: 150GiB
      └─ Datastore: backup-store
```

PBS가 운영 VM과 다른 물리 서버에 있으므로 운영 Proxmox 호스트 장애 시에도 백업 데이터가 함께 사라지는 위험을 줄일 수 있다. 다만 PBS 물리 서버와 백업 디스크 자체의 장애까지 대비하려면 별도 복제나 추가 백업이 필요하다.

### 6.3 GPT와 파일시스템

```text
/dev/sdb 원시 디스크
→ GPT 파티션 테이블 생성
→ 파티션 생성
→ ext4 파일시스템 생성
→ PBS Datastore 등록
```

- GPT: 디스크 파티션 정보를 관리하는 형식
- ext4: Linux의 범용 파일시스템으로 관리와 복구가 비교적 단순함
- XFS: 대용량 병렬 입출력에 강하지만 파일시스템 축소를 지원하지 않음

---

## 7. PBS 구축 완료 내역

```text
[1] 서버 2 Proxmox 자원 확인
 → CPU, RAM, local, local-lvm 여유 공간 점검

[2] PBS 설치 ISO 등록
 → proxmox-backup-server_4.2-1.iso 다운로드

[3] PBS VM 생성
 → VM 104 / CPU 2 / RAM 2GiB / OS 디스크 32GiB

[4] PBS 4.2 설치
 → 관리 계정과 FQDN 설정

[5] PBS 네트워크 설정
 → 192.168.10.105/24 / Gateway 192.168.10.1 / TCP 8007

[6] PBS 관리 화면 확인
 → https://192.168.10.105:8007

[7] 별도 백업 디스크 추가
 → PBS VM에 150GiB 가상 디스크 연결

[8] 백업 디스크 초기화
 → GPT / ext4 파일시스템 생성

[9] PBS Datastore 생성
 → backup-store 등록

[10] FortiGate 내부망 정책 구성
 → INT(port2) 운영 Proxmox → DEV(port3) PBS TCP 8007 허용

[11] 운영 Proxmox에서 PBS 통신 확인
 → 192.168.0.109 → 192.168.10.105:8007

[12] PBS 백업 사용자 생성
 → pve-backup@pbs

[13] Datastore 권한 부여
 → /datastore/backup-store / DatastoreBackup

[14] PBS 인증서 지문 확인
 → SHA-256 Fingerprint 확보

[15] 운영 Proxmox에 PBS 등록
 → pbs-remote

[16] 원격 스토리지 연결 검증
 → pvesm status에서 pbs-remote active 확인

[17] 테스트 VM 생성 및 백업
 → VM 900 / 1GiB 디스크 / PBS 백업 성공
```

---

## 8. PBS 사용자와 권한 설정

다음 명령은 PBS VM의 Shell에서 실행한다.

### 8.1 사용자 생성

```bash
proxmox-backup-manager user create pve-backup@pbs
```

### 8.2 사용자 조회

```bash
proxmox-backup-manager user list
```

PBS GUI에서 다음 권한을 부여한다.

```text
Path: /datastore/backup-store
User: pve-backup@pbs
Role: DatastoreBackup
Propagate: 활성화
```

CLI와 GUI는 같은 PBS 설정을 사용하므로 CLI로 생성한 사용자도 GUI에 표시된다.

---

## 9. 운영 Proxmox에 PBS 등록

다음 명령은 운영 Proxmox 호스트 `pve01`에서 실행한다.

```bash
pvesm add pbs pbs-remote \
  --server 192.168.10.105 \
  --datastore backup-store \
  --username 'pve-backup@pbs' \
  --fingerprint '<PBS_SHA256_FINGERPRINT>' \
  --password
```

이 명령은 백업을 실행하는 것이 아니라 PBS의 `backup-store`를 `pbs-remote`라는 원격 스토리지로 등록한다.

비밀번호와 실제 인증서 지문은 문서나 Shell History에 평문으로 남기지 않는다.

### 9.1 등록 상태 확인

```bash
pvesm status
```

확인 결과:

```text
Name        Type  Status  Total       Used      Available   Usage
pbs-remote pbs   active  약 146GiB  약 259MiB 약 139GiB   0.17%
```

### 9.2 PBS 백업 목록 조회

```bash
pvesm list pbs-remote
```

백업 전에는 헤더만 출력되어도 정상이다.

---

## 10. 테스트 VM 백업 검증

운영 VM을 건드리지 않고 PBS 연결과 백업 기능을 확인하기 위해 임시 VM 900을 사용했다.

### 10.1 테스트 VM 생성

```bash
qm create 900 --name pbs-backup-test --memory 512 --cores 1
```

### 10.2 테스트 디스크 추가

```bash
qm set 900 --scsi0 local-lvm:1
```

생성 결과:

```text
local-lvm:vm-900-disk-0,size=1G
```

### 10.3 테스트 백업 실행

```bash
vzdump 900 --storage pbs-remote --mode snapshot
```

VM 900이 원래 정지 상태였기 때문에 실제 실행 로그에는 다음과 같이 표시됐다.

```text
status = stopped
backup mode: stop
Backup job finished successfully
```

`--mode snapshot`을 지정해도 대상 VM이 이미 정지되어 있으면 Proxmox는 정지 상태 그대로 백업한다.

테스트 디스크 전체가 비어 있어 다음과 같이 처리됐다.

```text
backup is sparse: 1.00 GiB (100%) total zero data
reused 1.00 GiB (100%)
```

이는 논리 디스크 크기는 1GiB지만 실제 저장할 데이터가 거의 없다는 뜻이다.

### 10.4 백업 생성 확인

```bash
pvesm list pbs-remote
```

확인 결과:

```text
pbs-remote:backup/vm/900/2026-07-23T07:09:11Z
```

PBS GUI에서는 다음 위치에서 확인할 수 있다.

```text
Datastore → backup-store → Content → vm/900
```

---

## 11. 복원 시험

백업 목록에 표시되는 것만으로는 실제 복구 가능성까지 완전히 검증한 것은 아니다. 별도 VMID로 복원하고 설정과 디스크를 확인해야 한다.

### 11.1 별도 VMID로 복원

```bash
qmrestore \
  pbs-remote:backup/vm/900/2026-07-23T07:09:11Z \
  901 \
  --storage local-lvm
```

이 명령은 백업된 VM 900을 새 VMID 901로 복원한다. 기존 VM 900과 운영 VM 101은 변경하지 않는다.

### 11.2 복원 후 확인 항목

```bash
qm config 901
qm start 901
qm status 901
```

실제 운영 VM을 별도 VMID로 복원할 경우 다음 충돌을 주의한다.

- 원본과 동일한 MAC 주소
- 게스트 OS 내부의 동일한 고정 IP
- 동일한 hostname
- 동일 서비스의 외부 포트 점유
- 운영 DB나 외부 시스템으로의 자동 연결

따라서 운영 VM 복원 시험은 NIC를 분리하거나 격리된 브리지에서 부팅하는 방식이 더 안전하다.

현재 복원 시험은 보류한 상태다.

---

## 12. 운영 VM 백업 방식

### 12.1 Snapshot 방식

```bash
vzdump 101 --storage pbs-remote --mode snapshot
```

- VM을 계속 실행한 상태로 백업할 수 있다.
- VM 중단은 피할 수 있지만 디스크 읽기와 네트워크 전송에 따른 I/O 부하는 발생한다.
- QEMU Guest Agent가 구성되어 있으면 파일시스템 정합성 확보에 도움이 된다.
- 애플리케이션과 DB의 완전한 트랜잭션 정합성은 별도 DB 백업 정책으로 보완한다.

### 12.2 Stop 방식

```bash
vzdump 101 --storage pbs-remote --mode stop
```

- VM을 종료한 상태로 백업한다.
- 정합성은 비교적 명확하지만 백업 시간 동안 서비스가 중단된다.
- 운영 서비스에서는 사전 점검과 승인 없이 사용하지 않는다.

PBS는 자체 청크 압축과 중복 제거를 사용하므로 일반적으로 `--compress zstd`를 별도로 지정하지 않는다.

---

## 13. 자동 백업 일정과 보존 정책

### 13.1 GUI 설정

```text
Datacenter → Backup → Add
```

```text
Storage: pbs-remote
Selection mode: Include selected VMs
VM: 백업 대상 VM
Schedule: 원하는 실행 시각
Mode: Snapshot
Retention / Keep Last: 보존 개수
Enabled: 활성화
```

### 13.2 CLI 설정 예시

VM 101을 매일 02:00에 Snapshot 방식으로 백업하고 최신 백업 하나만 유지하는 예시다.

```bash
pvesh create /cluster/backup \
  --storage pbs-remote \
  --vmid 101 \
  --schedule '02:00' \
  --mode snapshot \
  --prune-backups 'keep-last=1' \
  --enabled 1
```

`keep-last=1`은 저장 공간을 절약하지만 최신 백업에 문제가 있을 경우 이전 복구 지점이 없다. 가능하다면 용량을 확보해 두 개 이상의 복구 지점을 유지하는 편이 위험을 줄인다.

### 13.3 Prune과 Garbage Collection

- Prune: 보존 정책에서 제외된 오래된 백업 스냅샷을 제거 대상으로 정리
- Garbage Collection: 더 이상 어떤 백업에서도 참조하지 않는 Chunk를 실제로 정리
- Verify: 백업 Chunk와 메타데이터의 무결성을 검사

백업 항목을 삭제하거나 Prune했다고 해서 사용 공간이 즉시 모두 줄어들지 않을 수 있다. 실제 미참조 Chunk 회수는 Garbage Collection 과정에서 처리된다.

---

## 14. 월요일 운영 적용 준비 절차

운영 VM에 불필요한 영향을 주지 않기 위한 작업 순서다.

```text
[1] VMID 재확인
 → 개발 VM 100 / 운영 VM 101

[2] PBS 연결 확인
 → pvesm status에서 pbs-remote active 확인

[3] PBS 여유 공간 확인
 → 현재 약 139GiB 사용 가능

[4] 개발 VM 100 정상 종료
 → 종료 전 개발 서비스 이관 및 미사용 여부 확인

[5] 개발 VM 영향 관찰
 → 바로 삭제하지 않고 정지 상태로 유지

[6] 운영 VM 101 사전 점검
 → 상태, 디스크, 주요 서비스, DB 백업 확인

[7] 저부하 시간대에 운영 VM 수동 백업
 → vzdump 101 --storage pbs-remote --mode snapshot

[8] 백업 작업 결과 확인
 → Backup job finished successfully

[9] PBS 백업 목록 확인
 → pvesm list pbs-remote

[10] 운영 서비스 재점검
 → API, DB, MQTT, Nginx, 주요 컨테이너 상태 확인

[11] 자동 백업 일정 등록
 → 최초 수동 백업과 서비스 검증 후 적용
```

### 14.1 작업 전 주의사항

- `qm stop`, `qm destroy`, `vzdump --mode stop` 실행 전 VMID를 다시 확인한다.
- VM 100을 종료하더라도 즉시 삭제하지 않아 롤백 여지를 남긴다.
- 첫 운영 백업은 자동 일정이 아니라 수동으로 실행해 시간과 I/O 영향을 관찰한다.
- PBS 여유 공간이 부족하면 백업이 실패할 수 있으므로 실행 전 용량을 확인한다.
- 현재 PBS 백업 디스크는 150GiB이므로 운영 백업 크기에 따라 증설이 필요할 수 있다.
- DB 데이터는 VM 백업과 별도로 애플리케이션 일관성이 보장되는 DB 백업을 유지한다.
- 복원 시험을 완료하지 않은 백업은 실제 복구 가능성이 완전히 검증된 것으로 간주하지 않는다.

---

## 15. 작업 확인 명령어 모음

```bash
# VM 목록
qm list

# VM 설정
qm config 100
qm config 101

# Proxmox 스토리지 상태
pvesm status

# PBS 백업 목록
pvesm list pbs-remote

# LVM-thin 사용량
lvs -a -o lv_name,lv_size,data_percent,metadata_percent

# PBS Datastore 목록(PBS VM에서 실행)
proxmox-backup-manager datastore list

# 테스트 VM 백업
vzdump 900 --storage pbs-remote --mode snapshot

# 운영 VM Snapshot 백업
vzdump 101 --storage pbs-remote --mode snapshot

# 테스트 백업 복원
qmrestore <PBS_BACKUP_VOLID> 901 --storage local-lvm
```

---

## 16. 현재 진행 상태

| 항목 | 상태 |
|---|---|
| PBS VM 104 설치 | 완료 |
| PBS 네트워크 및 TCP 8007 연결 | 완료 |
| `backup-store` Datastore 생성 | 완료 |
| PBS 사용자 및 권한 설정 | 완료 |
| 운영 Proxmox에 `pbs-remote` 등록 | 완료 |
| 테스트 VM 900 생성 | 완료 |
| 테스트 VM 900 PBS 백업 | 완료 |
| 테스트 백업 목록 확인 | 완료 |
| 테스트 VM 901 복원 시험 | 보류 |
| 운영 VM 101 최초 수동 백업 | 미실행 |
| 운영 VM 자동 백업 일정 | 미설정 |

