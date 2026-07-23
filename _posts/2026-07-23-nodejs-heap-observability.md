---
layout: post
title: "Node.js Heap 메모리 관측 기능을 직접 만든 과정"
excerpt: "컨테이너 OOM 장애를 파고들며 Node.js heap 메모리를 직접 관측 가능하게 만든 기록"

categories:
  - Blog

tags:
  - [nodejs, observability, docker]

toc: true
toc_sticky: true

date: 2026-07-23
---

# Node.js Heap Memory 관측 기능을 직접 만든 과정

## 상황 정리 및 해결

### 01. Nodejs 메모리 부족에 의한 컨테이너 장애

mqtt 부하 테스트 중 발견 한, nodejs 메모리 부족 현상에 대한 정리.
mqtt 부하 테스트 실행 시, 요청을 처리하는 컨테이너가 죽으면서 재시작되는 현상 발견.

### 02. 처음에는 컨테이너가 죽은 것처럼 보임

```text
OOMKilled=false
```

Docker가 cgroup memory limit 초과로 컨테이너를 죽인 경우라면 보통 `OOMKilled=true`가 나온다.

그런데 false였다.

반면 로그에는 다음과 같은 문구가 있었다.

```text
FATAL ERROR: Ineffective mark-compacts near heap limit
Allocation failed - JavaScript heap out of memory
```

이 문구는 Docker가 죽였다는 뜻이 아니다.

Node.js 내부에서 V8 heap이 한계에 가까워졌고, GC를 해도 충분히 회수하지 못해서, Node.js 프로세스가 스스로 abort했다는 쪽에 가깝다.

### 03. Docker 메모리와 nodejs 메모리

Docker나 cAdvisor가 보여주는 컨테이너 메모리는 보통 컨테이너 전체가 얼마나 메모리를 쓰는지 알려준다.

하지만 Node.js 내부에서 어떤 영역이 늘고 있는지는 말해주지 않는다.

예를 들어 컨테이너 메모리가 증가했다고 하자.

이때 가능한 원인은 여러 가지다.

```text
1. JavaScript 객체가 계속 쌓인다.
2. Buffer가 계속 쌓인다.
3. native memory가 늘어난다.
4. malloc fragmentation 때문에 RSS가 내려오지 않는다.
5. 처리 중인 MQTT payload가 너무 많다.
6. Redis/Postgres 대기 때문에 큰 객체 참조가 오래 유지된다.
```

다양한 원인들이 존재하지만, `OOMKilled=false` 였던 것처럼 원인을 정확하게 알 수는 없다.

컨테이너 메모리 구조 분석

```text
Host memory
  -> Docker container memory
    -> Node.js process RSS
      -> V8 heap
      -> external memory
      -> ArrayBuffer / Buffer
      -> native memory
```





### 04. process.memoryUsage()

Node.js는 현재 프로세스의 메모리 상태를 볼 수 있는 API를 제공한다.

그중 가장 기본이 `process.memoryUsage()`다.

예시는 다음과 같다.

```js
process.memoryUsage()
```

이 함수는 대략 이런 값을 반환한다.

```js
{
  rss: 104689664,
  heapTotal: 46411776,
  heapUsed: 39577128,
  external: 6954878,
  arrayBuffers: 3505080
}
```

각 값은 의미가 다르다.

`rss`는 운영체제가 보는 Node.js 프로세스의 실제 메모리 점유량에 가깝다.

`heapTotal`은 V8이 확보해 둔 heap 영역이다.

`heapUsed`는 V8 heap 안에서 실제 JavaScript 객체가 사용 중인 메모리다.

`external`은 V8 heap 바깥에 있지만 JavaScript 객체와 연결된 메모리다.

`arrayBuffers`는 ArrayBuffer와 Buffer 계열 메모리다.

MQTT 서버에서는 이 값들이 중요했다.

MQTT payload는 처음에 Buffer로 들어온다.

그 다음 문자열로 변환된다.

그리고 JSON.parse를 통해 JavaScript 객체가 된다.

그 뒤 Redis summary merge, JSON.stringify, Postgres 저장까지 이어질 수 있다.

즉 하나의 payload가 처리 중에 여러 형태로 존재할 수 있다.

```text
Buffer
-> string
-> parsed object
-> merged object
-> stringified JSON
-> DB parameter
```

그래서 단순히 컨테이너 메모리만 볼 것이 아니라,
Node.js 프로세스 내부에서 heap이 늘었는지,
Buffer 계열이 늘었는지,
RSS만 늘었는지를 나눠 봐야 했다.

이 판단 때문에 `process.memoryUsage()`를 직접 호출하는 endpoint를 만들었다.

---

### 05. v8.getHeapStatistics()

