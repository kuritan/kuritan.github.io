---
title: MigrateIn-houseMavenRegistryToGCPArtifactRegistry
date: 2024-10-22 15:14:18
categories: infra
tags:
- JP
- GCP
---
まさか、二年ぶりの更新になるとは思わなかったなぁー  
二転三転して、今もまたパブリッククラウド様のおかけで何とか生活を賄っているが、今回はなんと大嫌いのjavaのネタです(笑)  
社内のMavenRegistry(ぶっちゃけNexus)にホスティングしているライブラリー達を如何にGCP環境のArtifactRegistyに移行するお話をまとめました。  
<!--more-->

## TL;DR

- (VPN接続)社内のMavenRegistry(Nexus)からSpringBoot PJT用のライブラリー一式をローカルPCにダウンロード
  - `$ gradlew clean build`成功したら下記pathにライブラリ群が格納されているはず
  - `~/.gradle/caches/modules-2/files-2.1/*`
- (VPN切断)GCPへの接続やTOKEN取得を行う
  - `$ gcloud auth application-default login`
  - `$ export GOOGLE_OAUTH_ACCESS_TOKEN=$(gcloud auth application-default print-access-token)`
- 自作簡易アプリでgradlew publishをGCPのArtifactRegistryに向けて実行
- SpringBoot PJTのmaven接続先をGCP ArtifactRegistry修正
- 依存関係リフレッシュbuildを行い、build成功を確認
  - `$ gradlew clean build --refresh-dependencies`

## ゴール

社内しかアクセスできないMavenRegistry(Nexus)からライブラリー群をGCPのArtifactRegistryに移行し、VPN無しの環境（例えばGithubActions）でもjavaアプリをbuildできるようにしたい

## お作法

### 事前準備

- 利用したいjavaアプリのチェックアウト
- 社内MavenRegistry(Nexus)へのアクセス方法取得
  - VPNやNexusユーザ名PWなど
- GCP環境へのアクセス方法取得
  - ArtifactRegistry作成済み確認
  - 権限付与など確認
    - `$ gcloud auth application-default login`
    - `$ export GOOGLE_OAUTH_ACCESS_TOKEN=$(gcloud auth application-default print-access-token)`
- 自作簡易アプリのチェックアウト
  - 簡易なのでbuild.gradleだけ掲載

