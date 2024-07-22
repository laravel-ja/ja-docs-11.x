# マイグレーション

- [イントロダクション](#introduction)
- [マイグレーションの生成](#generating-migrations)
    - [マイグレーションの圧縮](#squashing-migrations)
- [マイグレーションの構造](#migration-structure)
- [マイグレーションの実行](#running-migrations)
    - [マイグレーションのロールバック](#rolling-back-migrations)
- [テーブル](#tables)
    - [テーブルの生成](#creating-tables)
    - [テーブルの更新](#updating-tables)
    - [テーブルのリネーム／削除](#renaming-and-dropping-tables)
- [カラム](#columns)
    - [カラムの生成](#creating-columns)
    - [利用可能なカラムタイプ](#available-column-types)
    - [カラム修飾子](#column-modifiers)
    - [カラムの変更](#modifying-columns)
    - [カラムのリネーム](#renaming-columns)
    - [カラムの削除](#dropping-columns)
- [インデックス](#indexes)
    - [インデックスの生成](#creating-indexes)
    - [インデックスのリネーム](#renaming-indexes)
    - [インデックスの削除](#dropping-indexes)
    - [外部キー制約](#foreign-key-constraints)
- [イベント](#events)

<a name="introduction"></a>
## イントロダクション

マイグレーションはデータベースのバージョン管理のようなもので、チームがアプリケーションのデータベーススキーマを定義および共有できるようにします。ソース管理から変更を取得した後に、ローカルデータベーススキーマにカラムを手作業で追加するようにチームメートに指示する必要があったことを経験していれば、データベースのマイグレーションにより解決される問題に直面していたのです。

Laravelの`Schema`[ファサード](/docs/{{version}}/facades)は、Laravelがサポートするすべてのデータベースシステムに対し、テーブルを作成、操作するために特定のデータベースに依存しないサポートを提供します。通常、マイグレーションはこのファサードを使用して、データベースのテーブルとカラムを作成および変更します。

<a name="generating-migrations"></a>
## マイグレーションの生成

`make:migration` [Artisanコマンド](/docs/{{version}}/artisan)を使用して、データベースマイグレーションを生成します。新しいマイグレーションは、`database/migrations`ディレクトリに配置されます。各マイグレーションファイル名には、Laravelがマイグレーションの順序を決定できるようにするタイムスタンプを含めています。

```shell移行
php artisan make:migration create_flights_table
```

Laravelは、マイグレーションの名前からテーブル名と新しいテーブルを作成しようとしているかを推測しようとします。Laravelがマイグレーション名からテーブル名を決定できる場合、Laravelは生成するマイグレーションファイルへ指定したテーブル名を事前に埋め込みます。それ以外の場合は、マイグレーションファイルのテーブルを手作業で指定してください。

生成するマイグレーションのカスタムパスを指定する場合は、`make:migration`コマンドを実行するときに`--path`オプションを使用します。指定したパスは、アプリケーションのベースパスを基準にする必要があります。

> [!NOTE]
> マイグレーションのスタブは[スタブのリソース公開](/docs/{{version}}/artisan#stub-customization)を使用してカスタマイズできます。

<a name="squashing-migrations"></a>
### マイグレーションの圧縮

アプリケーションを構築していくにつれ、時間の経過とともに段々と多くのマイグレーションが蓄積されていく可能性があります。これにより、`database/migrations`ディレクトリが数百のマイグレーションで肥大化する可能性があります。必要に応じて、マイグレーションを単一のSQLファイルに「圧縮」できます。利用するには、`schema:dump`コマンドを実行します。

```shell移行
php artisan schema:dump

# 現在のデータベーススキームをダンプし、既存のマイグレーションをすべて整理する..
php artisan schema:dump --prune
```

このコマンドを実行すると、Laravelはアプリケーションの`database/schema`ディレクトリへ、「スキーマ」ファイルを書き出します。スキーマファイルの名前は、データベース接続に対応します。これで、データベースをマイグレーションするとき、他のマイグレーションを実行していなければ、まずLaravelは使用しているデータベース接続のスキーマファイル内のSQLステートメントを実行します。スキーマファイルのSQL文を実行した後、Laravelはスキーマダンプ以外の、残りのマイグレーションを実行します。

アプリケーションのテストで、ローカル開発時に通常使用するものとは異なるデータベース接続を使用する場合、そのデータベース接続を使用してスキーマファイルをダンプし、テストでデータベースを構築できるようにする必要があります。この作業は、ローカル開発で通常使用するデータベース接続をダンプした後に行うとよいでしょう。

```shell移行
php artisan schema:dump
php artisan schema:dump --database=testing --prune
```

チームの新しい開発者がアプリケーションの初期データベース構造をすばやく作成できるようにするため、データベーススキーマファイルはソース管理にコミットすべきでしょう。

> [!WARNING]
> マイグレーションの圧縮は、MariaDB、MySQL、PostgreSQL、SQLiteデータベースでのみ利用可能で、データベースのコマンドラインクライアントを利用しています。

<a name="migration-structure"></a>
## マイグレーションの構造

マイグレーションクラスには、`up`と`down`の2つのメソッドを用意します。`up`メソッドはデータベースに新しいテーブル、カラム、またはインデックスを追加するために使用します。`down`メソッドでは、`up`メソッドによって実行する操作を逆にし、以前の状態へ戻す必要があります。

これらの両方のメソッド内で、Laravelスキーマビルダを使用して、テーブルを明示的に作成および変更できます。`Schema`ビルダで利用可能なすべてのメソッドを学ぶには、[ドキュメントをチェックしてください](#creating-tables)。たとえば、次のマイグレーションでは、`flights`テーブルが作成されます。

    <?php

    use Illuminate\Database\Migrations\Migration;
    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    return new class extends Migration
    {
        /**
         * マイグレーションの実行
         */
        public function up(): void
        {
            Schema::create('flights', function (Blueprint $table) {
                $table->id();
                $table->string('name');
                $table->string('airline');
                $table->timestamps();
            });
        }

        /**
         * マイグレーションを戻す
         */
        public function down(): void
        {
            Schema::drop('flights');
        }
    };

<a name="setting-the-migration-connection"></a>
#### マイグレーション接続の指定

マイグレーションがアプリケーションのデフォルトのデータベース接続以外のデータベース接続を操作する場合は、マイグレーションの`$connection`プロパティを設定する必要があります。

    /**
     * マイグレーションが使用するデータベース接続
     *
     * @var string
     */
    protected $connection = 'pgsql';

    /**
     * マイグレーションの実行
     */
    public function up(): void
    {
        // ...
    }

<a name="running-migrations"></a>
## マイグレーションの実行

未処理のマイグレーションをすべて実行するには、`migrate` Artisanコマンドを実行します。

```shell移行
php artisan migrate
```

これまでどのマイグレーションが実行されているかを確認したい場合は、`migrate:status` Artisanコマンドを使用してください。

```shell移行
php artisan migrate:status
```

マイグレーションが実行するSQL文を実際に実行せずに確認したい場合は、`migrate`コマンドに`--pretend`フラグを指定してください。

```shell移行
php artisan migrate --pretend
```

#### マイグレーションの排他実行

アプリケーションを複数のサーバに分散配置し、デプロイプロセスの一環としてマイグレーションを実行する場合、おそらく２つのサーバで同時にデータベースマイグレーションの実行は避けたいでしょう。これを避けるには、`migrate`コマンド実行時に、`isolated`オプションを使用してください。

`isolated`オプションを指定すると、Laravelはアプリケーションのキャッシュドライバを使用してアトミックロックを取得してから、マイグレーションを実行しようとします。ロックがかかっている間、他のすべての`migrate`コマンドの実行はできません。

```shell移行
php artisan migrate --isolated
```

> [!WARNING]
> この機能を利用するには、アプリケーションで`memcached`、`redis`、`dynamodb`、`database`、`file`、`array`キャッシュドライバをアプリケーションのデフォルトキャッシュドライバとして使用する必要があります。さらに、すべてのサーバから同じセントラルキャッシュサーバと通信する必要があります。

<a name="forcing-migrations-to-run-in-production"></a>
#### 本番環境におけるマイグレーション強制

一部のマイグレーション操作は破壊的です。つまり、データーが失われる可能性を持っています。本番データベースに対してこれらのコマンドを実行しないように保護するために、コマンドを実行する前に確認を求めるプロンプトが表示されます。プロンプトなしでコマンドを強制的に実行するには、`--force`フラグを使用します。

```shell移行
php artisan migrate --force
```

<a name="rolling-back-migrations"></a>
### マイグレーションのロールバック

最新のマイグレーション操作をロールバックするには、`rollback` Artisanコマンドを使用します。このコマンドは、マイグレーションの最後の「バッチ」をロールバックします。これは、複数のマイグレーションファイルを含む場合があります。

```shell移行
php artisan migrate:rollback
```

`rollback`コマンドに`step`オプションを提供することにより、限られた数のマイグレーションをロールバックできます。たとえば、次のコマンドは最後の5つのマイグレーションをロールバックします。

```shell移行
php artisan migrate:rollback --step=5
```

`rollback`コマンドで`batch`オプションを指定し、特定の 「バッチ」のマイグレーションをロールバックできます。このとき、`batch`オプションはアプリケーションの`migrations`データベーステーブル内のバッチの値に対応します。例えば、次のコマンドはバッチ3のすべてのマイグレーションをロールバックします。

 ```shell
php artisan migrate:rollback --batch=3
 ```

実際にマイグレーションを実行せず、そのマイグレーションが実行するSQL文を確認したい場合は、`--pretend`フラグを`migrate:rollback`コマンドへ指定します。

```shell移行
php artisan migrate:rollback --pretend
```

`migrate:reset`コマンドは、アプリケーションのすべてのマイグレーションをロールバックします。

```shell移行
php artisan migrate:reset
```

<a name="roll-back-migrate-using-a-single-command"></a>
#### ロールバック後マイグレーション実行

`migrate:refresh`コマンドは、すべてのマイグレーションをロールバックしてから、`migrate`コマンドを実行します。このコマンドは、データベース全体を効果的に再作成します。

```shell移行
php artisan migrate:refresh

# データベースを真新しくし、データベースの初期設定を実行する
php artisan migrate:refresh --seed
```

`refresh`コマンドに`step`オプションを指定し、特定の数のマイグレーションをロールバックしてから再マイグレーションできます。たとえば、次のコマンドは、最後の５マイグレーションをロールバックして再マイグレーションします。

```shell移行
php artisan migrate:refresh --step=5
```

<a name="drop-all-tables-migrate"></a>
#### 全テーブル削除後マイグレーション

`migrate:fresh`コマンドは、データベースからすべてのテーブルを削除したあと、`migrate`コマンドを実行します。

```shell移行
php artisan migrate:fresh

php artisan migrate:fresh --seed
```

デフォルトで`migrate:fresh`コマンドは、デフォルトのデータベース接続からテーブルを削除するだけす。しかし、`--database`オプションを使用すれば、マイグレートするデータベース接続を指定できます。データベース接続名は、アプリケーションの`database`[設定ファイル](/docs/{{version}}/configuration)で定義している接続と対応させる必要があります。

```shell移行
php artisan migrate:fresh --database=admin
```

> [!WARNING]
> `migrate:fresh`コマンドは、プレフィックスに関係なく、すべてのデータベーステーブルを削除します。このコマンドは、他のアプリケーションと共有されているデータベースで開発している場合は注意して使用する必要があります。

<a name="tables"></a>
## テーブル

<a name="creating-tables"></a>
### テーブルの生成

新しいデータベーステーブルを作成するには、`Schema`ファサードで`create`メソッドを使用します。`create`メソッドは２つの引数を取ります。１つ目はテーブルの名前で、２つ目は新しいテーブルを定義するために使用できる`Blueprint`オブジェクトを受け取るクロージャです。

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::create('users', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->string('email');
        $table->timestamps();
    });

テーブルを作成するときは、スキーマビルダの[カラムメソッド](#creating-columns)のいずれかを使用して、テーブルのカラムを定義します。

<a name="checking-for-table-column-existence"></a>
#### テーブル／カラムの存在の確認

`hasTable`メソッド、`hasColumn`メソッド、`hasIndex`メソッドを使って、テーブル、カラム、インデックスの存在を判定できます。

    if (Schema::hasTable('users')) {
        // "users"テーブルは存在していた
    }

    if (Schema::hasColumn('users', 'email')) {
        // "email"カラムを持つ"users"テーブルが存在していた
    }

    if (Schema::hasIndex('users', ['email'], 'unique')) {
        // The "users" table exists and has a unique index on the "email" column...
    }

<a name="database-connection-table-options"></a>
#### データベース接続とテーブルオプション

アプリケーションのデフォルトではないデータベース接続でスキーマ操作を実行する場合は、`connection`メソッドを使用します。

    Schema::connection('sqlite')->create('users', function (Blueprint $table) {
        $table->id();
    });

さらに、他のプロパティやメソッドを使用して、テーブル作成の他の部分を定義できます。`engine`プロパティはMariaDBとMySQLを使用するとき、テーブルのストレージエンジンを指定するために使用します。

    Schema::create('users', function (Blueprint $table) {
        $table->engine('InnoDB');

        // ...
    });

`charset`プロパティと`collat​​ion`プロパティはMariaDBとMySQLを使用するとき、作成するテーブルの文字セットと照合順序を指定するために使用します。

    Schema::create('users', function (Blueprint $table) {
        $table->charset('utf8mb4');
        $table->collation('utf8mb4_unicode_ci');

        // ...
    });

`temporary`メソッドを使用して、テーブルを「一時的」にする必要があることを示すことができます。一時テーブルは、現在の接続のデータベースセッションにのみ表示され、接続が閉じられると自動的に削除されます。

    Schema::create('calculations', function (Blueprint $table) {
        $table->temporary();

        // ...
    });

データベーステーブルに「コメント」を追加したい場合は、テーブルインスタンスに対して、`comment`メソッドを呼び出してください。テーブルコメントは現在、MariaDB、MySQL、PostgreSQLでのみサポートしています。

    Schema::create('calculations', function (Blueprint $table) {
        $table->comment('Business calculations');

        // ...
    });

<a name="updating-tables"></a>
### テーブルの更新

`Schema`ファサードの`table`メソッドを使用して、既存のテーブルを更新できます。`create`メソッドと同様に、`table`メソッドは２つの引数を取ります。テーブルの名前とテーブルにカラムやインデックスを追加するために使用できる`Blueprint`インスタンスを受け取るクロージャです。

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('users', function (Blueprint $table) {
        $table->integer('votes');
    });

<a name="renaming-and-dropping-tables"></a>
### テーブルのリネーム／削除

既存のデータベーステーブルの名前を変更するには、`rename`メソッドを使用します。

    use Illuminate\Support\Facades\Schema;

    Schema::rename($from, $to);

既存のテーブルを削除するには、`drop`または`dropIfExists`メソッドを使用できます。

    Schema::drop('users');

    Schema::dropIfExists('users');

<a name="renaming-tables-with-foreign-keys"></a>
#### 外部キーを使用したテーブルのリネーム

テーブルをリネームする前に、Laravelのテーブル名ベースの命名規約で外部キーを割り当てさせるのではなく、マイグレーションファイルでテーブルの外部キー制約の名前を明示的に指定していることを確認する必要があります。そうでない場合、外部キー制約名は古いテーブル名で参照されることになるでしょう。

<a name="columns"></a>
## カラム

<a name="creating-columns"></a>
### カラムの生成

`Schema`ファサードの`table`メソッドを使用して、既存のテーブルを更新できます。`create`メソッドと同様に、`table`メソッドは２つの引数を取ります。テーブルの名前とテーブルに列を追加するために使用できる`Illuminate\Database\Schema\Blueprint`インスタンスを受け取るクロージャです。

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('users', function (Blueprint $table) {
        $table->integer('votes');
    });

<a name="available-column-types"></a>
### 利用可能なカラムタイプ

スキーマビルダのBlueprintは、データベーステーブルに追加できるさまざまなタイプのカラムに対応する、多くのメソッドを提供しています。使用可能な各メソッドを以下に一覧します。

<style>
    .collection-method-list > p {
        columns: 10.8em 3; -moz-columns: 10.8em 3; -webkit-columns: 10.8em 3;
    }

    .collection-method-list a {
        display: block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    .collection-method code {
        font-size: 14px;
    }

    .collection-method:not(.first-collection-method) {
        margin-top: 50px;
    }
</style>

<div class="collection-method-list" markdown="1">

[bigIncrements](#column-method-bigIncrements)
[bigInteger](#column-method-bigInteger)
[binary](#column-method-binary)
[boolean](#column-method-boolean)
[char](#column-method-char)
[dateTimeTz](#column-method-dateTimeTz)
[dateTime](#column-method-dateTime)
[date](#column-method-date)
[decimal](#column-method-decimal)
[double](#column-method-double)
[enum](#column-method-enum)
[float](#column-method-float)
[foreignId](#column-method-foreignId)
[foreignIdFor](#column-method-foreignIdFor)
[foreignUlid](#column-method-foreignUlid)
[foreignUuid](#column-method-foreignUuid)
[geography](#column-method-geography)
[geometry](#column-method-geometry)
[id](#column-method-id)
[increments](#column-method-increments)
[integer](#column-method-integer)
[ipAddress](#column-method-ipAddress)
[json](#column-method-json)
[jsonb](#column-method-jsonb)
[longText](#column-method-longText)
[macAddress](#column-method-macAddress)
[mediumIncrements](#column-method-mediumIncrements)
[mediumInteger](#column-method-mediumInteger)
[mediumText](#column-method-mediumText)
[morphs](#column-method-morphs)
[nullableMorphs](#column-method-nullableMorphs)
[nullableTimestamps](#column-method-nullableTimestamps)
[nullableUlidMorphs](#column-method-nullableUlidMorphs)
[nullableUuidMorphs](#column-method-nullableUuidMorphs)
[rememberToken](#column-method-rememberToken)
[set](#column-method-set)
[smallIncrements](#column-method-smallIncrements)
[smallInteger](#column-method-smallInteger)
[softDeletesTz](#column-method-softDeletesTz)
[softDeletes](#column-method-softDeletes)
[string](#column-method-string)
[text](#column-method-text)
[timeTz](#column-method-timeTz)
[time](#column-method-time)
[timestampTz](#column-method-timestampTz)
[timestamp](#column-method-timestamp)
[timestampsTz](#column-method-timestampsTz)
[timestamps](#column-method-timestamps)
[tinyIncrements](#column-method-tinyIncrements)
[tinyInteger](#column-method-tinyInteger)
[tinyText](#column-method-tinyText)
[unsignedBigInteger](#column-method-unsignedBigInteger)
[unsignedInteger](#column-method-unsignedInteger)
[unsignedMediumInteger](#column-method-unsignedMediumInteger)
[unsignedSmallInteger](#column-method-unsignedSmallInteger)
[unsignedTinyInteger](#column-method-unsignedTinyInteger)
[ulidMorphs](#column-method-ulidMorphs)
[uuidMorphs](#column-method-uuidMorphs)
[ulid](#column-method-ulid)
[uuid](#column-method-uuid)
[year](#column-method-year)

</div>

<a name="column-method-bigIncrements"></a>
#### `bigIncrements()` {.collection-method .first-collection-method}

`bigIncrements`メソッドは、自動増分する`UNSIGNED BIGINT`(主キー)カラムを作成します。

    $table->bigIncrements('id');

<a name="column-method-bigInteger"></a>
#### `bigInteger()` {.collection-method}

`bigInteger`メソッドは`BIGINT`カラムを作成します。

    $table->bigInteger('votes');

<a name="column-method-binary"></a>
#### `binary()` {.collection-method}

`binary`メソッドは`BLOB`カラムを作成します。

    $table->binary('photo');

MySQL、MariaDB、SQLServerを使用する場合は、`length`と`fixed`引数を渡して、`VARBINARY`または`BINARY`相当のカラムを作成できます。

    $table->binary('data', length: 16); // VARBINARY(16)

    $table->binary('data', length: 16, fixed: true); // BINARY(16)

<a name="column-method-boolean"></a>
#### `boolean()` {.collection-method}

`boolean`メソッドは`BOOLEAN`カラムを作成します。

    $table->boolean('confirmed');

<a name="column-method-char"></a>
#### `char()` {.collection-method}

`char`メソッドは、指定した長さの`CHAR`カラムを作成します。

    $table->char('name', length: 100);

<a name="column-method-dateTimeTz"></a>
#### `dateTimeTz()` {.collection-method}

`dateTimeTz`メソッドは`DATETIME`(タイムゾーン付き)カラムを作成します。

    $table->dateTimeTz('created_at', precision: 0);

<a name="column-method-dateTime"></a>
#### `dateTime()` {.collection-method}

`dateTime`メソッドは、`DATETIME`カラムを作成します。

    $table->dateTime('created_at', precision: 0);

<a name="column-method-date"></a>
#### `date()` {.collection-method}

`date`メソッドは`DATE`カラムを作成します。

    $table->date('created_at');

<a name="column-method-decimal"></a>
#### `decimal()` {.collection-method}

`decimal`メソッドは、指定した精度(合計桁数)とスケール(小数桁数)で`DECIMAL`カラムを作成します。

    $table->decimal('amount', total: 8, places: 2);

<a name="column-method-double"></a>
#### `double()` {.collection-method}

`double`メソッドは、`DOUBLE`カラムを作成します。

    $table->double('amount');

<a name="column-method-enum"></a>
#### `enum()` {.collection-method}

`enum`メソッドは、指定した有効な値で`ENUM`カラムを作成します。

    $table->enum('difficulty', ['easy', 'hard']);

<a name="column-method-float"></a>
#### `float()` {.collection-method}

`float`メソッドは、指定した精度の`FLOAT`カラムを作成します。

    $table->float('amount', precision: 53);

<a name="column-method-foreignId"></a>
#### `foreignId()` {.collection-method}

`foreignId`メソッドは`UNSIGNED BIGINT`カラムを作成します。

    $table->foreignId('user_id');

<a name="column-method-foreignIdFor"></a>
#### `foreignIdFor()` {.collection-method}

`foreignIdFor` メソッドは、指定したモデルクラスに `{column}_id` に相当するカラムを追加します。カラムの型はモデルのキーの型に依存し、`UNSIGNED BIGINT`、`CHAR(36)`、`CHAR(26)`のいずれかです。

    $table->foreignIdFor(User::class);

<a name="column-method-foreignUlid"></a>
#### `foreignUlid()` {.collection-method}

`foreignUlid`メソッドは`ULID`カラムを作成します。

    $table->foreignUlid('user_id');

<a name="column-method-foreignUuid"></a>
#### `foreignUuid()` {.collection-method}

`foreignUuid`メソッドは`UUID`カラムを作成します。

    $table->foreignUuid('user_id');

<a name="column-method-geography"></a>
#### `geography()` {.collection-method}

`geography`メソッドは、指定した空間タイプとSRID（空間参照システム識別子）を持つ、`GEOGRAPHY`カラムを作成します。

    $table->geography('coordinates', subtype: 'point', srid: 4326);

> [!NOTE]
> 空間タイプのサポートは、ご使用のデータベース・ドライバに依存します。データベースのドキュメントを参照してください。アプリケーションが、PostgreSQLデータベースを使用している場合は、`geography`メソッドを使用する前に、[PostGIS](https://postgis.net)拡張モジュールをインストールする必要があります。

<a name="column-method-geometry"></a>
#### `geometry()` {.collection-method}

`geometry`メソッドは、指定した空間タイプとSRID（空間参照システム識別子）を持つ、`GEOMETRY`カラムを作成します。

    $table->geometry('positions', subtype: 'point', srid: 0);

> [!NOTE]
> 空間タイプのサポートは、ご使用のデータベース・ドライバに依存します。データベースのドキュメントを参照してください。アプリケーションがPostgreSQLデータベースを使用している場合は、`geometry`メソッドを使用する前に、[PostGIS](https://postgis.net)拡張モジュールをインストールする必要があります。

<a name="column-method-id"></a>
#### `id()` {.collection-method}

`id`メソッドは`bigIncrements`メソッドのエイリアスです。デフォルトでは、メソッドは`id`カラムを作成します。ただし、カラムに別の名前を割り当てたい場合は、カラム名を渡すことができます。

    $table->id();

<a name="column-method-increments"></a>
#### `increments()` {.collection-method}

`increments`メソッドは、主キーとして自動増分の`UNSIGNED INTEGER`カラムを作成します。

    $table->increments('id');

<a name="column-method-integer"></a>
#### `integer()` {.collection-method}

`integer`メソッドは`INTEGER`カラムを作成します。

    $table->integer('votes');

<a name="column-method-ipAddress"></a>
#### `ipAddress()` {.collection-method}

`ipAddress`メソッドは`VARCHAR`カラムを作成します。

    $table->ipAddress('visitor');

PostgreSQLを使用する場合、`INET`カラムが作成されます。

<a name="column-method-json"></a>
#### `json()` {.collection-method}

`json`メソッドは`JSON`カラムを作成します。

    $table->json('options');

<a name="column-method-jsonb"></a>
#### `jsonb()` {.collection-method}

`jsonb`メソッドは`JSONB`カラムを作成します。

    $table->jsonb('options');

<a name="column-method-longText"></a>
#### `longText()` {.collection-method}

`longText`メソッドは`LONGTEXT`カラムを作成します。

    $table->longText('description');

MySQLやMariaDBを使用する場合は、`LONGBLOB`カラムを作成するために、カラムへ`binary`文字セットを適用してください。

    $table->longText('data')->charset('binary'); // LONGBLOB

<a name="column-method-macAddress"></a>
#### `macAddress()` {.collection-method}

`macAddress`メソッドは、MACアドレスを保持することを目的としたカラムを作成します。PostgreSQLなどの一部のデータベースシステムには、このタイプのデータ専用のカラムタイプがあります。他のデータベースシステムでは、文字カラムに相当するカラムを使用します。

    $table->macAddress('device');

<a name="column-method-mediumIncrements"></a>
#### `mediumIncrements()` {.collection-method}

`mediumIncrements`メソッドは、主キーが自動増分の`UNSIGNED MEDIUMINT`カラムを作成します。

    $table->mediumIncrements('id');

<a name="column-method-mediumInteger"></a>
#### `mediumInteger()` {.collection-method}

`mediumInteger`メソッドは`MEDIUMINT`カラムを作成します。

    $table->mediumInteger('votes');

<a name="column-method-mediumText"></a>
#### `mediumText()` {.collection-method}

`mediumText`メソッドは`MEDIUMTEXT`カラムを作成します。

    $table->mediumText('description');

MySQLやMariaDBを使用する場合は、`MEDIUMBLOB`カラムを作成するために、カラムに`binary`文字セットを適用してください。

    $table->mediumText('data')->charset('binary'); // MEDIUMBLOB

<a name="column-method-morphs"></a>
#### `morphs()` {.collection-method}

`morphs`メソッドは、`{column}_id`、`{column}_type`、`VARCHAR`型のカラムを追加する便利なメソッドです。`{column}_id`のカラム型は、モデルキーの型に応じて`UNSIGNED BIGINT`、`CHAR(36)`、`CHAR(26)`のいずれかになります。

このメソッドは、ポリモーフィック[Eloquentリレーション](/docs/{{version}}/eloquent-relationships)に必要なカラムを定義するときに使用することを目的としています。次の例では、`taggable_id`カラムと`taggable_type`カラムが作成されます。

    $table->morphs('taggable');

<a name="column-method-nullableTimestamps"></a>
#### `nullableTimestamps()` {.collection-method}

`nullableTimestamps`メソッドは[timestamps](#column-method-timestamps)メソッドのエイリアスです。

    $table->nullableTimestamps(precision: 0);

<a name="column-method-nullableMorphs"></a>
#### `nullableMorphs()` {.collection-method}

このメソッドは、[morphs](#column-method-morphs)メソッドに似ています。ただし、作成するカラムは"NULLABLE"になります。

    $table->nullableMorphs('taggable');

<a name="column-method-nullableUlidMorphs"></a>
#### `nullableUlidMorphs()` {.collection-method}

このメソッドは[ulidMorphs](#column-method-ulidMorphs)メソッドと似ていますが、作成するカラムは"NULLABLE"になります。

    $table->nullableUlidMorphs('taggable');

<a name="column-method-nullableUuidMorphs"></a>
#### `nullableUuidMorphs()` {.collection-method}

このメソッドは、[uuidMorphs](#column-method-uuidMorphs)メソッドに似ています。ただし、作成するカラムは"NULLABLE"になります。

    $table->nullableUuidMorphs('taggable');

<a name="column-method-rememberToken"></a>
#### `rememberToken()` {.collection-method}

`rememberToken`メソッドは、現在の「ログイン持続（"remember me"）」[認証トークン](/docs/{{version}}/authentication#remembering-users)を格納することを目的としたNULL許容の`VARCHAR(100)`相当のカラムを作成します。

    $table->rememberToken();

<a name="column-method-set"></a>
#### `set()` {.collection-method}

`set`メソッドは、指定した有効な値のリストを使用して、`SET`カラムを作成します。

    $table->set('flavors', ['strawberry', 'vanilla']);

<a name="column-method-smallIncrements"></a>
#### `smallIncrements()` {.collection-method}

`smallIncrements`メソッドは、主キーとして自動増分の`UNSIGNED SMALLINT`カラムを作成します。

    $table->smallIncrements('id');

<a name="column-method-smallInteger"></a>
#### `smallInteger()` {.collection-method}

`smallInteger`メソッドは`SMALLINT`カラムを作成します。

    $table->smallInteger('votes');

<a name="column-method-softDeletesTz"></a>
#### `softDeletesTz()` {.collection-method}

`softDeletesTz`メソッドは、null値可能でオプションの小数秒の精度を持つ、`deleted_at`のタイムスタンプ（`TIMESTAMP`）(タイムゾーン付き)カラムを追加します。このカラムはEloquentの「ソフトデリート」機能に必要な、`deleted_at`タイムスタンプを格納するためのものです。

    $table->softDeletesTz('deleted_at', precision: 0);

<a name="column-method-softDeletes"></a>
#### `softDeletes()` {.collection-method}

`softDeletes`メソッドは、null値可能でオプションの小数秒の精度を持つ、`deleted_at`タイムスタンプ（`TIMESTAMP`）カラムを追加します。このカラムはEloquentの「ソフトデリート」機能に必要な、`deleted_at`タイムスタンプを格納するためのものです。

    $table->softDeletes('deleted_at', precision: 0);

<a name="column-method-string"></a>
#### `string()` {.collection-method}

`string`メソッドは、指定された長さの`VARCHAR`カラムを作成します。

    $table->string('name', length: 100);

<a name="column-method-text"></a>
#### `text()` {.collection-method}

`text`メソッドは`TEXT`カラムを作成します。

    $table->text('description');

MySQLまたはMariaDBを使用する場合、`BLOB`カラムを作成するには、カラムへ`binary`文字セットを適用してください。

    $table->text('data')->charset('binary'); // BLOB

<a name="column-method-timeTz"></a>
#### `timeTz()` {.collection-method}

`timeTz`メソッドは、オプションの小数秒精度を持つ`TIME`(タイムゾーン付き) カラムを作成します。

    $table->timeTz('sunrise', precision: 0);

<a name="column-method-time"></a>
#### `time()` {.collection-method}

`time`メソッドは、オプションの小数秒精度を持つ`TIME`カラムを作成します。

    $table->time('sunrise', precision: 0);

<a name="column-method-timestampTz"></a>
#### `timestampTz()` {.collection-method}

`timestampTz`メソッドは、オプションで小数秒の精度を持つ、タイムスタンプ（`TIMESTAMP`）(タイムゾーン付き)カラムを作成します。

    $table->timestampTz('added_at', precision: 0);

<a name="column-method-timestamp"></a>
#### `timestamp()` {.collection-method}

`timestamp`メソッドは、オプションで小数秒の精度を持つ、`TIMESTAMP`カラムを作成します。

    $table->timestamp('added_at', precision: 0);

<a name="column-method-timestampsTz"></a>
#### `timestampsTz()` {.collection-method}

`timestampsTz`メソッドは、オプションで小数秒の精度を持つ、`created_at`と`updated_at`のタイムスタンプ（`TIMESTAMP`）(タイムゾーンあり)カラムを作成します。

    $table->timestampsTz(precision: 0);

<a name="column-method-timestamps"></a>
#### `timestamps()` {.collection-method}

`timestamps`メソッドは、オプションで小数秒の精度を持つ、`created_at`と`updated_at`のタイムスタンプ（`TIMESTAMP`）カラムを作成します。

    $table->timestamps(precision: 0);

<a name="column-method-tinyIncrements"></a>
#### `tinyIncrements()` {.collection-method}

`tinyIncrements`メソッドは、主キーとして自動増分の`UNSIGNED TINYINT`カラムを作成します。

    $table->tinyIncrements('id');

<a name="column-method-tinyInteger"></a>
#### `tinyInteger()` {.collection-method}

`tinyInteger`メソッドは`TINYINT`カラムを作成します。

    $table->tinyInteger('votes');

<a name="column-method-tinyText"></a>
#### `tinyText()` {.collection-method}

`TinyText`メソッドは`TINYTEXT`カラムを作成します。

    $table->tinyText('notes');

MySQLまたはMariaDBを使用する場合、`TINYBLOB`カラムを作成するには、カラムへ`binary`文字セットを適用してください。

    $table->tinyText('data')->charset('binary'); // TINYBLOB

<a name="column-method-unsignedBigInteger"></a>
#### `unsignedBigInteger()` {.collection-method}

`unsignedBigInteger`メソッドは`UNSIGNED BIGINT`カラムを作成します。

    $table->unsignedBigInteger('votes');

<a name="column-method-unsignedInteger"></a>
#### `unsignedInteger()` {.collection-method}

`unsignedInteger`メソッドは`UNSIGNED INTEGER`カラムを作成します。

    $table->unsignedInteger('votes');

<a name="column-method-unsignedMediumInteger"></a>
#### `unsignedMediumInteger()` {.collection-method}

`unsignedMediumInteger`メソッドは、`UNSIGNED　MEDIUMINT`カラムを作成します。

    $table->unsignedMediumInteger('votes');

<a name="column-method-unsignedSmallInteger"></a>
#### `unsignedSmallInteger()` {.collection-method}

`unsignedSmallInteger`メソッドは`UNSIGNED SMALLINT`カラムを作成します。

    $table->unsignedSmallInteger('votes');

<a name="column-method-unsignedTinyInteger"></a>
#### `unsignedTinyInteger()` {.collection-method}

`unsignedTinyInteger`メソッドは` UNSIGNED　TINYINT`カラムを作成します。

    $table->unsignedTinyInteger('votes');

<a name="column-method-ulidMorphs"></a>
#### `ulidMorphs()` {.collection-method}

`ulidMorphs`メソッドは、`{column}_id` `CHAR(26)`カラムと、`{column}_type` `VARCHAR`カラムを追加する便利なメソッドです。

このメソッドは、ULID識別子を使用するポリモーフィック[Eloquentリレーション](/docs/{{version}}/eloquent-relationships)に必要なカラムを定義するために使用することを想定しています。以下の例では、`taggable_id`と`taggable_type`というカラムが作成されます。

    $table->ulidMorphs('taggable');

<a name="column-method-uuidMorphs"></a>
#### `uuidMorphs()` {.collection-method}

`uuidMorphs`メソッドは、`{column}_id` `CHAR(36)`カラムと、`{column}_type` `VARCHAR`カラムを追加する便利なメソッドです。

このメソッドは、UUID識別子を使用するポリモーフィックな[Eloquentリレーション](/docs/{{version}}/eloquent-relationships)に必要なカラムを定義するときに使用します。以下の例では、`taggable_id`カラムと`taggable_type`カラムが作成されます。

    $table->uuidMorphs('taggable');

<a name="column-method-ulid"></a>
#### `ulid()` {.collection-method}

`ulid`メソッドは`ULID`カラムを作成します。

    $table->ulid('id');

<a name="column-method-uuid"></a>
#### `uuid()` {.collection-method}

`uuid`メソッドは`UUID`カラムを作成します。

    $table->uuid('id');

<a name="column-method-year"></a>
#### `year()` {.collection-method}

`year`メソッドは`YEAR`カラムを作成します。

    $table->year('birth_year');

<a name="column-modifiers"></a>
### カラム修飾子

上記リストのカラムタイプに加え、データベーステーブルにカラムを追加するときに使用できるカラム「修飾子」もあります。たとえば、カラムを"NULLABLE"へするために、`nullable`メソッドが使用できます。

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('users', function (Blueprint $table) {
        $table->string('email')->nullable();
    });

次の表は、使用可能なすべてのカラム修飾子を紹介しています。このリストには[インデックス修飾子](#creating-indexes)は含まれていません。

<div class="overflow-auto">

| 修飾子                              | 説明                                                                                      |
| ----------------------------------- | ----------------------------------------------------------------------------------------- |
| `->after('column')`                 | カラムを別のカラムの「後に」配置（MariaDB／MySQL）                                                 |
| `->autoIncrement()`                 | INTEGERカラムを自動増分（主キー）として設定                                               |
| `->charset('utf8mb4')`              | カラムの文字セットを指定（MariaDB／MySQL）                                                         |
| `->collation('utf8mb4_unicode_ci')` | カラムのコロケーションを指定                                                              |
| `->comment('my comment')`           | カラムへコメントを追加（MariaDB／MySQL／PostgreSQL）                                               |
| `->default($value)`                 | カラムの「デフォルト」値を指定                                                            |
| `->first()`                         | テーブルの「最初の」カラムを配置（MariaDB／MySQL）                                                 |
| `->from($integer)`                  | 自動増分フィールドの開始値を設定（MariaDB／MySQL／PostgreSQL）                                     |
| `->invisible()`                     | `SELECT *`クエリに対しカラムを「不可視」にする（MariaDB／MySQL）                                   |
| `->nullable($value = true)`         | NULL値をカラムに保存可能に設定                                                            |
| `->storedAs($expression)`           | storedカラムを生成（MariaDB／MySQL／PostgreSQL／SQLite）                                           |
| `->unsigned()`                      | INTEGERカラムをUNSIGNEDとして設定（MariaDB／MySQL）                                                |
| `->useCurrent()`                    | CURRENT_TIMESTAMPをデフォルト値として使用するようにTIMESTAMPカラムを設定                  |
| `->useCurrentOnUpdate()`            | レコードが更新されたときにCURRENT_TIMESTAMPを使用するようにTIMESTAMPカラムを設定（MariaDB／MySQL） |
| `->virtualAs($expression)`          | 仮想カラムを生成（MariaDB／MySQL／SQLite）                                                         |
| `->generatedAs($expression)`        | 指定のシーケンスオプションで、識別カラムを生成（PostgreSQL）                              |
| `->always()`                        | IDカラムの入力に対するシーケンス値の優先順位を定義（PostgreSQL）                          |

</div>

<a name="default-expressions"></a>
#### デフォルト式

`default`修飾子は、値または`Illuminate\Database\Query\Expression`インスタンスを受け入れます。`Expression`インスタンスを使用すると、Laravelが値を引用符で囲むのを防ぎ、データベース固有の関数を使用できるようになります。これがとくに役立つ状況の１つは、JSONカラムにデフォルト値を割り当てる必要がある場合です。

    <?php

    use Illuminate\Support\Facades\Schema;
    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Database\Query\Expression;
    use Illuminate\Database\Migrations\Migration;

    return new class extends Migration
    {
        /**
         * マイグレーションの実行
         */
        public function up(): void
        {
            Schema::create('flights', function (Blueprint $table) {
                $table->id();
                $table->json('movies')->default(new Expression('(JSON_ARRAY())'));
                $table->timestamps();
            });
        }
    };

> [!WARNING]
> デフォルト式のサポートは、データベースドライバ、データベースのバージョン、フィールドタイプに依存します。お使いのデータベースのドキュメントを参照してください。

<a name="column-order"></a>
#### カラム順序

MariaDBとMySQLデータベースを使用するときは、スキーマ内の既存の列の後に列を追加するために`after`メソッドを使用できます。

    $table->after('password', function (Blueprint $table) {
        $table->string('address_line1');
        $table->string('address_line2');
        $table->string('city');
    });

<a name="modifying-columns"></a>
### カラムの変更

`change`メソッドを使用すると、既存のカラムのタイプと属性を変更できます。たとえば、`string`カラムのサイズを大きくしたい場合があります。`change`メソッドの動作を確認するために、`name`カラムのサイズを25から50に増やしてみましょう。これを実行するには、カラムの新しい状態を定義してから、`change`メソッドを呼び出します。

    Schema::table('users', function (Blueprint $table) {
        $table->string('name', 50)->change();
    });

カラムを変更する際には、カラム定義に保持したいすべての修飾子を明示的に含める必要があります。例えば、`unsigned`属性、`default`属性、`comment`属性を保持するには、カラムを変更する際にそれぞれの修飾子を明示的に呼び出す必要があります。

    Schema::table('users', function (Blueprint $table) {
        $table->integer('votes')->unsigned()->default(1)->comment('my comment')->change();
    });

`change`メソッドはカラムのインデックスを変更しません。そのため、カラムを変更する際には、インデックス修飾子を使って明示的にインデックスを追加もしくは、削除してください。

```php
// インデックス追加
$table->bigIncrements('id')->primary()->change();

// インデックス削除
$table->char('postal_code', 10)->unique(false)->change();
```

<a name="renaming-columns"></a>
### カラムのリネーム

カラムの名前を変更するには、スキーマビルダが提供する、`renameColumn`メソッドを使用します。

    Schema::table('users', function (Blueprint $table) {
        $table->renameColumn('from', 'to');
    });

<a name="dropping-columns"></a>
### カラムの削除

カラムを削除するには、スキーマビルダの`dropColumn`メソッドを使用します。

    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('votes');
    });

カラム名の配列を`dropColumn`メソッドに渡すことにより、テーブルから複数のカラムを削除できます。

    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn(['votes', 'avatar', 'location']);
    });

<a name="available-command-aliases"></a>
#### 使用可能なコマンドエイリアス

Laravelは、一般的なタイプのカラムの削除の便利な方法を提供しています。各メソッドは以下の表で説明します。

<div class="overflow-auto">

| コマンド                           | 説明                                               |
| ---------------------------------- | -------------------------------------------------- |
| `$table->dropMorphs('morphable');` | `morphable_id`カラムと`morphable_type`カラムを削除 |
| `$table->dropRememberToken();`     | `remember_token`カラムを削除                       |
| `$table->dropSoftDeletes();`       | `deleted_at`カラムを削除                           |
| `$table->dropSoftDeletesTz();`     | `dropSoftDeletes（）`メソッドのエイリアス          |
| `$table->dropTimestamps();`        | `created_at`カラムと`updated_at`カラムを削除       |
| `$table->dropTimestampsTz();`      | `dropTimestamps（）`メソッドのエイリアス           |

</div>

<a name="indexes"></a>
## インデックス

<a name="creating-indexes"></a>
### インデックスの生成

Laravelスキーマビルダは多くのタイプのインデックスをサポートしています。次の例では、新しい`email`カラムを作成し、その値が一意であることを指定しています。インデックスを作成するには、`unique`メソッドをカラム定義にチェーンします。

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('users', function (Blueprint $table) {
        $table->string('email')->unique();
    });

または、カラムを定義した後にインデックスを作成することもできます。これを行うには、スキーマビルダBlueprintで`unique`メソッドを呼び出す必要があります。このメソッドは、一意のインデックスを受け取る必要があるカラムの名前を引数に取ります。

    $table->unique('email');

カラムの配列をindexメソッドに渡して、複合インデックスを作成することもできます。

    $table->index(['account_id', 'created_at']);

インデックスを作成するとき、Laravelはテーブル、カラム名、およびインデックスタイプに基づいてインデックス名を自動的に生成しますが、メソッドに2番目の引数を渡して、インデックス名を自分で指定することもできます。

    $table->unique('email', 'unique_email');

<a name="available-index-types"></a>
#### 利用可能なインデックスタイプ

LaravelのスキーマビルダBlueprintクラスは、Laravelでサポートしている各タイプのインデックスを作成するメソッドを提供しています。各indexメソッドは、オプションの２番目の引数を取り、インデックスの名前を指定します。省略した場合、名前は、インデックスに使用されるテーブルとカラムの名前、およびインデックスタイプから派生します。使用可能な各インデックスメソッドは、以下の表で説明します。

<div class="overflow-auto">

| コマンド                                         | 説明                                            |
| ------------------------------------------------ | ----------------------------------------------- |
| `$table->primary('id');`                         | 主キーを追加                                    |
| `$table->primary(['id', 'parent_id']);`          | 複合キーを追加                                  |
| `$table->unique('email');`                       | 一意のインデックスを追加                        |
| `$table->index('state');`                        | インデックスを追加                              |
| `$table->fullText('body');`                      | 全文検索インデックスを追加（MariaDB／MySQL／PostgreSQL） |
| `$table->fullText('body')->language('english');` | 特定言語のフルテキストインデックス追加          |
| `$table->spatialIndex('location');`              | 空間インデックスを追加（SQLiteを除く）          |

</div>

<a name="renaming-indexes"></a>
### インデックスのリネーム

インデックスの名前を変更するには、スキーマビルダBlueprintが提供する`renameIndex`メソッドを使用します。このメソッドは、現在のインデックス名を最初の引数として取り、目的の名前を２番目の引数として取ります。

    $table->renameIndex('from', 'to')

<a name="dropping-indexes"></a>
### インデックスの削除

インデックスを削除するには、インデックスの名前を指定する必要があります。デフォルトでは、Laravelはテーブル名、インデックス付きカラムの名前、およびインデックスタイプに基づいてインデックス名を自動的に割り当てます。ここではいくつかの例を示します。

<div class="overflow-auto">

| コマンド                                                 | 説明                                                    |
| -------------------------------------------------------- | ------------------------------------------------------- |
| `$table->dropPrimary('users_id_primary');`               | "users"テーブルから主キーを削除                         |
| `$table->dropUnique('users_email_unique');`              | "users"テーブルから一意のインデックスを削除             |
| `$table->dropIndex('geo_state_index');`                  | "geo"テーブルから基本インデックスを削除                 |
| `$table->dropFullText('posts_body_fulltext');`           | "posts"テーブルからフルテキストインデックスを削除       |
| `$table->dropSpatialIndex('geo_location_spatialindex');` | "geo"テーブルから空間インデックスを削除（SQLiteを除く） |

</div>

インデックスを削除するメソッドにカラムの配列を渡すと、テーブル名、カラム、およびインデックスタイプに基づいてインデックス名が生成されます。

    Schema::table('geo', function (Blueprint $table) {
        $table->dropIndex(['state']); // 'geo_state_index'インデックスを削除
    });

<a name="foreign-key-constraints"></a>
### 外部キー制約

Laravelは、データベースレベルで参照整合性を強制するために使用される外部キー制約の作成もサポートしています。たとえば、`users`テーブルの`id`カラムを参照する`posts`テーブルの`user_id`カラムを定義しましょう。

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('posts', function (Blueprint $table) {
        $table->unsignedBigInteger('user_id');

        $table->foreign('user_id')->references('id')->on('users');
    });

この構文はかなり冗長であるため、Laravelは、より良い開発者エクスペリエンスを提供するため、規約を使用した簡潔なメソッドをさらに提供します。`foreignId`メソッドを使用すると、上記の例は次のように書き直すことができます。

    Schema::table('posts', function (Blueprint $table) {
        $table->foreignId('user_id')->constrained();
    });

`foreignId`メソッドは、`UNSIGNED BIGINT`カラムを作成し、`constrained`メソッドは規約を利用し、参照するテーブルとカラムを決定します。テーブル名がLaravelの規約と合わない場合は、手作業で`constrained`メソッドへ指定してください。さらに、生成するインデックスへ割り当てる名前も指定できます。

    Schema::table('posts', function (Blueprint $table) {
        $table->foreignId('user_id')->constrained(
            table: 'users', indexName: 'posts_user_id'
        );
    });

必要なアクションに"on delete"や"on update"の制約プロパティを指定することもできます。

    $table->foreignId('user_id')
          ->constrained()
          ->onUpdate('cascade')
          ->onDelete('cascade');

これらのアクションには、表現力の高い別構文も用意しています。

<div class="overflow-auto">

| メソッド                        | 解説                                   |
| ----------------------------- | ------------------------------------------------- |
| `$table->cascadeOnUpdate();`  | 更新をカスケードします                 |
| `$table->restrictOnUpdate();` | 更新を制限します                       |
| `$table->noActionOnUpdate();` | 更新では何もしません                   |
| `$table->cascadeOnDelete();`  | 削除をカスケードします                 |
| `$table->restrictOnDelete();` | 削除を制限します。                     |
| `$table->nullOnDelete();`     | 削除時に外部キーへNULLをセットします。 |

</div>

追加の[カラム修飾子](#column-modifiers)は、`constrained`メソッドの前に呼び出す必要があります。

    $table->foreignId('user_id')
          ->nullable()
          ->constrained();

<a name="dropping-foreign-keys"></a>
#### 外部キーの削除

外部キーを削除するには、`dropForeign`メソッドを使用して、削除する外部キー制約の名前を引数として渡してください。外部キー制約は、インデックスと同じ命名規約を使用しています。つまり、外部キー制約名は、制約内のテーブルとカラムの名前に基づいており、その後に「_foreign」サフィックスが続きます。

    $table->dropForeign('posts_user_id_foreign');

または、外部キーを保持するカラム名を含む配列を`dropForeign`メソッドに渡すこともできます。配列は、Laravelの制約命名規約を使用して外部キー制約名に変換されます。

    $table->dropForeign(['user_id']);

<a name="toggling-foreign-key-constraints"></a>
#### 外部キー制約の切り替え

次の方法を使用して、マイグレーション内の外部キー制約を有効または無効にできます。

    Schema::enableForeignKeyConstraints();

    Schema::disableForeignKeyConstraints();

    Schema::withoutForeignKeyConstraints(function () {
        // Constraints disabled within this closure...
    });

> [!WARNING]
> SQLiteは、デフォルトで外部キー制約を無効にします。SQLiteを使用する場合は、マイグレーションでデータベースを作成する前に、データベース設定の[外部キーサポートを有効にする](/docs/{{version}}/database#configuration)を確実に行ってください。さらに、SQLiteはテーブルの作成時にのみ外部キーをサポートし、[テーブルを変更する場合はサポートしません](https://www.sqlite.org/omitted.html)。

<a name="events"></a>
## イベント

利便が良いように、各マイグレート操作は[イベント](/docs/{{version}}/events)を発行します。以下のイベントはすべて、`Illuminate\Database\Events\MigrationEvent`基本クラスを継承しています。

<div class="overflow-auto">

 | クラス                                             | 説明                                  |
 | ---------------------------------------------- | ------------------------------------------------ |
 | `Illuminate\Database\Events\MigrationsStarted`   | マイグレーションのバッチが実行されようとしている |
 | `Illuminate\Database\Events\MigrationsEnded`     | マイグレーションのバッチが実行終了した           |
 | `Illuminate\Database\Events\MigrationStarted`    | 単一マイグレーションが実行されようとしている     |
 | `Illuminate\Database\Events\MigrationEnded`      | 単一マイグレーションが実行終了した |               |
 | `Illuminate\Database\Events\NoPendingMigrations` | 未適用のマイグレーションをみつけられなかった     |
 | `Illuminate\Database\Events\SchemaDumped`        | データベーススキマのダンプが終了した             |
 | `Illuminate\Database\Events\SchemaLoaded`        | 既存のデータベーススキマのダンプをロードした     |

</div>