`process.memoryUsage()`만으로는 부족한 부분이 있다.

가장 중요한 값은 V8 heap의 한계다.

Node.js는 V8 엔진 위에서 JavaScript를 실행한다.

V8은 JavaScript 객체를 저장하는 heap을 따로 관리한다.

이 heap에는 한계가 있다.

컨테이너에 명시적인 memory limit이 없더라도,
V8 heap이 먼저 한계에 닿으면 Node.js는 `JavaScript heap out of memory`로 죽을 수 있다.

이 한계를 확인하기 위해 사용한 API가 다음이다.

```js
const v8 = require('v8');
v8.getHeapStatistics();
```

여기서 중요한 값은 `heap_size_limit`이다.

예를 들어 다음과 같은 값이 나올 수 있다.

```js
{
  heap_size_limit: 549453824
}
```

또는 런타임 조건에 따라 더 큰 값이 나올 수 있다.

이 값은 V8 heap이 커질 수 있는 최대 크기를 의미한다.

따라서 heap 사용률은 다음처럼 계산할 수 있다.

```text
heapUsed / heapSizeLimit * 100
```

Prometheus에서는 다음처럼 볼 수 있다.

```promql
100 * iot_nodejs_heap_used_bytes / iot_nodejs_heap_size_limit_bytes
```

이 쿼리를 만들기 위해서는 `heapUsed`뿐 아니라 `heap_size_limit`도 metric으로 노출해야 했다.

그래서 `process.memoryUsage()`와 `v8.getHeapStatistics()`를 같이 호출했다.

---

### 06. `/debug/memory`와 `/metrics/nodejs`를 분리한 이유

메모리 값을 노출하는 방법은 두 가지 목적이 있었다.

첫 번째는 사람이 바로 확인하는 목적이다.

두 번째는 Prometheus가 주기적으로 수집하는 목적이다.

이 둘은 출력 형식이 다르다.

사람이 볼 때는 JSON이 좋다.

Prometheus가 수집할 때는 Prometheus text exposition format이 필요하다.

그래서 endpoint를 분리했다.

```text
/debug/memory
/metrics/nodejs
```

`/debug/memory`는 사람이 curl로 확인하기 위한 API다.

예상 형태는 다음과 같다.

```bash
curl http://app-kiosk-iot-server:9200/debug/memory
```

이 API는 JSON으로 메모리 상태를 보여준다.

예를 들면 다음과 같은 구조다.

```json
{
  "memoryUsage": {
    "rss": 104689664,
    "heapTotal": 46411776,
    "heapUsed": 39577128,
    "external": 6954878,
    "arrayBuffers": 3505080
  },
  "heapStatistics": {
    "heap_size_limit": 549453824
  }
}
```

반면 `/metrics/nodejs`는 Prometheus가 읽기 위한 endpoint다.

예상 호출은 다음과 같다.

```bash
curl http://app-kiosk-iot-server:9200/metrics/nodejs
```

출력은 이런 형태다.

```text
# HELP iot_nodejs_process_rss_bytes Node.js process resident set size in bytes.
# TYPE iot_nodejs_process_rss_bytes gauge
iot_nodejs_process_rss_bytes 104689664

# HELP iot_nodejs_heap_total_bytes V8 heap total size in bytes.
# TYPE iot_nodejs_heap_total_bytes gauge
iot_nodejs_heap_total_bytes 46411776

# HELP iot_nodejs_heap_used_bytes V8 heap used size in bytes.
# TYPE iot_nodejs_heap_used_bytes gauge
iot_nodejs_heap_used_bytes 39577128
```

사람용 API와 수집용 API를 분리한 것은 운영상 의미가 있다.

사람용 API는 상세하고 읽기 쉬워야 한다.

수집용 API는 Prometheus가 안정적으로 scrape할 수 있어야 한다.

### 07. 코드 구현

현재 코드 기준으로는 다음 파일들이 근거가 된다.

```text
src/routes/debug.js
src/routes/nodeMetrics.js
src/app.js
```

`src/routes/debug.js`에는 사람이 직접 확인하는 메모리 API가 있다.

핵심 호출은 다음 두 가지다.

```js
process.memoryUsage()
v8.getHeapStatistics()
```

`src/routes/nodeMetrics.js`에는 Prometheus 수집용 endpoint가 있다.

핵심 path는 다음이다.

```text
/metrics/nodejs
```

그리고 다음 metric들이 노출된다.

```text
iot_nodejs_process_rss_bytes
iot_nodejs_heap_total_bytes
iot_nodejs_heap_used_bytes
iot_nodejs_external_memory_bytes
iot_nodejs_array_buffers_bytes
iot_nodejs_heap_size_limit_bytes
iot_nodejs_active_handles
iot_nodejs_active_requests
iot_nodejs_process_uptime_seconds
iot_nodejs_process_start_time_seconds
```

