---
layout: post
title: "Docker daemon API hang 장애 분석 및 복구"
excerpt: "active(running)인데 API 무응답 — 좀비 dockerd, stale docker.pid, Loki 로깅 드라이버 원인 추적"

categories:
  - Blog

tags:
  - [docker, devops, troubleshooting]

toc: true
toc_sticky: true

date: 2026-07-20
---

## 1. 문제

```bash
docker ps
docker version
```

docker 명령이 응답하지 않았다.

```bash
sudo curl --unix-socket /var/run/docker.sock http://localhost/_ping
```

docker socket에 직접 `_ping` 요청을 보내도 응답하지 않았다.

- docker CLI 자체는 실행이 되지만, docker daemon으로부터 `server` 정보를 얻오오지 못하는 문제가 발생
- loki를 컨테이너 로깅 드라이버로 채택한 이유로 몇 달에 1번씩 해당 문제가 발생을 하고 있다(문제는 loki 컨테이너가 죽어서 그런 것)
- 뭔가 도입 시 놓친 것이 있을 수도 있어서, 컨테이너 로깅 방법에 대해서는 다른 방법에 대한 검토가 필요함

## 2. 해결과정

결론적으로는 서비스 재부팅만으로는 안되서, 서버 자체를 재부팅하여 해결함

### 2.1 systemd 상태 확인

Docker 서비스 상태를 확인했다.

```bash
sudo systemctl status docker --no-pager
```

처음에는 다음처럼 보였다.

```text
Active: active (running)
Main PID: 2564062 (dockerd)
```

즉 systemd 입장에서는 Docker가 살아 있었다.

하지만 실제 Docker API는 응답하지 않았다.

이런 상태는 “프로세스 생존”과 “서비스 정상”이 다르다는 것을 보여준다.

### 2.2 containerd 상태 확인

containerd 상태도 확인했다.

```bash
sudo systemctl status containerd --no-pager
```

결과:

```text
Active: active (running)
Main PID: 894 (containerd)
```

containerd 자체는 살아 있었다.

이 시점에서는 `dockerd` 쪽 문제로 보는 것이 더 자연스러웠다.

### 2.3 Docker 재시작 시도

Docker를 재시작했다.

```bash
sudo systemctl restart docker
```

하지만 정상 종료되지 않았다.

상태는 다음처럼 바뀌었다.

```text
Active: deactivating (final-sigterm) (Result: timeout)
```

systemd 로그에는 다음 내용이 있었다.

```text
docker.service: State 'stop-sigterm' timed out. Killing.
docker.service: Killing process 2564062 (dockerd) with signal SIGKILL.
docker.service: Processes still around after SIGKILL. Ignoring.
```

의미:

- systemd가 Docker를 정상 종료하려고 SIGTERM을 보냄
- timeout 발생
- systemd가 SIGKILL을 보냄
- 그런데도 프로세스가 완전히 정리되지 않음

### 2.4 프로세스 상태 확인

남아 있는 `dockerd` PID를 확인했다.

```bash
ps -fp 2564062
```

결과:

```text
UID          PID    PPID  C STIME TTY          TIME CMD
root     2564062       1  1 May15 ?        13:52:20 [dockerd] <defunct>
```

`<defunct>`는 좀비 프로세스라는 뜻이다.

좀비 프로세스는 이미 죽은 프로세스다. 다만 부모 프로세스가 종료 상태를 회수하지 못해서 프로세스 테이블에 남아 있다.

중요한 점:

```text
kill -9로 좀비 프로세스를 제거할 수 없음
```

이미 죽었기 때문이다.

### 2.5 docker-proxy 상태 확인

systemd 상태에는 남은 `docker-proxy`도 보였다.

```text
1658672 [docker-proxy]
3042120 /usr/bin/docker-proxy ... host-port 9200 ...
```

상태를 확인했다.

```bash
ps -fp 1658672 3042120
```

초기 결과:

```text
root 1658672 2564062 Zl [docker-proxy] <defunct>
root 3042120 2564062 Sl /usr/bin/docker-proxy ...
```

하나는 이미 좀비였고, 하나는 살아 있었다.

살아 있던 `docker-proxy`에 종료 신호를 보냈다.

```bash
sudo kill -TERM 1658672 3042120
```

그래도 남아 있어 강제 종료했다.

```bash
sudo kill -KILL 1658672 3042120
```

이후 둘 다 좀비가 되었다.

```text
Zl [docker-proxy] <defunct>
```

이 상태에서는 더 이상 kill로 지울 수 없다.

### 2.6 Docker start 실패 원인 1: stale docker.pid

Docker를 다시 시작했다.

```bash
sudo systemctl reset-failed docker docker.socket
sudo systemctl start docker
```

하지만 실패했다.

로그를 확인했다.

