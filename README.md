# Ubuntu20.04 zabbixサーバインストール

## ansibleインストール
`sudo apt install ansible`

## ファイルのバックアップ
```
cd /etc/ansible
mkdir bk
sudo cp -p ansible.cfg bk/ansible.cfg.org
sudo cp -p hosts bk/hosts
```

##gitの設定
```
git config --global user.name [ここにgithubのユーザ名を入力]
git config --global user.email [ここにgithubのメールアドレスを入力]
```

## git clone
```
cd ~
git clone https://github.com/katsuoki-nonaka-ag/ansible-zabbix.git
```

## ファイルのコピー
`sudo cp -p ./ansible-zabbix/* /etc/ansible`

## playbook実行
`ansible-playbook zabbix-install.yml`

