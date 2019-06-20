---
title: zabbixのAlertをSNS経由でMailやHangout-chatに送信
date: 2018-08-16 22:09:13
categories: infra
tags: 
 - G-suits
 - JP
 - AWS
---

# オンプレミスZabbix改造
元々は、オンプレミスのサーバでSendmail様の力を借り、アラートメールを指定のメアドたちに転送する仕組みですが、今流行りのawsでやったらどうでしょう？という上からの期待に答えたく、かつ、いつもgamilに嫌がらせメアドと判定しBANされた窮地から抜け出すことも兼ねて、SNSを使って、アラートの転送をチャレンジしましたー  
<!--more-->

- 元々の構成ですー  
![オンプレミス構成](http://wx2.sinaimg.cn/mw690/735d420agy1fubv4wtub3j213w0lw46r.jpg)
※本件の前提は、オンプレミス環境とAWS環境間で、DirectConnectを有すること

# Zabbix → SNS → Mail
![SNSで送信](http://wx3.sinaimg.cn/mw690/735d420agy1fubv4uvp7mj21400mldlc.jpg)
- zabbixサーバでaws-cliをインストールし、アカウント情報をconfigに入れる  
- zabbixのMediaTypeでsns追加
- zabbixのalertがトリガーされるたびに、SNS publishコマンドを発火
- 予め設定済みのSNSサブスクリプションで各通知先メアドへ送信

# Zabbix → SNS → Hangout-Chat
![SNSでHangout-Chatに送信](http://wx1.sinaimg.cn/mw690/735d420agy1fubv4vwezsj21400m0n4k.jpg)
- zabbixサーバでaws-cliをインストールし、アカウント情報をconfigに入れる  
- zabbixのMediaTypeでsns追加
- zabbixのalertがトリガーされるたびに、SNS publishコマンドを発火
- 予め設定済みのSNSサブスクリプションでlambda関数がトリガーされ、jsonデータを指定のhangout-chatルームに送る