---
weight: 1
bookFlatSection: true
title: "EC2でDPDK動作環境を作る"
draft: true
---

# EC2でDPDK動作環境を作る

dpdkを利用したアプリケーションを作る機会があったので，勉強用の環境を作ってみた．



## EC2の起動

- Ubuntuインスタンスの作成
  - 2コア
  - Ubuntu Server 20.04 LTS (HVM), SSD Volume Type
  - t2.large vCPU:4, Memory: 8GB
- ネットワークインタフェースの作成
- ネットワークインタフェースのアタッチ

### ネットワークインタフェースの作成
デフォルトの場合，EC2作成時にはネットワークインタフェースは1つアタッチされている．
今回はネットワークインタフェースが2つ必要なため，追加を行う．
こちらの[複数の IPv4 アドレスの使用](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/MultipleIP.html#working-with-multiple-ipv4)でEC2インスタンスにネットワークインタフェースを追加する方法があった．

## 作成したEC2環境の確認

インスタンス上でipコマンドを使用して,ネットワークインタフェースが2個あることを確認しておく．

```bash
ubuntu@ip-172-xx-xx-xxx:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc fq_codel state UP group default qlen 1000
    link/ether 06:5c:26:35:0f:a9 brd ff:ff:ff:ff:ff:ff
    inet 172.xx.xx.xxx/20 brd 172.xx.xx.255 scope global dynamic eth0
       valid_lft 2950sec preferred_lft 2950sec
    inet6 fe80::45c:26ff:fe35:fa9/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 06:03:9b:80:f5:e5 brd ff:ff:ff:ff:ff:ff
```

## DPDKのインストール

[DPDK公式ページ](http://core.dpdk.org/download/)から，最新LTSの`DPDK 20.11.3`(2021/12/19時点)をダウンロードする


## 詰まった点
- DPDKのビルドが途中で止まる
```
$ ninja -C build
[2049/2419] Compiling C object 'drivers/a715181@@tmp_rte_event_octeontx2@sta/event_octeontx2_otx2_worker.c.o'.(ここでコンパイルが止まる)
```