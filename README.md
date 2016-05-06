# ansible-sample

Pythonで実装された構成管理ツール    
Ansible社が開発(2015/10/16 Red Hat社が回収発表)    
SSHでホストに接続してスクリプトを実行するシンプルな形式    
ホスト側にClientが不要。    

## ファイル構成
| 名称        | 説明           |
| ------------- |:-------------:|
| Playbook      | 構築処理を定義するもの。YAML形式,単純なコマンド実行であれば無くても利用可能|
|インベントリ   | 構築対象ホストを定義するもの。INI形式|

# インベントリ サンプル

```
vm1 ansible_ssh_host=104.197.59.209  # 構築対象のホストの定義, vm1 は別名

[development] # [xx] でホストをグルーピング
vm1

[web] 
vm[1:3]       # vm1 vm2 vm3 と同意
```

# ansible コマンド
- インベントリのホストに対してタスク(モジュール)を実行するコマンド
- 以下の様なコマンドがある
 
## インベントリに定義されたホスト一覧の取得
- ansible -i インベントリ [ホストグループ] --list-hosts
```
$ ansible -i sample.ini web --list-hosts
    vm1
    vm2
    vm3
$ ansible -i sample.ini development --list-hosts
    vm1
```

## タスク実行(ansible モジュールの利用)
- リモートホストでシェル実行
-- ansible -i sample.ini [ホストグループ] -m shell -a シェルコマンド

```
$ ansible -i sample.ini development -m shell -a uname 
vm1 | success | rc=0 >>
Linux
```

-- リモートホストでyum install 

```
$ ansible -i sample.ini development -m yum -a 'name=openssl state=latest'
vm1 | success >> {
    "changed": false,
    "msg": "",
    "rc": 0,
    "results": [
        "All packages providing openssl are up to date",
        ""
    ]
}
```

## Facts
- ホスト情報の自動収集機能
- 以下で取得できる情報は, モジュールや後述するPlaybookで利用可能
```
$ ansible vm1 -i sample.ini -m setup
vm1 | success >> {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "10.128.0.2"
        ],
        "ansible_all_ipv6_addresses": [
            "fe80::4001:aff:fe80:2"
        ],
        "ansible_architecture": "x86_64",
```

-- モジュールでで 変数 (例: {{ ansible_all_ipv4_addresses[0] }}) として利用可能
---  また、確認用に debug: var=ansible_all_ipv4_addresses でFactsの変数を標準出力できます。

# Playbook
- インベントリで定義したホストに対して実行するタスクを記述するファイル
- YAML 形式


## Playbook のサンプル

```
---
- hosts: vm1          # タスクを実行するホスト, group で指定することも可
  become: true        # 実行ユーザを切り替える (task毎に上書き可)
  become_user: root   # 実行ユーザ(省略するとroot)
  become_method: sudo # ユーザの切替方法(デフォルトsudo)
  tasks:             # 実行タスクリスト(上から順に実行される)
  - name: Install Apache          # 実行タスク名
    yum: name=httpd state=latest  # yum モジュールを使って httpd を最新版までupdate
    notify: Boot Apache           # モジュールの実行結果がchangedのときにhandlersのBoot Apacheを実行
    when: "'development' in group_names" and ansible_os_family == 'RedHat' # フィルタ機能 タスクの実行条件を指定できる。
  handlers:
   - name: Boot Apache
     service: >            # モジュールの定義は改行可能 (key=value の形式で指定できる。)
      name=httpd
      state=started
      enabled=yes
      sudo: yes
```

## roles
- 
- [Best Practive][1] 参照
[1]: http://docs.ansible.com/ansible/playbooks_best_practices.html "link title Playbook Best Practices"

## Playbookの実行
 - ansible-playbook -i sample.ini sample.yml [--tags "xxx,xxx"]
 - tagsを指定すると,該当タグが設定されたタスクのみを実行します。 詳細はsample.ymlをご参照ください。
 
```
$ ansible-playbook -i sample.ini sample.yml --tags "debug"
PLAY [vm1] ********************************************************************   <= タスク開始
GATHERING FACTS ***************************************************************   <= 設定対象のサーバの情報収集 
ok: [vm1]

TASK: [Install libselinux-python] *********************************************   <= タスク実行
ok: [vm1]

TASK: [Disable SELinux] *******************************************************   <= 〃
changed: [vm1]
.....
....
PLAY RECAP ********************************************************************
vm1                        : ok=3    changed=1    unreachable=0    failed=0       <= 実行結果 
                            # taskが3つ成功, そのうちホストに変更を加えたのが1つ。ホストへ接続不可0 失敗 0
```
 

## Playbook のデバック
- syntax check

```
$ ansible-playbook -i sample.ini sample.yml --syntax-check

playbook: sample.yml

```

- 対象ホストに接続してPlaybookを実行した際にchanged(変更)になることをチェックする機能,あまり複雑なチェックはできない。

```
$ ansible-playbook -i sample.ini sample.yml --check -vvv

PLAY [vm1] ********************************************************************

GATHERING FACTS ***************************************************************
ok: [vm1]

TASK: [Install Apache] ********************************************************
changed: [vm1]

TASK: [Boot Apache] ***********************************************************
failed: [vm1] => {"failed": true}
msg: unsupported parameter for module: enable

FATAL: all hosts have already failed -- aborting

PLAY RECAP ********************************************************************
           to retry, use: --limit @/Users/keiichi/sample.retry

vm1                        : ok=2    changed=1    unreachable=0    failed=1
```
- その他 デバックに役立つ ansible-playbookコマンドのオプション
 - "-v" 詳細表示
 - "-vvv" さらに詳細表示

