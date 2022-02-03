---
title: chatworkとlambdaでセルフサービスしようぜ
date: 2021-04-21 11:52:17
categories: dev
tags:
- JP
- aws
- python
---

# ニーズ
DoS攻撃などを防止するため、sshguardを標準装備として各本番サーバにはインストールされている。  
昨今、リモートワークが当たり前になって、従業員からのアクセスをBANされたケースが多発され（特にgitまわり）、  
そのBANのIPの解除はSREメンバーが都度手動でしないといけない状況になっています。  
エンジニアのtoil撲滅の一環として、これらの作業をセルフサービス化としたいのは今回のゴールである。  
<!--more-->
# 考慮ポイント
- 対象
- コスト
- 実装方法
  - webサービス
    - 実装が遅い
    - 自前でユーザ認証する必要あり（リモートワーク前提ので）
    - 実装まわりで経験値を大量ゲットできる
  - botユーザ(chatwork)
    - 実装が速い
    - ユーザ認証不要（すでにchatworkの方で認証済み）
    - 実装まわりで経験値少量ゲット

# アーキテクチャ
![Remote-Containers](アーキテクチャ.png)
## TO:bot
専用のbotユーザにTOをつけて話しかけると、webhookが叩かれ、apigatewayに送信

- IPアドレスのみの場合、IP一覧チェックを行い、解除&更新もする
- BAN部屋のメッセージを引用された場合、IP一覧チェックを行い、解除&更新もする
- それ以外の場合、デフォルトhelpメッセージを返す

## APIgateway - lambda
chatwork webhookから受信し、lambdaでtoken検証をする。
問題ないの場合、bodyの解析を行う

- IPアドレスのみの場合、IP一覧チェックを行い、解除&更新もする
  - ない場合は、その旨をchatworkに返す
- BAN部屋のメッセージを引用された場合、IP一覧チェックを行い、解除&更新もする
  - ない場合は、その旨をchatworkに返す
- それ以外の場合、デフォルトhelpメッセージを返す


### 実装ポイント

- timeout 3s要注意(apigateway仕様)
- systemmanagerとs3のSDKを使って、shell実行やipチェックを行う
- 返信メッセージを丁寧に表現

## 一覧チェック
現在BANされているIPアドレスの一覧リストをテキストファイルとして、s3の指定場所に格納
lambdaでs3 SDK経由、都度指定されたIPが、リストに載ってるかをチェック

- 載ってる場合、trueをlambdaに返し、その解除&更新もする
- 載ってない場合、falseをlambdaに返し、その旨をchatworkに返す

## kick system manager run command
lambdaでsystemmanager SDKを利用し、指定EC2インスタンスにて、shell commandを実行

## shell実行(解除&更新)
- iptables -L sshguard --line-numbers -n | grep #BANされたIP# | cut -d' ' -f1 | xargs -L 1 iptables -D sshguard 
- systemctl restart sshguard
- iptables -L sshguard --line-numbers -n > /tmp/sshguard_ban_ip/`hostname`.txt
- aws s3 sync /tmp/sshguard_ban_ip/ s3://#S3の指定bucket名#/
実行成功/失敗後、その旨をchatworkに返す

## レスポンス
↑の各ステップでのレスポンス

## cloudwatch event kick system manager run command
短時間/回で、BANされたIPアドレス一覧を最新化する

## shell実行(更新)
- iptables -L sshguard --line-numbers -n > /tmp/sshguard_ban_ip/`hostname`.log
- aws s3 sync /tmp/sshguard_ban_ip/ #S3の指定bucket名#

## 一覧最新化
↑での結果が反映される

ついでに、s3バケットを静的コンテンツとして表示させるのもあり
```
<!DOCTYPE html>
<html lang="ja">
<head>
<meta http-equiv="CONTENT-TYPE" cont
ent="text/html; charset=utf-8" />
<title>BAN IP List</title>
</head>
<body>
<OBJECT DATA="./<filename.txt>"TYPE="text/plain" WIDTH="100%" HEIGHT="100%"></OBJECT>
</body>
</html>
```

