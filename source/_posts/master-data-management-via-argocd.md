---
title: Argocdで自家製マスターデータ管理を
date: 2022-02-04 11:37:41
categories: dev
tags:
- JP
- k8s
---

ちょっと開発よりのネタをお話できればと、今回思ってましたね。  
k8s(GKE)を触りつつ、バックエンドのロジックにも関わりまして、モバイルゲームの開発として、  
やっぱ避けられないのはマスターデータの管理と思い、  
自家製mater data運用パイプラインをご紹介しましょうー
<!--more-->

## 目的

モバイルゲームにおいて、ユーザ情報、所持アイテム、イベント情報などなど、様々な情報を管理しないといけない課題がありまして、  
それらこそがmaster dataです。  
非エンジニアの方(planer, directorなど)に情報の編集をしてもらい、データ保管、共有、更新、デリバリー、ステップ・バイ・ステップで、  
本番環境に適用できるまでのパイプラインを実現したい。

## 背景

考案時点では、それ相応なライブラリーがまだ存在しておらず、ツールで担保することを決めました。  
後に、社内Golang専門のミドルウェアGが発足され、ライブラリーでも担保できるようになっていますが、  
今回はあくまでツールの話をさせてくださーい  

## 実装方法

### 構成

#### 必要ツール

- スプレッドシート
  - データ編集＆保管
- Slack channel
  - bot(別途gceにて実装)
    - 理由としては、単純にk8sではうまく起動できず、単体運用でも問題ないし、変更も少ないのでgceにて建造した
- Circle ci(別CIでもok)
- ArgoCD
  - 主役
- gitリポジトリー
  - サーバーサイドコード/master data(json)格納用
    - master dataファイルのハッシュチェック機能
    - dbとcdnへの適用機能、及びそれらのアクションを実行するdocker環境（以下でseederという）
    - スプレッドシート->json変換機能

#### フロー

![data-flow](data-flow.png)

![argocd-work-flow](argocd-work-flow.png)

##### 流れ説明

1. スプレッドシートでのmaster data情報を編集
1. botでmaster data更新を実行(gitリポジトリの最新をfetch、スプレッドシート->jsonに変換し、gitリポジトリーの指定dirへcommit)
1. botでmaster dataのpublishを実行(gitリポジトリの最新をfetch、circle-ciの指定workflowをキック)
1. circle-ciで最新seeder imageを作り、cloud storageの指定bucketの指定位置に、master dataを更新すると示すflagファイル(空ファイル)をuploadしてから、Argocdのsyncをキック
1. Argocdのresource hook機能を使い、メインリソースsyncの前に、seederをjobとして実行する(argocd.argoproj.io/hook:PreSync)
1. seeder job内で、master dataのハッシュチェック(cloud storageにて保存されたhash listと一致するか)やmaster data更新flagの存在確認(存在の場合のみ実行)を行い、最新master dataをデリバリーする(cdnへupload、dbへinsert)
1. flagファイルを削除し、パイプライン終了

##### 豆知識

API経由でCircleCIをキックする時、利用するtokenはPersonal API Tokenでないと動けない！！！  

- [Note: keep in mind that you have to use a personal API token; project tokens are currently not supported on CircleCI API (v2).](https://support.circleci.com/hc/en-us/articles/360050351292-How-to-trigger-a-workflow-via-CircleCI-API-v2)
- [Personal API Token](https://app.circleci.com/settings/user/tokens)
- [CircleCI API doc](https://circleci.com/docs/api/v2/#operation/triggerPipeline)

## 思った事

ツールだけでパイプラインを担保するには、ツールロックインのリスクがあり、  
もっといい方法がある時それらに使うことに越したことがないのだが、  
それなりに業界ではメジャーのツールであれば、そこまで心配する必要がなく、開発効率を優先した方が、  
みんなハッピーになれますねー
