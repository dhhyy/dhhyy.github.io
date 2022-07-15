---
layout: post
# title: [TIL_20220715]
excerpt: "dockerfile 작성 연습해보기"

categories:
  - Blog
  
tags:
  - [docker]

toc: true
toc_sticky: true
 
date: 2022-07-15
last_modified_at: 2022-07-15
---

<!--
2022-06-28-DOCKER 설치
-->

_docker에 대한 전반적인 내용 정리와 함께 공부한 내용을 기록합니다._
_[[실전-도커-이미지로-서버-애플리케이션-배포하기](https://www.44bits.io/ko/post/easy-deploy-with-docker)] 거의 이 블로그를 보고 참고하였습니다. 인용한 경우에도 출처 남겨 놓았습니다._
<br>

## docker container 재시작

```bash
docker restart --help
```

컨테이너를 시작하는 방법을 몰라서 help 옵션 사용

```bash
❯ docker restart 33eb583946a2                          
33eb583946a2

❯ docker ps                                                                               
CONTAINER ID   IMAGE                            COMMAND   CREATED        STATUS          PORTS     NAMES
33eb583946a2   ansible/centos7-ansible:latest   "bash"    47 hours ago   Up 12 seconds             nervous_shannon
 ❯ docker attach 33eb583946a2
```

먼저 다시 실행 시키고 싶은 컨테이너를 찾은 후, `restart`, `attach`를 사용하면 된다.

## docker image를 추가하는 방법
1) pull을 사용해 만들어진 이미지를 가져오는 방법
2) 컨테이너 변경 사항을 반영하면서 이미지를 다시 만드는 방법
3) Dockerfile을 작성하는 방법

## dockerizing
어플리케이션 실행을 위해 도커 이미지를 만드는 작업

## dockerfile - RUN

```bash
# 어떤 이미지로부터 새로운 이미지를 생성할 것인가?
FROM ubuntu:14.04 

RUN apt-get update &&\ 
	apt-get -qq -y install git curl build-essential apache2 php5 libapache2-mod-php5 rcs 

# WORKDIR 지시자는 이후에 실행되는 모든 작업의 실행 디렉토리를 변경
WORKDIR /tmp


RUN \ 
	curl -L -O https://github.com/wkpark/moniwiki/archive/v1.2.5p1.tar.gz &&\ tar xf /tmp/v1.2.5p1.tar.gz &&\
	# github 저장소에서 파일 다운로드, 압축풀기
	mv moniwiki-1.2.5p1 /var/www/html/moniwiki &&\ 
	chown -R www-data:www-data /var/www/html/moniwiki &&\ 
	chmod 777 /var/www/html/moniwiki/data/ /var/www/html/moniwiki/ &&\ 
	chmod +x /var/www/html/moniwiki/secure.sh &&\ 
	/var/www/html/moniwiki/secure.sh 
	
RUN a2enmod rewrite 

# apache 환경변수 설정
ENV APACHE_RUN_USER www-data 
ENV APACHE_RUN_GROUP www-data 
ENV APACHE_LOG_DIR /var/log/apache2 

EXPOSE 80 

CMD bash -c "source /etc/apache2/envvars && /usr/sbin/apache2 -D FOREGROUND"
```

Dorkerfile의 한 줄 한 줄은 모두 레이어라는 개념에 의해 하나씩 쌓이게 된다. 그래서 RUN 명령어를 줄인다면 레이어도 줄어 들고, 레이어 관리를 위한 캐시도 효율적으로 관리할 수 있게 된다.

```bash
RUN apt-get update &&\ 
	apt-get -qq -y install git curl build-essential apache2 php5 libapache2-mod-php5 rcs 
```

&& : 여러 개의 명령을 한 번에 실행 할 수 있도록 하는 명령어. "\\\" 띄어쓰기로 구분 후에 실행한다. 

## docker file 실행하기
```bash
docker build -t nacyot/moniwiki:latest /Users/yudongheon/Desktop/development/git-from-dockerfile/docker-moniwiki
```

## docker 컨테이너 실행
```bash
docker run -d -p 9999:80 nacyot/moniwiki:latest
```

-d 플래그와 -i 플래그는 반대 역할을 한다. -d는 컨테이너를 데몬으로, 백그라운드 실행을 하는 플래그 옵션이다.
-p는 포트포워딩을 지정하는 옵션이며, ":"을 경계로 앞에는 외부 포트, 뒤에는 컨테이너 내부 포트를 지정한다.
컨테잉너 안에서 아파치가 80포트로 실행이 되며, 9999포트로 들어오는 연결을 컨테이너에서 실행된 서버의 80포트로 보낸다.

## 정리
앞서 설명된 것처럼 이 글은 [도커(Docker) 입문편 : 컨테이너 기초부터 서버 배포까지](https://www.44bits.io/ko/post/easy-deploy-with-docker) 글을 참고하여 작성되었다. 배포과정까지는 따라해 보지 않았으나, EC2 상에서 docker를 배포해 본 경험이 있기 때문에 쉽게 할 수 있을 것 같다. 하지만 그 과정도 오래 전 일이어서.. 이미지 만들고 컨테이너 다루는 것까지는 이렇게 복습을 했고, 나머지는 따로 공부를 해야할 것 같다.
