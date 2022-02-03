---
title: Assume roleでクロスアカウントのAWSリソースを弄ろう
date: 2022-02-02 16:21:24
categories: infra
tags:
- JP
- aws
---

ようやく気が向いてて、何か書こうかと思っていたら、やっぱAWSネタになっちょうね。  
今回は二つのAWSアカウントの間で、双方向assume roleでAWSリソースを操作する方法を試してみました。  
良かったら、しばしお付き合いくださいましー
<!--more-->

## 背景

以前も、こちらのブログで書いたことがある[chatworkとlambdaでセルフサービスしようぜ](https://kuritan.github.io/selfservice-with-chatworkwebhook-apigateway-and-lambda/)ですが、  
iptablesによるsshguardの解除セルフサービスは、また新しいニーズが出てきて、それを対応するための実装は、今回のお話です。  

### 新ニーズ

- 既存を維持しつつ、別のec2インスタンスもbot経由で、解除できるようにしたい
- 解除対象となる2台のec2インスタンスは、別々のAWSアカウントに存在する
- コストとセキュリティをある程度、両立したい

## 構成

おやおや、困ったものだね〜と呟きつつ、前回の設計を見直し、ズバリこれだ！  
![構成図](bidirectional-assume-role-for-aws.png)   

基本は、前回の設計を踏襲したモノとなりますが、  
その上、クロスアカウント対応のため、lambdaからassume roleをリクエストし、その権限をもって、別AWSアカウントのリソースを操作する。  
※双方向assume roleが必要  

次に詳しく、各登場キャラクターの役割を説明しようと思います。  

## 各キャラクターの役割

### Account A

#### bot user

専用のbotユーザにTOをつけて話しかけると、webhookが叩かれ、apigatewayに送信  
※ここではchatwork userとなります  

IPアドレスのみの場合、IP一覧チェックを行い、解除&更新もする  
BAN部屋のメッセージを引用された場合、IP一覧チェックを行い、解除&更新もする  
それ以外の場合、デフォルトhelpメッセージを返す  

#### API gateway

chatwork webhookから受信し、lambdaでtoken検証をする  
問題ないの場合、bodyの解析を行う  

- IPアドレスのみの場合、IP一覧チェックを行い、解除&更新もする
  - ない場合は、その旨をchatworkに返す
- BAN部屋のメッセージを引用された場合、IP一覧チェックを行い、解除&更新もする
  - ない場合は、その旨をchatworkに返す
- それ以外の場合、デフォルトhelpメッセージを返す

#### lambda function

systemmanagerとs3のSDKを使って、shell実行やipチェックを行う

- s3の指定場所に、現在BANされているIPアドレスの一覧リスト(対象サーバー分のファイルが存在、ファイル名はサーバー名となっている)をダウンロード
- bot userから指定されたIPが、リストに載ってるかをチェック
  - 載ってる場合、BANリストのパスとBANされたiptables ruleと組み合わせたdictをlambdaに返し、その解除&更新もする
  - 載ってない場合、Noneをlambdaに返し、該当IPが存在しない旨をchatworkに返す

※api gatewayは最大3秒のtimeoutが設定されており（変更不可）、都度各サーバーでiptablesをチェックするより、  
s3上にBAN listをアップロードし、それらをチェックした方がレスポンスが早いので、こちらのいうな実装になっております。

[前回](https://kuritan.github.io/selfservice-with-chatworkwebhook-apigateway-and-lambda/)の実装をいつくか変更していたが、イメージだけこちらで書きます。(そのままでは動けない可能性あり)  

```python
from boto3.session import Session

# assume roleのセッションを取得
def get_sts_session():
    role_arn = "arn:aws:iam::[Account B]:role/[role name]"
    session_name = "aws-infra-sg" #なんか適当な名前
    region = "ap-northeast-1"
    client = boto3.client('sts')

    # assume_roleでロールに設定された権限のクレデンシャル(一時キー)を発行する
    response = client.assume_role(
        RoleArn=role_arn,
        RoleSessionName=session_name
    )
    session = Session(
        aws_access_key_id=response['Credentials']['AccessKeyId'],
        aws_secret_access_key=response['Credentials']['SecretAccessKey'],
        aws_session_token=response['Credentials']['SessionToken'],
        region_name=region
    )
    return session

# 取得したassume roleセッションを利用
def send_ssm_cmd(ban_ip: str, pool: str) -> bool:
    try:
        if pool == opengate_ban_ip_pool:
            session = get_sts_session()
            ec2 = session.client('ec2')
            ssm = session.client('ssm', region_name='ap-northeast-1')
            ban_ip_pool = opengate_ban_ip_pool
            aws_profile = "--profile infra"
        else:
            ban_ip_pool = gitlab_ban_ip_pool
            aws_profile = ""

```

#### cloudwatch log

lambda functionの実行logのたまり場です。  
debug時には、相当役に立つ

#### system manager

実際リソース操作するインターフェイスです。  
run command経由で、amazon-ssm-agentに命令し、shellコマンドを実行してもらうようにしています  

```shell
echo %s" % ban_ip
echo `date` > /tmp/`hostname`.txt
echo `hostname` >> /tmp/`hostname`.txt
iptables -L --line-numbers -n | grep %s | cut -d' ' -f1 | xargs -L 1 iptables -D sshguard" % ban_ip
systemctl restart sshguard
iptables -L sshguard --line-numbers -n > /tmp/`hostname`.txt
aws s3 cp %s s3://%s/ %s" % (ban_ip_pool, s3_bucket_name, aws_profile)
```

#### ec2

amazon-ssm-agentを事前にインストールし、起動させる必要があります。  
system managerからの命令を受け、それを実行する。  
※インスタンス特定できるように、指定のtagをつける  
※bot: release-sshguard  

#### s3 bucket

BAN listファイルの格納場所  
サーバーごとのBAN listファイルがここに羅列されています  
※ファイル名は、サーバー名となってます  

#### cloudwatch events(今event bridgeという名前になった)

BAN listが随時更新させるように、1分ごとで、system manager経由で最新BAN listファイルをs3にアップロードさせるための機能  
EC2インスタンスのBAN listが実際更新されつつではあるが、頻度やコストの兼ね合いで、1分間/回の更新はもう十分と考えていました。  

#### iam(sts)

クロスアカウントのassume roleのため、新しいiam roleを作り、assumeできる権限を付与しています。  
※Account Bに利用させるため  

```terraform
# Account Aのlambda実行roleに、Account Bのassume roleを許可
resource "aws_iam_role_policy" "lambda_cross_account" {
  name   = "lambda_cross_account"
  role   = aws_iam_role.lambda.id
  policy = <<JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
        "Action": "sts:AssumeRole",
        "Effect": "Allow",
        "Resource": "arn:aws:iam::[Account B]:role/InstanceProfile-for-infra-sts"
        }
  ]
}
JSON
 
}
```

```terraform
# Account Aで新規s3を操作できるroleを作り、Account Bのassumeroleも許可
 
resource "aws_iam_role" "ssm_cross_account" {
  name               = "AssumeRole-for-sshban-s3"
  assume_role_policy = <<JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::[Account B]:role/InstanceProfile-for-infra-sts"
      },
      "Action": "sts:AssumeRole",
      "Condition": {}
    }
  ]
}
JSON
 
}
 
resource "aws_iam_role_policy" "ssm_cross_account" {
  name   = "ssm_cross_account"
  role   = aws_iam_role.ssm_cross_account.id
  policy = <<JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
        "Action": "sts:AssumeRole",
        "Effect": "Allow",
        "Resource": "arn:aws:iam::[Account B]:role/InstanceProfile-for-infra-sts"
        }
  ]
}
JSON
 
}
 
resource "aws_iam_role_policy" "s3_cross_account" {
  name   = "handle-sshguard-ban-list"
  role   = aws_iam_role.ssm_cross_account.id
  policy = <<JSON
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:ListAllMyBuckets",
            "Resource": "arn:aws:s3:::*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": "arn:aws:s3:::sshguard-ban-list"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::sshguard-ban-list/*"
        }
    ]
}
JSON
 
}
```

### Account B

#### system manager

Account Aのlambdaによって呼び出され(assume role権限)、対象ec2インスタンスを操作するインターフェイスです

#### ec2

対象インスタンスにamazon-ssm-agentの事前インストールが必要

#### cloudwatch events(今はevent bridgeという名前になった)

Account Aと同じく

#### iam(sts)

Account A時と逆で、Account Aに利用させるためのモノになります

## 最後

まとめてみたら、意外とめっちゃ長くなったね...  
セキュアで別AWSアカウントのリソースを利用するには、assume roleは避けて通れない機能なので、ぜひ使ってみてくださーい！  
