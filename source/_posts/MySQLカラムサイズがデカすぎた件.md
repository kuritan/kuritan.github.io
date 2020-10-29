---
title: MySQLカラムサイズがデカすぎた件
date: 2019-12-05 15:32:01
categories: infra
tags:
- JP
- database
- rails
- ruby
---
古いrailsアプリをrancher(kubernetes)でデプロイしようと、頑張ってコンテナ化の改造を何日やって参りました。  
なぜかmysql周りで手こずったので、対処方法をここでメモします。  
同じようなエラーで脳細胞を酷使する人がないよう、お祈り申し上げます。
<!--more-->

# APP周りdependencies

```shell
ruby v2.2.x
    bundler

mysql v5.x

redis v3.x

elasticsearch v1.x
    kuromoji analysis plugin

nodejs 0.10.x

npm

mecab

libreoffice

poppler
```
......すごっ！いっぱいー

# 早速問題だ
コンテナ化ですから、コンテナ環境でアプリを立ち上げようぜー  
......  
......  
......  
まぁ、いっぱいやりました～（雑すぎ）  

重要なのは、DB周りだー 

## Unsupport utf
```
rake db:createしようとしたら、エラー！
```
![UTF_unsupport](http://ae02.alicdn.com/kf/U5d9c612db09c4957a10160c3a04481a77.png)
utf charsetサポートされていないって、そんなバカな！

ちょっとconfig/database.ymlをいじってみよう。
```
default: &default
  adapter: mysql2
  encoding: utf8mb4                 #utfから変更した
  charset: utf8mb4                  #追加した
  collation: utf8mb4_general_ci     #追加した
  pool: 5
  username: root
  password:
  socket: /var/run/mysqld/mysqld.sock

production:
  <<: *default
  database: app_production
  host: <%= ENV['RDS_ENDPOINT'] %>
  username: root
  password: <%= ENV['DATABASE_PASSWORD'] %>
```
utf -> utf8mb4に変更したらどうだ！喰らえー  
まぁ、無事DBが作られた。セーフー  
続いて〜db:migrate

## column size too large
またエラーだ！！！  
今回のエラーメッセージがめちゃくちゃ長い！  
![column_size_too_large](http://ae02.alicdn.com/kf/Ud187ec87db5a4e45be26d91333f0cbafW.png)

......むむ......  
どういう事だ......さっぱりわからん。  
要は、utf8mb4だと、カラムのバイト数オーバーでしょう。  

### 原因
[参考情報源](https://qiita.com/xhnagata/items/4d5c3333cbae53888f37)  
>ActiveRecordのstring型カラムがvarchar(255)で定義されるので、utf8mb4ではインデックスのキープレフィックスが767byteを超えてしまう。

>MySQL5.7未満では、テーブル作成時にROW_FORMAT=DYNAMICを渡してやらなければならない。

>Rails4だとモンキーパッチを使って、テーブル作成時のオプションにROW_FORMAT=DYNAMICを追加してやる

じゃ、手を打うー  
database(mysql)の設定を見直し、767超えても大丈夫なように、ファイルフォーマットをAntelopeからBarracudaに変更。  
AWS RDSを使っているので、パラメータグループで、「innodb_file_format」を「Barracuda」を指定する。  

また、config/initializers/ar_innodb_row_format.rbを追加し、デフォルトのテーブル作成時の
ROW_FORMATをCOMPACT -> DYNAMICに指定変更
```
module InnodbRowFormat
  def create_table(table_name, options = {})
    table_options = options.merge(options: 'ENGINE=InnoDB ROW_FORMAT=DYNAMIC')
    super(table_name, table_options) do |td|
      yield td if block_given?
    end
  end
end

ActiveSupport.on_load :active_record do
  module ActiveRecord::ConnectionAdapters
    class AbstractMysqlAdapter
      prepend InnodbRowFormat
    end
  end
end
```

これでイケるはず！


## retry!
直前失敗したので、rake db:migrate:resetにする。

![migrate](http://ae04.alicdn.com/kf/U3aa7ae7f1fc3433fb9492c7536098e9ds.png)

やった！！！！！作られた！！！！

# それで？
引き続き、コンテナ改造続行します。  
何か別のネタがあたっら、またここでシェアしますんで、ご期待のないようにお待ち下さいw