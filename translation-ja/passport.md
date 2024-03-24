# Laravel Passport

- [イントロダクション](#introduction)
    - [Passportか？Sanctumか？？](#passport-or-sanctum)
- [インストール](#installation)
    - [Passportのデプロイ](#deploying-passport)
    - [Passportのアップグレード](#upgrading-passport)
- [設定](#configuration)
    - [クライアントシークレットハッシュ](#client-secret-hashing)
    - [トークン持続時間](#token-lifetimes)
    - [デフォルトモデルのオーバーライド](#overriding-default-models)
    - [ルートのオーバーライド](#overriding-routes)
- [アクセストークンの発行](#issuing-access-tokens)
    - [クライアント管理](#managing-clients)
    - [トークンのリクエスト](#requesting-tokens)
    - [トークンのリフレッシュ](#refreshing-tokens)
    - [トークンの破棄](#revoking-tokens)
    - [トークンの破棄](#purging-tokens)
- [PKCEを使った認可コードグラント](#code-grant-pkce)
    - [クライアント生成](#creating-a-auth-pkce-grant-client)
    - [トークンのリクエスト](#requesting-auth-pkce-grant-tokens)
- [パスワードグラントのトークン](#password-grant-tokens)
    - [パスワードグラントクライアントの作成](#creating-a-password-grant-client)
    - [トークンのリクエスト](#requesting-password-grant-tokens)
    - [全スコープの要求](#requesting-all-scopes)
    - [ユーザープロバイダのカスタマイズ](#customizing-the-user-provider)
    - [ユーザー名フィールドのカスタマイズ](#customizing-the-username-field)
    - [パスワードバリデーションのカスタマイズ](#customizing-the-password-validation)
- [暗黙のグラントトークン](#implicit-grant-tokens)
- [クライアント認証情報グラントトークン](#client-credentials-grant-tokens)
- [パーソナルアクセストークン](#personal-access-tokens)
    - [パーソナルアクセスクライアントの作成](#creating-a-personal-access-client)
    - [パーソナルアクセストークンの管理](#managing-personal-access-tokens)
- [ルート保護](#protecting-routes)
    - [ミドルウェアによる保護](#via-middleware)
    - [アクセストークンの受け渡し](#passing-the-access-token)
- [トークンのスコープ](#token-scopes)
    - [スコープの定義](#defining-scopes)
    - [デフォルトスコープ](#default-scope)
    - [トークンへのスコープ割り付け](#assigning-scopes-to-tokens)
    - [スコープのチェック](#checking-scopes)
- [APIをJavaScriptで利用](#consuming-your-api-with-javascript)
- [イベント](#events)
- [テスト](#testing)

<a name="introduction"></a>
## イントロダクション

[Laravel Passport](https://github.com/laravel/passport)は、Laravelアプリケーションへ完全なOAuth2サーバの実装を数分で提供します。Passportは、Andy MillingtonとSimon Hampがメンテナンスしている[League OAuth2 server](https://github.com/thephpleague/oauth2-server)の上に構築しています。

> [!WARNING]
> このドキュメントは、皆さんがOAuth2に慣れていることを前提にしています。OAuth2について知らなければ、この先を続けて読む前に、一般的な[用語](https://oauth2.thephpleague.com/terminology/)とOAuth2の機能について予習してください。

<a name="passport-or-sanctum"></a>
### Passportか？Sanctumか？？

始める前に、アプリケーションがLaravel Passport、もしくは[Laravel Sanctum](/docs/{{version}}/sanctum)のどちらにより適しているかを検討することをお勧めします。アプリケーションが絶対にOAuth2をサポートする必要がある場合は、Laravel　Passportを使用する必要があります。

しかし、シングルページアプリケーションやモバイルアプリケーションを認証したり、APIトークンを発行したりする場合は、[Laravel Sanctum](/docs/{{version}}/sanctum)を使用する必要があります。Laravel SanctumはOAuth2をサポートしていません。ただし、はるかにシンプルなAPI認証開発エクスペリエンスを提供します。

<a name="installation"></a>
## インストール

Laravel Passportは、`install:api` Artisanコマンドでインストールします。

```shell
php artisan install:api --passport
```

このコマンドは、アプリケーションがOAuth2クライアントとアクセストークンを格納するために必要なテーブルを作成するために必要なデータベースのマイグレーションをリソース公開し、実行します。このコマンドは、セキュアなアクセストークンを生成するために必要な暗号化キーも作成します。

さらに、このコマンドは、Passport `Client`モデルの主キー値として、自動インクリメントの整数の代わりに、UUIDを使用するかを尋ねます。

`install:api`コマンドを実行した後に、`App\Models\User`モデルへ`Laravel\Passport\HasApiTokens`トレイトを追加してください。このトレイトは、認証済みユーザのトークンとスコープを検査できる、いくつかのヘルパーメソッドをモデルに提供します。

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Factories\HasFactory;
    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Laravel\Passport\HasApiTokens;

    class User extends Authenticatable
    {
        use HasApiTokens, HasFactory, Notifiable;
    }

最後に、アプリケーションの`config/auth.php`設定ファイルで、`api`認証ガードを定義して、`driver`オプションを`passport`に設定します。これにより、API リクエストを認証する際に Passportの`TokenGuard`を使用するようアプリケーションに指示します。

    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],

        'api' => [
            'driver' => 'passport',
            'provider' => 'users',
        ],
    ],

<a name="deploying-passport"></a>
### Passportのデプロイ

Passportをアプリケーションのサーバへ初めてデプロイするときは、`passport:keys`コマンドを実行する必要があります。このコマンドは、アクセストークンを生成するためにPassportが必要とする暗号化キーを生成します。生成されたキーは通常、ソース管理しません。

```shell
php artisan passport:keys
```

必要であれば、Passportのキーをロードするパスを定義することもできます。これには`Passport::loadKeysFrom`メソッドを使用します。通常、このメソッドはアプリケーションの`App\Providers\AppServiceProvider`クラスの `boot`メソッドから呼び出します。

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Passport::loadKeysFrom(__DIR__.'/../secrets/oauth');
    }

<a name="loading-keys-from-the-environment"></a>
#### 環境からのキーのロード

または、`vendor:publish` Artisanコマンドを使用してPassportの設定ファイルをリソース公開することもできます。

```shell
php artisan vendor:publish --tag=passport-config
```

設定ファイルをリソース公開した後、環境変数として定義することにより、アプリケーションの暗号化キーをロードできます。

```ini
PASSPORT_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
<プライベートキーをここに記述>
-----END RSA PRIVATE KEY-----"

PASSPORT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----
<パブリックキーをここに記述>
-----END PUBLIC KEY-----"
```

<a name="upgrading-passport"></a>
### Passportのアップグレード

Passportの新しいメジャーバージョンにアップグレードするときは、[アップグレードガイド](https://github.com/laravel/passport/blob/master/UPGRADE.md)を注意深く確認することが重要です。

<a name="configuration"></a>
## 設定

<a name="client-secret-hashing"></a>
### クライアントシークレットハッシュ

クライアントの秘密をデータベースに格納するときにハッシュ化したい場合は、`App\Providers\AppServiceProvider`クラスの`boot`メソッドで、`Passport::hashClientSecrets`メソッドを呼び出す必要があります。

    use Laravel\Passport\Passport;

    Passport::hashClientSecrets();

有効にすると、すべてのクライアントシークレットは、作成した直後のみユーザーへ表示されます。平文テキストのクライアントシークレット値がデータベースに保存されることはないため、シークレットの値が失われた場合にその値を回復することはできません。

<a name="token-lifetimes"></a>
### トークン持続時間

Passportはデフォルトで、１年後に失効する長寿命のアクセストークンを発行します。トークンの有効期限を長く／短く設定したい場合は、`tokensExpireIn`、`refreshTokensExpireIn`、`personalAccessTokensExpireIn`メソッを使用してください。これらのメソッドは、アプリケーションの `App\Providers\AppServiceProvider`クラスの`boot`メソッドから呼び出す必要があります。

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Passport::tokensExpireIn(now()->addDays(15));
        Passport::refreshTokensExpireIn(now()->addDays(30));
        Passport::personalAccessTokensExpireIn(now()->addMonths(6));
    }

> [!WARNING]
> Passportのデータベーステーブルの`expires_at`カラムは読み取り専用であり、表示のみを目的としています。トークンを発行するとき、Passportは署名および暗号化されたトークン内に有効期限情報を保存します。トークンを無効にする必要がある場合は、[取り消す](#revoking-tokens)必要があります。

<a name="overriding-default-models"></a>
### デフォルトモデルのオーバーライド

独自のモデルを定義し、対応するPassportモデルを拡張することにより、Passportにより内部的に使用されるモデルを自由に拡張できます。

    use Laravel\Passport\Client as PassportClient;

    class Client extends PassportClient
    {
        // ...
    }

モデルを定義した後、`Laravel\Passport\Passport`クラスを使い、カスタムモデルを使うようにPassportへ指示できます。通常、アプリケーションの`App\Providers\AppServiceProvider`クラスの`boot`メソッドで、カスタムモデルについてPassportへ通知します。

    use App\Models\Passport\AuthCode;
    use App\Models\Passport\Client;
    use App\Models\Passport\PersonalAccessClient;
    use App\Models\Passport\RefreshToken;
    use App\Models\Passport\Token;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Passport::useTokenModel(Token::class);
        Passport::useRefreshTokenModel(RefreshToken::class);
        Passport::useAuthCodeModel(AuthCode::class);
        Passport::useClientModel(Client::class);
        Passport::usePersonalAccessClientModel(PersonalAccessClient::class);
    }

<a name="overriding-routes"></a>
### ルートのオーバーライド

Passportが定義するルートをカスタマイズしたい場合もあるでしょう。そのためには、まずアプリケーションの`AppServiceProvider`の`register`メソッドへ、`Passport::ignoreRoutes`を追加し、Passportが登録したルートを無視する必要があります。

    use Laravel\Passport\Passport;

    /**
     * 全アプリケーションサービスの登録
     */
    public function register(): void
    {
        Passport::ignoreRoutes();
    }

そして、Passport自身の[ルートファイル](https://github.com/laravel/passport/blob/11.x/routes/web.php)で定義しているルートをアプリケーションの`routes/web.php`ファイルへコピーして、好みに合わせ変更してください。

    Route::group([
        'as' => 'passport.',
        'prefix' => config('passport.path', 'oauth'),
        'namespace' => '\Laravel\Passport\Http\Controllers',
    ], function () {
        // Passportのルート…
    });CheckClientCredentials

<a name="issuing-access-tokens"></a>
## アクセストークンの発行

認証コードを介してOAuth2を使用することは、OAuth2を扱う時にほとんどの開発者が精通している方法です。認証コードを使用する場合、クライアントアプリケーションはユーザーをサーバにリダイレクトし、そこでユーザーはクライアントへアクセストークンを発行するリクエストを承認または拒否します。

<a name="managing-clients"></a>
### クライアント管理

あなたのアプリケーションのAPIと連携する必要のある、アプリケーションを構築しようとしている開発者たちは、最初に「クライアント」を作成することにより、彼らのアプリケーションを登録しなくてはなりません。通常、アプリケーションの名前と、許可のリクエストをユーザーが承認した後に、アプリケーションがリダイレクトされるURLにより、登録情報は構成されます。

<a name="the-passportclient-command"></a>
#### `passport:client`コマンド

クライアントを作成する一番簡単な方法は、`passport:client` Artisanコマンドを使うことです。このコマンドは、OAuth2の機能をテストするため、皆さん自身のクライアントを作成する場合に使用できます。`client`コマンドを実行すると、Passportはクライアントに関する情報の入力を促し、クライアントIDとシークレットを表示します。

```shell
php artisan passport:client
```

**リダイレクトURL**

クライアントに複数のリダイレクトURLを許可する場合は、`passport:client`コマンドでURLの入力を求められたときに、コンマ区切りのリストを使用して指定してください。コンマを含むURLは、URLエンコードする必要があります。

```shell
http://example.com/callback,http://examplefoo.com/callback
```

<a name="clients-json-api"></a>
#### JSON API

アプリケーションのユーザーは`client`コマンドを利用できないため、Passportはクライアントの作成に使用できるJSON APIを提供します。これにより、クライアントを作成、更新、および削除するためにコントローラを手作業でコーディングする手間が省けます。

しかし、ユーザーにクライアントを管理してもらうダッシュボードを提供するために、PassportのJSON APIと皆さんのフロントエンドを結合する必要があります。以降から、クライアントを管理するためのAPIエンドポイントをすべて説明します。エンドポイントへのHTTPリクエスト作成をデモンストレートするため利便性を考慮し、[Axios](https://github.com/mzabriskie/axios)を使用していきましょう。

JSON APIは`web`と`auth`ミドルウェアにより保護されています。そのため、みなさん自身のアプリケーションからのみ呼び出せます。外部ソースから呼び出すことはできません。

<a name="get-oauthclients"></a>
#### `GET /oauth/clients`

このルートは認証されたユーザーの全クライアントを返します。ユーザーのクライアントの全リストは、主にクライアントを編集、削除する場合に役立ちます。

```js
axios.get('/oauth/clients')
    .then(response => {
        console.log(response.data);
    });CheckClientCredentials
```

<a name="post-oauthclients"></a>
#### `POST /oauth/clients`

このルートは新クライアントを作成するために使用します。これには２つのデータが必要です。クライアントの名前（`name`）と、リダイレクト（`redirect`）のURLです。`redirect`のURLは許可のリクエストが承認されるか、拒否された後のユーザーのリダイレクト先です。

クライアントを作成すると、クライアントIDとクライアントシークレットが発行されます。これらの値はあなたのアプリケーションへリクエストし、アクセストークンを取得する時に使用されます。クライアント作成ルートは、新しいクライアントインスタンスを返します。

```js
const data = {
    name: 'Client Name',
    redirect: 'http://example.com/callback'
};

axios.post('/oauth/clients', data)
    .then(response => {
        console.log(response.data);
    })
    .catch (response => {
        // レスポンスのエラーをリストする処理…
    });CheckClientCredentials
```

<a name="put-oauthclientsclient-id"></a>
#### `PUT /oauth/clients/{client-id}`

このルートはクライアントを更新するために使用します。それには２つのデータが必要です。クライアントの`name`と`redirect`のURLです。`redirect`のURLは許可のリクエストが承認されるか、拒否され後のユーザーのリダイレクト先です。このルートは更新されたクライアントインスタンスを返します。

```js
const data = {
    name: 'New Client Name',
    redirect: 'http://example.com/callback'
};

axios.put('/oauth/clients/' + clientId, data)
    .then(response => {
        console.log(response.data);
    })
    .catch (response => {
        // レスポンスのエラーをリストする処理…
    });CheckClientCredentials
```

<a name="delete-oauthclientsclient-id"></a>
#### `DELETE /oauth/clients/{client-id}`

このルートはクライアントを削除するために使用します。

```js
axios.delete('/oauth/clients/' + clientId)
    .then(response => {
        // ...
    });CheckClientCredentials
```

<a name="requesting-tokens"></a>
### トークンのリクエスト

<a name="requesting-tokens-redirecting-for-authorization"></a>
#### 許可のリダイレクト

クライアントが作成されると、開発者はクライアントIDとシークレットを使用し、あなたのアプリケーションへ許可コードとアクセストークンをリクエストするでしょう。まず、API利用側アプリケーションは以下のように、あなたのアプリケーションの`/oauth/authorize`ルートへのリダイレクトリクエストを作成する必要があります。

    use Illuminate\Http\Request;
    use Illuminate\Support\Str;

    Route::get('/redirect', function (Request $request) {
        $request->session()->put('state', $state = Str::random(40));

        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'response_type' => 'code',
            'scope' => '',
            'state' => $state,
            // 'prompt' => '', // "none", "consent", or "login"
        ]);

        return redirect('http://passport-app.test/oauth/authorize?'.$query);
    });CheckClientCredentials

`prompt`パラメータは、Passportアプリケーションの認証動作を指定するために使用します。

`prompt`の値が`none`の場合、ユーザーがPassportアプリケーションで認証されていないとき、Passportは認証エラーを常時スローします。値が`consent`の場合、すべてのスコープが事前に利用者側アプリケーションへ許可されていても、Passportは常に承認承認スクリーンを表示します。値が`login`である場合、Passportアプリケーションは、ユーザーが既にセッションを持っていても、アプリケーションへ再ログインするように常に促します。

`prompt`値を指定しない場合、要求されたスコープに対する消費者側アプリケーションへのアクセスをそのユーザーへ以前に許可していない場合のみ、認可のためのプロンプトを表示します。

> [!NOTE]
> `/oauth/authorize`ルートは、すでにPassportが定義づけていることを覚えておいてください。このルートを自分で定義する必要はありません。

<a name="approving-the-request"></a>
#### リクエストの承認

認証リクエストを受け取ると、Passportは`prompt`パラメータが指定されている場合は、その値に基づいて自動的に応答し、認証リクエストを承認または拒否するためのテンプレートをユーザーに表示します。ユーザーがリクエストを承認した場合、消費者側アプリケーションにより指定された、`redirect_uri`へリダイレクトします。この`redirect_uri`は、クライアントが作成されたときに指定した、`redirect` URLと一致しなければなりません。

承認画面をカスタマイズする場合は、`vendor:publish` Artisanコマンドを使用してPassportのビューをリソース公開します。公開したビューは、`resources/views/vendor/passport`ディレクトリに配置されます。

```shell
php artisan vendor:publish --tag=passport-views
```

ファーストパーティクライアントを認証するときなど、認証プロンプトを飛ばしたいことも起きるでしょう。このような場合は、[`Client`モデルを拡張し](#overriding-default-models)、`skipsAuthorization`メソッドを定義すれば実現できます。`skipsAuthorization`が、`true`を返したら、クライアントは承認され、ユーザーはすぐに`redirect_uri`へリダイレクトされます。ただし、消費者側アプリケーションが承認のためのリダイレクト時に、明示的に`prompt`パラメータを設定した場合はこの限りではありません。

    <?php

    namespace App\Models\Passport;

    use Laravel\Passport\Client as BaseClient;

    class Client extends BaseClient
    {
        /**
         * クライアントが認可プロンプトを飛ばすべきか判定
         */
        public function skipsAuthorization(): bool
        {
            return $this->firstParty();
        }
    }

<a name="requesting-tokens-converting-authorization-codes-to-access-tokens"></a>
#### 許可コードからアクセストークンへの変換

ユーザーが承認リクエストを承認すると、ユーザーは利用側アプリケーションにリダイレクトされます。利用側はまず、リダイレクトの前に保存した値に対して`state`パラメーターを確認する必要があります。状態パラメータが一致する場合、利用側はアプリケーションへ`POST`リクエストを発行してアクセストークンをリクエストする必要があります。リクエストには、ユーザーが認証リクエストを承認したときにアプリケーションが発行した認証コードを含める必要があります。

    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Http;

    Route::get('/callback', function (Request $request) {
        $state = $request->session()->pull('state');

        throw_unless(
            strlen($state) > 0 && $state === $request->state,
            InvalidArgumentException::class,
            'Invalid state value.'
        );

        $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
            'grant_type' => 'authorization_code',
            'client_id' => 'client-id',
            'client_secret' => 'client-secret',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'code' => $request->code,
        ]);

        return $response->json();
    });CheckClientCredentials

この`/oauth/token`ルートは、`access_token`、`refresh_token`、`expires_in`属性を含むJSONレスポンスを返します。`expires_in`属性は、アクセストークンが無効になるまでの秒数を含んでいます。

> [!NOTE]
> `/oauth/authorize`ルートと同様に、`/oauth/token`ルートはPassportによって定義されます。このルートを手作業で定義する必要はありません。

<a name="tokens-json-api"></a>
#### JSON API

Passportには、承認済みアクセストークンを管理するためのJSON APIも含んでいます。これを独自のフロントエンドと組み合わせ、アクセストークンを管理するダッシュボードをユーザーへ提供できます。便宜上、[Axios](https://github.com/mzabriskie/axios)をエンドポイントへのHTTPリクエストを生成するデモンストレーションのため使用しています。JSON APIは`web`と`auth`ミドルウェアにより保護されているため、自身のアプリケーションからのみ呼び出しできます。

<a name="get-oauthtokens"></a>
#### `GET /oauth/tokens`

このルートは、認証されたユーザーが作成した、承認済みアクセストークンをすべて返します。これは主に取り消すトークンを選んでもらうため、ユーザーの全トークンを一覧リスト表示するのに便利です。

```js
axios.get('/oauth/tokens')
    .then(response => {
        console.log(response.data);
    });CheckClientCredentials
```

<a name="delete-oauthtokenstoken-id"></a>
#### `DELETE /oauth/tokens/{token-id}`

このルートは、認証済みアクセストークンと関連するリフレッシュトークンを取り消すために使います。

```js
axios.delete('/oauth/tokens/' + tokenId);
```

<a name="refreshing-tokens"></a>
### トークンのリフレッシュ

アプリケーションが短期間のアクセストークンを発行する場合、ユーザーはアクセストークンが発行されたときに提供された更新トークンを利用して、アクセストークンを更新する必要があります。

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'refresh_token',
        'refresh_token' => 'the-refresh-token',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'scope' => '',
    ]);

    return $response->json();

この`/oauth/token`ルートは、`access_token`、`refresh_token`、`expires_in`属性を含むJSONレスポンスを返します。`expires_in`属性は、アクセストークンが無効になるまでの秒数を含んでいます。

<a name="revoking-tokens"></a>
### トークンの取り消し

`Laravel\Passport\TokenRepository`の`revokeAccessToken`メソッドを使用してトークンを取り消すことができます。`Laravel\Passport\RefreshTokenRepository`の`revokeRefreshTokensByAccessTokenId`メソッドを使用して、トークンの更新トークンを取り消すことができます。これらのクラスは、Laravelの[サービスコンテナ](/docs/{{version}}/container)を使用して解決できます。

    use Laravel\Passport\TokenRepository;
    use Laravel\Passport\RefreshTokenRepository;

    $tokenRepository = app(TokenRepository::class);
    $refreshTokenRepository = app(RefreshTokenRepository::class);

    // アクセストークンの取り消し
    $tokenRepository->revokeAccessToken($tokenId);

    // そのトークンのリフレッシュトークンを全て取り消し
    $refreshTokenRepository->revokeRefreshTokensByAccessTokenId($tokenId);

<a name="purging-tokens"></a>
### トークンの破棄

トークンが取り消されたり期限切れになったりした場合は、データベースからトークンを削除することを推奨します。Passportに含まれている`passport:purge` Artisanコマンドでこれを実行できます。

```shell
# 取り消されたか期限が切れた、トークンと認証コードの削除
php artisan passport:purge

# 期限切れから６時間以上経っているトークンのみ削除
php artisan passport:purge --hours=6

# 取り消されたトークンと認証コードのみを削除
php artisan passport:purge --revoked

# 期限切れのトークンと認証コードの削除
php artisan passport:purge --expired
```

アプリケーションの`routes/console.php`ファイルで[ジョブのスケジュール](/docs/{{version}}/scheduling)を設定して、このスケジュールに従い自動的にトークンを削除することもできます。

    use Laravel\Support\Facades\Schedule;

    Schedule::command('passport:purge')->hourly();

<a name="code-grant-pkce"></a>
## PKCEを使った認可コードグラント

"Proof Key for Code Exchange" (PKCE)を使用する認可コードグラントは、シングルページアプリケーションやネイティブアプリケーションが、APIへアクセスするための安全な認証方法です。このグラントはクライアントの秘密コードを十分な機密を保ち保存できないか、もしくは認可コード横取り攻撃の危険を軽減する必要がある場合に、必ず使用すべきです。アクセストークンのために認可コードを交換するときに、クライアントの秘密コードを「コードベリファイヤ(code verifier)」と「コードチャレンジ(code challenge)」のコンピネーションに置き換えます。

<a name="creating-a-auth-pkce-grant-client"></a>
### クライアント生成

アプリケーションがPKCEでの認証コードグラントを介してトークンを発行する前に、PKCE対応のクライアントを作成する必要があります。これは、`passport:client` Artisanコマンドと`--public`オプションを使用して行えます。

```shell
php artisan passport:client --public
```

<a name="requesting-auth-pkce-grant-tokens"></a>
### トークンのリクエスト

<a name="code-verifier-code-challenge"></a>
#### コードベリファイヤとコードチャレンジ

この認可グラントではクライアント秘密コードが提供されないため、開発者はトークンを要求するためにコードベリファイヤとコードチャレンジのコンビネーションを生成する必要があります。

コードベリファイアは、[RFC 7636 仕様](https://tools.ietf.org/html/rfc7636)で定義されているように、文字、数字、`"-"`、`"."`、`"_"`、`"~"`文字を含む４３文字から１２８文字のランダムな文字列でなければなりません。

コードチャレンジはURL／ファイルネームセーフな文字をBase64エンコードしたものである必要があります。文字列終端の`'='`文字を削除し、ラインブレイクやホワイトスペースを含まず、その他はそのままにします。

    $encoded = base64_encode(hash('sha256', $code_verifier, true));

    $codeChallenge = strtr(rtrim($encoded, '='), '+/', '-_');

<a name="code-grant-pkce-redirecting-for-authorization"></a>
#### 許可のリダイレクト

クライアントが生成できたら、アプリケーションから認可コードとアクセストークンをリクエストするために、クライアントIDと生成したコードベリファイヤ、コードチャレンジを使用します。最初に、認可要求側のアプリケーションは、あなたのアプリケーションの`/oauth/authorize`ルートへのリダイレクトリクエストを生成する必要があります。

    use Illuminate\Http\Request;
    use Illuminate\Support\Str;

    Route::get('/redirect', function (Request $request) {
        $request->session()->put('state', $state = Str::random(40));

        $request->session()->put(
            'code_verifier', $code_verifier = Str::random(128)
        );

        $codeChallenge = strtr(rtrim(
            base64_encode(hash('sha256', $code_verifier, true))
        , '='), '+/', '-_');

        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'response_type' => 'code',
            'scope' => '',
            'state' => $state,
            'code_challenge' => $codeChallenge,
            'code_challenge_method' => 'S256',
            // 'prompt' => '', // "none", "consent", or "login"
        ]);

        return redirect('http://passport-app.test/oauth/authorize?'.$query);
    });CheckClientCredentials

<a name="code-grant-pkce-converting-authorization-codes-to-access-tokens"></a>
#### 許可コードからアクセストークンへの変換

ユーザーが認可リクエストを承認すると、認可要求側のアプリケーションへリダイレクで戻されます。認可要求側では認可コードグラントの規約に従い、リダイレクトの前に保存しておいた値と、`state`パラメータを検証する必要があります。

stateパラメータが一致したら、要求側はアクセストークンをリクエストするために、あなたのアプリケーションへ`POST`リクエストを発行する必要があります。そのリクエストは最初に生成したコードベリファイヤと同時に、ユーザーが認可リクエストを承認したときにあなたのアプリケーションが発行した認可コードを持っている必要があります。

    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Http;

    Route::get('/callback', function (Request $request) {
        $state = $request->session()->pull('state');

        $codeVerifier = $request->session()->pull('code_verifier');

        throw_unless(
            strlen($state) > 0 && $state === $request->state,
            InvalidArgumentException::class
        );

        $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
            'grant_type' => 'authorization_code',
            'client_id' => 'client-id',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'code_verifier' => $codeVerifier,
            'code' => $request->code,
        ]);

        return $response->json();
    });CheckClientCredentials

<a name="password-grant-tokens"></a>
## パスワードグラントのトークン

> [!WARNING]
> パスワードグラントトークンの使用は、現在推奨していません。代わりに、[OAuth2サーバが現在推奨しているグラントタイプ](https://oauth2.thephpleague.com/authorization-server/which-grant/) を選択する必要があります。

OAuth2パスワードグラントにより、モバイルアプリケーションなどの他のファーストパーティクライアントは、電子メールアドレス／ユーザー名とパスワードを使用してアクセストークンを取得できます。これにより、ユーザーがOAuth2認証コードのリダイレクトフロー全体を実行しなくても、ファーストパーティクライアントにアクセストークンを安全に発行できます。

パスワードグラントを有効にするには、アプリケーションの`App\Providers\AppServiceProvider`クラスの`boot`メソッドで、`enablePasswordGrant`メソッドを呼び出してください。

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Passport::enablePasswordGrant();
    }

<a name="creating-a-password-grant-client"></a>
### パスワードグラントクライアントの作成

アプリケーションがパスワードグラントを介してトークンを発行する前に、パスワードグラントクライアントを作成する必要があります。これは、`--password`オプションを指定した`passport:client` Artisanコマンドを使用して行えます。**すでに`passport:install`コマンドを実行している場合は、次のコマンドを実行する必要はありません:**

```shell
php artisan passport:client --password
```

<a name="requesting-password-grant-tokens"></a>
### トークンのリクエスト

パスワードグラントクライアントを作成したら、ユーザーのメールアドレスとパスワードを指定し、`/oauth/token`ルートへ`POST`リクエストを発行することで、アクセストークンをリクエストできます。このルートは、Passportが登録しているため、自分で定義する必要がないことを覚えておきましょう。リクエストに成功すると、サーバから`access_token`と`refresh_token`のJSONレスポンスを受け取ります。

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'password',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'username' => 'taylor@laravel.com',
        'password' => 'my-password',
        'scope' => '',
    ]);

    return $response->json();

> [!NOTE]
> アクセストークンはデフォルトで、長期間有効であることを記憶しておきましょう。ただし、必要であれば自由に、[アクセストークンの最長持続時間を設定](#configuration)できます。

<a name="requesting-all-scopes"></a>
### 全スコープの要求

パスワードグラント、またはクライアント認証情報グラントを使用時は、あなたのアプリケーションでサポートする全スコープを許可するトークンを発行したいと考えるかと思います。`*`スコープをリクエストすれば可能です。`*`スコープをリクエストすると、そのトークンインスタンスの`can`メソッドは、いつも`true`を返します。このスコープは`password`か`client_credentials`グラントを使って発行されたトークのみに割り付けるのが良いでしょう。

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'password',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'username' => 'taylor@laravel.com',
        'password' => 'my-password',
        'scope' => '*',
    ]);

<a name="customizing-the-user-provider"></a>
### ユーザープロバイダのカスタマイズ

アプリケーションが複数の[認証ユーザープロバイダ](/docs/{{version}}/authentication#introduction)を使用している場合は、`artisan passport:client --password`コマンドを介してクライアントを作成する時に、`--provider`オプションを指定することで、パスワードグラントクライアントが使用するユーザープロバイダを指定できます。指定するプロバイダ名は、アプリケーションの`config/auth.php`設定ファイルで定義している有効なプロバイダと一致する必要があります。次に、[ミドルウェアを使用してルートを保護](#via-middleware)して、ガードの指定するプロバイダのユーザーのみが許可されるようにすることができます。

<a name="customizing-the-username-field"></a>
### Customizing the Username Field

パスワードグラントを使用して認証する場合、Passportは認証可能なモデルの`email`属性を「ユーザー名」として使用します。ただし、モデルで`findForPassport`メソッドを定義することにより、この動作をカスタマイズできます。

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Laravel\Passport\HasApiTokens;

    class User extends Authenticatable
    {
        use HasApiTokens, Notifiable;

        /**
         * 指定されたユーザー名のユーザーインスタンスを見つける
         */
        public function findForPassport(string $username): User
        {
            return $this->where('username', $username)->first();
        }
    }

<a name="customizing-the-password-validation"></a>
### パスワードバリデーションのカスタマイズ

パスワードガードを使用して認証している場合、Passportは指定されたパスワードを確認するためにモデルの`password`属性を使用します。もし、`password`属性を持っていないか、パスワードのバリデーションロジックをカスタマイズしたい場合は、モデルの`validateForPassportPasswordGrant`メソッドを定義してください。

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Support\Facades\Hash;
    use Laravel\Passport\HasApiTokens;

    class User extends Authenticatable
    {
        use HasApiTokens, Notifiable;

        /**
         * Passportパスワードグラントのために、ユーザーのパスワードをバリデート
         */
        public function validateForPassportPasswordGrant(string $password): bool
        {
            return Hash::check($password, $this->password);
        }
    }

<a name="implicit-grant-tokens"></a>
## 暗黙のグラントトークン

> [!WARNING]
> 暗黙的のグラント・トークンの使用は、現在推奨していません。代わりに、[OAuth2サーバが現在推奨しているグラントタイプ](https://oauth2.thephpleague.com/authorization-server/which-grant/) を選択する必要があります。

暗黙のグラントは、認証コードグラントと似ていますが、トークンは認証コードを交換せずにクライアントへ返します。このグラントは、JavaScriptやモバイルアプリケーションで、クライアントの認証情報を安全に保存できない場合によく使われます。このグラントを有効にするには、アプリケーションの`App\Providers\AppServiceProvider`クラスの`boot`メソッドで`enableImplicitGrant`メソッドを呼び出してください。

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Passport::enableImplicitGrant();
    }

暗黙グラントが有効になると、開発者はクライアントIDを使用してアプリケーションにアクセストークンをリクエストできます。利用側アプリケーションは、次のようにアプリケーションの`/oauth/authorize`ルートにリダイレクトリクエストを行う必要があります。

    use Illuminate\Http\Request;

    Route::get('/redirect', function (Request $request) {
        $request->session()->put('state', $state = Str::random(40));

        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'response_type' => 'token',
            'scope' => '',
            'state' => $state,
            // 'prompt' => '', // "none", "consent", or "login"
        ]);

        return redirect('http://passport-app.test/oauth/authorize?'.$query);
    });CheckClientCredentials

> [!NOTE]
> `/oauth/authorize`ルートは、すでにPassportが定義づけていることを覚えておいてください。このルートを自分で定義する必要はありません。

<a name="client-credentials-grant-tokens"></a>
## クライアント認証情報グラントトークン

クライアント認証情報グラントはマシンーマシン間の認証に最適です。たとえば、APIによりメンテナンスタスクを実行する、定期実行ジョブに使用できます。

アプリケーションがクライアント利用資格情報グラントを介してトークンを発行する前に、クライアント利用資格情報グラントクライアントを作成する必要があります。これは、`passport:client` Artisanコマンドの`--client`オプションを使用して行うことができます。

```shell
php artisan passport:client --client
```

次に、このグラントタイプを使用するには、`CheckClientCredentials`ミドルウェアのミドルウェアエイリアスを登録します。ミドルウェアエイリアスはアプリケーションの`bootstrap/app.php`ファイルで定義できます。

    use Laravel\Passport\Http\Middleware\CheckClientCredentials;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'client' => CheckClientCredentials::class
        ]);
    })

それから、ルートへこのミドルウェアを指定します。

    Route::get('/orders', function (Request $request) {
        ...
    })->middleware('client');

ルートへのアクセスを特定のスコープに制限するには、`client`ミドルウェアをルートに接続するときに、必要なスコープのコンマ区切りのリストを指定できます。

    Route::get('/orders', function (Request $request) {
        ...
    })->middleware('client:check-status,your-scope');

<a name="retrieving-tokens"></a>
### トークンの取得

このグラントタイプを使用してトークンを取得するには、`oauth/token`エンドポイントにリクエストを送信します。

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'client_credentials',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'scope' => 'your-scope',
    ]);

    return $response->json()['access_token'];

<a name="personal-access-tokens"></a>
## パーソナルアクセストークン

ときどき、あなたのユーザーが典型的なコードリダイレクションフローに従うのではなく、自分たち自身でアクセストークンを発行したがることもあるでしょう。あなたのアプリケーションのUIを通じて、ユーザー自身のトークンを発行を許可することにより、あなたのAPIをユーザーに経験してもらう事ができますし、全般的なアクセストークン発行するシンプルなアプローチとしても役立つでしょう。

> [!NOTE]
> あなたのアプリケーションが主にPassportを使用して個人アクセストークンを発行している場合、APIアクセストークンを発行するためのLaravelの軽量なファーストパーティーライブラリ、[Laravel Sanctum](/docs/{{version}}/sanctum)の使用を検討してください。

<a name="creating-a-personal-access-client"></a>
### パーソナルアクセスクライアントの作成

アプリケーションがパーソナルアクセストークンを発行する前に、パーソナルアクセスクライアントを作成する必要があります。これを行うには、`--personal`オプションを指定して`passport:client` Artisanコマンドを実行します。すでに`passport:install`コマンドを実行している場合は、次のコマンドを実行する必要はありません。

```shell
php artisan passport:client --personal
```

パーソナルアクセスクライアントを制作したら、クライアントIDと平文シークレット値をアプリケーションの`.env`ファイルに設定してください。

```ini
PASSPORT_PERSONAL_ACCESS_CLIENT_ID="client-id-value"
PASSPORT_PERSONAL_ACCESS_CLIENT_SECRET="unhashed-client-secret-value"
```

<a name="managing-personal-access-tokens"></a>
### パーソナルアクセストークンの管理

パーソナルアクセスクライアントを作成したら、`App\Models\User`モデルインスタンスで`createToken`メソッドを使用して特定のユーザーにトークンを発行できます。`createToken`メソッドは、最初の引数にトークン名、２番目の引数にオプションの[スコープ](#token-scopes)の配列を取ります。

    use App\Models\User;

    $user = User::find(1);

    // スコープ無しのトークンを作成する
    $token = $user->createToken('Token Name')->accessToken;

    // スコープ付きのトークンを作成する
    $token = $user->createToken('My Token', ['place-orders'])->accessToken;

<a name="personal-access-tokens-json-api"></a>
#### JSON API

Passportにはパーソナルアクセストークンを管理するためのJSON APIも含まれています。ユーザーにパーソナルアクセストークンを管理してもらうダッシュボードを提供するため、APIと皆さんのフロントエンドを結びつける必要があるでしょう。以降から、パーソナルアクセストークンを管理するためのAPIエンドポイントをすべて説明します。利便性を考慮し、エンドポイントへのHTTPリクエスト作成をデモンストレートするために、[Axios](https://github.com/mzabriskie/axios)を使用していきましょう。

JSON APIは`web`と`auth`ミドルウェアにより保護されています。そのため、みなさん自身のアプリケーションからのみ呼び出せます。外部ソースから呼び出すことはできません。

<a name="get-oauthscopes"></a>
#### `GET /oauth/scopes`

このルートはあなたのアプリケーションで定義した、全[スコープ](#token-scopes)を返します。このルートを使い、ユーザーがパーソナルアクセストークンに割り付けたスコープをリストできます。

```js
axios.get('/oauth/scopes')
    .then(response => {
        console.log(response.data);
    });CheckClientCredentials
```

<a name="get-oauthpersonal-access-tokens"></a>
#### `GET /oauth/personal-access-tokens`

このルートは認証中のユーザーが作成したパーソナルアクセストークンをすべて返します。ユーザーがトークンの編集や取り消しを行うため、全トークンをリストするために主に使われます。

```js
axios.get('/oauth/personal-access-tokens')
    .then(response => {
        console.log(response.data);
    });CheckClientCredentials
```

<a name="post-oauthpersonal-access-tokens"></a>
#### `POST /oauth/personal-access-tokens`

このルートは新しいパーソナルアクセストークンを作成します。トークンの名前(`name`)と、トークンに割り付けるスコープ(`scope`)の、２つのデータが必要です。

```js
const data = {
    name: 'Token Name',
    scopes: []
};

axios.post('/oauth/personal-access-tokens', data)
    .then(response => {
        console.log(response.data.accessToken);
    })
    .catch (response => {
        // レスポンスのエラーをリストする処理…
    });CheckClientCredentials
```

<a name="delete-oauthpersonal-access-tokenstoken-id"></a>
#### `DELETE /oauth/personal-access-tokens/{token-id}`

このルートはパーソナルアクセストークンを取り消すために使用します。

```js
axios.delete('/oauth/personal-access-tokens/' + tokenId);
```

<a name="protecting-routes"></a>
## ルート保護

<a name="via-middleware"></a>
### ミドルウェアによる保護

Passportは、受信リクエストのアクセストークンを検証する[認証グラント](/docs/{{version}}/authentication#adding-custom-guards)を用意しています。`passport`ドライバを使用するように`api`ガードを設定したら、有効なアクセストークンを必要とするルートで`auth:api`ミドルウェアを指定するだけで済みます。

    Route::get('/user', function () {
        // ...
    })->middleware('auth:api');

> [!WARNING]
> [クライアント認証情報グラント](#client-credentials-grant-tokens)を使用している場合、`auth:api`ミドルウェアではなく、[`client`ミドルウェア](#client-credentials-grant-tokens)でルートを保護する必要があります。

<a name="multiple-authentication-guards"></a>
#### 複数認証ガード

アプリケーションの認証でたぶんまったく異なるEloquentモデルを使用する、別々のタイプのユーザーを認証する場合、それぞれのユーザープロバイダタイプごとにガード設定を定義する必用があるでしょう。これにより特定ユーザープロバイダ向けのリクエストを保護できます。例として`config/auth.php`設定ファイルで以下のようなガード設定を行っているとしましょう。

    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],

    'api-customers' => [
        'driver' => 'passport',
        'provider' => 'customers',
    ],

以下のルートは受信リクエストを認証するため`customers`ユーザープロバイダを使用する`api-customers`ガードを使用します。

    Route::get('/customer', function () {
        // ...
    })->middleware('auth:api-customers');

> [!NOTE]
> Passportを使用する複数ユーザープロバイダ利用の詳細は、[パスワードグラントのドキュメント](#customizing-the-user-provider)を調べてください。

<a name="passing-the-access-token"></a>
### アクセストークンの受け渡し

Passportにより保護されているルートを呼び出す場合、あなたのアプリケーションのAPI利用者は、リクエストの`Authorization`ヘッダとして、アクセストークンを`Bearer`トークンとして指定する必要があります。Guzzle HTTPライブラリを使う場合を例として示します。

    use Illuminate\Support\Facades\Http;

    $response = Http::withHeaders([
        'Accept' => 'application/json',
        'Authorization' => 'Bearer '.$accessToken,
    ])->get('https://passport-app.test/api/user');

    return $response->json();

<a name="token-scopes"></a>
## トークンのスコープ

スコープは、あるアカウントにアクセスする許可がリクエストされたとき、あなたのAPIクライアントに限定された一連の許可をリクエストできるようにします。たとえば、eコマースアプリケーションを構築している場合、全API利用者へ発注する許可を与える必要はないでしょう。代わりに、利用者へ注文の発送状況にアクセスできる許可を与えれば十分です。言い換えれば、スコープはアプリケーションユーザーに対し、彼らの代理としてのサードパーティアプリケーションが実行できるアクションを制限できるようにします。

<a name="defining-scopes"></a>
### スコープの定義

APIのスコープは、アプリケーションの`App\Providers\AppServiceProvider`クラスの`boot`メソッドで、`Passport::tokensCan`メソッドを使用して定義してください。`tokensCan`メソッドは、スコープ名とスコープの説明の配列を引数に取ります。スコープの説明は何でもよく、認可の承認画面でユーザーへ表示します。

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Passport::tokensCan([
            'place-orders' => 'Place orders',
            'check-status' => 'Check order status',
        ]);
    }

<a name="default-scope"></a>
### デフォルトスコープ

クライアントが特定のスコープを要求しない場合に、`setDefaultScope`メソッドを使用して、トークンへデフォルトのスコープを指定するように、Passportサーバを設定できます。通常、このメソッドはアプリケーションの`App\Providers\AppServiceProvider`クラスの`boot`メソッドから呼び出します。

    use Laravel\Passport\Passport;

    Passport::tokensCan([
        'place-orders' => 'Place orders',
        'check-status' => 'Check order status',
    ]);

    Passport::setDefaultScope([
        'check-status',
        'place-orders',
    ]);

> [!NOTE]
> Passportのデフォルトスコープは、ユーザーによって生成される個人用アクセストークンには適用されません。

<a name="assigning-scopes-to-tokens"></a>
### トークンへのスコープ割り付け

<a name="when-requesting-authorization-codes"></a>
#### 許可コードのリクエスト時

許可コードグラントを用い、アクセストークンをリクエストする際、利用者は`scope`クエリ文字列パラメータとして、希望するスコープを指定する必要があります。`scope`パラメータはスコープを空白で区切ったリストです。

    Route::get('/redirect', function () {
        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://example.com/callback',
            'response_type' => 'code',
            'scope' => 'place-orders check-status',
        ]);

        return redirect('http://passport-app.test/oauth/authorize?'.$query);
    });CheckClientCredentials

<a name="when-issuing-personal-access-tokens"></a>
#### パーソナルアクセストークン発行時

`App\Models\User`モデルの`createToken`メソッドを使用してパーソナルアクセストークンを発行している場合は、メソッドの２番目の引数に目的のスコープの配列を渡すことができます。

    $token = $user->createToken('My Token', ['place-orders'])->accessToken;

<a name="checking-scopes"></a>
### スコープのチェック

Passportは２つのミドルウェアを用意しており、受信リクエストが指定したスコープで付与したトークンにより認証されていることを確認するために使用できます。使用するには、アプリケーションの`bootstrap/app.php`ファイルで、以下のミドルウェアのエイリアスを定義します。

    use Laravel\Passport\Http\Middleware\CheckForAnyScope;
    use Laravel\Passport\Http\Middleware\CheckScopes;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'scopes' => CheckScopes::class,
            'scope' => CheckForAnyScope::class,
        ]);
    })

<a name="check-for-all-scopes"></a>
#### 全スコープの確認

`scopes`ミドルウェアをルートに割り当てて、受信リクエストのアクセストークンがリストするスコープをすべて持っていることを確認できます。

    Route::get('/orders', function () {
        // アクセストークンは"check-status"と"place-orders"、両スコープを持っている
    })->middleware(['auth:api', 'scopes:check-status,place-orders']);

<a name="check-for-any-scopes"></a>
#### 一部のスコープの確認

`scope`ミドルウエアは、リストしたスコープのうち、**最低１つ**を送信されてきたリクエストのアクセストークンが持っていることを確認するため、ルートへ指定します。

    Route::get('/orders', function () {
        // アクセストークンは、"check-status"か"place-orders"、どちらかのスコープを持っている
    })->middleware(['auth:api', 'scope:check-status,place-orders']);

<a name="checking-scopes-on-a-token-instance"></a>
#### トークンインスタンスでのスコープチェック

アクセストークンの認証済みリクエストがアプリケーションに入力された後でも、認証済みの`App\Models\User`インスタンスで`tokenCan`メソッドを使用して、トークンに特定のスコープがあるかどうかを確認できます。

    use Illuminate\Http\Request;

    Route::get('/orders', function (Request $request) {
        if ($request->user()->tokenCan('place-orders')) {
            // ...
        }
    });CheckClientCredentials

<a name="additional-scope-methods"></a>
#### その他のスコープメソッド

`scopeIds`メソッドは定義済みの全ID／名前の配列を返します。

    use Laravel\Passport\Passport;

    Passport::scopeIds();

`scopes`メソッドは定義済みの全スコープを`Laravel\Passport\Scope`のインスタンスの配列として返します。

    Passport::scopes();

`scopesFor`メソッドは、指定したID／名前に一致する`Laravel\Passport\Scope`インスタンスの配列を返します。

    Passport::scopesFor(['place-orders', 'check-status']);

指定したスコープが定義済みであるかを判定するには、`hasScope`メソッドを使います。

    Passport::hasScope('place-orders');

<a name="consuming-your-api-with-javascript"></a>
## APIをJavaScriptで利用

API構築時にJavaScriptアプリケーションから、自分のAPIを利用できたらとても便利です。このAPI開発のアプローチにより、世界中で共有されるのと同一のAPIを自身のアプリケーションで使用できるようになります。自分のWebアプリケーションやモバイルアプリケーション、サードパーティアプリケーション、そしてさまざまなパッケージマネージャ上で公開するSDKにより、同じAPIが使用されます。

通常、JavaScriptアプリケーションからAPIを利用する場合、手作業でアクセストークンをアプリケーションへ送信し、アプリケーションへリクエストするたび、アクセストークンを渡す必要があります。しかし、Passportは、この処理を代行するミドルウェアを用意しています。アプリケーションの`bootstrap/app.php`ファイルの`web`ミドルウェアグループに`CreateFreshApiToken`ミドルウェアを追加するだけです。

    use Laravel\Passport\Http\Middleware\CreateFreshApiToken;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->web(append: [
            CreateFreshApiToken::class,
        ]);
    })

> [!WARNING]
> ミドルウェアの指定の中で、`CreateFreshApiToken`ミドルウェアは確実に最後へリストしてください。

このミドルウェアは、送信レスポンスに`laravel_token`クッキーを添付します。このクッキーには、PassportがJavaScriptアプリケーションからのAPIリクエストを認証するために使用する暗号化されたJWTが含まれています。JWTの有効期間は、`session.lifetime`設定値と同じです。これで、ブラウザは後続のすべてのリクエストでクッキーを自動的に送信するため、アクセストークンを明示的に渡さなくても、アプリケーションのAPIにリクエストを送信できます。

    axios.get('/api/user')
        .then(response => {
            console.log(response.data);
        });

<a name="customizing-the-cookie-name"></a>
#### クッキー名のカスタマイズ

必要であれば、`Passport::cookie`メソッドをつかい、`laravel_token`クッキーの名前をカスタマイズできます。通常、このメソッドはアプリケーションの`App\Providers\AppServiceProvider`クラスの`boot`メソッドから呼び出します。

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Passport::cookie('custom_name');
    }

<a name="csrf-protection"></a>
#### CSRF保護

この認証方法を使用する場合、リクエストのヘッダに有効なCSRFトークンを確実に含める必要があります。デフォルトのLaravel JavaScriptスカフォールドはAxiosインスタンスを含み、同一オリジンリクエスト上に`X-XSRF-TOKEN`ヘッダを送るために、暗号化された`XSRF-TOKEN`クッキーを自動的に使用します。

> [!NOTE]
> `X-XSRF-TOKEN`の代わりに`X-CSRF-TOKEN`ヘッダを送る方法を取る場合は、`csrf_token()`により提供される復元したトークンを使用する必要があります。

<a name="events"></a>
## イベント

Passportはアクセストークンやリフレッシュトークンを発行する際に、イベントを発生させます。これらの[イベントをリッスン](/docs/{{version}}/events)して、データベース内の他のアクセストークンを削除したり取り消したりできます。

イベント名 |
------------- |
`Laravel\Passport\Events\AccessTokenCreated` |
`Laravel\Passport\Events\RefreshTokenCreated` |

<a name="testing"></a>
## テスト

Passportの`actingAs`メソッドは、現在認証中のユーザーを指定すると同時にスコープも指定します。`actingAs`メソッドの最初の引数はユーザーのインスタンスで、第２引数はユーザートークンに許可するスコープ配列を指定します。

```php tab=Pest
use App\Models\User;
use Laravel\Passport\Passport;

test('servers can be created', function () {
    Passport::actingAs(
        User::factory()->create(),
        ['create-servers']
    );

    $response = $this->post('/api/create-server');

    $response->assertStatus(201);
});
```

```php tab=PHPUnit
use App\Models\User;
use Laravel\Passport\Passport;

public function test_servers_can_be_created(): void
{
    Passport::actingAs(
        User::factory()->create(),
        ['create-servers']
    );

    $response = $this->post('/api/create-server');

    $response->assertStatus(201);
}
```

Passportの`actingAsClient`メソッドは、現在認証中のクライアントを指定すると同時にスコープも指定します。`actingAsClient`メソッドの最初の引数はクライアントインスタンスで、第２引数はクライアントのトークンへ許可するスコープの配列です。

```php tab=Pest
use Laravel\Passport\Client;
use Laravel\Passport\Passport;

test('orders can be retrieved', function () {
    Passport::actingAsClient(
        Client::factory()->create(),
        ['check-status']
    );

    $response = $this->get('/api/orders');

    $response->assertStatus(200);
});
```

```php tab=PHPUnit
use Laravel\Passport\Client;
use Laravel\Passport\Passport;

public function test_orders_can_be_retrieved(): void
{
    Passport::actingAsClient(
        Client::factory()->create(),
        ['check-status']
    );

    $response = $this->get('/api/orders');

    $response->assertStatus(200);
}
```
