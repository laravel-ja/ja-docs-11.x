# エラー処理

- [イントロダクション](#introduction)
- [設定](#configuration)
- [例外処理](#handling-exceptions)
    - [例外のレポート](#reporting-exceptions)
    - [例外のログレベル](#exception-log-levels)
    - [タイプによる例外の無視](#ignoring-exceptions-by-type)
    - [例外のレンダ](#rendering-exceptions)
    - [Reportable／Renderable例外](#renderable-exceptions)
- [例外レポートの限定](#throttling-reported-exceptions)
- [HTTP例外](#http-exceptions)
    - [カスタムHTTPエラーページ](#custom-http-error-pages)

<a name="introduction"></a>
## イントロダクション

新しいLaravelプロジェクトを開始すると、エラーと例外処理はあらかじめ設定済みです。しかし、いつでも、アプリケーションの`bootstrap/app.php`の`withExceptions`メソッドを使用して、例外をどのように報告し、アプリケーションによってレンダするかを管理できます。

`withExceptions`クロージャへ渡される`$exceptions`オブジェクトは`Illuminate\Foundation\Configuration\Exceptions`インスタンスで、アプリケーションの例外処理を管理します。このドキュメントを通して、このオブジェクトを深く掘り下げていきましょう。

<a name="configuration"></a>
## 設定

`config/app.php`設定ファイルの`debug`オプションは、エラーに関する情報が実際にユーザーに表示される量を決定します。デフォルトでは、このオプションは、`.env`ファイルに保存されている`APP_DEBUG`環境変数の値を尊重するように設定されています。

ローカル開発中は、`APP_DEBUG`環境変数を`true`に設定する必要があります。**実稼働環境では、この値は常に`false`である必要があります。本番環境で値が`true`に設定されていると、機密性の高い設定値がアプリケーションのエンドユーザーに公開されるリスクが起きます。**

<a name="handling-exceptions"></a>
## 例外処理

<a name="reporting-exceptions"></a>
### 例外のレポート

Laravelでは、例外レポートを使用して、例外をログに記録したり、外部サービスの[Sentry](https://github.com/getsentry/sentry-laravel)、[Flare](https://flareapp.io)へ送信したりします。例外はデフォルトで、[ログ](/docs/{{version}}/logging)設定に基づき、ログへ記録します。しかし、例外をどのようにログに記録するかは自由です。

異なるタイプの例外を別々の方法で報告する必要がある場合、アプリケーションの`bootstrap/app.php`で`report`例外メソッドを使用して、特定タイプの例外を報告する必要があるときに実行するクロージャを登録できます。Laravelは、クロージャのタイプヒントを調べ、クロージャが報告する例外のタイプを決定します。

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->report(function (InvalidOrderException $e) {
            // ...
        });
    })

`report`メソッドを使用してカスタム例外報告コールバックを登録すると、Laravelはアプリケーションのデフォルトのログ設定を使用して例外をログに記録します。デフォルトのログスタックへ例外が伝わるのを停止したい場合は、報告コールバックを定義するときに`stop`メソッドを使用するか、コールバックから`false`を返します：

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->report(function (InvalidOrderException $e) {
            // ...
        })->stop();

        $exceptions->report(function (InvalidOrderException $e) {
            return false;
        });
    })

> [!NOTE]
> 特定の例外のレポートをカスタマイズするには、[レポート可能な例外](/docs/{{version}}/errors#renderable-exceptions)を利用することもできます。

<a name="global-log-context"></a>
#### グローバルログコンテキスト

可能であれば、Laravelは自動的に現在のユーザーIDをコンテキストデータとしてすべての例外のログメッセージへ追加します。アプリケーションの`bootstrap/app.php`ファイルの`context`例外メソッドを使用して、独自のグローバルなコンテキストデータを定義可能です。この情報は、アプリケーションにが書き出す、すべての例外のログメッセージに含まれます。

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->context(fn () => [
            'foo' => 'bar',
        ]);
    })

<a name="exception-log-context"></a>
#### 例外ログコンテキスト

すべてのログメッセージにコンテキストを追加することは便利ですが、特定の例外にはログに含めたい固有のコンテキストがある場合もあります。アプリケーションの例外へ`context`メソッドを定義することにより、例外のログエントリに追加すべき関連データを指定できます。

    <?php

    namespace App\Exceptions;

    use Exception;

    class InvalidOrderException extends Exception
    {
        // ...

        /**
         * 例外のコンテキスト情報を取得
         *
         * @return array<string, mixed>
         */
        public function context(): array
        {
            return ['order_id' => $this->orderId];
        }
    }

<a name="the-report-helper"></a>
#### `report`ヘルパ

場合により、例外を報告する必要があるが、現在のリクエストの処理を続けなければならないこともあるでしょう。ヘルパ関数`report`を使うと、ユーザーにエラーページを表示することなく、例外を素早く報告できます。

    public function isValid(string $value): bool
    {
        try {
            // 値のバリデーション…
        } catch (Throwable $e) {
            report($e);

            return false;
        }
    }

<a name="deduplicating-reported-exceptions"></a>
#### レポート済み例外の重複

アプリケーション全体で`report`関数を使用している場合、同じ例外を複数回報告することがあり、ログに重複したエントリが作成されることがあります。

例外で単一インスタンスが一度だけ報告されることを保証したい場合、アプリケーションの`bootstrap/app.php`ファイルで`dontReportDuplicates`例外メソッドを呼び出してください。

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->dontReportDuplicates();
    })

これで、同じ例外インスタンスで`report`ヘルパが呼び出された場合、最初に呼び出されたものだけが報告されるようになります。

```php
$original = new RuntimeException('Whoops!');

report($original); // レポートされる

try {
    throw $original;
} catch (Throwable $caught) {
    report($caught); // 無視される
}

report($original); // 無視される
report($caught); // 無視される
```

<a name="exception-log-levels"></a>
### 例外のログレベル

アプリケーションの[ログ](/docs/{{version}}/logging)にメッセージが書き込まれるとき、そのメッセージは指定した[ログレベル](/docs/{{version}}/logging#log-levels)で書かれ、これは書き込まれるメッセージの緊急度や重要度を表します。

上記のように、`report`メソッドを使用してカスタム例外報告コールバックを登録した場合でも、Laravelはアプリケーションのデフォルトのログ設定を使用して例外をログに記録します。しかし、ログレベルはメッセージがログに記録されるチャンネルに影響を与えることがあるので、特定の例外をログに記録するログレベルを設定したい場合もあるでしょう。

これを行うには、アプリケーションの`bootstrap/app.php`ファイルで、`level`例外メソッドを使用します。このメソッドは最初の引数として例外タイプを取り、２番目の引数にログレベルを取ります：

    use PDOException;
    use Psr\Log\LogLevel;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->level(PDOException::class, LogLevel::CRITICAL);
    })

<a name="ignoring-exceptions-by-type"></a>
### タイプによる例外の無視

アプリケーションを構築するとき、報告したくないタイプの例外があるでしょう。これらの例外を無視するためには、アプリケーションの`bootstrap/app.php`ファイルで`dontReport`例外メソッドを使ってください。このメソッドへ指定したクラスは、報告しません。しかしながら、それらのクラスが、カスタムレンダロジックを持っている可能性はあります。

    use App\Exceptions\InvalidOrderException;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->dontReport([
            InvalidOrderException::class,
        ]);
    })

Laravelは内部的に、あらかじめいくつかのタイプのエラーを無視しています。例えば、404 HTTPエラーや無効なCSRFトークンによって生成された419 HTTPレスポンスから生じる例外などです。Laravelが指定しているタイプの例外を無視しないように指示したい場合は、アプリケーションの`bootstrap/app.php`ファイルで、`stopIgnoring`例外メソッドを使用してください。

    use Symfony\Component\HttpKernel\Exception\HttpException;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->stopIgnoring(HttpException::class);
    })

