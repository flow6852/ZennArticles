---
title: "AnsibleNetbox間連携と仮想マシンクローンの例"
published: true
type: tech
---

## はじめに


下記の続きです。

https://zenn.dev/flow6852/articles/router-in-kvm

自宅PCで仮想マシンをつくって遊ぶことをよくしています。
IPアドレス管理仮想マシンを一杯つくりたいなぁと考えていたところ、
NetboxというIPAMサービスをみつけました。

https://github.com/netbox-community/netbox
https://qiita.com/k-maki/items/e30e312dcbc448ebafeb

Ansibleと連携できるらしいので連携してみました。
結果として、NetboxにVM 名とIPアドレスを登録すると、AnsibleがKVM 上へVMを自動作成し、
LinuxやWindows の初期設定まで実施できる環境を作りました。

## 目標

Netboxで作成した仮想マシンとIPアドレスから
指定された仮想マシンを特定のテンプレートからクローンすることが目標です。
LinuxならArchLinuxテンプレートから、WindowsならWindows Server 2025から展開するようにしています。

## 環境

- Hypervisor: KVM/libvirt
- 仮想化管理: virsh
- 自動化: Ansible
- IPAM/DCIM: Netbox

## 完成イメージ

今回は下記手順で仮想マシンを作ることを想定して環境を作成しました。

1. Netboxで作成したい仮想マシンの情報を入力します。
2. 入力した仮想マシンにロールを付与します。ここではLinuxとWindowsとしています。
3. 仮想マシンに割り当てる仮想nicを作成します。
4. IPアドレスを仮想nicに割り当てます。
5. 管理用仮想マシンへ移動します。
6. playbookを実行します。

#### 1. Netboxで作成したい仮想マシンの情報を入力する。

「デバイス」に登録も可能ですが、
私は「仮想化」>「仮想マシン」>「仮想マシン」に登録しました。

#### 2. 入力した仮想マシンにロールを付与します。

私はLinuxとWindowsに分けました。
今後はセグメントごとだったりADごとだったりに分けようかなとおもっています。

#### 3. 仮想マシンに割り当てる仮想nicを作成します。

「仮想化」>「仮想マシン」>「インタフェース」に2.の仮想マシン用NICを作成します。

#### 4. IPアドレスを仮想nicに割り当てます。

「IPAM」>「IPアドレス」>「IPアドレス」でIPアドレスを設定します。

:::message alert
Ansibleで最初に登録したいIPアドレスはプライマリIPアドレスのチェックボックスを有効にしてください。
これを設定しておかないと、Ansible inventory から `ansible_host` が取得できません。
:::


#### 5. 管理用仮想マシンへ移動する。

Ansible を実行する管理用仮想マシンへログインします。

#### 6. playbookを実行する。

```
cd ~/ansible/virtmachine_generate
ansible-playbook site.yml
```

## 完成イメージのための設計方針

- Netbox を Source of Truth とします。
- inventory は Netbox plugin から生成します
- host の再生成(`add_host`)は行わず、`group_by` で動的分類します。
- libvirt 操作は `delegate_to` で hypervisor 上から行う

## 動作概要

1. Netboxから登録された仮想マシン一覧を取得します。`device_roles_virtmachine`に登録されます。
2. 登録された全ての`device_roles_virtmachine`の仮想マシンに対してpingを打って未到達なものを`unreachable_hosts`に登録します。
3. 登録された全ての`unreachable_hosts`の仮想マシンが存在しないことを調べます。存在しないものをOS毎に登録します。Linuxなら`targets_linux`、Windowsなら`targets_windows`に登録します。
4. `targets_linux`ならlinuxのtemplateから、`targets_windows`ならWindowsのテンプレートからクローンし、CPUとメモリを変更します。必要なら仮想ディスクを追加します。
5. IPアドレスを変更します。
6. その他設定変更が必要なものを設定します。

## 実装

### tree

```
.
├── ansible.cfg
├── check.yml
├── group_vars
│   ├── all.yml
│   └── windows_template.yml
├── inventory
│   ├── device_roles_windows.yml
│   ├── grouping.yml
│   ├── libvirt_host.yml
│   ├── netbox.yml
│   └── zabbix.yml
├── playbooks
│   ├── create_virtmachines.yml
│   ├── machine_existance_check.yml
│   ├── machine_ip_change.yml
│   ├── machine_settings.yml
│   ├── ping_check.yml
│   ├── setting_virtmachines.yml
│   ├── template_init_windows.yml
│   └── windows_add_domain.yml
├── site.yml
├── ssh_keys
│   ├── ansible_keys
│   ├── ansible_keys.pub
│   └── id_ed25519.pub
└── templates
    └── inner.network
```

### netbox.yml

netboxのAPIを叩くところです。

```
plugin: netbox.netbox.nb_inventory
api_endpoint: https://ipaddr
token: APIKey

group_by:
    - device_roles

fetch_interfaces: true
fetch_device_roles: true
fetch_platforms: true
config_context: true
virtual_disks: true

inventory:
  plugin: netbox.netbox.nb_inventory
```

### grouping.yml

