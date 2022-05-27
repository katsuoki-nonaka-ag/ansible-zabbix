# ansibleによるzabbixサーバー構築の自動化

初めまして、AGESTでエンジニアをしているのなかです。
<br>
今回はansibleという自動化ツールによるzabbixサーバーの構築について書いていきます。
ansibleの説明は後で行いますが、本構成だと10分程度でサーバーを構築できるようになります。

<br>

- [ansibleとは](#ansible)

- [前提条件](#prerequisite)

- [開発環境](#environment)

- [構築](#build)
  1. [ansibleをインストール](#install-ansible)
  2. [ansible用にファイルを用意](#set-conf-ansible)
  3. [zabbixをインストールするplaybook実行](#build-playbook)
  4. [WEBでzabbixの初期設定を実施](#setting)
  5. [zabbixを設定するplaybookを実行](#configure-playbook)
  6. [動作確認](#check)

- [注意点](#important)

- [改善点](#improvement)

<a id="ansible"></a>

## ansibleとは
[ansibleの公式サイト](https://docs.ansible.com/ansible/2.9_ja/index.html)では次のように説明されています。
> Ansible は IT 自動化ツールです。 このツールを使用すると、
> システムの構成、ソフトウェアの展開、
> より高度なITタスク (継続的なデプロイメントやダウンタイムなしのローリング更新など) 
> のオーケストレーションが可能になります。

つまりansibleはサーバーやルーターの構築・管理・設定を自動化します。

例えば本構成ではzabbixサーバーを構築する[公式サイト手順](https://www.zabbix.com/download?zabbix=6.0&os_distribution=ubuntu&os_version=20.04_focal&db=postgresql&ws=apache)を自動化します。

またansibleは[様々なモジュール](https://docs.ansible.com/ansible/2.9_ja/modules/list_of_all_modules.html)を利用可能で、本構成ではDBとzabbixを設定します。

<br>

<a id="prerequisite"></a>

## 前提条件
- [virtualbox](https://www.virtualbox.org/wiki/Downloads)インストール済み
- 仮想環境上で[Ubuntu desktop 20.04](http://cdimage.ubuntulinux.jp/releases/20.04.1/)インストール済み
- zabbixサーバーの設定ファイル[zabbix_server.conf](https://www.zabbix.com/documentation/1.8/jp/manual/processes/zabbix_server)やapache.confを取得済み

### zabbix_server.conf

```
LogFile=/var/log/zabbix/zabbix_server.log
LogFileSize=0
PidFile=/run/zabbix/zabbix_server.pid
SocketDir=/run/zabbix
DBName=zabbix
DBUser=zabbix
DBPassword=hogehoge
SNMPTrapperFile=/var/log/snmptrap/snmptrap.log
Timeout=4
FpingLocation=/usr/bin/fping
Fping6Location=/usr/bin/fping6
LogSlowQueries=3000
StatsAllowedIP=127.0.0.1

```

### apache.conf

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

<br>

<a id="environment"></a>

## 開発環境
- (ホストOS) windows 10
- (ゲストOS) Ubuntu desktop 20.04.4
- virtualbox 6.1
- ansible 2.9.6

<br>

<a id="build"></a>

## 構築

<a id="install-ansible"></a>

### 1. ansibleをインストール

```
sudo apt install -y ansible
```

<a id="set-conf-ansible"></a>

### 2. ansible用にファイルを用意

#### /etc/ansibleにzabbix-install.ymlを用意

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

### 3. zabbixをインストールするplaybook実行

```
ansible-playbook zabbix-install.yml
```

<a id="setting"></a>

### 4. WEBでzabbixの初期設定を実施




<a id="configure-playbook"></a>

### 5. zabbixを設定するplaybookを実行

/etc/ansibleにconfigure-zabbix.ymlを用意

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



<a id="important"></a>

## 注意点
playbookを実行する前にパッケージをサーバーを最新の状態にしていないとpipをインストールする際にエラーが発生することがあります。

<a id="improvement"></a>

## 改善点
現状のコードだと冪等性が無いため[特定の条件(DBが既にある等)による処理のスキップ](https://docs.ansible.com/ansible/2.9_ja/user_guide/playbooks_conditionals.html#when)や[OS・バージョンによる分岐](https://docs.ansible.com/ansible/2.9_ja/user_guide/playbooks_conditionals.html#id8)等を作る必要があります。

zabbixの初期設定をWEB上で実施していますが、この設定を自動化する方法があると、ホストの作成までを1つのplaybookで実行できるようになります。

今回はパスワードを平文で設定していますが、[暗号化](https://docs.ansible.com/ansible/2.9_ja/user_guide/vault.html)することを検討する必要があります。