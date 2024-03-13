# ミドルウェア

- [イントロダクション](#introduction)
- [ミドルウェアの定義](#defining-middleware)
- [ミドルウェアの登録](#registering-middleware)
    - [グローバルミドルウェア](#global-middleware)
    - [ルートに対するミドルウェアの指定](#assigning-middleware-to-routes)
    - [ミドルウェアグループ](#middleware-groups)
    - [Middleware Aliases](#middleware-aliases)
    - [ミドルウェアの順序](#sorting-middleware)
- [ミドルウェアのパラメータ](#middleware-parameters)
- [終了処理ミドルウェア](#terminable-middleware)

<a name="introduction"></a>
## イントロダクション

ミドルウェアは、アプリケーションに入るHTTPリクエストを検査およびフィルタリングするための便利なメカニズムを提供します。たとえば、Laravelには、アプリケーションのユーザーが認証されていることを確認するミドルウェアが含まれています。ユーザーが認証されていない場合、ミドルウェアはユーザーをアプリケーションのログイン画面にリダイレクトします。逆に、ユーザーが認証されている場合、ミドルウェアはリクエストをアプリケーションへ進めることを許可します。

Additional middleware can be written to perform a variety of tasks besides authentication. For example, a logging middleware might log all incoming requests to your application. A variety of middleware are included in Laravel, including middleware for authentication and CSRF protection; however, all user-defined middleware are typically located in your application's `app/Http/Middleware` directory.

<a name="defining-middleware"></a>
## ミドルウェアの定義

新しいミドルウェアを作成するには、`make:middleware`　Artisanコマンドを使用します。

```shell
php artisan make:middleware EnsureTokenIsValid
```

このコマンドは、新しい`EnsureTokenIsValid`クラスを`app/Http/Middleware`ディレクトリ内に配置します。例としてこのミドルウェアで、リクエストが供給する`token`入力が、指定値と一致する場合にのみ、ルートへのアクセスを許可します。それ以外の場合は、ユーザーを`home` URIへリダイレクトしましょう。

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
                return redirect('home');
            }

            return $next($request);
        }
    }

ご覧のとおり、与えられた`token`がシークレットトークンと一致しない場合、ミドルウェアはHTTPリダイレクトをクライアントに返します。それ以外の場合、リクエストはさらにアプリケーションに渡されます。リクエストをアプリケーションのより深いところに渡す(ミドルウェアが「パス」できるようにする)には、`$request`を使用して`$next`コールバックを呼び出す必要があります。

ミドルウェアは、HTTPリクエストがアプリケーションに到達する前に通過しなければならない一連の「レイヤー」として考えるのがベストです。各レイヤーはリクエストを検査したり、完全に拒否したりすることができます。

> [!NOTE]
> すべてのミドルウェアは[サービスコンテナ](/docs/{{version}}/container)を介して依存解決されるため、ミドルウェアのコンストラクター内で必要な依存関係をタイプヒントで指定できます。

<a name="before-after-middleware"></a>
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

If you want a middleware to run during every HTTP request to your application, you may append it to the global middleware stack in your application's `bootstrap/app.php` file:

    use App\Http\Middleware\EnsureTokenIsValid;

    ->withMiddleware(function (Middleware $middleware) {
         $middleware->append(EnsureTokenIsValid::class);
    })

The `$middleware` object provided to the `withMiddleware` closure is an instance of `Illuminate\Foundation\Configuration\Middleware` and is responsible for managing the middleware assigned to your application's routes. The `append` method adds the middleware to the end of the list of global middleware. If you would like to add a middleware to the beginning of the list, you should use the `prepend` method.

<a name="manually-managing-laravels-default-global-middleware"></a>
#### Manually Managing Laravel's Default Global Middleware

If you would like to manage Laravel's global middleware stack manually, you may provide Laravel's default stack of global middleware to the `use` method. Then, you may adjust the default middleware stack as necessary:

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

Sometimes you may want to group several middleware under a single key to make them easier to assign to routes. You may accomplish this using the `appendToGroup` method within your application's `bootstrap/app.php` file:

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

Middleware groups may be assigned to routes and controller actions using the same syntax as individual middleware:

    Route::get('/', function () {
        // ...
    })->middleware('group-name');

    Route::middleware(['group-name'])->group(function () {
        // ...
    });

<a name="laravels-default-middleware-groups"></a>
#### Laravel's Default Middleware Groups

Laravel includes predefined `web` and `api` middleware groups that contain common middleware you may want to apply to your web and API routes. Remember, Laravel automatically applies these middleware groups to the corresponding `routes/web.php` and `routes/api.php` files:

| The `web` Middleware Group
|--------------
| `Illuminate\Cookie\Middleware\EncryptCookies`
| `Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse`
| `Illuminate\Session\Middleware\StartSession`
| `Illuminate\View\Middleware\ShareErrorsFromSession`
| `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken`
| `Illuminate\Routing\Middleware\SubstituteBindings`

| The `api` Middleware Group
|--------------
| `Illuminate\Routing\Middleware\SubstituteBindings`

If you would like to append or prepend middleware to these groups, you may use the `web` and `api` methods within your application's `bootstrap/app.php` file. The `web` and `api` methods are convenient alternatives to the `appendToGroup` method:

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

You may even replace one of Laravel's default middleware group entries with a custom middleware of your own:

    use App\Http\Middleware\StartCustomSession;
    use Illuminate\Session\Middleware\StartSession;

    $middleware->web(replace: [
        StartSession::class => StartCustomSession::class,
    ]);

Or, you may remove a middleware entirely:

    $middleware->web(remove: [
        StartSession::class,
    ]);

<a name="manually-managing-laravels-default-middleware-groups"></a>
#### Manually Managing Laravel's Default Middleware Groups

If you would like to manually manage all of the middleware within Laravel's default `web` and `api` middleware groups, you may redefine the groups entirely. The example below will define the `web` and `api` middleware groups with their default middleware, allowing you to customize them as necessary:

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
> By default, the `web` and `api` middleware groups are automatically applied to your application's corresponding `routes/web.php` and `routes/api.php` files by the `bootstrap/app.php` file.

<a name="middleware-aliases"></a>
### Middleware Aliases

You may assign aliases to middleware in your application's `bootstrap/app.php` file. Middleware aliases allows you to define a short alias for a given middleware class, which can be especially useful for middleware with long class names:

    use App\Http\Middleware\EnsureUserIsSubscribed;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'subscribed' => EnsureUserIsSubscribed::class
        ]);
    })

Once the middleware alias has been defined in your application's `bootstrap/app.php` file, you may use the alias when assigning the middleware to routes:

    Route::get('/profile', function () {
        // ...
    })->middleware('subscribed');

For convenience, some of Laravel's built-in middleware are aliased by default. For example, the `auth` middleware is an alias for the `Illuminate\Auth\Middleware\Authenticate` middleware. Below is a list of the default middleware aliases:

| Alias | Middleware
|-------|------------
`auth` | `Illuminate\Auth\Middleware\Authenticate`
`auth.basic` | `Illuminate\Auth\Middleware\AuthenticateWithBasicAuth`
`auth.session` | `Illuminate\Session\Middleware\AuthenticateSession`
`cache.headers` | `Illuminate\Http\Middleware\SetCacheHeaders`
`can` | `Illuminate\Auth\Middleware\Authorize`
`guest` | `Illuminate\Auth\Middleware\RedirectIfAuthenticated`
`password.confirm` | `Illuminate\Auth\Middleware\RequirePassword`
`precognitive` | `Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests`
`signed` | `Illuminate\Routing\Middleware\ValidateSignature`
`subscribed` | `\Spark\Http\Middleware\VerifyBillableIsSubscribed`
`throttle` | `Illuminate\Routing\Middleware\ThrottleRequests` or `Illuminate\Routing\Middleware\ThrottleRequestsWithRedis`
`verified` | `Illuminate\Auth\Middleware\EnsureEmailIsVerified`

<a name="sorting-middleware"></a>
### ミドルウェアの順序

Rarely, you may need your middleware to execute in a specific order but not have control over their order when they are assigned to the route. In these situations, you may specify your middleware priority using the `priority` method in your application's `bootstrap/app.php` file:

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

複数のパラメーターはコンマで区切る必要があります。

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

The `terminate` method should receive both the request and the response. Once you have defined a terminable middleware, you should add it to the list of routes or global middleware in your application's `bootstrap/app.php` file.

ミドルウェアで`terminate`メソッドを呼び出すと、Laravelは[サービスコンテナ](/docs/{{version}}/container)からミドルウェアの新しいインスタンスを依存解決します。`handle`メソッドと`terminate`メソッドが呼び出されたときに同じミドルウェアインスタンスを使用する場合は、コンテナの`singleton`メソッドを使用してミドルウェアをコンテナに登録します。通常、これは`AppServiceProvider`の`register`メソッドで実行する必要があります。

    use App\Http\Middleware\TerminatingMiddleware;

    /**
     * 全アプリケーションサービスの登録
     */
    public function register(): void
    {
        $this->app->singleton(TerminatingMiddleware::class);
    }
