---
layout: post
title: "느려도 너무 느린 Docker for mac의 bind mount 속도 개선"
date: 2020-03-23 19:30:00 +0900
author: 이현민 lhm1442@gmail.com
tags:
  - Docker
---

## 똑같이 도커 컨테이너에서 돌리는데...
AWS Codebuild 리눅스 호스트에서는 5분만에 전체 테스트가 돌아가는데 제 맥북에서는 25분이 걸립니다. 아래는 AWS 코드 빌드 general1.small(3 GB RAM, 2 vCPU ) 에서의 실행 결과입니다.
```sh
Finished in 5 minutes 17 seconds (files took 6.32 seconds to load)
1965 examples, 0 failures
```

## 원인
[도커 컨테이너가 MacOS 호스트에 있는 파일을 읽고 쓸 때는 시스템 콜 변환 과정을 거칩니다. 시스템 콜이 즉각적이라는 가정 하에 작성된 소프트웨어들은 이 사실을 모르기 때문에 수 만건 반복 요청을 보내도 합니다.](https://www.docker.com/blog/user-guided-caching-in-docker-for-mac/) 그래서 Docker-for-mac의 bind mount엔 작업에 따라 [속도 저하 문제](https://docs.docker.com/docker-for-mac/osxfs/#performance-issues-solutions-and-roadmap)가 생길 수 있습니다.

### 잠깐! bind mount와 volume
둘 다 도커 호스트 머신에 파일을 저장하는 방법입니다. [Manage data in Docker](https://docs.docker.com/storage/)에 따르면 다음과 같은 차이가 있습니다.
#### volumes
- 호스트 파일 시스템 중 도커에 의해 관리되는 `/var/lib/docker/volumes/` 에 저장됩니다.
- 도커 이외의 프로세스가 수정해서는 안 됩니다.
- 볼륨의 이름만 명시하면 된다. 따라서 호스트에서의 파일 위치가 정해져 있지 않을 때, 예를 들면 개발 환경에서 호스트에서 편집한 코드를 컨테이너에 공유할 때 많이 쓴다.

#### bind mounts
- [Bind mount](https://docs.docker.com/storage/bind-mounts/)는 도커 프로세스와 텍스트 편집기와 같은 도커 이외의 프로세스가 모두 내부 파일을 수정할 수 있습니다. 따라서 개발 환경에서 코드를 동기화하는 용도로 널리 쓰입니다.
- 마운트할 host machine의 full path를 명시해야 합니다.

## 해결
정말 답답해서 눈물이 나와 nfs 등 여러 가지를 시도해보았고 [docker-sync](https://docker-sync.readthedocs.io/)에 정착했습니다. 개인 맥북(2.7 GHz dual-core Intel Core i5, 8GB RAM, Docker에 2cpu, 4GB RAM 할당) 에선 전체 테스트 실행에 10분이 걸립니다. 개발할 때는 테스트 파일을 하나씩만 돌리니 유사 TDD를 할 만했습니다.

### Step1. Bind mount를 external로 선언
`docker-compose.override.yml`를 `.gitignore`에 추가하면 docker-sync를 사용하지 않는 개발자들과의 호환성도 유지할 수 있습니다.
```yaml
# docker-compose.override.yml
version: '3.4'
services:
  app: &app
    volumes:
      - app-code/:/var/www/app

volumes:
  app-code:
    external: true
```

### Step2. Install docker-sync
[docker-sync](https://docker-sync.readthedocs.io/en/latest/getting-started/installation.html#installation) 를 다운로드합니다.
```sh
$ gem install docker-sync
$ docker-sync start
```

### Step3. `docker-sync.yml` 설정 파일 작성
```yaml
version: "2"
options:
  verbose: true
syncs:
  app-code:
    src: './'
    sync_strategy: 'native_osx'
    sync_excludes: ['.git', '*.log', 'public/uploads']
```

### Step4. 기존 컨테이너 삭제 후 재생성
```sh
docker-compose down;docker-compose up -d
```

### 문제 해결

#### 1. "xxx file not found" 에러가 발생합니다.
기존 볼륨을 지운 뒤 다시 Step 4번을 진행해주세요.
```sh
docker volume rm app-code
```

#### 2. 도커 데몬을 껐다 켰더니 동기화가 안 됩니다.
docker-sync가 별도의 컨테이너를 띄워서 bind mount와 내부 volume을 동기화해주는 방식으로 동작합니다. Docker-for-mac을 재시작한 경우 `docker-sync start`를 하지 않으면 파일이 업데이트되지 않습니다. 업데이트 내역은 `docker-sync logs` 로 확인할 수 있습니다. 자세한 원리는 [docker-sync 공식 문서](https://docker-sync.readthedocs.io/en/latest/advanced/how-it-works.html)를 참고하세요.

## 비교
레일즈 프로젝트에서 rspec을 실행한 결과입니다. 둘 다 [Spring preloader](https://github.com/rails/spring)를 사용했습니다.
1. Default volume
```sh
$ bin/rspec spec/api/v1/task_boards/index_spec.rb
Finished in 14.64 seconds (files took 6.91 seconds to load)
```

2. docker-sync
```sh
$ bin/rspec spec/api/v1/task_boards/index_spec.rb
Finished in 6.38 seconds (files took 0.7989 seconds to load)
```

## Q & A
### `cached` 또는 `delegated` 설정은 어떨까요?
[`delegated`를 적용하면 container의 파일 변화를 host에 전달하는 데 시간차가 생깁니다. 반대로 `cached` 를 적용하면 host의 파일 변화가 container에 즉시 전달되지 않을 수 있습니다.](https://docs.docker.com/docker-for-mac/osxfs-caching/#tuning-with-consistent-cached-and-delegated-configurations%23tuning-with-consistent-cached-and-delegated-configuration) 우리는 호스트의 코드 변경 사항이 바로 컨테이너에 반영되기를 원하는 것이므로 어느 경우에도 해당되지 않습니다.
