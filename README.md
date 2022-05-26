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

## <a href="#ansible">ansibleとは</a>
[ansibleの公式サイト](https://docs.ansible.com/ansible/2.9_ja/index.html)では次のように説明されています。
> Ansible は IT 自動化ツールです。 このツールを使用すると、
> システムの構成、ソフトウェアの展開、
> より高度なITタスク (継続的なデプロイメントやダウンタイムなしのローリング更新など) 
> のオーケストレーションが可能になります。

つまりansibleはサーバーやルーターの構築・管理・設定を自動化します。

例えばzabbixサーバーだと次の項目を自動で実行します。
- パッケージのインストール
- リポジトリのインストール
- DBの作成
- DBのユーザー作成
- ファイル配置
- サービスの再起動
- サービスの自動起動有効
- zabbixのホスト作成

またansibleは[様々なモジュール](https://docs.ansible.com/ansible/2.9_ja/modules/list_of_all_modules.html)を利用可能で、本構成ではDBとzabbixを設定します。

<br>

<a id="prerequisite"></a>

## <a href="#prerequisite">前提条件</a>
- [virtualbox](https://www.virtualbox.org/wiki/Downloads)インストール済み
- 仮想環境上で[Ubuntu desktop 20.04](http://cdimage.ubuntulinux.jp/releases/20.04.1/)インストール済み
- zabbixサーバーのデフォルトの設定ファイルzabbix_server.confやapache.confを取得済み

<br>

<a id="environment"></a>

## <a href="#environment">開発環境</a>
- (ホストOS) windows 10
- (ゲストOS) Ubuntu desktop 20.04.4
- virtualbox 6.1
- ansible 2.9.6

<br>

<a id="build"></a>

## <a href="#build">構築</a>

<a id="install-ansible"></a>

1. <a href="#install">ansibleをインストール</a>

```
sudo apt install -y ansible
```

<a id="set-conf-ansible"></a>

2. <a href="#set-conf-ansible">ansible用にファイルを用意</a>

### /etc/ansibleにzabbix-install.ymlを用意

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

### zabbix_server.confを用意

zabbix_server.confの以下の行を編集したファイルを/etc/ansibleに用意する。

※zabbix-install.ymlのpostgresqlのユーザ作成で設定したパスワードを入力する。

```
【変更前】
# DBPassword=
【変更後】
DBPassword=[パスワードを入力]
```

### apache.confを用意

apache.confの以下の行を編集したファイルを/etc/ansibleに用意する。

```
【変更前】
# php_value date.timezone Europe/Riga
【変更後】
php_value date.timezone Asia/Tokyo
```

<a id="build-playbook"></a>

3. <a href="#build-playbook">zabbixをインストールするplaybook実行</a>

```
ansible-playbook zabbix-install.yml
```

<a id="setting"></a>

4. <a href="#setting">WEBでzabbixの初期設定を実施</a>

<a id="configure-playbook"></a>

5. <a href="#configure-playbook">zabbixを設定するplaybookを実行</a>

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

6. <a href="#check">動作確認</a>

<a id="important"></a>

## <a href="#important">注意点</a>
playbookを実行する前にパッケージをサーバーを最新の状態にしていないとpipをインストールする際にエラーが発生する可能性があります。

<a id="improvement"></a>

## <a href="#improvement">改善点</a>
現状のコードだと冪等性が無いためファイルがある場合の処理や[OS・バージョン](https://www.zabbix.com/download?zabbix=6.0&os_distribution=ubuntu&os_version=20.04_focal&db=postgresql&ws=apache)による分岐等を作る必要があります。

zabbixの初期設定をWEB上で実施していますが、この設定を自動化する方法があると、ホストの作成までを1つのplaybookで実行できるようになります。

パスワードを平文で設定していますが、[ansible-vault](https://docs.ansible.com/ansible/2.9_ja/user_guide/vault.html)を利用し、暗号化をした方が良い。