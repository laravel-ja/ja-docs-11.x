# アップグレードガイド

- [10.xから11.xへのアップグレード](#upgrade-11.0)

<a name="high-impact-changes"></a>
## 影響度の高い変更

<div class="content-list" markdown="1">

- [依存パッケージの更新](#updating-dependencies)
- [アプリケーション構造](#application-structure)
- [浮動小数点タイプ](#floating-point-types)
- [カラムの変更](#modifying-columns)
- [SQLite最小バージョン](#sqlite-minimum-version)
- [Sanctumアップデート](#updating-sanctum)

</div>

<a name="medium-impact-changes"></a>
## 影響度が中程度の変更

<div class="content-list" markdown="1">

- [Carbon3](#carbon-3)
- [パスワードの再ハッシュ](#password-rehashing)
- [毎秒のレート制限](#per-second-rate-limiting)

</div>

<a name="low-impact-changes"></a>
## 影響度の低い変更

<div class="content-list" markdown="1">

- [Doctrine DBALの削除](#doctrine-dbal-removal)
- [Eloquentモデルの`casts`メソッド](#eloquent-model-casts-method)
- [空間タイプ](#spatial-types)
- [SpatieのOnceパッケージ](#spatie-once-package)
- [`Enumerable`契約](#the-enumerable-contract)
- [`UserProvider`契約](#the-user-provider-contract)
- [`Authenticatable`契約](#the-authenticatable-contract)

</div>

<a name="upgrade-11.0"></a>
## 10.xから11.xへのアップグレード

<a name="estimated-upgrade-time-??-minutes"></a>
#### アップグレード見積もり時間：１５分

> [!NOTE]
> 私たちは、互換性のない変更を可能な限りすべて文書化するよう努力しています。これらの互換性のない変更の一部はフレームワークの不明瞭な部分であり、こうした変更の一部が実際にあなたのアプリケーションに影響を与える可能性があります。時間を節約したいですか？[Laravel Shift](https://laravelshift.com/) を使用すると、アプリケーションのアップグレードを自動化できます。

<a name="updating-dependencies"></a>
### 依存パッケージのアップデート

**影響の可能性： 高い**

#### PHP8.2.0必須

Laravelは、PHP8.2.0以上が必要になりました。

#### curl7.34.0必須

LaravelのHTTPクライアントは、curl7.34.0以上が必要になりました。

#### Composerの依存パッケージ

アプリケーションの`composer.json`ファイルにある、以下の依存パッケージを更新してください。

<div class="content-list" markdown="1">

- `laravel/framework`を`^11.0`へ
- `nunomaduro/collision`を`^8.1`へ
- `laravel/breeze`を`^2.0`へ（インストール済みの場合）
- `laravel/cashier`を`^15.0`へ（インストール済みの場合）
- `laravel/dusk`を`^8.0`へ（インストール済みの場合）
- `laravel/jetstream`を`^5.0`へ（インストール済みの場合）
- `laravel/octane`を`^2.3`へ（インストール済みの場合）
- `laravel/passport`を`^12.0`へ（インストール済みの場合）
- `laravel/sanctum`を`^4.0`へ（インストール済みの場合）
- `laravel/scout`を`^10.0`へ（インストール済みの場合）
- `laravel/spark-stripe`を`^5.0`へ（インストール済みの場合）
- `laravel/telescope`を`^5.0`へ（インストール済みの場合）
- `inertiajs/inertia-laravel`を`^1.0`へ（インストール済みの場合）

</div>

アプリケーションがLaravel Cashier Stripe、Passport、Sanctum、Spark Stripe、Telescopeを使用している場合、それらのマイグレーションをアプリケーションへリソース公開する必要があります。Cashier Stripe、Passport、Sanctum、Spark Stripe、Telescopeは、**それ自体のマイグレーションディレクトリから自動的にマイグレーションをロードしなくなりました**。そのため、以下のコマンドを実行して、アプリケーションへマイグレーションをリソース公開する必要があります。

```bash
php artisan vendor:publish --tag=cashier-migrations
php artisan vendor:publish --tag=passport-migrations
php artisan vendor:publish --tag=sanctum-migrations
php artisan vendor:publish --tag=spark-migrations
php artisan vendor:publish --tag=telescope-migrations
```

さらに、これらの各パッケージのアップグレードガイドを確認して、追加の変更点を確実に把握してください。

- [Laravel Cashier Stripe](#cashier-stripe)
- [Laravel Passport](#passport)
- [Laravel Sanctum](#sanctum)
- [Laravel Spark Stripe](#spark-stripe)
- [Laravel Telescope](#telescope)

Laravelインストーラを自分でインストールした場合は、Composer経由でインストーラを更新してください：

```bash
composer global require laravel/installer:^5.6
```

最後に、Laravelがこのパッケージに依存しなくなったので、`doctrine/dbal` Composer依存パッケージを削除してください。

<a name="application-structure"></a>
### アプリケーション構造

Laravel11では、新しいデフォルトのアプリケーション構造が導入され、デフォルトのファイルが少なくなっています。つまり、新しいLaravelアプリケーションには、サービスプロバイダ、ミドルウェア、設定ファイルの数が少なくなっています。

しかし、Laravel10アプリケーションをLaravel11にアップグレードするときに、アプリケーション構造の移行を試みることは**お勧めしません**。Laravel11はLaravel10のアプリケーション構造もサポートするように注意深く調整してあります。

<a name="authentication"></a>
### 認証

<a name="password-rehashing"></a>
#### パスワードの再ハッシュ

Laravel11は、パスワードが最後にハッシュ化されてからハッシュ化アルゴリズムの「ストレッチング（work factor）」が更新された場合、認証時にユーザーのパスワードを自動的に再ハッシュ化します。

通常、これによってアプリケーションが中断されることはありませんが、アプリケーションの`config/hashing.php`構成ファイルに`rehash_on_login`オプションを追加と、この動作を無効にできます。

    'rehash_on_login' => false,

<a name="the-user-provider-contract"></a>
#### `UserProvider`契約

**影響の可能性： 低い**

`Illuminate\Contracts\Auth\UserProvider`に新しい`rehashPasswordIfRequired`メソッドを追加しました。このメソッドは、アプリケーションのハッシュアルゴリズムのワークファクタが変更されたときに、ユーザーのパスワードを再ハッシュしてストレージに保存する役割を担っています。

アプリケーションやパッケージがこのインタフェースを実装するクラスを定義している場合、新しい`rehashPasswordIfRequired`メソッドを追加で実装する必要があります。参照になる実装は、`Illuminate\Auth\EloquentUserProvider`クラス内にあります。

```php
public function rehashPasswordIfRequired(Authenticatable $user, array $credentials, bool $force = false);
```

<a name="the-authenticatable-contract"></a>
#### `Authenticatable`契約

**影響の可能性： 低い**

`Illuminate\Contracts\Auth\Authenticatable`契約に、新しく`getAuthPasswordName`メソッドを追加しました。このメソッドは、認証可能エンティティのパスワードカラムの名前を返す役割を負っています。

アプリケーションやパッケージがこのインターフェイスを実装したクラスを定義している場合は、新しい `getAuthPasswordName`メソッドを実装に追加する必要があります：

```php
public function getAuthPasswordName()
{
    return 'password';
}
```

Laravelに含まれるデフォルトの`User`モデルでは、このメソッドが`Illuminate\Auth\Authenticatable`トレイトに含まれているため、自動的にこのメソッドを受け取ります。

<a name="the-authentication-exception-class"></a>

#### `AuthenticationException`クラス

**影響の可能性： かなり低い**

`Illuminate\Auth\AuthenticationException`クラスの`redirectTo`メソッドは、最初の引数に`Illuminate\Http\Request`インスタンスが必要になりました。この例外を手作業でキャッチし、`redirectTo`メソッドを呼び出している場合は、それに応じてコードを更新してください。

```php
if ($e instanceof AuthenticationException) {
    $path = $e->redirectTo($request);
}
```

<a name="cache"></a>
### キャッシュ

<a name="cache-key-prefixes"></a>
#### Cache Key Prefixes

**影響の可能性： かなり低い**

以前は、DynamoDB、Memcached、Redisのキャッシュストアで、キャッシュキーの接頭辞を定義した場合、Laravelは接頭辞へ`:`を追加していました。Laravel11では、キャッシュキーの接頭辞へ`:`を追加しなくなりました。以前の接頭辞動作を維持したい場合は、キャッシュキーの接頭辞の最後に手作業で`:`を追加してください。

<a name="collections"></a>
### コレクション

<a name="the-enumerable-contract"></a>
#### `Enumerable`契約

**影響の可能性： 低い**

`Illuminate\Support\Enumerable`契約の`dump`メソッドを更新し、`...$args`可変長引数を取るようにしました。このインタフェースを実装している場合は、これに合わせて実装を更新する必要があります。

```php
public function dump(...$args);
```

<a name="database"></a>
### データベース

<a name="sqlite-minimum-version"></a>
#### SQLite 3.35.0+

**影響の可能性： 高い**

アプリケーションでSQLiteデータベースを使用する場合は、SQLite3.35.0以上が必要です。

<a name="eloquent-model-casts-method"></a>
#### Eloquentモデルの`casts`メソッド

**影響の可能性： 低い**

Eloquentモデルのベースクラスは、属性キャストの定義をサポートするため、`casts`メソッドを定義するようになりました。もし、アプリケーションのモデルで`casts`リレーションを定義している場合、Eloquentベースモデルクラスの`casts`メソッドと衝突する可能性があります。

<a name="modifying-columns"></a>
#### カラムの変更

**影響の可能性： 高い**

カラムを変更する場合、変更後もカラム定義を維持したいすべての修飾子を明示的に含める必要があります。足りない属性は削除されます。例えば、`unsigned`属性、`default`属性、`comment`属性を保持するためには、カラムを変更する際にそれぞれの修飾子を明示的に呼び出す必要があります。

例えば、`unsigned`、`default`、`comment`属性を持つ `votes`カラムを作成するマイグレーションがあるとします。

```php
Schema::create('users', function (Blueprint $table) {
    $table->integer('votes')->unsigned()->default(1)->comment('The vote count');
});
```

その後で、カラムを`nullable`に変更するマイグレーションを書いてみましょう。

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')->nullable()->change();
});
```

Laravel10では、このマイグレーションはカラムの`unsigned`、`default`、`comment`属性を保持していました。しかし、Laravel11からマイグレーションにはカラムに以前定義していた属性もすべて含める必要があります。含めなければ、それらを削除します。

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')
        ->unsigned()
        ->default(1)
        ->comment('The vote count')
        ->nullable()
        ->change();
});
```

`change`メソッドはカラムのインデックスを変更しません。そのため、カラムを変更する際には、インデックス修飾子を使って明示的にインデックスを追加したり削除したりしてください。

```php
// インデックス追加
$table->bigIncrements('id')->primary()->change();

// インデックス削除
$table->char('postal_code', 10)->unique(false)->change();
```

既存カラムの属性を保持するために、アプリケーション内の既存の全「変更（change）」マイグレーションを更新したくない場合は、単純に[マイグレーションを圧縮](/docs/{{version}}/migrations#squashing-migrations)してください。

```bash
php artisan schema:dump
```

一度マイグレーション圧縮したら、Laravelは保留中のマイグレーションを実行する前に、アプリケーションのスキーマファイルを使ってデータベースを「マイグレーション」します。

<a name="floating-point-types"></a>
#### 浮動小数点タイプ

**影響の可能性： 高い**

`double`と`float`のマイグレーションカラム型は、すべてのデータベースで一貫性があるように書き直されました。

`double`カラムタイプは、合計桁数と桁数(小数点以下の桁数) を含まない、`DOUBLE`と同等のカラムを作成するようになりました。そのため、`$total`と`$places`の引数を削除してください。

```php
$table->double('amount');
```

`float`カラム型は、合計桁数と桁数(小数点以下の桁数)のない`FLOAT`相当の列を作成するようになりましたが、格納サイズを4バイトの単精度列または8バイトの倍精度列として決定するためのオプションの`$precision`指定があります。したがって、`$total`と`$places`の引数を削除し、オプションの`$precision`をデータベースのドキュメントに従って目的の値に指定することができます。

```php
$table->float('amount', precision: 53);
```

`unsignedDecimal`、`unsignedDouble`、`unsignedFloat`メソッドは削除しました。これは、これらのカラム型のunsigned修飾子がMySQLによって廃止され、他のデータベースシステムでは標準化されなかったからです。ただし、これらのカラム型に対して廃止されたunsigned属性を引き続き使用する場合は、`unsigned`メソッドをカラムの定義にチェーンできます。

```php
$table->decimal('amount', total: 8, places: 2)->unsigned();
$table->double('amount')->unsigned();
$table->float('amount', precision: 53)->unsigned();
```

<a name="dedicated-mariadb-driver"></a>
#### MariaDB専用ドライバ

**影響の可能性： かなり低い**

Laravel11では、MariaDBデータベースに接続する際、常にMySQLドライバを利用しなくなり、MariaDB専用のデータベースドライバを追加しました。

アプリケーションがMariaDBデータベースへ接続する場合、将来のMariaDB固有の機能を利用するために、接続設定を新しい`mariadb`ドライバへ更新することができます。

    'driver' => 'mariadb',
    'url' => env('DB_URL'),
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '3306'),
    // ...

現在のところ、新しいMariaDBドライバは現在のMySQLドライバと同じように動作しますが、一つだけ例外があります。それは、`uuid`スキーマビルダメソッドは、`char(36)`カラムの代わりに、ネイティブのUUIDカラムを作成することです。

既存のマイグレーションが、`uuid`スキーマビルダメソッドを使用していて、新しい`mariadb`データベースドライバを使用する場合は、マイグレーションの`uuid`メソッドの呼び出しを`char`に更新して、ブレイキングチェンジや想定外の振る舞いを防いでください。

```php
Schema::table('users', function (Blueprint $table) {
    $table->char('uuid', 36);

    // ...
});
```

<a name="spatial-types"></a>
#### 空間タイプ

**影響の可能性： 低い**

データベースマイグレーションの空間カラムタイプは、すべてのデータベースに渡り一貫性があるように書き換えました。そのため、`point`,`lineString`,`polygon`,`geometryCollection`,`multiPoint`,`multiLineString`,`multiPolygon`,`multiPolygonZ`メソッドをマイグレーションから削除し、代わりに`geometry`または`geography`メソッドを使用してください。

```php
$table->geometry('shapes');
$table->geography('coordinates');
```

MySQL、MariaDB、PostgreSQLのカラムに格納する値の型や空間参照システムの識別子を明示的に制限するには、`subtype`と`srid`をメソッドへ渡します。

```php
$table->geometry('dimension', subtype: 'polygon', srid: 0);
$table->geography('latitude', subtype: 'point', srid: 4326);
```

これに伴い、PostgreSQL文法の`isGeometry`および`projection`カラム修飾子を削除しました。

<a name="doctrine-dbal-removal"></a>
#### Doctrine DBALの削除

**影響の可能性： 低い**

以下のリスト中のDoctrine DBAL関連のクラスとメソッドを削除しました。Laravelはこのパッケージに依存しなくなり、以前はカスタム型を必要としていたさまざまなカラム型の適切な作成と変更のために、Doctrineのカスタム型を登録する必要はなくなりました。

<div class="content-list" markdown="1">

- `Illuminate\Database\Schema\Builder::$alwaysUsesNativeSchemaOperationsIfPossible` class property
- `Illuminate\Database\Schema\Builder::useNativeSchemaOperationsIfPossible()` method
- `Illuminate\Database\Connection::usingNativeSchemaOperations()` method
- `Illuminate\Database\Connection::isDoctrineAvailable()` method
- `Illuminate\Database\Connection::getDoctrineConnection()` method
- `Illuminate\Database\Connection::getDoctrineSchemaManager()` method
- `Illuminate\Database\Connection::getDoctrineColumn()` method
- `Illuminate\Database\Connection::registerDoctrineType()` method
- `Illuminate\Database\DatabaseManager::registerDoctrineType()` method
- `Illuminate\Database\PDO` directory
- `Illuminate\Database\DBAL\TimestampType` class
- `Illuminate\Database\Schema\Grammars\ChangeColumn` class
- `Illuminate\Database\Schema\Grammars\RenameColumn` class
- `Illuminate\Database\Schema\Grammars\Grammar::getDoctrineTableDiff()` method

</div>

さらに、アプリケーションの`database`設定ファイルで `dbal.types`を使い、カスタムDoctrineの型を登録する必要はなくなりました。

Doctrine DBALを使ってデータベースと関連テーブルを調べていたのであれば、代わりに Laravelの新しいネイティブスキーマメソッド (`Schema::getTables()`、`Schema::getColumns()`、`Schema::getIndexes()`、`Schema::getForeignKeys()`など) を使用してください。

<a name="deprecated-schema-methods"></a>
#### 非推奨のスキマメソッド

**影響の可能性： かなり低い**

Doctrineベースの非推奨であった`Schema::getAllTables()`、`Schema::getAllViews()`、`Schema::getAllTypes()`メソッドを削除し、新しいLaravelネイティブの`Schema::getTables()`、`Schema::getViews()`、`Schema::getTypes()`メソッドを使ってください。

PostgreSQLとSQLServerを使用する場合、新しいスキーマメソッドはどれも３つの部分からなる参照（例えば `database.schema.table`）を受け付けません。したがって、代わりに`connection()`を使用してデータベースを宣言する必要があります。

```php
Schema::connection('database')->hasTable('schema.table');
```

<a name="get-column-types"></a>
#### スキマビルダの`getColumnType()`メソッド

**影響の可能性： かなり低い**

`Schema::getColumnType()`メソッドは、Doctrine DBALと同等の型ではなく、常に指定カラムの実際の型を返すようになりました。

<a name="database-connection-interface"></a>
#### データベース接続インターフェイス

**影響の可能性： かなり低い**

`Illuminate\Database\ConnectionInterface`インターフェイスに、新しい`scalar`メソッドを追加しました。このインターフェイスの独自の実装を定義している場合は、`scalar`メソッドを実装に追加する必要があります。

```php
public function scalar($query, $bindings = [], $useReadPdo = true);
```

<a name="dates"></a>
### 日付

<a name="carbon-3"></a>
#### Carbon3

**影響の可能性： 中程度**

Laravel11は、Carbon2とCarbon3の両方をサポートしています。Carbonは、Laravelやエコシステム全体のパッケージで広く利用されている日付操作ライブラリです。Carbon3にアップグレードする場合、`diffIn*`メソッドが浮動小数点数を返すようになり、時差を示す負の値を返す可能性があることに注意してください。Carbonの [変更履歴](https://github.com/briannesbitt/Carbon/releases/tag/3.0.0)に、これやその他の変更点に対する処理の詳細が記載されています。

<a name="mail"></a>
### メール

<a name="the-mailer-contract"></a>
#### `Mailer`契約

**影響の可能性： かなり低い**

`Illuminate\Contracts\Mail\Mailer`契約に新しい`sendNow`メソッドを追加しました。この契約を実装しているアプリケーションやパッケージは、新しく`sendNow`メソッドを実装に追加してください。

```php
public function sendNow($mailable, array $data = [], $callback = null);
```

<a name="packages"></a>
### パッケージ

<a name="publishing-service-providers"></a>
#### アプリケーションへのサービスプロバイダのリソース公開

**影響の可能性： かなり低い**

アプリケーションの`app/Providers`ディレクトリに手作業でサービスプロバイダをリソース公開し、アプリケーションの`config/app.php`設定ファイルを手作業で修正してサービスプロバイダを登録していたLaravelパッケージを書いていた場合、新しい`ServiceProvider::addProviderToBootstrapFile`メソッドを利用するように、パッケージを更新する必要があります。

新しい Laravel11アプリケーションでは、`config/app.php`設定ファイル内に`providers`配列が存在しないため、`addProviderToBootstrapFile`メソッドは、リソース公開したサービスプロバイダを自動的にアプリケーションの`bootstrap/providers.php`ファイルへ追加します。

```php
use Illuminate\Support\ServiceProvider;

ServiceProvider::addProviderToBootstrapFile(Provider::class);
```

<a name="queues"></a>
### キュー

<a name="the-batch-repository-interface"></a>
#### `BatchRepository`インターフェイス

**影響の可能性： かなり低い**

`Illuminate\Bus\BatchRepository`インターフェイスに新しい`rollBack`メソッドを追加しました。自分のパッケージやアプリケーションでこのインタフェースを実装している場合、このメソッドを実装に追加する必要があります。

```php
public function rollBack();
```

<a name="synchronous-jobs-in-database-transactions"></a>
#### データベーストランザクション中の同期ジョブ

**影響の可能性： かなり低い**

以前は、キュー接続の`after_commit`設定オプションが`true`に設定されているか、ジョブに対して`afterCommit`メソッドが呼び出されているかに関係なく、同期ジョブ (`sync`キュードライバを使用するジョブ)すぐを即時に実行していました。

Laravel11では、同期キュージョブはキュー接続やジョブの「コミット後（after commit）」設定を尊重するようにしました。

<a name="rate-limiting"></a>
### レート制限

<a name="per-second-rate-limiting"></a>
#### 毎秒のレート制限

**影響の可能性： 中程度**

Laravel11では、これまで分単位のレート制限をサポートしていましたが、秒単位になりました。この変更に関連して、様々な潜在的なブレイキングチェンジに注意する必要があります。

`GlobalLimit`クラスのコンストラクタは、分単位の代わりに秒単位で引数を取るようにしました。このクラスはドキュメント化しておらず、通常アプリケーションで使用されることはないでしょう。

```php
new GlobalLimit($attempts, 2 * 60);
```

`Limit`クラスのコンストラクタは、分単位ではなく秒単位で指定できるようにしました。このクラスの使用法はすべて`Limit::perMinute`や`Limit::perSecond`などの静的コンストラクタに限定しています。しかし、このクラスを手作業でインスタンス化する場合は、このクラスのコンストラクタへ秒を指定するようにアプリケーションを更新する必要があります。

```php
new Limit($key, $attempts, 2 * 60);
```

`Limit`クラスの`decayMinutes`プロパティの名前を`decaySeconds`へ変更し、分単位ではなく秒単位になりました。

`Illuminate\Queue\Middleware\ThrottlesExceptions`と`Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis`クラスのコンストラクタが、分単位ではなく秒単位で受け付けるようにしました。

```php
new ThrottlesExceptions($attempts, 2 * 60);
new ThrottlesExceptionsWithRedis($attempts, 2 * 60);
```

<a name="cashier-stripe"></a>
### Cashier Stripe

<a name="updating-cashier-stripe"></a>
#### Cashier Stripeのアップデート

**影響の可能性： 高い**

Laravel11はCashier Stripe14.xをサポートしなくなりました。そのため、`composer.json`ファイルでアプリケーションのLaravel Cashier Stripeパッケージ依存関係を`^15.0`へ更新する必要があります。

Cashier Stripe15.0では、migrationディレクトリからマイグレーションを自動的にロードしなくなりました。代わりに、以下のコマンドを実行してCashier Stripeのマイグレーションをアプリケーションへリソース公開する必要があります。

```shell
php artisan vendor:publish --tag=cashier-migrations
```

その他の変更点については、[Cashier Stripeアップグレードガイド](https://github.com/laravel/cashier-stripe/blob/15.x/UPGRADE.md)を確認してください。

<a name="spark-stripe"></a>
### Spark (Stripe)

<a name="updating-spark-stripe"></a>
#### Spark Stripeアップデート

**影響の可能性： 高い**

Laravel11はLaravel Spark Stripe4.xをサポートしなくなったため、`composer.json`ファイルでアプリケーションのLaravel Spark Stripeパッケージ依存関係を`^5.0`へ更新する必要があります。

Spark Stripe5.0では、migrationsディレクトリからマイグレーションを自動的にロードしなくなりました。代わりに、以下のコマンドを実行してSpark Stripeのマイグレーションをアプリケーションへリソース公開する必要があります：

```shell
php artisan vendor:publish --tag=spark-migrations
```

その他の変更点については、[Spark Stripeアップグレードガイド](https://spark.laravel.com/docs/spark-stripe/upgrade.html)をご覧ください。

<a name="passport"></a>
### Passport

<a name="updating-telescope"></a>
#### Passportアップデート

**影響の可能性： 高い**

Laravel11はLaravel Passport11.xをサポートしなくなったため、`composer.json`ファイルでアプリケーションのLaravel Passportパッケージ依存関係を`^12.0`へ更新する必要があります。

Passport12.0では、自身のmigrationsディレクトリからマイグレーションを自動的にロードしなくなりました。代わりに、以下のコマンドを実行して、Passportのマイグレーションをアプリケーションへリソース公開する必要があります：

```shell
php artisan vendor:publish --tag=passport-migrations
```

さらに、パスワードグラントタイプはデフォルトで無効にしてあります。有効にするには、アプリケーションの `AppServiceProvider`の`boot`メソッドで`enablePasswordGrant`メソッドを呼び出してください。

    public function boot(): void
    {
        Passport::enablePasswordGrant();
    }

<a name="sanctum"></a>
### Sanctum

<a name="updating-sanctum"></a>
#### Sanctumアップデート

**影響の可能性： 高い**

Laravel11はLaravel Sanctum3.xをサポートしなくなったため、`composer.json`ファイルでアプリケーションのLaravel Sanctumパッケージ依存関係を`^4.0`へ更新する必要があります。

Sanctum 4.0は migrationsディレクトリから自動的にマイグレーションをロードしなくなりました。代わりに、次のコマンドを実行してSanctumのマイグレーションをアプリケーションへリソース公開する必要があります。

```shell
php artisan vendor:publish --tag=sanctum-migrations
```

次に、アプリケーションの`config/sanctum.php`設定ファイルで、`authenticate_session`, `encrypt_cookies`, `validate_csrf_token`ミドルウェアへの参照を以下のように更新してください。

    'middleware' => [
        'authenticate_session' => Laravel\Sanctum\Http\Middleware\AuthenticateSession::class,
        'encrypt_cookies' => Illuminate\Cookie\Middleware\EncryptCookies::class,
        'validate_csrf_token' => Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
    ],

<a name="telescope"></a>
### Telescope

<a name="updating-telescope"></a>
#### Telescopeアップデート

**影響の可能性： 高い**

Laravel11はLaravel Telescope4.xをサポートしなくなったため、`composer.json`ファイルでアプリケーションのLaravel Telescopeパッケージ依存関係を`^5.0`へ更新する必要があります。

Telescope5.0 では、migrationsディレクトリから自動的にマイグレーションをロードしなくなりました。代わりに、以下のコマンドを実行してTelescopeのマイグレーションをアプリケーションへリソース公開する必要があります：

```shell
php artisan vendor:publish --tag=telescope-migrations
```

<a name="spatie-once-package"></a>
### SpatieのOnceパッケージ

**影響の可能性： 中程度**

Laravel11は独自の[`once`関数](/docs/{{version}}/helpers#method-once)を提供し、指定クロージャを一度だけ実行することを保証します。そのため、アプリケーションが`spatie/once`パッケージへ依存している場合は、競合を避けるためにアプリケーションの `composer.json` ファイルからこれを削除する必要があります。

<a name="miscellaneous"></a>
### その他

また、`laravel/laravel` [GitHubリポジトリ](https://github.com/laravel/laravel)の変更点をご覧になることをお勧めします。これらの変更の多くは必須ではありませんが、これらのファイルをあなたのアプリケーションと同期させておくとよいでしょう。これらの変更のいくつかはこのアップグレードガイドでカバーしていますが、設定ファイルやコメントの変更など、他のものはカバーしていません。[GitHub 比較ツール](https://github.com/laravel/laravel/compare/10.x...11.x) を使えば簡単に変更点を確認することができ、どこのアップデートがあなたにとって重要かを選ぶことができます。
