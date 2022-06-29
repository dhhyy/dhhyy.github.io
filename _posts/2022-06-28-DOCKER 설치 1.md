---
layout: post
# title: [TIL_20220628]
excerpt: "개발 환경 셋팅을 위한 docker 설치"

categories:
  - Blog
  
tags:
  - [docker]

toc: true
toc_sticky: true
 
date: 2022-06-28
last_modified_at: 2022-06-28
---

<!--
2022-06-28-DOCKER 설치
-->

_docker에 대한 전반적인 내용 정리와 함께 공부한 내용을 기록합니다._
_[[실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker)] 거의 이 블로그를 보고 참고하였습니다. 인용한 경우에도 출처 남겨 놓았습니다._
<br>

## docker를 사용하는 이유

> 도커는 리눅스 상에서 컨테이너 방식으로 **프로세스를 격리해서 실행하고 관리**할 수 있도록 도와주며, **계층화**된 파일 시스템에 기반해 효율적으로 이미지(프로세스 실행 환경)을 구축할 수 있도록 해줍니다. 도커를 사용하면 이 이미지를 기반으로 컨테이너를 실행할 수 있으며, 다시 특정 컨테이너의 상태를 변경해 이미지로 만들 수 있습니다. 이렇게 만들어진 이미지는 파일로 보관하거나 원격 저장소를 사용해 쉽게 공유할 수 있으며, **도커만 설치되어 있다면 필요할 때 언제 어디서나 컨테이너로 실행하는 것이 가능**합니다.
> 
> 출처 : [[실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker)]

## docker의 컨테이너와 VM의 차이

![[Pasted image 20220628144807.png]]

하이퍼바이저 : Virtual Box 나 VM Ware 와 같은 가상 머신을 생성하고 실행하는 프로세스

docker와 VM의 가장 큰 차이점은 하이퍼바이저의 유무이다. 하이퍼바이저는 하드웨어를 에뮬레이션하여 하나의 컴퓨터에서 다수의 운영체제를 운영할 수 있게 도와주는 소프트웨어이다.

![[스크린샷 2022-06-28 오후 9.31.31.png]]

도커의 경우, 도커의 컨테이너는 Kernel 데이터를 가지고 있지 않으며, 컨테이너가 필요로 하는 Kernel 데이터는 호스트 OS의 것을 그대로 사용한다. 그리고 Kernel 외 데이터만을 패키징하여 컨테이너가 가지고 있다.

그림에서처럼 가상화 단계를 하나 거치지 않고 도커 내부 엔진으로 바로 운영이 되기에, 도커가 효율적이라고 한다.

## docker Hub에서 이미지 검색하고 실행하기
### docker search

```bash
❯ docker search CentOS                                                                                               
NAME    DESCRIPTION     STARS     OFFICIAL   AUTOMATED
centos          
centos/systemd        
```

`docker search {필요한 이미지명}` 명령어로 다운 가능한 이미지 확인 가능. 

### docker pull

```bash
❯ docker pull centos                                                                                               
Using default tag: latest
latest: Pulling from library/centos
52f9ef134af7: Pull complete
Digest: sha256:a27fd8080b517143cbbbab9dfb7c8571c40d67d534bbdee55bd6c473f432b177
Status: Downloaded newer image for centos:latest
docker.io/library/centos:latest
```

애초에 컨테이너 실행과 이미지 다운로드를 함께 할 수도 있고 따로 따로 할 수도 있음. 나의 경우에는 pull로 다운 받고 이미지 확인하기

### docker run

`docker run {이미지명:tag}` 이런 식으로 run 명령어를 실행하면 컨테이너까지 바로 실행할 수 있다.  

**run : Run a command in a new container**

`docker run --help` 를 찍어 확인하니 새로운 컨테이너에서 실행한다는 의미로 쓰이는 것 같다.

### docker images

```bash
❯ docker images                                                                           

REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
centos       latest    e6a0117ec169   9 months ago   272MB
```

`docker images` 해당 명령어 통해 현재 이미지 리스트 확인 가능

## 에러 상황 정리
### Cannot connect to the Docker daemon at unix

``` 
❯ docker run ubuntu:latest                    
docker: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?.
See 'docker run --help'.
```

간단하다. background로 docker desktop이 실행되지 않은 상태에서 터미널 명령을 수행하지 못하는 것 같다. 나의 경우엔 docker desktop을 실행함과 동시에 해당 에러는 해결이 되었다. background 실행이 되어야 도커 소켓에 접근할 권한이 생긴다고 이해하면 될 것 같다.

### Error: Failed to download metadata for repo 'appstream': Cannot prepare internal mirrorlist: No URLs in mirrorlist

패키지 매니저 apt-get을 지원하지 않는 CentOS에서 apt-get을 실행했을 때 발생하는 에러.

### WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested

실행할 컨테이너의 환경이 m1과 호환되지 않아 발생하는 에러.

```bash
docker run --platform linux/amd64 -it ansible/centos7-ansible:latest bash
```

이와 같이 명령어로 해당 컨테이너의 실행 환경 플랫폼을 지정해주면 해결.

## 새로운 이미지 만들기
### docker commit

