---
title: ルータconfig自動化にいざ挑戦
date: 2018-05-20 11:24:15
categories: infra
tags: 
 - python
 - JP
---

# 背景
新拠点立ち上げ時、必ず本社ネットワークと接続しなければならない要件がある。
今まで、毎回手入力で拠点ルータ作成＆本社ルータコンフィグ修正を行いましたが、ミスの可能性が多く、ダブルチェックもかなり時間がかかる。これを改善したく、全プロセスの自動化改造に挑戦。
<!--more-->

# デバイス
- Yamaha RTX 1200（本社）
- Yamaha RTX 1200（拠点）

# 備考
- スクリプト本体とパラメータファイル両方に分けて、毎回違うIPやPP情報をパラメータファイルに記入しスクリプトを動かせば、自動的にconfigをルータに入力
- paramikoライブラリーはルータやスィッチのような機器に対応するので、pythonを使う
- SSHを利用することになるので、事前にルータのSSH configを手入力
- SSH維持のため、事後自動削除はできない。LAN3ポートIPとルート一つ追加、業務に影響なし
- 本社側ルータconfigの追加＆削除も視野に入れる
- 事前config、作業分log、事後config自動生成＆保存
- 事前に店舗の現地IPを知ることができないので、pp固定IPとtunnel内local ipは自動化対象外（手入力必要）
- 実行環境標準化のため、dockerを使う
- 比較的新しいバージョンのdockerとdocker-composeが入っている機器をホスト機として選定

# 拠点新設時、ルータ作成
- 事前手入力で、ssh関連、lan3 ip、DCへのルート、ログインユーザとPW、admin pwを設定しなければならない
- SSH維持のため、事後自動削除はできない。LAN3ポートIPとルート一つ追加、業務に影響なし
- 事前必ず「SetRTX1200_Config_parameter.ini」を状況に応じて修正

