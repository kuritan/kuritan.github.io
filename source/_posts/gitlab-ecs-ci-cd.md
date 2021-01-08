---
title: Gitlab→AWS ECSクラスタのCI/CDパイプライン
date: 2020-02-03 18:25:19
categories: infra
tags:
- JP
- aws
- docker
- CI/CD
---

気がついたら、2020年の1月ってブログ更新ナーシング！これはヤバい......  
何とかしないと、もうこのブロクの更新が停止しまいそうー  
とりあえず、書こう。
<!--more-->

# 経緯
とある新規プロダクトは基盤を普通のEC2からECSに変えたいとの要望があって、  
一週間漬け込んで、環境構築とdockerイメージのCI/CDパイプライン作成に投げました。  

全体環境はterraformで構築し、今回はその一部であるCI/CDパイプラインについて、お話させていただきます。  

# 全体設計
まずは、全体イメージ図から見てみましょうー  
![全体イメージ図](ci1.jpg)
ご覧の通り、developerとして、必須ファイルをgitlabの指定リポジトリにpushし、  
masterブランチにmergeされたら、あとの工程は自動的に回します。  
シンプルかつカンタン、万々歳〜  

# gitlabステージ
このステージで、Dockerfileを含め、  
image buildが必要とするすべてのファイルをzipに固めて、s3の指定バケットに転送する。  
> ここは、gitlab-ci機能を利用

## WebHookでカバーできない?
githubだと簡単にできるが、あいにくうちはプライベートのgitlabを使っており、  
止む得なく、gitlab+S3+cloudtrail+cloudwatch eventのセットで対応することにした。  

## 必要ファイルとvars
- gitlab CI/CDで（S3にputするアクションはgitlab-ciを利用）
  - AWS_ACCESS_KEY
  - AWS_SECRET_KEY
- サービスアカウント内app_deployユーザのクレデンシャル、infraに聞く
  - SSH_PRIVATE_KEY
  - 該当リポジトリをpullできるユーザのkey
- .gitlab-ci.ymlファイル内(適宜に変更)
  - UPLOAD_REGION : ap-northeast-1(※東京リージョン)
  - UPLOAD_BUCKET : code-pipeline
  - UPLOAD_FOLDER : docker-image
  - UPLOAD_FILE_ALL : docker_image.zip(※出力zipファイルのフルネーム)
- Dockerfile
- buildspec.yml(buildステージ必須)
- taskdef.json(deployテージ必須)
- appspec.yml(deployテージ必須)

## gitlab-ci.ymlサンプル

```yaml
#とにかくalpineのイメージしてしましょう、ここは社内のURLので
image: "XXXXXXXXX/alpine:latest"
services: []

variables:
#適宜に変更
  UPLOAD_REGION      : ap-northeast-1
  UPLOAD_BUCKET      : code-pipeline
  UPLOAD_FOLDER      : docker-image
  UPLOAD_FILE_ALL    : docker_image.zip
before_script:
  - date
  - export LANG=C
  - export LC_ALL=C
  - |-
    if [ "$(type ssh-agent)" = "" ]; then
      apk --update add --no-cache openssh-client git zip > /dev/null
    fi
  - eval $(ssh-agent -s)
  - echo "$SSH_PRIVATE_KEY" | ssh-add -
  - |-
    if [ "$(type aws)" = "" ]; then
      apk --update add --no-cache python3 > /dev/null
      pip3 install --upgrade --no-cache-dir pip awscli
    fi
  - mkdir -p ~/.ssh
  - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config

stages:
  - apply

apply-job:
  stage: apply
  script:
  - date
  #適宜に変更
  - git clone ssh://git@git.XXXXXXXXX.demo-nginx.git /docker_image
  - cd /docker_image
  - git rev-parse HEAD | cut -c 1-8 > git-commit-hash.txt
  - zip -r9 /tmp/${UPLOAD_FILE_ALL}  ./* > /dev/null
  - export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY}
  - export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_KEY}
  - date
  - aws s3 cp /tmp/${UPLOAD_FILE_ALL}  s3://${UPLOAD_BUCKET}/${UPLOAD_FOLDER}/${UPLOAD_FILE_ALL}  --region ${UPLOAD_REGION}
  only:
    - master
```