# 成果物
__構成は全部terraformに載っており、lambda functionはpython 3.6を使わせていただきました。__
## terraformコード
```terraform
#
# for sshguard self release
#
locals {
  ip_tables_chain_path = "/tmp"
  # EC2 instance's name for running release action
  target_instance_name = ""
  ban_ip_pool = format("%s/%s.txt", local.ip_tables_chain_path, local.target_instance_name)
  # S3 bucket name for contain ban list
  s3_bucket_name = ""
  tag_key = "tag:bot"
  tag_value = "release-sshguard"
}
resource "aws_api_gateway_rest_api" "release-sshguard" {
  name = "release-sshguard"
}

resource "aws_api_gateway_resource" "proxy" {
   rest_api_id = aws_api_gateway_rest_api.release-sshguard.id
   parent_id   = aws_api_gateway_rest_api.release-sshguard.root_resource_id
   path_part   = "{proxy+}"
}

resource "aws_api_gateway_method" "proxyMethod" {
   rest_api_id   = aws_api_gateway_rest_api.release-sshguard.id
   resource_id   = aws_api_gateway_resource.proxy.id
   http_method   = "ANY"
   authorization = "NONE"
}

resource "aws_api_gateway_integration" "release-sshguard" {
   rest_api_id = aws_api_gateway_rest_api.release-sshguard.id
   resource_id = aws_api_gateway_method.proxyMethod.resource_id
   http_method = aws_api_gateway_method.proxyMethod.http_method

   integration_http_method = "POST"
   type                    = "AWS_PROXY"
   uri                     = aws_lambda_function.release-sshguard.invoke_arn
}

resource "aws_api_gateway_method" "proxy_root" {
   rest_api_id   = aws_api_gateway_rest_api.release-sshguard.id
   resource_id   = aws_api_gateway_rest_api.release-sshguard.root_resource_id
   http_method   = "ANY"
   authorization = "NONE"
}

resource "aws_api_gateway_integration" "lambda_root" {
   rest_api_id = aws_api_gateway_rest_api.release-sshguard.id
   resource_id = aws_api_gateway_method.proxy_root.resource_id
   http_method = aws_api_gateway_method.proxy_root.http_method

   integration_http_method = "POST"
   type                    = "AWS_PROXY"
   uri                     = aws_lambda_function.release-sshguard.invoke_arn
}


resource "aws_api_gateway_deployment" "apideploy" {
   depends_on = [
     aws_api_gateway_integration.release-sshguard,
     aws_api_gateway_integration.lambda_root,
   ]

   rest_api_id = aws_api_gateway_rest_api.release-sshguard.id
   stage_name  = "test"
}

resource "aws_lambda_function" "release-sshguard" {
  function_name     = "release-sshguard"
  handler           = "release-sshguard.lambda_handler"
  s3_bucket         = local.lambda_bucket
  s3_key            = local.lambda_main_key
  s3_object_version = data.aws_s3_bucket_object.lambda_main.version_id

  layers = [aws_lambda_layer_version.system-lib.arn]

  memory_size = 512
  timeout     = 3

  runtime = local.lambda_runtime
  role    = "arn:aws:iam::596431367989:role/lambda_basic_vpc_execution"

  environment {
    variables = {
      ec2_tag_key = local.tag_key
      ec2_tag_value = local.tag_value
      ban_list_s3_bucket_name = local.s3_bucket_name
      ec2_hostname = local.target_instance_name
    }
  }
}

resource "aws_cloudwatch_log_group" "release-sshguard" {
  name              = format("%s%s", local.lambda_log_group_prefix, "release-sshguard")
  retention_in_days = 7
}

resource "aws_lambda_permission" "release-sshguard" {
  function_name = aws_lambda_function.release-sshguard.arn
  principal     = "apigateway.amazonaws.com"
  action        = "lambda:InvokeFunction"

  source_arn = "${aws_api_gateway_rest_api.release-sshguard.execution_arn}/*/*"
}

resource "aws_s3_bucket" "ban-list" {
    bucket = local.s3_bucket_name
    acl    = "private"

    website {
        index_document = "index.html"
        error_document = "index.html"
    }

    policy = <<EOS
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Office",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::${local.s3_bucket_name}/*",
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": [
                        "your branch's ip address",
                    ]
                }
            }
        }
    ]
}
EOS

}
output "api_url" {
  value = aws_api_gateway_deployment.apideploy.invoke_url
}


#
# update ban ip list via cloudwatch event
#
resource "aws_iam_role" "update-sshguard-ban-ip" {
  
  name = "update-sshguard-ban-ip"
  assume_role_policy = <<EOF
{
      "Version": "2012-10-17",
      "Statement": [
        {
          "Action": "sts:AssumeRole",
          "Principal": {
            "Service": "events.amazonaws.com"
          },
          "Effect": "Allow",
          "Sid": ""
        }
      ]
    }
EOF
}

resource "aws_iam_role_policy" "update-sshguard-ban-ip" {
  name   = "update-sshguard-ban-ip_additional"
  role   = aws_iam_role.update-sshguard-ban-ip.id
  policy = <<JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "ssm:*",
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
JSON

}


resource "aws_cloudwatch_event_target" "update-sshguard-ban-ip" {
  target_id = "UpdateBanIPList"
  arn       = "arn:aws:ssm:${var.aws_region}::document/AWS-RunShellScript"
  input     = <<JSON
  {
    "commands": [
      "echo `date` > ${local.ban_ip_pool}",
      "echo `hostname` >> ${local.ban_ip_pool}",
      "iptables -L sshguard --line-numbers -n >> ${local.ban_ip_pool}",
      "aws s3 cp ${local.ban_ip_pool} s3://${local.s3_bucket_name}/"
    ]
  }
  JSON

  rule      = aws_cloudwatch_event_rule.lambda_every_one_minute.name
  role_arn  = aws_iam_role.update-sshguard-ban-ip.arn

  run_command_targets {
    key    = "tag:bot"
    values = [local.tag_value]
  }
}
```

