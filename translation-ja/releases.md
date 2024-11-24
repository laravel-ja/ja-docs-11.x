# リリースノート

- [バージョニング規約](#versioning-scheme)
- [サポートポリシー](#support-policy)
- [Laravel11](#laravel-11)

<a name="versioning-scheme"></a>
## バージョニング規約

Laravelとファーストパーティパッケージは、[セマンティックバージョニング](https://semver.org)にしたがっています。メジャーなフレームのリリースは、毎年（第１四半期に）リリースします。マイナーとパッチリリースはより頻繁に毎週リリースします。マイナーとパッチリリースは、**決して**ブレーキングチェンジを含みません

アプリケーションやパッケージからLaravelフレームワークやそのコンポーネントを参照する場合、Laravelのメジャーリリースにはブレークチェンジが含まれるため、常に`^11.0`のようなバージョン制約を使用する必要があります。しかし、常に１日かからずに新しいメジャーリリースへアップデートできるように、私たちは努めています。

<a name="named-arguments"></a>
#### 名前付き引数

[名前付き引数](https://www.php.net/manual/ja/functions.arguments.php#functions.named-arguments)は、Laravelの下位互換性ガイドラインの対象外です。Laravelコードベースを改善するために、必要に応じて関数の引数の名前を変更することもできます。したがって、Laravelメソッドを呼び出すときに名前付き引数を使用する場合は、パラメータ名が将来変更される可能性があることを理解した上で、慎重に行う必要があります。

<a name="support-policy"></a>
## サポートポリシー

Laravelのすべてのリリースは、バグフィックスは１８ヶ月、セキュリティフィックスは２年です。Lumenのようなその他の追加ライブラリでは、最新のメジャーリリースのみでバグフィックスを受け付けています。また、[Laravelがサポートする](/docs/{{version}}/database#introduction)データベースのサポートについても確認してください。

<div class="overflow-auto">

| バージョン | PHP (*)   | リリース             | バグフィックス期日   | セキュリティ修正期日 |
| ---------- | --------- | -------------------- | -------------------- | -------------------- |
| 9          | 8.0 - 8.2 | ２０２２年２月８日   | ２０２３年８月８日   | ２０２４年２月６日   |
| 10         | 8.1 - 8.3 | ２０２３年２月１４日 | ２０２４年８月６日   | ２０２５年２月４日   |
| 11         | 8.2 - 8.4 | ２０２４年３月１２日 | ２０２５年９月３日   | ２０２６年３月１２日  |
| 12         | 8.2 - 8.4 | ２０２５年第１四半期 | ２０２６年第３四半期  | ２０２７年第１四半期 |

</div>

<div class="version-colors">
    <div class="end-of-life">
        <div class="color-box"></div>
        <div>End of life</div>
    </div>
    <div class="security-fixes">
        <div class="color-box"></div>
        <div>Security fixes only</div>
    </div>
</div>

(*) 対応PHPバージョン

<a name="laravel-11"></a>
## Laravel 11

Laravel11は、Laravel10.xで行われた改善を引き継ぎ、アプリケーション構造を合理化し、秒単位のレート制限、ヘルスルーティング、グレースフル暗号化キーローテーション、キューテストの改善、[Resend](https://resend.com)メールトランスポート、Promptとバリデータの統合、新しいArtisanコマンドなどを導入しました。さらに、ファーストパーティのスケーラブルなWebSocketサーバであるLaravel Reverbを導入し、アプリケーションに堅牢なリアルタイム機能を提供します。

<a name="php-8"></a>
### PHP 8.2

Laravel11.xには、最低でPHP8.2のバージョンが必要です。

<a name="structure"></a>
### 合理化したアプリケーション構造

*Laravelの合理化したアプリケーション構造は、[Taylor Otwell](https://github.com/taylorotwell)と[Nuno Maduro](https://github.com/nunomaduro)が開発しました。*

Laravel11では、既存のアプリケーションに変更を加えることなく、**新しい**Laravelアプリケーション向けに合理化したアプリケーション構造を導入しました。新しいアプリケーション構造は、Laravel開発者がすでに慣れ親しんでいるコンセプトの多くを保持しながら、よりスリムでモダンなエクスペリエンスを提供することを目的としています。以降で、Laravelの新しいアプリケーション構造のハイライトについて説明します。

#### アプリケーションの初期起動処理ファイル

`bootstrap/app.php`ファイルは、コード・ファーストのアプリケーション設定ファイルとして復活しました。このファイルから、アプリケーションのルーティング、ミドルウェア、サービスプロバイダ、例外処理などをカスタマイズできます。このファイルは、以前はアプリケーションのファイル構造全体に散らばっていたさまざまな高レベルのアプリケーション動作設定を統一しています。

```php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        //
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

<a name="service-providers"></a>
#### サービスプロバイダ

デフォルトのLaravelアプリケーション構造には５つのサービスプロバイダを持っていましたが、Laravel11では１つの`AppServiceProvider`しかありません。以前のサービスプロバイダの機能は、`bootstrap/app.php`に組み込まれたり、フレームワークが自動的に処理したり、アプリケーションの`AppServiceProvider`へ配置されたりしました。

例えば、イベントディスカバリはデフォルトで有効になり、イベントとそのリスナを手作業で登録する必要をほぼ無くしました。しかし、イベントを手作業で登録する必要がある場合は、`AppServiceProvider`に登録するだけです。同様に、以前 `AuthServiceProvider`で登録していた、ルートモデル結合や認証ゲートも、`AppServiceProvider`に登録できます。

<a name="opt-in-routing"></a>
#### オプトインAPIとブロードキャストルーティング

多くのアプリケーションがこうしたファイルを必要としないため、`api.php`と`channels.php`ルートファイルは、デフォルトでもはや存在しなくなりました。代わりに、簡単なArtisanコマンドを使用して、生成できます。

```shell
php artisan install:api

php artisan install:broadcasting
```

<a name="middleware"></a>
#### ミドルウェア

以前は、新しいLaravelアプリケーションは、９つのミドルウェアを持っていました。これらのミドルウェアは、リクエストの認証、入力文字列のトリミング、CSRFトークンの検証など、さまざまなタスクを実行していました。

Laravel11では、これらのミドルウェアをフレームワーク自体に移しました。これらのミドルウェアの動作をカスタマイズするための新しいメソッドをフレームワークへ追加しており、アプリケーションの`bootstrap/app.php`ファイルから呼び出せます。

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->validateCsrfTokens(
        except: ['stripe/*']
    );

    $middleware->web(append: [
        EnsureUserIsSubscribed::class,
    ])
})
```

すべてのミドルウェアは、アプリケーションの`bootstrap/app.php`から簡単にカスタマイズできるため、HTTP "kernel"クラスを別に用意する必要がなくなりました。

<a name="scheduling"></a>
#### スケジュール

新しい`Schedule`ファサードを使用して、タスクのスケジュールをアプリケーションの`routes/console.php`ファイルで直接定義できるようになりました。

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')->daily();
```

<a name="exception-handling"></a>
#### 例外処理

ルーティングやミドルウェアと同様に、例外処理も、例外ハンドラクラスを個別に作成する代わりに、アプリケーションの `bootstrap/app.php`ファイルからカスタマイズできるようになり、新しいLaravelアプリケーションに含まれるファイル全体の数を減らせました。

```php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->dontReport(MissedFlightException::class);

    $exceptions->report(function (InvalidOrderException $e) {
        // ...
    });
})
```

<a name="base-controller-class"></a>
#### `Controller`ベースクラス

新しいLaravelアプリケーションのベースコントローラを簡素化しました。Laravel内部の`Controller`クラスを継承しなくなり、`AuthorizesRequests`と`ValidatesRequests`トレイトを削除しました。これらは、必要に応じてアプリケーションの個々のコントローラで含めてください。

    <?php

    namespace App\Http\Controllers;

    abstract class Controller
    {
        //
    }

<a name="application-defaults"></a>
#### アプリケーションのデフォルト

新しいLaravelアプリケーションはデフォルトで、データベースストレージにSQLiteを使用し、Laravelのセッション、キャッシュ、キューに`database`ドライバを使用します。これにより、追加のソフトウェアをインストールしたり、追加のデータベースマイグレーションを作成したりする必要がなく、新しいLaravelアプリケーションを作成した後、すぐにアプリケーションの構築を開始できます。

加えて、これらのLaravelサービスの`データベース`ドライバは、多くのアプリケーションコンテキストで本番環境で使用するのに十分な堅牢性を持つようになり、ローカルアプリケーションとプロダクションアプリケーションの両方に対し、理にかなった統一した選択を提供しています。

<a name="reverb"></a>
### Laravel Reverb

*Laravel Reverbは、[Joe Dixon](https://github.com/joedixon)が、開発しました。*

[Laravel Reverb](https://reverb.laravel.com)は、高速でスケーラブルなリアルタイムWebSocket通信をLaravelアプリケーションに直接もたらし、Laravel EchoのようなLaravelの既存のイベントブロードキャストツール群とのシームレスな統合を提供します。

```shell
php artisan reverb:start
```

さらに、ReverbはRedisのPub／Sub機能により、水平スケーリングをサポートし、WebSocketトラフィックを複数のバックエンドReverbサーバへ分散して、単一の高需要アプリケーションをサポートします。

Laravel Reverbの詳細は、完全な[Reverbドキュメント](/docs/{{version}}/reverb)を参照してください。

<a name="rate-limiting"></a>
### 秒単位のレート制限

*秒単位のレート制限は、[Tim MacDonald](https://github.com/timacdonald)が貢献しました。*

Laravelは、HTTPリクエストやキュー投入したジョブを含む全てのレートリミッタで、「秒単位」のレート制限をサポートするようになりました。以前、Laravelのレートリミッタは 「分単位」で制限していました。

```php
RateLimiter::for('invoices', function (Request $request) {
    return Limit::perSecond(1);
});
```

Laravelのレート制限の詳細は、[レート制限ドキュメント](/docs/{{version}}/routing#rate-limiting)をチェックしてください。

<a name="health"></a>
### ヘルスルート

*ヘルスルートは、[Taylor Otwell](https://github.com/taylorotwell)が貢献しました。*

新しいLaravel11アプリケーションは、単純なヘルスチェックエンドポイントを定義するようLaravelに指示する、`health`ルートディレクティブが含まれています。このエンドポイントは、サードパーティのアプリケーションヘルスモニタリングサービスやKubernetesなどのオーケストレーションシステムから呼び出される可能性があり、デフォルトでこのルートは`/up`で提供しています。

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
)
```

このルートに対し、HTTPリクエストが行われると、Laravelは`DiagnosingHealth`イベントもディスパッチし、アプリケーションに関連する追加のヘルスチェックを実行できるようにします。

<a name="encryption"></a>
### 優しい暗号化キーのローテーション

*優しい暗号化キーのローテーションは、[Taylor Otwell](https://github.com/taylorotwell)が貢献しました。*

Laravelはアプリケーションのセッションクッキーを含むすべてのクッキーを暗号化するため、基本的にLaravelアプリケーションへのリクエストはすべて暗号化に依存しています。しかし、このため、アプリケーションの暗号化キーをローテーションすると、すべてのユーザーがアプリケーションからログアウトすることになります。さらに、以前の暗号化キーで暗号化されたデータを復号化できなくなります。

Laravel11では、`APP_PREVIOUS_KEYS`環境変数を使って、アプリケーションの以前の暗号化キーをカンマ区切りのリストとして定義することができます。

値を暗号化するとき、Laravelは常に`APP_KEY`環境変数の「現在の」暗号化キーを使用します。値を復号化するとき、Laravelはまず現在のキーを試します。現在のキーで復号化に失敗した場合、Laravelは値を復号化できるキーが見つかるまで、以前のキーをすべて試します。

この優しい復号化アプローチにより、暗号化キーがローテーションされていても、ユーザーはアプリケーションを中断なく使い続けられます。

Laravelでの暗号化の詳細は、[暗号化のドキュメント](/docs/{{version}}/encryption)をチェックしてください。

<a name="automatic-password-rehashing"></a>
### パスワードの自動再ハッシュ

*パスワードの自動再ハッシュは、[Stephen Rees-Carter](https://github.com/valorin)が貢献しました。*

Laravelのデフォルトのパスワードハッシュアルゴリズムはbcryptです。bcryptハッシュの「ワークファクター」は、`config/hashing.php`設定ファイル、または`BCRYPT_ROUNDS`環境変数で調整できます。

通常、bcryptのワークファクターは、CPU／GPUの処理能力が向上するにつれて、増加させ時間をかける必要があります。アプリケーションのbcryptワークファクターを上げると、Laravelはユーザーがアプリケーションで認証する際に、ユーザーパスワードをうまく自動的に再ハッシュするようになります。

<a name="prompt-validation"></a>
### プロンプトバリデーション

*プロンプトのバリデーション統合は、[Andrea Marco Sartori](https://github.com/cerbero90)が貢献しました。*

[Laravel Prompts](/docs/{{version}}/prompts)は、美しくユーザーフレンドリーなフォームをコマンドラインアプリケーションへ追加するためのPHPパッケージで、プレースホルダテキストやバリデーションなどのブラウザライクな機能を備えています。

Laravel Promptsはクロージャによる入力バリデーションをサポートしています。

```php
$name = text(
    label: 'What is your name?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

しかし、多くの入力や複雑なバリデーションシナリオを扱う場合、これは面倒です。そこでLaravel11では、プロンプト入力のバリデーションを行う際に、Laravelの[バリデータ](/docs/{{version}}/validation)をフルに活用できるようになりました。

```php
$name = text('What is your name?', validate: [
    'name' => 'required|min:3|max:255',
]);
```

<a name="queue-interaction-testing"></a>
### キュー操作のテスト

*キュー操作のテストは、[Taylor Otwell](https://github.com/taylorotwell)が貢献しました。*

以前は、キュー投入したジョブがリリースされたり、削除されたり、失敗したりを手作業でテストしようとすると面倒で、カスタムキューFakeやスタブを定義する必要がありました。しかし、Laravel11では、`withFakeQueueInteractions`メソッドを使用することで、これらのキューのやり取りを簡単にテストできます。

```php
use App\Jobs\ProcessPodcast;

$job = (new ProcessPodcast)->withFakeQueueInteractions();

$job->handle();

$job->assertReleased(delay: 30);
```

キューに投入したジョブのテストの詳細は、[キューのドキュメント](/docs/{{version}}/queues#testing)を参照してください。

<a name="new-artisan-commands"></a>
### 新しいArtisanコマンド

*クラス生成Artisanコマンドは、[Taylor Otwell](https://github.com/taylorotwell)が貢献しました。*

新しいArtisanコマンドを追加し、クラス、列挙型、インターフェイス、トレイトを素早く作成できるようになりました。

```shell
php artisan make:class
php artisan make:enum
php artisan make:interface
php artisan make:trait
```

<a name="model-cast-improvements"></a>
### モデルのキャストの向上

*モデルのキャストの向上は、[Nuno Maduro](https://github.com/nunomaduro)が貢献しました。*

Laravel11では、モデルのキャストをプロパティではなくメソッドで定義できるようになりました。これにより、特に引数を持つキャストを使用する場合に、合理的で流暢なキャスト定義が可能になりました。

    /**
     * キャストする属性を取得
     *
     * @return array<string, string>
     */
    protected function casts(): array
    {
        return [
            'options' => AsCollection::using(OptionCollection::class),
                      // AsEncryptedCollection::using(OptionCollection::class),
                      // AsEnumArrayObject::using(OptionEnum::class),
                      // AsEnumCollection::using(OptionEnum::class),
        ];
    }

属性のキャストの詳細は、[Eloquentのドキュメント](/docs/{{version}}/eloquent-mutators#attribute-casting)を参照してください。

<a name="the-once-function"></a>
### `once`関数

*`once`ヘルパは[Taylor Otwell](https://github.com/taylorotwell)*と*[Nuno Maduro](https://github.com/nunomaduro)が貢献しました。*

`once`ヘルパ関数は、指定したコールバックを実行し、リクエストの間、結果をメモリにキャッシュします。以降、同じコールバックで`once`関数を呼び出すと、以前にキャッシュした結果を返します。

    function random(): int
    {
        return once(function () {
            return random_int(1, 1000);
        });
    }

    random(); // 123
    random(); // 123 (cached result)
    random(); // 123 (cached result)

`once`ヘルパの詳細は、[ヘルパのドキュメント](/docs/{{version}}/helpers#method-once)を参照してください。

<a name="database-performance"></a>
### 内部メモリデータベースを使ったテストのパフォーマンス向上

*内部メモリデータベースを使ったテストのパフォーマンス向上は、[Anders Jenbo](https://github.com/AJenbo)が貢献しました。*

Laravel11では、テスト中に`:memory:` SQLiteデータベースを使用する際のスピードが大幅に向上しました。これを実現するために、LaravelはPHPのPDOオブジェクトへの参照を保持し、接続をまたいで再利用するようになりました。大抵、テスト時間の合計が半分になりました。

<a name="mariadb"></a>
### MariaDBのサポート向上

*MariaDBのサポート向上は、[Jonas Staudenmeir](https://github.com/staudenmeir)*と*[Julius Kiekbusch](https://github.com/Jubeki)が貢献しました。*

Laravel11では、MariaDBのサポートを改善しました。以前のLaravelリリースでは、LaravelのMySQLドライバ経由でMariaDBを使用できました。しかし、Laravel11では専用のMariaDBドライバが搭載され、このデータベースシステムのデフォルトが改善されました。

Laravelのデータベースドライバの詳細は、[データベースドキュメント](/docs/{{version}}/database)をチェックしてください。

<a name="inspecting-database"></a>
### データベースの調査とスキマ操作の向上

*データベースの調査とスキマ操作の向上は、[Hafez Divandari](https://github.com/hafezdivandari)が貢献しました。*

Laravel11では、カラムのネイティブな変更、リネーム、削除を含む、データベーススキーマの操作と検査メソッドを追加しています。さらに、高度な空間型、デフォルト以外のスキーマ名、テーブル、ビュー、カラム、インデックス、外部キーを操作するためのネイティブスキーマメソッドを提供しています。

    use Illuminate\Support\Facades\Schema;

    $tables = Schema::getTables();
    $views = Schema::getViews();
    $columns = Schema::getColumns('users');
    $indexes = Schema::getIndexes('users');
    $foreignKeys = Schema::getForeignKeys('users');
