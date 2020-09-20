# Ansible Playbook: nginx_php7_mariadb

## 概要

* Ansibleで nginx + PHP7 + MariaDB の環境を作成する
* 本番環境はAWSのAmazonLinux2で作成する
* ローカル開発環境はVagrantのCentOS7で作成する

## 本番環境: Amazon Linux 2を準備

### Ansibleをインストール

以下のようにして、ExtrasリポジトリからAnsible・nginx・PHP7を手動でインストールする

```bash
$ sudo su -
# amazon-linux-extras install ansible2 -y
# amazon-linux-extras install nginx1.12 -y
# amazon-linux-extras install php7.3 -y
```

本番用の構築なら、以下なども作業する

* スワップ領域を設定
* SSHのポート番号を変更
* root宛メールを転送
* CloudWatch Logsを設定

Playbookは `/home/ec2-user/ansible` に配置するものとする

## ローカル開発環境: Vagrantを準備

VagrantでCentOS7を新規にインストールし、そこに開発環境（nginx 1.16 + PHP 7.3 + MariaDB 5.5）を構築する<br>
作業フォルダは、ここでは `C:\Users\refirio\Vagrant\nginx_php7_mariadb` とする<br>
VagrantのIPアドレスは、ここでは `192.168.33.10` とする

### ボックスの追加

以下のコマンドで公式のCentOS7を追加

```
>vagrant box add centos/7
```

以下のコマンドで確認すると `centos/7 (virtualbox, 1905.1)` が追加されている

```
>vagrant box list
```

### Vagrantfile

```
Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  config.vm.box_check_update = false
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.synced_folder "./code", "/var/www"
end
```

Vagrantfile と同階層に、同期用フォルダとして `code` を作成しておく<br>
Playbook は `code/ansible-develop` に配置するものとする（つまりサーバ内の `/var/www/ansible-develop` に配置される）

### 起動

```
>cd C:\Users\refirio\Vagrant\nginx_php7_mariadb
>vagrant up
```

### 終了する場合

```
>cd C:\Users\refirio\Vagrant\nginx_php7_mariadb
>vagrant halt
```

### 破棄する場合

```
>cd C:\Users\refirio\Vagrant\nginx_php7_mariadb
>vagrant destroy
```

### 初期起動時にエラーになった場合

```
>vagrant plugin install vagrant-vbguest
>vagrant halt
>vagrant up
```

### Ansibleをインストール

```bash
$ sudo su -
# yum -y install epel-release
# yum -y install ansible
```

## Ansibleで環境構築

### Ansibleを実行

Ansibleのhostsファイルの最後に設定を追加

```bash
# vi /etc/ansible/hosts
- - - - - - - - - - - - - - - - - - - - - - - - -
[localhost]
127.0.0.1
- - - - - - - - - - - - - - - - - - - - - - - - -
# exit
```

接続をテスト

```bash
$ ansible localhost -m ping --connection=local
```

Playbookの場所へ移動してから以下を実行

```bash
$ ansible-playbook site.yml --connection=local
```

ブラウザから以下にアクセスして確認<br>
http://192.168.33.10/

### データベースと接続ユーザを作成

データベースと接続ユーザを作成

```bash
$ mysql -u root -p
（パスワードなし）
mysql> GRANT ALL PRIVILEGES ON main.* TO webmaster@localhost IDENTIFIED BY '1234';
mysql> FLUSH PRIVILEGES;
mysql> CREATE DATABASE main DEFAULT CHARACTER SET utf8mb4;
mysql> QUIT;
```

### データベースへの接続をテスト

```bash
# mysql -u webmaster -p
1234
mysql> QUIT;
```

以下でPHPからアクセスできる
```php
<?php

try {
    $pdo = new PDO(
        'mysql:dbname=main;host=localhost',
        'webmaster',
        '1234'
    );

    $stmt = $pdo->query('SELECT NOW() AS now;');
    $data = $stmt->fetch(PDO::FETCH_ASSOC);
    echo "<p>" . $data['now'] . "</p>\n";

    $pdo = null;
} catch (PDOException $e) {
    exit($e->getMessage());
}
```

### メモ

Amazon Linux 2でPlaybookを実行すると以下の警告が表示される

```bash
 [WARNING]: Platform linux on host 127.0.0.1 is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python
interpreter could change this. See https://docs.ansible.com/ansible/2.8/reference_appendices/interpreter_discovery.html for more information.
```

「Pythonが /usr/bin/python にあるが、将来別のPythonがインストールされる予定です」とのこと<br>
ひとまずこのままでも問題無さそうだが、そのうち詳細を調べたい

## 本番環境: バーチャルホストの設定

上の手順で構築すると、ドメインでのアクセスでもIPアドレスのアクセスでも同じ場所を参照する<br>
一つのサーバで複数サイトを管理したい場合や、SEOなどの理由でIPアドレスでアクセスされたくない場合や、「IPアドレスでアクセスされたときは別の画面を表示したい」
のような場合、別途バーチャルホストを設定する

ただしドメイン割り当てまでは、アクセスするにはhostsの設定が必要になるので注意が必要<br>
またAWSでロードバランサー使う場合、ロードバランサーには固定IPアドレスが無いので注意（hostsの定期的な変更が必要になる可能性がある）
