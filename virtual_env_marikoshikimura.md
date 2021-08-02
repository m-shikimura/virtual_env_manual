# 環境構築手順書

### バージョン一覧
| | |
| :---: | :---: |
| PHP | 7.3 |
| Nginx | 1.21.1 |
| MySQL | 5.7 |
| Laravel | 6.0 |

## VagrantでCentOS7環境を作成
Vagrantの作業ディレクトリを用意しましょう
以下のいずれかのディレクトリの下に virtual_env_test という名前でディレクトリを作成しましょう。

* 自分の作業用ディレクトリ
* デスクトップ

```
$ mkdir virtual_env_test

$ cd virtual_env_test

$ vagrant init centos/7
```

実行後問題なければ以下のような文言が表示されます。

```
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
```

## Vagrantfileの編集
今回行う編集は三箇所です。三箇所すべて#を外してください。
変更点②と③は下記の通り編集してください。

```
変更点①
config.vm.network "forwarded_port", guest: 80, host: 8080

変更点②
config.vm.network "private_network", ip: "192.168.33.10"
↓ 以下に編集
config.vm.network "private_network", ip: "192.168.33.19"

変更点③
config.vm.synced_folder "../data", "/vagrant_data"
↓ 以下に編集
config.vm.synced_folder "./", "/vagrant", type:"virtualbox"
```

## Vagrantを使用してゲストOSの起動
```$ vagrant up```

## ゲストOSへのログイン
```$ vagrant ssh```

## グループパッケージのインストール
```$ sudo yum -y groupinstall "development tools"```

## PHPのインストール

```
$ sudo yum -y install epel-release wget
$ sudo wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
$ sudo rpm -Uvh remi-release-7.rpm
$ sudo yum -y install --enablerepo=remi-php73 php php-pdo php-mysqlnd php-mbstring php-xml php-fpm php-common php-devel php-mysql unzip
```
下記コマンドを入力し、PHP7.3をインストールできていたら問題ありません。
```$ php -v```

## composerのインストール

```
$ php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
$ php composer-setup.php
$ php -r "unlink('composer-setup.php');"

どのディレクトリにいてもcomposerコマンドを使用できるようfileの移動を行います
$ sudo mv composer.phar /usr/local/bin/composer

下記コマンドを実行し、composerのバージョンを確認できたら問題ありません。
$ composer -v
```

## Laravelのインストール

```
$ cd /vagrant
$ omposer create-project --prefer-dist laravel/laravel laravel_app "6.*"

$ cd laravel_app
```

下記コマンドを実行し、Laravelのバージョンが6.xになっていたら問題ありません。
``` $ php artisan --version```

## Nginxのインストール

ファイルを作成します。
```$ sudo vi /etc/yum.repos.d/nginx.repo```


下記内容を記述し保存してください。

```
[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/mainline/centos/\$releasever/\$basearch/
gpgcheck=0
enabled=1
```

Nginxをインストールします。
```$ sudo yum install -y nginx```

下記コマンドを実行し、Nginxのバージョンが確認できたら問題ありません。
```$ nginx -v```

Nginxを起動します。
```$ sudo systemctl start nginx```

ブラウザに http://192.168.33.19 と入力しNginxのWelcomeページが表示されていれば問題ありません。

## Laravelを動かす

Nginxの設定ファイルを編集します。
```sudo vi /etc/nginx/conf.d/default.conf```

下記の通り編集してください。

```
server {
  listen       80;
  server_name  192.168.33.19; # Vagranfileでコメントを外した箇所のipアドレスを記述してください
  root /vagrant/laravel_app/public; # 追記
  index  index.html index.htm index.php; # 追記

  #charset koi8-r;
  #access_log  /var/log/nginx/host.access.log  main;

  location / {
      #root   /usr/share/nginx/html; # コメントアウト
      #index  index.html index.htm;  # コメントアウト
      try_files $uri $uri/ /index.php$is_args$args;  # 追記
  }

  # 省略

  以下の該当箇所のコメントアウトを指定の箇所外し、変更する場所もあるので変更を加える
  location ~ \.php$ {
  #    root           html;
      fastcgi_pass   127.0.0.1:9000;
      fastcgi_index  index.php;
      fastcgi_param  SCRIPT_FILENAME  /$document_root/$fastcgi_script_name;  # $fastcgi_script_name以前を /$document_root/に変更
      include        fastcgi_params;
  }
```