```bash
sudo journalctl -xeu docker.service --no-pager -n 80
```

핵심 로그:

```text
failed to start daemon, ensure docker is not running or delete /var/run/docker.pid:
process with PID 2564062 is still running
```

`2564062`는 이미 좀비 `dockerd`였다.

PID 파일을 확인했다.

```bash
sudo cat /var/run/docker.pid
```

결과:

```text
2564062
```

즉 `/var/run/docker.pid`가 죽은 `dockerd` PID를 계속 가리키고 있었다.

Docker는 시작할 때 이 파일을 보고 “이미 Docker가 실행 중”이라고 판단하고 종료했다.

### 2.7 docker.pid 처리

소켓 파일을 건드린 것이 아니다.

건드린 파일:

```text
/var/run/docker.pid
```

건드리지 않은 파일:

```text
/var/run/docker.sock
/var/lib/docker
```

안전을 위해 삭제하지 않고 이름을 바꿨다.

```bash
sudo mv /var/run/docker.pid /var/run/docker.pid.stale-$(date +%Y%m%d-%H%M%S)
```

이 조치는 Docker 데이터, 이미지, 볼륨, 컨테이너 파일을 삭제하지 않는다. stale PID 파일만 치운 것이다.

## 3. 사용했던 명령어 정리

### 3.1 Docker 메타데이터 파일로 컨테이너 추적한 방법

Docker CLI가 멈춘 상황에서는 `docker inspect`, `docker ps`를 쓸 수 없다.

그래서 Docker 메타데이터 파일을 직접 확인했다.

컨테이너 ID:

```text
028e5ec3bea8449e09c402200968488c173bd7836bd7fa6031326b1b03c90d27
```

확인 명령:

```bash
sudo grep -R '"Name"' /var/lib/docker/containers/028e5ec3bea8449e09c402200968488c173bd7836bd7fa6031326b1b03c90d27/config.v2.json
```

결과에서 다음 정보를 확인했다.

```text
"Name":"/app-kiosk-update-server"
"Image":"app-kiosk-update-server-app"
"Running":true
```

즉 Docker CLI가 안 될 때도 `/var/lib/docker/containers/<container-id>/config.v2.json`을 보면 컨테이너 이름, 이미지, 네트워크, 마운트 정보를 일부 확인할 수 있다.

### 3.2 docker api 확인

```bash
sudo curl --unix-socket /var/run/docker.sock http://localhost/_ping
```

### 3.3 Docker 서비스 상태 확인

```bash
sudo systemctl status docker --no-pager
```

### 3.4 Docker 실패 상태 초기화

```bash
sudo systemctl reset-failed docker docker.socket
```

목적:

```text
systemd의 failed 상태와 start limit 상태 초기화
```

### 3.5 Docker 로그 확인

```bash
sudo journalctl -xeu docker.service --no-pager -n 80
```

목적:

```text
Docker start/stop 실패 원인 확인
```

### 3.6 파일 읽기 I/O 확인

```bash
sudo timeout 5s dd if=/var/lib/docker/volumes/metadata.db of=/dev/null bs=64K status=progress
```

목적:

```text
metadata.db 파일 읽기 자체가 지연되는지 확인
```

### 3.7 컨테이너별 logging driver 확인

Docker가 복구된 후 컨테이너별 logging driver를 확인할 수 있다.

```bash
docker inspect app-kiosk-update-server --format '{{json .HostConfig.LogConfig}}'
```

다른 컨테이너도 확인한다.

```bash
docker inspect app-kiosk-iot-server --format '{{json .HostConfig.LogConfig}}'
docker inspect nginx --format '{{json .HostConfig.LogConfig}}'
docker inspect postgres --format '{{json .HostConfig.LogConfig}}'
docker inspect redis --format '{{json .HostConfig.LogConfig}}'
```

예상되는 Loki 설정 형태:

```json
{
  "Type": "loki",
  "Config": {
    "loki-url": "http://192.168.0.110:3100/loki/api/v1/push",
    "loki-external-labels": "service=app,env=dev"
  }
}
```

## 4. Docker 구성 요소 기초

### 4.1 docker CLI

사용자가 터미널에서 실행하는 명령이다.

```bash
docker ps
docker version
docker inspect ...
```

`docker` 명령어 자체는 클라이언트다. 직접 컨테이너를 관리하지 않는다. 실제 작업은 Docker daemon에게 요청한다.

### 4.2 dockerd

Docker daemon이다.

프로세스 이름은 보통 `dockerd`다.

```bash
ps -ef | grep dockerd
```

컨테이너 생성, 삭제, 네트워크, 볼륨, 이미지, 로그 드라이버 등을 총괄한다.

이번 장애의 핵심은 `dockerd` 프로세스가 살아 있는 것처럼 보였지만 Docker API 요청에 응답하지 않았다는 점이다.

