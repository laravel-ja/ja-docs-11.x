# キャッシュ

- [イントロダクション](#introduction)
- [設定](#configuration)
    - [ドライバ要件](#driver-prerequisites)
- [キャッシュ使用法](#cache-usage)
    - [キャッシュインスタンスの取得](#obtaining-a-cache-instance)
    - [キャッシュからのアイテム取得](#retrieving-items-from-the-cache)
    - [キャッシュへのアイテム保存](#storing-items-in-the-cache)
    - [キャッシュからのアイテム削除](#removing-items-from-the-cache)
    - [キャッシュヘルパ](#the-cache-helper)
- [アトミックロック](#atomic-locks)
    - [ロック管理](#managing-locks)
    - [プロセス間でのロック管理](#managing-locks-across-processes)
- [カスタムキャッシュドライバの追加](#adding-custom-cache-drivers)
    - [ドライバの作成](#writing-the-driver)
    - [ドライバの登録](#registering-the-driver)
- [イベント](#events)

<a name="introduction"></a>
## イントロダクション

アプリケーションによって実行されるデータ取得または処理タスクの一部は、ＣＰＵに負荷がかかるか、完了するまでに数秒かかる場合があります。この場合、取得したデータを一時的にキャッシュして、同じデータに対する後続のリクエストですばやく取得できるようにするのが一般的です。キャッシュするデータは通常、[Memcached](https://memcached.org)や[Redis](https://redis.io)などの非常に高速なデータストアに保存します。

幸いLaravelはさまざまなキャッシュバックエンドに表現力豊かで統一されたAPIを提供し、その超高速データ取得を利用してWebアプリケーションを高速化できるようにします。

<a name="configuration"></a>
## 設定

アプリケーションのキャッシュ設定ファイルは、`config/cache.php`にあります。アプリケーション全体でデフォルトとして使用するキャッシュストアをこのファイルで指定します。Laravelは、[Memcached](https://memcached.org)、[Redis](https://redis.io)、[DynamoDB](https://aws.amazon.com/dynamodb)、リレーショナルデータベースなど一般的なキャッシュバックエンドをサポートしています。さらに、ファイルベースのキャッシュドライバも利用でき、`array`キャッシュドライバと"null"キャッシュドライバは、自動テストに便利なキャッシュバックエンドを提供します。

キャッシュ設定ファイルには、他にも様々なオプションがあるので確認してください。Laravelはデフォルトで、`database`キャッシュドライバを使用するように設定してあり、シリアライズ済みのキャッシュオブジェクトをアプリケーションのデータベースに保存します。

<a name="driver-prerequisites"></a>
### ドライバ要件

<a name="prerequisites-database"></a>
#### データベース

`database`キャッシュドライバを使用する場合、キャッシュデータを格納するデータベーステーブルが必要になります。通常、これはLaravelの`0001_01_01_000001_create_cache_table.php` [データベースマイグレーション](/docs/{{version}}/migrations)にデフォルトで含まれていますが、アプリケーションにこのマイグレーションが含まれていない場合は、`make:cache-table` Artisanコマンドを使用して生成してください。

```shell
php artisan make:cache-table

php artisan migrate
```

<a name="memcached"></a>
#### Memcached

Memcachedドライバを使用するには、[Memcached PECLパッケージ](https://pecl.php.net/package/memcached)がインストールされている必要があります。すべてのMemcachedサーバを`config/cache.php`設定ファイルにリストしてください。このファイルには、設定しやすいように`memcached.servers`エントリがはじめから用意しています。

    'memcached' => [
        // …

        'servers' => [
            [
                'host' => env('MEMCACHED_HOST', '127.0.0.1'),
                'port' => env('MEMCACHED_PORT', 11211),
                'weight' => 100,
            ],
        ],
    ],

必要に応じて、`host`オプションをUNIXソケットパスに設定できます。これを行う場合は、`port`オプションを`0`に設定する必要があります。

    'memcached' => [
        // …

        'servers' => [
            [
                'host' => '/var/run/memcached/memcached.sock',
                'port' => 0,
                'weight' => 100
            ],
        ],
    ],

<a name="redis"></a>
#### Redis

LaravelでRedisキャッシュを使用する前に、PECL経由でPhpRedis PHP拡張をインストールするか、Composer経由で`predis/predis`パッケージ（~2.0）をインストールする必要があります。[Laravel Sail](/docs/{{version}}/sail)はあらかじめ、この拡張機能をふくんでいます。さらに、[Laravel Forge](https://forge.laravel.com)や[Laravel Vapor](https://vapor.laravel.com)などの公式Laravelデプロイメントプラットフォームには、デフォルトでPhpRedis拡張をインストールしています。

Redisの設定の詳細については、[Laravelドキュメントページ](/docs/{{version}}/redis#configuration)を参照してください。

<a name="dynamodb"></a>
#### DynamoDB

[DynamoDB](https://aws.amazon.com/dynamodb)キャッシュドライバを使用する前に、すべてのキャッシュデータを格納するDynamoDBテーブルを作成する必要があります。通常、このテーブルは`cache`という名前にします。ただし、`cache`設定ファイル内の`stores.dynamodb.table`設定値に基づいてテーブル名を付ける必要があります。テーブル名は`DYNAMODB_CACHE_TABLE`環境変数で設定することもできます。

このテーブルには、アプリケーションの`cache`設定ファイル内の`stores.dynamodb.attributes.key`設定項目の値に対応する名前の、文字列パーティションキーもあります。デフォルトでは、パーティションキーは`key`という名前にする必要があります。

次に、LaravelアプリケーションがDynamoDBと通信できるように、AWS SDKをインストールします。

```shell
composer require aws/aws-sdk-php
```

加えて、DynamoDBキャッシュストアの設定オプションへ値を確実に指定してください。`AWS_ACCESS_KEY_ID`や`AWS_SECRET_ACCESS_KEY`などのオプションは、アプリケーションの`.env`設定ファイルで定義する必要があります。

```php
'dynamodb' => [
    'driver' => 'dynamodb',
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'table' => env('DYNAMODB_CACHE_TABLE', 'cache'),
    'endpoint' => env('DYNAMODB_ENDPOINT'),
],
```

<a name="cache-usage"></a>
## キャッシュ使用法

<a name="obtaining-a-cache-instance"></a>
### キャッシュインスタンスの取得

キャッシュ保存域インスタンスを取得するには、`Cache`ファサードを使用できます。これは、このドキュメント全体で使用します。`Cache`ファサードは、Laravelキャッシュ契約の基盤となる実装への便利で簡潔なアクセスを提供します。

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Support\Facades\Cache;

    class UserController extends Controller
    {
        /**
         * アプリケーションのすべてのユーザーのリストを表示
         */
        public function index(): array
        {
            $value = Cache::get('key');

            return [
                // …
            ];
        }
    }

<a name="accessing-multiple-cache-stores"></a>
#### 複数のキャッシュ保存域へのアクセス

`Cache`ファサードを使用すると、`store`メソッドを介してさまざまなキャッシュ保存域にアクセスできます。`store`メソッドに渡されるキーは、`cache`設定ファイルの`stores`設定配列にリストされている保存域の１つに対応している必要があります。

    $value = Cache::store('file')->get('foo');

    Cache::store('redis')->put('bar', 'baz', 600); // １０分

<a name="retrieving-items-from-the-cache"></a>
### キャッシュからのアイテム取得

`Cache`ファサードの`get`メソッドは、キャッシュからアイテムを取得するために使用します。アイテムがキャッシュに存在しない場合、`null`を返します。必要に応じて、アイテムが存在しない場合に返されるデフォルト値を指定する２番目の引数を`get`メソッドに渡すことができます。

    $value = Cache::get('key');

    $value = Cache::get('key', 'default');

デフォルト値としてクロージャを渡すこともできます。指定されたアイテムがキャッシュに存在しない場合、クロージャの結果が返されます。クロージャを渡すことで、データベースまたは他の外部サービスからのデフォルト値の取得を延期できるようになります。

    $value = Cache::get('key', function () {
        return DB::table(/* … */)->get();
    });

<a name="checking-for-item-existence"></a>
#### アイテムの存在を判定

`has`メソッドを使用して、アイテムがキャッシュに存在するかを判定できます。このメソッドは、アイテムが存在するがその値が`null`の場合にも、`false`を返します。

    if (Cache::has('key')) {
        // …
    }

<a name="incrementing-decrementing-values"></a>
#### 値の増減

`increment`メソッドと`decrement`メソッドを使用して、キャッシュ内の整数項目の値を増減できます。これらのメソッドは両方とも、アイテムの値をインクリメントまたはデクリメントする数を示すオプションの２番目の引数を取ります。

    // 存在しない場合、値を初期設定
    Cache::add('key', 0, now()->addHours(4));

    // 値の増分と減分
    Cache::increment('key');
    Cache::increment('key', $amount);
    Cache::decrement('key');
    Cache::decrement('key', $amount);

<a name="retrieve-store"></a>
#### 取得か保存

時に、キャッシュからアイテムを取得したいが、リクエストされたアイテムが存在しない場合はデフォルト値を保存したい場合があります。たとえば、すべてのユーザーをキャッシュから取得するか、存在しない場合はデータベースから取得してキャッシュに追加できます。これは、`Cache::remember`メソッドを使用して行えます。

    $value = Cache::remember('users', $seconds, function () {
        return DB::table('users')->get();
    });

アイテムがキャッシュに存在しない場合、`remember`メソッドに渡されたクロージャが実行され、その結果がキャッシュに配置されます。

`rememberForever`メソッドを使用して、キャッシュからアイテムを取得するか、アイテムが存在しない場合は永久に保存できます。

    $value = Cache::rememberForever('users', function () {
        return DB::table('users')->get();
    });

<a name="retrieve-delete"></a>
#### Retrieve and Delete

キャッシュからアイテムを取得してからアイテムを削除する必要がある場合は、`pull`メソッドを使用できます。`get`メソッドと同様に、アイテムがキャッシュに存在しない場合は`null`が返されます。

    $value = Cache::pull('key');

    $value = Cache::pull('key', 'default');

<a name="storing-items-in-the-cache"></a>
### キャッシュへのアイテム保存

`Cache`ファサードで`put`メソッドを使用して、アイテムをキャッシュに保存できます。

    Cache::put('key', 'value', $seconds = 10);

保存時間が`put`メソッドに渡されない場合、アイテムは無期限に保存されます。

    Cache::put('key', 'value');

秒数を整数として渡す代わりに、キャッシュするアイテムの有効期限を表す`DateTime`インスタンスを渡すこともできます。

    Cache::put('key', 'value', now()->addMinutes(10));

<a name="store-if-not-present"></a>
#### 存在しない場合は保存

`add`メソッドは、アイテムがキャッシュストアにまだ存在しない場合にのみ、アイテムをキャッシュに追加します。アイテムが実際にキャッシュに追加された場合、メソッドは`true`を返します。それ以外の場合にメソッドは`false`を返します。`add`メソッドはアトミック操作です。

    Cache::add('key', 'value', $seconds);

<a name="storing-items-forever"></a>
#### アイテムを永久に保存

`forever`メソッドを使用して、アイテムをキャッシュに永続的に保存できます。保存アイテムは期限切れにならないため、`forget`メソッドを使用して手作業でキャッシュから削除する必要があります。

    Cache::forever('key', 'value');

> [!NOTE]
> Memcachedドライバを使用している場合、「永久に」保存されているアイテムは、キャッシュがサイズ制限に達すると削除される可能性があります。

<a name="removing-items-from-the-cache"></a>
### キャッシュからのアイテム削除

`forget`メソッドを使用してキャッシュからアイテムを削除できます。

    Cache::forget('key');

有効期限の秒数をゼロまたは負にすることで、アイテムを削除することもできます。

    Cache::put('key', 'value', 0);

    Cache::put('key', 'value', -5);

`flush`メソッドを使用してキャッシュ全体をクリアできます。

    Cache::flush();

> [!WARNING]
> キャッシュのフラッシュは、設定したキャッシュの「プレフィックス」を尊重せず、キャッシュからすべてのエントリを削除します。他のアプリケーションと共有するキャッシュをクリアするときは、これを慎重に検討してください。

<a name="the-cache-helper"></a>
### キャッシュヘルパ

`Cache`ファサードの使用に加え、グローバルな`cache`関数を使用して、キャッシュによるデータの取得および保存もできます。`cache`関数が単一の文字列引数で呼び出されると、指定されたキーの値を返します。

    $value = cache('key');

キーと値のペアの配列と有効期限を関数に指定すると、指定された期間、値がキャッシュに保存されます。

    cache(['key' => 'value'], $seconds);

    cache(['key' => 'value'], now()->addMinutes(10));

`cache`関数を引数なしで呼び出すと、`Illuminate\Contracts\Cache\Factory`実装のインスタンスが返され、他のキャッシュメソッドを呼び出せます。

    cache()->remember('users', $seconds, function () {
        return DB::table('users')->get();
    });

> [!NOTE]
> グローバルな`cache`関数の呼び出しをテストするときは、[ファサードをテストする](/docs/{{version}}/mocking#mocking-facades)のように`Cache::shouldReceive`メソッドを使用できます。

<a name="atomic-locks"></a>
## アトミックロック

> [!WARNING]
> この機能を利用するには、アプリケーションのデフォルトのキャッシュドライバとして、`memcached`、`redis`、`dynamodb`、`database`、`file`、`array`キャッシュドライバを使用する必要があります。さらに、すべてのサーバが同じ中央キャッシュサーバと通信している必要があります。

<a name="managing-locks"></a>
### ロック管理

アトミックロックを使用すると、競合状態を気にすることなく分散ロックを操作できます。たとえば、[Laravel Forge](https://forge.laravel.com)は、アトミックロックを使用して、サーバ上で一度に1つのリモートタスクのみが実行されるようにしています。`Cache::lock`メソッドを使用してロックを作成および管理できます。

    use Illuminate\Support\Facades\Cache;

    $lock = Cache::lock('foo', 10);

    if ($lock->get()) {
        // ロックを10秒間取得

        $lock->release();
    }

`get`メソッドもクロージャを受け入れます。クロージャが実行された後、Laravelは自動的にロックを解除します。

    Cache::lock('foo', 10)->get(function () {
        // ロックを１０秒間取得し、自動的に開放
    });

リクエストした時点でロックが利用できない場合に、指定された秒数待つようにLaravelへ指示できます。指定された制限時間内にロックを取得できない場合、`Illuminate\Contracts\Cache\LockTimeoutException`を投げます。

    use Illuminate\Contracts\Cache\LockTimeoutException;

    $lock = Cache::lock('foo', 10);

    try {
        $lock->block(5);

        // ロック取得を最大５秒待つ
    } catch (LockTimeoutException $e) {
        // ロック取得失敗
    } finally {
        $lock->release();
    }

上記の例は、クロージャを`block`メソッドに渡すことで簡略化できます。クロージャがこのメソッドに渡されると、Laravelは指定された秒数の間ロックを取得しようとし、クロージャが実行されると自動的にロックを解放します。

    Cache::lock('foo', 10)->block(5, function () {
        // ロック取得を最大５秒待つ
    });

<a name="managing-locks-across-processes"></a>
### プロセス間でのロック管理

あるプロセスでロックを取得し、別のプロセスでそれを解放したい場合があります。たとえば、Webリクエスト中にロックを取得し、そのリクエストによってトリガーされたキュー投入済みジョブの終了時にロックを解放したい場合があるでしょう。このシナリオでは、ロックのスコープ付き「所​​有者トークン」をキュー投入済みジョブに渡し、ジョブが渡されたトークンを使用してロックを再インスタンス化できるようにする必要があります。

以下の例では、ロックが正常に取得された場合に、キュー投入済みジョブをディスパッチします。さらに、ロックの`owner`メソッドを介して、ロックの所有者トークンをキュー投入済みジョブに渡します。

    $podcast = Podcast::find($id);

    $lock = Cache::lock('processing', 120);

    if ($lock->get()) {
        ProcessPodcast::dispatch($podcast, $lock->owner());
    }

アプリケーションの`ProcessPodcast`ジョブ内で、所有者トークンを使用してロックを復元し、解放できます。

    Cache::restoreLock('processing', $this->owner)->release();

現在の所有者を尊重せずにロックを解放したい場合は、`forceRelease`メソッドを使用できます。

    Cache::lock('processing')->forceRelease();

<a name="adding-custom-cache-drivers"></a>
## カスタムキャッシュドライバの追加

<a name="writing-the-driver"></a>
### ドライバの作成

カスタムキャッシュドライバを作成するには、最初に`Illuminate\Contracts\Cache\Store`[契約](/docs/{{version}}/Contracts)を実装する必要があります。したがって、MongoDBキャッシュの実装は次のようになります。

    <?php

    namespace App\Extensions;

    use Illuminate\Contracts\Cache\Store;

    class MongoStore implements Store
    {
        public function get($key) {}
        public function many(array $keys) {}
        public function put($key, $value, $seconds) {}
        public function putMany(array $values, $seconds) {}
        public function increment($key, $value = 1) {}
        public function decrement($key, $value = 1) {}
        public function forever($key, $value) {}
        public function forget($key) {}
        public function flush() {}
        public function getPrefix() {}
    }

MongoDB接続を使用してこれらの各メソッドを実装する必要があります。これらの各メソッドを実装する方法の例については、[Laravelフレームワークのソースコード](https://github.com/laravel/framework)の`Illuminate\Cache\MemcachedStore`をご覧ください。実装が完了したら、`Cache`ファサードの`extend`メソッドを呼び出してカスタムドライバの登録を完了してください。

    Cache::extend('mongo', function (Application $app) {
        return Cache::repository(new MongoStore);
    });

> [!NOTE]
> カスタムキャッシュドライバコードをどこに置くか迷っている場合は、`app`ディレクトリ内に`Extensions`名前空間を作成できます。ただし、Laravelには厳密なアプリケーション構造がなく、好みに応じてアプリケーションを自由にオーガナイズできることに注意してください。

<a name="registering-the-driver"></a>
### Registering the Driver

カスタムキャッシュドライバをLaravelに登録するには、`Cache`ファサードで`extend`メソッドを使用します。他のサービスプロバイダは`boot`メソッド内でキャッシュされた値を読み取ろうとする可能性があるため、`booting`コールバック内にカスタムドライバを登録します。`booting`コールバックを使用することで、アプリケーションのサービスプロバイダで`boot`メソッドが呼び出される直前で、すべてのサービスプロバイダで`register`メソッドが呼び出された後にカスタムドライバが登録されるようにすることができます。アプリケーションの`App\Providers\AppServiceProvider`クラスの`register`メソッド内に`booting`コールバックを登録します。

    <?php

    namespace App\Providers;

    use App\Extensions\MongoStore;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\Facades\Cache;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * アプリケーションの全サービスの登録
         */
        public function register(): void
        {
            $this->app->booting(function () {
                 Cache::extend('mongo', function (Application $app) {
                     return Cache::repository(new MongoStore);
                 });
             });
        }

        /**
         * 全アプリケーションサービスの初期起動処理
         */
        public function boot(): void
        {
            // …
        }
    }

`extend`メソッドに渡す最初の引数はドライバの名前です。これは、`config/cache.php`設定ファイルの`driver`オプションに対応させます。２番目の引数は、`Illuminate\Cache\Repository`インスタンスを返す必要があるクロージャです。クロージャには、[サービスコンテナ](/docs/{{version}}/container)のインスタンスである`$app`インスタンスが渡されます。

拡張機能を登録したら、アプリケーションの`config/cache.php`設定ファイル内の`CACHE_STORE`環境変数、または`default`オプションを拡張機能の名前に更新します。

<a name="events"></a>
## イベント

すべてのキャッシュ操作でコードを実行できるように、キャッシュがディスパッチするさまざまな[イベント](/docs/{{version}}/events)をリッスンできます。

| イベント名 |
| --- |
| `Illuminate\Cache\Events\CacheHit` |
| `Illuminate\Cache\Events\CacheMissed` |
| `Illuminate\Cache\Events\KeyForgotten` |
| `Illuminate\Cache\Events\KeyWritten` |

パフォーマンスを向上させるために、`config/cache.php`設定ファイルの`events`設定オプションを`false`に設定し、イベントのキャッシュ保存を無効にできます。

```php
'database' => [
    'driver' => 'database',
    // ...
    'events' => false,
],
```
