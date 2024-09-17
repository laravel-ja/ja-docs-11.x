# ミドルウェア

- [イントロダクション](#introduction)
- [ミドルウェアの定義](#defining-middleware)
- [ミドルウェアの登録](#registering-middleware)
    - [グローバルミドルウェア](#global-middleware)
    - [ルートに対するミドルウェアの指定](#assigning-middleware-to-routes)
    - [ミドルウェアグループ](#middleware-groups)
    - [ミドルウェアエイリアス](#middleware-aliases)
    - [ミドルウェアの順序](#sorting-middleware)
- [ミドルウェアのパラメータ](#middleware-parameters)
- [終了処理ミドルウェア](#terminable-middleware)

<a name="introduction"></a>
## イントロダクション

ミドルウェアは、アプリケーションに入るHTTPリクエストを検査およびフィルタリングするための便利なメカニズムを提供します。たとえば、Laravelには、アプリケーションのユーザーが認証されていることを確認するミドルウェアが含まれています。ユーザーが認証されていない場合、ミドルウェアはユーザーをアプリケーションのログイン画面にリダイレクトします。逆に、ユーザーが認証されている場合、ミドルウェアはリクエストをアプリケーションへ進めることを許可します。

認証以外のさまざまなタスクを実行するために、追加のミドルウェアを書くこともできます。例えば、ログミドルウェアは、アプリケーションへの全受信リクエストを記録するために用意できます。Laravelには、認証用ミドルウェアやCSRF保護用ミドルウェアなど、様々なミドルウェアを用意していますが、ユーザー定義のミドルウェアは通常、アプリケーションの`app/Http/Middleware`ディレクトリへ配置します。

<a name="defining-middleware"></a>
## ミドルウェアの定義

新しいミドルウェアを作成するには、`make:middleware`　Artisanコマンドを使用します。

```shell
php artisan make:middleware EnsureTokenIsValid
```

このコマンドは、新しい`EnsureTokenIsValid`クラスを`app/Http/Middleware`ディレクトリ内に配置します。例としてこのミドルウェアで、リクエストが供給する`token`入力が、指定値と一致する場合にのみ、ルートへのアクセスを許可します。それ以外の場合は、ユーザーを`/home` URIへリダイレクトしましょう。

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class EnsureTokenIsValid
    {
        /**
         * 受信リクエストの処理
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            if ($request->input('token') !== 'my-secret-token') {
                return redirect('/home');
            }

            return $next($request);
        }
    }

ご覧のとおり、与えられた`token`がシークレットトークンと一致しない場合、ミドルウェアはHTTPリダイレクトをクライアントに返します。それ以外の場合、リクエストはさらにアプリケーションに渡されます。リクエストをアプリケーションのより深いところに渡す(ミドルウェアが「パス」できるようにする)には、`$request`を使用して`$next`コールバックを呼び出す必要があります。

ミドルウェアは、HTTPリクエストがアプリケーションに到達する前に通過しなければならない一連の「レイヤー」として考えるのがベストです。各レイヤーはリクエストを検査したり、完全に拒否したりすることができます。

> [!NOTE]
> すべてのミドルウェアは[サービスコンテナ](/docs/{{version}}/container)を介して依存解決されるため、ミドルウェアのコンストラクター内で必要な依存関係をタイプヒントで指定できます。

<a name="middleware-and-responses"></a>
#### ミドルウェアとレスポンス

もちろん、ミドルウェアはアプリケーションのより深部へリクエストの処理を委ねるその前後にあるタスクを実行できます。たとえば、次のミドルウェアは、リクエストがアプリケーションによって処理される**前に**いくつかのタスクを実行します。

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class BeforeMiddleware
    {
        public function handle(Request $request, Closure $next): Response
        {
            // アクションの実行…

            return $next($request);
        }
    }

一方、このミドルウェアは、リクエストがアプリケーションによって処理された**後に**そのタスクを実行します。

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class AfterMiddleware
    {
        public function handle(Request $request, Closure $next): Response
        {
            $response = $next($request);

            // アクションの実行…

            return $response;
        }
    }

<a name="registering-middleware"></a>
## ミドルウェアの登録

<a name="global-middleware"></a>
### グローバルミドルウェア

アプリケーションがHTTPリクエストを受信するたび、ミドルウェアを実行したい場合は、アプリケーションの`bootstrap/app.php`ファイルにあるグローバルミドルウェアスタックにミドルウェアを追加します：

    use App\Http\Middleware\EnsureTokenIsValid;

    ->withMiddleware(function (Middleware $middleware) {
         $middleware->append(EnsureTokenIsValid::class);
    })

`withMiddleware`クロージャへ渡す`$middleware`オブジェクトは、`Illuminate\Foundation\Configuration\Middleware`インスタンスで、アプリケーションのルートへ割り当てたミドルウェアを管理します。`append`メソッドは、ミドルウェアをグローバルミドルウェアリストの最後に追加します。ミドルウェアをリストの先頭に追加したい場合は、`prepend`メソッドを使うべきです。

<a name="manually-managing-laravels-default-global-middleware"></a>
#### Laravelのデフォルトグローバルミドルウェアの手作業による管理

Laravelのグローバルミドルウェアスタックを手作業で管理したい場合は、`use`メソッドへLaravelのデフォルトのグローバルミドルウェアスタックを指定します。その後、必要に応じてデフォルトのミドルウェアスタックを調整してください。

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->use([
            // \Illuminate\Http\Middleware\TrustHosts::class,
            \Illuminate\Http\Middleware\TrustProxies::class,
            \Illuminate\Http\Middleware\HandleCors::class,
            \Illuminate\Foundation\Http\Middleware\PreventRequestsDuringMaintenance::class,
            \Illuminate\Http\Middleware\ValidatePostSize::class,
            \Illuminate\Foundation\Http\Middleware\TrimStrings::class,
            \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
        ]);
    })

<a name="assigning-middleware-to-routes"></a>
### ルートに対するミドルウェアの指定

特定のルートにミドルウェアを割り当てたい場合は、ルート定義時に`middleware`メソッドを呼び出してください。

    use App\Http\Middleware\EnsureTokenIsValid;

    Route::get('/profile', function () {
        // ...
    })->middleware(EnsureTokenIsValid::class);

ミドルウェア名の配列を`middleware`メソッドへ渡し、ルートに複数のミドルウェアを割り当てることもできます。

    Route::get('/', function () {
        // ...
    })->middleware([First::class, Second::class]);

<a name="excluding-middleware"></a>
#### 除外ミドルウェア

ミドルウェアをルートのグループに割り当てる場合、あるミドルウェアをグループ内の個々のルートに適用しないようにする必要が起きることもあります。これは、`withoutMiddleware`メソッドを使用して実行できます。

    use App\Http\Middleware\EnsureTokenIsValid;

    Route::middleware([EnsureTokenIsValid::class])->group(function () {
        Route::get('/', function () {
            // …
        });

        Route::get('/profile', function () {
            // …
        })->withoutMiddleware([EnsureTokenIsValid::class]);
    });

また、ルート定義の[グループ](/docs/{{version}}/routing#route-groups)全体から特定のミドルウェアのセットを除外することもできます。

    use App\Http\Middleware\EnsureTokenIsValid;

    Route::withoutMiddleware([EnsureTokenIsValid::class])->group(function () {
        Route::get('/profile', function () {
            // …
        });
    });

`withoutMiddleware`メソッドはルートミドルウェアのみを削除でき、[グローバルミドルウェア](#global-middleware)には適用されません。

<a name="middleware-groups"></a>
### ミドルウェアグループ

ルートに割り当てやすくするために、複数のミドルウェアを１つのキーでグループ化したいことがあります。その場合はアプリケーションの`bootstrap/app.php`ファイルで、`appendToGroup`メソッドを使ってください。

    use App\Http\Middleware\First;
    use App\Http\Middleware\Second;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->appendToGroup('group-name', [
            First::class,
            Second::class,
        ]);

        $middleware->prependToGroup('group-name', [
            First::class,
            Second::class,
        ]);
    })

ミドルウェアグループは、個々のミドルウェアと同じ構文を使用して、ルートとコントローラアクションへ指定できます。

    Route::get('/', function () {
        // ...
    })->middleware('group-name');

    Route::middleware(['group-name'])->group(function () {
        // ...
    });

<a name="laravels-default-middleware-groups"></a>
#### Laravelのデフォルトミドルウェアグループ

Laravelには定義済みの`web`ミドルウェアグループと`api`ミドルウェアグループがあり、WebルートとAPIルートへ適用する一般的なミドルウェアを用意しています。Laravelはこれらのミドルウェアグループを`routes/web.php`と`routes/api.php`ファイルに対応して、自動的に適用します。

<div class="overflow-auto">

|`web`ミドルウェアグループ |
| --- |
| `Illuminate\Cookie\Middleware\EncryptCookies` |
| `Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse` |
| `Illuminate\Session\Middleware\StartSession` |
| `Illuminate\View\Middleware\ShareErrorsFromSession` |
| `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` |
| `Illuminate\Routing\Middleware\SubstituteBindings` |

</div>

<div class="overflow-auto">

| The `api`ミドルウェアグループ |
| --- |
| `Illuminate\Routing\Middleware\SubstituteBindings` |

</div>

これらのグループの前後へミドルウェアを追加したい場合は、アプリケーションの`bootstrap/app.php`ファイル内で`web`メソッドと`api`メソッドを使用してください。`web`メソッドと`api`メソッドは、`appendToGroup`メソッドに代わる便利なメソッドです。

    use App\Http\Middleware\EnsureTokenIsValid;
    use App\Http\Middleware\EnsureUserIsSubscribed;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->web(append: [
            EnsureUserIsSubscribed::class,
        ]);

        $middleware->api(prepend: [
            EnsureTokenIsValid::class,
        ]);
    })

Laravelのデフォルトミドルウェアグループのエントリを、独自のカスタムミドルウェアへ置き換えることもできます。

    use App\Http\Middleware\StartCustomSession;
    use Illuminate\Session\Middleware\StartSession;

    $middleware->web(replace: [
        StartSession::class => StartCustomSession::class,
    ]);

あるいは、ミドルウェアを完全に削除することもできます。

    $middleware->web(remove: [
        StartSession::class,
    ]);

<a name="manually-managing-laravels-default-middleware-groups"></a>
#### Laravelのデフォルトミドルウェアグループを手作業で管理

Laravelのデフォルトの`web`ミドルウェアグループと`api`ミドルウェアグループ内のすべてのミドルウェアを手作業で管理したい場合は、グループを完全に再定義してください。以下の例は、`web`ミドルウェアグループと`api`ミドルウェアグループをデフォルトのミドルウェアで定義し、必要に応じてカスタマイズできるようにしています。

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->group('web', [
            \Illuminate\Cookie\Middleware\EncryptCookies::class,
            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
            \Illuminate\Session\Middleware\StartSession::class,
            \Illuminate\View\Middleware\ShareErrorsFromSession::class,
            \Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
            // \Illuminate\Session\Middleware\AuthenticateSession::class,
        ]);

        $middleware->group('api', [
            // \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
            // 'throttle:api',
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ]);
    })

> [!NOTE]
> デフォルトでは、`bootstrap/app.php`ファイルにより、`web`と`api`ミドルウェアグループがアプリケーションの対応する`routes/web.php`と`routes/api.php`ファイルへ自動的に適用されます。

<a name="middleware-aliases"></a>
### ミドルウェアエイリアス

アプリケーションの`bootstrap/app.php`ファイルで、ミドルウェアへエイリアスを割り当てられます。ミドルウェアのエイリアスを使うと、指定するミドルウェアクラスに対して、短いエイリアス名を定義できます。

    use App\Http\Middleware\EnsureUserIsSubscribed;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'subscribed' => EnsureUserIsSubscribed::class
        ]);
    })

ミドルウェアのエイリアスをアプリケーションの`bootstrap/app.php`ファイルで定義したら、ミドルウェアをルーティングに割り当てるときにエイリアスを使えます。

    Route::get('/profile', function () {
        // ...
    })->middleware('subscribed');

便利なように、Laravelの組み込みミドルウェアのいくつかは、デフォルトでエイリアスを定義しています。例えば、`auth`ミドルウェアは`Illuminate\Auth\Middleware\Authenticate`ミドルウェアのエイリアスです。下記は、デフォルトミドルウェアのエイリアスのリストです。

<div class="overflow-auto">

| エイリアス | ミドルウェア |
| --- | --- |
| `auth` | `Illuminate\Auth\Middleware\Authenticate` |
| `auth.basic` | `Illuminate\Auth\Middleware\AuthenticateWithBasicAuth` |
| `auth.session` | `Illuminate\Session\Middleware\AuthenticateSession` |
| `cache.headers` | `Illuminate\Http\Middleware\SetCacheHeaders` |
| `can` | `Illuminate\Auth\Middleware\Authorize` |
| `guest` | `Illuminate\Auth\Middleware\RedirectIfAuthenticated` |
| `password.confirm` | `Illuminate\Auth\Middleware\RequirePassword` |
| `precognitive` | `Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests` |
| `signed` | `Illuminate\Routing\Middleware\ValidateSignature` |
| `subscribed` | `\Spark\Http\Middleware\VerifyBillableIsSubscribed` |
| `throttle` | `Illuminate\Routing\Middleware\ThrottleRequests` or `Illuminate\Routing\Middleware\ThrottleRequestsWithRedis` |
| `verified` | `Illuminate\Auth\Middleware\EnsureEmailIsVerified` |

</div>

<a name="sorting-middleware"></a>
### ミドルウェアの順序

まれに、ミドルウェアを特定の順番で実行する必要があるけれども、ルート指定時に順番を制御できないことがあります。このような状況では、アプリケーションの `bootstrap/app.php`ファイルで、`priority`メソッドを使用して、ミドルウェアの優先順位を指定してください。

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->priority([
            \Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests::class,
            \Illuminate\Cookie\Middleware\EncryptCookies::class,
            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
            \Illuminate\Session\Middleware\StartSession::class,
            \Illuminate\View\Middleware\ShareErrorsFromSession::class,
            \Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
            \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
            \Illuminate\Routing\Middleware\ThrottleRequests::class,
            \Illuminate\Routing\Middleware\ThrottleRequestsWithRedis::class,
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
            \Illuminate\Contracts\Auth\Middleware\AuthenticatesRequests::class,
            \Illuminate\Auth\Middleware\Authorize::class,
        ]);
    })

<a name="middleware-parameters"></a>
## ミドルウェアのパラメータ

ミドルウェアは追加のパラメータを受け取ることもできます。たとえば、アプリケーションが特定のアクションを実行する前に、認証済みユーザーが特定の「役割り（role）」を持っていることを確認する必要がある場合、追加の引数として役割名を受け取る`EnsureUserHasRole`ミドルウェアを作成できます。

追加のミドルウェアパラメータは、`$next`引数の後にミドルウェアに渡されます。

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class EnsureUserHasRole
    {
        /**
         * 受信リクエストの処理
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next, string $role): Response
        {
            if (! $request->user()->hasRole($role)) {
                // リダイレクト…
            }

            return $next($request);
        }

    }

ミドルウェアのパラメータはルート定義時に、ミドルウェア名とパラメータを「:」で区切って指定します。

    Route::put('/post/{id}', function (string $id) {
        // ...
    })->middleware('role:editor');

複数のパラメータはコンマで区切る必要があります。

    Route::put('/post/{id}', function (string $id) {
        // ...
    })->middleware('role:editor,publisher');

<a name="terminable-middleware"></a>
## 終了処理ミドルウェア

HTTPレスポンスがブラウザに送信された後、ミドルウェアが何らかの作業を行う必要がある場合があります。ミドルウェアで`terminate`メソッドを定義し、WebサーバがFastCGIを使用している場合、レスポンスがブラウザに送信された後、`terminate`メソッドが自動的に呼び出されます。

    <?php

    namespace Illuminate\Session\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class TerminatingMiddleware
    {
        /**
         * 受信リクエストの処理
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            return $next($request);
        }

        /**
         * レスポンスがブラウザに送信された後にタスクを処理
         */
        public function terminate(Request $request, Response $response): void
        {
            // …
        }
    }

`terminate`メソッドはリクエストとレスポンスの両方を受け取る必要があります。終了処理ミドルウェアを定義したら、アプリケーションの`bootstrap/app.php`ファイルにあるルートまたはグローバルミドルウェアのリストに追加する必要があります。

ミドルウェアで`terminate`メソッドを呼び出すと、Laravelは[サービスコンテナ](/docs/{{version}}/container)からミドルウェアの新しいインスタンスを依存解決します。`handle`メソッドと`terminate`メソッドが呼び出されたときに同じミドルウェアインスタンスを使用する場合は、コンテナの`singleton`メソッドを使用してミドルウェアをコンテナに登録します。通常、これは`AppServiceProvider`の`register`メソッドで実行する必要があります。

    use App\Http\Middleware\TerminatingMiddleware;

    /**
     * 全アプリケーションサービスの登録
     */
    public function register(): void
    {
        $this->app->singleton(TerminatingMiddleware::class);
    }
