---
layout: post
title: "AWS Codepipeline에서 파일 실행 권한이 사라지는 문제"
date: 2019-03-20 12:00:00 +0900
tags:
  - Docker
  - AWS
summary: "AWS Codebuild를 Codepipeline과 연동할 때, docker entrypoint script의 file permission이 사라지기 때문에 buildspec.yml에서 다시 chmod +x 를 해줘야 합니다. "
---

# AWS Codepipeline에서 파일 실행 권한이 사라지는 문제

## 문제 발생
Dockerfile에서 `ENTRYPOINT` 스크립트를 실행하도록 변경했다. 이후 AWS Codebuild만 단독으로 실행하면 정상 작동했는데 Codepipeline으로 Codebuild를 실행하니  `permission denied` 에러가 발생했다.
참고로 `docker run [SERVICE] [COMMAND]`  를 실행하면 항상 `ENTRYPOINT`로 지정된 스크립트가 실행된다.
```
[Container] 2019/03/20 08:48:36 Running command echo Testing...
Testing...
 [Container] 2019/03/20 08:48:36 Running command docker-compose run -e RAILS_ENV=test \
-e RAILS_MASTER_KEY_TEST=$RAILS_MASTER_KEY_TEST \
app /bin/sh -c "rspec"
 Creating network "src_default" with the default driver
Creating src_db_1 ...
·[1A·[2K
Creating src_db_1 ... ·[32mdone·[0m
·[1BError response from daemon: OCI runtime create failed: container_linux.go:348: starting container process caused "exec: \"./docker/app/docker-entry.sh\": permission denied": unknown
```

## 원인
S3로 소스코드를 압축해서 보내는 Codepipeline에서는 permission bits가 사라지는 문제가 있다고 한다. https://forums.aws.amazon.com/thread.jspa?threadID=235452

## 해결
Codebuild의  `buildspec.yml` 에서 다시 `chmod +x ./docker/app/docker-entry.sh`를 실행했다.
