# デプロイ

- [イントロダクション](#introduction)
- [サーバ要件](#server-requirements)
- [サーバ設定](#server-configuration)
    - [Nginx](#nginx)
    - [FrankenPHP](#frankenphp)
    - [ディレクトリパーミッション](#directory-permissions)
- [最適化](#optimization)
    - [設定のキャッシュ](#optimizing-configuration-loading)
    - [イベントのキャッシュ](#caching-events)
    - [ルートのキャッシュ](#optimizing-route-loading)
    - [ビューのキャッシュ](#optimizing-view-loading)
- [デバッグモード](#debug-mode)
- [ヘルスルート](#the-health-route)
- [Forge／Vaporを利用する簡単なデプロイ](#deploying-with-forge-or-vapor)

<a name="introduction"></a>
## イントロダクション

Laravelアプリケーションをプロダクションとしてデプロイする準備ができたら、アプリケーションをできるだけ確実かつ、効率的な実行を行うには、いくつか重要な手順を行う必要があります。このドキュメントでは、アプリケーションを確実にデプロイするため、重要なポイントを説明します。

<a name="server-requirements"></a>
## サーバ要件

Laravelフレームワークにはいくつかのシステム要件があります。Webサーバへ確実に以下のPHP最低バージョンと拡張機能を用意してください。

<div class="content-list" markdown="1">

- PHP8.2以上
- Ctype PHP拡張
- cURL PHP拡張
- DOM PHP拡張
- Fileinfo PHP拡張
- Filter PHP拡張
- Hash PHP拡張
- Mbstring PHP拡張
- OpenSSL PHP拡張
- PCRE PHP拡張
- PDO PHP拡張
- Session PHP拡張
- Tokenizer PHP拡張
- XML PHP拡張

</div>

<a name="server-configuration"></a>
## サーバ設定

<a name="nginx"></a>
### Nginx

Nginxを実行しているサーバにアプリケーションをデプロイする場合は、以下の設定ファイルをWebサーバを設定するためのサンプルとして使用できます。大抵の場合、このファイルはサーバの設定に応じてカスタマイズする必要があります。**サーバの管理についてサポートが必要な場合は、[Laravel Forge](https://forge.laravel.com)などのファーストパーティのLaravelサーバ管理とデプロイサービスの使用を検討してください。**

以下の設定のように、Webサーバがすべてのリクエストをアプリケーションの`public/index.php`ファイルへ確実に送信してください。プロジェクトルートからアプリケーションを提供すると、多くの機密性の高い設定ファイルがパブリックインターネットに公開されるため、`index.php`ファイルをプロジェクトのルートに移動しようとしないでください。

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com;
    root /srv/example.com/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

<a name="frankenphp"></a>
### FrankenPHP

[FrankenPHP](https://frankenphp.dev/)は、Laravelアプリケーションのサーバとしても使用できます。FrankenPHPはGoで書かれたモダンなPHPアプリケーションサーバです。FrankenPHPを利用してLaravelのPHPアプリケーションを提供するには、`php-server`コマンドを実行するだけです：

```shell
frankenphp php-server -r public/
```

[Laravel Octane](/docs/{{version}}/octane)統合、HTTP/3、最新の圧縮、Laravelアプリケーションをスタンドアロンバイナリとしてパッケージ化する機能など、FrankenPHPがサポートするより強力な機能を利用するには、FrankenPHPの[Laravelドキュメント](https://frankenphp.dev/docs/laravel/)を参照してください。

<a name="directory-permissions"></a>
### ディレクトリパーミッション

Laravelは、`bootstrap/cache`と`storage`ディレクトリに書き込む必要があるため、Webサーバのプロセスオーナーがこれらのディレクトリへの書き込み権限を持っていることを確認してください。

<a name="optimization"></a>
## 最適化

アプリケーションを本番環境にデプロイする際は、設定、イベント、ルート、ビューなど、キャッシュすべき様々なファイルがあります。Laravelはこれらのファイルをすべてキャッシュする便利な`optimize` Artisanコマンドを提供しています。このコマンドは通常、アプリケーションのデプロイプロセスの一部として起動する必要があります：

```shell
php artisan optimize
```

`optimize:clear`メソッドは、`optimize`コマンドが生成したキャッシュファイルをすべて削除するために使います。

```shell
php artisan optimize:clear
```

以下の文書では、`optimize`コマンドが実行する、最適化コマンドの詳細それぞれについて説明します。

<a name="optimizing-configuration-loading"></a>
### 設定のキャッシュ

アプリケーションをプロダクションへデプロイする場合、デプロイプロセスの中で、確実に`config:cache` Artisanコマンドを実行してください。

```shell
php artisan config:cache
```

このコマンドは、Laravelの全設定ファイルをキャッシュされる一つのファイルへまとめるため、設定値をロードする場合に、フレームワークがファイルシステムを数多くアクセスする手間を大いに減らします。

> [!WARNING]
> 開発時に`config:cache`コマンドを実行する場合は、設定ファイルの中だけで、`env`関数を呼び出していることを確認してください。設定ファイルがキャッシュされてしまうと、`.env`ファイルはロードされなくなり、`.env`変数に対する`env`関数の呼び出し結果はすべて`null`になります。

<a name="caching-events"></a>
### イベントのキャッシュ

デプロイ処理中で、アプリケーションの自動検出イベントとリスナのマッピングをキャッシュする必要があります。これは、デプロイ中に`event:cache` Artisan コマンドを呼び出すことで行えます。

```shell
php artisan event:cache
```

<a name="optimizing-route-loading"></a>
### ルートのキャッシュ

多くのルートを持つ大きなアプリケーションを構築した場合、デプロイプロセス中に、`route:cache` Artisanコマンドを確実に実行すべきでしょう。

```shell
php artisan route:cache
```

このコマンドはキャッシュファイルの中の、一つのメソッド呼び出しへ全ルート登録をまとめるため、数百のルートを登録する場合、ルート登録のパフォーマンスを向上します。

<a name="optimizing-view-loading"></a>
### ビューのキャッシュ

実機環境へアプリケーションをデプロイする場合は、その手順の中で`view:cache` Artisanコマンドを実行すべきでしょう。

```shell
php artisan view:cache
```

このコマンドは全Bladeビューを事前にコンパイルし、要求ごとにコンパイルしなくて済むため、ビューを返すリクエストすべてでパフォーマンスが向上します。

<a name="debug-mode"></a>
## デバッグモード

`config/app.php`設定ファイルのdebugオプションは、エラーに関する情報を実際にユーザーへ表示する量を決定します。このオプションはデフォルトで、アプリケーションの`.env`ファイルに格納している、`APP_DEBUG`環境変数の値を利用するように設定されています。

> [!WARNING]
> **実稼働環境下では、この値は常に`false`である必要があります。本番環境で`APP_DEBUG`変数が`true`に設定されていると、機密性の高い設定値がアプリケーションのエンドユーザーに公開されるリスクがあります。**

<a name="the-health-route"></a>
## ヘルスルート

Laravelは、アプリケーションのステータスを監視するために使用できる組み込みのヘルスチェックルートを用意しています。本番環境では、このルートを使用して、アプリケーションの状態をアップタイムモニタ、ロードバランサ、またはKubernetesのようなオーケストレーションシステムへ報告できます。

ヘルスチェックのルートはデフォルトで、`/up`で提供しており、アプリケーションが例外なく起動した場合は 200 HTTPレスポンスを返します。そうでない場合は、500 HTTPレスポンスを返します。このルートのURIは、アプリケーションの`bootstrap/app`ファイルで設定できます。

    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up', // [tl! remove]
        health: '/status', // [tl! add]
    )

HTTPリクエストがこのルートへ行われると、Laravelは `Illuminate\Foundation\Events\DiagnosingHealth`イベントもディスパッチするため、アプリケーションに関連する追加のヘルスチェックを実行できます。このイベントの[リスナ](/docs/{{version}}/events)内で、アプリケーションのデータベースやキャッシュの状態をチェックできます。アプリケーションに問題が見つかった場合は、リスナから例外を投げるだけです。

<a name="deploying-with-forge-or-vapor"></a>
## Forge／Vaporを利用する簡単なデプロイ

<a name="laravel-forge"></a>
#### Laravel Forge

自分のサーバ設定管理に準備不足であったり、堅牢なLaravelアプリケーション実行に必要な数多くのサービスすべての設定について慣れていなければ、[Laravel Forge](https://forge.laravel.com)は素晴らしい代替案です。

Laravel ForgeはDigitalOcean、Linode、AWSなど数多くのインフラプロバイダ上に、サーバを作成できます。それに加え、ForgeはNginx、MySQL、Redis、Memcached、Beanstalkなどのような、堅牢なLaravelアプリケーションを構築するために必要なツールを全部インストールし、管理します。

> [!NOTE]
> Laravel Forgeでデプロイするための完全なガイドが必要ですか？[Laravel Bootcamp](https://bootcamp.laravel.com/deploying)と[Laracastsで視聴できるForgeのビデオシリーズ](https://laracasts.com/series/learn-laravel-forge-2022-edition)をご覧ください。

<a name="laravel-vapor"></a>
#### Laravel Vapor

Laravel向け完全サーバレスのオートスケーリング開発プラットフォームが必要な場合は、[Laravel Vapor](https://vapor.laravel.com)をチェックしてください。Laravel Vaporは、AWSで動作するLaravelのサーバレス開発プラットフォームです。Vapor上でLaravelインフラを起動し、サーバレスのスケーラブルなシンプルさに魅了されてください。Laravel Vaporは、Laravelの作成者によりフレームワークとシームレスに連携するように調整されているため、普段通りにLaravelアプリケーションを書き続けられます。
