---
title: "routerをKVMに立てる"
published: false
---

## はじめに

qemuからLinuxで作成した仮想routerをKVMの上に建てて他ゲストとホストと相互に通信することを目標とします。

:::message alert
筆者はrouterのことをrooterと思いこんでネットワークデバイスを用意していました。
戒めのためルーターをrouterと書いています。
:::


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

## 作成手順

### nicの準備

ホストもrouterも外部と直接通信したいが、ゲストは外部と直接通信したくはなく、ルーターを経由したいため、下記nicを準備します。
今回作成したconfigは補足に記載します。

1. ホスト/routerが使用するための外部向けnic
    1.  ホスト用ブリッジを作成します。
    1.  物理nicをブリッジに接続します。
    1.  仮想nicを作成します。
1. ゲスト/routerが使用するための内部向けnic
    1. ゲスト用ブリッジを作成します。
    1. tunデバイス用のブリッジを作成します
    1. tunデバイスを作成します。

### router作成

routerとしたいゲストを下記コマンドにて起動し作成します。Linuxの場合、 `net.ipv4.ip_forward = 1`を設定します。
リソースやOVMFのファイルはお好みで変更してください。

```
qemu-system-x86_64 \
    -boot menu=on ${iso}
    -drive file=disks/rooter/rooter.img,if=ide,format=qcow2 \
    -enable-kvm -machine q35,accel=kvm  -cpu host,hv_vendor_id=whatever,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time,kvm=off \
    -smp cpus=4,cores=2,threads=2 -m 4G \
    -drive if=pflash,format=raw,readonly=on,file=disks/rooter/rooter_OVMF_CODE.fd \
    -drive if=pflash,format=raw,file=disks/rooter/rooter_OVMF_VARS.fd \
    -net nic,model=virtio -net tap,ifname=kvm_rooter,script=no,downscript=no
```

### router起動

routerとしたいゲストを下記コマンドにて起動し作成します。
```
qemu-system-x86_64 \
    -drive file=disks/rooter/rooter.img,if=ide,format=qcow2 \
    -enable-kvm -machine q35,accel=kvm \
    -cpu host,hv_vendor_id=whatever,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time,kvm=off \
    -smp cpus=4,cores=2,threads=2 -m 4G \
    -drive if=pflash,format=raw,readonly=on,file=disks/rooter/rooter_OVMF_CODE.fd \
    -drive if=pflash,format=raw,file=disks/rooter/rooter_OVMF_VARS.fd \
    -net nic,model=virtio -net tap,ifname=kvm_rooter,script=no,downscript=no \
    -device virtio-net,netdev=kvm_mcast -netdev socket,id=kvm_mcast,mcast=239.0.0.1:1234
```

### routing設定

下記コマンドにてrouterとしたいゲストにrouting tableを設定します。
```
ip route add default via 外側 dev 外側nic
ip route add default via 内側 dev 内側nic
```

### ゲスト作成/起動

ゲストマシンの仮想nicとして `kvm_mcast`を指定して作成/起動し、ゲストマシンで

```
qemu-system-x86_64 \
    -drive file=disks/manage/manage.img,if=ide,format=qcow2 \
    -enable-kvm -machine q35,accel=kvm \
    -cpu host,hv_vendor_id=whatever,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time,kvm=off \
    -smp cpus=4,cores=2,threads=2 -m 4G \
    -drive if=pflash,format=raw,readonly=on,file=disks/manage/manage_OVMF_CODE.fd \
    -drive if=pflash,format=raw,file=disks/manage/manage_OVMF_VARS.fd \
    -device virtio-net,netdev=kvm_mcast -netdev socket,id=kvm_mcast,mcast=239.0.0.1:1234
```

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

#### 41-virtbr0.netdev

```
[NetDev]
Name=virtbr0
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

#### 53-kvm_mcast

```
[Match]
Name=kvm_mcast
[Network]
Bridge=virtbr0
DHCP=ipv4
```
### NetworkManagerを使う場合

下記ファイルが生成されるようコマンドを実行するか、 `/etc/NetworkManager/system-connections` に配置してください。

