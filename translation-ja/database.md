# データベース：準備

- [イントロダクション](#introduction)
    - [設定](#configuration)
    - [読み／書き接続](#read-and-write-connections)
- [SQLクエリの実行](#running-queries)
    - [複数データベース接続の使用](#using-multiple-database-connections)
    - [クエリイベントのリッスン](#listening-for-query-events)
    - [累積クエリ時間の監視](#monitoring-cumulative-query-time)
- [データベーストランザクション](#database-transactions)
- [データベースCLIへの接続](#connecting-to-the-database-cli)
- [データベースの調査](#inspecting-your-databases)
- [データベースの監視](#monitoring-your-databases)

<a name="introduction"></a>
## イントロダクション

最近のウェブアプリケーションは、ほとんどすべてデータベースを操作します。Laravelは、素のSQL、[Fluentクエリビルダ](/docs/{{version}}/queries)、[Eloquent ORM](/docs/{{version}}/eloquent)を使用し、サポートしている様々なデータベースの操作をとてもシンプルにしています。現在、Laravelは５つのデータベースのファーストパーティーサポートを提供しています。

<div class="content-list" markdown="1">

- MariaDB10.3以上 ([バージョンポリシー](https://mariadb.org/about/#maintenance-policy))
- MySQL5.7以上 ([バージョンポリシー](https://en.wikipedia.org/wiki/MySQL#Release_history))
- PostgreSQL10.0以上 ([バージョンポリシー](https://www.postgresql.org/support/versioning/))
- SQLite3.26.0以上
- SQL Server2017以上 ([バージョンポリシー](https://docs.microsoft.com/en-us/lifecycle/products/?products=sql-server))

</div>

さらに、MongoDBによって公式にメンテナンスされている`mongodb/laravel-mongodb`パッケージにより、MongoDBをサポートしています。詳しくは[Laravel MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/)ドキュメントをチェックしてください。

<a name="configuration"></a>
### 設定

Laravelのデータベースサービスの設定は、アプリケーションの`config/database.php`設定ファイルにあります。このファイルは、全データベース接続を定義し、デフォルトで使用する接続を指定できます。このファイル内のほとんどの設定オプションは、アプリケーションの環境変数の値によって決まります。Laravelがサポートしているデータベースシステムのほとんどの設定例をこのファイルに用意しています。

デフォルトのLaravelのサンプル[環境設定](/docs/{{version}}/configuration#environment-configuration)は、[Laravel Sail](/docs/{{version}}/sail)で使用できるようにしています。SailはローカルマシンでLaravelアプリケーションを開発するためのDocker環境です。ただし、ローカルデータベースの必要に合わせ、このデータベース設定は自由に変更してください。

<a name="sqlite-configuration"></a>
#### SQLite設定

SQLiteデータベースは、ファイルシステム上の単一ファイルです。ターミナルで`touch`コマンドを使用して新しいSQLiteデータベースを作成してください。（`touch database/database.sqlite`）データベースを作成したあと、このデータベースへの絶対パスを`DB_DATABASE`環境変数に指定することにより、簡単にこれを使用するように設定できます。

```ini
DB_CONNECTION=sqlite
DB_DATABASE=/absolute/path/to/database.sqlite
```

SQLite接続の外部キー制約は、デフォルトで有効になっています。これを無効にしたい場合は、 `DB_FOREIGN_KEYS`環境変数を`false`に設定してください。

```ini
DB_FOREIGN_KEYS=false
```

> [!NOTE]
> [Laravelインストーラ](/docs/{{version}}/installation#creating-a-laravel-project)を使用してLaravelアプリケーションを作成し、データベースにSQLiteを選択すると、Laravelは自動的に`database/database.sqlite`ファイルを作成し、デフォルトの[データベースマイグレーション](/docs/{{version}}/migrations)を実行します。

<a name="mssql-configuration"></a>
#### Microsoft SQLサーバ設定

Microsoft　SQL Serverデータベースを使用するには、`sqlsrv`、`pdo_sqlsrv`PHP拡張機能と、Microsoft SQL ODBCドライバなど必要な依存関係パッケージを確実にインストールしてください。

<a name="configuration-using-urls"></a>
#### URLを使用した設定

通常、データベース接続は、`host`、`database`、`username`、`password`などの複数の設定値により構成します。こうした設定値には、それぞれ対応する環境変数があります。つまり、運用サーバでデータベース接続情報を設定するときに、これら複数の環境変数を管理する必要があることを意味します。

AWSやHerokuなどの一部のマネージドデータベースプロバイダは、データベースのすべての接続情報を単一の文字カラムで含む単一のデータベース「URL」を提供しています。データベースURLの例は、次のようになります。

```html
mysql://root:password@127.0.0.1/forge?charset=UTF-8
```

こうしたURLは通常、標準のスキーマ規約に従います。

```html
driver://username:password@host:port/database?options
```

便利が良いように、Laravelは複数の設定オプションを使用してデータベースを構成する代わりに、こうしたURLをサポートしています。`url`(または対応する`DB_URL`環境変数)設定オプションが存在する場合は、データベース接続と接続情報を抽出するためにそれを使用します。

<a name="read-and-write-connections"></a>
### 読み／書き接続

SELECTステートメントに１つのデータベース接続を使用し、INSERT、UPDATE、およびDELETEステートメントに別のデータベース接続を使用したい場合があるでしょう。Laravelでは簡単に、素のクエリ、クエリビルダ、もしくはEloquent ORMのいずれを使用していても、常に適切な接続が使用されます。

読み取り/書き込み接続を設定する方法を確認するため、以下の例を見てみましょう。

    'mysql' => [
        'read' => [
            'host' => [
                '192.168.1.1',
                '196.168.1.2',
            ],
        ],
        'write' => [
            'host' => [
                '196.168.1.3',
            ],
        ],
        'sticky' => true,

        'database' => env('DB_DATABASE', 'laravel'),
        'username' => env('DB_USERNAME', 'root'),
        'password' => env('DB_PASSWORD', ''),
        'unix_socket' => env('DB_SOCKET', ''),
        'charset' => env('DB_CHARSET', 'utf8mb4'),
        'collation' => env('DB_COLLATION', 'utf8mb4_unicode_ci'),
        'prefix' => '',
        'prefix_indexes' => true,
        'strict' => true,
        'engine' => null,
        'options' => extension_loaded('pdo_mysql') ? array_filter([
            PDO::MYSQL_ATTR_SSL_CA => env('MYSQL_ATTR_SSL_CA'),
        ]) : [],
    ],

設定配列には、`read`、`write`、`sticky`の３キーが追加されていることに注目してください。`read`キーと`write`キーには、単一のキーとして`host`を含む配列値があります。`read`および`write`接続の残りのデータベースオプションは、メインの`mysql`設定配列からマージします。

メインの`mysql`配列の値をオーバーライドする場合にのみ、`read`配列と`write`配列へ項目を配置する必要があります。したがって、この場合、`192.168.1.1`は「読み取り」接続のホストとして使用し、`192.168.1.3`は「書き込み」接続に使用します。データベースの接続情報、プレフィックス、文字セット、およびメインの`mysql`配列内の他のすべてのオプションは、両方の接続で共有します。`host`設定配列に複数の値が存在する場合、リクエストごとランダムにデータベースホストを選択します。

<a name="the-sticky-option"></a>
#### `sticky`オプション

`sticky`オプションは、現在のリクエストサイクル中にデータベースへ書き込まれたレコードをすぐに読み取るため使用する**オプション**値です。`sticky`オプションが有効になっており、現在のリクエストサイクル中にデータベースへ対し「書き込み」操作が実行された場合、それ以降の「読み取り」操作では「書き込み」接続が使用されます。これにより、要求サイクル中に書き込まれたデータを、同じ要求中にデータベースからすぐに読み戻すことができます。これがアプリケーションにとって望ましい動作であるかどうかを判断するのは使用者の皆さん次第です。

<a name="running-queries"></a>
## SQLクエリの実行

データベース接続を設定したら、`DB`ファサードを使用してクエリを実行できます。`DB`ファサードは、クエリのタイプごとに`select`、`update`、`insert`、`delete`、`statement`メソッドを提供します。

<a name="running-a-select-query"></a>
#### SELECTクエリの実行

基本的なSELECTクエリを実行するには、`DB`ファサードで`select`メソッドを使用します。

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Support\Facades\DB;
    use Illuminate\View\View;

    class UserController extends Controller
    {
        /**
         * アプリケーションの全ユーザーのリストを表示
         */
        public function index(): View
        {
            $users = DB::select('select * from users where active = ?', [1]);

            return view('user.index', ['users' => $users]);
        }
    }

`select`メソッドの最初の引数はSQLクエリであり、２番目の引数はクエリにバインドする必要のあるパラメータバインディングです。通常、これらは `where`句の制約の値です。パラメータバインディングは、SQLインジェクションに対する保護を提供します。

`select`メソッドは常に結果の`配列`を返します。配列内の各結果は、データベースのレコードを表すPHPの`stdClass`オブジェクトになります。

    use Illuminate\Support\Facades\DB;

    $users = DB::select('select * from users');

    foreach ($users as $user) {
        echo $user->name;
    }

<a name="selecting-scalar-values"></a>
#### スカラー値のセレクト

データベースへの問い合わせの結果が、単一のスカラー値であることがあります。Laravelでは、クエリのスカラー結果をレコードオブジェクトから取得する代わりに、`scalar`メソッドを使用し、この値を直接取得できます。

    $burgers = DB::scalar(
        "select count(case when food = 'burger' then 1 end) as burgers from menu"
    );

<a name="selecting-multiple-result-sets"></a>
#### 複数のクエリ結果セットのセレクト

アプリケーションが複数の結果セットを返すストアドプロシージャを呼び出す場合、`selectResultSets`メソッドを使用して、返される全ての結果セットを取得できます：

    [$options, $notifications] = DB::selectResultSets(
        "CALL get_user_options_and_notifications(?)", $request->user()->id
    );

<a name="using-named-bindings"></a>
#### 名前付きバインディングの使用

パラメータバインディングを表すために`?`を使用する代わりに、名前付きバインディングを使用してクエリを実行できます。

    $results = DB::select('select * from users where id = :id', ['id' => 1]);

<a name="running-an-insert-statement"></a>
#### INSERT文の実行

`insert`ステートメントを実行するには、`DB`ファサードで`insert`メソッドを使用します。`select`と同様に、このメソッドはSQLクエリを最初の引数に取り、バインディングを２番目の引数に取ります。

    use Illuminate\Support\Facades\DB;

    DB::insert('insert into users (id, name) values (?, ?)', [1, 'Marc']);

<a name="running-an-update-statement"></a>
#### 更新文の実行

データベース内の既存のレコードを更新するには、`update`メソッドを使用する必要があります。メソッドは実行の影響を受けた行数を返します。

    use Illuminate\Support\Facades\DB;

    $affected = DB::update(
        'update users set votes = 100 where name = ?',
        ['Anita']
    );

<a name="running-a-delete-statement"></a>
#### DELETE文の実行

データベースからレコードを削除するには、`delete`メソッドを使用する必要があります。`update`と同様に、メソッドは影響を受けた行数を返します。

    use Illuminate\Support\Facades\DB;

    $deleted = DB::delete('delete from users');

<a name="running-a-general-statement"></a>
#### 一般的な文の実行

一部のデータベース操作文は値を返しません。こうしたタイプの操作では、`DB`ファサードで`statement`メソッドを使用します。

    DB::statement('drop table users');

<a name="running-an-unprepared-statement"></a>
#### プリペアドではない文の実行

値をバインドせずSQL文を実行したい場合があります。それには、`DB`ファサードの`unprepared`メソッドを使用します。

    DB::unprepared('update users set votes = 100 where name = "Dries"');

> [!WARNING]
> プリペアドではない文はパラメータをバインドしないため、SQLインジェクションに対して脆弱である可能性があります。プリペアドではない文内では、ユーザーによる値のコントロールを許可しないでください。

<a name="implicit-commits-in-transactions"></a>
#### 暗黙のコミット

トランザクション内で`DB`ファサードの`statement`および`unprepared`メソッドを使用する場合、[暗黙のコミット](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)を引き起こすステートメントを回避するように注意する必要があります。これらのステートメントにより、データベースエンジンはトランザクション全体を間接的にコミットし、Laravelはデータベースのトランザクションレベルを認識しなくなります。このようなステートメントの例は、データベーステーブルの作成です。

    DB::unprepared('create table a (col varchar(1) null)');

暗黙的なコミットを引き起こす、[すべてのステートメントのリスト](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)は、MySQLのマニュアルを参照してください。

<a name="using-multiple-database-connections"></a>
### 複数データベース接続の使用

アプリケーションが`config/database.php`設定ファイルで複数の接続を定義している場合、`DB`ファサードが提供する`connection`メソッドを使い、各接続にアクセスできます。`connection`メソッドに渡す接続名は、`config/database.php`設定ファイルにリストしている接続、または実行時に`config`ヘルパを使用して設定した接続の１つに対応させる必要があります。

    use Illuminate\Support\Facades\DB;

    $users = DB::connection('sqlite')->select(/* ... */);

接続インスタンスで`getPdo`メソッドを使用し、接続の基になる素のPDOインスタンスにアクセスできます。

    $pdo = DB::connection()->getPdo();

<a name="listening-for-query-events"></a>
### クエリイベントのリッスン

アプリケーションが実行するSQLクエリごとに呼び出すクロージャを指定する場合は、`DB`ファサードの`listen`メソッドを使用します。このメソッドは、クエリのログ記録やデバッグに役立ちます。クエリリスナクロージャは、[サービスプロバイダ](/docs/{{version}}/providers)の`boot`メソッドで登録します。

    <?php

    namespace App\Providers;

    use Illuminate\Database\Events\QueryExecuted;
    use Illuminate\Support\Facades\DB;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * アプリケーションの全サービスの登録
         */
        public function register(): void
        {
            // ...
        }

        /**
         * アプリケーションの全サービスの起動初期処理
         */
        public function boot(): void
        {
            DB::listen(function (QueryExecuted $query) {
                // $query->sql;
                // $query->bindings;
                // $query->time;
                // $query->toRawSql();
            });
        }
    }

<a name="monitoring-cumulative-query-time"></a>
### 累積クエリ時間の監視

モダンなWebアプリケーションの一般的なパフォーマンスボトルネックは、データベースのクエリに費やす時間の長さです。幸いなことに、Laravelでは、単一リクエスト中でデータベースのクエリに時間がかかりすぎる場合、指定クロージャやコールバックを呼び出せます。これを使うには、`whenQueryingForLongerThan`メソッドへ、クエリ時間しきい値(ミリ秒単位)とクロージャを指定します。このメソッドは、[サービスプロバイダ](/docs/{{version}}/providers)の`boot`メソッドで呼び出します

    <?php

    namespace App\Providers;

    use Illuminate\Database\Connection;
    use Illuminate\Support\Facades\DB;
    use Illuminate\Support\ServiceProvider;
    use Illuminate\Database\Events\QueryExecuted;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * アプリケーションの全サービスの登録
         */
        public function register(): void
        {
            // ...
        }

        /**
         * アプリケーションの全サービスの起動初期処理
         */
        public function boot(): void
        {
            DB::whenQueryingForLongerThan(500, function (Connection $connection, QueryExecuted $event) {
                // 開発チームへ通知を送る…
            });
        }
    }

<a name="database-transactions"></a>
## データベーストランザクション

`DB`ファサードが提供する`transaction`メソッドを使用して、データベーストランザクション内で一連の操作を実行できます。トランザクションクロージャ内で例外が投げられた場合、トランザクションを自動的にロールバックし、その例外を再度投げます。クロージャが正常に実行されると、トランザクションを自動的にコミットします。`transaction`メソッドの使用中にロールバックやコミットを手作業で実行する心配はありません。

    use Illuminate\Support\Facades\DB;

    DB::transaction(function () {
        DB::update('update users set votes = 1');

        DB::delete('delete from posts');
    });

<a name="handling-deadlocks"></a>
#### デッドロックの処理

`transaction`メソッドは、デッドロックが発生したときのトランザクション再試行回数をオプションとして、２番目の引数に取ります。試行回数を終えた場合は、例外を投げます。

    use Illuminate\Support\Facades\DB;

    DB::transaction(function () {
        DB::update('update users set votes = 1');

        DB::delete('delete from posts');
    }, 5);

<a name="manually-using-transactions"></a>
#### トランザクションを手作業で使用

トランザクションを手作業で開始し、ロールバックとコミットを自分で完全にコントロールしたい場合は、`DB`ファサードが提供する`beginTransaction`メソッドを使用します。

    use Illuminate\Support\Facades\DB;

    DB::beginTransaction();

`rollBack`メソッドにより、トランザクションをロールバックできます。

    DB::rollBack();

`commit`メソッドにより、トランザクションをコミットできます。

    DB::commit();

> [!NOTE]
> `DB`ファサードのトランザクションメソッドは、[クエリビルダ](/docs/{{version}}/queries)と[Eloquent ORM](/docs/{{version}}/eloquent)の両方のトランザクションを制御します。

<a name="connecting-to-the-database-cli"></a>
## データベースCLIへの接続

データベースのCLIに接続する場合は、`db` Artisanコマンドを使用します。

```shell
php artisan db
```

必要に応じて、データベース接続名を指定して、デフォルト接続以外のデータベースへ接続できます。

```shell
php artisan db mysql
```

<a name="inspecting-your-databases"></a>
## データベースの調査

Artisanの`db:show`コマンドと、`db:table`コマンドを使用すると、データベースのサイズや種類、開いている接続の数、テーブルの概要など、データベースと関連するテーブルの貴重な情報を取得できます。データベースの概要を確認するには、`db:show`コマンドを使用します。

```shell
php artisan db:show
```

このコマンドで、どのデータベース接続を検査するかを`--database`オプションを使い接続名で指定できます。

```shell
php artisan db:show --database=pgsql
```

もし、このコマンドの出力にテーブルの行数とデータベースのビューの詳細を含めたい場合は、それぞれ`--counts`と`--views`オプションを指定します。大きなデータベースでは、行数やビューの詳細の取得に時間がかかることがあります。

```shell
php artisan db:show --counts --views
```

加えて、以下の`Schema`メソッドを使ってデータベースを調べられます。

    use Illuminate\Support\Facades\Schema;

    $tables = Schema::getTables();
    $views = Schema::getViews();
    $columns = Schema::getColumns('users');
    $indexes = Schema::getIndexes('users');
    $foreignKeys = Schema::getForeignKeys('users');

アプリケーションのデフォルト接続ではないデータベース接続を調べたい場合は、`connection`メソッドを使用してください。

    $columns = Schema::connection('sqlite')->getColumns('users');

<a name="table-overview"></a>
#### テーブルの概要

データベース内の個々のテーブルの概要を知りたい場合には、`db:table` Artisanコマンドを実行してください。このコマンドは、カラム、タイプ、属性、キー、インデックスを含む、データベーステーブルの一般的な概要を表示します。

```shell
php artisan db:table users
```

<a name="monitoring-your-databases"></a>
## データベースの監視

`DB:monitor` Artisanコマンドを使用すると、管理しているデータベースが指定した数以上の接続を開いている場合、Laravelに`Illuminate\Database\Events\DatabaseBusy`イベントを発行するように指示できます。

利用するには、`db:monitor`コマンドを[毎分実行](/docs/{{version}}/scheduling)するようにタスクスケジュールしてください。このコマンドは、監視したいデータベース接続設定の名前と、イベントを発行するまで許容するオープン中の最大接続数を引数に取ります。

```shell
php artisan db:monitor --databases=mysql,pgsql --max=100
```

このコマンドをスケジューリングするだけでは、オープン接続数を警告する通知を発行できません。コマンドが閾値を超えるオープン接続数を持つデータベースに遭遇すると、`DatabaseBusy`イベントがディスパッチされます。アプリケーションの`AppServiceProvider`内でこのイベントをリッスンして、あなたや開発チームに通知を送る必要があります。

```php
use App\Notifications\DatabaseApproachingMaxConnections;
use Illuminate\Database\Events\DatabaseBusy;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Notification;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Event::listen(function (DatabaseBusy $event) {
        Notification::route('mail', 'dev@example.com')
                ->notify(new DatabaseApproachingMaxConnections(
                    $event->connectionName,
                    $event->connections
                ));
    });
}
```
