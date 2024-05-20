# Laravel Pulse

- [イントロダクション](#introduction)
- [インストール](#installation)
    - [設定](#configuration)
- [ダッシュボード](#dashboard)
    - [認可](#dashboard-authorization)
    - [カスタマイズ](#dashboard-customization)
    - [ユーザーの解決](#dashboard-resolving-users)
    - [カード](#dashboard-cards)
- [エンティティのキャプチャ](#capturing-entries)
    - [レコード](#recorders)
    - [フィルタリング](#filtering)
- [パフォーマンス](#performance)
    - [他のデータベースの使用](#using-a-different-database)
    - [Redis統合](#ingest)
    - [サンプリング](#sampling)
    - [トリミング](#trimming)
    - [Pulse例外の処理](#pulse-exceptions)
- [カスタムカード](#custom-cards)
    - [カードコンポーネント](#custom-card-components)
    - [スタイル](#custom-card-styling)
    - [データキャプチャと集積](#custom-card-data)

<a name="introduction"></a>
## イントロダクション

[Laravel Pulse](https://github.com/laravel/pulse)で、アプリケーションのパフォーマンスと使用状況を一目で把握できます。Pulseを使えば、遅いジョブやエンドポイントなどのボトルネックを突き止めたり、最もアクティブなユーザーを見つけたりできます。

個々のイベントの詳細なデバッグには、[Laravel Telescope](/docs/{{version}}/telescope)をチェックしてください。

<a name="installation"></a>
## インストール

> [!WARNING]
> 現在、Pulseのファーストパーティストレージの実装には、MySQL、MariaDB、PostgreSQLデータベースが必要です。他のデータベースエンジンを使用する場合は、Pulseデータ用のため別にMySQL、MariaDB、またはPostgreSQLデータベースを必要する必要があります。

Pulseは、Composerパッケージマネージャを使ってインストールしてください。

```sh
composer require laravel/pulse
```

次に、`vendor:publish` Artisanコマンドを使用し、Pulse設定ファイルとマイグレーションファイルをリソース公開します。

```shell
php artisan vendor:publish --provider="Laravel\Pulse\PulseServiceProvider"
```

最後に、Pulseのデータを格納するテーブルを作成するため、`migrate`コマンドを実行します。

```shell
php artisan migrate
```

Pulseのデータベースマイグレーションを完了したら、`/pulse`ルートでPulseダッシュボードへアクセスできます。

> [!NOTE]
> パルスデータをアプリケーションのプライマリデータベースに保存したくない場合は、[専用のデータベース接続を指定](#using-a-different-database)できます。

<a name="configuration"></a>
### 設定

Pulseの設定オプションの多くは、環境変数を使い制御できます。利用可能なオプションを確認したり、新しいレコーダを登録したり、高度なオプションを設定したりするには、`config/pulse.php`設定ファイルをリソース公開してください：

```sh
php artisan vendor:publish --tag=pulse-config
```

<a name="dashboard"></a>
## ダッシュボード

<a name="dashboard-authorization"></a>
### 認可

Pulseダッシュボードは、`/pulse`ルートでアクセスできます。このダッシュボードにアクセスできるのは、デフォルトでは`local`環境だけなため、`'viewPulse'`認証ゲートをカスタマイズして、本番環境の認証を設定する必要があります。この設定は、アプリケーションの`app/Providers/AppServiceProvider.php`ファイルで行います。

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

/**
 * 全アプリケーションサービスの初期起動処理
 */
public function boot(): void
{
    Gate::define('viewPulse', function (User $user) {
        return $user->isAdmin();
    });

    // ...
}
```

<a name="dashboard-customization"></a>
### カスタマイズ

Pulseダッシュボードのカードとレイアウトは、ダッシュボードビューをリソース公開することで設定できます。ダッシュボードビューは、`resources/views/vendor/pulse/dashboard.blade.php`として公開します。

```sh
php artisan vendor:publish --tag=pulse-dashboard
```

ダッシュボードは、[Livewire](https://livewire.laravel.com/)を活用しており、JavaScriptアセットをまったく再構築しなくとも、カードやレイアウトをカスタマイズできます。

このファイルの中では、`<x-pulse>`コンポーネントがダッシュボードのレンダを担当し、カードのグリッドレイアウトを提供しています。ダッシュボードを画面の幅いっぱいに表示したい場合は、`full-width`プロパティをコンポーネントへ指定します。

```blade
<x-pulse full-width>
    ...
</x-pulse>
```

`<x-pulse>`コンポーネントはデフォルトで、１２カラムのグリッドを作成しますが、`cols`プロップを使ってカスタマイズ可能です。

```blade
<x-pulse cols="16">
    ...
</x-pulse>
```

各カードは、`cols`と`rows`プロップを引数に取り、スペースと位置をコントロールできます。

```blade
<livewire:pulse.usage cols="4" rows="2" />
```

ほとんどのカードは、`expand`プロップも引数に取り、スクロールする代わりにカード全体を表示します。

```blade
<livewire:pulse.slow-queries expand />
```

<a name="dashboard-resolving-users"></a>
### ユーザーの解決

アプリケーション使用状況カードなど、ユーザーに関する情報を表示するカードの場合、PulseはユーザーのIDのみ記録します。ダッシュボードをレンダする際、Pulseはデフォルトの`Authenticatable`モデルから、`name`フィールドと`email`フィールドを解決し、Gravatarウェブサービスを使用してアバターを表示します。

アプリケーションの`App\Providers\AppServiceProvider`クラス内の、`Pulse::user`メソッドを呼び出せば、フィールドとアバターをカスタマイズできます。

`user`メソッドはクロージャを引数に取ります。そのクロージャは表示する`Authenticatable`モデルを引数に受け、ユーザーの`name`、`extra`、`avatar`情報を含む配列を返す必要があります。

```php
use Laravel\Pulse\Facades\Pulse;

/**
 * 全アプリケーションサービスの初期起動処理
 */
public function boot(): void
{
    Pulse::user(fn ($user) => [
        'name' => $user->name,
        'extra' => $user->email,
        'avatar' => $user->avatar_url,
    ]);

    // ...
}
```

> [!NOTE]
> `Laravel\Pulse\Contracts\ResolvesUsers`契約を実装し、Laravelの[サービスコンテナ](/docs/{{version}}/container#binding-a-singleton)でバインドすることで、認証されたユーザーを取得する方法を完全にカスタマイズできます。

<a name="dashboard-cards"></a>
### カード

<a name="servers-card"></a>
#### サーバ

`<livewire:pulse.servers />`カードは、`pulse:check`コマンドを実行しているすべてのサーバのシステムリソースの使用状況を表示します。システムリソースのレポートについては、[サーバレコーダ](#servers-recorder) のドキュメントを参照してください。

インフラのサーバをリプレースしている場合、非アクティブなサーバをPulseダッシュボードへ表示しないようにしたい場合が起きるかと思います。この場合は、`ignore-after`プロップを使用します。このプロップには、非アクティブなサーバをPulseダッシュボードから削除するまでの秒数を指定します。もしくは、`1 hour`や`3 days and 1 hour`のように、相対時間形式の文字列を指定することもできます。

```blade
<livewire:pulse.servers ignore-after="3 hours" />
```

<a name="application-usage-card"></a>
#### アプリケーション使用状況

`<livewire:pulse.usage />`カードは、アプリケーションへのリクエスト、ジョブのディスパッチ、遅いリクエストを作り出した上位１０ユーザーを表示します。

すべての使用量メトリクスを同時に画面に表示したい場合は、カードを複数インクルードし、`type`属性を指定してください。

```blade
<livewire:pulse.usage type="requests" />
<livewire:pulse.usage type="slow_requests" />
<livewire:pulse.usage type="jobs" />
```

Pulseがユーザー情報を取得・表示する方法をカスタマイズする方法については、[ユーザーの解決](#dashboard-resolving-users)のドキュメントを参照してください。

> [!NOTE]
> アプリケーションがたくさんのリクエストを受け取ったり、たくさんのジョブをディスパッチしたりする場合は、[サンプリング](#sampling)を有効にするとよいでしょう。詳しくは、[ユーザーリクエストのレコーダ](#user-requests-recorder)、[ユーザージョブのレコーダ](#user-jobs-recorder)、[遅いジョブのレコーダ](#slow-jobs-recorder)のドキュメントを参照してください。

<a name="exceptions-card"></a>
#### 例外

`<livewire:pulse.exceptions />`カードは、アプリケーションで発生した例外の発生頻度と間隔を表示します。デフォルトでは、例外は例外クラスと発生場所に基づいてグループ化されます。詳しくは [例外レコーダ](#exceptions-recorder)のドキュメントを参照してください。

<a name="queues-card"></a>
#### キュー

`<livewire:pulse.queues />`カードは、キュー投入されたジョブ、処理中、処理済み、解放、失敗などの数を含む、アプリケーション内のキューのスループットを表示します。詳細は [キューレコーダ](#queues-recorder) のドキュメントを参照してください。

<a name="slow-requests-card"></a>
#### スローリクエスト

`<livewire:pulse.slow-requests />`カードは、設定したしきい値(デフォルトでは1,000ms)を超えたアプリケーションへのリクエストを表示します。詳しくは、[スローリクエストレコーダ](#slow-requests-recorder)のドキュメントを参照してください。

<a name="slow-jobs-card"></a>
#### スロージョブ

`<livewire:pulse.slow-jobs />`カードは、設定したしきい値(デフォルトでは1,000ms)を超えたアプリケーションのキュー投入したジョブを表示します。詳しくは、[スロージョブレコーダ](#slow-jobs-recorder)のドキュメントを参照してください。

<a name="slow-queries-card"></a>
#### スロークエリ

`<livewire:pulse.slow-queries />`カードは、設定したしきい値（デフォルトでは1,000ms）を超えるアプリケーションのデータベースクエリを表示します。

スロークエリはデフォルトで、ＳＱＬクエリ（バインディングなし）と発生場所に基づいてグループ化されますが、ＳＱＬクエリのみでグループ化したい場合は、発生場所をキャプチャしないこともできます。

非常に大きなSQLクエリがシンタックスハイライトのため、レンダーのパフォーマンスに問題が発生する場合は、`without-highlighting`プロパティを追加し、ハイライトを無効にできます。

```blade
<livewire:pulse.slow-queries without-highlighting />
```

詳しくは[スロークエリレコーダ](#slow-queries-recorder)のドキュメントを参照してください。

<a name="slow-outgoing-requests-card"></a>
#### スロー送信リクエスト

`<livewire:pulse.slow-outgoing-requests />`カードは、Laravelの[HTTPクライアント](/docs/{{version}}/http-client)を使用して送信したたリクエストのうち、設定したしきい値（デフォルトでは1,000ms）を超えたものを表示します。

エントリはデフォルトで、完全なURLでグループ化されます。しかし、正規表現を使うことで、似たようなリクエストを正規化したり、グループ化したりできます。詳しくは [スロー送信リクエストレコーダ](#slow-outgoing-requests-recorder)のドキュメントを参照してください。

<a name="cache-card"></a>
#### キャッシュ

`<livewire:pulse.cache />`カードは、アプリケーションのキャッシュのヒットとミスの統計を、グローバルと個々のキーの両方で表示します。

エントリはデフォルトで、キーによりグループ化されます。ただし、正規表現を使用して、類似したキーを正規化またはグループ化できます。詳細は[キャッシュ操作レコーダ](#cache-interactions-recorder)のドキュメントを参照してください。

<a name="capturing-entries"></a>
## エンティティのキャプチャ

ほとんどのPulseレコーダは、Laravelが発行するフレームワークイベントに基づき、自動的にエントリーをキャプチャします。しかし、[サーバレコーダ](#servers-recorder)やいくつかのサードパーティカードは、定期的に情報をポーリングする必要があります。これらのカードを使用するには、個々のアプリケーションサーバ上で`pulse:check`デーモンを実行する必要があります：

```php
php artisan pulse:check
```

> [!NOTE]
> `pulse:check`プロセスをバックグラウンドで永続的に実行し続けるには、Supervisorのようなプロセスモニタを使い、コマンドの実行が止まらないようにする必要があります。

`pulse:check`コマンドは起動終了しないプロセスなため、再起動しないとコードベースの変更が現れません。アプリケーションのデプロイ処理中に、`pulse:restart`コマンドを呼び出し、コマンドを悠然と再起動する必要があります。

```sh
php artisan pulse:restart
```

> [!NOTE]
> Pulseは再起動シグナルを保持するために[キャッシュ](/docs/{{version}}/cache)を使用します。そのため、この機能を使用する前に、アプリケーション用にキャッシュドライバを適切に設定済みであることを確認する必要があります。

<a name="recorders"></a>
### レコーダ

レコーダは、Pulseデータベースに記録するため、アプリケーションからのエントリをキャプチャする役割を果たします。レコーダは[Pulse設定ファイル](#configuration)の`recorders`セクションで登録・設定します。

<a name="cache-interactions-recorder"></a>
#### キャッシュ操作

`CacheInteractions`レコーダは、アプリケーションで発生した[キャッシュ](/docs/{{version}}/cache)のヒットとミスに関する情報を取得し、[キャッシュ](#cache-card)カードに表示します。

オプションで[サンプルレート](#sampling)と無視するキーのパターンを調整できます。

また、キーのグループ化を設定し、類似のキーを同じエントリとしてグループ化することもできます。例えば、同じタイプの情報をキャッシュするキーから、一意のIDを削除したい場合などです。グループは、キーの一部を「FindしてReplaceする」正規表現を使い設定する。一例は設定ファイルに含まれています。

```php
レコード\CacheInteractions::class => [
    // ...
    'groups' => [
        // '/:\d+/' => ':*',
    ],
],
```

最初にマッチしたパターンを適用します。マッチするパターンがなければ、キーをそのままキャプチャします。

<a name="exceptions-recorder"></a>
#### 例外

`Exceptions`レコーダは、アプリケーションで発生した報告可能(reportable)な例外に関する情報を記録し、[例外](#exceptions-card)カードに表示します。

オプションで[サンプルレート](#sampling)と無視する例外パターンを調整できます。例外が発生した場所をキャプチャするかどうかも設定可能です。キャプチャした場所は、Pulseダッシュボードに表示され、例外の発生元を追跡するのに役立ちます。ただし、同じ例外が複数の場所で発生した場合は、固有の場所ごとに複数回表示します。

<a name="queues-recorder"></a>
#### キュー

`Queues`レコーダーはアプリケーションのキューに関する情報を記録し、[キュー](#queues-card)に表示します。

オプションで[サンプルレート](#sampling)と無視するジョブパターンを調整できます。

<a name="slow-jobs-recorder"></a>
#### スロージョブ

`SlowJobs`レコーダーは、アプリケーションで発生したスロージョブの情報を[スロージョブ](#slow-jobs-recorder)カードに表示するためにキャプチャします。

オプションで、スロージョブのしきい値、[サンプル・レート](#sampling)、無視するジョブのパターンを調整できます。

他よりも時間がかかると予想されるジョブがあるかもしれません。その場合、ジョブごとにしきい値を設定してください。

```php
Recorders\SlowJobs::class => [
    // ...
    'threshold' => [
        '#^App\Jobs\GenerateYearlyReports$#' => 5000,
        'default' => env('PULSE_SLOW_JOBS_THRESHOLD', 1000),
    ],
],
```

ジョブのクラス名にマッチする正規表現パターンがない場合は、`'default'`値を使用します。

<a name="slow-outgoing-requests-recorder"></a>
#### スロー送信リクエスト

`SlowOutgoingRequests`レコーダーは、Laravelの[HTTPクライアント](/docs/{{version}}/http-client)を使用して行われた送信HTTPリクエストのうち、設定した閾値を超えたリクエストに関する情報を取得し、[スロー送信リクエスト](#slow-outgoing-requests-card)カードに表示します。

オプションで、スロー送信リクエストのしきい値、[サンプルレート](#sampling)、無視するURLパターンを調整できます。

他より時間がかかると予想される送信リクエストがあるかもしれません。その場合、リクエスト毎のしきい値を設定してください。

```php
Recorders\SlowOutgoingRequests::class => [
    // ...
    'threshold' => [
        '#backup.zip$#' => 5000,
        'default' => env('PULSE_SLOW_OUTGOING_REQUESTS_THRESHOLD', 1000),
    ],
],
```

リクエストのURLにマッチする正規表現パターンがない場合は、`'default'`値を使います。

また、URLのグループ化を設定して、類似のURLを同じエントリとしてグループ化することもできます。例えば、URLパスから一意のIDを削除したり、ドメインのみでグループ化したりすることができます。グループは、URLの一部を「FindしてReplaceする」正規表現を使用して設定します。一例を設定ファイルに用意しています。

```php
Recorders\SlowOutgoingRequests::class => [
    // ...
    'groups' => [
        // '#^https://api\.github\.com/repos/.*$#' => 'api.github.com/repos/*',
        // '#^https?://([^/]*).*$#' => '\1',
        // '#/\d+#' => '/*',
    ],
],
```

最初にマッチしたパターンを使用します。マッチするパターンがない場合、URLをそのままキャプチャします。

<a name="slow-queries-recorder"></a>
#### スロークエリ

`SlowQueries`レコーダは、[スロークエリ](#slow-queries-card)カードに表示するため、設定したしきい値を超えるアプリケーションのデータベースクエリをキャプチャします。

オプションで、スロークエリのしきい値、[サンプルレート](#sampling)、無視するクエリパターンを調整できます。また、クエリの場所をキャプチャするかも設定可能です。キャプチャした場所は、Pulseダッシュボードに表示され、クエリの発信元を追跡するのに役立ちますが、同じクエリが複数の場所で行われた場合は、それぞれの場所で複数回表示します。

他よりも時間がかかると予想されるクエリがあるかもしれません。その場合、クエリ毎にしきい値を設定してください。

```php
Recorders\SlowQueries::class => [
    // ...
    'threshold' => [
        '#^insert into `yearly_reports`#' => 5000,
        'default' => env('PULSE_SLOW_QUERIES_THRESHOLD', 1000),
    ],
],
```

クエリのSQLにマッチする正規表現パターンがない場合は、`'default'`値を使います。

<a name="slow-requests-recorder"></a>
#### スローリクエスト

`Requests`レコーダは、アプリケーションへのリクエストに関する情報を記録し、[スローリクエスト](#slow-requests-card)カードと[アプリケーションの使用状況](#application-usage-card)カードに表示します。

オプションで、スロールートのしきい値、[サンプルレート](#sampling)、無視するパスを調整できます。

他よりも時間がかかると予想されるリクエストがあるかもしれません。その場合は、リクエスト毎にしきい値を設定してください。

```php
Recorders\SlowRequests::class => [
    // ...
    'threshold' => [
        '#^/admin/#' => 5000,
        'default' => env('PULSE_SLOW_REQUESTS_THRESHOLD', 1000),
    ],
],
```

リクエストのURLにマッチする正規表現パターンがない場合は、`'default'`値を使います。

<a name="servers-recorder"></a>
#### サーバ

`Servers`レコーダは、アプリケーションを動かしているサーバのCPU、メモリ、ストレージの使用状況をキャプチャし、[サーバ](#servers-card)カードに表示します。このレコーダを使うには、[`pulse:check`コマンド](#capturing-entries)を監視する各サーバ上で実行する必要があります。

報告する各サーバは、一意な名前を持っていなければなりません。Pulseはデフォルトで、PHPの`gethostname`関数が返す値を使用します。これをカスタマイズしたい場合は、`PULSE_SERVER_NAME`環境変数を設定します。

```env
PULSE_SERVER_NAME=load-balancer
```

Pulse設定ファイルでは、監視するディレクトリをカスタマイズすることもできます。

<a name="user-jobs-recorder"></a>
#### ユーザージョブ

`UserJobs`レコーダは、[アプリケーションの使用状況](#application-usage-card)カードに表示するために、アプリケーションでジョブをディスパッチしているユーザーに関する情報を取得します。

オプションで[サンプルレート](#sampling)と無視するジョブパターンを調整できます。

<a name="user-requests-recorder"></a>
#### ユーザーリクエスト

`UserRequests`レコーダは、あなたのアプリケーションにリクエストをしているユーザーに関する情報を取得し、[アプリケーションの使用状況](#application-usage-card)カードに表示します。

オプションで[サンプルレート](#sampling)と無視するジョブパターンを調整できます。

<a name="filtering"></a>
### フィルタリング

これまで見てきたように、多くの[レコーダ](#recorder)は設定により、リクエストURLのような値に基づき、受信エントリを「無視」する機能を提供しています。しかし、時には他の要素、例えば現在認証済みのユーザーに基づいてレコードをフィルタリングすることが役立つこともあるでしょう。そのようにレコードをフィルタリングするには、Pulseの`filter`メソッドへクロージャを渡します。通常、`filter`メソッドはアプリケーションの`AppServiceProvider`の`boot`メソッド内で呼び出す必要があります。

```php
use Illuminate\Support\Facades\Auth;
use Laravel\Pulse\Entry;
use Laravel\Pulse\Facades\Pulse;
use Laravel\Pulse\Value;

/**
 * 全アプリケーションサービスの初期起動処理
 */
public function boot(): void
{
    Pulse::filter(function (Entry|Value $entry) {
        return Auth::user()->isNotAdmin();
    });

    // ...
}
```

<a name="performance"></a>
## パフォーマンス

Pulseはインフラを追加することなく、既存のアプリケーションに組み込めるように設計しています。しかし、トラフィックの多いアプリケーションでは、Pulseがアプリケーションのパフォーマンスに与える影響を取り除く方法がいくつかあります。

<a name="using-a-different-database"></a>
### 他のデータベースの使用

トラフィックの多いアプリケーションでは、アプリケーション・データベースへの影響を避けるため、Pulse専用データベース接続を使用を望まれるでしょう。

`PULSE_DB_CONNECTION`環境変数を設定すれば、Pulseで使用する[データベース接続](/docs/{{version}}/database#configuration)をカスタマイズできます。

```env
PULSE_DB_CONNECTION=pulse
```

<a name="ingest"></a>
### Redis統合

> [!WARNING]
> Redisを使用するには、Redis6.2以上と、アプリケーションの設定済みRedisクライアントドライバに、`phpredis`または`predis`が必要です。

Pulseはデフォルトで、HTTPレスポンスがクライアントに送信した後、またはジョブを処理した後に、[設定されたデータベース接続](#using-a-different-database)に直接エントリを保存します。しかし、PulseのRedisインジェストドライバを代わりに使用し、Redisストリームへエントリを送信することもできます。これは`PULSE_INGEST_DRIVER`環境変数を設定すると有効になります。

```
PULSE_INGEST_DRIVER=redis
```

Pulseはデフォルトの[Redis接続](/docs/{{version}}/redis#configuration)を使いますが、`PULSE_REDIS_CONNECTION`環境変数を使い、カスタマイズすることもできます。

```
PULSE_REDIS_CONNECTION=pulse
```

Redisインジェストを使用する場合は、`pulse:work`コマンドを実行する必要があります。ストリームを監視し、RedisからPulseのデータベーステーブルへエントリを移動させるためにです。

```php
php artisan pulse:work
```

> [!NOTE]
> `pulse:work`プロセスをバックグラウンドで永久に実行し続けるには、Supervisorのようなプロセスモニタを使い、Pulseワーカの実行を止めないようにする必要があります。

`pulse:work`コマンドは起動終了しないプロセスなため、再起動しないとコードベースの変更が現れません。アプリケーションのデプロイ処理中に、`pulse:restart`コマンドを呼び出し、コマンドを悠然と再起動する必要があります。

```sh
php artisan pulse:restart
```

> [!NOTE]
> Pulseは再起動シグナルを保持するために[キャッシュ](/docs/{{version}}/cache)を使用します。そのため、この機能を使用する前に、アプリケーション用にキャッシュドライバを適切に設定済みであることを確認する必要があります。

<a name="sampling"></a>
### サンプリング

Pulseはデフォルトで、アプリケーションで発生するすべての関連イベントをキャプチャします。トラフィックの多いアプリケーションの場合、特に長期間では、ダッシュボードに何百万ものデータベース行を集約する必要が生じる可能性があります。

その代わりに、特定のPulseデータレコーダでは「サンプリング」を有効にできます。例えば、[`ユーザーリクエスト`](#user-requests-recorder)レコーダのサンプルレートを`0.1`に設定すると、アプリケーションへのリクエストの約10%だけを記録することになります。ダッシュボードでは、値はスケールアップされ、近似値であることを示すために、`~`を先頭に付けています。

一般的に、特定のメトリックのエントリが多いほど、精度を犠牲にすることなく、サンプルレートを低く設定できます。

<a name="trimming"></a>
### トリミング

一度ダッシュボードのウィンドウの外に出ると、Pulseは保存したエントリーを自動的にトリミングします。トリミングは、Pulseの[設定ファイル](#configuration)でカスタマイズできる抽選システムを使ってデータを取り込むときに行われます。

<a name="pulse-exceptions"></a>
### Pulse例外の処理

保存域のデータベースに接続できないなど、Pulseデータの取り込み中に例外が発生した場合、Pulseはアプリケーションに影響を与えないように静かに失敗します。

例外の処理方法をカスタマイズしたい場合は、`handleExceptionsUsing`メソッドにクロージャを指定してください。

```php
use Laravel\Pulse\Facades\Pulse;
use Illuminate\Support\Facades\Log;

Pulse::handleExceptionsUsing(function ($e) {
    Log::debug('An exception happened in Pulse', [
        'message' => $e->getMessage(),
        'stack' => $e->getTraceAsString(),
    ]);
});
```

<a name="custom-cards"></a>
## カスタムカード

Pulseでは、アプリケーションの特定のニーズに関連したデータを表示するためのカスタムカードを構築できます。Pulseは[Livewire](https://livewire.laravel.com)を使うので、最初のカスタムカードを作る前に[そのドキュメントを確認](https://livewire.laravel.com/docs)してください。

<a name="custom-card-components"></a>
### カードコンポーネント

Laravel Pulseでカスタムカードを作成するには、ベースとなる`Card` Livewireコンポーネントを拡張し、対応するビューを定義することから始めます。

```php
namespace App\Livewire\Pulse;

use Laravel\Pulse\Livewire\Card;
use Livewire\Attributes\Lazy;

#[Lazy]
class TopSellers extends Card
{
    public function render()
    {
        return view('livewire.pulse.top-sellers');
    }
}
```

Livewireの[遅延ロード](https://livewire.laravel.com/docs/lazy)機能を使用する場合、`Card`コンポーネントは、コンポーネントに渡された`cols`と`rows`属性を尊重したプレースホルダを自動的に提供します。

Pulseカードに対応するビューを記述するときは、PulseのBladeコンポーネントを活用して、一貫したルック＆フィールを実現できます。

```blade
<x-pulse::card :cols="$cols" :rows="$rows" :class="$class" wire:poll.5s="">
    <x-pulse::card-header name="Top Sellers">
        <x-slot:icon>
            ...
        </x-slot:icon>
    </x-pulse::card-header>

    <x-pulse::scroll :expand="$expand">
        ...
    </x-pulse::scroll>
</x-pulse::card>
```

ダッシュボードビューからカードのレイアウトをカスタマイズできるように、`$cols`、`$rows`、`$class`、`$expand`変数は、個別のBladeコンポーネントへ渡す必要があります。また、カードが自動的に更新されるように、ビューへ`wire:poll.5s=""`属性を含めてください。

Livewireコンポーネントとテンプレートを定義したら、そのカードを[ダッシュボードビュー](#dashboard-customization)に含めることができます。

```blade
<x-pulse>
    ...

    <livewire:pulse.top-sellers cols="4" />
</x-pulse>
```

> [!NOTE]
> カードがパッケージに含まれている場合、`Livewire::component`メソッドを使用してコンポーネントをLivewireに登録する必要があります。

<a name="custom-card-styling"></a>
### スタイル

Pulseに含まれているクラスやコンポーネント以上のスタイリングが必要な場合、カードにカスタムCSSを含めるための選択肢がいくつかあります。

<a name="custom-card-styling-vite"></a>
#### Laravel Vite統合

カスタムカードがアプリケーションのコードベース内にあり、Laravelの[Vite統合](/docs/{{version}}/vite)を使用している場合、カード専用のCSSエントリーポイントを含めるために、`vite.config.js`ファイルを更新してください。

```js
laravel({
    input: [
        'resources/css/pulse/top-sellers.css',
        // ...
    ],
}),
```

次に、[ダッシュボードビュー](#dashboard-customization)内で`@vite` Bladeディレクティブを使用し、カードのCSSエントリポイントを指定してください。

```blade
<x-pulse>
    @vite('resources/css/pulse/top-sellers.css')

    ...
</x-pulse>
```

<a name="custom-card-styling-css"></a>
#### CSSファイル

パッケージ内にPulseカードを持つ構成を含む他の使用例では、CSSファイルへのファイルパスを返す`css`メソッドをLivewireコンポーネントに定義することで、Pulseに追加のスタイルシートを読み込むように指示できます。

```php
class TopSellers extends Card
{
    // ...

    protected function css()
    {
        return __DIR__.'/../../dist/top-sellers.css';
    }
}
```

このカードがダッシュボードに含まれる場合、Pulseは自動的にこのファイルの内容を`<style>`タグ内に含めますので、`public`ディレクトリで公開する必要はありません。

<a name="custom-card-styling-tailwind"></a>
#### Tailwind CSS

Tailwind CSSを使用する場合は、不要なCSSを読み込んだり、PulseのTailwindクラスと競合しないように、専用のTailwind設定ファイルを作成する必要があります。

```js
export default {
    darkMode: 'class',
    important: '#top-sellers',
    content: [
        './resources/views/livewire/pulse/top-sellers.blade.php',
    ],
    corePlugins: {
        preflight: false,
    },
};
```

それから、CSSエントリーポイントに設定ファイルを指定します。

```css
@config "../../tailwind.top-sellers.config.js";
@tailwind base;
@tailwind components;
@tailwind utilities;
```

また、カードのビューに、Tailwindの[`important`セレクタ戦略](https://tailwindcss.com/docs/configuration#selector-strategy)へ渡したセレクタにマッチする`id`属性または`class`属性を含める必要があります。

```blade
<x-pulse::card id="top-sellers" :cols="$cols" :rows="$rows" class="$class">
    ...
</x-pulse::card>
```

<a name="custom-card-data"></a>
### データキャプチャと集積

カスタムカードはどこからでもデータを取得し表示できますが、Pulseの強力で効率的なデータ記録・集計システムを活用したくなるでしょう。

<a name="custom-card-data-capture"></a>
#### エンティティのキャプチャ

Pulseは、`Pulse::record`メソッドを使い、「エントリ」を記録しています。

```php
use Laravel\Pulse\Facades\Pulse;

Pulse::record('user_sale', $user->id, $sale->amount)
    ->sum()
    ->count();
```

`record`メソッドに渡す最初の引数は、記録するエントリの`type`で、第２引数は集計したデータをどのようにグループ化するかを決定するための`key`です。ほとんどの集約メソッドでは、集約する値（`value`）も指定する必要があります。上記例では、集約値は`$sale->amount`です。その後、（`sum`のような）集約メソッドを１つ以上呼び出すことで、Pulseは集約前の値を「バケット」に取り込み、後で効率的に取り出します。

利用可能な集計方法は以下の通りです。

* `avg`
* `count`
* `max`
* `min`
* `sum`

> [!NOTE]
> 現在認証済みのユーザーIDをキャプチャするカードパッケージを構築するときは、`Pulse::resolveAuthenticatedUserId()` メソッドを使うべきでしょう。これは、アプリケーションに対して行う、[ユーザーリゾルバのカスタマイズ](#dashboard-resolving-users)を尊重します。

<a name="custom-card-data-retrieval"></a>
#### 集積データの取得

Pulseの`Card` Livewireコンポーネントを拡張する場合、`aggregate`メソッドを使用して、ダッシュボードで表示する期間の集計データを取得できます。

```php
class TopSellers extends Card
{
    public function render()
    {
        return view('livewire.pulse.top-sellers', [
            'topSellers' => $this->aggregate('user_sale', ['sum', 'count']);
        ]);
    }
}
```

`aggregate`メソッドは、PHPの`stdClass`オブジェクトのコレクションを返します。各オブジェクトは、先に取得した`key`プロパティと、リクエストした集約キーを含みます。

```
@foreach ($topSellers as $seller)
    {{ $seller->key }}
    {{ $seller->sum }}
    {{ $seller->count }}
@endforeach
```

Pulseは事前に集約したバケットからデータを主に取得します。したがって、指定する集約は`Pulse::record`メソッドを使用して事前にキャプチャしておく必要があります。通常、最も古いバケットは部分的に期間外になっているため、Pulseはギャップを埋めるために、最も古いエントリを集約して、各ポーリングリクエストで期間全体を集約する必要なく、期間全体の正確な値を提供します。

`aggregateTotal`メソッドを使用して、指定するタイプの合計値を取得することもできます。たとえば、次のメソッドでは、ユーザーごとにグループ化するのではなく、全ユーザーの売上合計を取得します。

```php
$total = $this->aggregateTotal('user_sale', 'sum');
```

<a name="custom-card-displaying-users"></a>
#### ユーザーの表示

キーとしてユーザーIDを記録する集約を扱う場合、`Pulse::resolveUsers`メソッドを使用し、キーをユーザーレコードにして解決できます。

```php
$aggregates = $this->aggregate('user_sale', ['sum', 'count']);

$users = Pulse::resolveUsers($aggregates->pluck('key'));

return view('livewire.pulse.top-sellers', [
    'sellers' => $aggregates->map(fn ($aggregate) => (object) [
        'user' => $users->find($aggregate->key),
        'sum' => $aggregate->sum,
        'count' => $aggregate->count,
    ])
]);
```

`find`メソッドは`name`、`extra`、`avatar` のキーを含むオブジェクトを返します。このオブジェクトはオプションとして、`<x-pulse::user-card>` Bladeコンポーネントへ直接渡せます。

```blade
<x-pulse::user-card :user="{{ $seller->user }}" :stats="{{ $seller->sum }}" />
```

<a name="custom-recorders"></a>
#### カスタムレコーダ

パッケージの制作者は、ユーザーがデータの取り込みを設定できるようにレコーダクラスの提供を望むかもしれません。

レコーダは、アプリケーションの設定ファイル`config/pulse.php`の`recorders`セクションで登録しています。

```php
[
    // ...
    'recorders' => [
        Acme\Recorders\Deployments::class => [
            // ...
        ],

        // ...
    ],
]
```

レコーダは、`$listen`プロパティを指定することで、イベントをリッスンします。Pulseは自動的にリスナを登録し、レコーダーの`record`メソッドを呼び出します：

```php
<?php

namespace Acme\Recorders;

use Acme\Events\Deployment;
use Illuminate\Support\Facades\Config;
use Laravel\Pulse\Facades\Pulse;

class Deployments
{
    /**
     * リッスンするイベント
     *
     * @var array<int, class-string>
     */
    public array $listen = [
        Deployment::class,
    ];

    /**
     * デプロイメントを記録
     */
    public function record(Deployment $event): void
    {
        $config = Config::get('pulse.recorders.'.static::class);

        Pulse::record(
            // ...
        );
    }
}
```