이 지표들은 단순히 라이브러리를 켠 것이 아니라,
Node.js 런타임 API를 직접 호출해서 만든 값이다.

이 점이 중요하다.

cAdvisor는 컨테이너 바깥에서 컨테이너 메모리를 본다.

node-exporter는 호스트 바깥에서 서버 메모리를 본다.

하지만 `/metrics/nodejs`는 애플리케이션 프로세스 안에서 직접 값을 읽는다.

즉, 관측 위치가 다르다.

## Prometheus에서 이 값을 수집하는 구조

Prometheus는 자기 마음대로 Node.js 내부 메모리를 알 수 없다.

Prometheus는 HTTP endpoint를 주기적으로 호출한다.

이 과정을 scrape라고 한다.

구조는 다음과 같다.

```text
Prometheus
  -> http://app-kiosk-iot-server:9200/metrics/nodejs
    -> Node.js app
      -> process.memoryUsage()
      -> v8.getHeapStatistics()
      -> Prometheus text format 응답
```

즉 Prometheus가 직접 heap을 읽는 것이 아니다.

애플리케이션이 heap 값을 읽어서 `/metrics/nodejs`로 제공한다.

Prometheus는 그 값을 가져갈 뿐이다.

이 차이를 이해해야 한다.

`http://cadvisor:8080/metrics`는 cAdvisor 컨테이너가 제공한다.

`http://node-exporter:9100/metrics`는 node-exporter 컨테이너가 제공한다.

`http://app-kiosk-iot-server:9200/metrics/nodejs`는 app-kiosk-iot-server 애플리케이션이 제공한다.

따라서 `/metrics/nodejs`는 내가 만든 애플리케이션 계층 metric에 가깝다.

---

## Grafana 연동

이 작업은 단순히 metric을 많이 만들기 위한 작업이 아니었다.

운영 중에 답해야 할 질문이 있었다.

첫 번째 질문.

```text
컨테이너가 죽은 것이 Docker OOMKill 때문인가, Node.js heap OOM 때문인가?
```

두 번째 질문.

```text
Node.js 프로세스의 실제 RSS가 늘고 있는가?
```

세 번째 질문.

```text
V8 heap 사용률이 heap limit에 가까워지고 있는가?
```

네 번째 질문.

```text
Buffer나 ArrayBuffer 계열 메모리가 늘고 있는가?
```

다섯 번째 질문.

```text
프로세스가 최근에 재시작됐는가?
```

여섯 번째 질문.

```text
active handles가 계속 증가해서 socket/timer 누수가 의심되는가?
```

이 질문에 답하기 위해 metric을 만들었다.

그래서 Grafana 패널도 단순한 CPU/Memory 패널이 아니라,
Node.js 내부 상태를 구분하는 패널이 필요했다.

---

### 01. 핵심 PromQL

V8 heap 사용률은 다음으로 본다.

```promql
100 * iot_nodejs_heap_used_bytes / iot_nodejs_heap_size_limit_bytes
```

Node.js 프로세스 RSS는 다음으로 본다.

```promql
iot_nodejs_process_rss_bytes
```

external memory는 다음으로 본다.

```promql
iot_nodejs_external_memory_bytes
```

ArrayBuffer/Buffer 계열은 다음으로 본다.

```promql
iot_nodejs_array_buffers_bytes
```

프로세스 재시작 여부는 uptime으로 본다.

```promql
iot_nodejs_process_uptime_seconds
```

최근 5분 이내 재시작 여부를 보고 싶으면 다음처럼 볼 수 있다.

```promql
iot_nodejs_process_uptime_seconds < 300
```

active handles는 다음으로 본다.

```promql
iot_nodejs_active_handles
```

active requests는 다음으로 본다.

```promql
iot_nodejs_active_requests
```

---

### 02. 이 작업으로 구분할 수 있게 된 것

이전에는 이런 식으로 볼 수 있었다.

```text
컨테이너가 떠 있는가?
컨테이너가 재시작됐는가?
컨테이너 메모리가 높은가?
```

하지만 이 정도로는 부족했다.

이 작업 이후에는 다음 질문에 더 가까이 갈 수 있게 됐다.

```text
Node.js 프로세스 RSS가 늘었는가?
V8 heap이 늘었는가?
V8 heap limit에 가까운가?
Buffer 계열이 늘었는가?
프로세스가 방금 재시작됐는가?
```

즉, 장애 원인을 더 좁힐 수 있게 됐다.

컨테이너 관측에서 애플리케이션 런타임 관측으로 들어간 것이다.

---

### 03. 이 작업이 직접 만든 것이라고 말할 수 있는 이유

