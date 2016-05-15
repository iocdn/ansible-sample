# 目次
- [目的](#目的)
- [手順 概要](#手順概要)
- [手順 詳細](#手順詳細)
 - [selinuxのdisable](#selinuxのdisable)
 - [Apacheのインストール](#apacheのインストール)
 - [mysqlのインストール](#mysqlのインストール)
 - [mysql 設定](#mysql設定)
 - [wordpressのインストール](#wordpressのインストール)
  - [remiリポジトリのインストール](#remiリポジトリのインストール)
  - [php,php-mbstring,php-mysqlのインストール](#php_php-mbstring_php-mysqlのインストール)
  - [php.iniの修正](#phpiniの修正)
  - [wordpressのダウンロードと展開](#wordpressのダウンロードと展開)
  - [wp-configの作成](#wp-configの作成)
  - [wordpress.confの作成](#wordpress_confの作成)
  - [適用](#適用)
 - [elasticsearchのインストール](#elasticsearchのインストール)
 - [td-agentのインストール](#td-agentのインストール)
 - [kibanaのインストール](#kibanaのインストール)
 - [動作確認](#動作確認)
 
## 目的
- サーバにwordpressをインストールすること
- kibanaでwordpressへのアクセス統計をグラフ表示すること

## 手順概要
- selinuxのdisable
- Apacheのインストール
- mysqlのインストール
- wordpressのインストール
- elasticsearchのインストール
- td-agentのインストール
- kibanaのインストール
- 動作確認

## 手順詳細
### selinuxのdisable
```shell-session
 $ su -
 # cd /etc/selinux/
 # cp -vip config config.`date +"%Y%m%d"`.bk
 - SELINUX=disabled
 + SELINUX=enforcing
 # diff -u config.`date +"%Y%m%d"`.bk config
 # setenforce 0
 # getenforce
 permissive
```

### apacheのインストール
```shell-session
 # yum install httpd
 # chkconfig httpd on
 # service mysqld start
```

### mysqlのインストール
```shell-session
 # yum install mysql-server
 # chkconfig mysql on
 # service mysqld start
 # mysql_secure_installation
  Set root password? [Y/n] Y
  New password:          <- rootパスワードを設定
  Re-enter new password: <- 〃
  Disallow root login remotely? [Y/n] Y
  Remove test database and access to it? [Y/n] Y 
  Reload privilege tables now? [Y/n] Y
```
### mysql設定
### wordpressのインストール
#### remiリポジトリのインストール
#### php_php-mbstring_php-mysqlのインストール
#### phpiniの修正
#### wordpressのダウンロードと展開
#### wp-configの作成
#### wordpress_confの作成
#### 適用

### elasticsearchのインストール
#### open-jdkのインストール
#### elasticsearchリポジトリの追加
#### elasticsearchのインストール
#### 起動

### td-agentのインストール
#### td-agentリポジトリの追加
#### td-agentのインストール
#### fluent-plugin-elasticsearchのインストール
#### td-agent_confの作成
#### ログポジション保存ディレクトリの作成
#### apacheログディレクトリの権限変更
#### td-agent再起動
### kibanaのインストール
### 動作確認
