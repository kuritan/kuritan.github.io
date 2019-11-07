---
title: AWS ConfigとLambda使って全開放ポート(0.0.0.0/0)を抹殺しよう
date: 2019-11-07 18:17:34
categories: infra
tags:
- JP
- AWS
---
突然ですが、こんなお悩みがお持ちでしょうか。
- 社内developer共用のAWSアカウントを作って、クラウド知識を社内に布教しようと思った
- 勝手に全開放(ingress 0.0.0.0/0)のSecurityGroupが適用された
- 気づいたら、もうインターネットで天然のハニーポット扱い

こんなアナタに、今日の品物をオススメします！
<!--more-->

# 機能紹介
- AWS Configを利用しSecurityGroupを常時監視し、ingressで0.0.0.0/0 allowのルールを検索
- 発見次第、SNS経由で発信
- 予め設置したlambda関数がトリガーされ、該当全開放ルールをリプレース
- インシデントマネジャーからアラート受信（オプション）

![全体イメージ](http://wx3.sinaimg.cn/mw690/735d420agy1g8plt36jwyj20ne0gkdgf.jpg)

# どうやって使うの？
基本は全部githubにあげて、READMEに記載しましたが、ちょっとだけ解説します。  
セルフサービスの方は、下記リンクへどうぞ  
__[Github repo](https://github.com/kuritan/aws_config_replace_unauthorized_ports)__

## AWS Config
まず、Config機能を有効化し、EC2リソースにモニタリングできるように設定してください。（全リソースでも構いませんが、費用感が若干違う）


## AWS Config Rule
ルールを新規追加しましょう。  
AWS提供のモノで大丈夫です、名前は「VPC_SG_OPEN_ONLY_TO_AUTHORIZED_PORTS」  

![config](http://wx1.sinaimg.cn/mw690/735d420agy1g8pn6gn3w2j20rj0ow415.jpg)

## AWS Config Rule 修復アクション
ここは、今回の肝ですね。  
AWS提供の修復アクションはいろいろあって、中に「vpc-sg-open-only-to-authorized-ports」こいうものが使いそうだが、実際やってみました。  
結果、駄目でしたー  
何が駄目というと、デフォルトVPC以外のSecurityGroupには対応できないことです。  
これじゃ監視の意味が薄いので、VPC関係なく対応してもらいたいですね。  
ここは、やはり自前でlambda関数で対応する道を選びました。  

ちょっと話が長くなたっが、ここで「PublishSNSNotification」という修復アクションを選択してください。  
もちろん、それに合わせてIAMロールも準備してあげてくださいね。
![config rule](http://wx3.sinaimg.cn/mw690/735d420agy1g8pn6jy2kgj20qo0mqtbo.jpg)

修復アクションの実行履歴は、AWS SystemManagerで確認できます。

## AWS SNS トピック
新規SNSトピックを作りましょう。  
あとで、lambda関数とインシデントマネジャーをこちらのサブスクリプションに入れるような感じですね。

## AWS Lambda Function
コードは以下となります。
ENVとして、authorized_global_ipv4を設定しましょう。(0.0.0.0/0にリプレースするIPに設定、例えばオフィス拠点のグローバルIP)  
unauthorized_ipv4 は明示的に0.0.0.0/0を表明するためのもので、素直に0.0.0.0/0に設定してくださいね。

```python3:config-lambda.py
import os
import json
import boto3
 
def lambda_handler(event, context):
    message_unicode = event['Records'][0]['Sns']['Message']
    print(message_unicode)
    id = message_unicode.strip('{ "').strip('"}')
    print(id)

    unauthorized_ipv4 = os.environ['unauthorized_ipv4']
    authorized_global_ipv4 = os.environ['global_ipv4']

    describe_sg_all = boto3.client('ec2')
    handle_sg_all = boto3.resource('ec2')
    describe_sg = describe_sg_all.describe_security_groups(GroupIds=[id])
    handle_sg = handle_sg_all.SecurityGroup(id)
    
    print(describe_sg)
    
    for i in describe_sg['SecurityGroups']:
        print("Security Group Name: "+i['GroupName'])
        print("The Ingress rules are as follows: ")
        for j in i['IpPermissions']:
            print("IP Protocol: "+j['IpProtocol'])
            try:
                print("FromPORT: "+str(j['FromPort']))
                print("ToPORT: "+str(j['ToPort']))

                for k in j['IpRanges']:
                    print("IP Ranges: "+k['CidrIp'])

                    if k['CidrIp'] == unauthorized_ipv4 :
                        authorize_response = handle_sg.authorize_ingress(
                                    IpPermissions=[
                                        {
                                            'FromPort': int(j['FromPort']),
                                            'IpProtocol': j['IpProtocol'],
                                            'ToPort': int(j['ToPort']),
                                            'IpRanges': [
                                                {
                                                    'CidrIp': authorized_global_ipv4
                                                }
                                            ]
                                        }
                                    ]
                        )
                        revoke_response = handle_sg.revoke_ingress(
                                    IpPermissions=[
                                        {
                                            'FromPort': int(j['FromPort']),
                                            'IpProtocol': j['IpProtocol'],
                                            'ToPort': int(j['ToPort']),
                                            'IpRanges': [
                                                {
                                                    'CidrIp': unauthorized_ipv4
                                                }
                                            ]
                                        }
                                    ]
                        )
                        print("Security Group Changed")
                    else:
                        print("No Security Group Changed")
            except Exception as e:
                    print(e.args)
                    print("No value for ports and ip ranges available for this security group")
                    continue
        return 'end'
```


トリガーを先程作ったSNSトピックに設定し、テストに以下のようなモノを作りましょう。

```
{
  "Records": [
    {
      "Sns": {
        "Timestamp": "2016-11-17T08:34:04.436Z",
        "Message": "{ \"sg-XXXXX\"}"
      }
    }
  ]
}
```

テストのため、実際0.0.0.0/0のSecurityGroup（portはどうでもいいが、80,443でいいかなぁ）を作って、IDをとって、上記の「sg-XXXXX」に書き換えてください。

lambda関数のページで「テスト」を押したら、実行されるはずですね。  


# インシデントマネジャー(オプション)
今の所属会社はインシデント管理のため、Pagerdutyを使っています。  
それのEMAILアドレスを上記SNSトピックにサブスクすれば、全開放のSecurityGroup IDが送信されます。

# まとめ
ここまで終わりです。お疲れ様でした。  
一回苦労すれば、後々は安心ですがら、やる価値はあるかと思いますよー  

じゃ今回もここまでにします、またねー