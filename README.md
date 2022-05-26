# ansibleによるzabbixサーバー構築の自動化

初めまして、AGESTでエンジニアをしているのなかです。
<br>
今回はansibleという自動化ツールによるzabbixサーバーの構築について書いていきます。
ansibleの説明は後で行いますが、今回の構成だと10分程度でサーバーを構築出来るようになります。

<br>

- [ansibleとは](#ansible)

- [zabbixとは](#zabbix)

- [前提条件](#prerequisite)

- [開発環境](#environment)

- [構築](#build)
  - [概要](#summary)
  - [手順](#process)

- [注意点](#important)

- [改善点](#improvement)

<a id="ansible"></a>

## <a href="#ansible">ansibleとは</a>
[ansibleの公式サイト](https://docs.ansible.com/ansible/2.9_ja/index.html)では次のように説明されています。
> Ansible は IT 自動化ツールです。 このツールを使用すると、
> システムの構成、ソフトウェアの展開、
> より高度なITタスク (継続的なデプロイメントやダウンタイムなしのローリング更新など) 
> のオーケストレーションが可能になります。

この説明によるとansibleはサーバーやルーターの構築・管理・設定等を自動化することが可能です。

例えば今回のzabbixサーバー構築では次の項目を自動化しました。
※db関連はansible2.9で使用出来ないモジュールがあるため、モジュールを利用していません。

- パッケージのインストール
- リポジトリのインストール
- DBの作成
- DBのユーザー作成
- ファイル配置
- サービスの再起動
- サービスの自動起動有効



**※ansibleは自動化ツールなので、冪等性がある方が良いです。**

<br>

<a id="zabbix"></a>

## <a href="#zabbix">zabbixとは</a>
[zabbixの公式サイト](https://www.zabbix.com/documentation/2.2/jp/manual/introduction/about)では次のように説明されています。
> Zabbixは多数のネットワークのパラメータおよびサーバの稼働状態と整合性を監視するためのソフトウェアです。
> Zabbixは柔軟性の高い通知メカニズムを備え、ユーザはあらゆるイベントからメールベースの通知を行うように
> 設定することができます。これらの機能によりサーバの障害に迅速に対応することができます。
> Zabbixは保存されたデータをもとにすぐれたレポートやデータのグラフィカル表示機能を提供します。

この説明によるとzabbixはサーバーを監視し、グラフ化や通知を行うことが可能です。

ansibleのzabbixモジュールを利用することで、zabbixも設定出来ます。

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
- zabbix-server-pgsql 6.0.4
- zabbix-frontend-php 6.0.4
- php-pgsql 7.4.3
- zabbix-apache-conf 6.0.4
- zabbix-sql-scripts 6.0.4
- zabbix-agent 6.0.4
- postgresql 12

<br>

<a id="build"></a>

## <a href="#build">構築</a>

<a id="summary"></a>

### <a href="#summary">概要</a>
ゲストOS上でansibleのplaybookを実行し、zabbixサーバーを構築

<a id="process"></a>

### <a href="#process">手順</a>
1. ansibleをインストール

```
sudo apt install -y ansible
```

2. ansible用にファイルを用意

/etc/ansibleにzabbix-install.ymlを用意

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
        url: https://repo.zabbix.com/zabbix/6.1/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.1-1+ubuntu20.04_all.deb
        dest: /tmp

    - name: dpkg zabbix repos
      shell: dpkg -i /tmp/zabbix-release_6.1-1+ubuntu20.04_all.deb

    - name: apt update
      shell: apt update

    - name: install package  
      apt:
        name:
          - zabbix-server-pgsql
          - zabbix-frontend-php
          - php-pgsql
          - zabbix-apache-conf
          - zabbix-agent
          - postgresql

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
      shell: zcat /usr/share/doc/zabbix-server-pgsql*/create.sql.gz | sudo -u zabbix psql zabbix

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

#### zabbix_server.confを用意

zabbix_server.confの以下の行を編集したファイルを/etc/ansibleに用意する。

※zabbix-install.ymlのpostgresqlのユーザ作成で設定したパスワードを入力する。

```
【変更前】
# DBPassword=
【変更後】
DBPassword=[パスワードを入力]
```

#### apache.confを用意

apache.confの以下の行を編集したファイルを/etc/ansibleに用意する。

```
【変更前】
# php_value date.timezone Europe/Riga
【変更後】
php_value date.timezone Asia/Tokyo
```

3. playbook実行

```
ansible-playbook zabbix-install.yml
```

4. 動作確認

ゲストOS上でfirefoxを起動し、localhost/zabbixにアクセスし、WEB-UIが表示されることを確認する。

![画像](https://gyazo.com/19a090597da1ee4da8865c25a387d681)

<a id="important"></a>

## <a href="#important">注意点</a>


<a id="improvement"></a>

## <a href="#improvement">改善点</a>
