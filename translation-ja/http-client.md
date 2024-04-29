# HTTPクライアント

- [イントロダクション](#introduction)
- [リクエストの作成](#making-requests)
    - [リクエストデータ](#request-data)
    - [ヘッダ](#headers)
    - [認証](#authentication)
    - [タイムアウト](#timeout)
    - [再試行](#retries)
    - [エラー処理](#error-handling)
    - [Guzzleミドルウェア](#guzzle-middleware)
    - [Guzzleオプション](#guzzle-options)
- [同時リクエスト](#concurrent-requests)
- [マクロ](#macros)
- [テスト](#testing)
    - [レスポンスのfake](#faking-responses)
    - [レスポンスの検査](#inspecting-requests)
    - [行き先がないリクエストの防止](#preventing-stray-requests)
- [イベント](#events)

<a name="introduction"></a>
## イントロダクション

Laravelは、[Guzzle HTTPクライアント](http://docs.guzzlephp.org/en/stable/)の周りに表現力豊かで最小限のAPIを提供し、他のWebアプリケーションと通信するための外部HTTPリクエストをすばやく作成できるようにします。LaravelによるGuzzleのラッパーは、最も一般的なユースケースと素晴らしい開発者エクスペリエンスに焦点を当てています。

<a name="making-requests"></a>
## リクエストの作成

リクエストを行うには、`Http`ファサードが提供する`head`、`get`、`post`、`put`、`patch`、`delete`メソッドを使用します。まず、外部のURLに対して基本的な`GET`リクエストを行う方法を見てみましょう。

    use Illuminate\Support\Facades\Http;

    $response = Http::get('http://example.com');

`get`メソッドは`Illuminate\Http\Client\Response`のインスタンスを返します。これは、レスポンスを調べるために使用できるさまざまなメソッドを提供します。

    $response->body() : string;
    $response->json($key = null, $default = null) : mixed;
    $response->object() : object;
    $response->collect($key = null) : Illuminate\Support\Collection;
    $response->status() : int;
    $response->successful() : bool;
    $response->redirect(): bool;
    $response->failed() : bool;
    $response->clientError() : bool;
    $response->header($header) : string;
    $response->headers() : array;

`Illuminate\Http\Client\Response`オブジェクトはPHPの`ArrayAccess`インターフェイスも実装しており、そのレスポンスのJSONレスポンスデータへ直接アクセスできます。

    return Http::get('http://example.com/users/1')['name'];

上記レスポンスメソッドに加え、以下のメソッドにより、レスポンスが特定のステータスコードを持つか判断できます。

    $response->ok() : bool;                  // 200 OK
    $response->created() : bool;             // 201 Created
    $response->accepted() : bool;            // 202 Accepted
    $response->noContent() : bool;           // 204 No Content
    $response->movedPermanently() : bool;    // 301 Moved Permanently
    $response->found() : bool;               // 302 Found
    $response->badRequest() : bool;          // 400 Bad Request
    $response->unauthorized() : bool;        // 401 Unauthorized
    $response->paymentRequired() : bool;     // 402 Payment Required
    $response->forbidden() : bool;           // 403 Forbidden
    $response->notFound() : bool;            // 404 Not Found
    $response->requestTimeout() : bool;      // 408 Request Timeout
    $response->conflict() : bool;            // 409 Conflict
    $response->unprocessableEntity() : bool; // 422 Unprocessable Entity
    $response->tooManyRequests() : bool;     // 429 Too Many Requests
    $response->serverError() : bool;         // 500 Internal Server Error

<a name="uri-templates"></a>
#### URIテンプレート

HTTPクライアントは、[URIテンプレート仕様](https://www.rfc-editor.org/rfc/rfc6570)を用いて、リクエストURLを構築することも可能です。URIテンプレートで展開できるURLパラメータを定義するには、`withUrlParameters`メソッドを使用します。

```php
Http::withUrlParameters([
    'endpoint' => 'https://laravel.com',
    'page' => 'docs',
    'version' => '11.x',
    'topic' => 'validation',
])->get('{+endpoint}/{page}/{version}/{topic}');
```

<a name="dumping-requests"></a>
#### リクエストのダンプ

送信するリクエストインスタンスを送信して、スクリプトの実行を終了する前にダンプしたい場合は、リクエスト定義の先頭に`dd`メソッドを追加できます。

    return Http::dd()->get('http://example.com');

<a name="request-data"></a>
### リクエストデータ

もちろん、`POST`、`PUT`、`PATCH`リクエストを作成するときは、リクエストとともに追加のデータを送信するのが一般的であるため、これらのメソッドは２番目の引数としてデータの配列を受け入れます。デフォルトでデータは`application/json`コンテンツタイプを使用して送信されます。

    use Illuminate\Support\Facades\Http;

    $response = Http::post('http://example.com/users', [
        'name' => 'Steve',
        'role' => 'Network Administrator',
    ]);

<a name="get-request-query-parameters"></a>
#### GETリクエストクエリパラメータ

`GET`リクエストを行うときは、クエリ文字列をURLに直接追加するか、キー／値ペアの配列を`get`メソッドの２番目の引数として渡せます。

    $response = Http::get('http://example.com/users', [
        'name' => 'Taylor',
        'page' => 1,
    ]);

あるいは、`withQueryParameters`メソッドを使うこともできます。

    Http::retry(3, 100)->withQueryParameters([
        'name' => 'Taylor',
        'page' => 1,
    ])->get('http://example.com/users')

<a name="sending-form-url-encoded-requests"></a>
#### フォームURLエンコードされたリクエストの送信

`application/x-www-form-urlencoded`コンテンツタイプを使用してデータを送信する場合は、リクエストを行う前に`asForm`メソッドを呼び出す必要があります。

    $response = Http::asForm()->post('http://example.com/users', [
        'name' => 'Sara',
        'role' => 'Privacy Consultant',
    ]);

<a name="sending-a-raw-request-body"></a>
#### 素のリクエスト本文の送信

リクエストを行うときに素のリクエスト本文を指定する場合は、`withBody`メソッドを使用できます。コンテンツタイプは、メソッドの２番目の引数を介して提供できます。

    $response = Http::withBody(
        base64_encode($photo), 'image/jpeg'
    )->post('http://example.com/photo');

<a name="multi-part-requests"></a>
#### マルチパートリクエスト

ファイルをマルチパートリクエストとして送信する場合は、リクエストを行う前に`attach`メソッドを呼び出す必要があります。このメソッドは、ファイルの名前とその内容を引数に取ります。必要に応じて、ファイルの名前と見なす３番目の引数を指定できます。また、第４引数は、ファイルに関連するヘッダを指定するために使います。

    $response = Http::attach(
        'attachment', file_get_contents('photo.jpg'), 'photo.jpg', ['Content-Type' => 'image/jpeg']
    )->post('http://example.com/attachments');

ファイルの素の内容を渡す代わりに、ストリームリソースを渡すこともできます。

    $photo = fopen('photo.jpg', 'r');

    $response = Http::attach(
        'attachment', $photo, 'photo.jpg'
    )->post('http://example.com/attachments');

<a name="headers"></a>
### ヘッダ

ヘッダは、`withHeaders`メソッドを使用してリクエストに追加できます。この`withHeaders`メソッドは、キー／値ペアの配列を引数に取ります。

    $response = Http::withHeaders([
        'X-First' => 'foo',
        'X-Second' => 'bar'
    ])->post('http://example.com/users', [
        'name' => 'Taylor',
    ]);

`accept`メソッドを使って、アプリケーションがリクエストへのレスポンスとして期待するコンテンツタイプを指定できます。

    $response = Http::accept('application/json')->get('http://example.com/users');

利便性のため、`acceptJson`メソッドを使って、アプリケーションがリクエストへのレスポンスとして`application/json`コンテンツタイプを期待することを素早く指定できます。

    $response = Http::acceptJson()->get('http://example.com/users');

`withHeaders`メソッドは、新しいヘッダをリクエストの既存のヘッダへマージします。必要であれば、`replaceHeaders`メソッドを使用し、すべてのヘッダを完全に置き換えることもできます。

```php
$response = Http::withHeaders([
    'X-Original' => 'foo',
])->replaceHeaders([
    'X-Replacement' => 'bar',
])->post('http://example.com/users', [
    'name' => 'Taylor',
]);
```

<a name="authentication"></a>
### 認証

基本認証のログイン情報とダイジェスト認証ログイン情報は、それぞれ`withBasicAuth`メソッドと`withDigestAuth`メソッドを使用して指定します。

    // BASIC認証
    $response = Http::withBasicAuth('taylor@laravel.com', 'secret')->post(/* ... */);

    // ダイジェスト認証
    $response = Http::withDigestAuth('taylor@laravel.com', 'secret')->post(/* ... */);

<a name="bearer-tokens"></a>
#### Bearerトークン

リクエストの`Authorization`ヘッダにBearerトークンをすばやく追加したい場合は、`withToken`メソッドを使用できます。

    $response = Http::withToken('token')->post(/* ... */);

<a name="timeout"></a>
### タイムアウト

`timeout`メソッドを使用して、レスポンスを待機する最大秒数を指定できます。デフォルトでは、３０秒をすぎるとHTTPクライアントはタイム・アウトします。

    $response = Http::timeout(3)->get(/* ... */);

指定したタイムアウトを超えると、`Illuminate\Http\Client\ConnectionException`インスタンスを投げます。

サーバへの接続を試みる最長待ち秒数を`connectTimeout`メソッドで指定できます。

    $response = Http::connectTimeout(3)->get(/* ... */);

<a name="retries"></a>
### 再試行

クライアントまたはサーバのエラーが発生した場合に、HTTPクライアントがリクエストを自動的に再試行するようにしたい場合は、`retry`メソッドを使用します。`retry`メソッドは、リクエストを試行する最大回数とLaravelが試行の間に待機するミリ秒数を引数に取ります。

    $response = Http::retry(3, 100)->post(/* ... */);

試行間にスリープするミリ秒数を手作業で計算したい場合は、`retry`メソッドの第２引数へクロージャを渡してください。

    use Exception;

    $response = Http::retry(3, function (int $attempt, Exception $exception) {
        return $attempt * 100;
    })->post(/* ... */);

使いやすいように、`retry`メソッドの第１引数へ配列を指定することもできます。この配列は、次の試行までの間に何ミリ秒スリープするか決めるために使われます。

    $response = Http::retry([100, 200])->post(/* ... */);

必要であれば、`retry`メソッドに第３引数を渡せます。第３引数には、実際に再試行を行うかどうかを決定するCallableを指定します。例えば、最初のリクエストで`ConnectionException`が発生した場合にのみ、リクエストを再試行したいとしましょう。

    use Exception;
    use Illuminate\Http\Client\PendingRequest;

    $response = Http::retry(3, 100, function (Exception $exception, PendingRequest $request) {
        return $exception instanceof ConnectionException;
    })->post(/* ... */);

リクエストの試行に失敗した場合、新しく試みる前にリクエストへ変更を加えたい場合があります。これを実現するには、`retry`メソッドに渡すコールバックのrequest引数を変更します。例えば、最初の試行が認証エラーを返した場合、新しい認証トークンを使ってリクエストを再試行したいと思います。

    use Exception;
    use Illuminate\Http\Client\PendingRequest;
    use Illuminate\Http\Client\RequestException;

    $response = Http::withToken($this->getToken())->retry(2, 0, function (Exception $exception, PendingRequest $request) {
        if (! $exception instanceof RequestException || $exception->response->status() !== 401) {
            return false;
        }

        $request->withToken($this->getNewToken());

        return true;
    })->post(/* ... */);

すべてのリクエストが失敗した場合、 `Illuminate\Http\Client\RequestException`インスタンスを投げます。この動作を無効にする場合は、`throw`引数へ`false`を指定してください。無効にすると、すべての再試行のあと、クライアントが最後に受信したレスポンスを返します。

    $response = Http::retry(3, 100, throw: false)->post(/* ... */);

> [!WARNING]
> 接続の問題ですべてのリクエストが失敗した場合は、`throw`引数を`false`に設定していても`Illuminate\Http\Client\ConnectionException`が投げられます。

<a name="error-handling"></a>
### エラー処理

Guzzleのデフォルト動作とは異なり、LaravelのHTTPクライアントラッパーは、クライアントまたはサーバのエラー(サーバからの「400」および「500」レベルの応答)で例外を投げません。`successful`、`clientError`、`serverError`メソッドを使用して、これらのエラーのいずれかが返されたかどうかを判定できます。

    // ステータスコードが200以上300未満か判定
    $response->successful();

    // ステータスコードが400以上か判定
    $response->failed();

    // レスポンスに400レベルのステータスコードがあるかを判定
    $response->clientError();

    // レスポンスに500レベルのステータスコードがあるかを判定
    $response->serverError();

    // クライアントまたはサーバエラーが発生した場合、指定コールバックを即座に実行
    $response->onError(callable $callback);

<a name="throwing-exceptions"></a>
#### 例外を投げる

あるレスポンスインスタンスのレスポンスステータスコードがクライアントまたはサーバのエラーを示している場合に`Illuminate\Http\Client\RequestException`のインスタンスを投げたい場合場合は、`throw`か`throwIf`メソッドを使用します。

    use Illuminate\Http\Client\Response;

    $response = Http::post(/* ... */);

    // クライアントまたはサーバのエラーが発生した場合は、例外を投げる
    $response->throw();

    // エラーが発生し、指定条件が真の場合は、例外を投げる
    $response->throwIf($condition);

    // エラーが発生し、指定クロージャの結果が真の場合は例外を投げる
    $response->throwIf(fn (Response $response) => true);

    // エラーが発生し、指定条件が偽の場合は、例外を投げる
    $response->throwUnless($condition);

    // エラーが発生し、指定クロージャの結果が偽の場合は例外を投げる
    $response->throwUnless(fn (Response $response) => false);

    // レスポンスが特定のステータスコードの場合は、例外を投げる
    $response->throwIfStatus(403);

    // レスポンスが特定のステータスコードでない場合は、例外を投げる
    $response->throwUnlessStatus(200);

    return $response['user']['id'];

`Illuminate\Http\Client\RequestException`インスタンスにはパブリック`$response`プロパティがあり、返ってきたレスポンスを検査できます。

`throw`メソッドは、エラーが発生しなかった場合にレスポンスインスタンスを返すので、他の操作を`throw`メソッドにチェーンできます。

    return Http::post(/* ... */)->throw()->json();

例外がなげられる前に追加のロジックを実行したい場合は、`throw`メソッドにクロージャを渡せます。クロージャを呼び出した後に、例外を自動的に投げるため、クロージャ内から例外を再発行する必要はありません。

    use Illuminate\Http\Client\Response;
    use Illuminate\Http\Client\RequestException;

    return Http::post(/* ... */)->throw(function (Response $response, RequestException $e) {
        // ...
    })->json();

<a name="guzzle-middleware"></a>
### Guzzleミドルウェア

LaravelのHTTPクライアントはGuzzleで動いているので、[Guzzleミドルウェア](https://docs.guzzlephp.org/en/stable/handlers-and-middleware.html)を利用して、送信するリクエストの操作や受信したレスポンスの検査ができます。送信リクエストを操作するには、`withRequestMiddleware`でGuzzleミドルウェアを登録します。

    use Illuminate\Support\Facades\Http;
    use Psr\Http\Message\RequestInterface;

    $response = Http::withRequestMiddleware(
        function (RequestInterface $request) {
            return $request->withHeader('X-Example', 'Value');
        }
    )->get('http://example.com');

同様に、`withResponseMiddleware`メソッドでミドルウェアを登録すれば、受信HTTPレスポンスを検査できます。

    use Illuminate\Support\Facades\Http;
    use Psr\Http\Message\ResponseInterface;

    $response = Http::withResponseMiddleware(
        function (ResponseInterface $response) {
            $header = $response->getHeader('X-Example');

            // ...

            return $response;
        }
    )->get('http://example.com');

<a name="global-middleware"></a>
#### グローバルミドルウェア

時には、すべての送信リクエストと受信レスポンスへ適用するミドルウェアを登録したいこともあるでしょう。それには、`globalRequestMiddleware`メソッドと`globalResponseMiddleware`メソッドを使います。通常、これらのメソッドはアプリケーションの`AppServiceProvider`で、`boot`メソッドから呼び出す必要があります。

```php
use Illuminate\Support\Facades\Http;

Http::globalRequestMiddleware(fn ($request) => $request->withHeader(
    'User-Agent', 'Example Application/1.0'
));

Http::globalResponseMiddleware(fn ($response) => $response->withHeader(
    'X-Finished-At', now()->toDateTimeString()
));
```

<a name="guzzle-options"></a>
### Guzzleオプション

`withOptions`メソッドを使用すると、送信リクエストへ追加の[Guzzleリクエストオプション](http://docs.guzzlephp.org/en/stable/request-options.html)を指定できます。`withOptions`メソッドはキーと値のペアの配列を引数に取ります：

    $response = Http::withOptions([
        'debug' => true,
    ])->get('http://example.com/users');

<a name="global-options"></a>
#### グローバルオプション

すべての送信リクエストに対してデフォルトのオプションを設定するには、`globalOptions`メソッドを利用します。通常、このメソッドはアプリケーションの`AppServiceProvider`の`boot`メソッドから呼び出します。

```php
use Illuminate\Support\Facades\Http;

/**
 * 全アプリケーションサービスの初期起動処理
 */
public function boot(): void
{
    Http::globalOptions([
        'allow_redirects' => false,
    ]);
}
```

<a name="concurrent-requests"></a>
## 同時リクエスト

複数のHTTPリクエストを同時に実行したい場合があります。言い換えれば、複数のリクエストを順番に発行するのではなく、同時にディスパッチしたい状況です。これにより、低速なHTTP APIを操作する際のパフォーマンスが大幅に向上します。

さいわいに、`pool`メソッドを使い、これを実現できます。`pool`メソッドは、`Illuminate\Http\Client\Pool`インスタンスを受け取るクロージャを引数に取り、簡単にリクエストプールにリクエストを追加してディスパッチできます。

    use Illuminate\Http\Client\Pool;
    use Illuminate\Support\Facades\Http;

    $responses = Http::pool(fn (Pool $pool) => [
        $pool->get('http://localhost/first'),
        $pool->get('http://localhost/second'),
        $pool->get('http://localhost/third'),
    ]);

    return $responses[0]->ok() &&
           $responses[1]->ok() &&
           $responses[2]->ok();

ご覧のように、各レスポンスインスタンスは、プールに追加した順でアクセスできます。必要に応じ`as`メソッドを使い、リクエストに名前を付けると、対応するレスポンスへ名前でアクセスできるようになります。

    use Illuminate\Http\Client\Pool;
    use Illuminate\Support\Facades\Http;

    $responses = Http::pool(fn (Pool $pool) => [
        $pool->as('first')->get('http://localhost/first'),
        $pool->as('second')->get('http://localhost/second'),
        $pool->as('third')->get('http://localhost/third'),
    ]);

    return $responses['first']->ok();

<a name="customizing-concurrent-requests"></a>
#### 現在のリクエストのカスタマイズ

`pool`メソッドは、`withHeaders`や`middleware`メソッドのような、他のHTTPクライアントメソッドとチェーンできません。プールしたリクエストへカスタムヘッダやミドルウェアを適用したい場合は、プール内の各リクエストでそれらのオプションを設定する必要があります。

```php
use Illuminate\Http\Client\Pool;
use Illuminate\Support\Facades\Http;

$headers = [
    'X-Example' => 'example',
];

$responses = Http::pool(fn (Pool $pool) => [
    $pool->withHeaders($headers)->get('http://laravel.test/test'),
    $pool->withHeaders($headers)->get('http://laravel.test/test'),
    $pool->withHeaders($headers)->get('http://laravel.test/test'),
]);
```

<a name="macros"></a>
## マクロ

LaravelのHTTPクライアントでは、「マクロ」を定義可能です。マクロは、アプリケーション全体でサービスとやり取りする際に、共通のリクエストパスやヘッダを設定するために、流暢で表現力のあるメカニズムとして機能します。利用するには、アプリケーションの `App\Providers\AppServiceProvider`クラスの`boot`メソッド内で、マクロを定義します。

```php
use Illuminate\Support\Facades\Http;

/**
 * 全アプリケーションサービスの初期起動処理
 */
public function boot(): void
{
    Http::macro('github', function () {
        return Http::withHeaders([
            'X-Example' => 'example',
        ])->baseUrl('https://github.com');
    });
}
```

マクロを設定したら、アプリケーションのどこからでもマクロを呼び出し、保留中のリクエストを指定した設定で作成できます。

```php
$response = Http::github()->get('/');
```

<a name="testing"></a>
## テスト

Laravelの多くのサービスでは、テストを簡単かつ表現豊かに書くための機能を提供しており、HTTPクライアントも例外ではありません。`Http`ファサードの`fake`メソッドにより、リクエストが行われたときにスタブ/ダミーレスポンスを返すようにHTTPクライアントに指示できます。

<a name="faking-responses"></a>
### レスポンスのfake

たとえば、リクエストごとに空の`200`ステータスコードレスポンスを返すようにHTTPクライアントに指示するには、引数なしで`fake`メソッドを呼びだしてください。

    use Illuminate\Support\Facades\Http;

    Http::fake();

    $response = Http::post(/* ... */);

<a name="faking-specific-urls"></a>
#### 特定のURLのfake

もしくは、配列を`fake`メソッドに渡すこともできます。配列のキーは、fakeしたいURLパターンとそれに関連するレスポンスを表す必要があります。`*`文字はワイルドカード文字として使用できます。FakeしないURLに対して行うリクエストは、実際に実行されます。`Http`ファサードの`response`メソッドを使用して、これらのエンドポイントのスタブ/fakeのレスポンスを作成できます。

    Http::fake([
        // GitHubエンドポイントのJSONレスポンスをスタブ
        'github.com/*' => Http::response(['foo' => 'bar'], 200, $headers),

        // Googleエンドポイントの文字列レスポンスをスタブ
        'google.com/*' => Http::response('Hello World', 200, $headers),
    ]);

一致しないすべてのURLをスタブするフォールバックURLパターンを指定する場合は、単一の`*`文字を使用します。

    Http::fake([
        // GitHubエンドポイントのJSONレスポンスをスタブ
        'github.com/*' => Http::response(['foo' => 'bar'], 200, ['Headers']),

        // 他のすべてのエンドポイントの文字列をスタブ
        '*' => Http::response('Hello World', 200, ['Headers']),
    ]);

<a name="faking-response-sequences"></a>
#### fakeレスポンスの順番

場合によっては、単一のURLが特定の順序で一連のfakeレスポンスを返すように指定する必要があります。これは、`Http::sequence`メソッドを使用してレスポンスを作成することで実現できます。

    Http::fake([
        // GitHubエンドポイントの一連のレスポンスをスタブ
        'github.com/*' => Http::sequence()
                                ->push('Hello World', 200)
                                ->push(['foo' => 'bar'], 200)
                                ->pushStatus(404),
    ]);

レスポンスシーケンス内のすべてのレスポンスが消費されると、以降のリクエストに対し、レスポンスシーケンスは例外を投げます。シーケンスが空になったときに返すデフォルトのレスポンスを指定する場合は、`whenEmpty`メソッドを使用します。

    Http::fake([
        // GitHubエンドポイントの一連のレスポンスをスタブ
        'github.com/*' => Http::sequence()
                                ->push('Hello World', 200)
                                ->push(['foo' => 'bar'], 200)
                                ->whenEmpty(Http::response()),
    ]);

一連のレスポンスをfakeしたいが、fakeする必要がある特定のURLパターンを指定する必要がない場合は、`Http::fakeSequence`メソッドを使用します。

    Http::fakeSequence()
            ->push('Hello World', 200)
            ->whenEmpty(Http::response());

<a name="fake-callback"></a>
#### Fakeコールバック

特定のエンドポイントに対して返すレスポンスを決定するために、より複雑なロジックが必要な場合は、`fake`メソッドにクロージャを渡すことができます。このクロージャは`Illuminate\Http\Client\Request`インスタンスを受け取り、レスポンスインスタンスを返す必要があります。クロージャ内で、返すレスポンスのタイプを決定するために必要なロジックを実行できます。

    use Illuminate\Http\Client\Request;

    Http::fake(function (Request $request) {
        return Http::response('Hello World', 200);
    });

<a name="preventing-stray-requests"></a>
### 行き先がないリクエストの防止

HTTPクライアントから送信したすべてのリクエストを個々のテスト、またはテストスイート全体で確実にフェイクにしたい場合は、`preventStrayRequests`メソッドをコールします。このメソッドを呼び出すと、対応するフェイクレスポンスがないリクエストは、実際にHTTPリクエストを行うのではなく、例外を投げるようになります。

    use Illuminate\Support\Facades\Http;

    Http::preventStrayRequests();

    Http::fake([
        'github.com/*' => Http::response('ok'),
    ]);

    // "ok"レスポンスが返される
    Http::get('https://github.com/laravel/framework');

    // 例外が投げられる
    Http::get('https://laravel.com');

<a name="inspecting-requests"></a>
### レスポンスの検査

レスポンスをfakeする場合、アプリケーションが正しいデータまたはヘッダを送信していることを確認するために、クライアントが受信するリクエストを調べたい場合があります。これは、`Http::fake`を呼び出した後に`Http::assertSent`メソッドを呼び出し実現します。

`assertSent`メソッドは、`Illuminate\Http\Client\Request`インスタンスを受け取るクロージャを引数に受け、リクエストがエクスペクテーションに一致するかを示す論理値を返す必要があります。テストに合格するには、指定するエクスペクテーションに一致する少なくとも１つのリクエストが発行される必要があります。

    use Illuminate\Http\Client\Request;
    use Illuminate\Support\Facades\Http;

    Http::fake();

    Http::withHeaders([
        'X-First' => 'foo',
    ])->post('http://example.com/users', [
        'name' => 'Taylor',
        'role' => 'Developer',
    ]);

    Http::assertSent(function (Request $request) {
        return $request->hasHeader('X-First', 'foo') &&
               $request->url() == 'http://example.com/users' &&
               $request['name'] == 'Taylor' &&
               $request['role'] == 'Developer';
    });

必要に応じて、`assertNotSent`メソッドを使用して特定のリクエストが送信されないことを宣言できます。

    use Illuminate\Http\Client\Request;
    use Illuminate\Support\Facades\Http;

    Http::fake();

    Http::post('http://example.com/users', [
        'name' => 'Taylor',
        'role' => 'Developer',
    ]);

    Http::assertNotSent(function (Request $request) {
        return $request->url() === 'http://example.com/posts';
    });

テスト中にいくつのリクエストを「送信」"したかを宣言するため、`assertSentCount`メソッドを使用できます。

    Http::fake();

    Http::assertSentCount(5);

または、`assertNothingSent`メソッドを使用して、テスト中にリクエストが送信されないことを宣言することもできます。

    Http::fake();

    Http::assertNothingSent();

<a name="recording-requests-and-responses"></a>
#### リクエスト／レスポンスの記録

すべてのリクエストと、それに対応するレスポンスを収集するために、`recorded`メソッドが使用できます。`recorded`メソッドは、`Illuminate\Http\Client\Request`と`Illuminate\Http\Client\Response`インスタンスを含む配列のコレクションを返します。

```php
Http::fake([
    'https://laravel.com' => Http::response(status: 500),
    'https://nova.laravel.com/' => Http::response(),
]);

Http::get('https://laravel.com');
Http::get('https://nova.laravel.com/');

$recorded = Http::recorded();

[$request, $response] = $recorded[0];
```

さらに、`recorded`メソッドは、`Illuminate\Http\Client\Request`と`Illuminate\Http\Client\Response`インスタンスを受け取るクロージャを引数に取り、エクスペクテーションに基づいてリクエストとレスポンスのペアをフィルターするために使用できます。

```php
use Illuminate\Http\Client\Request;
use Illuminate\Http\Client\Response;

Http::fake([
    'https://laravel.com' => Http::response(status: 500),
    'https://nova.laravel.com/' => Http::response(),
]);

Http::get('https://laravel.com');
Http::get('https://nova.laravel.com/');

$recorded = Http::recorded(function (Request $request, Response $response) {
    return $request->url() !== 'https://laravel.com' &&
           $response->successful();
});
```

<a name="events"></a>
## イベント

LaravelはHTTPリクエストを送信する過程で、3つのイベントを発行します。`RequestSending`イベントはリクエストが送信される前に発生し、`ResponseReceived`イベントは指定したリクエストに対するレスポンスを受け取った後に発行します。`ConnectionFailed`イベントは、指定したリクエストに対するレスポンスを受信できなかった場合に発行します。

`RequestSending`イベントと`ConnectionFailed`イベントはどちらもpublicの`$request`プロパティを持ち、これを使用して`Illuminate\Http\Client\Request`インスタンスを調べられます。同様に、`ResponseReceived`イベントには`$request`プロパティと`$response`プロパティがあり、これを使用して`Illuminate\Http\Client\Response`インスタンスを調べられます。アプリケーション内で、こうしたイベントの[イベントリスナ](/docs/{{version}}/events)を作成できます。

    use Illuminate\Http\Client\Events\RequestSending;

    class LogRequest
    {
        /**
         * 指定イベントを処理する
         */
        public function handle(RequestSending $event): void
        {
            // $event->request …
        }
    }