```bash
docker commit 33eb583946a2 ubuntu:yum-package
````

컨데이너까지 한 번에 묶는 것. 새로운 이미지로 만든다.

### docker images, docker ps로 만ㅁ들어진 이미지 확인하기

```bash
❯ docker images                                                                                 
REPOSITORY                TAG           IMAGE ID       CREATED         SIZE
ubuntu                    yum-package   39676984431f   8 minutes ago   885MB
centos                    latest        e6a0117ec169   9 months ago    272MB
ansible/centos7-ansible   latest        688353a31fde   5 years ago     447MB
```

```bash
❯ docker ps -a --format 'table {{.ID}}\t{{.Image}}'

CONTAINER ID   IMAGE
33eb583946a2   ansible/centos7-ansible:latest
054427b5e550   ansible/centos7-ansible:latest
d00273a74a54   ansible/centos7-ansible:latest
dc5c7c74341e   centos:latest
974edc765e21   centos:latest
```




## [실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker) 글에서 필요한 부분 발췌

> 컨테이너는 하드웨어를 소프트웨어로 재구현하는 가상화와는 달리 프로세스의 실행 환경을 격리. 컨테이너가 실행되고 있는 호스트의 입장에서 컨테이너는 단순히 프로세스에 불과하지만 사용자나 컨테이너 입장에서는 호스트와는 무관하게 동작하는 가상머신처럼 느껴짐
> 
> 출처 : [[실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker)]

> 도커는 가상머신과 같이 하드웨어를 가상화하는 것이 아니라, 리눅스 운영체제에서 지원하는 다양한 기능을 사용해 컨테이너(하나의 프로세스)를 실행하기 위한 별도의 환경(파일 시스템)을 준비하고, 리눅스 네임스페이스와 다양한 커널 기능을 조합해 프로세스를 특별하게 실행시켜줍니다.
> 
> 출처 : [[실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker)]

> 도커는 크게 도커 엔진과 클라이언트로 나뉩니다. 도커 엔진은 서버로 동작하며, 시스템 상에 서비스로 등록 됩니다. 도커 클라이언트는 사용자가 입력하는 `docker`명령어입니다. 이 명령어를 실행하면 클라이언트는 도커 서버에 명령을 전달하고, 명령은 전적으로 서버에서 처리됩니다. 도커 클라이언트로 외부의 도커 서버에 명령을 내리는 것도 가능합니다. 이러한 아키텍처를 반영해 도커 엔진과 도커 클라이언트 패키지가 나눠졌습니다.
> 
> 출처 : [[실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker)]

> **이미지는 파일들의 집합이고, 컨테이너는 이 파일들의 집합 위에서 실행된 특별한 프로세스입니다.**
> 
> 출처 : [[실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker)]

> 셸 프롬프트가 `[root@d3fef9c0f9e9 /]#`로 바뀌었습니다. **우분투 환경에서 센트OS환경에 접속했습니다.** 하지만 **접속했다**는 말에는 함정이 있습니다. 이 말은 마치 다른 서버에 SSH 프로토콜을 사용해 접근한 것과 같은 착각을 일으킵니다. 여기서는 SSH를 통해 서버에 접속한 것이 아니라, 호스트OS와 격리된 환경에서 `bash` 프로그램을 실행했다고 이해하는 것이 더 정확합니다.
> 
> 출처 : [[실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker)]

> 셸은 대화형으로 리눅스 머신에 명령을 실행하기 위한 커맨드라인 도구입니다. 프로세스이기 때문에 셀을 종료하면, 그걸로 끝입니다. 반면에 SSH는 외부에서 접속하기 위해 설치해두는 서버 프로세스입니다. 따라서 SSH 서버에 접속해서 셸을 사용하고 종료하더라도 SSH 서버는 그대로 살아서 다른 접속을 기다립니다. 겉보기에는 비슷하지만 도커로 셸을 직접 실행해서 사용하는 것과 외부 서버에 SSH로 접속하는 것의 차이를 명확하게 이해해야 도커 컨테이너와 가상머신이 헷갈리지 않을 수 있습니다.
> 
> 출처 : [[실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker)]

> 특정한 이미지로부터 생성된 컨테이너에 어떤 변경사항을 더하고(파일들을 변경하고), 이 변경된 상태를 새로운 이미지로 만들어내는 것이 가능합니다. 도커의 모든 이미지는 기본적으로 이 원리로 만들어집니다. 바로 이러한 점 때문에 도커에서는 저장소Repository, 풀Pull, 푸시Push, 커밋Commit, 차분Diff 등 VCS에서 사용하는 용어들을 차용한 것으로 보입니다.
> 
> 출처 : [[실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker)]

>하나 알아두셔야 하는 중요한 사항은, 이미지에서 파생된 (종료 상태를 포함한) 컨테이너가 하나라도 남아있다면 이미지는 삭제할 수 없습니다. 따라서 먼저 컨테이너를 종료하고, 삭제까지 해주어야합니다. 
> 
> 출처 : [[실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker)]

## reference
[[이론과 실습을 통해 이해하는 Docker 기초](https://hudi.blog/about-docker/)]<br>
[[실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker)]