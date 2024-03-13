# リクエストのライフサイクル

- [イントロダクション](#introduction)
- [ライフサイクル概論](#lifecycle-overview)
    - [最初のステップ](#first-steps)
    - [HTTP／コンソールカーネル](#http-console-kernels)
    - [サービスプロバイダ](#service-providers)
    - [ルート](#routing)
    - [最後](#finishing-up)
- [サービスプロバイダに注目](#focus-on-service-providers)

<a name="introduction"></a>
## イントロダクション

「現実の世界」でツールを使用するとき、そのツールがどのように機能するかを理解していれば、より自信を感じられます。アプリケーション開発も例外ではありません。開発ツールがどのように機能するかを理解すると、それらをより快適に、自信を持って使用できるようになります。

このドキュメントの目的は、Laravelフレームワークがどのように機能するかについての概要を説明することです。フレームワーク全体をよりよく理解することで、すべてが「魔法」ではなくなり、アプリケーションの構築に自信が持てるようになります。すべての用語をすぐに理解できない場合でも、戸惑う必要はありません。何が起こっているのかという基本的な部分を把握しておけば、ドキュメントの他のセクションを探索するにつれて知識が深まります。

<a name="lifecycle-overview"></a>
## ライフサイクル概論

<a name="first-steps"></a>
### 最初のステップ

Laravelアプリケーションへのすべてのリクエストのエントリポイントは`public/index.php`ファイルです。すべてのリクエストは、Webサーバ(Apache/Nginx)設定によってこのファイルに送信されます。`index.php`ファイルには多くのコードが含まれていません。むしろ、フレームワークの残りの部分をロードするための開始点と言えるでしょう。

`index.php`ファイルは、Composerで生成したオートローダー定義をロードし、Laravelアプリケーションのインスタンスを`bootstrap/app.php`から取得します。Laravel自体が取る最初のアクションは、アプリケーション／[サービスコンテナ](/docs/{{version}}/container)のインスタンスを作成することです。

<a name="http-console-kernels"></a>
### HTTP／コンソールカーネル

Next, the incoming request is sent to either the HTTP kernel or the console kernel, using the `handleRequest` or `handleCommand` methods of the application instance, depending on the type of request entering the application. These two kernels serve as the central location through which all requests flow. For now, let's just focus on the HTTP kernel, which is an instance of `Illuminate\Foundation\Http\Kernel`.

The HTTP kernel defines an array of `bootstrappers` that will be run before the request is executed. These bootstrappers configure error handling, configure logging, [detect the application environment](/docs/{{version}}/configuration#environment-configuration), and perform other tasks that need to be done before the request is actually handled. Typically, these classes handle internal Laravel configuration that you do not need to worry about.

The HTTP kernel is also responsible for passing the request though the application's middleware stack. These middleware handle reading and writing the [HTTP session](/docs/{{version}}/session), determining if the application is in maintenance mode, [verifying the CSRF token](/docs/{{version}}/csrf), and more. We'll talk more about these soon.

HTTPカーネルの`handle`メソッドの引数は非常に単純です。`Request`を受け取り、`Response`を返します。カーネルをアプリケーション全体を表す大きなブラックボックスであると考えてください。HTTPリクエストを取り込み、HTTPレスポンスを返します。

<a name="service-providers"></a>
### サービスプロバイダ

One of the most important kernel bootstrapping actions is loading the [service providers](/docs/{{version}}/providers) for your application. Service providers are responsible for bootstrapping all of the framework's various components, such as the database, queue, validation, and routing components.

Laravelはこのプロバイダのリストを繰り返し処理し、それぞれをインスタンス化します。プロバイダをインスタンス化した後、すべてのプロバイダの`register`メソッドを呼び出します。次に、すべてのプロバイダを登録し、各プロバイダで`boot`メソッドを呼び出します。これは、`boot`メソッドが実行されるまでに、すべてのコンテナバインディングが登録され、利用可能になっていることに、サービスプロバイダが依存しているからです。

Laravelが提供する基本的なすべての主機能は、サービスプロバイダによって初期起動および設定されます。フレームワークによって提供される非常に多くの機能を初期起動し設定するため、サービスプロバイダはLaravel初期起動プロセス全体の最も重要な側面です。

While the framework internally uses dozens of service providers, you also have the option to create your own. You can find a list of the user-defined or third-party service providers that your application is using in the `bootstrap/providers.php` file.

<a name="routing"></a>
### ルート

アプリケーションが初期起動され、すべてのサービスプロバイダが登録されると、`Request`がルータに渡されてディスパッチされます。ルータは、ルートまたはコントローラへリクエストをディスパッチし、ルート固有のミドルウェアを実行します。

Middleware provide a convenient mechanism for filtering or examining HTTP requests entering your application. For example, Laravel includes a middleware that verifies if the user of your application is authenticated. If the user is not authenticated, the middleware will redirect the user to the login screen. However, if the user is authenticated, the middleware will allow the request to proceed further into the application. Some middleware are assigned to all routes within the application, like `PreventRequestsDuringMaintenance`, while some are only assigned to specific routes or route groups. You can learn more about middleware by reading the complete [middleware documentation](/docs/{{version}}/middleware).

リクエストが一致したルートへ割り当てられたすべてのミドルウェアをパスした場合は、ルートまたはコントローラメソッドが実行され、ルートまたはコントローラメソッドが返すレスポンスがルートのミドルウェアチェーンを介して返送されます。

<a name="finishing-up"></a>
### 最後

ルートまたはコントローラメソッドがレスポンスを返すと、レスポンスはルートのミドルウェアを介して外側に戻り、アプリケーションに送信レスポンスを変更または検査する機会を与えます。

Finally, once the response travels back through the middleware, the HTTP kernel's `handle` method returns the response object to the `handleRequest` of the application instance, and this method calls the `send` method on the returned response. The `send` method sends the response content to the user's web browser. We've now completed our journey through the entire Laravel request lifecycle!

<a name="focus-on-service-providers"></a>
## サービスプロバイダに注目

サービスプロバイダは、Laravelアプリケーションを初期起動するための真の鍵です。アプリケーションインスタンスが作成され、サービスプロバイダが登録され、リクエストが初期起動を終えたアプリケーションに渡されます。とても簡単です！

Having a firm grasp of how a Laravel application is built and bootstrapped via service providers is very valuable. Your application's user-defined service providers are stored in the `app/Providers` directory.

デフォルトの`AppServiceProvider`はほとんど空です。このプロバイダは、アプリケーション独自の初期起動処理およびサービスコンテナ結合を追加するのに最適な場所です。大規模なアプリケーションの場合、アプリケーションで使用する特定のサービスに対して、それぞれがよりきめ細かい初期処理を備えた複数のサービスプロバイダを作成することをお勧めします。