<a name="rendering-exceptions"></a>
### 例外のレンダ

Laravelの例外ハンドラはデフォルトで、例外をHTTPレスポンスへ変換します。しかし、特定タイプの例外のためにカスタムレンダクロージャを自由に登録することもできます。アプリケーションの`boostrap/app.php`ファイルで、`render`例外メソッドを使うことで、これを実現できます。

`render`メソッドへ渡すクロージャは`Illuminate\Http\Response`インスタンスを返す必要があり、これは`response`ヘルパで生成できます。Laravelは、クロージャのタイプヒントを調べ、クロージャがレンダする例外タイプを決定します。

    use App\Exceptions\InvalidOrderException;
    use Illuminate\Http\Request;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->render(function (InvalidOrderException $e, Request $request) {
            return response()->view('errors.invalid-order', [], 500);
        });
    })

`render`メソッドを使用して、`NotFoundHttpException`のようなLaravelやSymfonyの組み込み例外のレンダ動作をオーバーライドすることもできます。`render`メソッドに渡たしたクロージャが値を返さない場合、Laravelデフォルトの例外レンダを利用します。

    use Illuminate\Http\Request;
    use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->render(function (NotFoundHttpException $e, Request $request) {
            if ($request->is('api/*')) {
                return response()->json([
                    'message' => 'Record not found.'
                ], 404);
            }
        });
    })

