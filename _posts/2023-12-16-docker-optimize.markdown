---
title: "DockerFile 최적화" #Article title.
date: 2023-12-14
category: [INFRA, DOCKER] #One, more categories or no at all.
tag: [docker, dockerfile, optimize]
---

# DockerFile 최적화

docker 이미지를 빌드 할 때, 비슷한 dockerfile인데도 불구하고 빌드 속도 차이가 있는 것을 알게되었고 최적화 관련 자료를 찾아보았다.


## docker cache

```
FROM ubuntu:latest

RUN apt-get update && apt-get install -y build-essentials
COPY main.c Makefile /src/
WORKDIR /src/
RUN make build
```

docker는 이미지를 빌드 할 때 각 명령어 마다 layer를 만든다. 
위 예시로 보면 처음에 ubuntu:latest layer가 생성 되고 그 위에 RUN ~~~, 그 다음 COPY~~ 와 같이  stack같은 느낌으로 layer가 쌓이는 것이다.

매번 빌드 시 마다 이렇게 일일이 layer를 생성하는게 아니라 각 명령어를 캐싱 해두고 이전과 똑같다면 캐싱을 이용한다.

명령어가 변경되면 layer도 변경되어야 하기 때문에 rebuild되는데, 위 예시에서 COPY 단의 명령어가 변경되었다면 그 이후 WORKDOR, RUN layer까지 모두 재실행 된다. 

그러니 명령어의 순서도 빌드 효율에 영향을 미친다. 보통 requirements 같은 경우 자주 변경되니 맨 마지막 layer에 두는 것이 build 시간을 줄이는데 큰 도움을 줄것이다.

그리고 명령어가 변경되어야 캐싱하지 않고 새로 빌드하기 때문에, 몇달 전에 build 해뒀던 dockerfile을 변경 없이 그대로 rebuild해도 모든 라이브러리들이 최신버전으로 설치되지 않는다.

docker 공식 문서에서 docker build 최적화 방법에 대해 몇가지 설명해주고 있다.

### Keep Layer Small

당연한 말이지만 원래 이런 사소한 점들을 놓치는 경우가 많으니 한번 정리해두자

`COPY . /apps` 

위 명령어는 모든 파일들을 image로 빌드해야하기 때문에 당연히 이미지 크기가 커질 수 밖에 없다. 그러니 필요한 dir만 찾아서 copy 하는 것이 효율적이다.

`COPY ./apps/project /project`

아니면 `.dockerignore`를 활용해도 될듯!

### Use the dedicated RUN cache

`RUN` 명령어는 더 세분화된 캐시를 지원한다. 예를들어 패키지를 설치 할 때, 패키지 몇개 변경한다고 모든 패키지를 설치할 필요는 없다. 이럴 케이스에 `RUN --mount type=cache`를 사용하면 된다.

```
RUN \
    --mount type=cache, target=/var/cache/apt \
    apt-get update && apt-get install -y git
```

위 예시로보면, target으로 설정된 dir은 rebuild시에도 캐싱을 사용한다.

### base image tag 종류

python 베이스 이미지를 찾다보면 다양한 tag가 존재한다. 일반적으로 `slim` 태그를 가장 많이 사용하는데, OS 차이가 있어서 몇가지 알아 둘 필요가 있을 듯 하다.

- `slim` : 실행하기 위한 최소한의 것들만 포함되어있는 image
- `buster, bullseye` : Debian 계열의 os가 사용되는 image
- `alpine` : alpine linux os를 사용하는 image

이 외에 `ubuntu`와 같은 os도 있고 `amd` 같이 아키텍처에 따른 이미지도 있다.

### Use multi-stage builds

이 것은 여러개의 base image를 사용해서 build를 수행하는건데, base image는 FROM 명령어로 구분된다.

```

# stage 1
FROM alpine as git
RUN apk add git

# stage 2
FROM git as fetch
WORKDIR /repo
RUN git clone https://github.com/your/repository.git .

# stage 3
FROM nginx as site
COPY --from=fetch /repo/docs/ /usr/share/nginx/html
```

dockerfile은 FROM 명령어를 기준으로 작업 공간이 분리되고 이 작업 공간을 stage라고 부른다.
위 예시로보면 3개의 stage가 존재하는 것.

각 stage가 간단하게 구성되어 있으면 docker 는 병렬로 build 할 수 있다.
`--from=fetch`와 같이 이전 build 단계의 layer를 전달받을 수 있다. 이를 활용해서 마지막 stage에는 실행을 위한 layer와 설치를 위한 layer를 분리할 수 있다.

1. dependancy module 설치하고 app build
2. 1단계에서 build한 결과물을 복사해와서 test
3. 2단계 테스트에서 끝난 결과물을 복사해와서 실행

### 실제 반영

