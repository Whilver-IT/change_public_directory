# DocumentRootを変更できない環境でLaravelを移行してみる

## 1. 目的

先日、とある相談を受けたのでまとめてみる(2024-03-25に連絡を受ける)  
どうも、さくらのレンタルサーバに既に何かWebサービスが稼働している状態に、別環境のlaravelを移行したいとのこと(テストのためかな!?)

## 2. 事前情報

基本的にはさくらのレンタルサーバの基本仕様のままみたい  
[基本仕様を知りたい](https://help.sakura.ad.jp/rs/2251/)、[page2](https://help.sakura.ad.jp/rs/2251/?article_page=2)

さくら自体は情報を掲載していることは知っているが、私自身はコントロールパネルには入れないのと、既に動いているものがあるので(といっても、静的htmlがいくつかだけの模様)  
[さくらのレンタルサーバでLaravelを動かす方法](https://knowledge.sakura.ad.jp/35587)

環境構築しない場合(既に環境がある)は9-2まで進んでください  
さくらのレンタルサーバの環境を完全に模倣しているわけではないので、自分の環境ではうまくいっても、さくらのレンタルサーバではうまくいかないかもしれません  
あくまでも自己責任でお願い致します  

参考サイトは以下  
[Laravel の publicフォルダの階層を移動させたい](https://qiita.com/matsuoshi/items/b0dceb9b380a9282719a)  
[Laravelで、public(公開)フォルダの位置を変更する際の設定](https://qiita.com/1stMinos/items/d76a4e5268a8df5d5bd2)

## 3. 環境構築

以下のようにする

<table>
  <tr>
    <td><strong>OS</strong></td>
    <td>FreeBSD 13.2</td>
  </tr>
  <tr>
    <td><strong>Webサーバアプリケーション</strong></td>
    <td>Apache 2.4</td>
  </tr>
  <tr>
    <td><strong>PHP</strong></td>
    <td>7.4</td>
  </tr>
</table>

PHPはちょっと古いけど、見たとき7.4だったので…  
OSがFreeBSD 13.2くらいかな変わってるのは(とはいえ、自分の公開鯖の1つはFreeBSDでやってますがw)

### 3.1. OSのインストール

VirtualBoxにFreeBSD 13.2のISOディスク[boot1](https://ftp.riken.jp/FreeBSD/releases/ISO-IMAGES/13.2/FreeBSD-13.2-RELEASE-amd64-disc1.iso)からインストール  
以下環境を作成するような奇特な方は居ないとは思いますが、もし実施する場合はディスクは大きくしてください  
初め、16GBでは足りませんでした  
おそらくllvmとrust関連のパッケージのインストールで容量食ってると思うので、エラーが起こったら都度make clean(make clean-depends)すればイケるかもしれませんが…  
また、非力なマシンだと構築までかなり時間がかかると思います

### 3-2. OSの設定

さくらのレンタルサーバの大元の設定がどうなってるのかは分からないが、上記「基本仕様が知りたい」になるべく近づけて設定してみる  
どちらかといえばPostgreSQL派だけど使えないんだ…w

### 3-2-1. インストール後の設定

#### 3-2-1-1. 時刻合わせ

```console
# cp -p /etc/ntp.conf /etc/ntp.conf.default
# vi /etc/ntp.conf
server ntp-a3.nict.go.jp iburst #この行を追加
server ntp-a2.nict.go.jp iburst #この行を追加
server ntp-b3.nict.go.jp iburst #この行を追加
server ntp-b2.nict.go.jp iburst #この行を追加
# vi /etc/rc.conf
ntpd_enable="YES"
ntpd_sync_on_start="YES"
# cp -p /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
# ntpdate ntp.nict.jp
# /etc/rc.d/ntpd start
```

#### 3-2-1-2. ユーザの作成

シェルはcshだったので、シェルの選択はcshで  
sudoとかはできなそうだったので、wheelグループには入れないでおく

```console
# adduser
Username: test
Full name: TEST
Uid (Leave empty for default): 
Login group [test]: 
Login group is test. Invite test into other groups? []:      
Login class [default]: 
Shell (sh csh tcsh nologin) [sh]: csh
Home directory [/home/test]: 
Home directory permissions (Leave empty for default): 
Use password-based authentication? [yes]: 
Use an empty password? (yes/no) [no]: 
Use a random password? (yes/no) [no]: 
Enter password: 
Enter password again: 
Lock out the account after creation? [no]: 
Username   : test
Password   : *****
Full Name  : TEST
Uid        : 1002
Class      : 
Groups     : test 
Home       : /home/test
Home Mode  : 
Shell      : /bin/csh
Locked     : no
OK? (yes/no): yes
adduser: INFO: Successfully added (test) to the user database.
Add another user? (yes/no): no
Goodbye!
```

## 4. パッケージのインストールなど

portsをgitからcloneしてmakeする。pkgを使いたかったが、2024-03-27現在、どうもphp74がpkgからでは入れられないようなので…  
実際にさくらのレンタルサーバではどうしているのかは分からないが…

### 4-1. gitのインストール

とはいえ、gitがないとgit cloneできないので、一旦pkgでgit(git-lite)をインストール

```console
# pkg install -y git-lite
```

### 4-2. gitでportsを取得

php74のインストールは[Get back PHP7.4 on FreeBSD](https://setupexample.com/install-php74-on-freebsd)を参考にする

```console
# git clone https://git.freebsd.org/ports.git /usr/ports
# cd /usr/ports
# git checkout 27ac371f93d36f77f00b8da261e496904184dd33
```

### 4-3. php74のインストール

一旦さっき入れたgit-liteと依存パッケージを全消し

```console
# pkg delete git-lite
# pkg autoremove
```

改めてphp74をインストール  
脆弱性無視のオプションを付けないと怒られるので、以下コマンドで

```console
# cd /usr/ports/lang/php74
# env BATCH=yes DISABLE_VULNERABILITIES=yes make install clean
```

### 4-4. mysql57のインストール

今回は使用しないけど、php74-pdo_mysqlが入ってるようなので  
本当はクライアントだけでいいけど…

```console
# cd /usr/ports/databases/mysql57-server
# make -DBATCH install clean
```

php74のPDOのmysqlドライバであるphp74-pdo_mysqlを入れる

```console
# cd /usr/ports/databases/php74-pdo_mysql
# make -DBATCH install clean
```
### 4-5. その他必要なPHP拡張のインストール

さくらのレンタルサーバがどこまで拡張入れてるのか分からないけど、独断と偏見で以下を入れる

```console
# cd /usr/ports/converters/php74-mbstring
# make -DBATCH install clean
# cd /usr/ports/graphics/php74-gd
# env BATCH=yes DISABLE_VULNERABILITIES=yes make install clean
# cd /usr/ports/www/php74-tidy
# make -DBATCH install clean
# cd /usr/ports/security/php74-openssl
# make -DBATCH install clean
# cd /usr/ports/devel/php74-json
# make -DBATCH install clean
# cd /usr/ports/archivers/php74-phar
# make -DBATCH install clean
# cd /usr/ports/security/php74-filter
# make -DBATCH install clean
# cd /usr/ports/archivers/php74-zlib
# make -DBATCH install clean
# cd /usr/ports/textproc/php74-dom
# make -DBATCH install clean
# cd /usr/ports/textproc/php74-xml
# make -DBATCH install clean
# cd /usr/ports/textproc/php74-xmlwriter
# make -DBATCH install clean
# cd /usr/ports/sysutils/php74-fileinfo
# make -DBATCH install clean
# cd /usr/ports/devel/php74-tokenizer
# make -DBATCH install clean
# cd /usr/ports/www/php74-session
# make -DBATCH install clean
```

### 4-6. Apacheのインストール

```console
# cd /usr/ports/www/apache24
# env BATCH=yes DISABLE_VULNERABILITIES=yes make install clean
```

## 5. Apacheの設定

### 5-1. DocumentRootの設定

今回はDocumentRootを/home/{ユーザ名}/www  
として動かすので、先ずはディレクトリを作成しておく  
3-2-1-2で作成したtestユーザで以下を作成

```console
% mkdir /home/test/www
```

### 5-2. configuration

先ずは、/usr/local/etc/apache24/httpd.conf  
の設定  
さくらのレンタルサーバでは、php-fpmで動いていた(と思う。違っていたら申し訳ございません)ので、そのように設定していく
```console
# cp -p /usr/local/etc/apache24/httpd.conf /usr/local/etc/apache24/httpd.conf.default
# vi /usr/local/etc/apache24/httpd.conf

# 以下を設定
LoadModule mpm_event_module libexec/apache24/mod_mpm_event.so # 元々無効だったのを有効に
#LoadModule mpm_prefork_module libexec/apache24/mod_mpm_prefork.so 元々有効だったのを無効に
LoadModule proxy_module libexec/apache24/mod_proxy.so # 元々無効だったのを有効に
LoadModule proxy_fcgi_module libexec/apache24/mod_proxy_fcgi.so # 元々無効だったのを有効に
LoadModule proxy_fcgi_module libexec/apache24/mod_proxy_fcgi.so # 元々無効だったのを有効に
LoadModule rewrite_module libexec/apache24/mod_rewrite.so # 元々無効だったのを有効に

# 末尾付近に
<FilesMatch "\.php$">
    SetHandler "proxy:fcgi://127.0.0.1:9000/"
</FilesMatch>

Include etc/apache24/extra/test.conf
```

次に、/usr/local/etc/apache24/extra/test.confを設定
さくらのレンタルサーバでは.htaccessは使用できるが、OptionsにAllとFollowSymLinksは使用できない
しかし実際のAllowOverrideの設定値が分かりようがないので、ここではAllにしておく  
.htaccessでFollowSymLinksを使用しなければ問題ないと思われるので

```console
# vi /usr/local/etc/apache24/extra/test.conf
DocumentRoot /home/test/www
<Directory /home/test/www>
    AllowOverride All
    Require all granted
</Directory>
```

### 5-2. 起動設定

/etc/rc.confを設定する
```console
# vi /etc/rc.conf
# 以下を追加
apache24_enable="YES"
php_fpm_enable="YES"
```

サービスを起動する
```console
# /usr/local/etc/rc.d/apache24 start
# /usr/local/etc/rc.d/php-fpm start
```

## 6. 設定確認

phpinfo.phpを作って動かしてみる  
/home/test/www/phpinfo.php
```php
<?php
date_default_timezone_set("Asia/Tokyo");
mb_internal_encoding("UTF-8");
phpinfo();
```

ブラウザでhttp://{IPまたはドメイン}/phpinfo.php  
にアクセスして、phpinfoが表示されればOK.

## 7. Laravelのインストール

### 7-1. composerのインストール

以下、testユーザで行う

```console
% cd /home/test
% php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
% php composer-setup.php
% rm composer-setup.php
% mv composer.phar composer
```

### 7-2. Laravelのインストール

本来ならここはLaravelで作成されたものを持ってくるのがいいんでしょうが、6系で作成されたものが手元になかったので、一旦6系で作ったものでDocumentRootを変えて対応する

```console
% cd /home/test
% ./composer create-project --prefer-dist laravel/laravel sakura-rental "6.*"
% chmod -R 0777 sakura-rental/storage
% chmod -R 0777 sakura-rental/bootstrap/cache
```

## 8. Laravelの設定

### 8-1. Apacheの設定

一旦、/home/test/sakura-rentalにDocumentRootを変更して、Laravelが動くか確認する

```console
# vi /usr/local/etc/apache24/extra/test.conf
#DocumentRoot /home/test/www
DocumentRoot /home/test/sakura-rental/public
#<Directory /home/test/www>
<Directory /home/test/sakura-rental/public>
    AllowOverride All
    Require all granted
</Directory>
```

Apacheとphp-fpmを再起動

```console
# /usr/local/etc/rc.d/apache24 restart
# /usr/local/etc/rc.d/php-fpm restart
```

http://{IPまたはドメイン}
でLaravelが動作することを確認

### 8-2. 簡単なページを作成

ページと呼べるほどのものではないが、以下を作成

app/Http/Controller/TestController.php
```php
<?php

namespace App\Http\Controllers;

class TestController extends Controller
{

    public function index()
    {
        return view('test');
    }
}

```

resources/views/test.blade.php
```.blade
これはテストです
```

routes/web.php
```php
Route::get('/test', 'TestController@index');
```

## 9. さくらのレンタルサーバの環境に合わせる設定

### 9-1. 準備

一旦、失敗してもいいようにコピーを作成

```console
% cd /home/test
% cp -rp sakura-rental sakura-rental-back
```

DocumentRootを/home/test/wwwに戻して、サービスを再起動する
```console
# vi /usr/local/etc/apache24/extra/test.conf
DocumentRoot /home/test/www
<Directory /home/test/www>
	AllowOverride All
	Require all granted
</Directory>
```

変更後、サービスを再起動
```console
# /usr/local/etc/rc.d/apache24 restart
# /usr/local/etc/rc.d/php-fpm restart
```

再度  
http://{IPまらはドメイン}/phpinfo.php  
などで、出力されるか確認

### 9-2. 設定

コピーする場合は、既存のファイルを消さないように気をつけてください

```console
% cd /home/test/sakura-rental
% rm -rf vendor # どこかしらから移行してきたことを想定とするため、vendor配下を削除
% ../composer install
% cd /home/test
% cp -rp sakura-rental/public/* www
% cp -p sakura-rental/public/.htaccess www
```

あとは、地道にシンボリックリンクを設定すれば取り敢えずはなんとかなりそう  
うまくいかなければ、トライアンドエラーで動かしていくしかないのかなとは思いますが…  

```console
% cd /home/test
% ln -s sakura-rental/app
% ln -s sakura-rental/bootstrap
% ln -s sakura-rental/config
% ln -s sakura-rental/database
% ln -s sakura-rental/resources
% ln -s sakura-rental/routes
% ln -s sakura-rental/storage
% ln -s sakura-rental/test
% ln -s sakura-rental/vendor
% ln -s sakura-rental/artisan
% ln -s sakura-rental/.env
% ln -s www public # php artisan serveを使用するなら設定しておいた方がいいかも。ちなみになくても動いてるようでしたが…
```

http://{IPまたはドメイン}/test  
にアクセスして確認  
自分の環境では一応動きました  
