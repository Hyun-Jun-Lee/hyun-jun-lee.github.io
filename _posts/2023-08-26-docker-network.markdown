---
title: "Docker Netowrk" #Article title.
date: 2023-08-26
category: [INFRA, DOCKER] #One, more categories or no at all.
tag: [infra, docker]
---

# Docker Network

Docker로 배포한 고객사에서 접근 IP 가 변경되었다는 문의사항이 있었는데

우리는 Docker Continaer를 `network_mode:hosts`로 사용하고 있었고

테스트용으로 포트 바인딩에서 실행시켜둔 컨테이너들이 있었는데 docker compose up 되어 있는 상태에서

내가 folder를 rename 했더니 docker container 접근이 안됬고 이 때 network가 hosts에서 

포트 바인딩된 network로 변경되어진듯 하다.

고객사 db는 특정 IP:PORT만 접근이 허용되어 있는데 docker container에서는 접근이 안되는 것이다.

내가 알기로 docker container에 포트를 바인딩 시키면 알아서 서브넷 마스크 처럼 localhost로 변경해주는지 알았는데 아닌가보다.

그래서 docker network에 대해 자세히 공부해보려고 한다.

## Docker Network 구조

Docker container는 독립적으로 실행되기 때문에 기본적으로 다른 컨테이너와의 통신이 불가능한데, 여러 개의 컨테이너를 하나의 docker 네트워크에 연결 시키면 통신이 가능해진다.

![image](https://github.com/Hyun-Jun-Lee/mma-cast/assets/76996686/6ef44b81-c5c4-4046-94de-6383dfa9ee73)

도커 컨테이너는 `lo`, `eth0` 인터페이스를 가지고 있고, 각 컨테이너의 `eth0` 에는 `172.17.0.0/16` 대역의 IP가 할당된다.
각 컨테이너 내부에서 `ifconfig` 명령어로 컨테이너 네트워크의 인터페이스를 확인할 수 있다.
이 대역은 내부 private IP 이므로 외부 접속이 불가능 하다.

그래서 도커에서는 호스트에  `veth(Virtual Ethernet)` 라는 가상 네트워크 인터페이스를 만들고, 각 컨테이너의 `eth0`의 포트를 호스트의 IP/포트에 바인딩한다.

`veth`는 도커 호스트에서 `ifconfig` 명령어를 입력하면 확인할 수 있다. 

즉, `eht0`에 대응되는 veth---라는 이름의 veth 인터페이스와 브릿지 네트워크에 컨테이너의 인터페이스가 바인딩 되는 형태로 통신하는 것이다.

`veth` 인터페이스는 컨테이너가 생성될 때 도커엔진이 자동으로 생성하고, 네트워크를 따로 지정해주지 않으면 `docker0` 이라고 하는 브릿지 네트워크를 default로 사용한다. 이 브릿지 네트워크는 `veth`와 호스트의 `eth0` 의 다리 역활을 한다.

## Docker Network Driver

### bridge

도커 네트워크 드라이버의 default 설정이고, 사용자가 직접 브릿지 네트워크를 생성해서 적용할 수도 있다.

- bridge 생성
    `docker network create --driver bridge mybridge`

- 컨테이너 mybridge 연결하여 생성
    `docker run -it --name mynetwork_container --net mybridge ubuntu:18.04`

### host

컨테이너에서 호스트 네트워크의 환경을 그대로 사용하는 드라이버로, 컨테이너의 네트워크가 도커 호스트에서 따로 고립될 필욕 없을 때 사용한다.

### overlay

여러 도커 데몬에서 실행되는 컨테이너들을 한 네트워크로 묶어주는 드라이버로, Docker Swarm 서비스에서 사용된다.

### none

단어 그대로 컨테이너의 모든 네트워크를 비활성화 하는 것으로 외부와 통신하지 않는다.


## 불필요한 네트워크 제거

`docker network prune` 

위 명령어로 아무 컨테이너도 연결되어 있지 않은 네트워크를 한번에 모두 제거할 수 잇다.


- 참고 블로그
1. https://seosh817.tistory.com/373
2. https://www.daleseo.com/docker-networks/
3. https://captcha.tistory.com/70