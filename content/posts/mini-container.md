---
weight: 1
bookFlatSection: true
title: "Linux namespaces+cgroupによる簡易コンテナの自作"
---


# Linux namespaces+cgroupによる簡易コンテナの自作
最近，仮想化技術 (VMM, NFV, コンテナ等) について勉強をはじめました．
仮想化に関連する技術の中でも，普段から開発，実験でよく利用しているDocker (コンテナ型仮想化) について，理解を深めるために，簡易なコンテナの実装を行いました．
今回実装する上で，参考にした記事がこちらです．

[「【Go言語】自作コンテナ沼。スクラッチでミニDockerを作ろう - カミナシ開発者ブログ」](https://kaminashi-developer.hatenablog.jp/entry/dive-into-swamp-container-scratch)


コンテナを実現するために使用される基礎技術のみに絞って実装，解説を行なっており，コンテナ技術の本質部分を理解するための非常に良いチュートリアルだと感じました．
これをきっかけに自作コンテナ沼にハマっていきたいと思います．

## Githubリポジトリ

Golangで実装したコンテナ作成コマンドはこちらのGithubリポジトリで公開しています．

https://github.com/takashimakazuki/mini-container

## 実装した機能
簡易コンテナということで，実装した機能は非常に単純です．
コンテナを作成し，作成したコンテナ内で指定したコマンドを実行するというものになります．
プログラム内でやっていることは以下です．

- Linux namespace機能を用いた実行環境の隔離。つまりコンテナ化を行う。
- chrootによってファイルシステムを分離
- cgroupを使ってコンテナが使える計算リソースを制限する
- コンテナプロセス内でのコマンド実行