## lambda functionコード
```python
import os
import boto3
import logging
import ipaddress
import json
import requests

logger = logging.getLogger()
logger.setLevel(logging.INFO)

ec2 = boto3.client('ec2')
ssm = boto3.client('ssm', region_name=os.environ.get('region_name'))
s3 = boto3.resource('s3')

s3_bucket_name = os.environ.get('ban_list_s3_bucket_name')
hostname = os.environ.get('ec2_hostname')
host_pool_filename = "%s.txt" % hostname
tag_key = os.environ.get('ec2_tag_key')
tag_value = os.environ.get('ec2_tag_value')
ip_tables_chain_path = '/tmp'
ban_ip_pool = "%s/%s.txt" % (ip_tables_chain_path, hostname)

# your token
API_TOKEN = ''
endpoint = 'https://api.chatwork.com/v2'
request_timeout = 3


def lambda_handler(event, context):
    data = json.dumps(event)
    j = json.loads(data)
    webhook_body = eval(j["body"])
    from_account_id = webhook_body["webhook_event"]["from_account_id"]
    room_id = webhook_body["webhook_event"]["room_id"]
    message_id = webhook_body["webhook_event"]["message_id"]
    webhook_text = webhook_body["webhook_event"]["body"]

    request_ip = ""
    open(ban_ip_pool, 'w').close()
    s3.meta.client.download_file(s3_bucket_name, host_pool_filename, ban_ip_pool)

    if is_valid_quote(webhook_text) is True:
        print("Mostly received a quote message")
        temp_file_path = "/tmp/ban_ip_message.txt"
        with open(temp_file_path, 'w') as f:
            print(webhook_text, file=f)
        filtered_ip_address = filter_ip_address_from_tempfile(temp_file_path)
        if is_valid_ip(filtered_ip_address) is not True:
            print("Do not find IP address from quote message.")
            print("Sent help message.")
            bot_message(from_account_id, room_id, message_id, "help")
            return
        else:
            print("ip valid succeed.")
            request_ip = filtered_ip_address
    elif pick_up_raw_ip(webhook_text) is True:
        if is_valid_ip(raw_ip) is True:
            print("Mostly received a single IP")
            print("ip valid succeed.")
            request_ip = raw_ip
        else:
            print("ip invalid.")
            if "ありがとう" in webhook_text or "thank" in webhook_text:
                print("Mostly got a thank you message.")
                bot_message(from_account_id, room_id, message_id, "pleasure")
                return
            else:
                bot_message(from_account_id, room_id, message_id, "help")
                print("Sent help message.")
                return
    else:
        bot_message(from_account_id, room_id, message_id, "help")
        print("Sent help message.")
        return

    if is_request_ip_existed(request_ip, ban_ip_pool) is not True:
        print("Request IP address has not existed on %s." % ban_ip_pool)
        # bot返信（from_account_idに）
        bot_message(from_account_id, room_id, message_id, "nohit")
    else:
        print("Request IP address has existed on %s." % ban_ip_pool)
        send_ssm_cmd(request_ip)
        # bot返信（from_account_idに）
        bot_message(from_account_id, room_id, message_id, "hit")
    return 'End'


def pick_up_raw_ip(post_message: str) -> bool:
    global raw_ip
    try:
        raw_ip = post_message.rsplit('\n', 1)[1]
        print(raw_ip)
        return True
    except (ValueError, IndexError):
        return False


def is_valid_ip(ip: str) -> bool:
    try:
        ipaddress.ip_address(ip)
        return True
    except ValueError:
        return False


def is_valid_quote(message: str) -> bool:
    try:
        if "[qt]" in message:
            return True
        else:
            return False
    except ValueError:
        return False


def filter_ip_address_from_tempfile(tempfile_path: str) -> str:
    with open(tempfile_path) as f:
        lines = f.readlines()
    lines_strip = []
    for line in lines:
        lines_strip.append(line.strip())
    l_message = []
    for line in lines_strip:
        if 'Message    :' in line:
            l_message.append(line)
    l_ip = ''.join(l_message).split(':')
    str_ip = ''.join(l_ip[1]).replace(' ', '')
    return str_ip


def is_request_ip_existed(request_ip: str, ban_ip_pool: str) -> bool:
    with open(ban_ip_pool) as f:
        lines = f.readlines()
        print(lines)
    lines_strip = []
    for line in lines:
        lines_strip.append(line.strip())
    l_message = []
    for line in lines_strip:
        if request_ip in line:
            l_message.append(line)
            print(l_message)
            return True


def send_ssm_cmd(ban_ip: str) -> bool:
    try:
        # EC2 instances which tagged with release-sshguard
        ec2_resp = ec2.describe_instances(Filters=[{'Name': tag_key, 'Values': [tag_value]}])

        ec2_count = len(ec2_resp['Reservations'])
        if ec2_count == 0:
            logger.info('No EC2 is running')

        instances = [i["InstanceId"] for r in ec2_resp["Reservations"] for i in r["Instances"]]

        ssm.send_command(
            InstanceIds=instances,
            DocumentName="AWS-RunShellScript",
            Parameters={
                "commands": [
                    "echo %s" % ban_ip,
                    "echo `date` > %s" % ban_ip_pool,
                    "echo `hostname` >> %s" % ban_ip_pool,
                    "iptables -L sshguard --line-numbers -n | grep %s | cut -d' ' -f1 | xargs -L 1 iptables -D sshguard" % ban_ip,
                    "systemctl restart sshguard",
                    "iptables -L sshguard --line-numbers -n >> %s" % ban_ip_pool,
                    "aws s3 cp %s s3://%s/" % (ban_ip_pool, s3_bucket_name),
                ],
                "executionTimeout": ["3600"]
            },
        )
        print("Released ip address %s" % ban_ip)
        return True

    except Exception as e:
        logger.error(e)
        raise e


def bot_message(account_id: int, room_id: int, message_id: str, message_type: str):
    if message_type is 'pleasure':
        message = "Always a great pleasure."
    elif message_type is 'nohit':
        message = '''
[info][title]作業失敗[/title]
BANリストに載ってないみたいので、
もう一度接続をお試しください。
駄目でしたら、SREメンバーまでご連絡ください。[/info]
'''.strip()
    elif message_type is "hit":
        message = '''
[info][title]作業成功[/title]
解除しました!!!
Retryをお願いしまーす[/info]
'''.strip()
    else:
        message = '''
[info][title]もしやBAN解除したいの？[/title]
- SSHブロック通知部屋の該当BAN通知メッセージを引用し、私にTOしてください
- 直接Global IPを私にTOしてください
    - https://www.cman.jp/network/support/go_access.cgi
    - ここで表示されたIPの事
- それでも駄目だったら、SREメンバーにお尋ねください[/info]
'''.strip()

    reply_message = "[rp aid=%s to=%s-%s]%s" % (account_id, room_id, message_id, message)
    post_message_url = "%s/rooms/%s/messages" % (endpoint, room_id)
    headers = {'X-ChatWorkToken': API_TOKEN}
    params = {'body': reply_message}

    res = requests.post(post_message_url, headers=headers, data=params)
    res.raise_for_status()


if __name__ == '__main__':
    print(lambda_handler("event", "context"))
```

誰かのご参考になって頂ければ幸いです。  
ではでは〜