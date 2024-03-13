# パッケージ開発

- [イントロダクション](#introduction)
    - [ファサード使用の注意](#a-note-on-facades)
- [パッケージディスカバリー](#package-discovery)
- [サービスプロバイダ](#service-providers)
- [リソース](#resources)
    - [設定](#configuration)
    - [マイグレーション](#migrations)
    - [ルート](#routes)
    - [言語ファイル](#language-files)
    - [ビュー](#views)
    - [ビューコンポーネント](#view-components)
    - ["About" Artisanコマンド](#about-artisan-command)
- [コマンド](#commands)
- [リソース公開アセット](#public-assets)
- [ファイルグループのリソース公開](#publishing-file-groups)

<a name="introduction"></a>
## イントロダクション

パッケージは、Laravelに機能を追加するための主要な方法です。パッケージは、[Carbon](https://github.com/briannesbitt/Carbon)のような日付を処理するための優れた方法から、Spatieの[Laravel Media Library](https://github.com/spatie/laravel-medialibrary)のようなEloquentモデルにファイルを関連付けることができるパッケージまであります。

There are different types of packages. Some packages are stand-alone, meaning they work with any PHP framework. Carbon and Pest are examples of stand-alone packages. Any of these packages may be used with Laravel by requiring them in your `composer.json` file.

逆にLaravelと一緒に使用することを意図したパッケージもあります。こうしたパッケージはLaravelアプリケーションを高めることをとくに意図したルート、コントローラ、ビュー、設定を持つことでしょう。このガイドはLaravelに特化したパッケージの開発を主に説明します。

<a name="a-note-on-facades"></a>
### ファサード使用の注意

Laravelアプリケーションを作成する場合、コントラクトとファサードのどちらを使用しても、どちらも基本的に同じレベルのテスト容易性を提供するため、通常は問題になりません。ただし、パッケージを作成する場合、通常、パッケージはLaravelのすべてのテストヘルパにアクセスできるわけではありません。パッケージが一般的なLaravelアプリケーション内にインストールされているかのようにパッケージテストを記述できるようにしたい場合は、[Orchestral Testbench](https://github.com/orchestral/testbench)パッケージを使用できます。

<a name="package-discovery"></a>
## パッケージディスカバリー

A Laravel application's `bootstrap/providers.php` file contains the list of service providers that should be loaded by Laravel. However, instead of requiring users to manually add your service provider to the list, you may define the provider in the `extra` section of your package's `composer.json` file so that it is automatically loaded by Laravel. In addition to service providers, you may also list any [facades](/docs/{{version}}/facades) you would like to be registered:

```json
"extra": {
    "laravel": {
        "providers": [
            "Barryvdh\\Debugbar\\ServiceProvider"
        ],
        "aliases": {
            "Debugbar": "Barryvdh\\Debugbar\\Facade"
        }
    }
},
```

ディスカバリー用にパッケージを設定したら、Laravelはサービスプロバイダとファサードをインストール時に自動的に登録します。皆さんのパッケージユーザーに、便利なインストール体験をもたらします。

<a name="opting-out-of-package-discovery"></a>
#### パッケージディスカバリーの不使用

パッケージを利用する場合に、パッケージディスカバリーを使用したくない場合は、アプリケーションの`composer.json`ファイルの`extra`セクションに、使用しないパッケージをリストしてください。

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "barryvdh/laravel-debugbar"
        ]
    }
},
```

全パッケージに対してディスカバリーを使用しない場合は、アプリケーションの`dont-discover`ディレクティブに、`*`文字を指定してください。

```json
"extra": {
    "laravel": {
        "dont-discover": [
            "*"
        ]
    }
},
```

<a name="service-providers"></a>
## サービスプロバイダ

[サービスプロバイダ](/docs/{{version}}/providers)は、あなたのパッケージとLaravelの接続ポイントです。サービスプロバイダは、Laravelの[サービスコンテナ](/docs/{{version}}/container)で必要なものを結合し、ビュー、設定、言語ファイルなどのパッケージリソースをロードする場所をLaravelへ知らせる役割を持っています。

サービスプロバイダは`Illuminate\Support\ServiceProvider`クラスを拡張し、`register`と`boot`の２メソッドを含んでいます。ベースの`ServiceProvider`クラスは、`illuminate/support` Composerパッケージにあります。 サービスプロバイダの構造と目的について詳細を知りたければ、[ドキュメント](/docs/{{version}}/providers)を調べてください。

<a name="resources"></a>
## リソース

<a name="configuration"></a>
### 設定

通常、パッケージの設定ファイルをアプリケーションの`config`ディレクトリにリソース公開する必要があります。これにより、パッケージのユーザーはデフォルトの設定オプションを簡単に上書きできます。設定ファイルをリソース公開できるようにするには、サービスプロバイダの`boot`メソッドから`publishes`メソッドを呼び出します。

    /**
     * 全パッケージサービスの初期起動処理
     */
    public function boot(): void
    {
        $this->publishes([
            __DIR__.'/../config/courier.php' => config_path('courier.php'),
        ]);
    }

これで、皆さんのパッケージのユーザーが、Laravelの`vendor:publish`コマンドを実行すると、特定のリソース公開場所へファイルがコピーされます。設定がリソース公開されても、他の設定ファイルと同様に値にアクセスできます。

    $value = config('courier.option');

> [!WARNING]
> 設定ファイルでクロージャを定義しないでください。ユーザーが`config:cache` Artisanコマンドを実行すると、正しくシリアル化できません。

<a name="default-package-configuration"></a>
#### デフォルトパッケージ設定

独自のパッケージ設定ファイルをアプリケーションのリソース公開コピーとマージすることもできます。これにより、ユーザーは、設定ファイルのリソース公開されたコピーで実際にオーバーライドするオプションのみを定義できます。設定ファイルの値をマージするには、サービスプロバイダの`register`メソッド内で`mergeConfigFrom`メソッドを使用します。

`mergeConfigFrom`メソッドは、パッケージの設定ファイルへのパスを最初の引数に取り、アプリケーションの設定ファイルのコピーの名前を２番目の引数に取ります。

    /**
     * 全アプリケーションサービスの登録
     */
    public function register(): void
    {
        $this->mergeConfigFrom(
            __DIR__.'/../config/courier.php', 'courier'
        );
    }

> [!WARNING]
> このメソッドは設定配列の一次レベルのみマージします。パッケージのユーザーが部分的に多次元の設定配列を定義すると、マージされずに欠落するオプションが発生します。

<a name="routes"></a>
### ルート

パッケージにルートを含めている場合は、`loadRoutesFrom`メソッドでロードします。このメソッドは自動的にアプリケーションのルートがキャッシュされているかを判定し、すでにキャッシュ済みの場合はロードしません。

    /**
     * 全パッケージサービスの初期起動処理
     */
    public function boot(): void
    {
        $this->loadRoutesFrom(__DIR__.'/../routes/web.php');
    }

<a name="migrations"></a>
### マイグレーション

If your package contains [database migrations](/docs/{{version}}/migrations), you may use the `publishesMigrations` method to inform Laravel that the given directory or file contains migrations. When Laravel publishes the migrations, it will automatically update the timestamp within their filename to reflect the current date and time:

    /**
     * 全パッケージサービスの初期起動処理
     */
    public function boot(): void
    {
        $this->publishesMigrations([
            __DIR__.'/../database/migrations' => database_path('migrations'),
        ]);
    }

<a name="language-files"></a>
### 言語ファイル

パッケージに[言語ファイル](/docs/{{version}}/localization)を用意している場合、`loadTranslationsFrom`メソッドを使用して、それらをロードする方法をLaravelへ知らせてください。例えば、パッケージの名前が`courier`の場合、サービスプロバイダの`boot`メソッドに以下を追加する必要があります。

    /**
     * 全パッケージサービスの初期起動処理
     */
    public function boot(): void
    {
        $this->loadTranslationsFrom(__DIR__.'/../lang', 'courier');
    }

パッケージの翻訳行は、`package::file.line`の規約で参照します。したがって、`courier`パッケージの`messages`ファイルから、`welcome`は以下のように読み込みます。

    echo trans('courier::messages.welcome');

`loadJsonTranslationsFrom`メソッドを使うと、パッケージへJSON翻訳ファイルを登録できます。このメソッドで、パッケージのJSON翻訳ファイルがあるディレクトリパスを指定します。

```php
/**
 * パッケージの全サービスの起動処理
 */
public function boot(): void
{
    $this->loadJsonTranslationsFrom(__DIR__.'/../lang');
}
```

<a name="publishing-language-files"></a>
#### 言語ファイルのリソース公開

パッケージの言語ファイルをアプリケーションの`lang/vendor`ディレクトリへリソース公開したい場合は、サービスプロバイダの`publishes`メソッドを使用します。`publishes`メソッドは、パッケージのパスと公開したい場所の配列を引数に取ります。例えば、`courier`パッケージの言語ファイルを公開するには、以下のようにします。

    /**
     * 全パッケージサービスの初期起動処理
     */
    public function boot(): void
    {
        $this->loadTranslationsFrom(__DIR__.'/../lang', 'courier');

        $this->publishes([
            __DIR__.'/../lang' => $this->app->langPath('vendor/courier'),
        ]);
    }

これで、パッケージのユーザーがLaravelの`vendor:publish` Artisanコマンドを実行すると、パッケージの言語ファイルが指定した公開場所にリソース公開されます。

<a name="views"></a>
### ビュー

パッケージの[ビュー](/docs/{{version}}/views)をLaravelへ登録するには、ビューがどこにあるのかをLaravelに知らせる必要があります。そのために、サービスプロバイダの`loadViewsFrom`メソッドを使用してください。`loadViewsFrom`メソッドは２つの引数を取ります。ビューテンプレートへのパスと、パッケージの名前です。たとえば、パッケージ名が`courier`であれば、以下の行をサービスプロバイダの`boot`メソッドに追加してください。

    /**
     * 全パッケージサービスの初期起動処理
     */
    public function boot(): void
    {
        $this->loadViewsFrom(__DIR__.'/../resources/views', 'courier');
    }

パッケージのビューは、`package::view`記法を使い参照します。そのため、ビューのパスを登録し終えたあとで、`courier`パッケージの`dashboard`ビューをロードする場合は、次のようになります。

    Route::get('/dashboard', function () {
        return view('courier::dashboard');
    });

<a name="overriding-package-views"></a>
#### パッケージビューのオーバーライド

`loadViewsFrom`メソッドを使用すると、Laravelはビューの２つの場所を実際に登録します。アプリケーションの`resources/views/vendor`ディレクトリと指定したディレクトリです。そのため、たとえば`courier`パッケージを使用すると、Laravelは最初にカスタムバージョンのビューが開発者によって`resources/views/vendor/courier`ディレクトリに配置されているかどうかを確認します。次に、ビューがカスタマイズされていない場合、Laravelは`loadViewsFrom`の呼び出しで指定したパッケージビューディレクトリを検索します。これにより、パッケージユーザーはパッケージのビューを簡単にカスタマイズ／オーバーライドできます。

<a name="publishing-views"></a>
#### ビューのリソース公開

パッケージのビューを`resources/views/vendor`ディレクトリでリソース公開したい場合は、サービスプロバイダの`publishes`メソッドを使ってください。`publishes`メソッドはパッケージのビューパスと、リソース公開場所の配列を引数に取ります。

    /**
     * 全パッケージサービスの初期起動処理
     */
    public function boot(): void
    {
        $this->loadViewsFrom(__DIR__.'/../resources/views', 'courier');

        $this->publishes([
            __DIR__.'/../resources/views' => resource_path('views/vendor/courier'),
        ]);
    }

これで皆さんのパッケージのユーザーが、Laravelの`vendor::publish` Artisanコマンドを実行すると、パッケージのビューは指定されたリソース公開場所へコピーされます。

<a name="view-components"></a>
### ビューコンポーネント

Bladeコンポーネントを利用するパッケージを構築する場合、またはコンポーネントを従来と異なるディレクトリへ配置する場合、コンポーネントクラスとそのHTMLタグエイリアスを手作業で登録し、Laravelがコンポーネントを見つける場所を認識できるようにする必要があります。通常、パッケージのサービスプロバイダの`boot`メソッドで、コンポーネントを登録する必要があります。

    use Illuminate\Support\Facades\Blade;
    use VendorPackage\View\Components\AlertComponent;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Blade::component('package-alert', AlertComponent::class);
    }

コンポーネントを登録したら、タグエイリアスを使いレンダリングします。

```blade
<x-package-alert/>
```

<a name="autoloading-package-components"></a>
#### パッケージコンポーネントの自動ロード

もしくは、`componentNamespace`メソッドを使用して、コンポーネントクラスを規約に従いオートロードできます。例えば、`Nightshade`パッケージに`Calendar`と`ColorPicker`コンポーネントがあり、これらが`Nightshade\Views\Components`名前空間内に存在しているとしましょう。

    use Illuminate\Support\Facades\Blade;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Blade::componentNamespace('Nightshade\Views\Components', 'nightshade');
    }

これにより、`パッケージ名::`記法を使用し、ベンダーの名前空間により、パッケージコンポーネントが利用できるようになります。

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Bladeは、コンポーネント名をパスカルケース化し、このコンポーネントとリンクしているクラスを自動的に検出します。サブディレクトリも「ドット」記法でサポートしています。

<a name="anonymous-components"></a>
#### 無名コンポーネント

パッケージが無名コンポーネントを持っている場合、"views"ディレクトリ（[`loadViewsFrom`](#views)で指定している場所）の`components`ディレクトリの中へ設置する必要があります。すると、パッケージのビュー名前空間を先頭に付けたコンポーネント名でレンダできます。

```blade
<x-courier::alert />
```

<a name="about-artisan-command"></a>
### "About" Artisanコマンド

Laravelの組み込み`about` Artisanコマンドは、アプリケーションの環境と設定の概要を表示します。パッケージでは`AboutCommand`クラスを使用して、このコマンドの出力に追加情報を追加できます。一般的に、この情報はパッケージサービスプロバイダの`boot`メソッドに追加します。

    use Illuminate\Foundation\Console\AboutCommand;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        AboutCommand::add('My Package', fn () => ['Version' => '1.0.0']);
    }

<a name="commands"></a>
## コマンド

パッケージのArtisanコマンドをLaravelへ登録するには、`commands`メソッドを使います。このメソッドは、コマンドクラス名の配列を引数に取ります。コマンドを登録したら、[Artisan CLI](/docs/{{version}}/artisan)を使い、実行できます。

    use Courier\Console\Commands\InstallCommand;
    use Courier\Console\Commands\NetworkCommand;

    /**
     * 全パッケージサービスの初期起動処理
     */
    public function boot(): void
    {
        if ($this->app->runningInConsole()) {
            $this->commands([
                InstallCommand::class,
                NetworkCommand::class,
            ]);
        }
    }

<a name="public-assets"></a>
## リソース公開アセット

パッケージには、JavaScript、CSS、画像などのアセットが含まれている場合があります。これらのアセットをアプリケーションの`public`ディレクトリにリソース公開するには、サービスプロバイダの`publishes`メソッドを使用します。この例では、関連するアセットのグループを簡単にリソース公開するために使用できる`public`アセットグループタグも追加します。

    /**
     * 全パッケージサービスの初期起動処理
     */
    public function boot(): void
    {
        $this->publishes([
            __DIR__.'/../public' => public_path('vendor/courier'),
        ], 'public');
    }

これで、パッケージのユーザーが`vendor:publish`コマンドを実行すると、アセットが指定するリソース公開場所にコピーされます。通常、ユーザーはパッケージが更新されるたびにアセットを上書きする必要があるため、`--force`フラグを使用できます。

```shell
php artisan vendor:publish --tag=public --force
```

<a name="publishing-file-groups"></a>
## ファイルグループのリソース公開

パッケージアセットとリソースのグループを個別にリソース公開することを推奨します。たとえば、パッケージのアセットをリソース公開することを強制されることなく、ユーザーがパッケージの設定ファイルをリソース公開できるようにしたい場合もあるでしょう。パッケージのサービスプロバイダから`publishes`メソッドを呼び出すときに、それらに「タグ付け」することでこれを行うことができます。例として、パッケージのサービスプロバイダの`boot`メソッドで、`courier`パッケージの２公開グループ （`courier-config`と`courier-migrations`）をタグを使い定義してみましょう。

    /**
     * 全パッケージサービスの初期起動処理
     */
    public function boot(): void
    {
        $this->publishes([
            __DIR__.'/../config/package.php' => config_path('package.php')
        ], 'courier-config');

        $this->publishesMigrations([
            __DIR__.'/../database/migrations/' => database_path('migrations')
        ], 'courier-migrations');
    }

これでユーザーは、`vendor::publish` Artisanコマンドを使用するときにタグ名を指定することで、グループを別々にリソース公開できます。

```shell
php artisan vendor:publish --tag=courier-config
```