## php-fpm の設定ファイルを編集
```sudo vi /etc/php-fpm.d/www.conf```

下記の通り編集してください。

```
user = apache
↓ 以下に編集
user = vagrant

group = apache
↓ 以下に編集
group = vagrant
```

## Nginxを再起動してphp-fpmを起動

```
$ sudo systemctl restart nginx
$ sudo systemctl start php-fpm
```

ブラウザにて、 http://192.168.33.19 を入力して確認してください。
LaravelのWelcome画面が表示できていれば問題ありません。

## 403 Forbiddenエラーが表示された場合

SELinuxを無効化します。
```udo vi /etc/selinux/config```

下記の通り編集してください。

```
SELINUX=enforcing
↓ 以下に編集
SELINUX=disabled
```

vagrantを再起動
```
$ exit
$ vagrant reload
```

Nginxを再起動してphp-fpmを起動
```
$ sudo systemctl restart nginx
$ sudo systemctl start php-fpm
```

再度ブラウザにて、 http://192.168.33.19 を入力して確認してください。

## Permission deniedエラーが表示された場合
以下のようなLaravelのエラーが表示されます。
```
The stream or file "/vagrant/laravel_app/storage/logs/laravel.log" could not be opened: failed to open stream: Permission denied
```

下記のコマンドを実行し、権限を与えます。
```
$ cd /vagrant/laravel_app
$ sudo chmod -R 777 storage
```

再度ブラウザにて、 http://192.168.33.19 を入力して確認してください。
LaravelのWelcome画面が表示できていれば問題ありません。

## MySQLのインストール
```
$ sudo wget https://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm
$ sudo rpm -Uvh mysql57-community-release-el7-7.noarch.rpm
$ sudo yum install -y mysql-community-server
```

バージョンの確認がとれたらインストール完了です
``` $ mysql --version ```

## MySQLへログインする

```
$ sudo systemctl start mysqld
$ sudo cat /var/log/mysqld.log | grep 'temporary password'

下記のように表示されます。'hogehoge'の部分がパスワードなのでコピーしましょう。
2021-08-02T04:13:02.647337Z 1 [Note] A temporary password is generated for root@localhost: hogehoge
```

下記コマンドを実行、コピーしたパスワードをペーストしてログインしてください。
```
$ mysql -u root -p
Enter password:
```

ログイン後、パスワードを変更します。
新たなpasswordは、大文字小文字の英数字 + 記号かつ8文字以上で設定してください。
``` mysql > set password = "新たなpassword"; ```

## データベース作成
``` mysql > create database laravel_app; ```

データベースを作成したら、MySQLからログアウトします。
``` mysql > exit ```

## laravelに認証機能をつける
``` $ cd /vagrant/laravel_app/ ```


認証機能を実装します。
```
$ composer require laravel/ui "^1.0" --dev
$ php artisan ui vue --auth
```

laravel_appディレクトリ下の .env ファイルの内容を編集します。
``` $ vi .env ```

下記の通り編集してください。

```
DB_DATABASE=
↓ 以下に編集
DB_DATABASE=laravel_app

DB_PASSWORD=
↓ 以下に編集
DB_PASSWORD=登録したパスワード
```

マイグレーションを実行します。
``` $ php artisan migrate ```

ブラウザにて、 http://192.168.33.19 を入力、認証機能が実装されていれば問題ありません。

# 参考サイト
[Laravel 6.x 認証](https://readouble.com/laravel/6.x/ja/authentication.html)
[Vagantを使ってLaravelを動かす](https://qiita.com/hot-and-cool/items/076c0bb2690c5241ebcf)
[VagrantにSaharaを導入](https://qiita.com/sudachi808/items/09cbd3dd1f5c25c23eaf)
[マークダウン記法 一覧表・チートシート](https://qiita.com/kamorits/items/6f342da395ad57468ae3)