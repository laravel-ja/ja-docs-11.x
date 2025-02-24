# Laravel Octane

- [イントロダクション](#introduction)
- [インストール](#installation)
- [サーバ要件](#server-prerequisites)
    - [FrankenPHP](#frankenphp)
    - [RoadRunner](#roadrunner)
    - [Swoole](#swoole)
- [アプリケーションの提供](#serving-your-application)
    - [HTTPSを使用するアプリケーションの提供](#serving-your-application-via-https)
    - [Nginxを使用するアプリケーションの提供](#serving-your-application-via-nginx)
    - [ファイル変更の監視](#watching-for-file-changes)
    - [ワーカ数の指定](#specifying-the-worker-count)
    - [最大リクエスト数の指定](#specifying-the-max-request-count)
    - [ワーカのリロード](#reloading-the-workers)
    - [サーバの停止](#stopping-the-server)
- [依存注入とOctane](#dependency-injection-and-octane)
    - [コンテナ注入](#container-injection)
    - [リクエストの注入](#request-injection)
    - [設定リポジトリ注入](#configuration-repository-injection)
- [メモリリークの管理](#managing-memory-leaks)
- [現在のタスク](#concurrent-tasks)
- [Tickと間隔](#ticks-and-intervals)
- [Octaneのキャッシュ](#the-octane-cache)
- [テーブル](#tables)

<a name="introduction"></a>
## イントロダクション

[Laravel Octane](https://github.com/laravel/octane)（オクタン）は、[FrankenPHP](https://frankenphp.dev/)、[Open Swoole](https://openswoole.com/)、[Swoole](https://github.com/swoole/swoole-src)、[RoadRunner](https://roadrunner.dev)などの高性能なアプリケーションサーバを使用し、アプリケーションを提供することで、アプリケーションのパフォーマンスを向上させます。Octaneはアプリケーションを一度起動したら、メモリ内に保持し、そして超音速でリクエストを送り返します。

<a name="installation"></a>
## インストール

Octaneは、Composerパッケージマネージャでインストールできます。

```shell
composer require laravel/octane
```

Octaneをインストールしたら、`octane:install` Artisanコマンドを実行してください。これにより、オクタンの設定ファイルをアプリケーションへインストールします。

```shell
php artisan octane:install
```

<a name="server-prerequisites"></a>
## サーバ要件

> [!WARNING]
> Laravel Octaneには、[PHP8.1以降](https://php.net/releases/)が必要です。

<a name="frankenphp"></a>
### FrankenPHP

[FrankenPHP](https://frankenphp.dev)はGoで書かれたPHPアプリケーションサーバです。Early hints、Brotli、Zstandard圧縮といった最新のウェブ機能をサポートしています。Octane をインストールし、FrankenPHP をサーバとして選択すると、Octaneが自動的にFrankenPHPのバイナリをダウンロードしてインストールします。

<a name="frankenphp-via-laravel-sail"></a>
#### Laravel SailでのFrankenPHPの利用

[Laravel Sail](/docs/{{version}}/sail)を使用し、アプリケーションを開発する場合は、以下のコマンドを実行し、OctaneとFrankenPHPをインストールしてください：

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane
```

次に、`octane:install` Artisanコマンドを使い、FrankenPHPバイナリをインストールします。

```shell
./vendor/bin/sail artisan octane:install --server=frankenphp
```

最後に、アプリケーションの`docker-compose.yml`ファイル内の`laravel.test`サービス定義へ、`SUPERVISOR_PHP_COMMAND`環境変数を追加します。この環境変数には、SailがPHP開発サーバの代わりにOctaneを使用してアプリケーションを提供する際に使用するコマンドを格納します。

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=frankenphp --host=0.0.0.0 --admin-port=2019 --port='${APP_PORT:-80}'" # [tl! add]
      XDG_CONFIG_HOME:  /var/www/html/config # [tl! add]
      XDG_DATA_HOME:  /var/www/html/data # [tl! add]
```

HTTPS、HTTP/2、HTTP/3を有効にするには、代わりに以下の修正をしてください。

```yaml
services:
  laravel.test:
    ports:
        - '${APP_PORT:-80}:80'
        - '${VITE_PORT:-5173}:${VITE_PORT:-5173}'
        - '443:443' # [tl! add]
        - '443:443/udp' # [tl! add]
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --host=localhost --port=443 --admin-port=2019 --https" # [tl! add]
      XDG_CONFIG_HOME:  /var/www/html/config # [tl! add]
      XDG_DATA_HOME:  /var/www/html/data # [tl! add]
```

通常、FrankenPHP Sailアプリケーションは、`https://localhost`よりアクセスします。`https://127.0.0.1`を使うには追加の設定が必要であり、[推奨していません](https://frankenphp.dev/docs/known-issues/#using-https127001-with-docker) 。

<a name="frankenphp-via-docker"></a>
#### DockerでのFrankenPHPの利用

FrankenPHPの公式Dockerイメージを使用すれば、パフォーマンスを向上させ、FrankenPHPの静的インストールには含まれていない拡張機能も使用できます。さらに、FrankenPHPの公式Dockerイメージは、 WindowsのようなFrankenPHPがネイティブにサポートしていないプラットフォームでの実行をサポートしています。FrankenPHPの公式Dockerイメージは、ローカルでの開発にも、本番環境での使用にも適しています。

FrankenPHPで動くLaravelアプリケーションをコンテナ化する出発点として、以下のDockerfileを使用してください。

```dockerfile
FROM dunglas/frankenphp

RUN install-php-extensions \
    pcntl
    # ここに他のPHP拡張…

COPY . /app

ENTRYPOINT ["php", "artisan", "octane:frankenphp"]
```

開発中はアプリケーションを実行するため、以下のDocker Composeファイルを利用してください。

```yaml
# compose.yaml
services:
  frankenphp:
    build:
      context: .
    entrypoint: php artisan octane:frankenphp --workers=1 --max-requests=1
    ports:
      - "8000:8000"
    volumes:
      - .:/app
```

`--log-level`オプションを`php artisan octane:start`コマンドへ明示的に渡した場合、OctaneはFrankenPHPのネイティブなロガーを使用し、別の設定をしない限り、構造化されたJSONログを生成します。

FrankenPHPをDockerで実行するための詳細は、[FrankenPHP公式ドキュメント](https://frankenphp.dev/docs/docker/)を参照してください。

<a name="roadrunner"></a>
### RoadRunner

[RoadRunner](https://roadrunner.dev)は、Goにより構築されたRoadRunnerバイナリで動作します。初めてRoadRunnerベースのオクタンサーバを起動するとき、OctaneはRoadRunnerバイナリをダウンロードしてインストールするよう指示します。

<a name="roadrunner-via-laravel-sail"></a>
#### Laravel SailによるRoadRunner

[Laravel Sail](/docs/{{version}}/sail)を使用してアプリケーションを開発予定の場合は、OctaneとRoadRunnerをインストールする、以下のコマンドを実行する必要があります。

```shell
./vendor/bin/sail up

./vendor/bin/sail composer require laravel/octane spiral/roadrunner-cli spiral/roadrunner-http
```

次に、Sailシェルを起動し、`rr`実行可能ファイルを使用して、RoadRunnerバイナリのLinuxベースの最新ビルドを取得します。

```shell
./vendor/bin/sail shell

# Sailシェルの中で実行
./vendor/bin/rr get-binary
```

次に、アプリケーションの`docker-compose.yml`ファイル内の`laravel.test`サービス定義へ、`SUPERVISOR_PHP_COMMAND`環境変数を追加します。この環境変数には、SailがPHP開発サーバの代わりにOctaneを使用してアプリケーションを提供する際に使用するコマンドを格納します。

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=roadrunner --host=0.0.0.0 --rpc-port=6001 --port='${APP_PORT:-80}'" # [tl! add]
```

最後に、`rr`バイナリが実行可能であることを確認し、Sailイメージを構築してください:

```shell
chmod +x ./rr

./vendor/bin/sail build --no-cache
```

<a name="swoole"></a>
### Swoole

Swooleアプリケーションサーバを使用してLaravel Octaneアプリケーションを提供する予定の場合は、Swool PHP拡張をインストールする必要があります。通常、これはPECLで行います。

```shell
pecl install swoole
```

<a name="openswoole"></a>
#### Open Swoole

Open Swooleアプリケーションサーバを使用してLaravel Octaneアプリケーションを提供したい場合、Open Swoole PHP拡張をインストールする必要があります。一般的に、これはPECLを介して行えます。

```shell
pecl install openswoole
```

Laravel OctaneとOpen Swooleを併用することで、同時並行タスク、tick、intervalなどSwooleが提供する機能と同じものが使えるようになります。

<a name="swoole-via-laravel-sail"></a>
#### Laravel Sailを使用するSwoole

> [!WARNING]
> Sailを介してOctaneアプリケーションを動作させる前に、最新バージョンのLaravel Sailであることを確認し、アプリケーションのルートディレクトリ内で`./vendor/bin/sail build --no-cache`を実行してください。

あるいは、Laravelの公式Dockerベース開発環境である[Laravel Sail](/docs/{{version}}/sail)を使用して、SwooleベースのOctaneアプリケーションを開発することもできます。Laravel SailにはデフォルトでSwooleエクステンションが含まれています。ただし、Sailが使用する`docker-compose.yml`ファイルを調整する必要があります。

これを使用するには、アプリケーションの`docker-compose.yml`ファイル内の`laravel.test`サービス定義へ、`SUPERVISOR_PHP_COMMAND`環境変数を追加します。この環境変数には、SailがPHP開発サーバの代わりにOctaneを使用してアプリケーションを提供する際に使用するコマンドを格納します。

```yaml
services:
  laravel.test:
    environment:
      SUPERVISOR_PHP_COMMAND: "/usr/bin/php -d variables_order=EGPCS /var/www/html/artisan octane:start --server=swoole --host=0.0.0.0 --port='${APP_PORT:-80}'" # [tl! add]
```

最後に、Sailイメージを構築します。

```shell
./vendor/bin/sail build --no-cache
```

<a name="swoole-configuration"></a>
#### Swoole設定

Swooleは追加設定オプションをサポートしており、必要に応じて`octane`設定ファイルへ追加できます。これらのオプションはほとんど変更する必要がないため、デフォルトのコンフィギュレーションファイルには含まれていません。

```php
'swoole' => [
    'options' => [
        'log_file' => storage_path('logs/swoole_http.log'),
        'package_max_length' => 10 * 1024 * 1024,
    ],
],
```

<a name="serving-your-application"></a>
## アプリケーションの提供

Octaneサーバは`octane:start` Artisanコマンドで起動できます。このコマンドは、デフォルトでアプリケーションの`octane`設定ファイルの`server`設定オプションで指定したサーバを利用します。

```shell
php artisan octane:start
```

Octaneはデフォルトで、ポート8000​​のサーバを起動するので、Webブラウザから`http://localhost:8000`で、アプリケーションへアクセスできます。

<a name="serving-your-application-via-https"></a>
### HTTPSを使用するアプリケーションの提供

デフォルトでは、Octaneを介して実行されているアプリケーションは、`http://`を先頭につけたリンクを生成します。アプリケーションの`config/octane.php`設定ファイル内で使用されている`OCTANE_HTTPS`環境変数は、HTTPS経由でアプリケーションのサービスを提供する場合、`true`と設定します。この設定値が`true`に設定されている場合、OctaneはLaravelに、生成するすべてのリンクに`https://`を先頭に付けるよう指示します。

```php
'https' => env('OCTANE_HTTPS', false),
```

<a name="serving-your-application-via-nginx"></a>
### Nginxを使用するアプリケーションの提供

> [!NOTE]
> あなた自身のサーバ設定を管理すること、または堅牢なLaravel Octaneアプリケーションを実行するのに必要なさまざまなサービスをすべて設定するのに慣れていない場合は、[Laravel Forge](https://forge.laravel.com)の使用を考慮してください。

本番環境では，NginxやApacheのような伝統的なWebサーバの背後で、Octaneアプリケーションを提供するべきです。そうすることでWebサーバは，画像やスタイルシートなどの静的資産を提供でき，またSSL証明書のターミネーションを管理できます。

下記のNginx設定例では、Nginxはサイトの静的アセットを提供し、ポート8000​​で実行されるOctanサーバへのプロキシリクエストを提供します。

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    listen [::]:80;
    server_name domain.com;
    server_tokens off;
    root /home/forge/domain.com/public;

    index index.php;

    charset utf-8;

    location /index.php {
        try_files /not_exists @octane;
    }

    location / {
        try_files $uri $uri/ @octane;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    access_log off;
    error_log  /var/log/nginx/domain.com-error.log error;

    error_page 404 /index.php;

    location @octane {
        set $suffix "";

        if ($uri = /index.php) {
            set $suffix ?$query_string;
        }

        proxy_http_version 1.1;
        proxy_set_header Host $http_host;
        proxy_set_header Scheme $scheme;
        proxy_set_header SERVER_PORT $server_port;
        proxy_set_header REMOTE_ADDR $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_pass http://127.0.0.1:8000$suffix;
    }
}
```

<a name="watching-for-file-changes"></a>
### ファイル変更の監視

Octane Serverが起動するとアプリケーションがメモリにロードされているので、ブラウザを更新してもアプリケーションのファイルの変更は反映されません。たとえば、`routes/web.php`ファイルに追加されたルート定義は、サーバが再起動されるまで反映されません。利便性を上げるため、アプリケーション内のいずれかのファイルの変更でサーバを自動的に再起動するように、`--watch`フラグを使用してOctaneへ指示できます。

```shell
php artisan octane:start --watch
```

この機能を使用する前に、[Node](https://nodejs.org)がローカル開発環境内にインストールされていることを確認する必要があります。さらに、プロジェクト内に[Chokidar](https://github.com/paulmillr/chokidar)ファイル監視ライブラリをインストールする必要もあります。

```shell
npm install --save-dev chokidar
```

アプリケーションの`config/octane.php`設定ファイル内の`watch`設定オプションを使用して、監視対象のディレクトリとファイルを設定できます。

<a name="specifying-the-worker-count"></a>
### ワーカ数の指定

Octaneはデフォルトで、マシンの各CPUコアごとにアプリケーションリクエストワーカを起動します。こうしたワーカは、アプリケーションがHTTPリクエストを受信するサーバとして動作します。`octane:start`コマンドを呼び出すときの`--workers`オプションにより、いくつの数のワーカをはじめに起動するか指定できます。

```shell
php artisan octane:start --workers=4
```

Swooleアプリケーションサーバを使用している場合は、[「タスクワーカ」](#concurrent-tasks)の数を指定することもできます。

```shell
php artisan octane:start --workers=4 --task-workers=6
```

<a name="specifying-the-max-request-count"></a>
### 最大リクエスト数の指定

メモリリークを防ぐために、Octaneは500リクエストを処理した時点でワーカを再起動します。この回数を調整するには、`--max-requests`オプションを使います。

```shell
php artisan octane:start --max-requests=250
```

<a name="reloading-the-workers"></a>
### ワーカのリロード

`octane:reload`コマンドを使用して、Octaneサーバのアプリケーションワーカを穏やかに再起動することができます。通常、これはデプロイ後に行われ、新しくデプロイしたコードがメモリへロードされ、後続のリクエストから使用されます。

```shell
php artisan octane:reload
```

<a name="stopping-the-server"></a>
### サーバの停止

`octane:stop` Artisanコマンドで、Octaneサーバを停止できます。

```shell
php artisan octane:stop
```

<a name="checking-the-server-status"></a>
#### サーバ状態の確認

`octane:status` Artisanコマンドで、Octaneサーバの現在のステータスを確認できます。

```shell
php artisan octane:status
```

<a name="dependency-injection-and-octane"></a>
## 依存注入とOctane

Octaneは一度あなたのアプリケーションを起動したら、リクエストを処理しながらメモリに常駐するので、アプリケーション構築中に考慮すべき警告がいくつかあります。たとえば、アプリケーションのサービスプロバイダの`register`と`boot`メソッドは、リクエストワーカが最初に起動したときにのみ実行されます。後続のリクエストでは、同じアプリケーションインスタンスが再利用されます。

このことから、アプリケーションサービスのコンテナやリクエストをオブジェクトのコンストラクタへ注入する際には、特に注意が必要になります。これにより後続のリクエストで、そのオブジェクトはコンテナやリクエストの古いバージョンを持つことになります。

Octaneは、リクエスト間でのファーストパーティフレームワークの状態のリセットを自動的に処理します。しかし、Octaneは、アプリケーションが作成する、グローバルな状態をリセットする方法を常に知っているわけでありません。したがって，どのようにしてあなたのアプリケーションをOctaneに適合するように構築するか認識しておく必要があります。以下では，Octaneを使用する際に問題となる可能性のある最も一般的な状況について説明します。

<a name="container-injection"></a>
### コンテナ注入

一般的に、アプリケーションサービスコンテナやHTTPリクエストインスタンスを他のオブジェクトのコンストラクタへ注入するのは避けるべきです。たとえば、以下の結合では、アプリケーションサービスコンテナ全体がシングルトンとして結合するオブジェクトへ注入されます。

```php
use App\Service;
use Illuminate\Contracts\Foundation\Application;

/**
 * 全アプリケーションサービスの登録
 */
public function register(): void
{
    $this->app->singleton(Service::class, function (Application $app) {
        return new Service($app);
    });
}
```

この例では、アプリケーションの起動プロセス中に`Service`インスタンスが解決されると、コンテナがサービスに注入され、その後のリクエストでも同じコンテナが`Service`インスタンスで保持されます。これは、特定のアプリケーションでは問題に**ならないかもしれません**が、起動サイクルの後半や後続のリクエストで追加される結合を、コンテナが予期せずに見落としてしまう可能性があります。

回避策は、結合をシングルトンとして登録しないか、現在のコンテナインスタンスを常に解決するコンテナリゾルバクロージャをサービスへ注入することです。

```php
use App\Service;
use Illuminate\Container\Container;
use Illuminate\Contracts\Foundation\Application;

$this->app->bind(Service::class, function (Application $app) {
    return new Service($app);
});

$this->app->singleton(Service::class, function () {
    return new Service(fn () => Container::getInstance());
});
```

グローバルな`app`ヘルパと`Container::getInstance()`メソッドは、常にアプリケーションコンテナの最新バージョンを返します。

<a name="request-injection"></a>
### リクエストの注入

一般的に、アプリケーションサービスコンテナやHTTPリクエストインスタンスを他のオブジェクトのコンストラクタへ注入するのは避けるべきです。たとえば、以下の結合では、シングルトンとして結合したオブジェクトへリクエストインスタンス全体が注入されます。

```php
use App\Service;
use Illuminate\Contracts\Foundation\Application;

/**
 * 全アプリケーションサービスの登録
 */
public function register(): void
{
    $this->app->singleton(Service::class, function (Application $app) {
        return new Service($app['request']);
    });
}
```

この例では、アプリケーションのプロセス時に`Service`インスタンスが解決されると、HTTPリクエストがサービスへ注入され、以降のリクエストでも`Service`インスタンスが同じリクエストを保持することになります。したがって、すべてのヘッダ、入力、およびクエリ文字列のデータは、他のすべてのリクエストデータと同様におかしくなります。

回避策は、結合をシングルトンとして登録するのをやめるか、現在のリクエストインスタンスを常に解決するリクエストリゾルバクロージャをサービスへ注入することです。あるいは、最も推奨される方法は、単純に、そのオブジェクトが必要とする特定のリクエスト情報を、実行時にオブジェクトのメソッドの１つへ渡すことです。

```php
use App\Service;
use Illuminate\Contracts\Foundation\Application;

$this->app->bind(Service::class, function (Application $app) {
    return new Service($app['request']);
});

$this->app->singleton(Service::class, function (Application $app) {
    return new Service(fn () => $app['request']);
});

// もしくは

$service->method($request->input('name'));
```

グローバルな`request`ヘルパは、常にアプリケーションが現在処理しているリクエストを返すので、アプリケーション内で安全に使用できます。

> [!WARNING]
> コントローラのメソッドやルートクロージャで、`Illuminate\Http\Request`インスタンスをタイプヒントしても構いません。

<a name="configuration-repository-injection"></a>
### 設定リポジトリ注入

一般的には、設定リポジトリのインスタンスを他のオブジェクトのコンストラクタに注入するのは避けるべきです。たとえば、次の結合は、シングルトンとして結合しているオブジェクトへ設定リポジトリを注入しています。

```php
use App\Service;
use Illuminate\Contracts\Foundation\Application;

/**
 * 全アプリケーションサービスの登録
 */
public function register(): void
{
    $this->app->singleton(Service::class, function (Application $app) {
        return new Service($app->make('config'));
    });
}
```

この例では、リクエスト間で設定値が変更された場合、そのサービスは元のリポジトリインスタンスに依存しているため、新しい値にアクセスできません。

回避策は、結合をシングルトンとして登録するのをやめるか、設定リポジトリのリゾルバクロージャをクラスへ注入することです。

```php
use App\Service;
use Illuminate\Container\Container;
use Illuminate\Contracts\Foundation\Application;

$this->app->bind(Service::class, function (Application $app) {
    return new Service($app->make('config'));
});

$this->app->singleton(Service::class, function () {
    return new Service(fn () => Container::getInstance()->make('config'));
});
```

グローバルな`config`は、常に最新バージョンの設定リポジトリを返すので、アプリケーション内で安全に使用できます。

<a name="managing-memory-leaks"></a>
### メモリリークの管理

Octaneはリクエストの間中、アプリケーションをメモリ内で常駐させることを忘れないでください。そのため、静的に保持している配列へデータを追加していくと，メモリリークが発生します。例えば，以下のコントローラは，アプリケーションへの各リクエストが静的な`$data`配列にデータを追加し続けるので，メモリリークが発生します。

```php
use App\Service;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

/**
 * 受信リクエストの処理
 */
public function index(Request $request): array
{
    Service::$data[] = Str::random(10);

    return [
        // ...
    ];
}
```

アプリケーションを構築する際は、このようなメモリリークを発生させないよう、特に注意する必要があります。ローカル開発時に、アプリケーションのメモリ使用量を監視し、アプリケーションに新たなメモリリークが発生するのを防ぐことを推奨します。

<a name="concurrent-tasks"></a>
## 現在のタスク

> [!WARNING]
> この機能は[Swoole](#swoole)が必要です。

Swooleを使用している場合，軽量のバックグラウンドタスクを介して，複数操作を同時に実行できます。これには，Octaneの`concurrently`メソッドを使用します。このメソッドとPHP配列のデストラクションを組み合わせて，各操作の結果を取得できます。

```php
use App\Models\User;
use App\Models\Server;
use Laravel\Octane\Facades\Octane;

[$users, $servers] = Octane::concurrently([
    fn () => User::all(),
    fn () => Server::all(),
]);
```

Octaneが処理する同時タスクは、Swooleの「タスクワーカ」を利用しており、受信リクエストとは全く別のプロセスで実行されます。同時タスクを処理するために利用できるワーカの数は、`octane:start`コマンドの`--task-workers`ディレクティブで決めます。

```shell
php artisan octane:start --workers=4 --task-workers=6
```

`concurrently`メソッドを呼び出す場合、Swoole のタスクシステムによる制限のため、1024個以上のタスクを指定してはいけません。

<a name="ticks-and-intervals"></a>
## Tickと間隔

> [!WARNING]
> この機能は[Swoole](#swoole)が必要です。

Swooleでは、指定した秒数ごとに実行される"tick"オペレーションが登録できます。"tick"コールバックの登録には、`tick`メソッドを使用します。`tick`メソッドの第１引数は、ティッカー(Ticker)の名前を表す文字列を指定します。２番目の引数は、指定した間隔で起動するコールバックを指定します。

以下の例では，10秒ごとに呼び出すクロージャを登録しています。通常`tick`メソッドは、アプリケーションのサービスプロバイダの`boot`メソッドの中で呼び出します。

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
    ->seconds(10);
```

`immediate`メソッドを使用すると，Octaneサーバが最初に起動したときに，直ちにtickコールバックを起動し，その後はN秒ごとに起動するように指示できます。

```php
Octane::tick('simple-ticker', fn () => ray('Ticking...'))
    ->seconds(10)
    ->immediate();
```

<a name="the-octane-cache"></a>
## Octaneのキャッシュ

> [!WARNING]
> この機能は[Swoole](#swoole)が必要です。

Swooleを使用する際には、最大２００万回／秒の読み取り／書き込み速度を実現するOctaneキャッシュドライバが活用できます。したがって、このキャッシュドライバは、キャッシング層からの極端なリード／ライト速度を必要とするアプリケーションに最適な選択肢です。

このキャッシュドライバは、[Swooleテーブル](https://www.swoole.co.uk/docs/modules/swoole-table)を利用しています。キャッシュに保存したデータは、サーバ上のすべてのワーカが利用できます。ただし、キャッシュされたデータは、サーバが再起動されるとフラッシュされます。

```php
Cache::store('octane')->put('framework', 'Laravel', 30);
```

> [!NOTE]
> Octaneキャッシュで許可するエントリの最大数は，アプリケーションの`octane`設定ファイルで定義できます。

<a name="cache-intervals"></a>
### キャッシュ間隔

Laravelのキャッシュシステムが提供する典型的な手法に加え、Octaneキャッシュドライバはインターバルベースのキャッシュを備えています。これらのキャッシュは指定された間隔で自動的にリフレッシュされ、アプリケーションのサービスプロバイダで`boot`メソッド内に登録する必要があります。例えば、以下のキャッシュは５秒ごとにリフレッシュされます。

```php
use Illuminate\Support\Str;

Cache::store('octane')->interval('random', function () {
    return Str::random(10);
}, seconds: 5);
```

<a name="tables"></a>
## テーブル

> [!WARNING]
> この機能は[Swoole](#swoole)が必要です。

Swooleを使用する場合は、任意に独自の[Swooleテーブル](https://www.swoole.co.uk/docs/modules/swoole-table)を定義し、操作できます。Swooleテーブルは、非常に高いパフォーマンスのスループットを提供し、これらのテーブルのデータは、サーバ上のすべてのワーカからアクセスできます。ただし、サーバを再起動するとテーブル内のデータは失われます。

テーブルは、アプリケーションの`octane`設定ファイルの`tables`設定配列で定義します。テーブルの例として、最大１０００行のテーブルが設定済みです。文字列カラムの最大サイズを設定するには、以下のようにカラムタイプの後にカラムサイズを指定します。

```php
'tables' => [
    'example:1000' => [
        'name' => 'string:1000',
        'votes' => 'int',
    ],
],
```

テーブルへアクセスするには、`Octane::table`メソッドを使います。

```php
use Laravel\Octane\Facades\Octane;

Octane::table('example')->set('uuid', [
    'name' => 'Nuno Maduro',
    'votes' => 1000,
]);

return Octane::table('example')->get('uuid');
```

> [!WARNING]
> Swooleのテーブルがサポートする、カラムの型は`string`、`int`、`float`です。
