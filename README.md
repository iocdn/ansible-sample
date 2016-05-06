# ansible-sample

ansibleはPythonで実装された構成管理ツール    
Ansible社が開発(2015/10/16 Red Hat社が回収発表)    
SSHでホストに接続してスクリプトを実行するシンプルな形式でホスト側にClientが不要。    
本 ansible-sampleではansibleの基本的な使い方を説明しています。

# 本サンプルについて
- 本サンプルのファイル郡を参照することでansibleのPlaybookに関する基本的な使い方を一通り把握できるようになっています(多分）
- 本サンプルの実行方法
 - ローカルホストにansibleをインストールしてください。
```
# mac
 # brew install ansible

# CentOS (EPEL repoのインストールが必要です)
 # yum install ansible

# windows
 TODO
```
 - セットアップを行う適当なホストを用意してください。 (CentOS 6.x を想定しています。)
 - 該当ホストにsshで公開鍵認証でログイン出来るようにしsample.ini のvm1 を修正してください
```
 vm1 ansible_host=ホストのIP　or ドメイン 
 or 
 vm1 ansible_host=ホストIP or ドメイン ansible_user=ログインユーザ ansible_private_key=秘密鍵のパス
```
 - 実行1
```
 $ cd ansible-sample
 $ ansible-playbook -i sample.ini sample.yml -l development
 Vault password: 秘密
``` 
  - 次の処理をします。(用語等は 下記で説明します。)
   - Factで収集したホストのipv4アドレスの1つめをデバック出力
   - osが RedHat系の場合はEcho出力
   - group_vers/development(暗号化されています)に定義した変数arg1の値をデバック出力

- 実行2
 
```
  $ cd ansible-sample
  $ ansible-playbook -i sample.ini sample.yml -l vm1
  Vault password: 秘密
```
  - 次の処理をします。(用語等は 下記で説明します。)
   - rolesで定義されたPlaybookを実行します。
    - elasticsearch , kibana, td-agentをインストールします。
   - apacheのインストールをします。
  - 正常に処理が完了したらapache にアクセスしてみましょう
```
 $ curl http://ホストのIP or ドメイン/
``` 
  - アクセスログが fluentd 経由で elasticsearchに格納され, kibanaで可視化されています。
  - ブラウザからkibanaにアクセスしてみましょう 
```
  ブラウザで次を実行
  $ curl http://ホストのIP or ドメイン:5601/
   * kibanaは初回ログイン時にindexを作る必要があります。
     topページでindexの作成を促されますので、そのまま確定ボタンを押下してください。
     次に,Discoverタグを開いてください。アクセス数のCountグラフが表示されます。
```

# 以下でansibleの基本的な使い方を解説します。 

## ansibleのファイル構成
| 名称        | 説明           |参考URL|
| ------------- |:-------------:|:------------|
| Playbook      | 構築処理を定義するもの。YAML形式,単純なコマンド実行であれば無くても利用可能|[Playbooks][1]|
| インベントリ  | 構築対象ホストを定義するもの。INI形式|[Inventory][2]|
| ansible.cfg   | ansibleの実行を制御する設定ファイル|[Configuration file][3]|

## インベントリ サンプル

```
vm1 ansible_ssh_host=104.197.59.209  # 構築対象のホストの定義, vm1 は別名

[development] # [xx] でホストをグルーピング
vm1

[web] 
vm[1:3]       # vm1 vm2 vm3 と同意
```

## インベントリのホストに対してタスク(モジュール)を実行するコマンド
- [Ad-Hoc Command][4] 参照
- 以下の様なコマンドがある

- インベントリに定義されたホスト一覧の取得
 - _書式:_  ansible -i インベントリ [ホストグループ] --list-hosts
```
$ ansible -i sample.ini web --list-hosts
    vm1
    vm2
    vm3
$ ansible -i sample.ini development --list-hosts
    vm1
```

 - タスク実行(ansible モジュールの利用)
  - リモートホストでシェル実行
   - _書式:_  ansible -i インベントリ [ホストグループ] -m shell -a シェルコマンド

```
$ ansible -i sample.ini development -m shell -a uname 
vm1 | success | rc=0 >>
Linux
```

  - リモートホストでyum install 
   - _書式:_  ansible -i インベントリ [ホストグループ] -m yum -a "name=パッケージ名 state=バージョンなど"
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
- [Information discovered from systems: Facts][5] 参照
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

 - モジュールで 変数 (例: {{ ansible_all_ipv4_addresses[0] }}) として利用可能
  -  また、確認用に debug: var=ansible_all_ipv4_addresses でFactsの変数を標準出力できます。

## Playbook
- インベントリで定義したホストに対して実行するタスクを記述するファイル
- YAML 形式

### Playbook のサンプル

```
---
- hosts: vm1          # タスクを実行するホスト, group で指定することも可
  become: true        # 実行ユーザを切り替える (task毎に上書き可)
  become_user: root   # 実行ユーザ(省略するとroot)
  become_method: sudo # ユーザの切替方法(デフォルトsudo)
    roles:             # roleの実行(roleについては後述)    
     - elasticsearch 
     - kibana 
     - td-agent 
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

### roles
- 詳細は[Best Practive][8] 参照
- カレントディレクトリ直下のrolesディレクトリに,以下の構成でPlaybookを配置することで、Playbookを分割管理できる。

```
roles/
└── elasticsearch
   ├── defaults       # 変数のデフォルト値
   │   └── main.yaml
   ├── files          # 設定ファイル置き場
   │   └── etc
   │       └── yum.repos.d
   │           └── elasticsearch.repo
   ├── handlers      # ハンドラ(taskからnotifyで呼ばれるタスク)
   │   └── main.yaml
   ├── meta          # メタデータ(依存ロールなどを記載するらしい)
   │   └── main.yaml
   ├── tasks         # タスク
   │   └── main.yaml
   ├── templates     # テンプレート置き場
   │   └── etc
   │       └── httpd
   │           └── httpd.conf.j2
   └── vars          # 変数
       └── main.yaml
```

## Playbookの実行
 - ansible-playbook -i sample.ini sample.yml [--tags "xxx,xxx"] [-l SUBSET]
 - 詳細は ansible-playbook -h 参照 
 - tagsを指定すると,該当タグが設定されたタスクのみを実行します。 詳細はsample.ymlをご参照ください。
 - l オプションは実行するplaybookのhostを指定する。
 
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

# その他
## 変数ファイルの暗号化
- ansible-vaultコマンドで暗号化する。
 - インベントリは無効
- 以下のコマンドで暗号/復号化を行う。

```
 $ ansible-vault encrypt ファイル名  # 暗号化, ファイル名はワイルドカード使用可
 Vault password:

 $ ansible-vault decrypt ファイル名 # 復号化
 Vault password:
 
```

- 暗号化したまま実行する場合は --ask-vault-passを付けてPlaybookを実行する。
- "--vault-password-file パスワードファイル" でパスワード入力プロンプトの表示無しで利用できる。

## サードパーティのroleを使う
- ansible-galaraxy コマンドで取得できる
- 詳細は[Ansible Galaxy][6] 参照
```
$ ansible-galaxy install bennojoy.mysql
$ ls roles
 bennojoy.mysql
```

## Ansible2 の変数の優先順位(上から順に優先)

1. ansible-playbookコマンドの -e オプションで指定した変数
2. task内のvarsの定義
3. include時に定義した変数
4. ブロック内のvarsによる定義? 
5. vars_fileによる定義
6. vars による定義
7. Facts
8. host_vars(ディレクトリ)による定義
9. host_vars(インベントリ内)による定義
10. group_vars(ディレクトリ)による定義
11. group_vars(インベントリ内)による定義
12. roleのdefaultsでの定義

[1]: http://docs.ansible.com/ansible/playbooks.html
[2]: http://docs.ansible.com/ansible/intro_inventory.html
[3]: http://docs.ansible.com/ansible/intro_configuration.html
[4]: http://docs.ansible.com/ansible/intro_adhoc.html
[5]: http://docs.ansible.com/ansible/playbooks_variables.html#information-discovered-from-systems-facts
[6]: https://galaxy.ansible.com
[8]: http://docs.ansible.com/ansible/playbooks_best_practices.html "link title Playbook Best Practices"

