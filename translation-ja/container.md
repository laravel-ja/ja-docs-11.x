# サービスコンテナ

- [イントロダクション](#introduction)
    - [設定なしの依存解決](#zero-configuration-resolution)
    - [いつコンテナを使用するか](#when-to-use-the-container)
- [結合](#binding)
    - [結合の基本](#binding-basics)
    - [インターフェイスと実装の結合](#binding-interfaces-to-implementations)
    - [コンテキストによる結合](#contextual-binding)
    - [コンテキスト属性](#contextual-attributes)
    - [プリミティブの結合](#binding-primitives)
    - [型指定した可変引数の結合](#binding-typed-variadics)
    - [タグ付け](#tagging)
    - [結合の拡張](#extending-bindings)
- [依存解決](#resolving)
    - [makeメソッド](#the-make-method)
    - [自動注入](#automatic-injection)
- [メソッドの起動と依存注入](#method-invocation-and-injection)
- [コンテナイベント](#container-events)
    - [再インデックス](#rebinding)
- [PSR-11](#psr-11)

<a name="introduction"></a>
## イントロダクション

Laravelサービスコンテナは、クラスの依存関係を管理し、依存注入を実行するための強力なツールです。依存の注入は、本質的にこれを意味する派手なフレーズです。クラスの依存は、コンストラクターまたは場合によっては「セッター」メソッドを介してクラスに「注入」されます。

簡単な例を見てみましょう。

    <?php

    namespace App\Http\Controllers;

    use App\Services\AppleMusic;
    use Illuminate\View\View;

    class PodcastController extends Controller
    {
        /**
         * 新しいコントローラインスタンスの生成
         */
        public function __construct(
            protected AppleMusic $apple,
        ) {}

        /**
         * 指定ユーザーのプロフィールを表示
         */
        public function show(string $id): View
        {
            return view('podcasts.show', [
                'podcast' => $this->apple->findPodcast($id)
            ]);
        }
    }

この例では、`PodcastController`はApple Musicなどのデータソースからポッドキャストを取得する必要があります。そこで、ポッドキャストを取得できるサービスを**依存注入**しています。サービスを依存注入することにより、アプリケーションをテストする際に`AppleMusic`サービスのダミー実装を簡単に作成できます。

Laravelサービスコンテナを深く理解することは、強力で大規模なアプリケーションを構築するため、およびLaravelコア自体に貢献するために不可欠です。

<a name="zero-configuration-resolution"></a>
### 設定なしの依存解決

クラスに依存関係がない場合、または他の具象クラス(インターフェイスではない)のみに依存している場合、そのクラスを依存解決する方法をコンテナへ指示する必要はありません。たとえば、以下のコードを`routes/web.php`ファイルに配置できます。

    <?php

    class Service
    {
        // ...
    }

    Route::get('/', function (Service $service) {
        die($service::class);
    });

この例でアプリケーションの`/`ルートを訪問すれば、自動的に`Service`クラスが依存解決され、ルートのハンドラに依存挿入されます。これは大転換です。これは、アプリケーションの開発において、肥大化する設定ファイルの心配をせず依存注入を利用できることを意味します。

幸いに、Laravelアプリケーションを構築するときに作成するクラスの多くは、[コントローラ](/docs/{{version}}/controllers)、[イベントリスナ](/docs/{{version}}/events)、[ミドルウェア](/docs/{{version}}/middleware)などの`handle`メソッドにより依存関係を注入ができます。設定なしの自動的な依存注入の力を味わったなら、これなしに開発することは不可能だと思うことでしょう。

<a name="when-to-use-the-container"></a>
### いつコンテナを使用するか

幸運にも、依存解決の設定がいらないため、ルート、コントローラ、イベントリスナ、その他どこでも、コンテナを手作業で操作しなくても、依存関係を頻繁にタイプヒントできます。たとえば、現在のリクエストに簡単にアクセスできるように、ルート定義で`Illuminate\Http\Request`オブジェクトをタイプヒントできます。このコードを書くため、コンテナを操作する必要はありません。コンテナはこうした依存関係の注入をバックグラウンドで管理しています。

    use Illuminate\Http\Request;

    Route::get('/', function (Request $request) {
        // ...
    });

多くの場合、自動依存注入と[ファサード](/docs/{{version}}/facades)のおかげで、コンテナから手作業で結合したり依存解決したりすることなく、Laravelアプリケーションを構築できます。**では、いつ手作業でコンテナを操作するのでしょう？**２つの状況を調べてみましょう。

第１に、インターフェイスを実装するクラスを作成し、そのインターフェイスをルートまたはクラスコンストラクターで型指定する場合は、[コンテナにそのインターフェイスを解決する方法を指示する](#binding-interfaces-to-implementations)必要があります。第２に、他のLaravel開発者と共有する予定の[Laravelパッケージの作成](/docs/{{version}}/packages)の場合、パッケージのサービスをコンテナに結合する必要がある場合があります。

<a name="binding"></a>
## 結合

<a name="binding-basics"></a>
### 結合の基本

<a name="simple-bindings"></a>
#### シンプルな結合

ほとんどすべてのサービスコンテナ結合は[サービスプロバイダ](/docs/{{version}}/provider)内で登録されるため、こうした例のほとんどは、この状況でのコンテナの使用法になります。

サービスプロバイダ内では、常に`$this->app`プロパティを介してコンテナにアクセスできます。`bind`メソッドを使用して結合を登録しできます。登録するクラスまたはインターフェイス名を、クラスのインスタンスを返すクロージャとともに渡します。

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->bind(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

リゾルバの引数としてコンテナ自体を受け取ることに注意してください。そのコンテナを使用して、構築中のオブジェクトの依存関係を解決できるのです。

前述のように、通常はサービスプロバイダ内のコンテナの中で操作します。ただし、サービスプロバイダの外部でコンテナとやり取りする場合は、`App`[ファサード](/docs/{{version}}/facades)を用いて操作します。

    use App\Services\Transistor;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\Facades\App;

    App::bind(Transistor::class, function (Application $app) {
        // ...
    });

指定するタイプに対する結合がまだ登録されていない場合にのみ、コンテナ結合を登録するには、`bindIf`メソッドを使用してください。

```php
$this->app->bindIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

> [!NOTE]
> クラスがどのインターフェイスにも依存しない場合、クラスをコンテナに結合する必要はありません。コンテナは、リフレクションを使用してこれらのオブジェクトを自動的に解決できるため、これらのオブジェクトの作成方法を指示する必要はありません。

<a name="binding-a-singleton"></a>
#### シングルトンの結合

`singleton`メソッドは、クラスまたはインターフェイスをコンテナに結合しますが、これは１回のみ依存解決される必要がある結合です。シングルトン結合が依存解決されたら、コンテナに対する後続の呼び出しで、同じオブジェクトインスタンスが返されます。

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->singleton(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

指定する型に対する結合がまだ登録されていない場合にのみ、シングルトンコンテナ結合を登録したい場合は`singletonIf`メソッドを使用します。

```php
$this->app->singletonIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

<a name="binding-scoped"></a>
#### スコープ付きシングルトンの結合

`scoped`メソッドは、Laravelのリクエストやジョブのライフサイクルの中で、一度だけ解決されるべきクラスやインターフェイスをコンテナへ結合します。このメソッドは`singleton`メソッドと似ていますが、`scoped`メソッドを使って登録したインスタンスは、Laravelアプリケーションが新しい「ライフサイクル」を開始するたびにフラッシュされます。例えば、[Laravel Octane](/docs/{{version}}/octane)ワーカが新しいリクエストを処理するときや、Laravel [キューワーカ](/docs/{{version}}/queues)が新しいジョブを処理するときなどです。

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->scoped(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

指定タイプの結合がまだ登録されていない場合のみ、スコープ付きコンテナ結合を登録するには、`scopedIf`メソッドを使用します。

    $this->app->scopedIf(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

<a name="binding-instances"></a>
#### インスタンスの結合

`instance`メソッドを使用して、既存のオブジェクトインスタンスをコンテナへ結合することもできます。指定したインスタンスは、コンテナに対する後続の呼び出しで常に返されます。

    use App\Services\Transistor;
    use App\Services\PodcastParser;

    $service = new Transistor(new PodcastParser);

    $this->app->instance(Transistor::class, $service);

<a name="binding-interfaces-to-implementations"></a>
### インターフェイスと実装の結合

サービスコンテナの非常に強力な機能は、インターフェイスを特定の実装に結合する機能です。たとえば、`EventPusher`インターフェイスと`RedisEventPusher`実装があると仮定しましょう。このインターフェイスの`RedisEventPusher`実装をコーディングしたら、次のようにサービスコンテナに登録できます。

    use App\Contracts\EventPusher;
    use App\Services\RedisEventPusher;

    $this->app->bind(EventPusher::class, RedisEventPusher::class);

この文は、クラスが`EventPusher`の実装を必要とするときに、`RedisEventPusher`を注入する必要があることをコンテナに伝えています。これで、コンテナにより依存解決されるクラスのコンストラクタで`EventPusher`インターフェイスをタイプヒントできます。Laravelアプリケーション内のコントローラ、イベントリスナ、ミドルウェア、およびその他のさまざまなタイプのクラスは、常にコンテナを使用して解決されることを忘れないでください。

    use App\Contracts\EventPusher;

    /**
     * 新しいクラスインスタンスの生成
     */
    public function __construct(
        protected EventPusher $pusher,
    ) {}

<a name="contextual-binding"></a>
### コンテキストによる結合

同じインターフェイスを利用する２つのクラスがある場合、各クラスに異なる実装を依存注入したい場合があります。たとえば、２つのコントローラは、`Illuminate\Contracts\Filesystem\Filesystem`[契約](/docs/{{version}}/Contracts)の異なる実装に依存する場合があります。Laravelは、この動作を定義するためのシンプルで流暢なインターフェイスを提供します。

    use App\Http\Controllers\PhotoController;
    use App\Http\Controllers\UploadController;
    use App\Http\Controllers\VideoController;
    use Illuminate\Contracts\Filesystem\Filesystem;
    use Illuminate\Support\Facades\Storage;

    $this->app->when(PhotoController::class)
              ->needs(Filesystem::class)
              ->give(function () {
                  return Storage::disk('local');
              });

    $this->app->when([VideoController::class, UploadController::class])
              ->needs(Filesystem::class)
              ->give(function () {
                  return Storage::disk('s3');
              });

<a name="contextual-attributes"></a>
### コンテキスト属性

コンテキストによる結合は、ドライバの実装や設定値のインジェクションに使用されることが多いため、Laravelでは、サービスプロバイダでコンテキストによる結合を手作業で定義しなくても、こうしたタイプの値を注入できるように、様々なコンテキストによる結合属性を提供しています。

例えば、`Storage`属性は特定の[ストレージディスク](/docs/{{version}}/filesystem)を注入するために使用できます。

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Container\Attributes\Storage;
use Illuminate\Contracts\Filesystem\Filesystem;

class PhotoController extends Controller
{
    public function __construct(
        #[Storage('local')] protected Filesystem $filesystem
    )
    {
        // ...
    }
}
```

`Storage`属性の他に、`Auth`、`Cache`、`Config`、`DB`、`Log`、`RouteParameter`、[`Tag`](#tagging)属性をLaravelは用意しています。

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Container\Attributes\Auth;
use Illuminate\Container\Attributes\Cache;
use Illuminate\Container\Attributes\Config;
use Illuminate\Container\Attributes\DB;
use Illuminate\Container\Attributes\Log;
use Illuminate\Container\Attributes\Tag;
use Illuminate\Contracts\Auth\Guard;
use Illuminate\Contracts\Cache\Repository;
use Illuminate\Contracts\Database\Connection;
use Psr\Log\LoggerInterface;

class PhotoController extends Controller
{
    public function __construct(
        #[Auth('web')] protected Guard $auth,
        #[Cache('redis')] protected Repository $cache,
        #[Config('app.timezone')] protected string $timezone,
        #[DB('mysql')] protected Connection $connection,
        #[Log('daily')] protected LoggerInterface $log,
        #[Tag('reports')] protected iterable $reports,
    )
    {
        // ...
    }
}
```

さらに、Laravelは現在の認証済みユーザーを指定ルートやクラスへ注入するために、`CurrentUser`属性を提供しています。

```php
use App\Models\User;
use Illuminate\Container\Attributes\CurrentUser;

Route::get('/user', function (#[CurrentUser] User $user) {
    return $user;
})->middleware('auth');
```

<a name="defining-custom-attributes"></a>
#### カスタム属性の定義

`Illuminate\Contracts\Container\ContextualAttribute`契約を実装すれば、独自のコンテキスト属性が作成できます。コンテナは属性の`resolve`メソッドを呼び出し、属性を利用するクラスへ注入する値を解決します。以下の例では、Laravelの組み込みの`Config`属性を再実装しています。

    <?php

    namespace App\Attributes;

    use Illuminate\Contracts\Container\ContextualAttribute;

    #[Attribute(Attribute::TARGET_PARAMETER)]
    class Config implements ContextualAttribute
    {
        /**
         * 新属性インスタンスの生成
         */
        public function __construct(public string $key, public mixed $default = null)
        {
        }

        /**
         * 設定値の解決
         *
         * @param  self  $attribute
         * @param  \Illuminate\Contracts\Container\Container  $container
         * @return mixed
         */
        public static function resolve(self $attribute, Container $container)
        {
            return $container->make('config')->get($attribute->key, $attribute->default);
        }
    }

<a name="binding-primitives"></a>
### プリミティブの結合

注入されたクラスを受け取るだけでなく、整数などのプリミティブ値も注入され、受け取るクラスがときにはあるでしょう。コンテキストによる結合を使用して、クラスへ必要な値を簡単に依存注入できます。

    use App\Http\Controllers\UserController;

    $this->app->when(UserController::class)
              ->needs('$variableName')
              ->give($value);

クラスが[タグ付き](#tagging)インスタンスの配列へ依存する場合があります。`giveTagged`メソッドを使用すると、そのタグを使用してすべてのコンテナバインディングを簡単に挿入できます。

    $this->app->when(ReportAggregator::class)
        ->needs('$reports')
        ->giveTagged('reports');

アプリケーションの設定ファイルの１つから値を注入する必要がある場合は、`giveConfig`メソッドを使用します。

    $this->app->when(ReportAggregator::class)
        ->needs('$timezone')
        ->giveConfig('app.timezone');

<a name="binding-typed-variadics"></a>
### 型指定した可変引数の結合

時折、可変コンストラクター引数を使用して型付きオブジェクトの配列を受け取るクラスが存在する場合があります。

    <?php

    use App\Models\Filter;
    use App\Services\Logger;

    class Firewall
    {
        /**
         * フィルタインスタンス
         *
         * @var array
         */
        protected $filters;

        /**
         * 新しいクラスインスタンスの生成
         */
        public function __construct(
            protected Logger $logger,
            Filter ...$filters,
        ) {
            $this->filters = $filters;
        }
    }

文脈による結合を使用すると、依存解決した`Filter`インスタンスの配列を返すクロージャを`give`メソッドへ渡すことで、この依存関係を解決できます。

    $this->app->when(Firewall::class)
              ->needs(Filter::class)
              ->give(function (Application $app) {
                    return [
                        $app->make(NullFilter::class),
                        $app->make(ProfanityFilter::class),
                        $app->make(TooLongFilter::class),
                    ];
              });

利便性のため、いつでも`Firewall`が`Filter`インスタンスを必要とするときは、コンテナが解決するクラス名の配列も渡せます。

    $this->app->when(Firewall::class)
              ->needs(Filter::class)
              ->give([
                  NullFilter::class,
                  ProfanityFilter::class,
                  TooLongFilter::class,
              ]);

<a name="variadic-tag-dependencies"></a>
#### 可変引数タグの依存

クラスには、特定のクラスとしてタイプヒントされた可変引数の依存関係を持つ場合があります(`Report ...$reports`)。`needs`メソッドと`giveTagged`メソッドを使用すると、特定の依存関係に対して、その[tag](#tagging)を使用してすべてのコンテナ結合を簡単に挿入できます。

    $this->app->when(ReportAggregator::class)
        ->needs(Report::class)
        ->giveTagged('reports');

<a name="tagging"></a>
### タグ付け

場合により、特定の結合「カテゴリ」をすべて依存解決する必要が起きます。たとえば、さまざまな`Report`インターフェイス実装の配列を受け取るレポートアナライザを構築しているとしましょう。`Report`実装を登録した後、`tag`メソッドを使用してそれらにタグを割り当てられます。

    $this->app->bind(CpuReport::class, function () {
        // ...
    });

    $this->app->bind(MemoryReport::class, function () {
        // ...
    });

    $this->app->tag([CpuReport::class, MemoryReport::class], 'reports');

サービスにタグ付けしたら、コンテナの`tagged`メソッドを使用して簡単にすべてを依存解決できます。

    $this->app->bind(ReportAnalyzer::class, function (Application $app) {
        return new ReportAnalyzer($app->tagged('reports'));
    });

<a name="extending-bindings"></a>
### 結合の拡張

`extend`メソッドを使用すると、依存解決済みのサービスを変更できます。たとえば、サービスを依存解決した後、追加のコードを実行してサービスをデコレートまたは設定できます。`extend`メソッドは拡張するサービスクラスと、変更したサービスを返すクロージャの２引数を取ります。クロージャは解決するサービスとコンテナインスタンスを引数に取ります。

    $this->app->extend(Service::class, function (Service $service, Application $app) {
        return new DecoratedService($service);
    });

<a name="resolving"></a>
## 依存解決

<a name="the-make-method"></a>
### `make`メソッド

`make`メソッドを使用して、コンテナからクラスインスタンスを解決します。`make`メソッドは、解決したいクラスまたはインターフェイスの名前を受け入れます。

    use App\Services\Transistor;

    $transistor = $this->app->make(Transistor::class);

クラスの依存関係の一部がコンテナを介して解決できない場合は、それらを連想配列として`makeWith`メソッドに渡すことでそれらを依存注入できます。たとえば、`Transistor`サービスに必要な`$id`コンストラクタ引数を手作業で渡すことができます。

    use App\Services\Transistor;

    $transistor = $this->app->makeWith(Transistor::class, ['id' => 1]);

`bound`メソッドは、クラスやインターフェイスをコンテナ内で明示的に結合しているかを判定するために使用します。

    if ($this->app->bound(Transistor::class)) {
        // ...
    }

サービスプロバイダの外部で、`$app`変数にアクセスできないコードの場所では、`App`[ファサード](/docs/{{version}}/facades)、`app`[ヘルパ](/docs/{{version}}/helpers#method-app)を使用してコンテナからクラスインスタンスを依存解決します。

    use App\Services\Transistor;
    use Illuminate\Support\Facades\App;

    $transistor = App::make(Transistor::class);

    $transistor = app(Transistor::class);

Laravelコンテナインスタンス自体をコンテナにより解決中のクラスへ依存注入したい場合は、クラスのコンストラクタで`Illuminate\Container\Container`クラスを入力してください。

    use Illuminate\Container\Container;

    /**
     * 新しいクラスインスタンスの生成
     */
    public function __construct(
        protected Container $container,
    ) {}

<a name="automatic-injection"></a>
### 自動注入

あるいは、そして重要なことに、[コントローラ](/docs/{{version}}/controllers)、[イベントリスナ](/docs/{{version}}/events)、[ミドルウェア](/docs/{{version}}/middleware)など、コンテナにより解決されるクラスのコンストラクターでは、依存関係をタイプヒントすることができます。さらに、[キュー投入するジョブ](/docs/{{version}}/queues)の`handle`メソッドでも、依存関係をタイプヒントできます。実践的に、ほとんどのオブジェクトはコンテナにより解決されるべきでしょう。

たとえば、コントローラのコンストラクタでアプリケーションが定義したサービスをタイプヒントすることができます。そのサービスを自動的に解決し、クラスに依存注入します。

    <?php

    namespace App\Http\Controllers;

    use App\Services\AppleMusic;

    class PodcastController extends Controller
    {
        /**
         * 新しいコントローラインスタンスの生成
         */
        public function __construct(
            protected AppleMusic $apple,
        ) {}

        /**
         * 指定ポッドキャストの情報を表示
         */
        public function show(string $id): Podcast
        {
            return $this->apple->findPodcast($id);
        }
    }

<a name="method-invocation-and-injection"></a>
## メソッドの起動と依存注入

コンテナがあるメソッドの依存関係を自動的に注入できるようにしたまま、そのオブジェクトインスタンス上のメソッドを起動したい場合があります。たとえば、以下のクラスを想定してください。

    <?php

    namespace App;

    use App\Services\AppleMusic;

    class PodcastStats
    {
        /**
         * 新しいポッドキャストの状況レポートを生成
         */
        public function generate(AppleMusic $apple): array
        {
            return [
                // ...
            ];
        }
    }

次のように、コンテナを介して`generate`メソッドを呼び出せます。

    use App\PodcastStats;
    use Illuminate\Support\Facades\App;

    $stats = App::call([new PodcastStats, 'generate']);

`call`メソッドは任意のPHP callableを受け入れます。コンテナの`call`メソッドは、依存関係を自動的に注入しながら、クロージャを呼び出すためにも使用できます。

    use App\Services\AppleMusic;
    use Illuminate\Support\Facades\App;

    $result = App::call(function (AppleMusic $apple) {
        // ...
    });

<a name="container-events"></a>
## コンテナイベント

サービスコンテナは、オブジェクトを依存解決するたびにイベントを発生させます。`resolving`メソッドを使用してこのイベントをリッスンできます。

    use App\Services\Transistor;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->resolving(Transistor::class, function (Transistor $transistor, Application $app) {
        // コンテナがTransistorタイプのオブジェクトを解決するときに呼び出される
    });

    $this->app->resolving(function (mixed $object, Application $app) {
        // コンテナが任意のタイプのオブジェクトを解決するときに呼び出される
    });

ご覧のとおり、解決しているオブジェクトがコールバックに渡され、利用側に渡される前にオブジェクトへ追加のプロパティを設定できます。

<a name="rebinding"></a>
### 再インデックス

`rebinding`メソッドを使用すると、サービスがコンテナに再結合されたとき、つまり最初の結合の後に再度登録されたりオーバーライドされたりしたときをリッスンできます。これは、特定の結合が更新されるたびに依存関係を更新したり、動作を変更したりする必要がある場合に便利です。

    use App\Contracts\PodcastPublisher;
    use App\Services\SpotifyPublisher;
    use App\Services\TransistorPublisher;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->bind(PodcastPublisher::class, SpotifyPublisher::class);

    $this->app->rebinding(
        PodcastPublisher::class,
        function (Application $app, PodcastPublisher $newInstance) {
            //
        },
    );

    // この新しい結合は、再インデックスクロージャを起動する
    $this->app->bind(PodcastPublisher::class, TransistorPublisher::class);

<a name="psr-11"></a>
## PSR-11

Laravelのサービスコンテナは、[PSR-11](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-11-container.md)インターフェイスを実装しています。したがって、PSR-11コンテナインターフェイスをタイプヒントして、Laravelコンテナのインスタンスを取得できます。

    use App\Services\Transistor;
    use Psr\Container\ContainerInterface;

    Route::get('/', function (ContainerInterface $container) {
        $service = $container->get(Transistor::class);

        // ...
    });

指定した識別子を解決できない場合は例外を投げます。識別子が結合されなかった場合、例外は`Psr\Container\NotFoundExceptionInterface`のインスタンスです。識別子が結合されているが依存解決できなかった場合、`Psr\Container\ContainerExceptionInterface`のインスタンスを投げます。
