# URL生成

- [イントロダクション](#introduction)
- [基礎](#the-basics)
    - [URLの生成](#generating-urls)
    - [現在のURLへのアクセス](#accessing-the-current-url)
- [名前付きルートのURL](#urls-for-named-routes)
    - [署名付きURL](#signed-urls)
- [コントローラアクションのURL](#urls-for-controller-actions)
- [デフォルト値](#default-values)

<a name="introduction"></a>
## イントロダクション

Laravelは、アプリケーションのURLを生成するのに役立つヘルパをいくつか提供しています。これらのヘルパは、主にテンプレートとAPIレスポンスでリンクを構築するとき、またはアプリケーションの別の部分へのリダイレクトレスポンスを生成するときに役立ちます。

<a name="the-basics"></a>
## 基礎

<a name="generating-urls"></a>
### URLの生成

`url`ヘルパは、アプリケーションの任意のURLを生成するために使用します。生成したURLは、アプリケーションが処理している現在のリクエストのスキーム(HTTPまたはHTTPS)とホストを自動的に使用します。

    $post = App\Models\Post::find(1);

    echo url("/posts/{$post->id}");

    // http://example.com/posts/1

<a name="accessing-the-current-url"></a>
### 現在のURLへのアクセス

`url`ヘルパにパスを指定しないと、`Illuminate\Routing\UrlGenerator`インスタンスが返され、現在のURLに関する情報へアクセスできます。

    // クエリ文字列を除いた現在のURL
    echo url()->current();

    // クエリ文字列を含んだ現在のURL
    echo url()->full();

    // 直前のリクエストの完全なURL
    echo url()->previous();

こうしたメソッドには、`URL`[ファサード](/docs/{{version}}/facades)を使用してもアクセスできます。

    use Illuminate\Support\Facades\URL;

    echo URL::current();

<a name="urls-for-named-routes"></a>
## 名前付きルートのURL

`route`ヘルパは、[名前付きルート](/docs/{{version}}/routing#named-routes)へのURLを生成するためにも使用できます。名前付きルートを使用すると、ルートで定義する実際のURLと結合せずにURLを生成できます。したがって、ルートのURLが変更された場合でも、`route`関数の呼び出しを変更する必要はありません。たとえば、アプリケーションに次のように定義されたルートが含まれているとします。

    Route::get('/post/{post}', function (Post $post) {
        // ...
    })->name('post.show');

このルートへのURLを生成するには、次のように`route`ヘルパを使用します。

    echo route('post.show', ['post' => 1]);

    // http://example.com/post/1

もちろん、`route`ヘルパを使用して、複数のパラメーターを持つルートのURLを生成することもできます。

    Route::get('/post/{post}/comment/{comment}', function (Post $post, Comment $comment) {
        // ...
    })->name('comment.show');

    echo route('comment.show', ['post' => 1, 'comment' => 3]);

    // http://example.com/post/1/comment/3

ルートの定義パラメータに対応しない過剰な配列要素は、URLのクエリ文字列として追加されます。

    echo route('post.show', ['post' => 1, 'search' => 'rocket']);

    // http://example.com/post/1?search=rocket

<a name="eloquent-models"></a>
#### Eloquentモデル

[Eloquentモデル](/docs/{{version}}/eloquent)のルートキー（通常は主キー）を使ってURLを生成することが多いでしょう。そのため、パラメータ値としてEloquentモデルを渡せます。`route`ヘルパは、モデルのルートキーを自動的に抽出します。

    echo route('post.show', ['post' => $post]);

<a name="signed-urls"></a>
### 署名付きURL

Laravelでは名前付きルートに対し、簡単に「署名付きURL」を作成できます。このURLは「署名」ハッシュをクエリ文字列として付加し、作成されてからそのURLが変更されていないかをLaravelで確認できるようにします。署名付きURLは公にアクセスさせるルートではあるが、URL操作に対する保護レイヤが必要な場合とくに便利です。

たとえば、公の「購読終了」リンクを顧客へのメールへ用意するために、署名付きURLが使用できます。名前付きルートに対し署名URLを作成するには、`URL`ファサードの`signedRoute`メソッドを使用します。

    use Illuminate\Support\Facades\URL;

    return URL::signedRoute('unsubscribe', ['user' => 1]);

`signedRoute`メソッドで`absolute`引数を指定すれば、署名付きURLハッシュからドメインを除外できます。

    return URL::signedRoute('unsubscribe', ['user' => 1], absolute: false);

指定する時間が経過すると期限切れになる一時的な署名付きルートURLを生成する場合は、`temporarySignedRoute`メソッドを使用します。Laravelが一時的な署名付きルートURLを検証するとき、署名付きURLにエンコードされている有効期限のタイムスタンプが経過していないことを確認します。

    use Illuminate\Support\Facades\URL;

    return URL::temporarySignedRoute(
        'unsubscribe', now()->addMinutes(30), ['user' => 1]
    );

<a name="validating-signed-route-requests"></a>
#### 署名付きルートリクエストの検査

受信リクエストに有効な署名があるかどうかを確認するには、受信した`Illuminate\Http\Request`インスタンスで`hasValidSignature`メソッドを呼び出します。

    use Illuminate\Http\Request;

    Route::get('/unsubscribe/{user}', function (Request $request) {
        if (! $request->hasValidSignature()) {
            abort(401);
        }

        // ...
    })->name('unsubscribe');

クライアントサイドのペジネーションなど、アプリケーションのフロントエンドが署名付きURLにデータを追加することを許可する必要が起きる場合があります。そのようなときは、`hasValidSignatureWhileIgnoring`メソッドを用いて、署名付きURLを検証する際に無視すべきリクエストクエリパラメータを指定します。パラメータの無視を許すと、誰でもリクエストのそのパラメータを変更できる点に注意してください。

    if (! $request->hasValidSignatureWhileIgnoring(['page', 'order'])) {
        abort(401);
    }

受信リクエストインスタンスを使って署名付きURLを検証する代わりに、`signed`（`Illuminate\Routing\Middleware\ValidateSignature`）[ミドルウェア](/docs/{{version}}/middleware)をルートへ指定できます。受信クエストが有効な署名を持っていない場合、ミドルウェアは自動的に `403` HTTP レスポンスをします。

    Route::post('/unsubscribe/{user}', function (Request $request) {
        // ...
    })->name('unsubscribe')->middleware('signed');

署名付きURLがURLハッシュにドメインを含んでいない場合、ミドルウェアで`relative`引数を与える必要があります。

    Route::post('/unsubscribe/{user}', function (Request $request) {
        // ...
    })->name('unsubscribe')->middleware('signed:relative');

<a name="responding-to-invalid-signed-routes"></a>
#### 無効な署名付きルートのレスポンス

有効期限が切れた署名付きURLへ誰かがアクセスすると、`403`HTTPステータスコードの一般的なエラーページを表示します。しかし、アプリケーションの`bootstrap/app.php`ファイルで、`InvalidSignatureException`例外用のカスタム「レンダ」クロージャを定義することで、この動作をカスタマイズできます。

    use Illuminate\Routing\Exceptions\InvalidSignatureException;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->render(function (InvalidSignatureException $e) {
            return response()->view('error.link-expired', [], 403);
        });
    })

<a name="urls-for-controller-actions"></a>
## コントローラアクションのURL

`action`関数は、指定するコントローラアクションに対するURLを生成します。

    use App\Http\Controllers\HomeController;

    $url = action([HomeController::class, 'index']);

コントローラメソッドがルートパラメータを受け入れる場合、関数の２番目の引数としてルートパラメータの連想配列を渡せます。

    $url = action([UserController::class, 'profile'], ['id' => 1]);

<a name="default-values"></a>
## デフォルト値

アプリケーションにより、特定のURLパラメータのデフォルト値をリクエスト全体で指定したい場合もあります。たとえば、多くのルートで`{locale}`パラメータを定義していると想像してください。

    Route::get('/{locale}/posts', function () {
        // ...
    })->name('post.index');

毎回`route`ヘルパを呼び出すごとに、`locale`をいつも指定するのは厄介です。そのため、現在のリクエストの間、常に適用されるこのパラメートのデフォルト値は、`URL::defaults`メソッドを使用し定義できます。現在のリクエストでアクセスできるように、[ルートミドルウェア](/docs/{{version}}/middleware#assigning-middleware-to-routes)から、このメソッドを呼び出したいかと思います。

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\URL;
    use Symfony\Component\HttpFoundation\Response;

    class SetDefaultLocaleForUrls
    {
        /**
         * 受信リクエストの処理
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            URL::defaults(['locale' => $request->user()->locale]);

            return $next($request);
        }
    }

一度`locale`パラメータに対するデフォルト値をセットしたら、`route`ヘルパを使いURLを生成する時に、値を渡す必要はもうありません。

<a name="url-defaults-middleware-priority"></a>
#### URLのデフォルトとミドルウェアの優先度

URLのデフォルト値を設定すると、Laravelの暗黙のモデル結合の処理と干渉することがあります。そのため、URLのデフォルト値を設定する[ミドルウェア](/docs/{{version}}/middleware#sorting-middleware)は、Laravel自身の`SubstituteBindings`ミドルウェアよりも先に実行されるように、[優先順位を付ける]必要があります。アプリケーションの`bootstrap/app.php`ファイルにある`priority`ミドルウェアメソッドを使用することで指定できます。

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->priority([
        \Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests::class,
        \Illuminate\Cookie\Middleware\EncryptCookies::class,
        \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
        \Illuminate\Contracts\Auth\Middleware\AuthenticatesRequests::class,
        \Illuminate\Routing\Middleware\ThrottleRequests::class,
        \Illuminate\Routing\Middleware\ThrottleRequestsWithRedis::class,
        \Illuminate\Session\Middleware\AuthenticateSession::class,
        \App\Http\Middleware\SetDefaultLocaleForUrls::class, // [tl! add]
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
        \Illuminate\Auth\Middleware\Authorize::class,
    ]);
})
```
