---
title: もう令和だから、開発環境くらいdockerに載せたらどうだい
date: 2020-06-21 19:36:19
categories: infra
tags:
- JP
- docker
---

月イチでブログを更新しようと思ったが、案外難しいすね...  
なんだいかんだい言っても、モチベション維持は一番ですから。  
くだらない話はこのへんにして、VSCodeのExtension(RemoteContainers)をお話しましょうー  
<!--more-->

# instruction

今の時点(2020/06)で、まだプレビュー段階のRemoteContainersですが、  
VSCodeにおいての開発手法に新鮮な風を吹き込んでくれることが間違いなし！  
![Remote-Containers](http://wx1.sinaimg.cn/mw690/735d420agy1gg0wbhyf3lj20ob05rdgm.jpg)

こいつのどこがいい？そりゃ環境構築の手間が省けることじゃー  
こんな話、あったことありますか？  

- 新しいプロジェクトの開発を取り込もうと思って、環境構築だけで丸一日かかった
- 新しいメンバーが入って、ドキュメントを読ませたり、環境構築をやらせたり、なんとなく一週間が過ぎた
- 手元のマシンで色んなバージョンコントローラーを入れて、別々のプロジェクトに行き渡りしたら、バージョン切り替えだけで大変

RemoteContainersを使ったら、こんなストレス全部いなくなるんだよ！  


# trial

早速、試してみましょう  

1. インストールは割愛させてくださいね（VSCodeのExtensionから検索してからのinstallだけ）
2. Command Paletteからremoteを検索し、Open Folder in Containerを選択
   ![extension](http://wx3.sinaimg.cn/mw690/735d420agy1gg0wlrkvuvj20gx02k0sw.jpg)
3. 色んなdocker imageが表示されるが、ここはぼくのブログを例にして、node.js 10をポッチと
4. ﾁｮｯﾄ処理を待ってから、.devcontainerのディレクトリが作られ、中にはDockerfileとdevcontainer.json２つファイルがあります
5. VSCodeのメニューバーから[View] -> [Terminal]をクリックし、ターミナルを開けると、今はもうcontainer環境内にいることがわかった
6. そこからもうﾁｮｯﾄアレンジが必要で、ぼくのブログはhexoを使ってるので、hexoのインストールや日本語の対応、timezoneの調整が必要ですね。それらは全部Dockerfileをいじれば簡単です。



### Dockerfile

ぼくがいじった後のモノをお見せしましょう。  

```dockerfile
#-------------------------------------------------------------------------------------------------------------
# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License. See https://go.microsoft.com/fwlink/?linkid=2090316 for license information.
#-------------------------------------------------------------------------------------------------------------

# To fully customize the contents of this image, use the following Dockerfile instead:
# https://github.com/microsoft/vscode-dev-containers/tree/v0.122.1/containers/javascript-node-10/.devcontainer/Dockerfile
FROM mcr.microsoft.com/vscode/devcontainers/javascript-node:0-10

# ** [Optional] Uncomment this section to install additional packages. **
#
# RUN apt-get update \
#     && export DEBIAN_FRONTEND=noninteractive \
#     && apt-get -y install --no-install-recommends <your-package-list-here>

# Verify git and needed tools are installed
RUN apt-get install -y git procps locales \
    && sed -i '/^#.* ja_JP.UTF-8 /s/^#//' /etc/locale.gen \
    && locale-gen \
    && ln -fs /usr/share/zoneinfo/Asia/Tokyo /etc/localtime\
    && dpkg-reconfigure -f noninteractive tzdata

# Remove outdated yarn from /opt and install via package 
# so it can be easily updated via apt-get upgrade yarn
RUN rm -rf /opt/yarn-* \
    && rm -f /usr/local/bin/yarn \
    && rm -f /usr/local/bin/yarnpkg \
    && apt-get install -y curl apt-transport-https lsb-release \
    && curl -sS https://dl.yarnpkg.com/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/pubkey.gpg | apt-key add - 2>/dev/null \
    && echo "deb https://dl.yarnpkg.com/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
    && apt-get update \
    && apt-get -y install --no-install-recommends yarn

# Install eslint
RUN npm install -g eslint \
    && npm install -g hexo-cli \
    && npm install hexo -g

# Clean up
RUN apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/* 

# Set the default shell to bash instead of sh
ENV SHELL /bin/bash
ENV LANG='ja_jp.utf8'
```



こうしたら、もう手元マシンの環境と関係なく、プロジェクトごとの開発環境をかんたんに使い分けられる。  
おまけに、gitでチーム共用すれば、誰しも同じ環境でサクサクと開発だけに集中できます。   
マシン乗り換えでも開発環境構築をもういっぺんやり直す必要がなくなった。  
これぞ神だ！   

最後に、今回の更新を書いてる時のVSCode画面キャプチャーの一枚で、締めとしましょう。  

![commit](http://wx2.sinaimg.cn/mw690/735d420agy1gg0x1mar5pj20yg0sk0zl.jpg)


__みなさんも、Remote-Containersで幸せになりましょう！__

