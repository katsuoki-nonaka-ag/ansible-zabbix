# Ubuntu20.04 zabbixサーバインストール

## ansibleインストール
`sudo apt install ansible

## ファイルのバックアップ
```
cd /etc/ansible
mkdir bk
sudo cp -p ansible.cfg bk/ansible.cfg.org
sudo cp -p hosts bk/hosts
```

## ホームディレクトリへ移動
`cd ~`

## git clone
`git clone https://github.com/katsuoki-nonaka-ag/ansible-zabbix.git`

## ファイルのコピー
`sudo cp -p ./ansible-zabbix/* /etc/ansible`

## ansible実行
`ansible-playbook zabbix-install.yml`