<a name="rendering-exceptions-as-json"></a>
#### 例外をJSONでレンダする

例外をレンダするとき、Laravelはリクエストの`Accept`ヘッダに基づいて、例外をHTMLレスポンスとしてレンダするか、JSONレスポンスとしてレンダするかを自動的に判断します。Laravelが例外レスポンスをHTMLとJSONのどちらでレンダするかを決定する方法をカスタマイズしたい場合は、`shouldRenderJsonWhen`メソッドを利用します。

    use Illuminate\Http\Request;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->shouldRenderJsonWhen(function (Request $request, Throwable $e) {
            if ($request->is('admin/*')) {
                return true;
            }

            return $request->expectsJson();
        });
    })

<a name="customizing-the-exception-response"></a>
#### 例外レスポンスのカスタマイズ

まれに、Laravelの例外ハンドラがレンダするHTTPレスポンス全体をカスタマイズする必要があるかもしれません。これを行うには、`respond`メソッドを使用してレスポンスのカスタマイズクロージャを登録します。

    use Symfony\Component\HttpFoundation\Response;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->respond(function (Response $response) {
            if ($response->getStatusCode() === 419) {
                return back()->with([
                    'message' => 'The page expired, please try again.',
                ]);
            }

            return $response;
        });
    })

<a name="renderable-exceptions"></a>
### Reportable／Renderable例外

アプリケーションの`bootstrap/app.php`ファイルでカスタムレポートとレンダ動作を定義する代わりに、アプリケーションの例外に直接、`report`と`render`メソッドを定義できます。これらのメソッドが存在するとき、フレームワークは自動的に呼び出します。

    <?php

    namespace App\Exceptions;

    use Exception;
    use Illuminate\Http\Request;
    use Illuminate\Http\Response;

    class InvalidOrderException extends Exception
    {
        /**
         * 例外をレポート
         */
        public function report(): void
        {
            // ...
        }

        /**
         * 例外をHTTPレスポンスへレンダ
         */
        public function render(Request $request): Response
        {
            return response(/* ... */);
        }
    }

LaravelやSymfonyの組み込み済み例外など、既存のレンダ可能（Renderable）な例外を拡張している場合は、例外の`render`メソッドから`false`を返し、例外のデフォルトHTTPレスポンスをレンダできます。

    /**
     * 例外をHTTPレスポンスへレンダする
     */
    public function render(Request $request): Response|bool
    {
        if (/** この例外をレポートする必要があるかを判断… */) {

            return response(/* ... */);
        }

        return false;
    }

特定の条件が満たされた場合にのみ必要なカスタムレポートロジックが例外に含まれている場合は、デフォルトの例外処理設定を使用して例外をレポートするようにLaravelに指示する必要が起き得ます。これを行うには、例外の`report`メソッドから`false`を返します。

    /**
     * 例外を報告
     */
    public function report(): bool
    {
        if (/** この例外をレポートする必要があるかを判断… */) {

            // ...

            return true;
        }

        return false;
    }

> [!NOTE]
> `report`メソッドで必要な依存関係をタイプヒントすると、Laravelの[サービスコンテナ](/docs/{{version}}/container)がメソッドへ自動的に依存を注入します。

<a name="throttling-reported-exceptions"></a>
### 例外レポートの限定

アプリケーションが非常に多くの例外を報告する場合、実際にログに記録されたり、アプリケーションの外部エラー追跡サービスに送信されたりする例外の数を絞り込みたくなるでしょう。