- BEFORE
```dockerfile
FROM amd64/python:3.9-slim-buster

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1
ENV DEBIAN_FRONTEND=noninteractive
ENV ORACLE_HOME=/apps/instantclient_21_10
ENV LD_LIBRARY_PATH=/apps/instantclient_21_10


COPY ./lobster  apps/lobster
COPY ./servers  apps/servers
COPY ./testing  apps/testing
COPY ./requirements  apps/requirements
WORKDIR /apps
EXPOSE 9000

RUN python -m venv /py && \
    /py/bin/pip install --upgrade pip && \
    apt-get update && apt-get install -y pkg-config && \
    apt-get clean -y && \
    apt-get update -y && \
    apt-get install -y python3-pymysql default-libmysqlclient-dev build-essential manpages-dev xvfb libaio1 unzip python3-dev libffi-dev wget openssl xorg freetds-dev && \
    mkdir -p /opt/oracle

WORKDIR /opt/oracle
RUN     apt-get update && apt-get install -y libaio1 wget unzip \
    && wget https://download.oracle.com/otn_software/linux/instantclient/218000/instantclient-basic-linux.x64-21.8.0.0.0dbru.zip \
    && unzip instantclient-basic-linux.x64-21.8.0.0.0dbru.zip\
    && rm -f instantclient-basic-linux.x64-21.8.0.0.0dbru.zip \
    && cd /opt/oracle/instantclient* \
    && rm -f *jdbc* *occi* *mysql* *README *jar uidrvci genezi adrci \
    && echo /opt/oracle/instantclient* > /etc/ld.so.conf.d/oracle-instantclient.conf \
    && ldconfig

WORKDIR /apps

RUN apt-get install -y gdb-multiarch gcc-aarch64-linux-gnu g++-aarch64-linux-gnu

ENV PATH="/py/bin:$PATH"

USER root


RUN /py/bin/pip install -r /apps/requirements/requirements.txt
```

- AFTER
```dockerfile
# ---- Base Stage ----
FROM amd64/python:3.9-slim-buster as base

ENV PYTHONDONTWRITEBYTECODE 1 \
    PYTHONUNBUFFERED 1 \
    DEBIAN_FRONTEND=noninteractive \
    ORACLE_HOME=/apps/instantclient_21_10 \
    LD_LIBRARY_PATH=/apps/instantclient_21_10 \
    PATH="/py/bin:$PATH"

WORKDIR /opt/oracle
RUN apt-get update && apt-get install -y --no-install-recommends libaio1 wget unzip  &&\
    wget https://download.oracle.com/otn_software/linux/instantclient/218000/instantclient-basic-linux.x64-21.8.0.0.0dbru.zip  &&\
    unzip instantclient-basic-linux.x64-21.8.0.0.0dbru.zip &&\
    rm -f instantclient-basic-linux.x64-21.8.0.0.0dbru.zip  &&\
    cd /opt/oracle/instantclient*  &&\
    rm -f *jdbc* *occi* *mysql* *README *jar uidrvci genezi adrci  &&\
    echo /opt/oracle/instantclient* > /etc/ld.so.conf.d/oracle-instantclient.conf  &&\
    ldconfig

# ---- Build Stage ----
FROM base as builder

RUN python -m venv /py && \
    /py/bin/pip install —upgrade pip && \
    apt-get update && apt-get install -y —no-install-recommends pkg-config python3-pymysql default-libmysqlclient-dev build-essential manpages-dev xvfb libaio1 unzip python3-dev libffi-dev wget openssl xorg freetds-dev && \


COPY ./requirements  apps/requirements
RUN /py/bin/pip install -r /apps/requirements/requirements.txt

COPY ./lobster  apps/lobster
COPY ./servers  apps/servers
COPY ./testing  apps/testing

# —— Production Stage ——
FROM base as production

WORKDIR /apps

# 빌드 스테이지에서 필요한 파일만 복사
COPY ——from=builder /py /py
COPY ——from=builder /apps/lobster /apps/lobster
COPY ——from=builder /apps/servers /apps/servers
COPY ——from=builder /apps/testing /apps/testing

# RUN apt-get install -y gdb-multiarch gcc-aarch64-linux-gnu g++-aarch64-linux-gnu

EXPOSE 9000
USER root
```

- builder 스테이지에서 필요한 모든 라이브러리, 의존성을 설치하고, production 에서 실제 실행에 필요한 파일만 복사하는 방식으로 이미지 크기를 줄임
- 애플리케이션 코드와 필요한 파일들을 빌드 스테이지에서 프로덕션 이미지로 효율적으로 복사

원리 dockerfile로 빌드한 이미지의 용량은 1.7GB였는데 절반 이상으로 줄임.


### Refernce
- https://docs.docker.com/build/cache/