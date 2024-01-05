---
title: "Docker 용량 줄이기"
date: 2024-01-06
category: [INFRA, DOCKER]
tag: [docker, dockerfile, capacity]
---

## Docker 용량 확인 방법

`docker info | grep "Docker Root Dir"`

위 명령어로 docker가 사용 중인 root 디렉토리를 알 수 있고 해당 디렉토리에서 `du -hs *` 명령어로 어떤 디렉토리의 용량이 큰지 확인할 수 있다.

나의 경우 overlay2 폴더의 용량이 가장 컸고, 이게 무엇인지에 대해서는 따로 포스팅 할 생각이다.

## 용량을 줄이는 방법

컨테이너 내부에서 사용되는 파일들의 저장소인 docker volume은 컨테이너에서 삭제되는 파일들이 즉시 제거되는 것이 아니라 `.Trash` 디렉토리에 쌓인다. 이 디렉토리를 삭제해서 용량을 줄일 수 있다.

- `prune` : docker에서 현재 사용되지 않는 오브텍트들을 삭제하는 명령어 

```
docker container prune : 중지된 모든 컨테이너 삭제
docker image prune : 사용하지 않는 이미지 삭제
docker volume prune : 컨테이너와 연결되지 않은 모든 볼륨 삭제
docker network prune : 컨테이너와 연결되지 않은 모든 네트워크 삭제
docker system prune -a : 위에 명령어를 통합해서 한번에 실행. 사용하지 않는 모든 오브젝트를 삭제
```

삭제 할 대상에 조건을 걸수도 있다.

`until {timestamp}` : timestamp 전에 생성된 컨테이너들 삭제
`label {label=name} or {label!=name}` : label이 있는 컨테이너들을 삭제하거나 제외하고 삭제

```
docker container prune --force --filter "until=5m" : 현재시간으로부터 생성된지 5분이 지난 컨테이너 삭제
docker container prune --force --filter "until=2017-01-04T13:10:00" : 2017-01-04T13:10:00 이전에 생성된 컨테이너 삭제
docker container prune --force --filter "label=devops" " label key가 devops인 컨테이너 삭제
docker container prune --force --filter "label!=devops" " label key가 devops가 아닌 컨테이너 삭제
docker container prune --force --filter "label=devops=qa" " label key가 devops이고 값이 qa인 컨테이너 삭제
docker container prune --force --filter "label!=devops=qa" " label key가 devops이고 값이 qa가 아닌 컨테이너 삭제
```

linux cron 에 등록해서 주기적으로 불필요한 것들을 삭제해주는 것도 좋을듯하다.