例外のランダムサンプルレートを取るには、アプリケーションの`bootstrap/app.php`ファイルで、`throttle`例外メソッドを使ってください。`throttle`メソッドは`Lottery`インスタンスを返すクロージャを引数に取ります。

    use Illuminate\Support\Lottery;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            return Lottery::odds(1, 1000);
        });
    })

また、例外の種類に基づいた条件付きでサンプリングすることも可能です。特定の例外クラスのインスタンスのみをサンプリングしたい場合は、そのクラスの`Lottery`インスタンスのみを返します

    use App\Exceptions\ApiMonitoringException;
    use Illuminate\Support\Lottery;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            if ($e instanceof ApiMonitoringException) {
                return Lottery::odds(1, 1000);
            }
        });
    })

また、`Lottery`の代わりに`Limit`インスタンスを返せば、ログに記録する例外や外部のエラー追跡サービスに送信する例外を制限できます。これは、アプリケーションで使用しているサードパーティのサービスがダウンした場合などで、突然例外がログに殺到するのを防ぎたい場合に便利です。

    use Illuminate\Broadcasting\BroadcastException;
    use Illuminate\Cache\RateLimiting\Limit;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            if ($e instanceof BroadcastException) {
                return Limit::perMinute(300);
            }
        });
    })

リミットは例外のクラスをレートリミットキーに、デフォルトで使用します。これをカスタマイズするには、`Limit`の`by`メソッドを使用して独自キーを指定します。

    use Illuminate\Broadcasting\BroadcastException;
    use Illuminate\Cache\RateLimiting\Limit;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            if ($e instanceof BroadcastException) {
                return Limit::perMinute(300)->by($e->getMessage());
            }
        });
    })


もちろん、異なる例外に対して`Lottery`と`Limit`のインスタンスを混ぜて返すこともできます。

    use App\Exceptions\ApiMonitoringException;
    use Illuminate\Broadcasting\BroadcastException;
    use Illuminate\Cache\RateLimiting\Limit;
    use Illuminate\Support\Lottery;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            return match (true) {
                $e instanceof BroadcastException => Limit::perMinute(300),
                $e instanceof ApiMonitoringException => Lottery::odds(1, 1000),
                default => Limit::none(),
            };
        });
    })

<a name="http-exceptions"></a>
## HTTP例外

一部の例外は、サーバからのHTTPエラーコードを表します。たとえば、「ページが見つかりません」エラー(404)、「不正なエラー」(401)、または開発者が500エラーを生成する可能性もあります。アプリケーションのどこからでもこのようなレスポンスを生成したい場合は、`abort`ヘルパを使用できます。

    abort(404);

<a name="custom-http-error-pages"></a>
### カスタムHTTPエラーページ

Laravelを使用すると、さまざまなHTTPステータスコードのカスタムエラーページを簡単に表示できます。たとえば、404 HTTPステータスコードのエラーページをカスタマイズする場合は、`resources/views/errors/404.blade.php`ビューテンプレートを作成します。このビューは、アプリケーションが生成するすべての404エラーでレンダされます。このディレクトリ内のビューには、対応するHTTPステータスコードと一致する名前を付ける必要があります。`abort`関数によって生成された`Symfony\Component\HttpKernel\Exception\HttpException`インスタンスは`$exception`変数としてビューに渡されます。

    <h2>{{ $exception->getMessage() }}</h2>

`vendor:publish` Artisanコマンドを使用して、Laravelのデフォルトのエラーページテンプレートをリソース公開できます。テンプレートをリソース公開したら、好みに合わせてカスタマイズしてください。

```shell
php artisan vendor:publish --tag=laravel-errors
```

<a name="fallback-http-error-pages"></a>
#### HTTPエラーページのフォールバック

一連のHTTPステータスコードに対応する「フォールバック」エラーページを定義することもできます。このページは、発生した特定のHTTPステータスコードに対応するページが存在しない場合にレンダされます。これには、アプリケーションの`resources/views/errors`ディレクトリに、`4xx.blade.php`テンプレートと`5xx.blade.php`テンプレートを定義します。

フォールバックエラーページを定義する場合、フォールバックページは`404`、`500`、`503`エラーレスポンスには影響しません。Laravelは内部的に、これらのステータスコード専用のページを持っているためです。これらのステータスコードに対してレンダするページをカスタマイズするには、個別にカスタムエラーページを定義する必要があります。