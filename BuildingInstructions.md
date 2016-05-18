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
 # service httpd start
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
```shell-session
 # mysql -uroot -e 'create database wordpress'
 # mysql -uroot -e "GRANT ALL PRIVILEGES ON wordpress.* TO 'user01'@'localhost' IDENTIFIED BY  'ユーザパスワード';"
```

### wordpressのインストール
#### remiリポジトリのインストール
```shell-session
# yum -y install http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
# yum -y install http://rpms.famillecollet.com/enterprise/remi-release-6.rpm
```

#### php_php-mbstring_php-mysqlのインストール
```shell-session
# yum -y install php php-mbstring php-mysql
```

#### phpiniの修正
```shell-session
sudo sed -i.bk -e 's#;date.timezone.*#date.timezone = "Asia/Tokyo"#;s/;mbstring.language\s*=\s*Japanese/mbstring.language = Japanese/' /etc/php.ini
```
#### wordpressのダウンロードと展開
```shell-session
# wget https://ja.wordpress.org/wordpress-4.5.2-ja.tar.gz -o /tmp/wordpress-4.5.2-ja.tar.gz
# cd /var/www/html 
# tar -zxvf /tmp/wordpress-4.5.2-ja.tar.gz
```

#### wp-configの作成
```shell-session
# sed "s/define('DB_NAME',.*)/define('DB_NAME', 'wordpress' )/;s/define('DB_USER',.*)/define('DB_USER', 'user01')/;s/define('DB_PASSWORD',.*)/define('DB_PASSWORD', 'ユーザ素ワード')/" /var/www/html/wordpress/wp-config-sample.php > wp-config.php
```
#### wordpress_confの作成
```shell-session
# vim /etc/httpd/conf.d/wordpress.conf
以下の通り記載する
<VirtualHost *:80>
  DocumentRoot /var/www/html/wordpress
  <Directory "/var/www/html/wordpress">
    AllowOverride All
    Options -Indexes
  </Directory>

  <Files wp-config.php>
    order allow,deny
    deny from all
  </Files>
</VirtualHost>
```
#### 適用
```shell-session
 # service httpd restart
```
### elasticsearchのインストール
#### open-jdkのインストール
```shell-session
 # yum -y install java
```
#### elasticsearchリポジトリの追加
```shell-session
# vim /etc/yum.repos.d/elasticsearch.repo
以下を追記
[elasticsearch-2.x]
name=Elasticsearch repository for 2.x packages
baseurl=https://packages.elastic.co/elasticsearch/2.x/centos
gpgcheck=1
gpgkey=https://packages.elastic.co/GPG-KEY-elasticsearch
enabled=0
```
#### elasticsearchのインストール
```shell-session
# yum -y --enablerepo elasticsearch install elasticsearch 
#  # chkconfig elasticsearch on
```

#### 起動
```shell-session
# service elasticsearch start
```
### td-agentのインストール
#### td-agentリポジトリの追加
```shell-session
# vim /etc/yum.repos.d/td-agent.repo
以下を追記
[treasuredata]
name=TreasureData
baseurl=http://packages.treasuredata.com/2/redhat/\$releasever/\$basearch
gpgcheck=1
gpgkey=https://packages.treasuredata.com/GPG-KEY-td-agent
```

#### td-agentのインストール
```shell-session
# yum -y --enablerepo td-agent install td-agent
# chkconfig td-agent on
```

#### fluent-plugin-elasticsearchのインストール
```shell-session
# td-agent-gem install fluent-plugin-elasticsearch
```

#### td-agent_confの作成
```shell-session
 # cp -vip /etc/td-agent/td-agent.conf /etc/td-agent/td-agent.conf.org
 # vim /etc/td-agent/td-agent.conf
 次のように修正
 <source>
  type tail
  path /var/log/httpd/access_log 
  format apache2
  pos_file /var/log/td-agent/position/apache_access.log.pos
  tag apache.access
</source>

<match apache.access>
  type copy
  <store>
    type elasticsearch
    logstash_format true
    hosts localhost:9200
    type_name application-log
    buffer_type memory
    retry_limit 17
    retry_wait 1.0
    num_threads 1
    flush_interval 60
    retry_limit 17
  </store>
</match>
```

#### ログポジション保存ディレクトリの作成
```shell-session
 # mkdir -p /var/log/td-agent/position
 # chown td-agent:td-agent /var/log/td-agent/position
 # chmod 755 /var/log/td-agent/position
```

#### apacheログディレクトリの権限変更
```shell-session
# chmod -R  777 /var/log/httpd
```

#### td-agent再起動
```shell-session
 # service td-agent start
```
## kibanaのインストール
### kibaraのインストール
```shell-session
 # wget https://download.elastic.co/kibana/kibana/kibana-4.5.0-linux-x64.tar.gz -o /tmp/kibana-4.5.0-linux-x64.tar.gz
 # cd /usr/local
 # tar -zxvf /tmp/kibana-4.5.0-linux-x64.tar.gz
 # ln -snf kibana-4.5.0 kibana
```
### kibana initスクリプトの作成
```sell-session
 # wget https://raw.githubusercontent.com/iocdn/ansible-sample/master/roles/kibana/files/etc/init.d/kibana -o /etc/init.d/kibana
 # chmod 755 /etc/init.d/kibana
```

### 起動
```sell-session
 # service kibana start
```
