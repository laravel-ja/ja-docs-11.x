# ルーティング

- [基本的なルーティング](#basic-routing)
    - [デフォルトルートファイル](#the-default-route-files)
    - [ルートのリダイレクト](#redirect-routes)
    - [ビュールート](#view-routes)
    - [ルート一覧](#listing-your-routes)
    - [ルートのカスタマイズ](#routing-customization)
- [ルートパラメータ](#route-parameters)
    - [必須パラメータ](#required-parameters)
    - [オプションパラメータ](#parameters-optional-parameters)
    - [正規表現の制約](#parameters-regular-expression-constraints)
- [名前付きルート](#named-routes)
- [ルートグループ](#route-groups)
    - [ミドルウェア](#route-group-middleware)
    - [コントローラ](#route-group-controllers)
    - [サブドメインルーティング](#route-group-subdomain-routing)
    - [ルートプレフィックス](#route-group-prefixes)
    - [ルート名のプレフィックス](#route-group-name-prefixes)
- [ルートモデル結合](#route-model-binding)
    - [暗黙の結合](#implicit-binding)
    - [暗黙のEnumバインディング](#implicit-enum-binding)
    - [明示的な結合](#explicit-binding)
- [フォールバックルート](#fallback-routes)
- [レート制限](#rate-limiting)
    - [レート制限の定義](#defining-rate-limiters)
    - [ルートへのレート制限指定](#attaching-rate-limiters-to-routes)
- [疑似フォームメソッド](#form-method-spoofing)
- [現在のルートへのアクセス](#accessing-the-current-route)
- [オリジン間リソース共有 (CORS)](#cors)
- [ルートのキャッシュ](#route-caching)

<a name="basic-routing"></a>
## 基本的なルーティング

もっとも基本的なLaravelルートはURIとクロージャを引数に取り、複雑なルーティング設定ファイルなしでもルートと動作を定義できる、非常にシンプルで表現力豊かなメソッドを提供しています。

    use Illuminate\Support\Facades\Route;

    Route::get('/greeting', function () {
        return 'Hello World';
    });

<a name="the-default-route-files"></a>
### デフォルトルートファイル

すべてのLaravelルートは、`routes`ディレクトリにあるルートファイルで定義されます。これらのファイルは、アプリケーションの`bootstrap/app.php`ファイルで指定されている設定を使い、Laravelが自動的にロードします。`routes/web.php`ファイルはウェブインターフェイス用のルートを定義します。これらのルートは、セッション状態やCSRF保護などの機能を提供している`web`[ミドルウェアグループ](/docs/{{version}}/middleware#laravels-default-middleware-groups)を指定してあります。

ほとんどのアプリケーションでは、`routes/web.php`ファイルでルートを定義することから始めます。`routes/web.php`で定義したルートは、ブラウザでその定義したルートのURLを入力することでアクセスできます。たとえば、ブラウザで`http：//example.com/user`に移動すると、次のルートにアクセスできます。

    use App\Http\Controllers\UserController;

    Route::get('/user', [UserController::class, 'index']);

<a name="api-routes"></a>
#### APIルート

あなたのアプリケーションが、ステートレスAPIも提供する場合、`install:api` Artisanコマンドを使用してAPIルーティングを有効にしてください。

```shell
php artisan install:api
```

`install:api`コマンドは、[Laravel Sanctum](/docs/{{version}}/sanctum)をインストールします。これは、サードパーティのAPIコンシューマ、SPA、またはモバイルアプリケーションの認証に使用できる、堅牢でありながらシンプルなAPIトークン認証ガードを提供します。さらに、`install:api`コマンドは`routes/api.php`ファイルを作成します。

    Route::get('/user', function (Request $request) {
        return $request->user();
    })->middleware('auth:sanctum');

`routes/api.php`のルートはステートレスで、`api`[ミドルウェアグループ](/docs/{{version}}/middleware#laravels-default-middleware-groups)が指定されます。さらに、`/api` URIプレフィックスがこれらのルートに自動的に適用されるため、ファイル内のすべてのルートへ手作業で適用する必要はありません。アプリケーションの`bootstrap/app.php`ファイルを修正して、このプレフィックスを変更できます。

    ->withRouting(
        api: __DIR__.'/../routes/api.php',
        apiPrefix: 'api/admin',
        // ...
    )

<a name="available-router-methods"></a>
#### 利用可能なルーターメソッド

ルーターを使用すると、全HTTP動詞に応答するルートを登録できます。

    Route::get($uri, $callback);
    Route::post($uri, $callback);
    Route::put($uri, $callback);
    Route::patch($uri, $callback);
    Route::delete($uri, $callback);
    Route::options($uri, $callback);

複数のHTTP動詞に応答するルートを登録する必要が起きる場合もあります。これは`match`メソッドを使用して行います。または、`any`メソッドを使用して、すべてのHTTP動詞に応答するルートを登録することもできます。

    Route::match(['get', 'post'], '/', function () {
        // ...
    });

    Route::any('/', function () {
        // ...
    });

> [!NOTE]
> 同じURIの複数のルートを定義する場合、`get`、`post`、`put`、`patch`、`delete`、`options`の各メソッドを使ったルートは、`any`、`match`、`redirect`の各メソッドを使ったルートよりも先に定義する必要があります。これにより、受信リクエストが正しいルートにマッチします。

<a name="dependency-injection"></a>
#### 依存注入

ルートのコールバックの引数にタイプヒントにより、そのルートで必要な依存関係を指定できます。宣言した依存関係は、Laravel[サービスコンテナ](/docs/{{version}}/container)により、自動的に解決されコールバックへ注入されます。たとえば、`Illuminate\Http\Request`クラスのタイプヒントを指定して、現在のHTTPリクエストをルートコールバックへ自動的に注入できます。

    use Illuminate\Http\Request;

    Route::get('/users', function (Request $request) {
        // ...
    });

<a name="csrf-protection"></a>
#### CSRF保護

`web`ルートファイルで定義した`POST`、`PUT`、`PATCH`、`DELETE`ルートへ送るHTMLフォームには、CSRFトークンフィールドを含める必要があることに注意してください。含めていない場合、リクエストは拒否されます。CSRF保護の詳細については、[CSRFドキュメント](/docs/{{version}}/csrf)をご覧ください。

    <form method="POST" action="/profile">
        @csrf
        ...
    </form>

<a name="redirect-routes"></a>
### ルートのリダイレクト

別のURIにリダイレクトするルートを定義する場合は、`Route::redirect`メソッドを使用します。このメソッドは便利なショートカットを提供しているため、単純なリダイレクトを実行するために完全なルートまたはコントローラを定義する必要はありません。

    Route::redirect('/here', '/there');

デフォルトで`Route::redirect`は`302`ステータスコードを返します。オプションの３番目のパラメータによりステータスコードをカスタマイズできます。

    Route::redirect('/here', '/there', 301);

もしくは、`Route::permanentRedirect`メソッドを使用して`301`ステータスコードを返すことも可能です。

    Route::permanentRedirect('/here', '/there');

> [!WARNING]
> リダイレクトルートでルートパラメータを使用する場合、以降のパラメータはLaravelによって予約されており、使用できません。：`destination`、`status`

<a name="view-routes"></a>
### ビュールート

ルートが[ビュー](/docs/{{version}}/views)のみを返す場合は、`Route::view`メソッドを使用します。`redirect`メソッドと同様に、このメソッドは単純なショートカットを提供しているため、完全なルートやコントローラを定義する必要はありません。`view`メソッドは最初の引数にURI、２番目にビュー名を取ります。さらに、オプションとして３番目の引数にビューへ渡すデータの配列を指定できます。

    Route::view('/welcome', 'welcome');

    Route::view('/welcome', 'welcome', ['name' => 'Taylor']);

> [!WARNING]
> ビュールートでルートパラメータを使用する場合、次のパラメータはLaravelによって予約されており、使用できません。：`view`、`data`、`status`、`headers`

<a name="listing-your-routes"></a>
### ルート一覧

`route:list` Artisanコマンドは、アプリケーションで定義しているすべてのルートの概要を簡単に提供してくれます。

```shell
php artisan route:list
```

デフォルトでは、各ルートへ割り付けたルートミドルウェアは、`route:list`出力に表示されませんが、コマンドに`-v`オプションを追加すると、ルートのミドルウェアとミドルウェアグループを表示できます。

```shell
php artisan route:list -v

# ミドルウェアグループも表示
php artisan route:list -vv
```

また、指定したURIで始まるルートのみを表示するように、Laravelへ指示することもできます。

```shell
php artisan route:list --path=api
```

さらに、`route:list`コマンドを実行するときに、`--except-vendor`オプションを指定すると、サードパーティーのパッケージが定義したルートを非表示にできます。

```shell
php artisan route:list --except-vendor
```

同様に、`route:list`コマンドを実行するときに、`--only-vendor`オプションを指定し、サードパーティーのパッケージが定義したルートだけを表示できます。

```shell
php artisan route:list --only-vendor
```

<a name="routing-customization"></a>
### ルートのカスタマイズ

デフォルトで、アプリケーションのルートは`bootstrap/app.php`ファイルで設定、ロードします。

```php
<?php

use Illuminate\Foundation\Application;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )->create();
```

しかし、アプリケーションのルートのサブセットを含む、全く新しいファイルを定義したいこともあるでしょう。これをおこなうには、`withRouting`メソッドで`then`クロージャを指定してください。このクロージャの中で、アプリケーションで必要な追加のルートを登録できます。

```php
use Illuminate\Support\Facades\Route;

->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
    then: function () {
        Route::middleware('api')
            ->prefix('webhooks')
            ->name('webhooks.')
            ->group(base_path('routes/webhooks.php'));
    },
)
```

あるいは、`withRouting`メソッドへ`using`クロージャを指定し、ルート登録を完全に制御することもできます。この引数を渡すと、フレームワークではHTTPルートを登録しないため、すべてのルートを手作業で登録してください。

```php
use Illuminate\Support\Facades\Route;

->withRouting(
    commands: __DIR__.'/../routes/console.php',
    using: function () {
        Route::middleware('api')
            ->prefix('api')
            ->group(base_path('routes/api.php'));

        Route::middleware('web')
            ->group(base_path('routes/web.php'));
    },
)
```

<a name="route-parameters"></a>
## ルートパラメータ

<a name="required-parameters"></a>
### 必須パラメータ

ルート内のURIのセグメントを取得したい場合があるでしょう。たとえば、URLからユーザーのIDを取得する必要のある場合があります。これを行うにはルートパラメータを定義します。

    Route::get('/user/{id}', function (string $id) {
        return 'User '.$id;
    });

ルートに必要なだけの数のルートパラメータを定義できます。

    Route::get('/posts/{post}/comments/{comment}', function (string $postId, string $commentId) {
        // ...
    });

ルートパラメータは常に`{}`中括弧で囲こみ、アルファベットで構成する必要があります。ルートパラメータ名にはアンダースコア(`_`)も使用できます。ルートパラメータは順序に基づいて、ルートのコールバック／コントローラに注入されます。ルートのコールバック／コントローラの引数名は考慮されません。

<a name="parameters-and-dependency-injection"></a>
#### パラメータと依存注入

Laravelサービスコンテナにより、ルートのコールバックへ自動的に注入してもらいたい依存がある場合は、依存のタイプヒントの後にルートパラメータをリストする必要があります。

    use Illuminate\Http\Request;

    Route::get('/user/{id}', function (Request $request, string $id) {
        return 'User '.$id;
    });

<a name="parameters-optional-parameters"></a>
### オプションパラメータ

場合により、いつもURIに存在するとは限らないルートパラメータを指定する必要が起きます。パラメータ名の後に`？`マークを付けるとオプション指定になります。ルートの対応する変数にデフォルト値も指定してください。

    Route::get('/user/{name?}', function (?string $name = null) {
        return $name;
    });

    Route::get('/user/{name?}', function (?string $name = 'John') {
        return $name;
    });

<a name="parameters-regular-expression-constraints"></a>
### 正規表現の制約

ルートインスタンスの`where`メソッドを使用して、ルートパラメータのフォーマットを制約できます。`where`メソッドは、パラメーターの名前と、パラメーターの制約方法を定義する正規表現を引数に取ります。

    Route::get('/user/{name}', function (string $name) {
        // ...
    })->where('name', '[A-Za-z]+');

    Route::get('/user/{id}', function (string $id) {
        // ...
    })->where('id', '[0-9]+');

    Route::get('/user/{id}/{name}', function (string $id, string $name) {
        // ...
    })->where(['id' => '[0-9]+', 'name' => '[a-z]+']);

使いやすいように、一般的に使用される正規表現パターンには、ルートにパターン制約をすばやく追加できるようにヘルパメソッドを用意しています。

    Route::get('/user/{id}/{name}', function (string $id, string $name) {
        // ...
    })->whereNumber('id')->whereAlpha('name');

    Route::get('/user/{name}', function (string $name) {
        // ...
    })->whereAlphaNumeric('name');

    Route::get('/user/{id}', function (string $id) {
        // ...
    })->whereUuid('id');

    Route::get('/user/{id}', function (string $id) {
        // ...
    })->whereUlid('id');

    Route::get('/category/{category}', function (string $category) {
        // ...
    })->whereIn('category', ['movie', 'song', 'painting']);

    Route::get('/category/{category}', function (string $category) {
        // ...
    })->whereIn('category', CategoryEnum::cases());

受信リクエストがルートパターンの制約と一致しない場合、404 HTTPレスポンスを返します。

<a name="parameters-global-constraints"></a>
#### グローバル制約

ルートパラメータを指定した正規表現で常に制約したい場合は、`pattern`メソッドを使用します。これらのパターンは、アプリケーションの`App\Providers\AppServiceProvider`クラスの`boot`メソッドで定義する必要があります。

    use Illuminate\Support\Facades\Route;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Route::pattern('id', '[0-9]+');
    }

パターンを定義すれば、パラメータ名によりすべてのルートで自動的に適用されます。

    Route::get('/user/{id}', function (string $id) {
        // {id}が数値の場合にのみ実行される
    });

<a name="parameters-encoded-forward-slashes"></a>
#### エンコードされたスラッシュ

Laravelルーティングコンポーネントでは、`/`を除くすべての文字をルートパラメータ値内に含めることができます。`/`がプレースホルダーの一部になることを明示的に許可する場合は、`where`条件の正規表現を使用する必要があります。

    Route::get('/search/{search}', function (string $search) {
        return $search;
    })->where('search', '.*');

> [!WARNING]
> エンコードされたスラッシュは、最後のルートセグメント内でのみサポートされます。

<a name="named-routes"></a>
## 名前付きルート

名前付きルートを使用すると、特定のルートのURLまたはリダイレクトが簡単に生成できます。`name`メソッドをルート定義にチェーンすることにより、ルートの名前を指定できます。

    Route::get('/user/profile', function () {
        // ...
    })->name('profile');

コントローラアクションのルート名を指定することもできます。

    Route::get(
        '/user/profile',
        [UserProfileController::class, 'show']
    )->name('profile');

> [!WARNING]
> ルート名は常に一意である必要があります。

<a name="generating-urls-to-named-routes"></a>
#### 名前付きルートのURL生成

特定のルートに名前を割り当てたら、Laravelの`route`および`redirect`ヘルパ関数を使い、URLやリダイレクトを生成するときにルートの名前を使用できます。

    // URLを生成
    $url = route('profile');

    // リダイレクトの生成
    return redirect()->route('profile');

    return to_route('profile');

名前付きルートがパラメータを定義している場合は、パラメータを２番目の引数として`route`関数に渡してください。指定したパラメータは、生成するURLの正しい位置に自動的に挿入されます。

    Route::get('/user/{id}/profile', function (string $id) {
        // ...
    })->name('profile');

    $url = route('profile', ['id' => 1]);

配列に追加のパラメーターを渡すと、それらのキー／値ペアが生成するURLのクエリ文字列へ自動的に追加されます。

    Route::get('/user/{id}/profile', function (string $id) {
        // ...
    })->name('profile');

    $url = route('profile', ['id' => 1, 'photos' => 'yes']);

    // /user/1/profile?photos=yes

> [!NOTE]
> 場合により現在のロケールなど、URLパラメータにリクエスト全体のデフォルト値を指定したいことがあります。これを実現するには、[`URL::defaults`メソッド](/docs/{{version}}/urls#default-values)を使用してください。

<a name="inspecting-the-current-route"></a>
#### 現在のルートの検査

現在のリクエストが特定の名前付きルートにルーティングされたかどうかを確認したい場合は、Routeインスタンスで`named`メソッドを使用できます。たとえば、ルートミドルウェアから現在のルート名を確認できます。

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    /**
     * 受信リクエストの処理
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        if ($request->route()->named('profile')) {
            // ...
        }

        return $next($request);
    }

<a name="route-groups"></a>
## ルートグループ

ルートグループを使用すると、ミドルウェアなどのルート属性を個々のルートごとに定義することなく、多数のルート間で共有できます。

ネストされたグループは、属性を親グループとインテリジェントに「マージ」しようとします。ミドルウェアと`where`条件は、指定する名前やプレフィックスと同時にマージされます。URIプレフィックスの名前空間区切り文字とスラッシュは、必要に応じて自動的に追加されます。

<a name="route-group-middleware"></a>
### ミドルウェア

[ミドルウェア](/docs/{{version}}/middleware)をグループ内すべてのルートに割り当てるには、グループを定義する前に`middleware`メソッドを使用します。ミドルウェアは、配列にリストする順序で実行します。

    Route::middleware(['first', 'second'])->group(function () {
        Route::get('/', function () {
            // １番目と２番目のミドルウェアを使用
        });

        Route::get('/user/profile', function () {
            // １番目と２番目のミドルウェアを使用
        });
    });

<a name="route-group-controllers"></a>
### コントローラ

ルートのグループがすべて同じ[コントローラ](/docs/{{version}}/controllers)を利用する場合、`controller`メソッドを使用して、グループ内のすべてのルートに共通のコントローラを定義できます。これにより、ルートを定義時に、そのルートが呼び出すコントローラメソッドを指定するだけでよくなります。

    use App\Http\Controllers\OrderController;

    Route::controller(OrderController::class)->group(function () {
        Route::get('/orders/{id}', 'show');
        Route::post('/orders', 'store');
    });

<a name="route-group-subdomain-routing"></a>
### サブドメインルーティング

ルートグループは、サブドメインルーティングを処理するためにも使用できます。サブドメインには、ルートURIと同じようにルートパラメータを割り当てることができ、ルートまたはコントローラで使用するためにサブドメインの一部を取得できます。サブドメインは、グループを定義する前に`domain`メソッドを呼び出し指定します。

    Route::domain('{account}.example.com')->group(function () {
        Route::get('/user/{id}', function (string $account, string $id) {
            // ...
        });
    });

> [!WARNING]
> ルーティングがサブドメインルートに到達できるようにするには、サブドメインなしのドメインルートを登録する前にサブドメインルートを登録しておく必要があります。これにより、サブドメインなしのドメインルートが同じURIパスを持つサブドメインルートを上書きするのを防げます。

<a name="route-group-prefixes"></a>
### ルートプレフィックス

`prefix`メソッドを使用して、グループ内の各ルートに特定のURIをプレフィックスとして付けることができます。たとえば、グループ内のすべてのルートURIの前に`admin`を付けることができます。

    Route::prefix('admin')->group(function () {
        Route::get('/users', function () {
            // /admin/usersのURLに一致
        });
    });

<a name="route-group-name-prefixes"></a>
### ルート名のプレフィックス

`name`メソッドを使用して、グループ内の各ルート名の前に特定の文字列を付け加えられます。例として、グループ内のすべてのルートの名前の前に`admin`を付けてみましょう。指定する文字列は、指定したとおりにルート名のプレフィックスとして付けられるため、必ず末尾の`.`文字を指定してください。

    Route::name('admin.')->group(function () {
        Route::get('/users', function () {
            // ルートに"admin.users"が名付けられる
        })->name('users');
    });

<a name="route-model-binding"></a>
## ルートモデル結合

ルートまたはコントローラアクションでモデルIDを取得する場合、多くはデータベースにクエリを実行して、そのIDに対応するモデルを取得することになります。Laravelルートモデル結合は、モデルインスタンスをルートに直接自動的に注入する利便性を提供しています。たとえば、ユーザーのIDを挿入する代わりに、指定されたIDに一致する`User`モデルインスタンス全体を注入できます。

<a name="implicit-binding"></a>
### 暗黙の結合

Laravelは、タイプヒントの変数名がルートセグメント名と一致する、ルートまたはコントローラアクションで定義したEloquentモデルを自動的に解決します。例をご覧ください。

    use App\Models\User;

    Route::get('/users/{user}', function (User $user) {
        return $user->email;
    });

`$user`変数は`App\Models\User`Eloquentモデルとしてタイプヒントされ、変数名は`{user}`URIセグメントと一致するため、LaravelはリクエストURIの対応する値と一致するIDを持つモデルインスタンスを自動的に挿入します。一致するモデルインスタンスがデータベースに見つからない場合、404 HTTPレスポンスを自動的に生成します。

もちろん、コントローラメソッドを使用する場合でも暗黙的な結合は可能です。繰り返しになりますが、`{user}`URIセグメントは`App\Models\User`タイプヒントを含むコントローラの`$user`変数と一致することに注意してください。

    use App\Http\Controllers\UserController;
    use App\Models\User;

    // ルート定義
    Route::get('/users/{user}', [UserController::class, 'show']);

    // コントローラメソッドの定義
    public function show(User $user)
    {
        return view('user.profile', ['user' => $user]);
    }

<a name="implicit-soft-deleted-models"></a>
#### ソフトデリートモデルバインディング

通常、暗黙のモデル結合は、[ソフトデリート](/docs/{{version}}/eloquent#soft-deleting)されたモデルを取得しません。しかし、ルートの定義で`withTrashed`メソッドをチェーンすることにより、こうしたモデルを取得するように暗黙のバインディングで指示できます。

    use App\Models\User;

    Route::get('/users/{user}', function (User $user) {
        return $user->email;
    })->withTrashed();

<a name="customizing-the-default-key-name"></a>
#### キーのカスタマイズ

`id`以外の列を使用してEloquentモデルを解決したい場合もあるでしょう。その場合は、ルートパラメータ定義でカラムを指定します。

    use App\Models\Post;

    Route::get('/posts/{post:slug}', function (Post $post) {
        return $post;
    });

特定のモデルクラスを取得するときのモデル結合で、常に`id`以外のデータベースカラムを使用する場合は、Eloquentモデルの`getRouteKeyName`メソッドをオーバーライドできます。

    /**
     * モデルのルートキーの取得
     */
    public function getRouteKeyName(): string
    {
        return 'slug';
    }

<a name="implicit-model-binding-scoping"></a>
#### カスタムキーとスコープ

単一のルート定義で複数のEloquentモデルを暗黙的にバインドする場合、前のEloquentモデルの子である必要があるように、２番目のEloquentモデルのスコープの設定をおすすめします。たとえば、特定のユーザーのブログ投稿をスラッグで取得する次のルート定義について考えてみます。

    use App\Models\Post;
    use App\Models\User;

    Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
        return $post;
    });

ネストしたルートパラメーターとしてカスタムキー付き暗黙的結合を使用する場合、Laravelはクエリのスコープを自動的に設定し、親の関係名を推測する規則を使用して、親によってネストされたモデルを取得します。この場合、`User`モデルには、`Post`モデルを取得するために使用できる`posts`(ルートパラメータ名の複数形)という名前のリレーションが存在していると仮定します。

必要であれば、カスタムキーを指定しない場合でも、Laravelへ「子」のバインドをスコープするように指示できます。それにはルート定義時に、`scopeBindings`メソッドを呼び出してください。

    use App\Models\Post;
    use App\Models\User;

    Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
        return $post;
    })->scopeBindings();

もしくは、グループのルート定義全体へスコープ付きバインドを使用するよう指示することもできます。

    Route::scopeBindings()->group(function () {
        Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
            return $post;
        });
    });

同様に、`withoutScopedBindings`メソッドを呼び出すことで、バインドをスコープしないよう明示的にLaravelへ指示できます。

    Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
        return $post;
    })->withoutScopedBindings();

<a name="customizing-missing-model-behavior"></a>
#### 見つからないモデルの動作をカスタマイズする

暗黙的にバインドされたモデルが見つからない場合、通常404のHTTPレスポンスが生成されます。ただし、ルートを定義するときに`Missing`メソッドを呼び出し、この動作をカスタマイズできます。`Missing`メソッドは、暗黙的にバインドされたモデルが見つからない場合に呼び出されるクロージャを引数に取ります。

    use App\Http\Controllers\LocationsController;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Redirect;

    Route::get('/locations/{location:slug}', [LocationsController::class, 'show'])
            ->name('locations.view')
            ->missing(function (Request $request) {
                return Redirect::route('locations.index');
            });

<a name="implicit-enum-binding"></a>
### 暗黙のEnumバインディング

PHP8.1から、[Enum](https://www.php.net/manual/ja/language.enumerations.backed.php)のサポートが導入されました。この機能を補完するために、Laravelではルート定義に[値に依存したEnums](https://www.php.net/manual/ja/language.enumerations.backed.php)をタイプヒントすることができ、Laravelはそのルートセグメントが有効なEnum値に対応する場合のみルートを呼び出します。そうでない場合は、404 HTTPレスポンスが自動的に返されます。例えば、次のようなEnumがあるとします。

```php
<?php

namespace App\Enums;

enum Category: string
{
    case Fruits = 'fruits';
    case People = 'people';
}
```

`{category}`ルートセグメントが、`fruits`または`people`である場合にのみ呼び出されるルートを定義できます。そうでなければ、Laravelは404 HTTPレスポンスを返します。

```php
use App\Enums\Category;
use Illuminate\Support\Facades\Route;

Route::get('/categories/{category}', function (Category $category) {
    return $category->value;
});
```

<a name="explicit-binding"></a>
### 明示的な結合

モデル結合を使用するために、Laravelの暗黙的な規約ベースのモデル解決を使用する必要はありません。ルートパラメータとモデルの対応を明示的に定義することもできます。明示的な結合を登録するときは、ルーターの`model`メソッドを使用して、パラメータのクラスを指定します。明示的なモデルバインディングは`AppServiceProvider`クラスの`boot`メソッドの最初で定義してください。

    use App\Models\User;
    use Illuminate\Support\Facades\Route;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Route::model('user', User::class);
    }

次に、`{user}`パラメータを含むルートを定義します。

    use App\Models\User;

    Route::get('/users/{user}', function (User $user) {
        // ...
    });

すべての`{user}`パラメータを`App\Models\User`モデルに結合したので、このクラスのインスタンスがルートに注入されます。したがって、たとえば`users/1`へのリクエストは、IDが`1`のデータベースから`User`インスタンスを注入します。

一致するモデルインスタンスがデータベースに見つからない場合、404 HTTPレスポンスが自動的に生成されます。

<a name="customizing-the-resolution-logic"></a>
#### 結合解決ロジックのカスタマイズ

独自のモデル結合解決ロジックを定義したい場合は、`Route::bind`メソッドを使います。`bind`メソッドへ渡すクロージャは、URIセグメントの値を受け取り、ルートへ注入するクラスのインスタンスを返します。繰り返しますが、このカスタマイズはアプリケーションの `AppServiceProvider`の`boot`メソッドで行う必要があります。

    use App\Models\User;
    use Illuminate\Support\Facades\Route;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Route::bind('user', function (string $value) {
            return User::where('name', $value)->firstOrFail();
        });
    }

別の方法として、Eloquentモデルの`resolveRouteBinding`メソッドをオーバーライドすることもできます。このメソッドはURIセグメントの値を受け取り、ルートに挿入する必要があるクラスのインスタンスを返す必要があります。

    /**
     * 値と結合するモデルの取得
     *
     * @param  mixed  $value
     * @param  string|null  $field
     * @return \Illuminate\Database\Eloquent\Model|null
     */
    public function resolveRouteBinding($value, $field = null)
    {
        return $this->where('name', $value)->firstOrFail();
    }

ルートが[明確な結合のスコープ](#implicit-model-binding-scoping)を利用している場合、`resolveChildRouteBinding`メソッドを使用して親モデルの子結合を解決します。

    /**
     * 値と結合するモデルの取得
     *
     * @param  string  $childType
     * @param  mixed  $value
     * @param  string|null  $field
     * @return \Illuminate\Database\Eloquent\Model|null
     */
    public function resolveChildRouteBinding($childType, $value, $field)
    {
        return parent::resolveChildRouteBinding($childType, $value, $field);
    }

<a name="fallback-routes"></a>
## フォールバックルート

`Route::fallback`メソッドを使用して、全ルートが受信リクエストと一致しない場合に実行するルートを定義できます。通常、未処理のリクエストは、アプリケーションの例外ハンドラを介して「404」ページを自動的にレンダします。しかし、通常は`routes/web.php`ファイル内で`fallback`ルートを定義しますので、`web`ミドルウェアグループ内のすべてのミドルウェアがルートに適用されるわけです。必要に応じ、このルートにミドルウェアを自由に追加できます。

    Route::fallback(function () {
        // ...
    });

> [!WARNING]
> フォールバックルートは、常にアプリケーションのルート登録で最後に指定してください。

<a name="rate-limiting"></a>
## レート制限

<a name="defining-rate-limiters"></a>
### レート制限の定義

Laravelには、強力でカスタマイズ可能なレート制限サービスがあり、特定のルートやルートグループのトラフィック量を制限するために利用できます。使い始めるには、アプリケーションのニーズに合わせ、レート制限の設定を定義する必要があります。

レート制限は、アプリケーションの`App\Providers\AppServiceProvider`クラスの`boot`メソッドの中で定義します。

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

/**
 * アプリケーションの全サービスの初期起動処理
 */
protected function boot(): void
{
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
    });
}
```

レート制限は、`RateLimiter`ファサードの`for`メソッドを使用して定義します。`for`メソッドは、レート制限名と、レート制限をしていするルートへ適用する必要がある制限構成を返すクロージャを引数に取ります。制限設定は、`Illuminate\Cache\RateLimiting\Limit`クラスのインスタンスです。このクラスには、制限を簡単に定義するのに役立つ「組み立て」メソッドを用意しています。レート制限名は、任意の文字列にできます。

    use Illuminate\Cache\RateLimiting\Limit;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\RateLimiter;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    protected function boot(): void
    {
        RateLimiter::for('global', function (Request $request) {
            return Limit::perMinute(1000);
        });
    }

受信リクエストが指定したレート制限を超えると、429 HTTPステータスコードのレスポンスをLaravelは自動的に返します。レート制限によって返す独自のレスポンスを定義する場合は、`response`メソッドを使用できます。

    RateLimiter::for('global', function (Request $request) {
        return Limit::perMinute(1000)->response(function (Request $request, array $headers) {
            return response('Custom response...', 429, $headers);
        });
    });

レート制限コールバックは受信HTTPリクエストインスタンスを受け取るため、受信リクエストや認証済みユーザーに基づいて適切なレート制限を動的に構築できます。

    RateLimiter::for('uploads', function (Request $request) {
        return $request->user()->vipCustomer()
                    ? Limit::none()
                    : Limit::perMinute(100);
    });

<a name="segmenting-rate-limits"></a>
#### レート制限のセグメント化

レート制限を任意の値でセグメント化したい場合があります。たとえばIPアドレスごとに、ユーザーに対し1分間に100回まで特定のルートへアクセス可能にしたい場合です。これを実現するには、レート制限を作成するときに`by`メソッドを使用できます。

    RateLimiter::for('uploads', function (Request $request) {
        return $request->user()->vipCustomer()
                    ? Limit::none()
                    : Limit::perMinute(100)->by($request->ip());
    });

別の例を使ってこの機能を説明すると、認証されたユーザーIDごとに１分間に１００回、ゲスト用のIPアドレスごとに１分間に１０回、ルートへのアクセス制限ができます。

    RateLimiter::for('uploads', function (Request $request) {
        return $request->user()
                    ? Limit::perMinute(100)->by($request->user()->id)
                    : Limit::perMinute(10)->by($request->ip());
    });

<a name="multiple-rate-limits"></a>
#### 複数のレート制限

必要に応じて、指定したレート制限設定のレート制限配列を返せます。各レート制限は、配列内へ配置した順序に基づいてルートに対して評価されます。

    RateLimiter::for('login', function (Request $request) {
        return [
            Limit::perMinute(500),
            Limit::perMinute(3)->by($request->input('email')),
        ];
    });

<a name="attaching-rate-limiters-to-routes"></a>
### ルートへのレート制限指定

レート制限は、`throttle` [ミドルウェア](/docs/{{version}}/middleware)を使用してルートまたはルートグループに指定します。このスロットルミドルウェアは、ルートに割り当てたいレート制限名を引数に取ります。

    Route::middleware(['throttle:uploads'])->group(function () {
        Route::post('/audio', function () {
            // ...
        });

        Route::post('/video', function () {
            // ...
        });
    });

<a name="throttling-with-redis"></a>
#### Redisによるスロットリング

`throttle`ミドルウェアはデフォルトで、`Illuminate\Routing\Middleware\ThrottleRequests`クラスへマッピングします。しかし、アプリケーションのキャッシュドライバとしてRedisを使う場合は、レート制限を管理するためにRedisを使うようにLaravelへ指示したいかもしれません。それには、アプリケーションの`bootstrap/app.php`ファイルで`throttleWithRedis`メソッドを使用します。このメソッドは`throttle`ミドルウェアを`Illuminate\Routing\Middleware\ThrottleRequestsWithRedis`ミドルウェアクラスへマッピングします

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->throttleWithRedis();
        // ...
    })

<a name="form-method-spoofing"></a>
## 疑似フォームメソッド

HTMLフォームは、`PUT`、`PATCH`、`DELETE`アクションをサポートしていません。したがって、HTMLフォームから呼び出される`PUT`、`PATCH`、または`DELETE`ルートを定義するときは、フォームに非表示の`_method`フィールドを追加する必要があります。`_method`フィールドで送信された値は、HTTPリクエストメソッドとして使用します。

    <form action="/example" method="POST">
        <input type="hidden" name="_method" value="PUT">
        <input type="hidden" name="_token" value="{{ csrf_token() }}">
    </form>

使いやすいように、`@method` [Bladeディレクティブ](/docs/{{version}}/blade)を使用しても、`_method`入力フィールドを生成できます。

    <form action="/example" method="POST">
        @method('PUT')
        @csrf
    </form>

<a name="accessing-the-current-route"></a>
## 現在のルートへのアクセス

`Route`ファサードの`current`、`currentRouteName`、`currentRouteAction`メソッドを使用して、受信リクエストを処理するルートに関する情報にアクセスできます。

    use Illuminate\Support\Facades\Route;

    $route = Route::current(); // Illuminate\Routing\Route
    $name = Route::currentRouteName(); // 文字列
    $action = Route::currentRouteAction(); // 文字列

ルートとルートクラスで使用可能な全メソッドを確認するには、[ルートファサードの元となるクラス](https://laravel.com/api/{{version}}/Illuminate/Routing/Router.html)と[ルートインスタンス](https://laravel.com/api/{{version}}/Illuminate/Routing/Route.html)の両方のAPIドキュメントを参照してください

<a name="cors"></a>
## オリジン間リソース共有 (CORS)

LaravelはCORS `OPTIONS` HTTPリクエストに対して、設定した値で自動的にレスポンスできます。`OPTIONS`リクエストは、アプリケーションのグローバルミドルウェアスタックに含めている`HandleCors`[ミドルウェア](/docs/{{version}}/middleware) が自動的に処理します。

アプリケーションのCORS設定値をカスタマイズする必要も起きるでしょう。この場合、`config:publish` Artisanコマンドを使用し、`cors`設定ファイルをリソース公開します。

```shell
php artisan config:publish cors
```

このコマンドは、`cors.php`設定ファイルをアプリケーションの`config`ディレクトリへ配置します。

> [!NOTE]
> CORSおよびCORSヘッダの詳細は、[CORSに関するMDN Webドキュメント](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#The_HTTP_response_headers)を参照してください。

<a name="route-caching"></a>
## ルートのキャッシュ

アプリケーションを本番環境へデプロイするときは、Laravelのルートキャッシュの利点を利用するべきでしょう。ルートキャッシュを使用すると、アプリケーションのルートをすべて登録ためにかかる時間が大幅に短縮できます。ルートキャッシュを生成するには、`route：cache` Artisanコマンドを実行します。

```shell
php artisan route:cache
```

このコマンドを実行した後、キャッシュされたルートファイルはすべてのリクエストでロードされます。新しいルートを追加する場合は、新しいルートキャッシュを生成する必要があることに注意してください。このため、プロジェクトのデプロイ中にのみ`route：cache`コマンドを実行する必要があります。

`route：clear`コマンドを使用して、ルートキャッシュをクリアできます。

```shell
php artisan route:clear
```
