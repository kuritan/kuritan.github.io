---
title: "Capistranoデプロイする際にCould not parse PKey: no start lineで困った件"
date: 2021-07-08 14:15:54
categories: dev
tags:
- JP
- ruby
- CI/CD
---
最近AWSへの移設案件とかで結構忙しくなってますが、ちゃんとブログ更新しないと、  
たぶん一気にダルくなって二度と更新しないじゃないかぁと思って、カキマス！  
<!--more-->

# 何があったの?
webアプリをAWSへ移設し、configなどを弄って、改めてdeployする際に、下記のいうなssh接続用のpublic鍵がうまく読み込まれないケースと遭遇しました。  

```
$ bundle install --path vendor/bundle
$ bundle exec cap staging deploy
(Backtrace restricted to imported tasks)
cap aborted!
SSHKit::Runner::ExecuteError: Exception while executing on host <host-address>: Could not parse PKey: no start line

ArgumentError: Could not parse PKey: no start line

Tasks: TOP => deploy:starting => deploy:init_permission
(See full trace by running task with --trace)
The deploy has failed with an error: Exception while executing on host <host-address>: Could not parse PKey: no start line
```

一応、手元の環境を一通り確認したところ、特に不足な部分が見当たらなかったが、エラーは一貫として、強く自分主張をされています。  
困ったなぁ〜  

# ググったら?
そりゃやりますよ〜  
まず、こちらのページにたどり着けました。  
- [Capistranoでの配備時に`Could not parse PKey: no start line`エラーが発生した時の対処法](https://qiita.com/kadoppe/items/ed1e9c6b75cb7e676322)

むむ......
でもね  
public鍵ちゃんとありますよ〜  
なんだろう......  

ここで、githubのissueやstackoverflowもいろいろ探していたが、収穫ゼロで、路頭に迷う状態になっちゃいました。  

## まさか...お前だと!?
探してる途中で、パット見関係ない記事を見つかったが、よく読んでると、ひょっとしたらssh agentに秘密鍵を登録しないとそもそも鍵が特定されてないと気づき、トライしてみました。  
- [macOS で再起動しても ssh agent に秘密鍵を保持させ続ける二つの方法](https://qiita.com/sonots/items/a6dec06f95fca4757d4a)

結果......大正解だ！！！！！！  

```
$ ssh-add ~/.ssh/id_rsa
$ bundle exec cap staging deploy
...
......
.........
INFO [43d9f003] Finished in 0.045 seconds with exit status 0 (successful).
```

デプロイできた！！！  
該当記事に書かれた通り、macだと再起動するたびにssh-add登録が無効となるんので、助言に甘えて、セッティングをしました。

```
$ vim ~/.ssh/config
Host *
  UseKeychain yes
  AddKeysToAgent yes

$ ssh-add -K ~/.ssh/id_rsa
```

__意外とした落とし穴で、かなり焦ったが、ちゃんと経緯をここで書いて、どなたの助けになれればと祈るばかりです。__  

__ではでは__