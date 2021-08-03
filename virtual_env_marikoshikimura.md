# 環境構築手順書

### 目次
1. VagrantでCentOS7環境を作成
2. Vagrantfileの編集
3. Vagrantを使用してゲストOSの起動
4. ゲストOSへのログイン
5. グループパッケージのインストール
6. PHPのインストール
7. composerのインストール
8. Laravelのインストール
9. Nginxのインストール
10. Laravelを動かす
11. 403 Forbiddenエラーが表示された場合
12. Permission deniedエラーが表示された場合
13. MySQLのインストール
14. MySQLへログインする
15. データベース作成
16. Laravelに認証機能をつける

### バージョン一覧
| | |
| :---: | :---: |
| PHP | 7.3 |
| Nginx | 1.21.1 |
| MySQL | 5.7 |
| Laravel | 6.0 |

## 1. VagrantでCentOS7環境を作成
Vagrantの作業ディレクトリを用意しましょう
以下のいずれかのディレクトリの下に virtual_env_test という名前でディレクトリを作成しましょう。

* 自分の作業用ディレクトリ
* デスクトップ

```sh
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

## 2. Vagrantfileの編集
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

## 3. Vagrantを使用してゲストOSの起動
```sh
$ vagrant up
```

## 4. ゲストOSへのログイン
```sh
$ vagrant ssh
```

## 5. グループパッケージのインストール
```sh
$ sudo yum -y groupinstall "development tools"
```

## 6. PHPのインストール

```sh
$ sudo yum -y install epel-release wget
$ sudo wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
$ sudo rpm -Uvh remi-release-7.rpm
$ sudo yum -y install --enablerepo=remi-php73 php php-pdo php-mysqlnd php-mbstring php-xml php-fpm php-common php-devel php-mysql unzip
```
下記コマンドを入力し、PHP7.3をインストールできていたら問題ありません。
```sh
$ php -v
```

## 7. composerのインストール

```sh
$ php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
$ php composer-setup.php
$ php -r "unlink('composer-setup.php');"

どのディレクトリにいてもcomposerコマンドを使用できるようfileの移動を行います
$ sudo mv composer.phar /usr/local/bin/composer

下記コマンドを実行し、composerのバージョンを確認できたら問題ありません。
$ composer -v
```

## 8. Laravelのインストール

```sh
$ cd /vagrant
$ omposer create-project --prefer-dist laravel/laravel laravel_app "6.*"

$ cd laravel_app
```

下記コマンドを実行し、Laravelのバージョンが6.xになっていたら問題ありません。
```sh
$ php artisan --version
```

## 9. Nginxのインストール

ファイルを作成します。
```sh
$ sudo vi /etc/yum.repos.d/nginx.repo
```


下記内容を記述し保存してください。

```
[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/mainline/centos/\$releasever/\$basearch/
gpgcheck=0
enabled=1
```

Nginxをインストールします。
```sh
$ sudo yum install -y nginx
```

下記コマンドを実行し、Nginxのバージョンが確認できたら問題ありません。
```sh
$ nginx -v
```

Nginxを起動します。
```sh
$ sudo systemctl start nginx
```

ブラウザに http://192.168.33.19 と入力しNginxのWelcomeページが表示されていれば問題ありません。

## 10. Laravelを動かす

Nginxの設定ファイルを編集します。
```sh
$ sudo vi /etc/nginx/conf.d/default.conf
```

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

次に、php-fpm の設定ファイルを編集します。
```sh
$ sudo vi /etc/php-fpm.d/www.conf
```

下記の通り編集してください。

```
user = apache
↓ 以下に編集
user = vagrant

group = apache
↓ 以下に編集
group = vagrant
```

Nginxを再起動してphp-fpmを起動しましょう。

```sh
$ sudo systemctl restart nginx
$ sudo systemctl start php-fpm
```

ブラウザにて、 http://192.168.33.19 を入力して確認してください。
LaravelのWelcome画面が表示できていれば問題ありません。

## 11. 403 Forbiddenエラーが表示された場合

SELinuxを無効化します。
```sh
$ udo vi /etc/selinux/config
```

下記の通り編集してください。

```
SELINUX=enforcing
↓ 以下に編集
SELINUX=disabled
```

vagrantを再起動
```sh
$ exit
$ vagrant reload
```

Nginxを再起動してphp-fpmを起動
```sh
$ sudo systemctl restart nginx
$ sudo systemctl start php-fpm
```

再度ブラウザにて、 http://192.168.33.19 を入力して確認してください。

## 12. Permission deniedエラーが表示された場合
以下のようなLaravelのエラーが表示されます。
```
The stream or file "/vagrant/laravel_app/storage/logs/laravel.log" could not be opened: failed to open stream: Permission denied
```

下記のコマンドを実行し、権限を与えます。
```sh
$ cd /vagrant/laravel_app
$ sudo chmod -R 777 storage
```

再度ブラウザにて、 http://192.168.33.19 を入力して確認してください。
LaravelのWelcome画面が表示できていれば問題ありません。

## 13. MySQLのインストール
```sh
$ sudo wget https://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm
$ sudo rpm -Uvh mysql57-community-release-el7-7.noarch.rpm
$ sudo yum install -y mysql-community-server
```

バージョンの確認がとれたらインストール完了です
```sql
 $ mysql --version
 ```

## 14. MySQLへログインする

```sh
$ sudo systemctl start mysqld
$ sudo cat /var/log/mysqld.log | grep 'temporary password'

下記のように表示されます。'hogehoge'の部分がパスワードなのでコピーしましょう。
2021-08-02T04:13:02.647337Z 1 [Note] A temporary password is generated for root@localhost: hogehoge
```

下記コマンドを実行、コピーしたパスワードをペーストしてログインしてください。
```sql
$ mysql -u root -p
Enter password:
```

ログイン後、パスワードを変更します。
新たなpasswordは、大文字小文字の英数字 + 記号かつ8文字以上で設定してください。
```sql
mysql > set password = "新たなpassword";
```

## 15. データベース作成
```sql
mysql > create database laravel_app;
```

データベースを作成したら、MySQLからログアウトします。
```sql
mysql > exit
```

## 16. Laravelに認証機能をつける
```sh
$ cd /vagrant/laravel_app/
```


認証機能を実装します。
```sh
$ composer require laravel/ui "^1.0" --dev
$ php artisan ui vue --auth
```

laravel_appディレクトリ下の .env ファイルの内容を編集します。
```sh
$ vi .env
```

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
```sh
$ php artisan migrate
```

ブラウザにて、 http://192.168.33.19 を入力、認証機能が実装されていれば問題ありません。

## 環境構築の所感
今回一から環境構築を行い、コマンドを実行した後に表示される内容を確認することが大切であると学びました。エラーが起きたり、自分の予想と異なる動きをした時、実行したコマンドの結果に目を通すことで解決することができました。

# 参考サイト
[Laravel 6.x 認証](https://readouble.com/laravel/6.x/ja/authentication.html)
[Vagantを使ってLaravelを動かす](https://qiita.com/hot-and-cool/items/076c0bb2690c5241ebcf)
[VagrantにSaharaを導入](https://qiita.com/sudachi808/items/09cbd3dd1f5c25c23eaf)
[マークダウン記法 一覧表・チートシート](https://qiita.com/kamorits/items/6f342da395ad57468ae3)