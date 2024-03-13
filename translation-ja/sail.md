# Laravel Sail

- [イントロダクション](#introduction)
- [インストールと準備](#installation)
    - [既存アプリケーションへのSailインストール](#installing-sail-into-existing-applications)
    - [シェルエイリアスの設定](#configuring-a-shell-alias)
- [Sailの開始と停止](#starting-and-stopping-sail)
- [コマンドの実行](#executing-sail-commands)
    - [PHPコマンドの実行](#executing-php-commands)
    - [Composerコマンドの実行](#executing-composer-commands)
    - [Arttisanコマンドの実行](#executing-artisan-commands)
    - [Node/NPMコマンドの実行](#executing-node-npm-commands)
- [データベース操作](#interacting-with-sail-databases)
    - [MySQL](#mysql)
    - [Redis](#redis)
    - [Meilisearch](#meilisearch)
    - [Typesense](#typesense)
- [ファイルストレージ](#file-storage)
- [テスト実行](#running-tests)
    - [Laravel Dusk](#laravel-dusk)
- [メールのプレビュー](#previewing-emails)
- [コンテナCLI](#sail-container-cli)
- [PHPバージョン](#sail-php-versions)
- [Nodeバージョン](#sail-node-versions)
- [サイトの共有](#sharing-your-site)
- [Xdebugによるデバッグ](#debugging-with-xdebug)
  - [Xdebug CLI使用法](#xdebug-cli-usage)
  - [Xdebug ブラウザ使用法](#xdebug-browser-usage)
- [カスタマイズ](#sail-customization)

<a name="introduction"></a>
## イントロダクション

[Laravel Sail](https://github.com/laravel/sail)（セイル、帆、帆船）は、LaravelのデフォルトのDocker開発環境を操作するための軽量コマンドラインインターフェイスです。 Sailは、Dockerの経験がなくても、PHP、MySQL、Redisを使用してLaravelアプリケーションを構築するための優れた出発点を提供します。

Sailの本質は、`docker-compose.yml`ファイルとプロジェクトのルートに保存されている`sail`スクリプトです。`sail`スクリプトは、`docker-compose.yml`ファイルで定義されたDockerコンテナを操作するための便利なメソッドをCLIで提供します。

Laravel Sailは、macOS、Linux、Windows（[WSL2](https://docs.microsoft.com/en-us/windows/wsl/about)を使用）に対応しています。

<a name="installation"></a>
## インストールと準備

Laravel Sailはいつも、新しいLaravelアプリケーションとともに自動的にインストールされるため、すぐ使用を開始できます。新しいLaravelアプリケーションを作成する方法については、使用しているオペレーティングシステム用のLaravel[インストールドキュメント](/docs/{{version}}/installation#docker-installation-using-sail)を参照してください。インストール時には、アプリケーションで操作するSailサポートサービスを選択するよう求められます。

<a name="installing-sail-into-existing-applications"></a>
### 既存アプリケーションへのSailインストール

既存のLaravelアプリケーションでSailを使うことに興味を持っている場合は、Composerパッケージマネージャを使用してSailをインストールできます。もちろん、以降の手順は、既存のローカル開発環境で、Composerの依存関係をインストールできると仮定しています。

```shell
composer require laravel/sail --dev
```

Sailをインストールしたら、`sail:install` Artisanコマンドを実行します。このコマンドはSailの`docker-compose.yml`ファイルをアプリケーションのルートに発行し、`.env`ファイル中でDockerサービスに接続するために必要な環境変数を変更します：

```shell
php artisan sail:install
```

最後に、Sailを立ち上げましょう。Sailの使い方を学び続けるには、このドキュメントの残りを続けてお読みください。

```shell
./vendor/bin/sail up
```

> [!WARNING]
> Linux用のDocker Desktopを使用している場合は、`docker context use default`コマンドを実行して、`default` Dockerコンテキストを使用する必要があります。

<a name="adding-additional-services"></a>
#### サービスの追加

既存のSailのインストールにサービスを追加したい場合は、`sail:add` Artisanコマンドを実行してください。

```shell
php artisan sail:add
```

<a name="using-devcontainers"></a>
#### Devcontainerの使用

もし、[Devcontainer](https://code.visualstudio.com/docs/remote/containers)内で開発したい場合は、`--devcontainer`オプションを`sail:install`コマンドで指定してください。`--devcontainer`オプションは、`sail:install`コマンドに、デフォルトの`.devcontainer/devcontainer.json`ファイルをアプリケーションのルートにリソース公開するように指示します。

```shell
php artisan sail:install --devcontainer
```

<a name="configuring-a-shell-alias"></a>
### シェルエイリアスの設定

デフォルトでSailコマンドは、すべての新しいLaravelアプリケーションに含まれている`vendor/bin/sail`スクリプトを使用して起動します。

```shell
./vendor/bin/sail up
```

しかし、Sailコマンドを実行するため、`vendor/bin/sail`、と繰り返しタイプする代わりに、Sailのコマンドをより簡単に実行できるようなシェルエイリアスを設定したいと思うでしょう。

```shell
alias sail='sh $([ -f sail ] && echo sail || echo vendor/bin/sail)'
```

これを常に利用できるようにするには、ホームディレクトリにあるシェルの設定ファイル、例えば `~/.zshrc` や `~/.bashrc` へ追加後、シェルを再起動するとよいでしょう。

シェルエイリアスを設定すると、`sail`と入力するだけで、Sailコマンドを実行できます。このドキュメントの残りの例では、このエイリアスを設定済みであることを前提にしています。

```shell
sail up
```

<a name="starting-and-stopping-sail"></a>
## Sailの開始と停止

Laravel Sailの`docker-compose.yml`ファイルは、Laravelアプリケーションの構築を支援するために連携するさまざまなDockerコンテナを定義します。これらの各コンテナは、`docker-compose.yml`ファイルの`services`設定内のエントリです。`laravel.test`コンテナは、アプリケーションを提供するメインのアプリケーションコンテナです。

Sailを開始する前に、ローカルコンピューターで他のWebサーバまたはデータベースが実行されていないことを確認する必要があります。アプリケーションの`docker-compose.yml`ファイルで定義されているすべてのDockerコンテナを起動するには、`up`コマンドを実行する必要があります。

```shell
sail up
```

すべてのDockerコンテナをバックグラウンドで起動するには、Sailを「デタッチdetached)」モードで起動します。

```shell
sail up -d
```

アプリケーションのコンテナが開始されると、Webブラウザ（http:// localhost）でプロジェクトにアクセスできます。

To stop all of the containers, you may simply press Control + C to stop the container's execution. Or, if the containers are running in the background, you may use the `stop` command:

```shell
sail stop
```

<a name="executing-sail-commands"></a>
## コマンドの実行

Laravel Sailを使用する場合、アプリケーションはDockerコンテナ内で実行され、ローカルコンピューターから分離されます。さらにSailは、任意のPHPコマンド、Artisanコマンド、Composerコマンド、Node / NPMコマンドなど、アプリケーションに対してさまざまなコマンドを実行するための便利な方法も提供します。

**Laravelのドキュメントを読むと、Sailを参照しないComposer、Artisan、Node／NPMコマンドの参照をよく目にするでしょう。**こうした実行例は、これらのツールがローカルコンピューターにインストールされていることを前提としています。ローカルのLaravel開発環境にSailを使用している場合は、Sailを使用してこれらのコマンドを実行する必要があります。

```shell
# Artisanコマンドをローカル環境で実行
php artisan queue:work

# Laravel Sailの中でArtisanコマンドを実行
sail artisan queue:work
```

<a name="executing-php-commands"></a>
### PHPコマンドの実行

PHPコマンドは、`php`コマンドを使用して実行します。もちろん、これらのコマンドは、アプリケーション用に構成されたPHPバージョンを使用して実行されます。Laravel Sailで利用可能なPHPバージョンの詳細については、[PHPバージョンのドキュメント](#sail-php-versions)を参照してください。

```shell
sail php --version

sail php script.php
```

<a name="executing-composer-commands"></a>
### Composerコマンドの実行

Composer commands may be executed using the `composer` command. Laravel Sail's application container includes a Composer installation:

```nothing
sail composer require laravel/sanctum
```

<a name="installing-composer-dependencies-for-existing-projects"></a>
#### 既存アプリケーションでComposer依存関係のインストール

チームでアプリケーションを開発している場合、最初にLaravelアプリケーションを作成するのは自分ではないかもしれません。そのため、アプリケーションのリポジトリをローカルコンピュータにクローンした後、Sailを含むアプリケーションのComposer依存関係は一切インストールされていません。

アプリケーションの依存関係をインストールするには、アプリケーションのディレクトリに移動し、次のコマンドを実行します。このコマンドは、PHPとComposerを含む小さなDockerコンテナを使用して、アプリケーションの依存関係をインストールします。

```shell
docker run --rm \
    -u "$(id -u):$(id -g)" \
    -v "$(pwd):/var/www/html" \
    -w /var/www/html \
    laravelsail/php82-composer:latest \
    composer install --ignore-platform-reqs
```

`laravelsail/phpXX-composer`イメージを使用する場合、アプリケーションで使用する予定のPHPと同じバージョン（`80`、`81`, または `82`）を使用する必要があります。

<a name="executing-artisan-commands"></a>
### Artisanコマンドの実行

Laravel Artisanコマンドは、`artisan`コマンドを使用して実行します。

```shell
sail artisan queue:work
```

<a name="executing-node-npm-commands"></a>
### Node／NPMコマンドの実行

Nodeコマンドは`npm`コマンドを使用して実行し、NPMコマンドは`npm`コマンドを使用して実行します。

```shell
sail node --version

sail npm run dev
```

ご希望であれば、NPMの代わりにYarnを使用できます。

```shell
sail yarn
```

<a name="interacting-with-sail-databases"></a>
## データベース操作

<a name="mysql"></a>
### MySQL

お気づきかもしれませんが、アプリケーションの`docker-compose.yml`ファイルには、MySQLコンテナのエントリが含まれています。このコンテナは[Dockerボリューム](https://docs.docker.com/storage/volumes/)を使用しているため、コンテナを停止して再起動しても、データベースに保存されているデータは保持されます。

さらに、MySQLコンテナの初回起動時に、２つのデータベースが作成されます。最初のデータベースは、環境変数`DB_DATABASE`の値で命名され、ローカル開発用に使用するものです。もうひとつは、`testing`という名前のテスト専用データベースで、テストが開発データに干渉しないようにするためのものです。

コンテナを起動したら、アプリケーションの`.env`ファイル内の`DB_HOST`環境変数を`mysql`に設定することで、アプリケーション内のMySQLインスタンスに接続できます。

ローカルマシンからアプリケーションのMySQLデータベースに接続するには、[TablePlus](https://tableplus.com)のようなグラフィカルなデータベース管理アプリケーションを使用する方が多いでしょう。デフォルトでは、MySQLデータベースへは`localhost`の3306ポートでアクセスでき、アクセス資格情報は`DB_USERNAME`と`DB_PASSWORD`環境変数の値に対応します。あるいは、`DB_PASSWORD`環境変数値をパスワードとして、`root`ユーザーとして接続することもできます。

<a name="redis"></a>
### Redis

アプリケーションの`docker-compose.yml`ファイルには、[Redis](https://redis.io)コンテナのエントリも含まれています。このコンテナは[Dockerボリューム](https://docs.docker.com/storage/volumes/)を使用しているため、コンテナを停止して再起動しても、Redisデータに保存されているデータは保持されます。コンテナを起動したら、アプリケーションの`.env`ファイル内の`REDIS_HOST`環境変数を`redis`に設定することで、アプリケーション内のRedisインスタンスに接続できます。

ローカルマシンからアプリケーションのRedisデータベースに接続するには、[TablePlus](https://tableplus.com)などのグラフィカルデータベース管理アプリケーションを使用できます。デフォルトでは、Redisデータベースは`localhost`のポート6379でアクセスできます。

<a name="meilisearch"></a>
### Meilisearch

Sailのインストール時に[Meilisearch](https://www.meilisearch.com)サービスのインストールを選択した場合、アプリケーションの`docker-compose.yml`ファイルには、[Laravel Scout](/docs/{{version}}/scout)と[コンパチブル](https://github.com/meilisearch/meilisearch-laravel-scout)である、この強力な検索エンジンのエントリが含まれます。コンテナを起動したら、環境変数`MEILISEARCH_HOST`に`http://meilisearch:7700`を設定すると、アプリケーション内のMeilisearchインスタンスに接続できます。

ローカルマシンから、Webブラウザの`http://localhost:7700`に移動して、MeilisearchのWebベース管理パネルへアクセスできます。

<a name="typesense"></a>
### Typesense

Sailインストール時に、[Typesense](https://typesense.org)サービスのインストールを選択した場合、これとネイティブに統合済みで、光のように速いオープンソース検索エンジンである[Laravel Scout](/docs/{{version}}/scout#typesense)のエントリが含まれます。コンテナを起動したら、以下の環境変数を設定することで、アプリケーション内からTypesenseインスタンスへ接続できます。

```ini
TYPESENSE_HOST=typesense
TYPESENSE_PORT=8108
TYPESENSE_PROTOCOL=http
TYPESENSE_API_KEY=xyz
```

ローカルマシンから、`http://localhost:8108`により、TypesenseのAPIへアクセスできます。

<a name="file-storage"></a>
## ファイルストレージ

本番環境でアプリケーションを実行する際に、Amazon S3を使用してファイルを保存する予定であれば、Sailをインストールする際に[MinIO](https://min.io)サービスをインストールするとよいでしょう。MinIOはS3互換のAPIを提供しており、本番のS3環境で「テスト」ストレージバケットを作成せずに、Laravelの`s3`ファイルストレージドライバを使ってローカルに開発するために使用できます。Sailのインストール時にMinIOのインストールを選択すると、アプリケーションの`docker-compose.yml`ファイルにMinIOの設定セクションが追加されます。

アプリケーションのデフォルト`filesystems`設定ファイルには、`s3`ディスクのディスク設定がすでに含まれています。このディスクを使ってAmazon S3と連携するだけでなく、その構成を制御する関連環境変数を変更するだけで、MinIOなどのS3互換のファイルストレージサービスと連携することができます。例えば、MinIOを使用する場合、ファイルシステムの環境変数の設定は次のように定義します。

```ini
FILESYSTEM_DISK=s3
AWS_ACCESS_KEY_ID=sail
AWS_SECRET_ACCESS_KEY=password
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=local
AWS_ENDPOINT=http://minio:9000
AWS_USE_PATH_STYLE_ENDPOINT=true
```

LaravelのFlysystemインテグレーションで、MinIOを使用している場合、適切なURLを生成するには、`AWS_URL`環境変数を定義してアプリケーションのローカルURLと一致させ、URLパスにバケット名を含める必要があります:

```ini
AWS_URL=http://localhost:9000/local
```

`http://localhost:8900`のMinIOコンソールからバケットを作成できます。MinIOコンソールのデフォルトユーザー名は`sail`、パスワードは`password`です。

> [!WARNING]
> MinIOを使用している場合の、`temporaryUrl`メソッドによる一時保存用URL生成はサポートしていません。

<a name="running-tests"></a>
## テスト実行

Laravel provides amazing testing support out of the box, and you may use Sail's `test` command to run your applications [feature and unit tests](/docs/{{version}}/testing). Any CLI options that are accepted by Pest / PHPUnit may also be passed to the `test` command:

```shell
sail test

sail test --group orders
```

Sailの`test`コマンドは、`test` Artisanコマンドを実行するのと同じです。

```shell
sail artisan test
```

デフォルトでSailは、テストがデータベースの現在の状態に影響を与えないよう、専用の`testing`データベースを作成します。Laravelのデフォルトインストールは、Sailがテストを実行するときにこのデータベースを使用するように、`phpunit.xml`ファイルを構成します。

```xml
<env name="DB_DATABASE" value="testing"/>
```

<a name="laravel-dusk"></a>
### Laravel Dusk

[Laravel Dusk](/docs/{{version}}/dusk)は、表現力豊かで使いやすいブラウザ自動化およびテストのAPIを提供します。Sailのおかげで、Seleniumやその他のツールをローカルコンピューターにインストールしなくても、これらのテストを実行できます。使い始めるには、アプリケーションの`docker-compose.yml`ファイルでSeleniumサービスのコメントを解除します。

```yaml
selenium:
    image: 'selenium/standalone-chrome'
    extra_hosts:
      - 'host.docker.internal:host-gateway'
    volumes:
        - '/dev/shm:/dev/shm'
    networks:
        - sail
```

次に、アプリケーションの`docker-compose.yml`ファイルの`laravel.test`サービスに`selenium`の`depends_on`エントリがあることを確認します。

```yaml
depends_on:
    - mysql
    - redis
    - selenium
```

最後に、Sailを起動して`dusk`コマンドを実行することにより、Duskテストスイートを実行できます。

```shell
sail dusk
```

<a name="selenium-on-apple-silicon"></a>
#### Apple Silicon上のSelenium

ローカルマシンに Apple Silicon チップが搭載されている場合、`selenium`サービスには`seleniarm/standalone-chromium`イメージを使用する必要があります。

```yaml
selenium:
    image: 'seleniarm/standalone-chromium'
    extra_hosts:
        - 'host.docker.internal:host-gateway'
    volumes:
        - '/dev/shm:/dev/shm'
    networks:
        - sail
```

<a name="previewing-emails"></a>
## メールのプレビュー

Laravel Sailのデフォルトの`docker-compose.yml`ファイルは、[Mailpit](https://github.com/axllent/mailpit)のサービスエントリを含んでいます。Mailpitは、ローカル開発中にアプリケーションが送信したメールを傍受し、ブラウザでメールメッセージをプレビューできるように、便利なWebインタフェースを提供します。Sailを使用している場合、Mailpitのデフォルトのホストは、`mailpit`で、ポート1025経由で利用可能です。

```ini
MAIL_HOST=mailpit
MAIL_PORT=1025
MAIL_ENCRYPTION=null
```

Sailが起動している場合、http://localhost:8025で、MailpitのWebインターフェイスにアクセスできます：

<a name="sail-container-cli"></a>
## コンテナCLI

アプリケーションのコンテナ内でBashセッションを開始したい場合があるでしょう。`shell`コマンドを使用してアプリケーションのコンテナに接続し、ファイルとインストールされているサービスを検査したり、コンテナ内で任意のシェルコマンドを実行したりできます。

```shell
sail shell

sail root-shell
```

新しい[LaravelTinker](https://github.com/laravel/tinker)セッションを開始するには、`tinker`コマンドを実行します。

```shell
sail tinker
```

<a name="sail-php-versions"></a>
## PHPバージョン

Sailは現在、PHP8.3、PHP8.2、PHP8.1、PHP8.0を利用したアプリケーションの実行をサポートしています。SailのデフォルトPHPバージョンは8.3です。アプリケーションの実行に使用するPHPバージョンを変更するには、アプリケーションの`docker-compose.yml`ファイル内の`laravel.test`コンテナの`build`定義を更新してください。

```yaml
# PHP 8.3
context: ./vendor/laravel/sail/runtimes/8.3

# PHP 8.2
context: ./vendor/laravel/sail/runtimes/8.2

# PHP 8.1
context: ./vendor/laravel/sail/runtimes/8.1

# PHP8.0
context: ./vendor/laravel/sail/runtimes/8.0
```

さらに、アプリケーションで使用するPHPのバージョンを反映するために、`image`名を更新することもできます。このオプションも、アプリケーションの`docker-compose.yml`ファイルで定義されています。

```yaml
image: sail-8.2/app
```

アプリケーションの`docker-compose.yml`ファイルを更新した後、コンテナイメージを再構築する必要があります。

```shell
sail build --no-cache

sail up
```

<a name="sail-node-versions"></a>
## Nodeバージョン

SailはデフォルトでNode20をインストールします。イメージをビルドする際にインストールするNodeバージョンを変更するには、アプリケーションの`docker-compose.yml`ファイル中の、`laravel.test`サービスの`build.args`定義を変更してください。

```yaml
build:
    args:
        WWWGROUP: '${WWWGROUP}'
        NODE_VERSION: '18'
```

アプリケーションの`docker-compose.yml`ファイルを更新した後、コンテナイメージを再構築する必要があります。

```shell
sail build --no-cache

sail up
```

<a name="sharing-your-site"></a>
## サイトの共有

同僚のためにサイトをプレビューしたり、アプリケーションとのWebhook統合をテストしたりするために、サイトを公開して共有する必要がある場合があります。サイトを共有するには、 `share`コマンドを使用します。このコマンドを実行すると、アプリケーションへのアクセスに使用するランダムな`laravel-sail.site` URLが発行されます。

```shell
sail share
```

When sharing your site via the `share` command, you should configure your application's trusted proxies using the `trustProxies` middleware method in your application's `bootstrap/app.php` file. Otherwise, URL generation helpers such as `url` and `route` will be unable to determine the correct HTTP host that should be used during URL generation:

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->trustProxies(at: [
            '*',
        ]);
    })

共有サイトでサブドメインを選択する場合は、`share`コマンドを実行するときに`subdomain`オプションを指定します。

```shell
sail share --subdomain=my-sail-site
```

> [!NOTE]
> `share`コマンドは、[BeyondCode](https://beyondco.de)によるオープンソースのトンネリングサービスである[Expose](https://github.com/beyondcode/expose)により提供しています。

<a name="debugging-with-xdebug"></a>
## Xdebugによるデバッグ

LaravelSailのDockerの設定には、PHP用の人気で強力なデバッガである[Xdebug](https://xdebug.org/)をサポートしています。XDebugを有効にするには、[Xdebugを設定](https://xdebug.org/docs/step_debug#mode)するために、アプリケーションの`.env`ファイルに変数を追加する必要があります。XDebugを有効にするには、Sailを開始する前に適切なモードを設定する必要があります。

```ini
SAIL_XDEBUG_MODE=develop,debug,coverage
```

#### LinuxホストIP設定

Internally, the `XDEBUG_CONFIG` environment variable is defined as `client_host=host.docker.internal` so that Xdebug will be properly configured for Mac and Windows (WSL2). If your local machine is running Linux, you should ensure that you are running Docker Engine 17.06.0+ and Compose 1.16.0+. Otherwise, you will need to manually define this environment variable as shown below.

まず、以下のコマンドを実行して、環境変数に追加する正しいホストIPアドレスを決定します。通常、`<container-name>`は、アプリケーションを提供するコンテナの名前であるべきで、多くの場合、`_laravel.test_1`で終わります。

```shell
docker inspect -f {{range.NetworkSettings.Networks}}{{.Gateway}}{{end}} <container-name>
```

正しいホストIPアドレスを取得したら、アプリケーションの`.env`ファイル内で`SAIL_XDEBUG_CONFIG`変数を定義する必要があります。

```ini
SAIL_XDEBUG_CONFIG="client_host=<host-ip-address>"
```

<a name="xdebug-cli-usage"></a>
### Xdebug CLI使用法

Artisanのコマンドを実行する際に、`sail debug`コマンドを使ってデバッグセッションを開始することができます。

```shell
# Xdebugなしでartisanコマンドを実行
sail artisan migrate

# Xdebugでartisanコマンドを実行
sail debug migrate
```

<a name="xdebug-browser-usage"></a>
### Xdebug ブラウザ使用法

Webブラウザでアプリケーションを操作しながらデバッグするには、WebブラウザからXdebugセッションを開始するための[Xdebugが提供する手順](https://xdebug.org/docs/step_debug#web-application)に従ってください。

PhpStormを使用している場合は、[設定なしのデバッグ](https://www.jetbrains.com/help/phpstorm/zero-configuration-debugging.html)に関するJetBrainのドキュメントを確認してください。

> [!WARNING]
> Laravel Sailはアプリケーション提供を`artisan serve`に依存しています。`artisan serve`コマンドは、Laravelバージョン8.53.0以降では、`XDEBUG_CONFIG`と`XDEBUG_MODE`変数のみを受け付けます。古いバージョンのLaravel（8.52.0以下）では、これらの変数をサポートしていないため、デバッグ接続を受け付けません。

<a name="sail-customization"></a>
## Sailカスタマイズ

Sailは単なるDockerであるため、Sailに関するほぼすべてを自由にカスタマイズできます。Sail独自のDockerfileをリソース公開するには、`sail:publish`コマンドを実行します。

```shell
sail artisan sail:publish
```

このコマンドを実行すると、Laravel Sailが使用するDockerfileとその他の設定ファイルが、アプリケーションのルートディレクトリ内の`docker`ディレクトリに配置されます。Sailのインストールをカスタマイズした後、アプリケーションの `docker-compose.yml` ファイル内のアプリケーションコンテナのイメージ名を変更したいと思うことでしょう。それを行ったら、`build`コマンドを使用してアプリケーションのコンテナを再構築してください。アプリケーション・イメージに一意な名前を割り当てることは、Sailを使用して1台のマシンで複数のLaravelアプリケーションを開発する場合に特に重要です。

```shell
sail build --no-cache
```
