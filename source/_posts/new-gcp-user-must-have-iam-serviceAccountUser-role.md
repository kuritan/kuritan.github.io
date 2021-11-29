---
title: 新規GCPユーザにiam.serviceAccountUserロールを付与してね
date: 2021-11-29 13:29:08
categories: infra
tags:
- JP
- GCP
---
最近AWSだけではなく、GCPにも触るようになっちゃったので、個人メモとしてこちらに書き込もうと思っていました。  
まずは、ユーザのIAM関連ですね。  
<!--more-->

## イシュー確認

早速ですが、同じGCPのProjectに複数のメンバーが作業するのは一般的かと思いますが、  
新しいメンバーを招待し、必要最小限な権限を付与したつもりで、なぜかVMが作られないとのご連絡が入りました。  

エラーメッセージを確認しましょうー

``` bash
Operation type [insert] failed with message "The user does not have access to service account '[PROJECT-NUMBER]-compute@developer.gserviceaccount.com'. User: '[user name]@[company domain]'. Ask a project owner to grant you the iam.serviceAccountUser role on the service account"
```

むむ......一言いうと、さっぱりわからん......  
なぜiam.serviceAccountUserのロールがここに出てくるの?おかしいくなぇ!?  

## 解決策

色々模索した結果、該当ユーザに必要最小限な権限を付与した上、さらに「サービス アカウント ユーザー」のロールを付与しないと、うまく機能されません。  
実際裏でAPIを実行する時、ユーザーアカウントではなく、サービスアカウントとしてパーミッションチェックを行いますので、サービスアカウントとして機能されるように権限を付与する必要があるようですね。  
![サービス アカウント ユーザー](gcp-iam-service-account.png)  
  
## 所感
他のサービスはともかく、IAMの運用として、AWSの方が楽な気がすげぇしますよね......  

