# Laravel Reverb

- [イントロダクション](#イントロダクション)
- [インストール](#installation)
- [設定](#configuration)
    - [アプリケーション認証情報](#application-credentials)
    - [オリジンの許可](#allowed-origins)
    - [アプリケーションの追加](#additional-applications)
    - [SSL](#ssl)
- [サーバの実行](#running-server)
    - [デバッグ](#debugging)
    - [リスタート](#restarting)
- [監視](#monitoring)
- [実機でのReverb実行](#production)
    - [Open Files](#open-files)
    - [イベントループ](#event-loop)
    - [Webサーバ](#web-server)
    - [ポート](#ports)
    - [プロセス管理](#process-management)
    - [Scaling](#scaling)

<a name="イントロダクション"></a>
## イントロダクション

[Laravel Reverb](https://github.com/laravel/reverb)（リバーブ：反響）は、高速でスケーラブルなリアルタイムWebSocket通信をLaravelアプリケーションへ直接もたらし、既存の[イベントブロードキャストツール](/docs/{{version}}/broadcasting)スイートとシームレスに統合します。

<a name="installation"></a>
## インストール

Reverbのインストールするには、`install:broadcasting` Artisanコマンドを使用します。

```
php artisan install:broadcasting
```

<a name="configuration"></a>
## 設定

`install:broadcasting` Artisanコマンドは、背後で`reverb:install`コマンドを実行し、適切なデフォルト設定オプションでReverbをインストールします。設定を変更したい場合は、Reverbの環境変数を更新するか、`config/reverb.php`設定ファイルを更新してください。

<a name="application-credentials"></a>
### アプリケーション認証情報

Reverbへの接続を確立するには、クライアントとサーバ間でReverbの「アプリケーション」認証情報を交換する必要があります。この認証情報はサーバ上で設定し、クライアントからのリクエストを確認するために使用します。この認証情報は、以下の環境変数を使用して定義できます。

```ini
REVERB_APP_ID=my-app-id
REVERB_APP_KEY=my-app-key
REVERB_APP_SECRET=my-app-secret
```

<a name="allowed-origins"></a>
### オリジンの許可

`config/reverb.php`設定ファイルの`apps`セクションにある、`allowed_origins`設定値を更新し、クライアントリクエストの送信元を定義することもできます。許可するオリジンにリストしていないオリジンからのリクエストは拒否します。すべてのオリジンを許可するには、`*`を使用します。

```php
'apps' => [
    [
        'id' => 'my-app-id',
        'allowed_origins' => ['laravel.com'],
        // ...
    ]
]
```

<a name="additional-applications"></a>
### アプリケーションの追加

通常Reverbは、インストール済みのアプリケーション用のWebSocketサーバを提供します。しかし、１つのReverbインストールで、複数のアプリケーションに対応することも可能です。

例えば、１つのLaravelアプリケーションを保守しており、Reverbを介して複数のアプリケーションにWebSocket接続を提供したいとしましょう。これは、アプリケーションの`config/reverb.php`設定ファイルへ、複数の`apps`を定義すれば可能です。

```php
'apps' => [
    [
        'app_id' => 'my-app-one',
        // ...
    ],
    [
        'app_id' => 'my-app-two',
        // ...
    ],
],
```

<a name="ssl"></a>
### SSL

ほとんどの場合、セキュアなWebSocket接続は、リクエストがReverbサーバへプロキシされる前に、上流のWebサーバ（Nginxなど）によって処理されます。

しかし、ローカル開発時などでは、セキュアな接続をReverbサーバが直接処理するのが便利な場合もあります。[Laravel Herd](https://herd.laravel.com)のセキュアサイト機能を使用しているか、[Laravel Valet](/docs/{{version}}/valet)を使用していて、アプリケーションに対して[secureコマンド](/docs/{{version}}/valet#securing-sites)を実行している場合、サイト用に生成されたHerd／Valet証明書を使用して、Reverb接続をセキュアにできます。これを行うには、`REVERB_HOST`環境変数へサイトのホスト名を設定するか、Reverbサーバの起動時に明示的にホスト名オプションを渡します：

```sh
php artisan reverb:start --host="0.0.0.0" --port=8080 --hostname="laravel.test"
```

HerdとValetのドメインは`localhost`で解決されるため、上記のコマンドを実行すると、Reverbサーバは`wss://laravel.test:8080`のセキュアなWebSocketプロトコル（`wss`）を介してアクセスできるようになります。

アプリケーションの`config/reverb.php`設定ファイルで、`tls`オプションを定義して、証明書を手作業で選択することもできます。`tls`オプションの配列には、[PHPのSSLコンテキストオプション](https://www.php.net/manual/en/context.ssl.php)でサポートされているオプションを指定できます。

```php
'options' => [
    'tls' => [
        'local_cert' => '/path/to/cert.pem'
    ],
],
```

<a name="running-server"></a>
## サーバの実行

Reverbサーバは、`reverb:start` Artisanコマンドで起動できます。

```sh
php artisan reverb:start
```

Reverbサーバはデフォルトで、`0.0.0.0:8080`で起動し、すべてのネットワークインターフェイスからアクセスできるようにします。

カスタムホストやカスタムポートを指定する必要がある場合は、サーバ起動時に`--host`と`--port`オプションで指定してください。

```sh
php artisan reverb:start --host=127.0.0.1 --port=9000
```

もしくは、アプリケーションの`.env`設定ファイルに、`REVERB_SERVER_HOST`と`REVERB_SERVER_PORT`環境変数を定義してください。

`REVERB_SERVER_HOST`と`REVERB_SERVER_PORT`環境変数は、`REVERB_HOST`と`REVERB_PORT`と混同してはいけません。前者はReverbサーバ自身を実行するホストとポートを指定するのに対し、一方、後者はブロードキャストメッセージの送信先をLaravelへ指示します。例えば、本番環境では、ポート`443`にある公開Reverbホスト名からのリクエストを、`0.0.0.0:8080`で動作するReverbサーバにルーティングするとしましょう。この場合、環境変数は以下のように定義します。

```ini
REVERB_SERVER_HOST=0.0.0.0
REVERB_SERVER_PORT=8080

REVERB_HOST=ws.laravel.com
REVERB_PORT=443
```

<a name="debugging"></a>
### デバッグ

パフォーマンスを向上させるため、Reverbはデフォルトではデバッグ情報を出力しません。Reverbサーバを通過するデータのストリームを見たい場合は、`reverb:start`コマンドへ`--debug`オプションを指定してください。

```sh
php artisan reverb:start --debug
```

<a name="restarting"></a>
### リスタート

Reverbは実行終了しないプロセスなのため、`reverb:restart` Artisanコマンドでサーバを再起動しない限り、コードの変更は反映されません。

`reverb:restart`コマンドは、サーバを停止する前にすべての接続を確実に終了させます。Supervisorのようなプロセスマネージャを使い、Reverbを実行している場合は、全ての接続が終了した後、プロセスマネージャによりサーバが自動的に再起動されます：

```sh
php artisan reverb:restart
```

<a name="monitoring"></a>
## 監視

Reverbは、[Laravel Pulse](/docs/{{version}}/pulse)との統合により、監視できます。ReverbのPulse統合を有効にすれば、サーバが処理している接続数とメッセージ数を追跡できます。

この統合を有効にするには、まずPulseを確実に[インストール](/docs/{{version}}/pulse#installation)してください。次に、アプリケーションの`config/pulse.php`設定ファイルで、Reverbのレコーダを追加してください。

```php
use Laravel\Reverb\Pulse\Recorders\ReverbConnections;
use Laravel\Reverb\Pulse\Recorders\ReverbMessages;

'recorders' => [
    ReverbConnections::class => [
        'sample_rate' => 1,
    ],

    ReverbMessages::class => [
        'sample_rate' => 1,
    ],

    ...
],
```

次に、各レコーダーのPulseカードを[Pulseダッシュボード](/docs/{{version}}/pulse#dashboard-customization)に追加します。

```blade
<x-pulse>
    <livewire:reverb.connections cols="full" />
    <livewire:reverb.messages cols="full" />
    ...
</x-pulse>
```

<a name="production"></a>
## 実機でのReverb実行

WebSocketサーバは長時間稼動するため、Reverbがサーバで利用可能なリソースに対して最適な数の接続を効果的に処理できるように、サーバとホスティング環境を最適化する必要が起きる場合があります。

> [!NOTE]
> あなたのサイトを[Laravel Forge](https://forge.laravel.com)で管理している場合、"Application"パネルから直接Reverb用にサーバを自動的に最適化できます。Reverbの統合を有効にすることで、Forgeは必要な拡張機能のインストールや接続許可数の増加など、サーバを本番環境に対応させます。

<a name="open-files"></a>
### ファイルオープン

各WebSocket接続は、クライアントまたはサーバのどちらかが切断するまでメモリに保持されます。UnixやUnixライクな環境では、各接続はファイルで表されます。しかし、多くの場合、オペレーティング・システムとアプリケーション・レベルの両方で、許可されるファイルオープン数に制限があります。

<a name="operating-system"></a>
#### オペレーティングシステム

Unixベースのオペレーティングシステムでは、`ulimit`コマンドを使ってオープンできるファイルの数を設定できます。

```sh
ulimit -n
```

このコマンドは、ユーザーごとに許可されているファイルオープンの上限を表示します。これらの値は、`/etc/security/limits.conf`ファイルを編集し更新できます。例えば、`forge`ユーザーのオープンファイルの最大数を10,000に更新するには、以下のようにします。

```ini
# /etc/security/limits.conf
forge        soft  nofile  10000
forge        hard  nofile  10000
```

<a name="event-loop"></a>
### イベントループ

Reverbはサーバ上のWebSocket接続を管理するために、ReactPHPのイベントループを使用しています。このイベントループはデフォルトで、`stream_select`によって動かされ、追加の拡張は必要ありません。しかし、`stream_select` は通常、1,024個のファイルオープンに制限されています。そのため、1,000を超える同時接続を処理する場合は、同じ制限に限定されない、別のイベントループを使用する必要があります。

利用可能な場合、Reverbは自動的に`ext-uv`ループに切り替わります。このPHPエクステンションはPECL経由でインストール可能です。

```sh
pecl install uv
```

<a name="web-server"></a>
### Webサーバ

ほとんどの場合、Reverbはサーバ上のウェブに関係ないポートで実行します。そのため、Reverbへのトラフィックをルーティングするために、リバースプロキシを設定する必要があります。Reverbがホスト`0.0.0.0`と、ポート`8080`で動作していて、サーバがNginxウェブサーバと仮定する場合、以下のNginxサイト設定を使用し、リバースプロキシをReverbサーバへ定義できます。

```nginx
server {
    ...

    location / {
        proxy_http_version 1.1;
        proxy_set_header Host $http_host;
        proxy_set_header Scheme $scheme;
        proxy_set_header SERVER_PORT $server_port;
        proxy_set_header REMOTE_ADDR $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";

        proxy_pass http://0.0.0.0:8080;
    }

    ...
}
```

> [!WARNING]
> Reverbは`/app`でWebSocket接続をリッスンし、`/apps`でAPIリクエストを処理します。Reverbリクエストを処理するウェブサーバで確実に、これら両方のURIを処理してください。サーバの管理に[Laravel Forge](https://forge.laravel.com)を使用している場合、Reverbサーバはデフォルトで正しく設定されます。

通常、Webサーバはサーバの過負荷を防ぐため、許可する接続数を制限するように設定します。Nginxウェブサーバで許可する接続数を１０，０００へ増やすには、`nginx.conf`ファイルの`worker_rlimit_nofile`と`worker_connections`の値を更新する必要があります。

```nginx
user forge;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;
worker_rlimit_nofile 10000;

events {
  worker_connections 10000;
  multi_accept on;
}
```

上記の設定により、１プロセスあたり最大１０，０００のNginxワーカを生成できるようになります。さらに、この設定は Nginxのファイルオープンの上限を１０，０００に設定します。

<a name="ports"></a>
### ポート

Unixベースのオペレーティング・システムでは通常、サーバ上でオープンできるポート数を制限しています。現在許可されている範囲は、以下のコマンドで確認できます。

 ```sh
 cat /proc/sys/net/ipv4/ip_local_port_range
# 32768	60999
```

各接続が空きポートを必要とするため、上記の出力はサーバが最大２８，２３１個 (60,999-32,768)の接続を処理できることを示しています。[水平スケーリング](#scaling)で許容接続数を増やすことを推奨しますが、サーバの`/etc/sysctl.conf`設定ファイルの許容ポート範囲を更新することで、利用可能なポートオープン数を増やせます。

<a name="process-management"></a>
### プロセス管理

ほとんどの場合、Supervisorなどのプロセスマネージャを使用して、Reverbサーバを確実に持続的に実行する必要があります。Reverbの実行にSupervisorを使用している場合は、サーバの`supervisor.conf`ファイルの`minfds`設定を更新して、SupervisorがReverbサーバへの接続を処理するために必要なファイルを開けるようにする必要があります。

```ini
[supervisord]
...
minfds=10000
```

<a name="scaling"></a>
### スケーリング

1台のサーバでは処理しきれない数の接続を処理する必要がある場合、Reverbサーバを水平方向に拡張できます。RedisのPub／Sub機能を利用することで、Reverbは複数のサーバにまたがる接続を管理できます。アプリケーションのReverbサーバの１つがメッセージを受信すると、そのサーバはRedisを使用して、受信したメッセージを他のすべてのサーバに公開します。

水平スケーリングを有効にするには、アプリケーションの`.env` 設定ファイルで、`REVERB_SCALING_ENABLED`環境変数を`true`に設定します。

```env
REVERB_SCALING_ENABLED=true
```

次に、すべてのReverbサーバが通信する専用の中央Redisサーバを用意します。Reverbは、[アプリケーション用に設定したデフォルトのRedis接続](/docs/{{version}}/redis#configuration)を使用して、すべてのReverbサーバへメッセージを公開します。

一度、Reverbのスケーリングオプションを有効にし、Redisサーバを設定したら、Redisサーバと通信できる複数のサーバ上で`reverb:start`コマンドを呼び出すだけです。これらのReverbサーバは、入ってくるリクエストをサーバ間で均等に分散させるロードバランサーの後ろに設置すべきです。
