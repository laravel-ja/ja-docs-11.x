# ログ

- [イントロダクション](#introduction)
- [設定](#configuration)
    - [利用可能なチャンネルドライバ](#available-channel-drivers)
    - [チャンネルの事前設定](#channel-prerequisites)
    - [廃止ワーニングのログ](#logging-deprecation-warnings)
- [ログスタックの構築](#building-log-stacks)
- [ログメッセージの書き込み](#writing-log-messages)
    - [コンテキスト情報](#contextual-information)
    - [特定チャンネルへの書き込み](#writing-to-specific-channels)
- [monologチャンネルカスタマイズ](#monolog-channel-customization)
    - [チャンネル向けMonologカスタマイズ](#customizing-monolog-for-channels)
    - [Monolog処理チャンネルの作成](#creating-monolog-handler-channels)
    - [ファクトリによるカスタムチャンネルの生成](#creating-custom-channels-via-factories)
- [Pailを使用したログの限定出力](#tailing-log-messages-using-pail)
    - [インストール](#pail-installation)
    - [使用法](#pail-usage)
    - [ログのフィルタリング](#pail-filtering-logs)

<a name="introduction"></a>
## イントロダクション

Laravelは、アプリケーション内で何が起こっているかを詳しく知らせるために、メッセージをファイル、システムエラーログ、さらにはSlackに記録してチーム全体に通知できる堅牢なログサービスを提供します。

Laravelのログは「チャンネル」に基づいています。各チャンネルは、ログ情報を書き込む特定の方法を表します。たとえば、`single`チャンネルはログファイルを単一のログファイルに書き込みますが、`slack`チャンネルはログメッセージをSlackへ送信します。ログメッセージは、重大度に基づいて複数のチャンネルに書き込まれる場合があります。

内部的にLaravelは[Monolog](https://github.com/Seldaek/monolog)ライブラリを利用しており、強力なログハンドラを多種に渡りサポートしています。Laravelでは、こうしたハンドラを簡単に設定できるため、ハンドラを組み合わせてアプリケーションのログ処理をカスタマイズできます。

<a name="configuration"></a>
## 設定

アプリケーションのログ動作を制御する、すべての設定オプションは、`config/logging.php`設定ファイルに格納しています。このファイルでアプリケーションのログチャンネルを設定でき、利用可能なチャンネルとそのオプションのそれぞれを確認してください。以下に、いくつかの一般的なオプションについて説明します。

Laravelはメッセージをログに記録するときに、デフォルトで`stack`チャンネルを使用します。`stack`チャンネルは、複数のログチャンネルを単一のチャンネルに集約するために使用します。スタックの構築の詳細については、[以降のドキュメント](#building-log-stacks)を確認してください。

<a name="available-channel-drivers"></a>
### 利用可能なチャンネルドライバ

各ログチャンネルは「ドライバ」によって駆動されます。ドライバは、ログメッセージが実際に記録される方法と場所を決定します。以下のログチャンネルドライバは、すべてのLaravelアプリケーションで利用できます。これらのドライバのほとんどのエントリは、アプリケーションの`config/logging.php`設定ファイルに予め用意しているため、このファイルを確認して内容をよく理解してください。

<div class="overflow-auto">

| 名前         | 説明                                                                   |
| ------------ | ---------------------------------------------------------------------- |
| `custom`     | 指定ファクトリを呼び出してチャンネルを作成するドライバ                 |
| `daily`      | 日毎にファイルを切り替える`RotatingFileHandler`ベースのMonologドライバ |
| `errorlog`   | `ErrorLogHandler`ベースのMonologドライバ                               |
| `monolog`    | Monologがサポートしているハンドラを使用するMonologファクトリドライバ   |
| `papertrail` | `SyslogUdpHandler`ベースのMonologドライバ                              |
| `single`     | 単一のファイルまたはパスベースのロガーチャンネル(`StreamHandler`)      |
| `slack`      | `SlackWebhookHandler`ベースのMonologドライバ                           |
| `stack`      | 「マルチチャンネル」チャンネルの作成を容易にするラッパー               |
| `syslog`     | `SyslogHandler`ベースのMonologドライバ                                 |

</div>

> [!NOTE]
> [高度なチャンネルのカスタマイズ](#monolog-channel-customization)のドキュメントをチェックして、`monolog`および`custom`ドライバの詳細を確認してください。

<a name="configuring-the-channel-name"></a>
#### チャンネル名の設定

Monologはデフォルトで、現在の環境にマッチする「チャンネル名」でインスタンス化します。この値を変更する場合は、チャネルの設定に`name`オプションを追加します。

    'stack' => [
        'driver' => 'stack',
        'name' => 'channel-name',
        'channels' => ['single', 'slack'],
    ],

<a name="channel-prerequisites"></a>
### チャンネルの事前設定

<a name="configuring-the-single-and-daily-channels"></a>
#### singleチャンネルとdailyチャンネルの設定

`single`チャンネルと`daily`チャンネルは、`bubble`、`permission`、`locking`の３オプションの設定オプションがあります。

<div class="overflow-auto">

| 名前         | 説明                                                                         | デフォルト |
| ------------ | ---------------------------------------------------------------------------- | ---------- |
| `bubble`     | メッセージが処理された後、他のチャンネルにバブルアップする必要があるかを示す | `true`     |
| `locking`    | ログファイルに書き込む前に、ログファイルのロックを試みるかを示す             | `false`    |
| `permission` | ログファイルのパーミッション                                                 | `0644`     |

</div>

さらに、`LOG_DAILY_DAYS`環境変数、または`days`設定オプションを設定することで、`daily`チャンネルの保持ポリシーを設定できます。

<div class="overflow-auto">

| 名前   | 説明                               | デフォルト |
| ------ | ---------------------------------- | ---------- |
| `days` | デイリーログファイルを保持する日数 | `7`        |

</div>

<a name="configuring-the-papertrail-channel"></a>
#### Papertrailチャンネルの設定

`papertrail`チャネルは、`host`と`port`の設定オプションが必要です。これらは`PAPERTRAIL_URL`と`PAPERTRAIL_PORT`環境変数で定義できます。これらの値は[Papertrail](https://help.papertrailapp.com/kb/configuration/configuring-centralized-logging-from-php-apps/#send-events-from-php-app)から取得できます。

<a name="configuring-the-slack-channel"></a>
#### Slackチャンネルの設定

`slack`チャネルには、`url`設定オプションが必要です。この値は`LOG_SLACK_WEBHOOK_URL`環境変数で定義します。このURLは、Slackチーム用に設定した[受信Webフック](https://slack.com/apps/A0F7XDUAZ-incoming-webhooks)のURLと一致する必要があります。

Slackはデフォルトで、`critical`レベル以上のログしか受け取りません。しかし`LOG_LEVEL`環境変数を使うか、Slackのログチャンネルの設定配列内の`level`設定オプションを変更することで調整できます。

<a name="logging-deprecation-warnings"></a>
### 廃止ワーニングのログ

PHPやLaravelなどのライブラリは、機能の一部が非推奨となり、将来のバージョンで削除されることをユーザーへ通知することがよくあります。このような非推奨の警告をログに記録したい場合は、`LOG_DEPRECATIONS_CHANNEL`環境変数を使用するか、アプリケーションの`config/logging.php`設定ファイル内で、好みの`deprecations`ログチャンネルを指定してください。

    'deprecations' => [
        'channel' => env('LOG_DEPRECATIONS_CHANNEL', 'null'),
        'trace' => env('LOG_DEPRECATIONS_TRACE', false),
    ],

    'channels' => [
        // ...
    ]

あるいは、`deprecations`という名前のログチャンネルを定義することもできます。この名前のログチャンネルが存在する場合、常にdeprecationsのログを記録するために使用されます。

    'channels' => [
        'deprecations' => [
            'driver' => 'single',
            'path' => storage_path('logs/php-deprecation-warnings.log'),
        ],
    ],

<a name="building-log-stacks"></a>
## ログスタックの構築

前述のように、`stack`ドライバを使用すると、便利に複数のチャンネルを１つのログチャンネルに組み合わせることができます。ログスタックの使用方法を説明するために、本番アプリケーションで使われる可能性のある構成例を見てみましょう。

```php
'channels' => [
    'stack' => [
        'driver' => 'stack',
        'channels' => ['syslog', 'slack'], // [tl! add]
        'ignore_exceptions' => false,
    ],

    'syslog' => [
        'driver' => 'syslog',
        'level' => env('LOG_LEVEL', 'debug'),
        'facility' => env('LOG_SYSLOG_FACILITY', LOG_USER),
        'replace_placeholders' => true,
    ],

    'slack' => [
        'driver' => 'slack',
        'url' => env('LOG_SLACK_WEBHOOK_URL'),
        'username' => env('LOG_SLACK_USERNAME', 'Laravel Log'),
        'emoji' => env('LOG_SLACK_EMOJI', ':boom:'),
        'level' => env('LOG_LEVEL', 'critical'),
        'replace_placeholders' => true,
    ],
],
```

この構成を分析してみましょう。まず、`stack`チャンネルが`channels`オプションを介して他の2つのチャンネル`syslog`と`slack`を集約している点に注目してください。したがって、メッセージをログに記録するとき、これらのチャンネルの両方にメッセージをログに記録する機会があります。ただし、以降で説明するように、これらのチャンネルが実際にメッセージをログに記録するかどうかは、メッセージの重大度／「レベル」によって決定されます。

<a name="log-levels"></a>
#### ログレベル

上記の例の`syslog`および`slack`チャンネル設定に存在する`level`設定オプションに注意してください。このオプションは、チャンネルによってログに記録されるためにメッセージが必要とする最小の「レベル」を決定します。Laravelのログサービスを強化するMonologは、[RFC5424仕様](https://tools.ietf.org/html/rfc5424)で定義されているすべてのログレベルを提供しています。これらのログレベルは、重要度の高い順に、**emergency**、**alert**、**critical**、**error**、**warning**、**notice**、**info**、**debug**です。

では、`debug`メソッドを使用してメッセージをログに記録してみましょう。

    Log::debug('An informational message.');

前記の設定により、`syslog`チャンネルはメッセージをシステムログに書き込みます。ただし、エラーメッセージは`critical`以上ではないため、Slackには送信されません。ただし、`emergency`メッセージをログに記録すると、`emergency`レベルが両方のチャンネルの最小レベルしきい値を超えるため、システムログとSlackの両方に送信されます。

    Log::emergency('The system is down!');

<a name="writing-log-messages"></a>
## ログメッセージの書き込み

`Log`[ファサード](/docs/{{version}}/facades)を使用してログに情報を書き込むことができます。前述のように、ロガーは[RFC5424仕様](https://tools.ietf.org/html/rfc5424)で定義されている8つのログレベルを提供します**emergency**、**alert**、**critical**、**error**、**warning**、**notice**、**info**、**debug**）

    use Illuminate\Support\Facades\Log;

    Log::emergency($message);
    Log::alert($message);
    Log::critical($message);
    Log::error($message);
    Log::warning($message);
    Log::notice($message);
    Log::info($message);
    Log::debug($message);

これらのメソッドのいずれかを呼び出して、対応するレベルのメッセージをログに記録できます。デフォルトでは、メッセージは`logging`設定ファイル中に設定しているデフォルトのログチャンネルに書き込まれます。

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Models\User;
    use Illuminate\Support\Facades\Log;
    use Illuminate\View\View;

    class UserController extends Controller
    {
        /**
         * 特定のユーザーのプロファイルの表示
         */
        public function show(string $id): View
        {
            Log::info('Showing the user profile for user: {id}', ['id' => $id]);

            return view('user.profile', [
                'user' => User::findOrFail($id)
            ]);
        }
    }

<a name="contextual-information"></a>
### コンテキスト情報

コンテキストデータの配列をlogメソッドへ渡せます。このコンテキストデータはフォーマットされ、ログメッセージとともに表示されます。

    use Illuminate\Support\Facades\Log;

    Log::info('User {id} failed to login.', ['id' => $user->id]);

特定のチャンネルで、後に続くすべてのログエントリに含まれるコンテキスト情報を指定したい場合もあるでしょう。例えば、アプリケーションに入ってくる各リクエストに関連付けたリクエストIDをログに記録したい場合があります。これを行うには、`Log`ファサードの`withContext`メソッドを呼び出します。

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Log;
    use Illuminate\Support\Str;
    use Symfony\Component\HttpFoundation\Response;

    class AssignRequestId
    {
        /**
         * 受信リクエストの処理
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            $requestId = (string) Str::uuid();

            Log::withContext([
                'request-id' => $requestId
            ]);

            $response = $next($request);

            $response->headers->set('Request-Id', $requestId);

            return $response;
        }
    }

*すべて*のログチャンネルでコンテキスト情報を共有したい場合は、`Log::shareContext()`メソッドを呼び出します。このメソッドは、作成したすべてのチャンネルと、その後に作成したすべてのチャンネルへ、コンテキスト情報を提供します。

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Log;
    use Illuminate\Support\Str;
    use Symfony\Component\HttpFoundation\Response;

    class AssignRequestId
    {
        /**
         * 受信リクエストの処理
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            $requestId = (string) Str::uuid();

            Log::shareContext([
                'request-id' => $requestId
            ]);

            // ...
        }
    }

> [!NOTE]
> キュー投入したジョブの処理中にログコンテキストを共有する必要がある場合は、[ジョブミドルウェア](/docs/{{version}}/queues#job-middleware)を利用してください。

<a name="writing-to-specific-channels"></a>
### 特定チャンネルへの書き込み

アプリケーションのデフォルトチャンネル以外のチャンネルにメッセージを記録したいことも起きるでしょう。`Log`ファサードの`channel`メソッドを使用して、設定ファイルで定義している任意のチャンネルを取得し、ログへ記録できます。

    use Illuminate\Support\Facades\Log;

    Log::channel('slack')->info('Something happened!');

複数のチャンネルで構成されるログスタックをオンデマンドで作成する場合は、`stack`メソッドを使用できます。

    Log::stack(['single', 'slack'])->info('Something happened!');

<a name="on-demand-channels"></a>
#### オンデマンドチャンネル

アプリケーションの `logging` 設定ファイルに設定を用意しなくても、実行時に構成を指定することにより、オンデマンドチャンネルを作成することも可能です。そのためには、設定の配列を`Log`ファサードの`build`メソッドに渡してください。

    use Illuminate\Support\Facades\Log;

    Log::build([
      'driver' => 'single',
      'path' => storage_path('logs/custom.log'),
    ])->info('Something happened!');

オンデマンドチャンネルをオンデマンドログスタックに含めたい場合もあるでしょう。これを実現するには、`stack`メソッドに渡す配列へオンデマンドチャンネルのインスタンスを含めます。

    use Illuminate\Support\Facades\Log;

    $channel = Log::build([
      'driver' => 'single',
      'path' => storage_path('logs/custom.log'),
    ]);

    Log::stack(['slack', $channel])->info('Something happened!');

<a name="monolog-channel-customization"></a>
## monologチャンネルカスタマイズ

<a name="customizing-monolog-for-channels"></a>
### チャンネル向けMonologカスタマイズ

場合によっては、既存のチャンネルに対してMonologを設定する方法を完全に制御する必要が起きます。たとえば、Laravelの組み込みの`single`チャンネル用にカスタムMonolog `FormatterInterface`実装を設定したい場合です。

このためには、チャンネルの設定で`tap`配列を定義します。`tap`配列には、Monologインスタンスの作成後にカスタマイズする(またはtap into：入れ込む)機会が必要なクラスのリストを含める必要があります。これらのクラスを配置する決まった場所はないため、アプリケーション内にこれらのクラスを含むディレクトリを自由に作成できます。

    'single' => [
        'driver' => 'single',
        'tap' => [App\Logging\CustomizeFormatter::class],
        'path' => storage_path('logs/laravel.log'),
        'level' => env('LOG_LEVEL', 'debug'),
        'replace_placeholders' => true,
    ],

チャンネルで`tap`オプションを設定したら、Monologインスタンスをカスタマイズするクラスを定義する準備が整います。このクラスに必要なメソッドは1つだけです。`__invoke`は`Illuminate\Log\Logger`インスタンスを受け取ります。`Illuminate\Log\Logger`インスタンスは、基礎となるMonologインスタンスへのすべてのメソッド呼び出しをプロキシします。

    <?php

    namespace App\Logging;

    use Illuminate\Log\Logger;
    use Monolog\Formatter\LineFormatter;

    class CustomizeFormatter
    {
        /**
         * 指定するロガーインスタンスをカスタマイズ
         */
        public function __invoke(Logger $logger): void
        {
            foreach ($logger->getHandlers() as $handler) {
                $handler->setFormatter(new LineFormatter(
                    '[%datetime%] %channel%.%level_name%: %message% %context% %extra%'
                ));
            }
        }
    }

> [!NOTE]
> すべての「tap」クラスは[サービスコンテナ](/docs/{{version}}/container)によって解決されるため、必要なコンストラクターの依存関係は自動的に依存注入されます。

<a name="creating-monolog-handler-channels"></a>
### Monolog処理チャンネルの作成

Monologにはさまざまな[利用可能なハンドラ](https://github.com/Seldaek/monolog/tree/main/src/Monolog/Handler)があり、Laravelはそれぞれに対する組み込みチャンネルを用意していません。場合によっては、対応するLaravelログドライバを持たない特定のMonologハンドラの単なるインスタンスであるカスタムチャンネルを作成したい場合があります。これらのチャンネルは、`monolog`ドライバを使用して簡単に作成できます。

`monolog`ドライバを使用する場合、`handler`設定オプションを使用してインスタンス化するハンドラを指定します。オプションで、ハンドラが必要とするコンストラクターパラメーターは、`with`設定オプションを使用して指定できます。

    'logentries' => [
        'driver'  => 'monolog',
        'handler' => Monolog\Handler\SyslogUdpHandler::class,
        'with' => [
            'host' => 'my.logentries.internal.datahubhost.company.com',
            'port' => '10000',
        ],
    ],

<a name="monolog-formatters"></a>
#### Monologフォーマッター

`monolog`ドライバを使用する場合、Monolog`LineFormatter`がデフォルトのフォーマッターとして使用されます。ただし、`formatter`および`formatter_with`設定オプションを使用して、ハンドラへ渡すフォーマッタータイプをカスタマイズできます。

    'browser' => [
        'driver' => 'monolog',
        'handler' => Monolog\Handler\BrowserConsoleHandler::class,
        'formatter' => Monolog\Formatter\HtmlFormatter::class,
        'formatter_with' => [
            'dateFormat' => 'Y-m-d',
        ],
    ],

独自のフォーマッターを提供できるMonologハンドラを使用している場合は、`formatter`構成オプションの値を`default`に設定できます。

    'newrelic' => [
        'driver' => 'monolog',
        'handler' => Monolog\Handler\NewRelicHandler::class,
        'formatter' => 'default',
    ],

<a name="monolog-processors"></a>
#### Monolog Processors

Monologは、メッセージをログに記録する前に処理することもできます。独自のプロセッサを作成したり、[Monologが提供する既存のプロセッサ](https://github.com/Seldaek/monolog/tree/main/src/Monolog/Processor)を使用したりできます。

`monolog`ドライバのプロセッサをカスタマイズしたい場合は、チャネルの設定へ`processors`設定値を追加してください。

     'memory' => [
         'driver' => 'monolog',
         'handler' => Monolog\Handler\StreamHandler::class,
         'with' => [
             'stream' => 'php://stderr',
         ],
         'processors' => [
             // シンプルな記法
             Monolog\Processor\MemoryUsageProcessor::class,

             // 使用するオプション
             [
                'processor' => Monolog\Processor\PsrLogMessageProcessor::class,
                'with' => ['removeUsedContextFields' => true],
            ],
         ],
     ],

<a name="creating-custom-channels-via-factories"></a>
### ファクトリによるカスタムチャンネルの生成

Monologのインスタンス化と設定を完全に制御する、完全なカスタムチャンネルを定義する場合は、`config/logging.php`設定ファイルで`custom`ドライバタイプを指定します。設定には、Monologインスタンスを作成するために呼び出すファクトリクラスの名前を含む`via`オプションを含める必要があります。

    'channels' => [
        'example-custom-channel' => [
            'driver' => 'custom',
            'via' => App\Logging\CreateCustomLogger::class,
        ],
    ],

`custom`ドライバチャンネルを設定したら、Monologインスタンスを作成するクラスを定義する準備が整います。このクラスには、Monologロガーインスタンスを返す単一の`__invoke`メソッドのみが必要です。このメソッドは、チャンネル設定配列を唯一の引数として受け取ります。

    <?php

    namespace App\Logging;

    use Monolog\Logger;

    class CreateCustomLogger
    {
        /**
         * カスタムMonologインスタンスの生成
         */
        public function __invoke(array $config): Logger
        {
            return new Logger(/* ... */);
        }
    }

<a name="tailing-log-messages-using-pail"></a>
## Pailを使用したログの限定出力

多くの場合リアルタイムで、アプリケーションのログを限定し出力する必要が起きるでしょう。例えば、問題をデバッグするときや、特定の種類のエラーについてアプリケーションのログを監視するときなどです。

Laravel Pail（ペール：バケツ、手杓）は、Laravelアプリケーションのログファイルにコマンドラインから直接簡単にアクセスできるパッケージです。標準の`tail`コマンドとは異なり、PailはSentryやFlareを含むあらゆるログドライバで動作するように設計されています。さらにPailは、探しているログをすぐに見つけのに役立つ、便利なフィルタのセットを用意しています。

<img src="https://laravel.com/img/docs/pail-example.png">

<a name="pail-installation"></a>
### インストール

> [!WARNING]
> Laravel Pail requires [PHP 8.2+](https://php.net/releases/) and the [PCNTL](https://www.php.net/manual/en/book.pcntl.php) extension.

使用開始するには、Composerパッケージマネージャを使い、プロジェクトにPailをインストールしてください。

```bash
composer require laravel/pail
```

<a name="pail-usage"></a>
### 使用法

ログを限定出力するには、`pail`コマンドを実行します。

```bash
php artisan pail
```

出力の冗長性を許し、切り捨て（...）を回避するには、`-v`オプションを使用します。

```bash
php artisan pail -v
```

冗長を最大限許し、例外スタックトレースを表示するには、`-vv` オプションを使用してください。

```bash
php artisan pail -vv
```

To stop tailing logs, press `Ctrl+C` at any time.

<a name="pail-filtering-logs"></a>
### ログのフィルタリング

<a name="pail-filtering-logs-filter-option"></a>
#### `--filter`

`--filter`オプションを使うと、ログのタイプ、ファイル、メッセージ、スタックトレースの内容でログをフィルタリングできます。

```bash
php artisan pail --filter="QueryException"
```

<a name="pail-filtering-logs-message-option"></a>
#### `--message`

ログのメッセージのみをフィルタリングするには、`--message` オプションを使います。

```bash
php artisan pail --message="User created"
```

<a name="pail-filtering-logs-level-option"></a>
#### `--level`

[logレベルl](#log-levels)により、ログをフィルタリングするには、`--level`オプションを使います。

```bash
php artisan pail --level=error
```

<a name="pail-filtering-logs-user-option"></a>
#### `--user`

指定するユーザーが認証されている間に書き込まれたログだけを表示するには、`--user`オプションへそのユーザーのIDを指定します。

```bash
php artisan pail --user=1
```
