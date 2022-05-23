# Ubuntu20.04 zabbixサーバインストール

## ansibleインストール
sudo apt install ansible

## git インストール
sudo apt git install

## ファイルのバックアップ
cd /etc/ansible
mkdir bk
sudo cp -p ansible.cfg bk/ansible.cfg.org
sudo cp -p hosts bk/hosts

## ホームディレクトリへ移動
cd ~

## git clone
git clone https://github.com/katsuoki-nonaka-ag/ansible-zabbix.git

## ファイルのコピー
sudo cp -p ./ansible-zabbix/* /etc/ansible

## ansible実行
ansible-playbook zabbix-install.yml

## zabbix_server.confを編集
sudo cp /etc/zabbix/zabbix_server.conf /etc/zabbix/zabbix_server.conf_org
sudo vi /etc/zabbix/zabbix_server.conf

"""
#DBPassword=password   //コメントアウトを外し
DBPassword=好きなパスワードを設定
"""

## timezoneを変更
cp /etc/zabbix/apache.conf /etc/zabbix/apache.conf_org
vi /etc/zabbix/apache.conf

"""
【変更前】
# php_value date.timezone Europe/Riga
【変更後】
php_value date.timezone Asia/Tokyo
"""

## デーモンをリスタート
systemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2
