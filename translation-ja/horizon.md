# Laravel Horizon

- [イントロダクション](#introduction)
- [インストール](#installation)
    - [設定](#configuration)
    - [バランス戦略](#balancing-strategies)
    - [ダッシュボードの認可](#dashboard-authorization)
    - [非表示のジョブ](#silenced-jobs)
- [Horizonのアップグレード](#upgrading-horizon)
- [Horizonの実行](#running-horizon)
    - [Horizonのデプロイ](#deploying-horizon)
- [タグ](#tags)
- [通知](#notifications)
- [メトリクス](#metrics)
- [失敗したジョブの削除](#deleting-failed-jobs)
- [キューのジョブをクリア](#clearing-jobs-from-queues)

<a name="introduction"></a>
## イントロダクション

> [!NOTE]
> Laravel Horizo​​nを掘り下げる前に、Laravelの基本的な[キューサービス](/docs/{{version}}/queues)をよく理解しておく必要があります。Horizo​​nは、Laravelが提供する基本的なキュー機能にまだ慣れていない場合は混乱してしまう可能性がある追加機能であり、Laravelのキューを拡張します。

[Laravel Horizon](https://github.com/laravel/horizon)（ホライズン、地平線）は、Laravelを利用した[Redisキュー](/docs/{{version}}/queues)に美しいダッシュボードとコード駆動型の設定を提供します。Horizo​​nを使用すると、ジョブのスループット、ランタイム、ジョブの失敗など、キューシステムの主要なメトリックを簡単に監視できます。

Horizo​​nを使用する場合、すべてのキューワーカ設定は単一の単純な設定ファイルへ保存します。バージョン管理されたファイルでアプリケーションのワーカ設定を定義することにより、アプリケーションのデプロイ時に、キューワーカを簡単にスケーリングや変更できます。

<img src="https://laravel.com/img/docs/horizon-example.png">

<a name="installation"></a>
## インストール

> [!WARNING]
> Laravel Horizo​​nは、[Redis](https://redis.io)を使用してキューを使用する必要があります。したがって、アプリケーションの`config/queue.php`設定ファイルでキュー接続が`redis`に設定されていることを確認する必要があります。

Composerパッケージマネージャを使用して、Horizo​​nをプロジェクトにインストールします。

```shell
composer require laravel/horizon
```

Horizo​​nをインストールした後、`horizo​​n:install` Artisanコマンドを使用してアセット公開します。

```shell
php artisan horizon:install
```

<a name="configuration"></a>
### 設定

Horizo​​nのアセットを公開すると、そのプライマリ設定ファイルは`config/horizo​​n.php`へ設置されます。この設定ファイルでアプリケーションのキューワーカオプションを設定できます。各設定オプションにはその目的の説明が含まれているため、このファイルを徹底的に調べてください。

> [!WARNING]
> Horizonは内部で`horizon`という名前のRedis接続を使用します。このRedis接続名は予約語であり、`database.php`設定ファイル中で他のRedis接続に割り当てたり、`horizon.php`設定ファイルの`use`オプションの値に使用したりしてはいけません。

<a name="environments"></a>
#### 環境

インストール後に、よく理解する必要のある主要なHorizo​​n設定オプションは、`environments`設定オプションです。この設定オプションは、アプリケーションを実行する環境の配列であり、各環境のワーカプロセスオプションを定義します。デフォルトのこのエントリは`production`環境と`local`環境です。ただし、環境は必要に応じ自由に追加できます。

    'environments' => [
        'production' => [
            'supervisor-1' => [
                'maxProcesses' => 10,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
            ],
        ],

        'local' => [
            'supervisor-1' => [
                'maxProcesses' => 3,
            ],
        ],
    ],

他に一致する環境が見つからない場合に使用する、ワイルドカード環境（`*`）を定義することもできます。

    'environments' => [
        // ...

        '*' => [
            'supervisor-1' => [
                'maxProcesses' => 3,
            ],
        ],
    ],

Horizo​​nを起動すると、アプリケーションを実行する環境のワーカープロセス設定オプションが使用されます。通常、環境は`APP_ENV`[環境変数](/docs/{{version}}/configuration#determining-the-current-environment)の値によって決定されます。たとえば、デフォルトの`local` Horizo​​n環境は、３つのワーカープロセスを開始し、各キューに割り当てられたワーカプロセスの数のバランスを自動的にとるように設定されています。デフォルトの`production`環境は、最大１０個のワーカプロセスを開始し、各キューに割り当てられたワーカプロセスの数のバランスを自動的にとるように設定されています。

> [!WARNING]
> `horizo​​n`設定ファイルの`environments`部分に、Horizonを実行する予定の各[環境](/docs/{{version}}/configuration#environment-configuration)のエントリを確実に指定してください。

<a name="supervisors"></a>
#### スーパーバイザ

Horizo​​nのデフォルトの設定ファイルでわかるように。各環境には、１つ以上の「スーパーバイザ（supervisor）」を含めることができます。デフォルトでは、設定ファイルはこのスーパーバイザを`supervisor-1`として定義します。ただし、スーパーバイザには自由に名前を付けることができます。各スーパーバイザは、基本的にワーカプロセスのグループを「監視」する責任があり、キュー間でワーカプロセスのバランスを取ります。

特定の環境で実行する必要があるワーカプロセスの新しいグループを定義する場合は、指定環境にスーパーバイザを追加します。アプリケーションが使用する特定のキューへ他のバランス戦略やワーカープロセス数を指定することもできます。

<a name="maintenance-mode"></a>
#### メンテナンスモード

アプリケーションが、[メンテナンスモード](/docs/{{version}}/configuration#maintenance-mode)にあるとき、Horizon設定ファイル内のスーパーバイザの`force`オプションを`true`で定義していない限り、キューに投入するジョブをHorizonは処理しません。

    'environments' => [
        'production' => [
            'supervisor-1' => [
                // ...
                'force' => true,
            ],
        ],
    ],

<a name="default-values"></a>
#### デフォルト値

Horizo​​nのデフォルト設定ファイル内に、`defaults`設定オプションがあります。この設定オプションにアプリケーションの[スーパーバイザ](#supervisors)のデフォルト値を指定します。スーパーバイザのデフォルト設定値は、各環境のスーパーバイザの設定にマージされるため、スーパーバイザを定義するときに不必要な繰り返しを回避できます。

<a name="balancing-strategies"></a>
### バランス戦略

Laravelのデフォルトのキューシステムとは異なり、Horizo​​nでは３つのワーカーバランス戦略(`simple`、`auto`、`false`)から選択できます。`simple`戦略は、受信ジョブをワーカープロセス間で均等に分割します。

    'balance' => 'simple',

設定ファイルのデフォルトである`auto`戦略は、キューの現在のワークロードに基づいて、キューごとのワーカープロセスの数を調整します。たとえば、`render`キューが空のときに`notifications`キューに1,000の保留中のジョブがある場合、Horizo​​nはキューが空になるまで`notifications`キューにさらに多くのワーカを割り当てます。

`auto`戦略を使用する場合、`minProcesses`と`maxProcesses`設定オプションを定義し、Horizonがスケールアップ／スケールダウンするキューの最小プロセス数とワーカープロセスの最大数を制御できます。

    'environments' => [
        'production' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue' => ['default'],
                'balance' => 'auto',
                'autoScalingStrategy' => 'time',
                'minProcesses' => 1,
                'maxProcesses' => 10,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
                'tries' => 3,
            ],
        ],
    ],

`autoScalingStrategy`設定値は、Horizonがキューをクリアするのにかかる総時間（`time`戦略）、またはキュー上のジョブの総数（`size`戦略）に基づいて、より多くのワーカープロセスをキューに割り当てるかを決めます。

`balanceMaxShift`と`balanceCooldown`の設定値は、Horizo​​nがワーカの需要を満たすためにどれだけ迅速にスケーリングするかを決定します。上記の例では、３秒ごとに最大１つの新しいプロセスが作成または破棄されます。アプリケーションのニーズに基づいて、必要に応じてこれらの値を自由に調整できます。

`balance`オプションを`false`に設定している場合、デフォルトのLaravel動作が使用され、設定にリストされている順序でキューを処理します。

<a name="dashboard-authorization"></a>
### ダッシュボードの認可

Horizonダッシュボードは、`/horizon`ルートでアクセスできます。このダッシュボードはデフォルトで、`local`環境でのみアクセスできます。ですが、`app/Providers/HorizonServiceProvider.php`ファイル内には、[認可ゲート](/docs/{{version}}/authorization#gates)の定義があります。この認可ゲートは、**非ローカル**環境におけるHorizonへのアクセスを制御します。Horizonインストールへのアクセスを制限するため、このゲートを必要に応じて自由に変更してください。

    /**
     * Horizonゲートの登録
     *
     * このゲートは、非ローカル環境で誰がHorizo​​nにアクセスできるかを決定します。
     */
    protected function gate(): void
    {
        Gate::define('viewHorizon', function (User $user) {
            return in_array($user->email, [
                'taylor@laravel.com',
            ]);
        });
    }

<a name="alternative-authentication-strategies"></a>
#### その他の認証戦略

Laravelは、認証したユーザーをゲートクロージャへ自動的に依存注入することを忘れないでください。アプリケーションがIP制限など別の方法でHorizonセキュリティを提供している場合、Horizonユーザーは「ログイン」する必要がない場合があります。したがって、Laravelの認証を必要としないようにするため、上記の`function (User $user)`クロージャの引数を`function (User $user = null)`に変更する必要があるでしょう。

<a name="silenced-jobs"></a>
### 非表示のジョブ

アプリケーションやサードパーティパッケージが送信する特定のジョブを見たくない場合があるでしょう。こうしたジョブで「完了したジョブ」リスト上の領域を占有させずに、非表示にできます。最初の方法は、アプリケーションの`horizon`設定ファイルにある、`silenced`設定オプションへジョブのクラス名を追加します。

    'silenced' => [
        App\Jobs\ProcessPodcast::class,
    ],

別の方法は、表示しないジョブへ`Laravel\Horizon\Contracts\Silenced`インターフェイスを実装します。このインターフェイスを実装したジョブは、`silenced`設定配列に存在しなくても、自動的に表示しません。

    use Laravel\Horizon\Contracts\Silenced;

    class ProcessPodcast implements ShouldQueue, Silenced
    {
        use Queueable;

        // ...
    }

<a name="upgrading-horizon"></a>
## Horizonのアップグレード

Horizonの新しいメジャーバージョンへアップグレードするときは、[アップグレードガイド](https://github.com/laravel/horizon/blob/master/UPGRADE.md)を注意深く確認することが重要です。

<a name="running-horizon"></a>
## Horizonの実行

アプリケーションの`config/horizo​​n.php`設定ファイルでスーパーバイザとワーカを設定したら、`horizo​​n` Artisanコマンドを使用してHorizo​​nを起動できます。この単一のコマンドは、現在の環境用に設定されたすべてのワーカプロセスを開始します。

```shell
php artisan horizon
```

`horizo​​n:pause`と`horizo​​n:continue` Artisanコマンドで、Horizo​​nプロセスを一時停止したり、ジョブの処理を続行するように指示したりできます。

```shell
php artisan horizon:pause

php artisan horizon:continue
```

`horizo​​n:pause-supervisor`と`horizo​​n:continue-supervisor` Artisanコマンドを使用して、特定のHorizo​​n[スーパーバイザ](#supervisors)を一時停止／続行することもできます。

```shell
php artisan horizon:pause-supervisor supervisor-1

php artisan horizon:continue-supervisor supervisor-1
```

`horizo​​n:status` Artisanコマンドを使用して、Horizo​​nプロセスの現在のステータスを確認できます。

```shell
php artisan horizon:status
```

`horizo​​n:terminate` Artisanコマンドを使用して、Horizo​​nプロセスを正常に終了できます。現在処理されているジョブがすべて完了してから、Horizo​​nは実行を停止します。

```shell
php artisan horizon:terminate
```

<a name="deploying-horizon"></a>
### Horizonのデプロイ

Horizo​​nをアプリケーションの実際のサーバにデプロイする準備ができたら、`php artisan horizo​​n`コマンドを監視するようにプロセスモニタを設定し、予期せず終了した場合は再起動する必要があります。心配ありません。以下からプロセスモニタのインストール方法について説明します。

アプリケーションのデプロイメントプロセス中で、Horizo​​nプロセスへ終了するように指示し、プロセスモニターによって再起動され、コードの変更を反映するようにする必要があります。

```shell
php artisan horizon:terminate
```

<a name="installing-supervisor"></a>
#### Supervisorのインストール

SupervisorはLinuxオペレーティングシステムのプロセスモニタであり、実行が停止すると`horizon`プロセスを自動的に再起動してくれます。UbuntuにSupervisorをインストールするには、次のコマンドを使用できます。Ubuntuを使用していない場合は、オペレーティングシステムのパッケージマネージャを使用してSupervisorをインストールしてください。

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 自分でSupervisorを設定するのが難しいと思われる場合は、[Laravel Forge](https://forge.laravel.com)の使用を検討してください。これにより、LaravelプロジェクトのSupervisorは自動的にインストールおよび設定されます。

<a name="supervisor-configuration"></a>
#### Supervisor設定

Supervisor設定ファイルは通常、サーバの`/etc/supervisor/conf.d`ディレクトリ内に保管されます。このディレクトリ内に、プロセスの監視方法をスSupervisorに指示する設定ファイルをいくつでも作成できます。たとえば、`horizo​​n`プロセスを開始および監視する`horizo​​n.conf`ファイルを作成しましょう。

```ini
[program:horizon]
process_name=%(program_name)s
command=php /home/forge/example.com/artisan horizon
autostart=true
autorestart=true
user=forge
redirect_stderr=true
stdout_logfile=/home/forge/example.com/horizon.log
stopwaitsecs=3600
```

Supervisorの設定を定義する際には、`stopwaitsecs`の値が、最も長く実行されるジョブが費やす秒数より確実に大きくしてください。そうしないと、Supervisorが処理を終える前にジョブを強制終了してしまう可能性があります。

> [!WARNING]
> 上記の設定例は、Ubuntuベースのサーバで有効ですが、Supervisor設定ファイルの場所とファイル拡張子は、他のサーバオペレーティングシステムで異なる場合があります。詳細は、お使いのサーバのマニュアルを参照してください。

<a name="starting-supervisor"></a>
#### Supervisorの開始

設定ファイルを作成したら、以下のコマンドを使用して、Supervisor設定を更新し、監視対象プロセスを開始できます。

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start horizon
```

> [!NOTE]
> Supervisorの実行の詳細は、[Supervisorのドキュメント](http://supervisord.org/index.html)を参照してください。

<a name="tags"></a>
## タグ

Horizo​​nを使用すると、メール可能、ブロードキャストイベント、通知、キュー投入するイベントリスナなどのジョブに「タグ」を割り当てることができます。実際、Horizo​​nは、ジョブに関連付けられているEloquentモデルに応じて、ほとんどのジョブにインテリジェントかつ自動的にタグを付けます。たとえば、以下のジョブを見てみましょう。

    <?php

    namespace App\Jobs;

    use App\Models\Video;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Foundation\Queue\Queueable;

    class RenderVideo implements ShouldQueue
    {
        use Queueable;

        /**
         * 新しいジョブインスタンスの生成
         */
        public function __construct(
            public Video $video,
        ) {}

        /**
         * ジョブの実行
         */
        public function handle(): void
        {
            // ...
        }
    }

このジョブが`id`属性は`1`の`App\Models\Video`インスタンスでキューに投入されると、タグ`App\Models\Video:1`が自動的に付けられます。これは、Horizo​​nがジョブのプロパティでEloquentモデルを検索するためです。Eloquentモデルが見つかった場合、Horizo​​nはモデルのクラス名と主キーを使用してジョブにインテリジェントにタグを付けます。

    use App\Jobs\RenderVideo;
    use App\Models\Video;

    $video = Video::find(1);

    RenderVideo::dispatch($video);

<a name="manually-tagging-jobs"></a>
#### ジョブに手作業でタグ付ける

Queueableオブジェクトの１つにタグを手作業で定義する場合は、クラスに`tags`メソッドを定義します。

    class RenderVideo implements ShouldQueue
    {
        /**
         * ジョブに割り当てるタグを取得
         *
         * @return array<int, string>
         */
        public function tags(): array
        {
            return ['render', 'video:'.$this->video->id];
        }
    }

<a name="manually-tagging-event-listeners"></a>
#### 手作業によるイベントリスナのタグ付け

キュー投入したイベントリスナのタグを取得する場合、イベントのデータをタグへ追加できるように、Horizonは自動的にそのイベントインスタンスを`tags`メソッドへ渡します。

    class SendRenderNotifications implements ShouldQueue
    {
        /**
         * リスナへ割り付けるタグの取得
         *
         * @return array<int, string>
         */
        public function tags(VideoRendered $event): array
        {
            return ['video:'.$event->video->id];
        }
    }

<a name="notifications"></a>
## 通知

> [!WARNING]
> SlackまたはSMS通知を送信するようにHorizo​​nを設定する場合は、[関連する通知チャネルの前提条件](/docs/{{version}}/notifications)を確認する必要があります。

キューの１つに長い待機時間があったときに通知を受け取りたい場合は、`Horizo​​n::routeMailNotificationsTo`、`Horizo​​n::routeSlackNotificationsTo`、および`Horizo​​n::routeSmsNotificationsTo`メソッドが使用できます。これらのメソッドは、アプリケーションの`App\Providers\Horizo​​nServiceProvider`の`boot`メソッドから呼び出せます。

    /**
     * 全アプリケーションサービスの初期起動処理
     */
    public function boot(): void
    {
        parent::boot();

        Horizon::routeSmsNotificationsTo('15556667777');
        Horizon::routeMailNotificationsTo('example@example.com');
        Horizon::routeSlackNotificationsTo('slack-webhook-url', '#channel');
    }

<a name="configuring-notification-wait-time-thresholds"></a>
#### 待機通知の時間のしきい値の設定

アプリケーションの`config/horizo​​n.php`設定ファイル内で「長時間待機」と見なす秒数を設定できます。このファイル内の`waits`設定オプションを使用すると、各接続/キューの組み合わせの長時間待機しきい値を制御できます。未定義の接続／キューの組み合わせの、長時間待機時間のしきい値はデフォルトで６０秒です。

    'waits' => [
        'redis:critical' => 30,
        'redis:default' => 60,
        'redis:batch' => 120,
    ],

<a name="metrics"></a>
## メトリクス

Horizonは、ジョブおよびキューの待ち時間とスループットに関する情報を提供する、メトリクスダッシュボードを用意しています。このダッシュボードを表示するには、アプリケーションの`routes/console.php`ファイルで、Horizonの`snapshot` Artisanコマンドを５分ごとに実行するように設定する必要があります。

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('horizon:snapshot')->everyFiveMinutes();

<a name="deleting-failed-jobs"></a>
## 失敗したジョブの削除

失敗したジョブを削除したい場合は、`horizo​​n:forget`コマンドを使用します。`horizo​​n:forget`コマンドは、失敗したジョブのIDかUUIDを唯一の引数に取ります。

```shell
php artisan horizon:forget 5
```

<a name="clearing-jobs-from-queues"></a>
## キューのジョブをクリア

アプリケーションのデフォルトキューからすべてのジョブを削除する場合は、`horizo​​n:clear` Artisanコマンドを使用して削除します。

```shell
php artisan horizon:clear
```

特定のキューからジョブを削除するために`queue`オプションが指定できます。

```shell
php artisan horizon:clear --queue=emails
```
