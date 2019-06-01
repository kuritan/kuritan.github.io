---
title: Gyazo(SS超便利に共有するサービス)をサーバレスで構築しますー
categories: infra
tags:

 - JP
 - Work
 - Serverless
---

久しぶりに、ブログを更新しないと〜

転職してからはや2ヶ月、入場した時と比べて、だいぶ仕事が増えました。（喜ぶところかなぁ？）
最近は、サーバレスを中心に何件ものサービスを再構築しましたので、今回その一つのGyazoというサービスについて、お話しようを思います。
<!--more-->

# 予備知識
- [Gyazoとは](https://gyazo.com/ja)  
一言で言うと、一瞬で取ったスクショを共有できるサービスです。  
- [GyazoのGithubページ](https://github.com/gyazo/)  
一般サービスとしても利用可能ですが、会社で使う時、機密情報などに配慮して、やはり自社専用のものがいいかも。  
- [サーバーレスな社内Gyazoの作り方(AWS SAM+Api Gateway+Lambda(Ruby)+S3)](https://tech.fusic.co.jp/aws/aws-serverless-gyazo/)  
重点的に参考させていただきました記事です。

# なぜやったの？
- 元々、社内Gyazoとして、AWSのEC２を立て、ストレージをS3にし、実際利用何年も利用されている
- サーバの面倒をみないといけないので、そろそろ辛い感が表に出そう
- 操作ミスで本番EC2インスタンスが削除された事故があったみたい
- 業務改善とコスト低減の旗のもと、サーバレスアーキテクチャをどんどんやりたい
- 新参者の自分には、このようなサービスのインパクトが丁度いい
  
# どうやったの？
## 元々の設計は以下となります。
![gyazo-sinatra](http://wx3.sinaimg.cn/mw690/735d420aly1g3knc2qbhhj20x30nedh2.jpg)  
ご察知の通り、かなりシンプルの構造です。  
rubyのsinatraをAPPサーバーにし、Nginxを先頭に構え、後ろはS3バケットをストレージ、シンプルだが有効なアーキテクチャです。  
実際、何年も問題なく、利用されていた。

## 今回の設計はこうだ
![gyazo-serverless](http://wx3.sinaimg.cn/mw690/735d420aly1g3knc9mtalj20x30ne41s.jpg)  
### 要件
1. ユーザが指定カスタムドメインにアクセスすると、クライアントのダウンロードページと、機能説明ページを見える
2. 1.のカスタムドメインに向けて、画像ファイルをPOSTすると、S3に保存され、アクセスできるURLが返される
3. 2.で返されたURLにアクセスすると、画像が見える
4. クライアントのコード改修をしない
5. 社内しかアクセスできない
6. リスポンスが許容範囲内にしておきたい、かつコストを抑える

__よっしゃー、じゃ今から、それぞれをご説明いたします__

### Lambda関数
クライアントのコードをしたくないので、無理やりでも、BOUNDARY処理を行う！

|key|value|
|:--:|:--:|
|Runtime|Ruby2.5|
|Handler|app.gyazo_upload|
|ENV|下に掲載|

rubyを使うので、gemファイルもアップしておこう〜
```Gemfile
source "https://rubygems.org"

gem "httparty"
gem "aws-sdk-s3"
```

関心のlambda関数がキタァァァー

```ruby
require 'json'
require 'aws-sdk-s3'
require 'base64'
require 'securerandom'

def gyazo_upload(event:, context:)
  body = event['body']
  if event['isBase64Encoded']
    # decode処理
    body = Base64.decode64(body)
    # boundary処理
    tbody = body.split(ENV['BOUNDARY'])
    sbody = tbody[2].to_s.split("\r\n\r\n")
    hbody = sbody[1].to_s.split("\r\n--")
    # randomのファイル名生成
    key = SecureRandom.urlsafe_base64
    # 保存パスを当日の日付に
    date_dir = Time.now.strftime("%y/%m/%d")
    # S3にアップする
    object = Aws::S3::Resource
             .new(region:ENV['REGION'])
             .bucket(ENV['BUCKET_NAME'])
             .put_object({ key: "#{date_dir}/#{key}.png", body: hbody[0] })
    # ユーザに返すURLを整形
    user_url = "https://" + ENV['DOMAIN'] + "/#{date_dir}/#{key}.png"
    p user_url
    {
      statusCode: 200,
      body: user_url
    }
  else
  # cloudwatch envent対応のための空振り処理
    mesbody = "exec by CloudWatch-Event."
    {
      statusCode: 200,
      body: mesbody
    }
    puts mesbody
  end
end
```
cloudwatch eventの空振り処理について、また後ほどお話しよう。


一応、ENVも貼っておきますー
```
#写真保存先バケット名
BUCKET_NAME = gyazo

#ユーザがアクセスしているドメイン名
DOMAIN = example.jp

#東京リージョン
REGION = ap-northeast-1

BOUNDARY = ご指定の文字
```


### S3 bucket
社内しかアクセスできないということにしたいので、S3バケットのアクセスポリシーを弄ってみます！

```
{
　　"Version": "2012-10-17",
　　"Id": "S3PolicyId1",
　　"Statement": [
　　　　{
　　　　　　"Sid": "IPAllow",
　　　　　　"Effect": "Allow",
　　　　　　"Principal": "*",
　　　　　　"Action": "s3:GetObject",
　　　　　　"Resource": "arn:aws:s3:::gyazo.drev.jp/*",
　　　　　　"Condition": {
　　　　　　　　"IpAddress": {
　　　　　　　　　"aws:SourceIp": [
　　　　　　　　　　　　"XXX.XXX.XXX.XXX" #このIPからしか見えない
　　　　　　　　　]
　　　　　　　　}
　　　　　　}
　　　　}
　　]
}
```

### htmlページ
htmlページは割愛させてください。  
社内gitlabのpages機能を使って、静的コンテンツをホスティングしています。

### ALB & Route53
ここは今回の関心のところですね。  
僕もずいぶん悩みました、なぜなら、元々のシステムでは、Nginxによるproxy_passで、URLを書き換えられることがあります。これによって、S3のendpointをユーザに隠し、ちょっとだけ短縮したURLを生成できた。  
なので、後方互換性を保つため、Nginxの機能の代打も考えないといけない。  
悩む末に、ALBのリスナールールを使うことになりました。  

ルールは以下になります。  
___HTTP:80___
```
IF
それ以外の場合はルーティングされないリクエスト
THEN
リダイレクト先https://#{host}:#{port}/#{path}?#{query}
ステータスコード:HTTP_301
```

___HTTPS:443___
```
IF
HTTP リクエストメソッドはGET
パスが/*/*/*
THEN
リダイレクト先#{protocol}://s3EndoPoint:#{port}/#{path}?#{query}
ステータスコード:HTTP_301

IF
HTTP リクエストメソッドはGET
THEN
リダイレクト先#{protocol}://downloadpage.html/?
ステータスコード:HTTP_301

IF
それ以外の場合はルーティングされないリクエスト
転送先 XXXXX(ターゲットグループ。そこからlambda関数に)
```

この３つのルールが実現できる機能を説明すると、
1. HTTPのリクエストが来たら、HTTPSにリダイレクト
2. GETのリクエスト＋パスが/* /* /*の形だったら、S3が格納している指定オブジェクトを表示
3. GETのリクエストが来たら、静的コンテンツをホスティングされているWEBページを表示
4. 上記以外の場合、lambda関数に引き渡す

### CloudWatch
ここはちょっとしたおまけですね。  
実際にテストすると、しょっちゅうURLが返るまで時間かかると時がありました。  
試行錯誤後、cloudwatch eventを設定し、3分1回指定のlambda関数を起動されることによって、常にlambda関数のアクティブ状態を確保できた。まぁ、ちょっとコストかかるが、これくらいならいいじゃないかなぁ〜  
もちろん、lambda関数の方の空振り処理はこれのためです。

# 成果物
![gyazo](http://wx2.sinaimg.cn/mw690/735d420agy1g3ljg2l3bzg20dc07p7wh.gif)

これで、ぱっとできる超簡単なスクショ共有サービス（ノーメンテ）バージョンが完成だ！
それでは、この辺に終わりにしましょうか。  
お疲れ様でした〜