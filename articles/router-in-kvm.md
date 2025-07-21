---
title: "routerをKVMに立てる"
published: false
---

## はじめに

qemuから仮想routerをKVMの上に建てて他ゲストとホストと相互に通信することを目標とします。

## 動機

OSSを試したり構築の練習をするためにqemu+KVMで仮想マシンを建てて遊んでいます。
単純に建てるだけであればテンプレートを作成して試したいことがでてきたら
テンプレートをコピーしてサービスを入れればいいですが、
複数のシステムを同時に構築したとき動いてくれるか/動かないよなを試しながら
ホスト側セグメントに影響を出さないようにするためには
仮想マシン用のセグメントが欲しいと思い作っていました。

## 実行環境

|    構成要素    | 使用環境  |
| :------------: | :-------: |
|   ディストロ   | ArchLinux |
| ハイパーバイザ |    KVM    |
|  エミュレータ  |   qemu    |

## 構成図

準備中

## nicの準備

ホストもrouterも外部と直接通信したいが、ゲストは外部と直接通信したくはなく、ルーターを経由したいため、下記nicを準備します。
今回作成したconfigは補足に記載します。

1. ホスト/routerが使用するための外部向けnic
    1.  ブリッジを作成します。
    1.  物理nicをブリッジに接続します。
    1.  仮想nicを作成します。
1. ゲスト/routerが使用するための内部向けnic
    1. tunデバイス用のブリッジを作成します
    1. tunデバイスを作成します。

## router作成

1. routerとしたいゲストを作成します。
1. Linuxの場合、 `net.ipv4.ip_forward = 1`を設定します。

## 配下に配置したいゲスト

## まとめ

## 補足: Network config

### systemd-networkdを使う場合

下記configを `/etc/systemd/network` に配置してsystemd-networkdを再起動します。

#### 20-wired.network

```
[Match]
Name = enp*
 
[Network]
Bridge = hostbr0
```

#### 40-hostbr0.netdev

```
[NetDev]
Name=hostbr0
Kind=bridge
```

#### 42-kvm_rooter.netdev

```
[NetDev]
Name=kvm_rooter
Kind=tap
```

#### 43-kvm_mcast.netdev

```
[NetDev]
Name=kvm_mcast
Kind=tap
```

#### 50-hostbr0.network

```
[Match]
Name=hostbr0
[Network]
Address = 192.168.1.100/24
DNS = 192.168.1.1
DNS = 192.168.50.254
Gateway = 192.168.1.1
```

#### 52-kvm_rooter.network

```
[Match]
Name=kvm_rooter
[Network]
Bridge=hostbr0
```

### NetworkManagerを使う場合

準備中
