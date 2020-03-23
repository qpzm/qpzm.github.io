---
layout: post
title: "도커(Docker), AWS Codebuild, CodePipeline을 활용한 레일즈(Ruby on Rails) 개발과 배포"
date: 2019-03-13 12:00:00 +0900
author: 이현민 lhm1442@gmail.com
tags:
  - Docker
  - AWS
permalink: "intro_to_docker"
---

도커에 대한 간략히 이해하고 레일즈 개발에 적용해 봅니다. 쓰다 보면 궁금해지므로 원리보다는 사용 방법에 집중합니다. 🙋 라벨이 붙은 내용은 넘어가도 좋습니다.

## 1. 굳이 왜?
### 쉽고 빠른 환경 설정
새 프로젝트를 시작하면 세팅하다 하루가 다 가곤 한다. 이제 Docker 를 이용하면 두 줄로 끝난다.
```bash
docker-compose build
docker-compose up
```

### 불변 배포 (Immutable deployment)

과거의 나는 서버에 도대체 무슨 짓을 벌였는가?  `rvm`  `nginx` 등 온갖 패키지, 설정 파일의 향연에 좌절하곤 했다.

쉽게 똑같은 서버를 구성할 수 있다면 서버 관리, 롤백, Scaling에 유리하다.



## 2. 도커 맛보기

도커는 어플리케이션을 개발, 배포, 실행하기 위한 오픈소스 플랫폼입니다.

