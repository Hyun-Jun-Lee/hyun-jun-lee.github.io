---
title: "CI/CD 도입기 with Github Action"
date: 2024-05-08
category: [INFRA,DEVOPS]
tag: [cicd, github_action, workflows]
---

- 지속적 통합 (Continuous Integration) : 개발자들이 작업하는 모든 변경 사항을 통합하고 테스트
- 지속적 전달 (Continuous Delivery) : 개발된 소프트웨어를 프로덕션 환경에 안정적으로 배포

사이드 프로젝트 서비스가 출시되었고 본격적으로 홍보를 시작하기 전에, 분명 서비스가 시작되면 예상치 못한 버그들이 많이 발생할 것 같아서 반복된 작업을 줄이고자 CICD를 구축하려 했다.
가장 간단한 Github Actions을 이용했고, 내 예상과 다르게 너무 오래 걸렷다;; 
그리고 실제 레포지토리에 Push 해보지 않으면 테스트해볼 수 없어서 commit이 엉망이 되었었는데 이것 때문에 git rebase에 대해 더 자세하게 알게되었다.

```yml
name: Dev CI-CD Pipeline

on:
  pull_request:
    branches:
      - release/dev
    types:
      - closed
```
`on`은 해당 workflows가 실행될 Event를 지정하는 것인데  [GitHub Action 문서](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows) 여기 참조해보면 단순하게 push, merge, pull_request 말고도 많은 event가 있다.

내가 만든 Jobs에는 test, push, deploy가 있는데, 나는 release/dev 브랜치에 PR이 생성되면 test 작업이 실행되고 통과된 후 PR이 승인되어 merge된 후에 deploy 작업이 실행되게 만들고 싶었다. 
처음에는 push와 pull_request 이벤트를 트리거로 설정했는데, workflows가 2번 실행되었다. 이유를 찾아보니 PR이 생성될 때 한 번 작동되고, 병합되는 과정에서는 push 이벤트로 감지되었다!
그래서 내가 원하는 의도대로 작동시키기 위해 `types: closed`를 추가하여 수정했다.


```yml
jobs:
  test:
    runs-on: ubuntu-latest
    if: github.event.pull_request.merged == false

    services:
      maindb:
        image: postgres:13
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: ${{ secrets.DB_ROOT_PASSWORD }}
          POSTGRES_DB: ${{ secrets.TEST_DB }}
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
          
      redis:
        image: redis:6.2.6
        ports: 
          - 6379:6379

    steps:
      - name: checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Create .env file
        run: |
          echo "BISKIT_USER=${{ secrets.BISKIT_USER }}" >> .env
          ...환경 변수 설정...
          echo "DEBUG=True" >> .env

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements/requirements.txt

      - name: set python path
        run: |
          echo "PYTHONPATH=${{ github.workspace }}/apps" >> $GITHUB_ENV

      - name: Run test
        run: |
          pytest --rootdir=apps/
```

내가 설정한 첫번째 jobs인 `test` 는 말 그대로 test code를 실행시키는 단계인데 `actions/checkout@v4` 는 github action에서 지원하는 기능으로 현재 git의 최신상태 코드로 pull 해준다.
내 프로젝트에서는 `.env`로 환경변수를 관리하고 있는데 Github actions에서는 repository scerts를 제공하고 이것을 이용해서 .env 파일을 만들어준다. `os.environ` 으로 환경변수를 읽어와 .env를 만들어주는 스크립트를 짯었는데 안되길래 찾아보니 github scerts는 보안상의 이유로 그렇게는 읽어올 수 없다고 하더라!

pytest를 실행시킬 때 경로 때문에 계속 에러가 났는데 `--rootdir` 옵션이 있었다, 역시 내가 겪어본 에러는 다들 한번 씩은 다 겪는 듯하다.

```yml
  push:
    runs-on: ubuntu-latest
    if: github.event.pull_request.merged == true
    steps:
      - name: Set DATE variable
        run: echo "DATE=$(date +'%Y%m%d-%H%M')" >> $GITHUB_ENV

      - name: checkout code
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/biskit:${{ env.DATE }}
```

이 단계는 test 성공 후 image 를 build & push하는 것이 목표이고, 여기에는 QEMU, Buildx가 포함되어 이미지가 여러 플랫폼에서 동작할 수 있도록한다.

```yml
  deploy:
    runs-on: ubuntu-latest
    if: github.event.pull_request.merged == true

    steps:
      - name: Remote ssh connect
        uses: appleboy/ssh-action@v0.1.9
        with:
          host: ${{ secrets.DEV_REMOTE_IP }}
          username: ${{ secrets.DEV_REMOTE_USER }}
          password: ${{ secrets.DEV_REMOTE_PASSWORD }}
          port: ${{ secrets.DEV_REMOTE_PORT }}
          script: |
            cd BISKIT-Backend
            git checkout ${{ secrets.DEV_BRANCH }}
            git pull origin ${{ secrets.DEV_BRANCH }}
            echo ${{ secrets.DOCKERHUB_TOKEN }} | docker login -u ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin
            docker pull ${{ secrets.DOCKERHUB_USERNAME }}/biskit
            docker compose -f docker-compose.local.yml up -d
```

마지막 단계는 PR 승인 후 closed 될 때 실행되는 deploy 단계로 서버에 접속해 docker image를 받고 container를 재실행시킨다.
push 단계에서 처럼 `docker/login-action@v3`을 했는데 불구하고, 계속 dockerhub가 안되길래 꽤 고생했다.
생각해보니 deploy 가 실행되고 있는 container에서가 아니라 배포 서버에 접속 한 후에 dockerhub에 로그인 해야하더라!