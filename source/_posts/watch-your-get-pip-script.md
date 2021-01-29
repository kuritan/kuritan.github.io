---
title: python2のサポートが切れたget-pipスクリプト
date: 2021-01-29 11:29:40
categories: infra
tags:
- JP
- aws
- python
---
超〜久しぶりの更新となります。  
今回は業務上で実際にあっていた、pythonのパッケージ管理システムのお話で花を咲かせようかと考えていますので、しばしお付き合いくださいね〜  
<!--more-->
# 何があったの？
先日、弊社多数サービスの本番環境において、スケールアウトしたEC2インスタンスがunhealthy状態で、ユーザからログインできない声が出てきました。  
休日でありながら、幸いコロナの流行りで、行くところもないし、家に引きこもり中の自分は、SREとしてトラブルシューティング開始しました。  

## 問題点
- LBから見ると殆どunhealthy状態(1~2台除く)
- healthyになっていたインスタンスは当日立ち上げのインスタンスではない
- 当日スケールアウトされたEC2インスタンスはデプロイされてなかった
- 内製プロビジョニングツールのログをチェックすると、プロビジョニングがエラーで中断
- エラー内容はpip installまわり（invalid syntax）
- 実機でpip listを打っても同じinvalid syntaxのエラー

ここまで確認したら、問題はだいぶ絞られましたね。  
むむ...pipね...お恥ずかしいのですが、うちの内製プロビジョニングツールはpython2.7依存(原因は後述)で、何か臭うなぁ〜  

エラーメッセージの一部を載せますが、ここで、ヒントとなっていたのは
__f-string__ だ！  
```
    :stderr: Traceback (most recent call last):
      File "/usr/bin/pip2", line 7, in <module>
        from pip._internal.cli.main import main
      File "/usr/lib/python2.7/site-packages/pip/_internal/cli/main.py", line 8, in <module>
        from pip._internal.cli.autocompletion import autocomplete
      File "/usr/lib/python2.7/site-packages/pip/_internal/cli/autocompletion.py", line 9, in <module>
        from pip._internal.cli.main_parser import create_main_parser
      File "/usr/lib/python2.7/site-packages/pip/_internal/cli/main_parser.py", line 86
        msg = [f'unknown command "{cmd_name}"']
                                             ^
    SyntaxError: invalid syntax
```

__f-string__
だからで何が？さきほども申し上げましたが、弊社の内製プロビジョニングツールはpython2.7依存ですが、__f-string__
というpythonのfeatureは3.6以後にリリースされたもので、ここで
__f-string__
のinvalid syntaxが出てくるのは極めて不自然なことが明白である。  

ここで、疑惑の目がpip自体に向けられました。  
しばらくgoogleセンセイに聞いてみたら、こんな記事に辿り着けました。  
[Announcement: pip 21.0 has now been released](https://mail.python.org/archives/list/python-announce-list@python.org/thread/XIPLSUI3IID33J5BWSOOPLE7IQ775HLP/)  
![Announcement](pip.jpg)
なに？なに？  

ハイライトとしては2点だけ
- Removal of Python 2.7 and 3.5 support.
- Dropped support for legacy cache entries from pip < 20.0.

つまり、python2.7と3.5のサポートが終了し、レガシーcacheのサポートも終了  

これだ！とひらめいた自分でした。  
該当記事をよく読んでみたら、下記記述があって、該当リンクでget-pip.pyスクリプトを入手し、無事問題解決に至りました。
> A Python 2.7 compatible version of get-pip.py is available at https://bootstrap.pypa.io/2.7/.

# 根本原因
インシデント前日で、最新のget-pip.pyがpython2.7との互換性が失い、  
インシデント当日スケールアウトしたEC2インスタンスが、内製プロビジョニングツールの駆使下、pip経由でawscliなどのツールをインストールしようとした際、見事エラーとなり、  
その後に続くデプロイも失敗し、  
最終的に、サービスログイン不能な状態に落ちた。

# 教訓
- python2.7そろそろやめようぜ
- SREのon-call制度はそこそこ意味があるように思っていた

# 言い訳
python3にしたいよ〜したいけど、  
版元様の指定td-agentプラグイン(お金まわりのログ転送)はpython2.7依存で、こちらからは何もできないもん〜