---
title: 俺のblogはこうやってデプロイ
date: 2018-09-10 20:41:36
categories: infra
tags:
- github
- JP
- CI&CD
---

# Just One Word
一言申し上げますと、本ブログは、GitHub + HEXOで構成され、travis-CIでCI＆CDを実行される仕組みである。
<!--more-->

# 諸々説明
![Gitpages+HEXO](http://wx4.sinaimg.cn/mw690/735d420agy1fv46ngs9xoj20hs07sdgp.jpg)

- [HEXO](https://hexo.io/)は、静的サイトジェネレーターの一つで、かなりシンプルで、今回採用しました。
- [Github pages](https://pages.github.com/)は、githubの一つの機能で、これを利用すると、静的サイトをgithubドメインで公開可能になります。
- [Travis-ci](https://travis-ci.org/)は、GitHubと連携できる継続的インテグレーションツールのひとつである。

# このブログの仕組み
![deploy流れ](http://wx2.sinaimg.cn/mw690/735d420agy1fv46o47v7qj20sx0kcjud.jpg)

1. githubでrepository新規作成し、branch追加する(pages-branch)。settingでsourceをpages-branchにすることで、同じrepositoryでソースコードも格納できるし、gitpagesも閲覧できるように実現。
2. hexoのインストール＆configure調整。こちらに関しては、割愛させて頂きます。詳細はgoogle先生に聞いてください。
3. hexoの_config.ymlで、「deploy」項目があって、それを1で作成したモノに修正する。
4. travis-ciでCI＆CD。

# travis-ciでCI＆CD
- [Travis-ci](https://travis-ci.org/)のアカウントを登録しログインする。まぁ、気軽くにgithubのアカウントでもログイン可能ですね。
- 先ほど新規作成したrepositoryとtravis-ciと紐づけられるように、スィッチをONにする。
- [Personal access token](https://github.com/settings/tokens)で、Generate new tokenを押し、新規のtokenを取得します。
※select scropesはrepoとuserにする
- Generate tokenを押すと、tokenが表示されます。必ずコピペしてください。（このページから離したら、もう二度と見えないので）
- Travic-ciに戻り、先ほど紐づけれたrepoで「more options」⇒[setting]⇒[environment variables]新規追加し、valueは先ほどのgithub token、nameはご自由に
- ローカルに戻り、hexoフォルダで、「.travis.yml」を追加してください。内容は下記通り

```
language: node_js  
node_js:  
 - "6"  
before_script:  
 - npm install hexo-cli -g  
 -   
script:  
 - hexo generate  
  
deploy:  
 provider: pages  
 local_dir: public  
 repo: xxxx/oooo.github.io  
 skip_cleanup: true  
 github_token: $XXX #ご自分設定のname 
 on:  
  branch: master  #ソースコードbranch名
  target_branch: pages-branch #pages-branch名
 ```

いかがでしょう、もしあなたのブログ作成にお役に立てると、嬉しい限りですー