![Containers sharing same image](https://docs.docker.com/storage/storagedriver/images/sharing-layers.jpg)

### 이미지

뜯기 전 3분 카레. Mysql docker image라면 mysql을 실행시킬 수 있게 코드, 의존성 패키지, 환경변수 등 모든 것이 준비되어 있다. Dockerfile을 빌드해서 만들며 여러 개의 레이어로 이뤄진다.



### 컨테이너

전자레인지에 돌린 3분 카레. 이미지를 실행시키면 컨테이너가 된다. Mysql 이미지의 컨테이너는 실제로 Mysql을 사용가능하다. 도커 이미지에 새 Read/Write layer를 올려서 만들어지고 종료 시 해당 레이어는 지워진다.



### 기본 명령어

``` bash
docker run docker/whalesay cowsay "난다 고래?"
docker run -it docker/whalesay /bin/sh
docker ps # 실행중인 컨테이너 목록
docker ps -a # 모든 컨테이너 목록
docker stop <container-id>
docker rm <container-id> # 컨테이너 삭제
docker image ls # 이미지 목록
docker rmi <image-id> # 이미지 삭제
docker system prune # 안 쓰는 컨테이너, 이미지 모두 삭제
```



### 레일즈 도커 파일 예시

```dockerfile
FROM ruby:2.5.3 # ubuntu with ruby 2.5.3

RUN apt-get update -qq && \
    apt-get install -y build-essential libpq-dev nodejs imagemagick && \
    rm -rf /var/lib/apt/lists/*

ENV RAILS_ROOT /var/www/app
RUN mkdir -p $RAILS_ROOT

WORKDIR $RAILS_ROOT

COPY Gemfile* ./
RUN gem install bundler -v '1.16.4' \
    && bundler config --global frozen 1
RUN bundle install

VOLUME $RAILS_ROOT

EXPOSE 3000

CMD bundle exec rails db:migrate && bundle exec puma -C config/puma.rb
```

#### VOLUME

볼륨은 외부 저장 장치처럼 호스트의 저장공간을 쓰기 위해. DB 저장에 흔히 사용하는데 이유는 다음과 같다.

* 컨테이너를 지워도 볼륨에 저장한 데이터는 남아있다.
* 볼륨에 기록하면 이미지 크기가 늘어나지 않는다.

단 dockerfile에서 VOLUME 명령어는 mount point, 즉 빈 폴더만 만든다. `docker run -v <host_path>:<container_path> ` 를 통해 실제로 마운트할 수 있다.



#### EXPOSE

docker run 할 때 해당 포트를 사용하라는 표시.
실효는 없고 사용자에게 남기는 주석이다. (AWS 배포하는 플랫폼에 따라 이 표시를 활용하는 경우도 있다)





**Tip**

> In Docker 1.10 and higher, only `RUN`, `COPY`, and `ADD` instructions create layers. Other instructions create temporary intermediate images, and no longer directly increase the size of the build.

https://docs.docker.com/v17.09/engine/userguide/eng-image/dockerfile_best-practices/#minimize-the-number-of-layers

`RUN`, `COPY`, `ADD` 세 명령어는 이미지 레이어를 만든다. 빌드할 때 아래 레이어가 변경된 경우, 그 위 레이어들을 전부 새로 빌드하므로 명령어 배치에 유의하자.


 🙋🏼‍♀️Dockerfile부터 직접 이미지를 빌드해 보자: [공식 튜토리얼](https://docs.docker.com/get-started/part2/)



### 정리

> 컨테이너는 추상화된 어플리케이션이다.

#### 추상화 (Abstraction)

중요한 것만 남기는 것. 3분 카레도 카레다.

- 숫자: 1000원을 받으면 100원 짜리가 몇 개인지 신경쓰지 않는다.
- OS의 하드웨어 추상화: 안드로이드 앱은 다양한 기기에서 실행된다.



#### VM vs Container

![img](https://www.docker.com/sites/default/files/d8/2018-11/docker-containerized-and-vm-transparent-bg.png)

[도커 공식문서](https://www.docker.com/resources/what-container)에 따르면 컨테이너는 코드와 의존 패키지를 포함한 어플리케이션, 즉 OS 커널 위에서 실행되는 프로그램을 추상화한다. 반면, 가상머신은 컴퓨터 하드웨어를 추상화한다. 도커 컨테이너들은 OS 커널을 공유하기 때문에 더 적은 용량을 차지한다.

**잠깐!**

도커는 namespace(독립된 프로세스 공간), cgroup(리소스 제한) 등의 리눅스의 기능을 활용해서 만들어졌다. 따라서 Docker for mac은 리눅스를 돌리기 위한 가상머신을 추가로 설치한다.



## 3. docker-compose

`docker-compose.yml` 파일을 정의하고 해당 파일이 있는 폴더에서 `docker-compose 명령어 ` 를 입력하면 여러 컨테이너를 한 번에 관리하고 옵션을 생략할 수 있어 편리하다.

### ⭐️ 기본 명령어

#### 이미지 빌드 후 컨테이너 실행

- 한 번도 빌드한 적이 없어 아직 이미지가 없을 때
- Gemfile을 수정했을 때

```bash
docker-compose build
docker-compose up -d
docker-compose logs -f app
```

#### 터미널 명령어 입력

```bash
$ docker-compose exec app bash
root@35b0e137ef35:/var/www/app# e.g. bundle exec rails db:migrate
```

####  컨테이너 종료

```
docker-compose down
```



 🙋 `docker-compose`를 이용해 레일즈, postgresql을 실행해보자: https://docs.docker.com/compose/rails/



###  🙋 `docker-compose.yml` 구성

예시 레일즈 프로젝트의  `/docker-compose.yml` 을 살펴보자.

```yaml
version: '3.2' # use compose format v3
services:
  app:
    build:
      context: .
      dockerfile: ./docker/app/Dockerfile
      cache_from:
        - ${AWS_ECR_REPO}/rails:1.0.3
    image: ${AWS_ECR_REPO}/rails:1.0.3
    volumes:
      - ./:/var/www/app
      - ./config/database-docker.yml:/var/www/app/config/database.yml
    expose:
      - 3000
    environment:
      - RAILS_ENV=development
    depends_on:
      - db # 시작 순서를 조정, 단 해당 container의 started 상태만 보장, 내부적 준비 여부는 모름.

  db:
    image: mysql:5.7
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    volumes:
      - ./db/development:/var/lib/mysql
    expose:
      - 3306
    environment:
      MYSQL_ROOT_PASSWORD: xxx
      MYSQL_DATABASE: xxx

  web:
    build:
      context: ./docker/web/
      cache_from:
        - ${AWS_ECR_REPO}/nginx:1.0.0
    image: ${AWS_ECR_REPO}/nginx:1.0.0
    ports:
      - 80:80
      - 4000:3000 # dev
    depends_on:
      - app
```



## 4. Staging 서버로 배포

### ⭐️ 배포 과정

1. `dev` 브랜치에서 `docker-compose.yml`과 ` Dockerrun.aws.json` 의 레일즈 이미지 태그를 새 릴리즈 버전과 동일하게 업데이트 후 커밋

2. `staging ` 브랜치로 pull request & squash and merge

3. AWS Codebuild 와 Codepipeline에 의해 ElasticBeanstalk에 자동 배포

`staging` 브랜치에 대한 푸시 권한을 제한하면 무분별한 배포를 막을 수 있다.



### Codebuild

도커 이미지를 빌드하고 이미지 저장소에 업로드

로컬에 AWS CLI가 설치되어 있다면 아래의 순서로 이미지 저장소에 푸시할 수 있다.

1. `aws configure` 명령어를 통해 해당 저장소에 업로드 권한이 있는 유저로 로그인
2. 아래 build script의 `-` 다음에 나오는 명령어들을 순서대로 입력

`/buildspec.yml`

```yaml
version: 0.2 # build spec version

env:
  variables:
    AWS_DEFAULT_REGION: "ap-northeast-2"

phases:
  pre_build:
    commands:
      - echo Logging into Amazon ECR...
      - $(aws ecr get-login --region $AWS_DEFAULT_REGION --no-include-email)
      - echo Pulling Docker Images for cache...
      - docker-compose pull --ignore-pull-failures -q
  build:
    commands:
      - docker-compose build
  post_build:
    commands:
      - docker-compose push

artifacts:
  files:
    - './**/*'
```



### Codepipeline

 Github 소스코드를 Codebuild에 전달하고 ElasticBeanstalk에 배포

![image-20190313041603062](https://user-images.githubusercontent.com/18223805/54255510-bc206d80-459b-11e9-983f-78aa42b95697.png)



###  🙋🏼‍♀️ AWS ElasticBeanstalk 세부 내용

AWS ElasticBeanstalk Multicontainer docker 환경을 사용한다. 이는 내부적으로 컨테이너 관리 서비스인 Elastic Container Service를 이용하며 `docker-compose.yml` 과 유사한 아래의 설정파일이 필요하다.

`/Dockerrun.aws.json`

```json
{
  "AWSEBDockerrunVersion": 2,
  "containerDefinitions": [
    {
      "environment": [
        {
          "name": "RAILS_ENV",
          "value": "production"
        }
      ],
      "essential": true,
      "memory": 512,
      "image": "386640523397.dkr.ecr.ap-northeast-2.amazonaws.com/rails:1.0.3",
      "mountPoints": [
        {
          "containerPath": "/var/www/app",
          "sourceVolume": "Performance_Plus_Rails"
        },
        {
          "containerPath": "/var/www/app/config/database.yml",
          "sourceVolume": "Performance_Plus_Rails_DB_Config"
        }
      ],
      "name": "app"
    },
    {
      "essential": true,
      "memory": 512,
      "image": "386640523397.dkr.ecr.ap-northeast-2.amazonaws.com/nginx:1.0.0",
      "mountPoints": [
        {
          "containerPath": "/var/www/app",
          "sourceVolume": "Performance_Plus_Rails"
        }
      ],
      "links": [
        "app"
      ],
      "name": "web",
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80
        }
      ]
    }
  ],
  "volumes": [
    {
      "name": "Performance_Plus_Rails",
      "host": {
        "sourcePath": "/var/app/current/" # elasticbeanstalk가 기본으로 파일을 업로드하는 위치
      }
    },
    {
      "name": "Performance_Plus_Rails_DB_Config",
      "host": {
        "sourcePath": "/var/app/current/config/database-docker.yml"
      }
    }
  ]
}
```

#### links
docker-compose는 자동으로 컨테이너 이름을 이용해 다른 컨테이너에 접속할 수 있게 도와준다.

`/docker/web/nginx.conf` 의 일부

```nginx
upstream rails_app {
  server app:3000; # app is defined in Dockerrun.aws.json "link": ["app"]
}
```

추후 [deprecated 예정](https://docs.docker.com/network/links/)이니 `network` 옵션으로 바꿀 계획. 자세한 옵션 설명은 [Task Definition Parameters - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html) 을 참조하자.



#### Q. db 이미지는 어디로 갔나요?
DB는 rds로 운영하므로 필요없다!



## 결론

⭐️ 만 익히고 쓰면서 찾아보자.







## Reference

**2. 도커 맛보기**

도커의 정의, 원리 https://docs.docker.com/engine/docker-overview/

Dockerfile부터 도커 허브에 푸시까지 [Get Started, Part 2: Containers | Docker Documentation](https://docs.docker.com/get-started/part2/)

Dockerfile 작성 시 유의사항 https://docs.docker.com/v17.09/engine/userguide/eng-image/dockerfile_best-practices/

Layer 관점에서 본 이미지와 컨테이너 https://docs.docker.com/storage/storagedriver/

컨테이너는 어플리케이션의 추상화

- [What is a Container? | Docker](https://www.docker.com/resources/what-container)
- [Containers are not VMs - Docker Blog](https://blog.docker.com/2016/03/containers-are-not-vms/)

namespace와 cgroup https://tech.ssut.me/what-even-is-a-container/


**3. docker-compose**

- [명령어 docs](https://docs.docker.com/compose/compose-file/)
- [docker-compose를 이용한 레일즈, postgresql 실행](https://docs.docker.com/compose/rails/)



**4, Staging 서버로 배포**

[AWS ElasticBeanstalk Multicontainer docker 환경 튜토리얼](https://docs.aws.amazon.com/ko_kr/elasticbeanstalk/latest/dg/create_deploy_docker_ecs.html)

`Dockerrun.aws.json` 의 설정 옵션 [Task Definition Parameters - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html)

도커 전반에 대한 한글 자료 http://pyrasis.com/docker.html
