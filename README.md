# ansibleによるzabbixサーバー構築の自動化

初めまして、AGESTでエンジニアをしているのなかです。
<br>
今回はansibleという自動化ツールによるzabbixサーバーの構築についての話になります。
<br>
ansibleを使えると10分程度でzabbixサーバーを構築できるようになります。
<br>
それではansibleを使えるようになるために**ansibleとは何か**や**どうやって使うのか**について説明していきます。

<br>

- [ansibleとは](#ansible)
- [開発環境](#environment)
- [注意点](#important)
- [前提条件](#prerequisite)
- [構築](#build)
  1. [ansibleをインストール](#install-ansible)
  2. [ansible用にファイルを用意](#set-conf-ansible)
  3. [zabbixをインストールするplaybook実行](#build-playbook)
  4. [WEBでzabbixの初期設定を実施](#setting)
  5. [zabbixを設定するplaybookを実行](#configure-playbook)
  6. [動作確認](#check)
- [改善点](#improvement)

<a id="ansible"></a>

## ansibleとは
[ansibleの公式ドキュメント](https://docs.ansible.com/ansible/2.9_ja/index.html)では次のように説明されています。
> Ansible は IT 自動化ツールです。 このツールを使用すると、
> システムの構成、ソフトウェアの展開、
> より高度なITタスク (継続的なデプロイメントやダウンタイムなしのローリング更新など) 
> のオーケストレーションが可能になります。

簡単に説明するとansibleはサーバーやルーターの構築・管理・設定を自動化することを目的として使用されます。
<br>
例えば今回のzabbixサーバー構築では[公式サイトの手順](https://www.zabbix.com/download?zabbix=6.0&os_distribution=ubuntu&os_version=20.04_focal&db=postgresql&ws=apache)を全て自動化しています。
<br>
さらにansibleは[様々なモジュール](https://docs.ansible.com/ansible/2.9_ja/modules/list_of_all_modules.html)が利用可能なため、DBの作成やzabbixのホスト作成等の様々な設定を自動化できます。

<br>

<a id="environment"></a>

## 開発環境
- (ホストOS) windows 10
- (ゲストOS) Ubuntu desktop 20.04.4
- virtualbox 6.1
- ansible 2.9.6

<a id="prerequisite"></a>

<br>

## 前提条件
- [virtualbox](https://www.virtualbox.org/wiki/Downloads)インストール済み
- 仮想環境上で[Ubuntu desktop 20.04](http://cdimage.ubuntulinux.jp/releases/20.04.1/)がインストール済み
- Ubuntuが20.04の最新版(Ubuntu 20.04.4)でパッケージリストが最新の状態
- zabbixサーバーの設定ファイル[zabbix_server.conf](https://www.zabbix.com/documentation/1.8/jp/manual/processes/zabbix_server)やapache.confを用意済み

<br>

<a id="important"></a>

## 注意点
2022年5月時点では[Ubuntu22.04でzabbixサーバーを構築](https://www.zabbix.com/download?zabbix=6.0&os_distribution=ubuntu&os_version=22.04_jammy&db=&ws=)することが出来ないためUbuntu 20.04を使用しています。
<br>
今回はパスワードを平文で設定していますが、実際にサーバーを構築する際は[暗号化する](https://docs.ansible.com/ansible/2.9_ja/user_guide/vault.html)等の対策をする必要があります。

<br>

<a id="build"></a>

## 構築
ansibleを使用して自動化するには[playbook](https://docs.ansible.com/ansible/2.9_ja/user_guide/playbooks.html)を作成する必要があります。

<a id="install-ansible"></a>

### 1. ansibleをインストール
```
sudo apt install -y ansible
```

<a id="set-conf-ansible"></a>

### 2. ansible用にファイルを配置

#### /etc/ansibleにzabbix-install.ymlを配置
```
- hosts: localhost
  become: yes
  tasks:
    - name: install pip
      apt:
        name:
          - python3
          - python3-pip

    - name: set python3 to default
      shell: update-alternatives --install /usr/bin/python python /usr/bin/python3.8 0

    - name: install psycopg2
      shell: pip install psycopg2-binary

    - name: install zabbix repos
      get_url:
        url: https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-1+ubuntu20.04_all.deb
        dest: /tmp

    - name: dpkg zabbix repos
      shell: dpkg -i /tmp/zabbix-release_6.0-1+ubuntu20.04_all.deb

    - name: apt update
      shell: apt update

    - name: install package  
      apt:
        name:
          - zabbix-server-pgsql
          - zabbix-frontend-php
          - php7.4-pgsql
          - zabbix-apache-conf
          - zabbix-sql-scripts
          - zabbix-agent
          - postgresql

    - name: install zabbix-api
      shell: pip install zabbix-api

    - name: create db
      become_user: postgres
      postgresql_db:
        name: zabbix

    - name: create user zabbix
      become_user: postgres
      postgresql_user:
        db: zabbix
        name: zabbix
        password: hogehoge

    - name: configure db
      shell: zcat /usr/share/doc/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix

    - name: get server conf backup
      shell: cp /etc/zabbix/zabbix_server.conf /etc/zabbix/zabbix_server.conf_org

    - name: get apache backup
      shell: cp /etc/zabbix/apache.conf /etc/zabbix/apache.conf_org

    - name: copy server conf
      shell: cp -p /etc/ansible/zabbix_server.conf /etc/zabbix/zabbix_server.conf

    - name: copy apache conf
      shell: cp -p /etc/ansible/apache.conf /etc/zabbix/apache.conf

    - name: restart zabbix server
      service: 
        name: zabbix-server
        state: restarted
        enabled: yes

    - name: restart zabbix agent
      service:
        name: zabbix-agent
        state: restarted
        enabled: yes

    - name:
      service:
        name: apache2
        state: restarted
        enabled: yes
```

#### /etc/ansibleにzabbix_server.confを配置
```
LogFile=/var/log/zabbix/zabbix_server.log

LogFileSize=0

PidFile=/run/zabbix/zabbix_server.pid

SocketDir=/run/zabbix

DBName=zabbix

DBUser=zabbix

# DBPasswordを追加
DBPassword=hogehoge

SNMPTrapperFile=/var/log/snmptrap/snmptrap.log

Timeout=4

FpingLocation=/usr/bin/fping

Fping6Location=/usr/bin/fping6

LogSlowQueries=3000

StatsAllowedIP=127.0.0.1
```

#### /etc/ansibleにapache.confを配置
```
# Define /zabbix alias, this is the default
<IfModule mod_alias.c>
    Alias /zabbix /usr/share/zabbix
</IfModule>

<Directory "/usr/share/zabbix">
    Options FollowSymLinks
    AllowOverride None
    Order allow,deny
    Allow from all

    <IfModule mod_php7.c>
        php_value max_execution_time 300
        php_value memory_limit 128M
        php_value post_max_size 16M
        php_value upload_max_filesize 2M
        php_value max_input_time 300
        php_value max_input_vars 10000
        php_value always_populate_raw_post_data -1
        # タイムゾーンを変更
        php_value date.timezone Asia/Tokyo
    </IfModule>
</Directory>

<Directory "/usr/share/zabbix/conf">
    Order deny,allow
    Deny from all
    <files *.php>
        Order deny,allow
        Deny from all
    </files>
</Directory>

<Directory "/usr/share/zabbix/app">
    Order deny,allow
    Deny from all
    <files *.php>
        Order deny,allow
        Deny from all
    </files>
</Directory>

<Directory "/usr/share/zabbix/include">
    Order deny,allow
    Deny from all
    <files *.php>
        Order deny,allow
        Deny from all
    </files>
</Directory>

<Directory "/usr/share/zabbix/local">
    Order deny,allow
    Deny from all
    <files *.php>
        Order deny,allow
        Deny from all
    </files>
</Directory>

<Directory "/usr/share/zabbix/vendor">
    Order deny,allow
    Deny from all
    <files *.php>
        Order deny,allow
        Deny from all
    </files>
</Directory>
```

### 3. zabbixをインストールするplaybook実行
```
ansible-playbook zabbix-install.yml
```

<a id="setting"></a>

### 4. WEBインターフェースによるzabbixインストール
localhost/zabbixにアクセスし、[公式ドキュメント](https://www.zabbix.com/documentation/current/en/manual/installation/frontend)を参考にインストールします。
<br>
Configure DB connectionの画面はポート番号とパスワードを入力することでDBに接続できます。
- Database port: 5432
- Password: hogehoge

[![Image from Gyazo](https://i.gyazo.com/dbddff5773451a35bab3eb891c92bbdd.png)](https://gyazo.com/dbddff5773451a35bab3eb891c92bbdd)

<a id="configure-playbook"></a>

### 5. zabbixを設定するplaybookを実行
#### /etc/ansibleにconfigure-zabbix.ymlを配置
```
- hosts: localhost
  become: yes
  tasks:
    - name: Create a new host or update an existing host's info
      local_action:
        module: zabbix_host
        server_url: http://localhost/zabbix/
        login_user: Admin
        login_password: zabbix
        host_name: ExampleHosts
        host_groups:
          - Linux servers
        link_templates:
          - Linux CPU by Zabbix agent
        interfaces:
          - type: 1
            main: 1
            useip: 1
            ip: 172.0.0.1
            dns: ""
            port: 10051
```

<a id="check"></a>

### 6. 動作確認

WEB画面でExampleHostsが作成されていることを確認します。

[![Image from Gyazo](https://i.gyazo.com/1582663304fa44f3605fcd4792af1938.png)](https://i.gyazo.com/1582663304fa44f3605fcd4792af1938.png)

<br>

<a id="improvement"></a>

## 改善点
現状では最小構成かつ冪等性があまり無いため[ロール](https://docs.ansible.com/ansible/2.9_ja/user_guide/playbooks_reuse_roles.html)や[特定の条件(DBが既にある等)における処理](https://docs.ansible.com/ansible/2.9_ja/user_guide/playbooks_conditionals.html#when)や[OS・バージョンによる分岐](https://docs.ansible.com/ansible/2.9_ja/user_guide/playbooks_conditionals.html#id8)等を設定することで、冪等性がかなりある状態になり複雑なサーバーの構築や設定も出来るようになります。
<br>
サーバーにansible等の不要なパッケージをインストールしたくない場合は、構築するサーバー側のSSHや[hosts](https://docs.ansible.com/ansible/2.9_ja/user_guide/playbooks_intro.html#playbook-hosts-and-users)を設定することでホストOSや別のサーバーからansibleを実行できます。