グループの階層構造を定義します。
各グループに`group_vars`を定義でき、配下グループでも`group_vars`その利用できます。
例えば`device_roles_virtmachine`に`test = 1`としたとき、`device_roles_windows`でも`device_roles_linux`でも使用できます。
逆に`device_roles_linux`に`test = 1`としたとき、`device_roles_windows`では使用できません。

```
all:
  children:
    device_roles_virtmachine:
      children:
        device_roles_windows:
        device_roles_Linux:

    hypervisors:
      children:
        device_roles_virtmachinehost:

    unreachable_hosts:
      children:
        targets:
          children:
            targets_linux:
            targets_windows:
```

### sites.yml

bootstrapです。

```
---
- import_playbook: playbooks/ping_check.yml
- import_playbook: playbooks/machine_existance_check.yml
- import_playbook: playbooks/template_init_windows.yml
- import_playbook: playbooks/create_virtmachines.yml
- import_playbook: playbooks/setting_virtmachines.yml
- import_playbook: playbooks/machine_ip_change.yml
- import_playbook: playbooks/machine_settings.yml
- import_playbook: playbooks/windows_add_domain.yml
```

### machine_existance_check.yml

仮想マシンの存在確認をします。`delegate_to`で`kvm-host`を指定することで`inventory_hostname`を対象として維持しながらkvm-host上でコマンドを実行してくれます。

```
---
- name: Create Host Sites
  hosts: unreachable_hosts
  gather_facts: no

  tasks:
    - name: Check exists VM
      delegate_to: kvm-host
      command: sudo virsh dominfo {{ inventory_hostname }}
      register: dominfo
      failed_when: false
      changed_when: false

    - name: Linux Collect undefined VMs
      group_by:
        key: "targets_linux"
      when: 
        - dominfo.rc != 0
        - "'Linux' in device_roles"
      
    - name: Windows Collect undefined VMs
      group_by:
        key: "targets_windows"
      when: 
        - dominfo.rc != 0
        - "'windows' in device_roles"

- name: create target
  hosts: localhost
  gather_facts: no
 
  tasks:
    - name: debug
      failed_when: false
      changed_when: false
      debug:
        msg: "{{groups['targets']}}"
```

### create_virtmachines.yml 

仮想マシンをクローンします。

```
---
- name: Get VM info from Netbox and use as Ansible variables
  hosts:  kvm-host
  gather_facts: no
  ignore_unreachable: yes

  tasks:
    - name: create dirs
      command: sudo mkdir /var/system/disks/{{item}}
      args: 
        creates: /var/system/disks/{{item}}
      loop: "{{ groups['targets']}}"
      
    - name: Clone Linux VirtMachine
      command: >
        sudo virt-clone
        --original linux-template
        --name {{ item }}
        --file /var/system/disks/{{item}}/{{item}}.qcow2
      args:
        creates: /var/system/disks/{{item}}/{{item}}.qcow2
      loop: "{{ groups['targets_linux']}}"

    - name: Clone Windows VirtMachine
      command: >
        sudo virt-clone
        --original WindowsRDS-Template
        --name {{ item }}
        --file /var/system/disks/{{item}}/{{item}}.qcow2
      args:
        creates: /var/system/disks/{{item}}/{{item}}.qcow2
      loop: "{{ groups['targets_windows']}}"
```

### machine_ip_change.yml

仮想マシンのIPアドレスを変更します。
`serial:1`を指定することでIPアドレスの衝突を回避しながら仮想マシンのIPアドレスを順次変更できます。
また、Windows 側は WinRM を利用しています。
NTLMを利用しています。

```
---
- name: Machine initialize
  hosts: targets
  serial: 1
  gather_facts: no
  ignore_unreachable: yes

  tasks:
    - name: Boot virtmachine
      delegate_to: kvm-host
      command: sudo virsh start {{ inventory_hostname }}
      when: dominfo.rc == 0

    - name: Linux test connection
      delegate_to: linux-template
      wait_for_connection:
        timeout: 180
        delay: 30
      when: "'targets_linux' in group_names"

... 省略

    - name: 固定IP設定
      delegate_to: WindowsRDS-Template
      ansible.windows.win_shell: |
        New-NetIPAddress `
          -InterfaceAlias "イーサネット 2" `
          -IPAddress "{{ hostvars[inventory_hostname].primary_ip4 }}" `
          -PrefixLength "{{ prefix | default(24) }}"
      when: "'targets_windows' in group_names"

... 省略

    - name: Linux test connection
      wait_for_connection:
        timeout: 180
        delay: 10
      when: "'targets_linux' in group_names"

    - name: Windows test connection
      wait_for_connection:
        timeout: 180
        delay: 10
      when: "'targets_windows' in group_names"
```

## 感想

Ansibleの知識がまったくなかったため、実装が思ったより大変でした。
group設計がいまだに綺麗かは不明です。
また、当初は`add_host` を利用していましたが、
`runtime inventory`を再生成してしまい、
Netboxの`inventory plugin`が生成した`ansible_host`が消えたりあまりきれいにならないとおもいました。
そのため、現在は `group_by` を利用して
既存 host を動的 group へ分類する方式にしています。

## まとめ

Netbox を Source of Truth とすることで、
VM 名・IP アドレス・ロール情報を一元管理できるようになりました。

また、`group_by` を利用することで、Netboxのinventory plugin と衝突せずに
Linux / Windows を動的分類できるようになりました。
