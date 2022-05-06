---
title: 忽然と、GitlabRunnerが使えなくなった
date: 2022-05-06 13:00:43
categories: infra
tags:
- JP
- aws
- docker
---
タイトルはちょっと大げさになっていますが、  
弊社で実際発生したインシデントを、今回ご紹介したいと思います。
<!--more-->

## サマリ

夜中からgitlab ciがうまく起動できず、jobがtimeoutまで詰まりつづけ、  
定時に入ったら、CI実行不能な不具合が各処から出てきた。  
状況確認＆一時対応策を講じるまで、1時間半ほどかかり、当日中に恒久対応を取りました。

## インパクト

時限式or手動によるgitlab-CIの実行が半日程度、利用不可となった

## 根本原因

CI用のベースimageがubuntuで、デフォルトでは、  
毎日osの自動updateを行うように設定されています。  
AMI からインスタンスを起動したときも同じ状況になるので、AMI が古ければ古いほど更新パッケージが多くなり、  
起動直後に負荷が高まったり、しばらく apt install できない (unattended-upgrade がロックを獲得しているので) 時間が続いたりします。  
その結果、gitlab-ciのbaseコントローラーがtimeoutまで、  
health checkが通らず、そのままシャットダウンされ、  
利用可能なgitlab-ci runnerいつまでも０のままで、誰もCIを使えない状態となりました。

## 発生原因

gitlab-ci runnerがうまく起動できず、利用可能runner数がずっと0

## 対応

- 状況確認し、障害をアナウンス
- runner baseでrunnerがうまく立ち上がらないことを確認
- logを細かく閲覧し、解決の糸口になれそうなキーワードを探す
- 立ち上げしたばかりのrunnerにログインし、プロセスを確認したところ、apt installがいつまでも終- わらないことを気づく
- 手動でプロセスをkillし、sudo apt install docker.ioすれば、無事health checkを通り、CIの- runnerとして機能されるようになった
- ↑を一時対応策とし、6台のrunnnerを確保したら、サービス復旧をアナウンス
- runnerがauto scalingと設定されており、このままだと、障害がかならず再発するので、恒久対策を模- 索
- 手かがりをゲットし、ubuntuのdaily更新が問題かも(AMIがかなり古いので)
- AMIを作り直し(aptを最新にする&daily更新を無効化)
  - /lib/systemd/system/apt-daily.timer
  - /lib/systemd/system/apt-daily-upgrade.timer
  - Persistent=true -> false
- 適用し、問題なく機能することを確認

## 検出

pagerdutyによるアラート＆社員の報告

## 教訓

### うまくいった事

CIコンテナ操作のためな手順書があって、コマンドがコピペで利用可能

### うまくいかなかった事

作業分担が難しく、結局ひとりで対応するしかなかった

### 幸運だった事

障害時間帯にメンテやインパクトの大き作業をされていなかった

### 再発防止の為にできる事

ベースAMIの再構築

### タイムライン

09:50 pagerdutyのgitlab-ci job大量詰まりアラートを気づき
10:05 CI利用不可の報告が来た
10:10 障害を全社アナウンス
11:00 SREチーム朝会で情報共有
11:30 一時対応策を実行し、システム順次復旧を確認
11:35 システム復旧を全社アナウンス
12:00 SREチームMTGで対応状況を共有
16:00 原因を究明し、恒久対策を実施

### 参考情報

[Ubuntu 18.04 で OS 起動時の apt update と unattended-upgrade を抑制する方法](https://hirose31.hatenablog.jp/entry/2020/02/19/165738)