첫째, 문제를 컨테이너 레벨에서 끝내지 않았다.

Docker inspect와 로그를 보고 `OOMKilled=false`와 V8 fatal error를 구분했다.

둘째, Node.js 런타임 내부 값을 직접 읽었다.

`process.memoryUsage()`와 `v8.getHeapStatistics()`를 사용했다.

셋째, 사람용 endpoint와 Prometheus용 endpoint를 나눴다.

`/debug/memory`는 운영자가 직접 확인하기 위한 endpoint다.

`/metrics/nodejs`는 Prometheus scrape를 위한 endpoint다.

넷째, Prometheus text exposition format으로 직접 지표를 노출했다.

다섯째, Grafana에서 볼 질문을 기준으로 metric을 설계했다.

여섯째, 이 작업은 단순한 라이브러리 설치가 아니라 장애 원인 분리를 위한 관측 경로 설계였다.

## 운영자가 실제로 확인하는 흐름

장애가 발생하면 다음 순서로 본다.

1. 컨테이너가 재시작됐는지 확인한다.

```bash
docker inspect app-kiosk-iot-server --format 'RestartCount={{.RestartCount}} OOMKilled={{.State.OOMKilled}} ExitCode={{.State.ExitCode}}'
```

2. 로그에서 V8 heap OOM 문구를 확인한다.

```bash
docker logs app-kiosk-iot-server | grep -E 'heap out of memory|Ineffective mark-compacts|Allocation failed'
```

3. 사람이 직접 메모리 상태를 본다.

```bash
curl http://127.0.0.1:9200/debug/memory
```

4. Prometheus용 metric이 열려 있는지 본다.

```bash
curl http://127.0.0.1:9200/metrics/nodejs
```

5. Prometheus에서 heap 사용률을 본다.

```promql
100 * iot_nodejs_heap_used_bytes / iot_nodejs_heap_size_limit_bytes
```

6. RSS와 heap을 같이 본다.

```promql
iot_nodejs_process_rss_bytes
```

```promql
iot_nodejs_heap_used_bytes
```

7. Buffer 계열을 본다.

```promql
iot_nodejs_external_memory_bytes
```

```promql
iot_nodejs_array_buffers_bytes
```

8. 재시작 여부를 uptime으로 확인한다.

```promql
iot_nodejs_process_uptime_seconds
```

이 흐름은 단순한 확인 명령 모음이 아니다.

바깥에서 안쪽으로 좁혀 들어가는 방식이다.

```text
Host
-> Container
-> Process
-> V8 heap
-> Buffer/external
-> App 처리 경로
```

---

## cAdvisor, node-exporter와 이 작업의 차이

cAdvisor는 컨테이너를 본다.

node-exporter는 호스트를 본다.

이번에 만든 `/metrics/nodejs`는 애플리케이션 프로세스 내부를 본다.

표로 보면 다음과 같다.

| 구분 | 누가 제공하는가 | 무엇을 보는가 |
|---|---|---|
| node-exporter | node-exporter 컨테이너 | 호스트 CPU, memory, disk, network |
| cAdvisor | cAdvisor 컨테이너 | Docker 컨테이너 CPU, memory, network |
| /metrics/nodejs | app-kiosk-iot-server | Node.js process RSS, V8 heap, Buffer, uptime |

그래서 서로 대체 관계가 아니다.

서로 보완 관계다.

node-exporter가 없으면 서버 전체 상태를 놓친다.

cAdvisor가 없으면 컨테이너 단위 상태를 놓친다.

`/metrics/nodejs`가 없으면 Node.js 내부 heap 상태를 놓친다.

---

## 왜 이 문서가 증빙이 되는가

이 문서가 증빙으로 의미가 있는 이유는 단순히 “메트릭을 추가했다”가 아니라,
문제 인식부터 설계 판단까지 연결되어 있기 때문이다.

문제 인식:

```text
컨테이너가 재시작되지만 Docker OOMKilled는 false였다.
```

분석:

```text
로그는 V8 heap out of memory를 가리켰다.
```

부족한 점:

```text
cAdvisor와 Docker inspect만으로는 Node.js 내부 heap 상태를 알 수 없었다.
```

설계:

```text
process.memoryUsage()와 v8.getHeapStatistics()를 직접 호출한다.
```

구현:

```text
/debug/memory와 /metrics/nodejs를 추가한다.
```

운영 연결:

```text
Prometheus scrape와 Grafana 패널로 연결한다.
```

운영 해석:

```text
RSS, heapUsed, heapSizeLimit, external, ArrayBuffer, uptime을 조합해 본다.
```

이 흐름이 있어야 “직접 만들었다”는 설명이 설득력을 가진다.

---
