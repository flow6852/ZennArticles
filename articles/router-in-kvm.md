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

## このドキュメントの記載方針について

ネットワーク設定については記載すると長くなってしまうため、
補足にconfigを記載し、
configの方針を本文に記載しています。

## 実行環境

|    構成要素    | 使用環境  |
| :------------: | :-------: |
|   ディストロ   | ArchLinux |
| ハイパーバイザ |    KVM    |
|  エミュレータ  |   qemu    |

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

## 参考文献


* [インストールガイド](https://wiki.archlinux.jp/index.php/%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%82%AC%E3%82%A4%E3%83%89)
* [23.17.8.8. マルチキャストトンネル](https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-manipulating_the_domain_xml-devices#sect-Network_interfaces-Multicast_tunnel)


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
Address = (address)
DNS = (address)
DNS = (address)
Gateway = (address)
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

#### bridge-hostbr0.nmconnection

```
[connection]
id=bridge-hostbr0
uuid=07c253b5-c30d-4dc6-b5e9-d95a7c050959
type=bridge
interface-name=hostbr0
timestamp=1752698683

[ethernet]

[bridge]
stp=false

[ipv4]
address1=(address)
dns=(address)
gateway=(address)
method=manual

[ipv6]
addr-gen-mode=default
method=ignore

[proxy]
```

#### bridge-slave-enp4s0.nmconnection

```
[connection]
id=bridge-slave-enp4s0
uuid=b8d93691-303d-4e1e-80c8-f8290e98e552
type=ethernet
controller=07c253b5-c30d-4dc6-b5e9-d95a7c050959
interface-name=enp4s0
port-type=bridge

[ethernet]

[bridge-port]
```

#### bridge-slave-kvm_mcast.nmconnection

```
[connection]
id=bridge-slave-kvm_mcast
uuid=c7d9fbee-6c4c-4019-8e33-80ce80fc1657
type=tun
controller=07c253b5-c30d-4dc6-b5e9-d95a7c050959
interface-name=kvm_mcast
port-type=bridge

[ethernet]

[bridge-port]
```

#### bridge-slave-kvm_rooter.nmconnection

```
[connection]
id=bridge-slave-kvm_rooter
uuid=83e8971d-974e-4c2f-9c37-5ffff0a69103
type=tun
controller=07c253b5-c30d-4dc6-b5e9-d95a7c050959
interface-name=kvm_rooter
port-type=bridge

[ethernet]

[bridge-port]
```

#### bridge-virtbr0.nmconnection

```
[connection]
id=bridge-virtbr0
uuid=5e2078f5-b571-4b3d-ab0b-e7036471ff78
type=bridge
interface-name=virtbr0

[ethernet]

[bridge]

[ipv4]
method=auto

[ipv6]
addr-gen-mode=default
method=auto

[proxy]
```

#### kvm_mcast.nmconnection

```
[connection]
id=kvm_mcast
uuid=2604088b-f6c0-4b42-8c5e-0298b5724bee
type=tun
interface-name=kvm_mcast
controller=virtbr0
port-type=bridge
timestamp=1737243648

[tun]
mode=2
owner=0

[bridge-port]
```

#### kvm_rooter.nmconnection

```
[connection]
id=kvm_rooter
uuid=f1784eb4-083c-477e-879c-4cdef482af2c
type=tun
controller=hostbr0
interface-name=kvm_rooter
port-type=bridge

[tun]
mode=2

[bridge-port]
```