``` java
plugins {
    id 'maven-publish'
}

publishing {
    publications {
        // 3パターンを例に掲載
        // 1.シンプルで更に依存関係がないやつ
        profileLoader(MavenPublication) {
            // アーティファクトを定義する
            artifact file('libs/profile-loader-1.2.1.jar')
            groupId = 'com.matilde-lab.gradle'
            artifactId = 'profile-loader'
            version = '1.2.1'
        }
        // 2.更に依存関係があるやつ
        asciidoc(MavenPublication) {
            // アーティファクトを定義する
            artifact file('libs/asciidoc-2.0.2.jar')
            groupId = 'com.matilde-lab.gradle'
            artifactId = 'asciidoc'
            version = '2.0.2'
            pom {
                withXml {
                    // pomで依存関係を確認し、こちらに記載
                    def dependenciesNode = asNode().appendNode('dependencies')
                    dependenciesNode.appendNode('dependency').with {
                        appendNode('groupId', 'commons-io')
                        appendNode('artifactId', 'commons-io')
                        appendNode('version', '2.16.1')
                        appendNode('scope', 'runtime')
                    }
                    dependenciesNode.appendNode('dependency').with {
                        appendNode('groupId', 'org.apache.pdfbox')
                        appendNode('artifactId', 'pdfbox')
                        appendNode('version', '2.0.31')
                        appendNode('scope', 'runtime')
                    }
                }
            }
        }
        // 3.更に複雑な依存関係があるやつ（Exclusionsあり）
        openapiGenerator(MavenPublication) {
            // アーティファクトを定義する
            artifact file('libs/openapi-generator-4.1.2.jar')
            groupId = 'com.matilde-lab.gradle'
            artifactId = 'openapi-generator'
            version = '4.1.2'
            pom {
                withXml {
                    def dependenciesNode = asNode().appendNode('dependencies')
                    // Profile Loader Dependency
                    def profileLoaderDep = dependenciesNode.appendNode('dependency')
                    profileLoaderDep.appendNode('groupId', 'com.matilde-lab.gradle')
                    profileLoaderDep.appendNode('artifactId', 'profile-loader')
                    profileLoaderDep.appendNode('version', '1.2.1')
                    profileLoaderDep.appendNode('scope', 'runtime')
                    // Commons IO Dependency
                    def commonsIODep = dependenciesNode.appendNode('dependency')
                    commonsIODep.appendNode('groupId', 'commons-io')
                    commonsIODep.appendNode('artifactId', 'commons-io')
                    commonsIODep.appendNode('version', '2.16.1')
                    commonsIODep.appendNode('scope', 'runtime')
                    // SnakeYAML Dependency
                    def snakeYamlDep = dependenciesNode.appendNode('dependency')
                    snakeYamlDep.appendNode('groupId', 'org.yaml')
                    snakeYamlDep.appendNode('artifactId', 'snakeyaml')
                    snakeYamlDep.appendNode('version', '2.3')
                    snakeYamlDep.appendNode('scope', 'runtime')
                    // Jackson Dataformat YAML Dependency
                    def jacksonYamlDep = dependenciesNode.appendNode('dependency')
                    jacksonYamlDep.appendNode('groupId', 'com.fasterxml.jackson.dataformat')
                    jacksonYamlDep.appendNode('artifactId', 'jackson-dataformat-yaml')
                    jacksonYamlDep.appendNode('version', '2.17.2')
                    jacksonYamlDep.appendNode('scope', 'runtime')
                    // OpenAPI Generator Gradle Plugin with Exclusion
                    def openapiGeneratorDep = dependenciesNode.appendNode('dependency')
                    openapiGeneratorDep.appendNode('groupId', 'org.openapitools')
                    openapiGeneratorDep.appendNode('artifactId', 'openapi-generator-gradle-plugin')
                    openapiGeneratorDep.appendNode('version', '7.8.0')
                    openapiGeneratorDep.appendNode('scope', 'runtime')
                    // Add Exclusions
                    def exclusionsNode = openapiGeneratorDep.appendNode('exclusions')
                    def exclusionNode = exclusionsNode.appendNode('exclusion')
                    exclusionNode.appendNode('groupId', 'org.slf4j')
                    exclusionNode.appendNode('artifactId', 'slf4j-simple')
                }
            }
        }
    }

    repositories {
        maven {
            name = "artifactRegistry"
            url = "https://asia-maven.pkg.dev/{GCP PROJECT名}/{ArtifactRegistry名}"
            credentials {
                username = "oauth2accesstoken"
                password = System.getenv("GOOGLE_OAUTH_ACCESS_TOKEN")
            }
        }
    }
}
```

### ライブラリーPublish方法

- 利用したいjavaアプリのチェックアウト
- `$ ./gradlew clean build`
- build成功したら、ローカルにダウンロードされているライブラリーを確認
  - `~/.gradle/caches/modules-2/files-2.1/*`
  - 社内groupIdもし確定でしたら、下記コマンドでも一覧を出せる
    - `./gradlew buildEnvironment | grep com.{会社名？など}`
    - `./gradlew dependencies | grep com.{会社名？など}`
- ローカルにダウンロードした移行対象ライブラリーを自作簡易アプリの./lib/直下にコピー
  - jarとpom両方も必要
- 自作簡易アプリの設定を修正し、必要なライブラリー名とpathと依存関係をセット
- ArtifactRegistryへ登録
  - `$ export GOOGLE_OAUTH_ACCESS_TOKEN=$(gcloud auth application-default print-access-token)`
  - `$ ./gradlew publish`

### ゴール確認

無事publish実行できたら、該当GCPのArtifactRegistryにブラウザでアクセスすると、publishされたライブラリーを確認できる。  
利用したいjavaアプリのbuild.gradleにて、buildscript->repositories->mavenを修正し、社内MavenRegistry(Nexus)の向き先情報をGCPのArtifactRegistry情報に置き換える

※下記一例  

``` java
buildscript {
  repositories {
    maven {
      name = "artifactRegistry"
      url = "https://asia-maven.pkg.dev/{GCP PROJECT名}/{ArtifactRegistry名}"
      credentials {
          username = "oauth2accesstoken"
          password = System.getenv("GOOGLE_OAUTH_ACCESS_TOKEN")
      }
    }
    mavenCentral()
  }
}
```

## 思った事

___やっぱjava嫌いだ___