### 4.3 docker.sock

Docker CLI와 Docker daemon이 통신하는 Unix socket이다.

경로는 보통 다음과 같다.

```text
/var/run/docker.sock
```

Docker CLI는 이 소켓을 통해 `dockerd`에게 요청을 보낸다.

이번 장애에서는 다음 명령이 응답하지 않았다.

```bash
sudo curl --unix-socket /var/run/docker.sock http://localhost/_ping
```

정상이라면 아래처럼 나와야 한다.

```text
OK
```

이 명령이 멈췄다는 것은 Docker CLI 문제가 아니라 Docker daemon API 응답 경로가 막혔다는 뜻이다.

### 4.4 docker.pid

Docker daemon의 PID를 기록하는 파일이다.

경로는 다음과 같다.

```text
/var/run/docker.pid
```

파일 내용은 단순히 숫자 하나다.

예:

```text
2564062
```

이 숫자는 현재 실행 중인 `dockerd`의 PID여야 한다.

이번 장애에서는 `dockerd`가 좀비 프로세스가 되었는데도 `/var/run/docker.pid`가 이전 PID `2564062`를 계속 가리키고 있었다. 그래서 새 `dockerd`가 뜰 때 다음 오류가 발생했다.

```text
failed to start daemon, ensure docker is not running or delete /var/run/docker.pid:
process with PID 2564062 is still running
```

### 4.5 containerd

Docker보다 아래 레벨에서 컨테이너 런타임을 관리하는 프로세스다.

```bash
sudo systemctl status containerd --no-pager
```

Docker는 내부적으로 containerd와 통신한다.

이번 장애에서는 `containerd` 자체는 살아 있었다.

```text
containerd.service - containerd container runtime
Active: active (running)
```

따라서 처음에는 containerd 전체 장애라기보다는 `dockerd` API 처리 경로 문제로 판단했다.

### 4.6 docker-proxy

Docker가 컨테이너 포트를 호스트 포트로 노출할 때 사용하는 보조 프로세스다.

예:

```text
/usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 9200 -container-ip 172.24.0.4 -container-port 9200
```

이 의미는 다음과 같다.

```text
호스트 0.0.0.0:9200
→ 컨테이너 172.24.0.4:9200
```

## 5. 문제 원인 파악

- loki 로깅 드라이버로 변경을 했으나, 해당 컨테이너 관리가 미흡했던 것으로 보임(갑자기 꺼져 있었음)
- 문제를 해결하면서 서비스를 계속 끄다보니 dockerd 좀비 프로세스 발생

### 5.1 loki 컨테이너 불안정

```text
error sending batch, will retry
Post "http://192.168.0.110:3100/loki/api/v1/push":
dial tcp 192.168.0.110:3100: connect ...
```

의미:

```text
컨테이너 로그를 Loki로 보내려고 함
→ 대상 Loki 192.168.0.110:3100 접속 실패
→ logging driver가 재시도
```
이것은 이번 Docker API hang의 직접 원인이라고 단정할 수는 없지만, 장애를 유발하거나 악화시켰을 가능성이 있는 중요한 후보로 남는다.

## 6. 얻은 것 

### 6.1 active running만 믿지 않기

```bash
sudo systemctl status docker --no-pager
```

여기서 `active (running)`이어도 Docker API가 죽어 있을 수 있다.

### 6.2 docker ps 무응답이면 바로 Docker API 확인

```bash
sudo curl --unix-socket /var/run/docker.sock http://localhost/_ping
```

이게 멈추면 `docker ps` 문제가 아니라 Docker daemon API 문제다.

### 6.3 stale docker.pid는 Docker start를 막을 수 있음

`/var/run/docker.pid`가 죽은 PID를 가리키면 새 Docker daemon이 뜨지 못한다.

## 7. 결론 

이번 장애의 핵심 흐름은 다음과 같다.

```text
docker ps 무응답
→ docker.sock _ping 무응답
→ dockerd API hang 판단
→ Docker restart 시도
→ dockerd 좀비화
→ docker-proxy 좀비 잔존
→ stale /var/run/docker.pid 때문에 Docker start 실패
→ stale pid 파일 이동
→ volume metadata.db timeout 발생
→ 재부팅
→ Docker API 복구
→ Loki logging driver 접속 실패 경고 확인
```

최종 복구 확인 근거:

```text
sudo curl --unix-socket /var/run/docker.sock http://localhost/_ping
→ OK

sudo systemctl status docker --no-pager
→ Active: active (running)
```

남은 핵심 검토 대상:

```text
Docker Loki logging driver
192.168.0.110:3100 Loki endpoint 상태
app-kiosk-update-server 정상 종료 처리
배포 시 Docker daemon 부담
```