___[Github Link](https://github.com/kuritan/paramiko)___
>- SetRTX1200_AddConfig_branch.py
>- SetRTX1200_Config_parameter.ini


# 本社ルータ修正
- 事前必ず「SetRTX1200_Config_parameter.ini」を状況に応じて修正

___[Github Link](https://github.com/kuritan/paramiko)___
>- DelRTX1200_DelConfig_HQ.py
>- AddRTX1200_AddConfig_HQ.py

パラメータファイルですが、毎回ワンセットで、店舗側と本社側は同じファイルを使う
SetRTX1200_Config_parameter.ini

# 実行環境
- centosVM(briana181)
- python2.7.X
- パッケージ: paramiko(バージョンこだわりがない)

# 環境準備
## 目標
** 柔軟な運用や継続改善をサポート(要は複数人でも、ローカルで編集し、コンテナで実行・テストを可能に)するため、本スクリプト必要になるpython2.7.Xおよび関連ライブラリーをdocker化し、centosVMで起動させる。**

コンテナbuildや管理の利便性の視点から、結構新しいバージョンのdocker-composeが入っているbriana181を今回のホスト機として利用。

作業として、主に下記のようになります
>- ボリューム準備
>- Dockerfile準備
>- docker-composeファイル準備
>- イメージbuild & コンテナUP


当然ですが、完成品も準備済みデス！

___[dockerHub link](https://hub.docker.com/r/kuritan/paramikoenv/)___

# ボリューム準備(ホストにて)
dockerデータの永続化を実現するため、ホスト側でボリュームを作成し、コンテナ側にマウントさせる。
- 今回のボリューム（ディリクトリ）
/home/AutoRTX-python2.7/

mkdirしてから、作成済みのpythonスクリプトを全部こちらにコピー。

以下のようなディレクトリ構成を想定しています。

```
/home/AutoRTX-python2.7/
　　　　　　　　　　　　　├── Dockerfile
　　　　　　　　　　　　　└── requirements.txt
　　　　　　　　　　　　　└── docker-compose.yml
　　　　　　　　　　　　　└── sample.py 　# 対象のスクリプト
　　　　　　　　　　　　　└── run_autoRTX.sh　# docker環境経由実行用スクリプト 
```

後程、詳しく各部分をご説明します。

# Dockerfile準備
必要な元imageとパッケージをインストールした Dockerfile を作成

```
      1 FROM python:2.7-jessie
      2 MAINTAINER zai
      3
      4 WORKDIR /home/AutoRTX-python2.7/
      5 COPY requirements.txt /home/AutoRTX-python2.7/
      6 RUN pip install --no-cache-dir -r requirements.txt
      7
      8 COPY . /home/AutoRTX-python2.7/
```

- 今回使いたいparamikoパッケージはpython2.7.Xまでサポートしてくれないので、2.7.Xのイメージを使います。[dockerHUB](https://hub.docker.com/)でpythonを検索すれば、公式版が見つけられる。
- WORKDIR はコンテナ側のディリクトリ（想定）
- RUN pip install --no-cache-dir -r requirements.txt　# requirements.txtに書かれた内容を全部pipでインストール（cacheを利用しない）
- COPY . /home/AutoRTX-python2.7/　# ホスト側今のディリクトリ内のものを全部コンテナ側の/home/AutoRTX-python2.7/ディリクトリにコピー

## requirements.txt作成
パッケージ要件を定義するファイルです。
今回は基本baseパッケージを使うほか、paramikoだけ追加インストールすれば十分ので、paramikoだけを記入

```
requirements.txt
     paramiko==2.4.1
```

バージョン2.4.1は開発時使われたものです。

>- ちなみに、開発環境の全パッケージ一覧をアウトプットできる方法もあります。
>- pip freeze > requirements.txt
>- ↑こうすれば、開発環境で使われたpipパッケージを全部requirements.txtにまとめる（docker識別可能のフォーマットので、そのまま使える。）
>- ただ、この方法でも、後程のbuildで、結構エラーになるケースが多いみたい、エラーになったら都度該当項目を削除、あるいは、必要最低限のものしか書かないとしましょう。

# docker-composeファイル準備
docker-composeバージョン確認

```
# docker-compose -v
docker-compose version 1.6.2, build 4d72027
# 
```

主に以下のように作成

```
      1 version: '2'
      2 services:
      3   AutoRTX-python2.7:
      4     image: kuritan/autortx-python2.7:0.1
      5     volumes:
      6      - './:/home/AutoRTX-python2.7/'
      7     container_name: AutoRTX-python2.7
      8     tty: true
```

- docker-compose 1.6.Xはversion2 referenceまでサポートしているので、2を指定
- 後程buildするimageを記入(想定)
- volumes：ホスト側:コンテナ側、ボリュームを組めばホスト側とコンテナ側のファイルがリアルタイムで同期できる

# イメージbuild & コンテナUP
下記コマンドを順番に実行

```
# docker build -t kuritan/autortx-python2.7:0.1 . --no-cache
# docker-compose up -d
```

-  -t --tag ：ビルド成功後、作成されたメッセージにリポジトリ名（とオプションタグ）を付与する
- docker-compose up -d　-d：デタッチド・モードで起動、デタッチド・モードのコンテナは停止しても自動的に削除できません。要はバックグラウンドで起動しっ放しになる。


最後まで成功したら、こうなります。
```
Creating AutoRTX-python2.7　
# AutoRTX-python2.7は今回の指定名
```

# 環境確認
上記のすべてをクリアすれば、理論上は準備終了ですが、動作確認として、実際コンテナの中に入って、pythonのバージョンやモジュールが正常にimportできるかを確認。

![container](https://github.com/kuritan/images/raw/master/docker.PNG)
- コンテナ稼働中確認
- pythonバージョンは2.7.XでOK
- import paramikoとエラーが出なければOK

ここまでOKであれば、毎回こちらのホストにSSHでログインしてから、下記のようにスクリプトを実行できる。

# pyスクリプト実行方法
- ホスト機にSSHログイン
- dockerコンテナを確認
 - docker ps -a
- 該当dockerコンテナに入る
 - docker exec -it XXXX bash
- スクリプトを実行
 - python XXXX.py

# ～番外編～Run.shスクリプトを用意
ただ、上記のように、毎回 docker コマンドにオプション渡すのも面倒くさいので、
かんたんな run.shを用意


```
      1 #!/bin/sh
      2
      3 CONTAINER_NAME="AutoRTX-python2.7"
      4
      5 OLD="$(docker ps --all --quiet --filter=name="$CONTAINER_NAME")"
      6 if [ -n "$OLD" ]; then
      7     docker exec -it \
      8     ${CONTAINER_NAME} "python" "$@"
      9 else
     10   echo 'No such container!' 1>&2
     11   exit 1
     12 fi
```

こうすれば、毎回ホスト側で、以下のように実行すれば、コンテナのpython環境を利用して、スクリプトを動かすことを可能に

```
#./run.sh XXXX.py
```

# 参考URL
- docker buildのオプションまとめ
https://qiita.com/Gin/items/dde3c3085f13f0a45c40

- docker-compose up したコンテナを起動させ続ける方法
https://qiita.com/sekitaka_1214/items/2af73d5dc56c6af8a167

- [Docker] 初心者が知っておくと便利かもしれない15の知識
https://qiita.com/enta0701/items/b872eef6d910908c0e6c#8-expose%E3%81%A3%E3%81%A6%E5%BF%85%E8%A6%81
