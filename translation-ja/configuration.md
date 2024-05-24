# 設定

- [イントロダクション](#introduction)
- [環境設定](#environment-configuration)
    - [環境変数タイプ](#environment-variable-types)
    - [環境設定の取得](#retrieving-environment-configuration)
    - [現在環境の決定](#determining-the-current-environment)
    - [環境ファイルの暗号化](#encrypting-environment-files)
- [設定値へのアクセス](#accessing-configuration-values)
- [設定キャッシュ](#configuration-caching)
- [設定のリソース公開](#configuration-publishing)
- [デバッグモード](#debug-mode)
- [メンテナンスモード](#maintenance-mode)

<a name="introduction"></a>
## イントロダクション

Laravelフレームワークの全設定ファイルは、`config`ディレクトリに保存されています。各オプションには詳しいコメントが付いているので、各ファイルを一読し、使用できるオプションを把握しておきましょう。

これら設定ファイルを使用すると、データベース接続情報、メールサーバ情報、およびアプリケーションのタイムゾーンや暗号化キーなどの他のさまざまなコア設定値などを設定できます。

<a name="the-about-command"></a>
#### `about`コマンド

Laravelでは、`about` Artisanコマンドでアプリケーションの設定、ドライバ、環境の概要を表示できます。

```shell
php artisan about
```

アプリケーション概要の出力のうち、特定のセクションのみ興味がある場合は、`--only`オプションを使用してそのセクションをフィルタリングすることができます。

```shell
php artisan about --only=environment
```

特定の設定ファイルの値を詳しく調べるには、`config:show` Artisanコマンドを使います。

```shell
php artisan config:show database
```

<a name="environment-configuration"></a>
## 環境設定

アプリケーションを実行している環境にもとづき、別の設定値に切り替えられると便利です。たとえば、ローカルと実働サーバでは、異なったキャッシュドライバを使いたいことでしょう。

これを簡単に実現するために、Laravelは[DotEnv](https://github.com/vlucas/phpdotenv) PHPライブラリを利用しています。Laravelの新規インストールでは、アプリケーションのルートディレクトリに、多くの一般的な環境変数を定義する`.env.example`ファイルが含まれます。Laravelのインストールプロセス中に、このファイルは自動的に`.env`へコピーされます。

Laravelのデフォルト`.env`ファイルは、一般的な設定値がいくつか用意していますが、アプリケーションがローカルで実行されているか、本番のWebサーバで実行されているかによって異なることでしょう。そうした値は、Laravelの`env`関数を使用して、`config`ディレクトリ内の設定ファイルから読み込まれるようにします。

チームで開発を行っているのであれば、`.env.example`ファイルをアプリケーションに含めて更新し続けることをお勧めします。example設定ファイルにプレースホルダの値を入れることで、チームの他の開発者はアプリケーションの実行に必要な環境変数を明確に知ることができます。

> [!NOTE]
> `.env`ファイルにあるすべての変数は、サーバレベルやシステムレベルで定義されている、外部の環境変数によってオーバーライドすることができます。

<a name="environment-file-security"></a>
#### 環境ファイルのセキュリティ

アプリケーションを使用する開発者/サーバごとに異なる環境設定が必要になる可能性があるため、`.env`ファイルをアプリケーションのソース管理にコミットしないでください。さらに、機密性の高い資格情報が公開されるため、侵入者がソース管理リポジトリにアクセスした場合のセキュリティリスクになります。

しかしながら、Laravelの組み込みの[環境の暗号化](#encrypting-environment-files)を使用して、環境ファイルを暗号化することも可能です。暗号化した環境ファイルは、安全にソース管理下に置けます。

<a name="additional-environment-files"></a>
#### 追加の環境ファイル

アプリケーションの環境変数を読み込む前に、Laravelは`APP_ENV`環境変数が外部から提供されているか、もしくは`--env` CLI引数が指定されているかを判断します。その場合、Laravelは`.env.[APP_ENV]`ファイルが存在すれば、それを読み込もうとします。存在しない場合は、デフォルトの`.env`ファイルを読み込みます。

<a name="environment-variable-types"></a>
### 環境変数タイプ

通常、`.env`ファイル内のすべての変数は文字列として解析されるため、`env()`関数からより広範囲の型を返せるように、いくつかの予約値が作成されています。

| `.env`値 | `env()`値    |
| -------- | ------------ |
| true     | (bool) true  |
| (true)   | (bool) true  |
| false    | (bool) false |
| (false)  | (bool) false |
| empty    | (string) ''  |
| (empty)  | (string) ''  |
| null     | (null) null  |
| (null)   | (null) null  |

スペースを含む値で環境変数を定義する必要がある場合は、値をダブルクォーテーションで囲むことによって定義できます。

```ini
APP_NAME="My Application"
```

<a name="retrieving-environment-configuration"></a>
### 環境設定の取得

`.env`ファイルでリストされているすべての変数は、アプリケーションがリクエストを受信すると、`$_ENV` PHPスーパーグローバルへロードされます。ただし、`env`関数を使用して、設定ファイル内のこれらの変数から値を取得することができます。実際、Laravel設定ファイルを確認すると、多くのオプションがすでにこの関数を使用していることがわかります。

    'debug' => env('APP_DEBUG', false),

`env`関数に渡す２番目の値は「デフォルト値」です。指定されたキーの環境変数が存在しない場合、この値が返されます。

<a name="determining-the-current-environment"></a>
### 現在環境の決定

現在のアプリケーション環境は、`.env`ファイルの`APP_ENV`変数により決まります。`APP`[ファサード](/docs/{{version}}/facades)の`environment`メソッドにより、この値へアクセスできます。

    use Illuminate\Support\Facades\App;

    $environment = App::environment();

`environment`メソッドに引数を渡して、環境が特定の値と一致するかどうかを判定することもできます。環境が指定された値のいずれかに一致する場合、メソッドは`true`を返します。

    if (App::environment('local')) {
        // 環境はlocal
    }

    if (App::environment(['local', 'staging'])) {
        // 環境はlocalかstaging
    }

> [!NOTE]
> 現在のアプリケーション環境の検出は、サーバレベルの`APP_ENV`環境変数を定義することで上書きできます。

<a name="encrypting-environment-files"></a>
### 環境ファイルの暗号化

暗号化してない環境ファイルは、絶対にソースコントロールで保存してはいけません。しかし、Laravelでは環境ファイルを暗号化でき、アプリケーションの他の部分と一緒にソースコントロールへ安全に追加できます。

<a name="encryption"></a>
#### 暗号化

環境ファイルを暗号化するには、`env:encrypt`コマンドを使用します。

```shell
php artisan env:encrypt
```

`env:encrypt`コマンドを実行すると、`.env`ファイルが暗号化され、暗号化した内容を`.env.encrypted`ファイルへ格納します。復号化キーはコマンドの出力として表示されますので、安全なパスワードマネージャで保存しておく必要があります。もし、自分自身で暗号化キーを指定する場合はコマンド実行時に、`--key`オプションを使用してください。

```shell
php artisan env:encrypt --key=3UVsEgGVK36XN82KKeyLFMhvosbZN1aF
```

> [!NOTE]
> 指定するキーの長さは、使用する暗号化方式で要求される鍵の長さと合わせる必要があります。デフォルトでLaravelは、３２文字のキーを必要とする`AES-256-CBC`暗号を使用します。コマンド起動時に、`--cipher`オプションを指定すれば、Laravelの [暗号化](/docs/{{version}}/encryption)でサポートする暗号を自由に指定できます。

アプリケーションで`.env`や`.env.staging`など、複数の環境ファイルを使用している場合は、`--env`オプションで環境名を指定することで、暗号化する環境ファイルを指定します。

```shell
php artisan env:encrypt --env=staging
```

<a name="decryption"></a>
#### 復号化

環境ファイルを復号化するには、`env:decrypt`コマンドを使用します。このコマンドは復号化キーを必要とし、Laravelは`LARAVEL_ENV_ENCRYPTION_KEY`環境変数からこれを取得します。

```shell
php artisan env:decrypt
```

もしくは、`--key`オプションで、キーを直接コマンドへ指定することもできます。

```shell
php artisan env:decrypt --key=3UVsEgGVK36XN82KKeyLFMhvosbZN1aF
```

`env:decrypt`コマンドを実行すると、Laravelは`.env.encrypted`ファイルの内容を復号化し、復号化した内容を`.env`ファイルへ格納します。

`env:decrypt`コマンドへ、`--cipher`オプションを指定すると、カスタム暗号を使用できます。

```shell
php artisan env:decrypt --key=qUWuNRdfuImXcKxZ --cipher=AES-128-CBC
```

アプリケーションが`.env`や`.env.staging`など、複数の環境ファイルを使用している場合は、`--env`オプションで環境名を指定することにより、復号化する環境ファイルを指定できます。

```shell
php artisan env:decrypt --env=staging
```

既存の環境ファイルを上書きするには、`env:decrypt`コマンドへ`--force`オプションを指定します。

```shell
php artisan env:decrypt --force
```

<a name="accessing-configuration-values"></a>
## 設定値へのアクセス

アプリケーションのどこからでも、`Config`ファサードと`config`グローバル関数を使い、簡単に設定値にアクセスできます。設定値には「ドット」構文を使いアクセスでき、アクセスしたいファイル名とオプション名を指定します。デフォルト値を指定することもでき、その設定オプションが存在しない場合に返します。

    use Illuminate\Support\Facades\Config;

    $value = Config::get('app.timezone');

    $value = config('app.timezone');

    // 設定値が存在しない場合、デフォルト値を取得する
    $value = config('app.timezone', 'Asia/Seoul');

実行時に設定値をセットするには、`Config`ファサードの`set`メソッドを呼び出すか、`config` 関数に配列を渡します。

    Config::set('app.timezone', 'America/Chicago');

    config(['app.timezone' => 'America/Chicago']);

静的解析を支援するため、`Config`ファサードは型付き設定値の取得メソッドも提供しています。取得した設定値が期待した型と一致しない場合、例外を投げます。

    Config::string('config-key');
    Config::integer('config-key');
    Config::float('config-key');
    Config::boolean('config-key');
    Config::array('config-key');

<a name="configuration-caching"></a>
## 設定キャッシュ

アプリケーションの速度を上げるには、`config:cache` Artisanコマンドを使用してすべての設定ファイルを1つのファイルへキャッシュする必要があります。これにより、アプリケーションのすべての設定オプションが１ファイルに結合され、フレームワークによってすばやくロードされます。

通常、本番デプロイメントプロセスの一部として`php artisan config:cache`コマンドを実行する必要があります。アプリケーションの開発中は設定オプションを頻繁に変更する必要があるため、ローカル開発中はコマンドを実行しないでください。

いったん設定がキャッシュされると、リクエスト時やArtisanコマンド実行時に、アプリケーションの`.env`ファイルをフレームワークは読み込みません。したがって、`env`関数は外部のシステムレベルの環境変数のみを返します。

この理由により、アプリケーションの設定ファイル（`config`）の中からしか、`env`関数を呼び出さないようにする必要があります。Laravelのデフォルトの設定ファイルを調べると、多くの例を確認できます。設定値には、アプリケーションのどこからでも、[上記](#accessing-configuration-values)の`config`関数を使用してアクセスできます。

`config:clear`コマンドは、キャッシュされた設定を消去するために使用します。

```shell
php artisan config:clear
```

> [!WARNING]
> 開発過程の一環として`config:cache`コマンド実行を採用する場合は、必ず`env`関数を設定ファイルの中だけで使用してください。設定ファイルがキャッシュされると、`.env`ファイルはロードされません。したがって、`env`関数は外部システムレベルの環境変数のみを返すだけです。

<a name="configuration-publishing"></a>
## 設定のリソース公開

Laravelのほとんどの設定ファイルは、あらかじめアプリケーションの`config`ディレクトリでリソース公開されていますが、`cors.php`や`view.php`のような特定の設定ファイルは、ほとんどのアプリケーションでは変更する必要がないため、デフォルトでは公開していません。

しかし、`config:publish` Artisanコマンドを使用し、デフォルトで公開していない設定ファイルをリソース公開できます。

```shell
php artisan config:publish

php artisan config:publish --all
```

<a name="debug-mode"></a>
## デバッグモード

`config/app.php`設定ファイルの`debug`オプションは、エラーに関する情報が実際にユーザーに表示される量を決定します。デフォルトでは、このオプションは、`.env`ファイルに保存されている`APP_DEBUG`環境変数の値を尊重するように設定されています。

> [!WARNING]
> ローカル開発の場合は、`APP_DEBUG`環境変数を`true`に設定する必要があります。**実稼働環境では、この値は常に`false`である必要があります。本番環境で変数が`true`に設定されていると、機密性の高い設定値がアプリケーションのエンドユーザーに公開されるリスクがあります。**

<a name="maintenance-mode"></a>
## メンテナンスモード

アプリケーションをメンテナンスモードにすると、アプリケーションに対するリクエストに対し、すべてカスタムビューが表示されるようになります。アプリケーションのアップデート中や、メンテナンス中に、アプリケーションを簡単に「停止」状態にできます。メンテナンスモードのチェックは、アプリケーションのデフォルトミドルウェアスタックに含まれています。アプリケーションがメンテナンスモードの時、ステータスコード503で`Symfony\Component\HttpKernel\Exception\HttpException`インスタンスを投げます。

メンテナンスモードにするには、`down` Artisanコマンドを実行します。

```shell
php artisan down
```

メンテナンスモードのすべてのレスポンスで、`Refresh`のHTTPヘッダを送信したい場合は、`down`コマンドを実行する際に、`refresh`オプションを指定します。`Refresh`ヘッダは、指定した秒数後にページを自動的に更新するようブラウザに指示します。

```shell
php artisan down --refresh=15
```

また、`down`コマンドに`retry`オプションを指定し、HTTPヘッダの`Retry-After`値を設定できますが、通常ブラウザはこのヘッダを無視します。

```shell
php artisan down --retry=60
```

<a name="bypassing-maintenance-mode"></a>
#### メンテナンスモードをバイパスする

シークレットトークンを使い、メンテナンスモードをバイパスできるようにするには、`secret`オプションを使い、メンテナンスモードパイパストークンを指定します。

```shell
php artisan down --secret="1630542a-246b-4b66-afa1-dd72a4c43515"
```

アプリケーションをメンテナンスモードにしたあとで、このトークンと同じURLによりブラウザでアプリケーションにアクセスすると、メンテナンスモードバイパスクッキーがそのブラウザへ発行されます。

```shell
https://example.com/1630542a-246b-4b66-afa1-dd72a4c43515
```

Laravelにシークレットトークンを生成してもらいたい場合は、`with-secret`オプションを使用します。アプリケーションがメンテナンスモードになると、シークレットが表示されます。

```shell
php artisan down --with-secret
```

この隠しルートへアクセスすると、次にアプリケーションの`/`ルートへリダイレクトされます。ブラウザへこのクッキーが一度発行されると、メンテナンスモードでない状態と同様に、アプリケーションへ普通にブラウズできます。

> [!NOTE]
> メンテナンスモードのシークレットは、通常、英数字とオプションでダッシュで構成されるべきです。URLの中で特別な意味を持つ文字、例えば`?`や`&`の使用は避けるべきです。

<a name="maintenance-mode-on-multiple-servers"></a>
#### 複数サーバでのメンテナンスモード

Laravelはデフォルトで、ファイルベースのシステムを使ってアプリケーションがメンテナンスモードかを判断します。つまり、メンテナンスモードを有効にするには、アプリケーションをホストしている各サーバ上で`php artisan down`コマンドを実行する必要があります。

別の方法としてLaravelは、メンテナンスモードをキャッシュベースで処理する方法も提供しています。この方法では、１つのサーバ上で`php artisan down`コマンドを実行する必要があるだけです。この方法を使用するには、アプリケーションの`config/app.php`ファイルの"driver"設定を`cache`に変更してください。そして、すべてのサーバからアクセス可能なキャッシュ`store`を選択してください。これにより、メンテナンスモードの状態がすべてのサーバで一貫して維持されるようになります。

```php
'maintenance' => [
    'driver' => 'cache',
    'store' => 'database',
],
```

<a name="pre-rendering-the-maintenance-mode-view"></a>
#### Viewメンテナンスモードビューの事前レンダリング

開発時に`php artisan down`コマンドを使うと、Composerの依存パッケージやその他の基盤コンポーネントのアップデート中に、アプリケーションへユーザーがアクセスすると、エラーが発生することがあります。この理由は、アプリケーションがメンテナンスモードであると判断することやテンプレートエンジンによりメンテナンスモードビューをレンダするには、Laravelフレームワークのかなりの部分が起動している必要があるからです。

このため、Laravelはリクエストサイクルの最初に返されるメンテナンスモードビューを事前レンダできます。このビューは、アプリケーションの依存関係が読み込まれる前にレンダされます。`down`コマンドの`render`オプションで、選んだテンプレートを事前レンダできます。

```shell
php artisan down --render="errors::503"
```

<a name="redirecting-maintenance-mode-requests"></a>
#### メンテナンスモードのリクエストのリダイレクト

URI:メンテナンスモード中、Laravelはユーザーがアクセスしてきたアプリケーションの全URLに対し、メンテナンスモードビューを表示します。お望みならば、全リクエストを特定のＵＲＬへリダイレクトすることも可能です。`redirect`オプションを使用してください。例として、全リクエストを`/`のＵＲＩへリダイレクトするとしましょう。

```shell
php artisan down --redirect=/
```

<a name="disabling-maintenance-mode"></a>
#### メンテナンスモードの無効化

メンテナンスモードから抜けるには、`up`コマンドを使います。

```shell
php artisan up
```

> [!NOTE]
> `resources/views/errors/503.blade.php`を独自に定義することにより、メンテナンスモードのデフォルトテンプレートをカスタマイズできます。

<a name="maintenance-mode-queues"></a>
#### メンテナンスモードとキュー

アプリケーションがメンテナンスモードの間、[キューされたジョブ](/docs/{{version}}/queues)は実行されません。メンテナンスモードから抜け、アプリケーションが通常状態へ戻った時点で、ジョブは続けて処理されます。

<a name="alternatives-to-maintenance-mode"></a>
#### メンテナンスモードの代替

メンテナンスモードではアプリケーションに数秒のダウンタイムが必要なため、Laravelを使う開発においてはダウンタイムゼロを達成するために、[Laravel Vapor](https://vapor.laravel.com)や[Envoyer](https://envoyer.io)などの代替手段を検討してください。
