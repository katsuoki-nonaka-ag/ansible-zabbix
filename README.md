# ansibleによるzabbixサーバー構築の自動化に挑戦

初めまして、AGESTでエンジニアをしているのなかです。
<br>
今回はansibleという自動化ツールによるzabbixサーバーの構築について書いていきます。
ansibleの説明は後で行いますが、使えるようになると「自動化って便利だなあ」と思えるので試してみるのをオススメします。
<br>

- [ansibleとは](#ansible)

- [zabbixとは](#zabbix)

- [前提条件](#prerequisite)

- [開発環境](#environment)

- [構築](#build)
  - [概要](#summary)
  - [手順](#process)

## <a id="#ansible">ansibleとは</a>
[ansibleの公式サイト](https://docs.ansible.com/ansible/2.9_ja/index.html)では次のように説明されています。
> Ansible は IT 自動化ツールです。 このツールを使用すると、
> システムの構成、ソフトウェアの展開、
> より高度なITタスク (継続的なデプロイメントやダウンタイムなしのローリング更新など) 
> のオーケストレーションが可能になります。

<br>

## <a id="#zabbix">zabbixとは</a>
[zabbixの公式サイト](https://www.zabbix.com/documentation/2.2/jp/manual/introduction/about)では次のように説明されています。
> Zabbixは多数のネットワークのパラメータおよびサーバの稼働状態と整合性を監視するためのソフトウェアです。
> Zabbixは柔軟性の高い通知メカニズムを備え、ユーザはあらゆるイベントからメールベースの通知を行うように
> 設定することができます。これらの機能によりサーバの障害に迅速に対応することができます。
> Zabbixは保存されたデータをもとにすぐれたレポートやデータのグラフィカル表示機能を提供します。

<br>

## <a id="#prerequisite">前提条件</a>
- [virtualbox](https://www.virtualbox.org/wiki/Downloads)インストール済み
- 仮想環境上で[Ubuntu desktop 20.04](http://cdimage.ubuntulinux.jp/releases/20.04.1/)インストール済み

<br>

## <a id="#environment">開発環境</a>
- (ホストOS) windows 10
- (ゲストOS) Ubuntu desktop 20.04.4
- virtualbox 6.1
- zabbix 6.0
- postgresql

<br>

## <a id="#build">構築</a>

### <a id="#summary">概要</a>
ゲストOS上でansibleのplaybookを実行し、zabbixサーバーを構築

### <a id="#process">手順</a>
1. ansibleをインストール

```
sudo -s
apt install -y ansible
```

2. バックアップ取得

```
cd /etc/ansilbe
mkdir bk
cp -p ansible.cfg bk/ansible.cfg.org
cp -p hosts bk/hosts
```

3. playbook作成

`vi zabbix-install.yml`でplaybookを作成し、以下のように編集

```
- hosts: all
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
      shell: wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-1+ubuntu20.04_all.deb

    - name: dpkg
      shell: dpkg -i zabbix-release_6.0-1+ubuntu20.04_all.deb

    - name: apt update
      shell: apt update

    - name: install package  
      apt:
        name:
          - zabbix-server-pgsql
          - zabbix-frontend-php
          - php7.4-pgsql
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
        password: [パスワードを入力]

    - name: configure db
      shell: zcat /usr/share/doc/zabbix-server-pgsql*/create.sql.gz | sudo -u zabbix psql zabbix
```


3. playbook実行

```
ansible-playbook zabbix-install.yml
```

4. zabbixサーバーの設定ファイルを編集し、設定を追加

※zabbix-install.ymlで設定したパスワードと同じパスワードを設定する。
#### 設定ファイル編集
```
cp /etc/zabbix/zabbix_server.conf /etc/zabbix/zabbix_server.conf_org
vi /etc/zabbix/zabbix_server.conf
```

#### 設定ファイルに追加

```
DBPassword=[パスワードを入力]`
```

5. apacheの設定ファイルを編集

#### 設定ファイル編集

```
cp /etc/zabbix/apache.conf /etc/zabbix/apache.conf_org
vi /etc/zabbix/apache.conf
```
#### 設定ファイルに追加

```
php_value date.timezone Asia/Tokyo`
```

6. デーモンを起動し、有効化

```
systemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2
```

7. 動作確認

ゲストOS上でfirefoxを起動し、localhost/zabbixにアクセスし、WEB-UIが表示されることを確認する。