ご察しの通り、branchがmasterにmergeした時、CIが回され、  
すべてのファイルをzipに固め、s3の指定bucketにコピーする仕組みとなっています。  

## 成果物
- Dockerfileを含め、工程必要ファイル一式をまとめたzipファイル

# buildステージ
このステージで、copyされたzipファイルを基づいて、docker imageをbuildする。  
## 必要ファイルとvars
- CODE_PACKAGE_NAME: "docker_image.zip"(※S3指定バケット内、gitlab-ci出力されたファイル)
- GIT_COMMIT_HASH_FILE: "git-commit-hash.txt"(※image最新タグを保存する一時ファイル名)
- buildspec.yml(build工程定義ファイル)
## buildspec.ymlサンプル

```yaml
version: 0.2

env:
  variables:
  #適宜に変更
    CODE_PACKAGE_NAME: "docker_image.zip"
    GIT_COMMIT_HASH_FILE: "git-commit-hash.txt"

phases:
  install:
    commands:
      - echo `date` Starting install phase...
      - echo `date` Finished install phase...
  pre_build:
    commands:
      - echo `date` Starting pre_build phase...
      - echo Logging in to Amazon ECR...
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
      - echo `date` Finished pre_build phase...
  build:
    commands:
      - echo `date` Starting build phase...
      - COMMIT_HASH=`cat ${GIT_COMMIT_HASH_FILE}`
      - echo Building the Docker image...
      - ECR_REPO_URL=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
      - docker build --no-cache=true --tag $ECR_REPO_URL:$IMAGE_TAG --tag $ECR_REPO_URL:$COMMIT_HASH .
      - echo `date` Finished build phase...
  post_build:
    commands:
      - echo `date` Starting post_build phase...
      - COMMIT_HASH=`cat ${GIT_COMMIT_HASH_FILE}`
      - ECR_REPO_URL=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
      - echo Pushing the Docker image...
      - docker push $ECR_REPO_URL:$COMMIT_HASH
      - docker push $ECR_REPO_URL:$IMAGE_TAG
      - printf '{"Version":"1.0","ImageURI":"%s"}' $ECR_REPO_URL:$COMMIT_HASH > imageDetail.json
      - echo `date` Finished post_build phase...
artifacts:
  files:
      - imageDetail.json
      - taskdef.json
      - appspec.yml
```

## 成果物
- docker image
  - {ECR_URL}:latest
  - {ECR_URL}:commit_hash
- next stage用ファイル
  - imageDetail.json
  - taskdef.json
  - appspec.yml

# deployステージ
このステージで、buildされたimageをECSクラスターにdeploy.  
appspec.ymlとtaskdef.jsonの組み合わせで、deploy先を特定し、  
imageDetail.jsonはimageのURL（tagを含む）を特定する。
## 必要ファイルとvars
- taskdef.jsonファイル内
  - 実行ARN
  - containname
  - cloudwatch logグループ関連name
  - taskdefinitionのfamily名
- appspec.ymlファイル内
  - LBのコンテナ名

## appspec.ymlサンプル

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        #<TASK_DEFINITION>変更しないで、そのまま
        TaskDefinition: "<TASK_DEFINITION>"
        LoadBalancerInfo:
        #適宜に変更
          ContainerName: "application"
          ContainerPort: "80"
```

## taskdef.jsonサンプル

```json
{
    //適宜に変更
  "executionRoleArn": "arn:aws:iam::XXXXX:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
        //適宜に変更
      "name": "application",
      "image": "<IMAGE1_NAME>",
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
            //適宜に変更
            "awslogs-group": "/ecs/web",
            "awslogs-region": "ap-northeast-1",
            "awslogs-stream-prefix": "container"
            }
      },
      "essential": true
    }
  ],
  //fargateを使います
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512",
  //適宜に変更
  "family": "web"
}
```


## 成果物
- Blue/Green deployを経て、置き換えられたコンテナ

# 最終成果物
## パイプライン
![codepipeline](ci2.jpg)
## WebPage
予め用意したALBにアクセスすると、カスタマイズしたnginx画面が表示された。  
![web](ci3.jpg)

__これで大成果です！お疲れ様〜__