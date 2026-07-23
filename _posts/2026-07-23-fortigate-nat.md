---
layout: post
title: "FortiGate NAT (Source NAT) 이해하기"
excerpt: "패킷의 출발지 IP를 언제 바꾸고 안 바꾸는가 — FortiGate SNAT 옵션 정리"

categories:
  - Blog

tags:
  - [network, fortigate, nat]

toc: true
toc_sticky: true

date: 2026-07-23
---

## FortiGate - NAT 설정

NAT = Source NAT(출발지 IP 바꾸기)

패킷의 "보내는 사람 IP"를 FortiGate가 바꾸느냐 마느냐를 결정하는 옵션

### 비유

- **NAT OFF** = 편지 봉투의 "보내는 사람" 주소를 그대로 둔다 → 받는 서버가 진짜 발신자(인터넷 클라이언트/Let's Encrypt) IP**를 봄.
- **NAT ON** = FortiGate가 "보내는 사람"을 자기 주소로 바꿔서 전달 → 받는 서버는 FortiGate가 보낸 것처럼 봄 (실제 발신자 IP 안 보임).

### 켜고 끄면 뭐가 달라지나

|  | **NAT OFF** | **NAT ON** |
| --- | --- | --- |
| 백엔드가 보는 출발지 | **실제 클라이언트 IP** | **FortiGate IP** |
| 로그/추적 | 진짜 IP 보임 | 누가 왔는지 가려짐 |
| 리턴 경로 | 백엔드 게이트웨이가 FortiGate라 자연히 되돌아감 | FortiGate로 강제로 돌아옴(비대칭 라우팅 방지) |
| 주 용도 | **인바운드(밖→안, VIP)** | **아웃바운드(안→인터넷 마스커레이드)** |

### 정리

- 인바운드 VIP는 보통 NAT OFF.
- 반대로 아웃바운드는 선택.
