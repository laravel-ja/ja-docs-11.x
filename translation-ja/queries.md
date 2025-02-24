# データベース：クエリビルダ

- [イントロダクション](#introduction)
- [データベースクエリの実行](#running-database-queries)
    - [結果の分割](#chunking-results)
    - [ルーズなストリーミング結果](#streaming-results-lazily)
    - [集計](#aggregates)
- [SELECT文](#select-statements)
- [素のSQL](#素のSQL)
- [JOIN](#joins)
- [UNION](#unions)
- [基本WHERE句](#basic-where-clauses)
    - [WHERE句](#where-clauses)
    - [OR WHERE句](#or-where-clauses)
    - [WHERE NOT句](#where-not-clauses)
    - [Where Any／All／None句](#where-any-all-none-clauses)
    - [JSON WHERE句](#json-where-clauses)
    - [その他のWHERE句](#additional-where-clauses)
    - [論理グループ化](#logical-grouping)
- [上級WHERE節](#advanced-where-clauses)
    - [WHERE EXISTS句](#where-exists-clauses)
    - [サブクエリWHERE句](#subquery-where-clauses)
    - [フルテキストのWhere句](#full-text-where-clauses)
- [順序、グループ化、件数制限、オフセット](#ordering-grouping-limit-and-offset)
    - [順序](#ordering)
    - [グループ化](#grouping)
    - [件数制限とオフセット](#limit-and-offset)
- [条件節](#conditional-clauses)
- [INSERT文](#insert-statements)
    - [UPSERTS](#upserts)
- [UPDATE文](#update-statements)
    - [JSONカラムの更新](#updating-json-columns)
    - [増分と減分](#increment-and-decrement)
- [DELETE文](#delete-statements)
- [悲観的ロック](#pessimistic-locking)
- [デバッグ](#debugging)

<a name="introduction"></a>
## イントロダクション

Laravelのデータベースクエリビルダは、データベースクエリを作成、実行するための便利で流暢(fluent)なインターフェイスを提供します。ほとんどのデータベース操作をアプリケーションで実行するために使用でき、Laravelがサポートするすべてのデータベースシステムで完全に機能します。

Laravelクエリビルダは、PDOパラメータバインディングを使用して、SQLインジェクション攻撃からアプリケーションを保護します。クエリバインディングとしてクエリビルダに渡す文字列をクリーンアップやサニタイズする必要はありません。

> [!WARNING]
> PDOはカラム名のバインドをサポートしていません。したがって、"order by"カラムを含む、クエリが参照するカラム名をユーザー入力で指定できないようにする必要があります。

<a name="running-database-queries"></a>
## データベースクエリの実行

<a name="retrieving-all-rows-from-a-table"></a>
#### テーブルからの全行の取得

`DB`ファサードが提供する`table`メソッドを使用してクエリの最初に使用します。`table`メソッドは、指定するテーブルの流暢(fluent)なクエリビルダインスタンスを返します。これによりクエリにさらに制約をチェーンし、最後に`get`メソッドを使用してクエリの結果を取得できます。

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Support\Facades\DB;
    use Illuminate\View\View;

    class UserController extends Controller
    {
        /**
         * アプリケーションの全ユーザーをリスト表示
         */
        public function index(): View
        {
            $users = DB::table('users')->get();

            return view('user.index', ['users' => $users]);
        }
    }

`get`メソッドは、クエリの結果を含む`Illuminate\Support\Collection`インスタンスを返します。各結果は、PHPの`stdClass`オブジェクトのインスタンスです。オブジェクトのプロパティとしてカラムにアクセスすることにより、各カラムの値にアクセスできます。

    use Illuminate\Support\Facades\DB;

    $users = DB::table('users')->get();

    foreach ($users as $user) {
        echo $user->name;
    }

> [!NOTE]
> Laravelコレクションは、データをマッピングや削減するためのさまざまなとても強力な方法を提供しています。Laravelコレクションの詳細は、[コレクションのドキュメント](/docs/{{version}}/collections)をご覧ください。

<a name="retrieving-a-single-row-column-from-a-table"></a>
#### テーブルから単一の行/カラムを取得する

データベーステーブルから単一の行だけを取得する必要がある場合は、`DB`ファサードの`first`メソッドを使用します。このメソッドは、単一の`stdClass`オブジェクトを返します。

    $user = DB::table('users')->where('name', 'John')->first();

    return $user->email;

データベーステーブルから１行だけ取得したいが、マッチする行が見つからなかった場合に`Illuminate\Database\RecordNotFoundException`を投げたい場合は、`firstOrFail`メソッドを使用します。もし、`RecordNotFoundException`をキャッチしなければ、自動的に404　HTTPレスポンスをクライアントへ返します。

    $user = DB::table('users')->where('name', 'John')->firstOrFail();

行全体が必要ない場合は、`value`メソッドを使用してレコードから単一の値を抽出できます。このメソッドは、カラムの値を直接返します。

    $email = DB::table('users')->where('name', 'John')->value('email');

`id`列の値で単一の行を取得するには、`find`メソッドを使用します。

    $user = DB::table('users')->find(3);

<a name="retrieving-a-list-of-column-values"></a>
#### カラム値のリストの取得

単一のカラムの値を含む`Illuminate\Support\Collection`インスタンスを取得する場合は、`pluck`メソッドを使用します。この例では、ユーザーのタイトルのコレクションを取得します。

    use Illuminate\Support\Facades\DB;

    $titles = DB::table('users')->pluck('title');

    foreach ($titles as $title) {
        echo $title;
    }

`pluck`メソッドに２番目の引数を指定し、結果のコレクションがキーとして使用する列を指定できます。

    $titles = DB::table('users')->pluck('title', 'name');

    foreach ($titles as $name => $title) {
        echo $title;
    }

<a name="chunking-results"></a>
### 結果の分割

何千ものデータベースレコードを処理する必要がある場合は、`DB`ファサードが提供する`chunk`メソッドの使用を検討してください。このメソッドは、一回に小さな結果のチャンク（小間切れ）を取得し、各チャンクをクロージャで処理するために送ります。たとえば、`users`テーブル全体を一回で100レコードのチャンクで取得してみましょう。

    use Illuminate\Support\Collection;
    use Illuminate\Support\Facades\DB;

    DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
        foreach ($users as $user) {
            // ...
        }
    });

クロージャから `false`を返えせば、それ以上のチャンクの処理を停止できます。

    DB::table('users')->orderBy('id')->chunk(100, function (Collection $users) {
        // レコードの処理…

        return false;
    });

結果をチャンク処理している途中にデータベースレコードを更新している場合、チャンクの結果が予期しない方法で変更される可能性があります。チャンク処理中に取得レコードが更新される場合は、代わりに`chunkById`メソッドを使用するのが常に最善です。このメソッドは、レコードの主キーに基づいて結果を自動的にページ分割します。

    DB::table('users')->where('active', false)
        ->chunkById(100, function (Collection $users) {
            foreach ($users as $user) {
                DB::table('users')
                    ->where('id', $user->id)
                    ->update(['active' => true]);
            }
        });

`chunkById`メソッドと`lazyById`メソッドは、実行するクエリに独自の"where"条件を追加するため、通常はクロージャ内へ独自条件を[論理的にグループ化](#logical-grouping)する必要があります。

```php
DB::table('users')->where(function ($query) {
    $query->where('credits', 1)->orWhere('credits', 2);
})->chunkById(100, function (Collection $users) {
    foreach ($users as $user) {
        DB::table('users')
            ->where('id', $user->id)
            ->update(['credits' => 3]);
    }
});
```

> [!WARNING]
> チャンクコールバックの中でレコードを更新または削除する場合、主キーまたは外部キーの変更がチャンククエリに影響を与える可能性があります。これにより、レコードがチャンク化された結果に含まれない可能性が発生します。

<a name="streaming-results-lazily"></a>
### ルーズなストリーミング結果

`lazy`メソッドは、チャンク単位でクエリを実行するという意味で、[`chunk`メソッド](#chunking-results)と似たような動作をします。しかし、各チャンクをコールバックに渡すのではなく、`lazy()`メソッドは [`LazyCollection`](/docs/{{version}}/collections#lazy-collections)を返し、結果をひとつのストリームとして扱えます。

```php
use Illuminate\Support\Facades\DB;

DB::table('users')->orderBy('id')->lazy()->each(function (object $user) {
    // ...
});
```

繰り返しになりますが、検索したレコードを反復しながら更新する予定がある場合は、代わりに`lazyById`か`lazyByIdDesc`メソッドの使用を推奨します。これらのメソッドは、レコードの主キーに基づいて結果を自動的にページ分割します。

```php
DB::table('users')->where('active', false)
    ->lazyById()->each(function (object $user) {
        DB::table('users')
            ->where('id', $user->id)
            ->update(['active' => true]);
    });
```

> [!WARNING]
> レコードの反復処理中にレコードの更新や削除を行うと、主キーや外部キーの変更がチャンククエリへ影響を与える可能性があります。これにより、レコードが結果に含まれない可能性があります。

<a name="aggregates"></a>
### 集計

クエリビルダは、`count`、`max`、`min`、`avg`、`sum`などの集計値を取得するさまざまなメソッドも用意しています。クエリを作成した後、こうしたメソッドのどれでも呼び出すことができます。

    use Illuminate\Support\Facades\DB;

    $users = DB::table('users')->count();

    $price = DB::table('orders')->max('price');

もちろん、これらのメソッドを他の句と組み合わせて、集計値の計算方法を調整できます。

    $price = DB::table('orders')
        ->where('finalized', 1)
        ->avg('price');

<a name="determining-if-records-exist"></a>
#### レコード存在の判定

`count`メソッドを使用して、クエリ制約に一致するレコードが存在するかどうかを判断する代わりに、`exists`メソッドと`doesntExist`メソッドが使用できます。

    if (DB::table('orders')->where('finalized', 1)->exists()) {
        // ...
    }

    if (DB::table('orders')->where('finalized', 1)->doesntExist()) {
        // ...
    }

<a name="select-statements"></a>
## SELECT文

<a name="specifying-a-select-clause"></a>
#### Specifying a Select Clause

データベーステーブルからすべての列を選択する必要があるとは限りません。`select`メソッドを使用して、クエリにカスタムの"select"句を指定できます。

    use Illuminate\Support\Facades\DB;

    $users = DB::table('users')
        ->select('name', 'email as user_email')
        ->get();

`distinct`メソッドを使用すると、クエリにダブりのない結果を返すように強制できます。

    $users = DB::table('users')->distinct()->get();

クエリビルダインスタンスがすでにあり、既存のSELECT句にカラムを追加する場合は、`addSelect`メソッドを使用できます。

    $query = DB::table('users')->select('name');

    $users = $query->addSelect('age')->get();

<a name="素のSQL"></a>
## 素のSQL

クエリへ任意の文字列を挿入する必要のある場合があります。素の文字列式を作成するには、`DB`ファサードが提供する`raw`メソッドを使用します。

    $users = DB::table('users')
        ->select(DB::raw('count(*) as user_count, status'))
        ->where('status', '<>', 1)
        ->groupBy('status')
        ->get();

> [!WARNING]
> 素のSQL文はそのまま文字列としてクエリへ挿入されるため、SQLインジェクションの脆弱性を含めぬように細心の注意を払う必要があります。

<a name="raw-methods"></a>
### rawメソッド

`DB::raw`メソッドを使用する代わりに以降のメソッドを使用して、クエリのさまざまな部分に素のSQL式を挿入することもできます。**素の式を使用するクエリでは、SQLインジェクションの脆弱性からの保護をLaravelは保証しないことに注意してください。**

<a name="selectraw"></a>
#### `selectRaw`

`addSelect(DB::raw(/*...*/))`の代わりに`selectRaw`メソッドを使用できます。このメソッドは、２番目の引数にバインディングのオプションの配列を取ります。

    $orders = DB::table('orders')
        ->selectRaw('price * ? as price_with_tax', [1.0825])
        ->get();

<a name="whereraw-orwhereraw"></a>
#### `whereRaw／orWhereRaw`

`whereRaw`メソッドと`orWhereRaw`メソッドを使用して、素の"where"句をクエリに挿入できます。これらのメソッドは、２番目の引数にバインディングのオプションの配列を取ります。

    $orders = DB::table('orders')
        ->whereRaw('price > IF(state = "TX", ?, 100)', [200])
        ->get();

<a name="havingraw-orhavingraw"></a>
#### `havingRaw／orHavingRaw`

`havingRaw`メソッドと`orHavingRaw`メソッドを使用して、"having"句の値として素の文字列を指定できます。これらのメソッドは、２番目の引数にバインディングのオプションの配列を取ります。

    $orders = DB::table('orders')
        ->select('department', DB::raw('SUM(price) as total_sales'))
        ->groupBy('department')
        ->havingRaw('SUM(price) > ?', [2500])
        ->get();

<a name="orderbyraw"></a>
#### `orderByRaw`

`orderByRaw`メソッドを使用して、"order by"句の値として素の文字列を指定できます。

    $orders = DB::table('orders')
        ->orderByRaw('updated_at - created_at DESC')
        ->get();

<a name="groupbyraw"></a>
### `groupByRaw`

`groupByRaw`メソッドを使用して、`groupby`句の値として素の文字列を指定できます。

    $orders = DB::table('orders')
        ->select('city', 'state')
        ->groupByRaw('city, state')
        ->get();

<a name="joins"></a>
## JOIN

<a name="inner-join-clause"></a>
#### INNER JOIN句

クエリビルダを使用して、クエリにJOIN句を追加することもできます。基本的な"inner join"を実行するには、クエリビルダインスタンスで`join`メソッドを使用します。`join`メソッドに渡す最初の引数は、結合するテーブルの名前であり、残りの引数は、結合のカラム制約を指定します。１つのクエリで複数のテーブルと結合することもできます。

    use Illuminate\Support\Facades\DB;

    $users = DB::table('users')
        ->join('contacts', 'users.id', '=', 'contacts.user_id')
        ->join('orders', 'users.id', '=', 'orders.user_id')
        ->select('users.*', 'contacts.phone', 'orders.price')
        ->get();

<a name="left-join-right-join-clause"></a>
#### LEFT JOIN／RIGHT JOIN句

"inner join"の代わりに"left join"や"right join"を実行する場合は、`leftJoin`か`rightJoin`メソッドを使用します。これらのメソッドは`join`メソッドと同じ引数を取ります。

    $users = DB::table('users')
        ->leftJoin('posts', 'users.id', '=', 'posts.user_id')
        ->get();

    $users = DB::table('users')
        ->rightJoin('posts', 'users.id', '=', 'posts.user_id')
        ->get();

<a name="cross-join-clause"></a>
#### CROSS JOIN句

`crossJoin`メソッドを使用して、"cross join"を実行できます。クロス結合は、最初のテーブルと結合するテーブルとの間のデカルト積を生成します。

    $sizes = DB::table('sizes')
        ->crossJoin('colors')
        ->get();

<a name="advanced-join-clauses"></a>
#### 上級JOIN句

より高度なJOIN句を指定することもできます。そのためには、`join`メソッドの２番目の引数にクロージャを渡します。クロージャは`Illuminate\Database\Query\JoinClause`インスタンスを受け取ります。これにより、"join"句に制約を指定できます。

    DB::table('users')
        ->join('contacts', function (JoinClause $join) {
            $join->on('users.id', '=', 'contacts.user_id')->orOn(/* ... */);
        })
        ->get();

テーブル結合で"where"句を使用する場合は、`JoinClause`インスタンスが提供する`where`と`orWhere`メソッドを使用します。２つのカラムを比較する代わりに、これらのメソッドはカラムを値と比較します。

    DB::table('users')
        ->join('contacts', function (JoinClause $join) {
            $join->on('users.id', '=', 'contacts.user_id')
                ->where('contacts.user_id', '>', 5);
        })
        ->get();

<a name="subquery-joins"></a>
#### サブクエリのJOIN

`joinSub`、`leftJoinSub`、`rightJoinSub`メソッドを使用して、クエリをサブクエリに結合できます。各メソッドは、サブクエリ、そのテーブルエイリアス、および関連するカラムを定義するクロージャの３引数を取ります。この例では、各ユーザーレコードにユーザーの最後に公開されたブログ投稿の`created_at`タイムスタンプも含まれているユーザーのコレクションを取得しています。

    $latestPosts = DB::table('posts')
        ->select('user_id', DB::raw('MAX(created_at) as last_post_created_at'))
        ->where('is_published', true)
        ->groupBy('user_id');

    $users = DB::table('users')
        ->joinSub($latestPosts, 'latest_posts', function (JoinClause $join) {
            $join->on('users.id', '=', 'latest_posts.user_id');
        })->get();

<a name="lateral-joins"></a>
#### ラテラルJoin

> [!WARNING]
> 現在、ラテラルJoinはPostgreSQL、MySQL8.0.14以上、SQL Serverでサポートされています。

`joinLateral` メソッドと `leftJoinLateral` メソッドを使用すると、サブクエリとの「ラテラル結合」を行えます。これらのメソッドはそれぞれ２つの引数を取ります。サブクエリとそのテーブルエイリアスです。結合条件は、指定するサブクエリの中で、`where`節で指定する必要があります。ラテラル結合は行ごとに評価され、サブクエリ外の列を参照できます。

この例では、ユーザーのコレクションと同時に、各ユーザーの最新ブログ記事を３つ取得しています。各ユーザーは結果セットへ最大３つの行を生成できます: 最新ブログ投稿それぞれに対して１つずつです。結合条件は、サブクエリ内の`whereColumn`節で指定し、現在のユーザー行を参照しています。

    $latestPosts = DB::table('posts')
        ->select('id as post_id', 'title as post_title', 'created_at as post_created_at')
        ->whereColumn('user_id', 'users.id')
        ->orderBy('created_at', 'desc')
        ->limit(3);

    $users = DB::table('users')
        ->joinLateral($latestPosts, 'latest_posts')
        ->get();

<a name="unions"></a>
## UNION

クエリビルダは、２つ以上のクエリを「結合(union)」するために便利な方法も提供します。たとえば、最初のクエリを作成したあとで、`union`メソッドを使用してより多くのクエリを結合できます。

    use Illuminate\Support\Facades\DB;

    $first = DB::table('users')
        ->whereNull('first_name');

    $users = DB::table('users')
        ->whereNull('last_name')
        ->union($first)
        ->get();

`union`メソッドに加えて、クエリビルダでは`unionAll`メソッドも提供しています。`unionAll`メソッドを使用して結合されたクエリでは、重複する結果は削除されません。`unionAll`メソッドは、`union`メソッドと同じメソッド引数です。

<a name="basic-where-clauses"></a>
## 基本WHERE句

<a name="where-clauses"></a>
### WHERE句

クエリビルダの`where`メソッドを使用して、クエリに"where"句を追加できます。`where`メソッドのもっとも基本的な呼び出しには、３つの引数が必要です。最初の引数はカラムの名前です。２番目の引数は演算子であり、データベースがサポートしている任意の演算子が指定できます。３番目の引数はカラムの値と比較する値です。

たとえば、以下のクエリは`votes`列の値が`100`に等しく、`age`列の値が`35`より大きいユーザーを取得します。

    $users = DB::table('users')
        ->where('votes', '=', 100)
        ->where('age', '>', 35)
        ->get();

使いやすいように、カラムが特定の値に対して`=`であることを確認する場合は、その値を２番目の引数として`where`メソッドに渡すことができます。Laravelは、`=`演算子を使用したと扱います。

    $users = DB::table('users')->where('votes', 100)->get();

前述のように、データベースシステムがサポートしている任意の演算子を使用できます。

    $users = DB::table('users')
        ->where('votes', '>=', 100)
        ->get();

    $users = DB::table('users')
        ->where('votes', '<>', 100)
        ->get();

    $users = DB::table('users')
        ->where('name', 'like', 'T%')
        ->get();

条件の配列を`where`関数に渡すこともできます。配列の各要素は、通常`where`メソッドに渡す３つの引数を含む配列である必要があります。

    $users = DB::table('users')->where([
        ['status', '=', '1'],
        ['subscribed', '<>', '1'],
    ])->get();

> [!WARNING]
> PDOはカラム名のバインドをサポートしていません。したがって、"order by"カラムを含む、クエリが参照するカラム名をユーザー入力で指定できないようにする必要があります。

> [!WARNING]
> MySQLとMariaDBは文字列と数値の比較において、文字列を自動的に整数にタイプキャストします。この処理で、数値以外の文字列は`0`へ変換され、予期しない結果になることがある。例えば、テーブルに`aaa`という値の`secret`カラムがあり、`User::where('secret', 0)`を実行すると、その行が返されます。これを避けるには、クエリで使用する前に、すべての値を適切な型に型キャストしてください。

<a name="or-where-clauses"></a>
### OR WHERE句

クエリビルダの`where`メソッドへの呼び出しをチェーン化する場合、"where"句は`and`演算子を使用して結合されます。ただし、`orWhere`メソッドを使用して、`or`演算子を使用して句をクエリに結合することもできます。`orWhere`メソッドは`where`メソッドと同じ引数を受け入れます。

    $users = DB::table('users')
        ->where('votes', '>', 100)
        ->orWhere('name', 'John')
        ->get();

括弧内に"or"条件をグループ化する必要がある場合は、`orWhere`メソッドの最初の引数としてクロージャを渡してください。

    $users = DB::table('users')
        ->where('votes', '>', 100)
        ->orWhere(function (Builder $query) {
            $query->where('name', 'Abigail')
                ->where('votes', '>', 50);
            })
        ->get();

上記の例では、以下のSQLを生成します。

```sql
select * from users where votes > 100 or (name = 'Abigail' and votes > 50)
```

> [!WARNING]
> グローバルスコープを適用する場合の予期しない動作を回避するために、常に`orWhere`呼び出しをグループ化する必要があります。

<a name="where-not-clauses"></a>
### WHERE NOT句

`whereNot`と`orWhereNot`メソッドを使用すると、指定したクエリ制約のグループを否定できます。例えば、以下のクエリは、クリアランス品や、価格が１０以下の商品を除外します。

    $products = DB::table('products')
        ->whereNot(function (Builder $query) {
            $query->where('clearance', true)
                ->orWhere('price', '<', 10);
            })
        ->get();

<a name="where-any-all-none-clauses"></a>
### Where Any／All／None句

複数の列に同じクエリ制約を適用する必要が起きる場合があります。例えば、指定リスト内のカラムが指定値と`LIKE`である全レコードを取得したい場合です。この場合は、`whereAny`メソッドを使用します。

    $users = DB::table('users')
        ->where('active', true)
        ->whereAny([
            'name',
            'email',
            'phone',
        ], 'like', 'Example%')
        ->get();

上記のクエリは以下のSQLになります。

```sql
SELECT *
FROM users
WHERE active = true AND (
    name LIKE 'Example%' OR
    email LIKE 'Example%' OR
    phone LIKE 'Example%'
)
```

同様に`whereAll`メソッドは、指定列のすべてが、指定した制約と一致するレコードを検索するために使用します。

    $posts = DB::table('posts')
        ->where('published', true)
        ->whereAll([
            'title',
            'content',
        ], 'like', '%Laravel%')
        ->get();

上記のクエリは以下のSQLになります。

```sql
SELECT *
FROM posts
WHERE published = true AND (
    title LIKE '%Laravel%' AND
    content LIKE '%Laravel%'
)
```

`whereNone`メソッドは、指定カラムのいずれもが指定制約に一致しないレコードを検索するために使用できます。

    $posts = DB::table('albums')
        ->where('published', true)
        ->whereNone([
            'title',
            'lyrics',
            'tags',
        ], 'like', '%explicit%')
        ->get();

上記のクエリは以下のSQLを生成します。

```sql
SELECT *
FROM albums
WHERE published = true AND NOT (
    title LIKE '%explicit%' OR
    lyrics LIKE '%explicit%' OR
    tags LIKE '%explicit%'
)
```

<a name="json-where-clauses"></a>
### JSON WHERE句

Laravelは、JSONカラム型をサポートしているデータベースでは、JSONカラム型のクエリもサポートしています。現在のところ、MariaDB10.3以上、MySQL8.0以上、PostgreSQL12.0以上、SQLServer2017以上、SQLite3.39.0以上をサポートしています。JSONカラムクエリするには、`->`演算子を使用します。

    $users = DB::table('users')
        ->where('preferences->dining->meal', 'salad')
        ->get();

`whereJsonContains`を使用してJSON配列をクエリできます。

    $users = DB::table('users')
        ->whereJsonContains('options->languages', 'en')
        ->get();

アプリケーションがMariaDB、MySQL、PostgreSQLデータベースを使用している場合は、値の配列を`whereJsonContains`メソッドで渡してください。

    $users = DB::table('users')
        ->whereJsonContains('options->languages', ['en', 'de'])
        ->get();

`whereJsonLength`メソッドを使用して、JSON配列をその長さでクエリできます。

    $users = DB::table('users')
        ->whereJsonLength('options->languages', 0)
        ->get();

    $users = DB::table('users')
        ->whereJsonLength('options->languages', '>', 1)
        ->get();

<a name="additional-where-clauses"></a>
### その他のWHERE句

**whereLike／orWhereLike／whereNotLike／orWhereNotLike**

`whereLike`メソッドにより、クエリへ"LIKE"句を追加してパターンマッチを行えます。これらのメソッドは、データベースに依存しない文字列マッチングクエリの実行方法を提供し、大文字小文字の区別を切り替えることができます。文字列マッチングはデフォルトで、大文字小文字を区別しません。

    $users = DB::table('users')
        ->whereLike('name', '%John%')
        ->get();

`caseSensitive`引数を使い、大文字小文字を区別することもできます。

    $users = DB::table('users')
        ->whereLike('name', '%John%', caseSensitive: true)
        ->get();

`orWhereLike`メソッドは、LIKE条件に"or"句を追加するために使います。

    $users = DB::table('users')
        ->where('votes', '>', 100)
        ->orWhereLike('name', '%John%')
        ->get();

The `whereNotLike`メソッドは、クエリへ"NOT LIKE"句を追加するために使います。

    $users = DB::table('users')
        ->whereNotLike('name', '%John%')
        ->get();

同様に、`orWhereNotLike`を使えば、NOT LIKE条件付きの"or"句を追加できます。

    $users = DB::table('users')
        ->where('votes', '>', 100)
        ->orWhereNotLike('name', '%John%')
        ->get();

> [!WARNING]
> 大文字と小文字を区別する、`whereLike`検索オプションは、現在SQL Serverではサポートされていません。

**whereIn／whereNotIn／orWhereIn／orWhereNotIn**

`whereIn`メソッドは、特定のカラム値が指定した配列内に含まれる条件を加えます。

    $users = DB::table('users')
        ->whereIn('id', [1, 2, 3])
        ->get();

`whereNotIn`メソッドは、特定のカラム値が指定した配列に含まれない条件を加えます。

    $users = DB::table('users')
        ->whereNotIn('id', [1, 2, 3])
        ->get();

`whereIn`メソッドの第２引数へ、クエリオブジェクトを指定することもできます。

    $activeUsers = DB::table('users')->select('id')->where('is_active', 1);

    $users = DB::table('comments')
        ->whereIn('user_id', $activeUsers)
        ->get();

上記の例では、以下のSQLを生成します。

```sql
select * from comments where user_id in (
    select id
    from users
    where is_active = 1
)
```

> [!WARNING]
> クエリに整数バインディングの大きな配列を追加する場合は、`whereIntegerInRaw`または`whereIntegerNotInRaw`メソッドを使用してメモリ使用量を大幅に削減できます。

**whereBetween／orWhereBetween**

`whereBetween`メソッドは、カラム値が２つの値の間にあるかを検証します。

    $users = DB::table('users')
        ->whereBetween('votes', [1, 100])
        ->get();

**whereNotBetween／orWhereNotBetween**

`whereNotBetween`メソッドは、カラム値が２つの値の間の外にあるかを検証します。

    $users = DB::table('users')
        ->whereNotBetween('votes', [1, 100])
        ->get();

**whereBetweenColumns／whereNotBetweenColumns／orWhereBetweenColumns／orWhereNotBetweenColumns**

`whereBetweenColumns`メソッドは、カラム値が同じテーブル行の２カラムの値の間にあるかを検証します。

    $patients = DB::table('patients')
        ->whereBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
        ->get();

`whereNotBetweenColumns`メソッドは、カラム値が同じテーブル行の２カラムの値の外側にあるかを検証します。

    $patients = DB::table('patients')
        ->whereNotBetweenColumns('weight', ['minimum_allowed_weight', 'maximum_allowed_weight'])
        ->get();

**whereNull／whereNotNull／orWhereNull／orWhereNotNull**

`whereNull`メソッドは、指定したカラムの値が`NULL`である条件を加えます。

    $users = DB::table('users')
        ->whereNull('updated_at')
        ->get();

`whereNotNull`メソッドは、カラムの値が`NULL`ではないことを確認します。

    $users = DB::table('users')
        ->whereNotNull('updated_at')
        ->get();

**whereDate／whereMonth／whereDay／whereYear／whereTime**

`whereDate`メソッドを使用して、カラム値を日付と比較できます。

    $users = DB::table('users')
        ->whereDate('created_at', '2016-12-31')
        ->get();

`whereMonth`メソッドを使用して、カラム値を特定の月と比較できます。

    $users = DB::table('users')
        ->whereMonth('created_at', '12')
        ->get();

`whereDay`メソッドを使用して、カラム値を月の特定の日と比較できます。

    $users = DB::table('users')
        ->whereDay('created_at', '31')
        ->get();

`whereYear`メソッドを使用して、カラム値を特定の年と比較できます。

    $users = DB::table('users')
        ->whereYear('created_at', '2016')
        ->get();

`whereTime`メソッドを使用して、カラム値を特定の時間と比較できます。

    $users = DB::table('users')
        ->whereTime('created_at', '=', '11:20:45')
        ->get();

**wherePast／whereFuture／whereToday／whereBeforeToday／whereAfterToday**

`wherePast`と`whereFuture`メソッドは、カラムの値が過去か未来かを判定するために使用します。

    $invoices = DB::table('invoices')
        ->wherePast('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereFuture('due_at')
        ->get();

`whereNowOrPast`と`whereNowOrFuture`メソッドは、現在の日時を含め、カラムの値が過去か未来かを判定するため使用します。

    $invoices = DB::table('invoices')
        ->whereNowOrPast('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereNowOrFuture('due_at')
        ->get();

`whereToday`、`whereBeforeToday`、`whereAfterToday`メソッドは、それぞれカラムの値が今日であるか、昨日以前か、明日以降かを判定するために使用します。

    $invoices = DB::table('invoices')
        ->whereToday('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereBeforeToday('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereAfterToday('due_at')
        ->get();

同様に、`whereTodayOrBefore`と`whereTodayOrAfter`メソッドはカラムの値が今日以前か今日以降かを判断するために使用します。

    $invoices = DB::table('invoices')
        ->whereTodayOrBefore('due_at')
        ->get();

    $invoices = DB::table('invoices')
        ->whereTodayOrAfter('due_at')
        ->get();

**whereColumn／orWhereColumn**

`whereColumn`メソッドは、２つのカラムが等しい条件を加えます。

    $users = DB::table('users')
        ->whereColumn('first_name', 'last_name')
        ->get();

比較演算子を `whereColumn`メソッドに渡すこともできます。

    $users = DB::table('users')
        ->whereColumn('updated_at', '>', 'created_at')
        ->get();

カラム比較の配列を`whereColumn`メソッドに渡すこともできます。これらの条件は、`and`演算子を使用して結合されます。

    $users = DB::table('users')
        ->whereColumn([
            ['first_name', '=', 'last_name'],
            ['updated_at', '>', 'created_at'],
        ])->get();

<a name="logical-grouping"></a>
### 論理グループ化

クエリを論理的にグループ化するため、括弧内のいくつかの"where"句をグループ化したい場合があります。実際、予期外のクエリ動作を回避するため、`orWhere`メソッドへの呼び出しを通常は常に括弧内へグループ化する必要があります。これには、`where`メソッドにクロージャを渡します。

    $users = DB::table('users')
        ->where('name', '=', 'John')
        ->where(function (Builder $query) {
            $query->where('votes', '>', 100)
                ->orWhere('title', '=', 'Admin');
        })
        ->get();

ご覧のとおり、クロージャを`where`メソッドへ渡すことで、クエリビルダへ制約のグループ化を開始するように指示しています。クロージャは、括弧グループ内に含める必要のある制約を構築するために使用するクエリビルダインスタンスを受け取ります。上記の例では、次のSQLが生成されます。

```sql
select * from users where name = 'John' and (votes > 100 or title = 'Admin')
```

> [!WARNING]
> グローバルスコープを適用する場合の予期しない動作を回避するために、常に`orWhere`呼び出しをグループ化する必要があります。

<a name="advanced-where-clauses"></a>
## 上級WHERE節

<a name="where-exists-clauses"></a>
### WHERE EXISTS句

`whereExists`メソッドを使用すると、"where exists"SQL句を記述できます。`whereExists`メソッドは、クエリビルダインスタンスを受け取るクロージャを引数に取り、"exists"句内へ配置するクエリを定義します。

    $users = DB::table('users')
        ->whereExists(function (Builder $query) {
            $query->select(DB::raw(1))
                ->from('orders')
                ->whereColumn('orders.user_id', 'users.id');
        })
        ->get();

もしくは、クロージャの代わりにクエリオブジェクトを`whereExists`メソッドへ渡すこともできます。

    $orders = DB::table('orders')
        ->select(DB::raw(1))
        ->whereColumn('orders.user_id', 'users.id');

    $users = DB::table('users')
        ->whereExists($orders)
        ->get();

上記の例では、どちらも以下のようなSQLを出力します。

```sql
select * from users
where exists (
    select 1
    from orders
    where orders.user_id = users.id
)
```

<a name="subquery-where-clauses"></a>
### サブクエリWHERE句

サブクエリの結果を特定の値と比較する"where"句を作成したい場合があるでしょう。これには、クロージャと値を`where`メソッドへ渡してください。たとえば、以下のクエリは、特定のタイプの最近の「メンバーシップ（membership）」を持つすべてのユーザーを取得します。

    use App\Models\User;
    use Illuminate\Database\Query\Builder;

    $users = User::where(function (Builder $query) {
        $query->select('type')
            ->from('membership')
            ->whereColumn('membership.user_id', 'users.id')
            ->orderByDesc('membership.start_date')
            ->limit(1);
    }, 'Pro')->get();

もしくは、カラムをサブクエリの結果と比較する"where"句を作成したい場合もあるでしょう。これは、カラム、演算子、およびクロージャを`where`メソッドに渡すことで実現できます。たとえば、次のクエリは、金額が平均より少ないすべての収入レコードを取得します。

    use App\Models\Income;
    use Illuminate\Database\Query\Builder;

    $incomes = Income::where('amount', '<', function (Builder $query) {
        $query->selectRaw('avg(i.amount)')->from('incomes as i');
    })->get();

<a name="full-text-where-clauses"></a>
### フルテキストのWHERE句

> [!WARNING]
> フルテキストのwhere句は現在、MariaDB、MySQL、PostgreSQLをサポートしています。

`whereFullText`と`orWhereFullText`メソッドを使用すると、[フルテキストインデックス](/docs/{{version}}/migrations#available-index-types)を持つカラムヘのクエリに、フルテキストの"where"句を追加できます。これらのメソッドは、Laravelによって、利用するデータベースシステムに適したSQLへ変換されます。例えば、MariaDBやMySQLを利用するアプリケーションでは、`MATCH AGAINST`句を生成します。

    $users = DB::table('users')
        ->whereFullText('bio', 'web developer')
        ->get();

<a name="ordering-grouping-limit-and-offset"></a>
## 順序、グループ化、件数制限、オフセット

<a name="ordering"></a>
### 順序

<a name="orderby"></a>
#### `orderBy`メソッド

`orderBy`メソッドを使用すると、クエリの結果を特定のカラムで並べ替えできます。`orderBy`メソッドの最初の引数は、並べ替えるカラムです。２番目の引数は、並べ替えの方向を決定し、`asc`または`desc`のいずれかです。

    $users = DB::table('users')
        ->orderBy('name', 'desc')
        ->get();

複数の列で並べ替えるには、必要な回数の`orderBy`を呼び出すだけです。

    $users = DB::table('users')
        ->orderBy('name', 'desc')
        ->orderBy('email', 'asc')
        ->get();

<a name="latest-oldest"></a>
#### `latest`と`oldest`メソッド

`latest`および`oldest`メソッドを使用すると、結果を日付順に簡単に並べ替えできます。デフォルトでは、結果をテーブルの`created_at`カラムによって順序付けします。もしくは、並べ替えるカラム名を渡すこともできます。

    $user = DB::table('users')
        ->latest()
        ->first();

<a name="random-ordering"></a>
#### ランダム順

`inRandomOrder`メソッドを使用して、クエリ結果をランダムに並べ替えできます。たとえば、このメソッドを使用して、ランダムにユーザーをフェッチできます。

    $randomUser = DB::table('users')
        ->inRandomOrder()
        ->first();

<a name="removing-existing-orderings"></a>
#### 既存の順序の削除

`reorder`メソッドは、以前にクエリへ指定したすべての"order by"句を削除します。

    $query = DB::table('users')->orderBy('name');

    $unorderedUsers = $query->reorder()->get();

`reorder`メソッドを呼び出すときにカラムと方向を渡して、既存の"order by"句をすべて削除しから、クエリにまったく新しい順序を適用することもできます。

    $query = DB::table('users')->orderBy('name');

    $usersOrderedByEmail = $query->reorder('email', 'desc')->get();

<a name="grouping"></a>
### グループ化

<a name="groupby-having"></a>
#### `groupBy`と`having`メソッド

ご想像のとおり、`groupBy`メソッドと`having`メソッドを使用してクエリ結果をグループ化できます。`having`メソッドの引数は`where`メソッドの引数の使い方に似ています。

    $users = DB::table('users')
        ->groupBy('account_id')
        ->having('account_id', '>', 100)
        ->get();

havingBetween`メソッドを使うと、指定した範囲内の結果をフィルタリングできます。

    $report = DB::table('orders')
        ->selectRaw('count(id) as number_of_orders, customer_id')
        ->groupBy('customer_id')
        ->havingBetween('number_of_orders', [5, 15])
        ->get();

`groupBy`メソッドに複数の引数を渡して、複数のカラムでグループ化できます。

    $users = DB::table('users')
        ->groupBy('first_name', 'status')
        ->having('account_id', '>', 100)
        ->get();

より高度な`having`ステートメントを作成するには、[`havingRaw`](#raw-methods)メソッドを参照してください。

<a name="limit-and-offset"></a>
### 件数制限とオフセット

<a name="skip-take"></a>
#### `skip`と`take`メソッド

`skip`メソッドと`take`メソッドを使用して、クエリから返される結果の数を制限したり、クエリ内の特定の数の結果をスキップしたりできます。

    $users = DB::table('users')->skip(10)->take(5)->get();

または、`limit`メソッドと`offset`メソッドを使用することもできます。これらのメソッドは、それぞれ「take」メソッドと「skip」メソッドと機能的に同等です。

    $users = DB::table('users')
        ->offset(10)
        ->limit(5)
        ->get();

<a name="conditional-clauses"></a>
## 条件節

特定のクエリ句を別の条件に基づいてクエリに適用したい場合があります。たとえば、指定された入力値が受信HTTPリクエストに存在する場合にのみ、`where`ステートメントを適用したい場合です。これは、`when`メソッドを使用して実現可能です。

    $role = $request->input('role');

    $users = DB::table('users')
        ->when($role, function (Builder $query, string $role) {
            $query->where('role_id', $role);
        })
        ->get();

`when`メソッドは、最初の引数が`true`の場合にのみ、指定されたクロージャを実行します。最初の引数が`false`の場合、クロージャは実行されません。したがって、上記の例では、`when`メソッドに指定されたクロージャは、`role`フィールドが受信リクエストに存在し、`true`と評価された場合にのみ呼び出されます。

`when`メソッドの３番目の引数として別のクロージャを渡すことができます。このクロージャは、最初の引数が「false」と評価された場合にのみ実行されます。この機能の使用方法を説明するために、クエリのデフォルトの順序を設定してみます。

    $sortByVotes = $request->boolean('sort_by_votes');

    $users = DB::table('users')
        ->when($sortByVotes, function (Builder $query, bool $sortByVotes) {
            $query->orderBy('votes');
        }, function (Builder $query) {
            $query->orderBy('name');
        })
        ->get();

<a name="insert-statements"></a>
## INSERT文

クエリビルダは、データベーステーブルにレコードを挿入するために使用できる`insert`メソッドも提供します。`insert`メソッドは、カラム名と値の配列を引数に取ります。

    DB::table('users')->insert([
        'email' => 'kayla@example.com',
        'votes' => 0
    ]);

二重の配列を渡すことにより、一度に複数のレコードを挿入できます。各配列は、テーブルに挿入する必要のあるレコードを表します。

    DB::table('users')->insert([
        ['email' => 'picard@example.com', 'votes' => 0],
        ['email' => 'janeway@example.com', 'votes' => 0],
    ]);

`insertOrIgnore`メソッドは、データベースにレコードを挿入する際に、発生するエラーを無視します。このメソッドを使う場合は、重複したレコードのエラーが無視され、データベースエンジンにより他のタイプのエラーも無視されることを留意してください。たとえば、`insertOrIgnore`は、[MySQLのstrictモードをバイパスします](https://dev.mysql.com/doc/refman/en/sql-mode.html#ignore-effect-on-execution)。

    DB::table('users')->insertOrIgnore([
        ['id' => 1, 'email' => 'sisko@example.com'],
        ['id' => 2, 'email' => 'archer@example.com'],
    ]);

`insertUsing`メソッドは、サブクエリを使用して挿入するべきデータを判定しながら、テーブルに新しいレコードを挿入します。

    DB::table('pruned_users')->insertUsing([
        'id', 'name', 'email', 'email_verified_at'
    ], DB::table('users')->select(
        'id', 'name', 'email', 'email_verified_at'
    )->where('updated_at', '<=', now()->subMonth()));

<a name="auto-incrementing-ids"></a>
#### 自動増分ID

テーブルに自動増分IDがある場合は、`insertGetId`メソッドを使用してレコードを挿入してから、IDを取得します。

    $id = DB::table('users')->insertGetId(
        ['email' => 'john@example.com', 'votes' => 0]
    );

> [!WARNING]
> PostgreSQLを使用する場合、`insertGetId`メソッドは自動増分カラムが`id`という名前であると想定します。別の「シーケンス」からIDを取得する場合は、`insertGetId`メソッドの第２パラメータとしてカラム名を渡してください。

<a name="upserts"></a>
### UPSERTS

`upsert` (update+insert)メソッドは、存在しない場合はレコードを挿入し、指定した新しい値でレコードを更新します。メソッドの最初の引数は、挿入または更新する値で構成され、２番目の引数は、関連付けたテーブル内のレコードを一意に識別するカラムをリストします。第３引数は、一致するレコードがデータベースですでに存在する場合に、更新する必要があるカラムの配列です。

    DB::table('flights')->upsert(
        [
            ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
            ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
        ],
        ['departure', 'destination'],
        ['price']
    );

上記の例では、Laravelは２つのレコードを挿入しようとします。同じ`departure`カラムと`destination`カラムの値を持つレコードがすでに存在する場合、Laravelはそのレコードの`price`カラムを更新します。

> [!WARNING]
> SQL Serverを除くすべてのデータベースでは、`upsert`メソッドの２番目の引数のカラムに「プライマリ」または「ユニーク」インデックスが必要です。また、MariaDBとMySQLデータベースドライバは、`upert`メソッドの第２引数を無視し、常にテーブルの「プライマリ」および「ユニーク」インデックスを使用して既存のレコードを検出します。

<a name="update-statements"></a>
## UPDATE文

データベースにレコードを挿入することに加え、クエリビルダは`update`メソッドを使用して既存のレコードを更新することもできます。`update`メソッドは、`insert`メソッドと同様に、更新するカラムを示すカラムと値のペアの配列を受け入れます。`update`メソッドは、影響を受けた行数を返します。`where`句を使用して`update`クエリを制約できます。

    $affected = DB::table('users')
        ->where('id', 1)
        ->update(['votes' => 1]);

<a name="update-or-insert"></a>
#### UpdateかInsert

データベース内の既存のレコードを更新、もしくは一致するレコードが存在しない場合は作成したい場合もあるでしょう。このシナリオでは、`updateOrInsert`メソッドを使用できます。`updateOrInsert`メソッドは、２つの引数を受け入れます。レコードを検索するための条件の配列と、更新するカラムを示すカラムと値のペアの配列です。

`updateOrInsert`メソッドは、最初の引数のカラムと値のペアを使用して、一致するデータベースレコードを見つけようとします。レコードが存在する場合は、２番目の引数の値で更新します。レコードが見つからない場合は、両方の引数の属性をマージした新しいレコードを挿入します。

    DB::table('users')
        ->updateOrInsert(
            ['email' => 'john@example.com', 'name' => 'John'],
            ['votes' => '2']
        );

`updateOrInsert`メソッドにクロージャを渡し、マッチするレコードの存在に基づいて、データベースに更新または挿入する属性をカスタマイズできます。

```php
DB::table('users')->updateOrInsert(
    ['user_id' => $user_id],
    fn ($exists) => $exists ? [
        'name' => $data['name'],
        'email' => $data['email'],
    ] : [
        'name' => $data['name'],
        'email' => $data['email'],
        'marketable' => true,
    ],
);
```

<a name="updating-json-columns"></a>
### JSONカラムの更新

JSONカラムを更新する場合は、JSONオブジェクトの適切なキーを更新するため、`->`構文を使用する必要があります。この操作は、MariaDB10.3以上、MySQL5.7以上、PostgreSQL9.5以上をサポートしています。

    $affected = DB::table('users')
        ->where('id', 1)
        ->update(['options->enabled' => true]);

<a name="increment-and-decrement"></a>
### 増分と減分

クエリビルダは、特定のカラムの値を増分または減分するための便利なメソッドも提供します。これらのメソッドは両方とも、少なくとも1つの引数(変更するカラム)を受け入れます。カラムを増分／減分する量を指定するために、２番目の引数を指定できます。

    DB::table('users')->increment('votes');

    DB::table('users')->increment('votes', 5);

    DB::table('users')->decrement('votes');

    DB::table('users')->decrement('votes', 5);

必要に応じ、増分または減分操作中に更新するカラムを追加で指定することもできます。

    DB::table('users')->increment('votes', 1, ['name' => 'John']);

さらに、`incrementEach`と`decrementEach`メソッドを使用すると、一度に複数のカラムを増分または減分できます。

    DB::table('users')->incrementEach([
        'votes' => 5,
        'balance' => 100,
    ]);

<a name="delete-statements"></a>
## DELETE文

クエリビルダの`delete`メソッドを使用して、テーブルからレコードを削除できます。`delete`メソッドは影響を受けた行数を返します。`delete`メソッドを呼び出す前に"where"句を追加することで、`delete`ステートメントを制約できます。

    $deleted = DB::table('users')->delete();

    $deleted = DB::table('users')->where('votes', '>', 100)->delete();

<a name="pessimistic-locking"></a>
## 悲観的ロック

クエリビルダには、`select`ステートメントを実行するときに「悲観的ロック」を行うために役立つ関数も含まれています。「共有ロック」を使用してステートメントを実行するには、`sharedLock`メソッドを呼び出すことができます。共有ロックは、トランザクションがコミットされるまで、選択した行が変更されないようにします。

    DB::table('users')
        ->where('votes', '>', 100)
        ->sharedLock()
        ->get();

または、`lockForUpdate`メソッドを使用することもできます。「更新用」ロックは、選択したレコードが変更されたり、別の共有ロックで選択されたりするのを防ぎます。

    DB::table('users')
        ->where('votes', '>', 100)
        ->lockForUpdate()
        ->get();

義務ではありませんが、悲観的ロックは[トランザクション](/docs/{{version}}/database#database-transactions)の中へラップすることを推奨します。これにより、操作全体が完了するまで、取得したデータがデータベース内で変更されないことが保証されます。失敗した場合、トランザクションは変更をロールバックし、ロックを自動的に解放します。

    DB::transaction(function () {
        $sender = DB::table('users')
            ->lockForUpdate()
            ->find(1);

        $receiver = DB::table('users')
            ->lockForUpdate();
            ->find(2);

        if ($sender->balance < 100) {
            throw new RuntimeException('Balance too low.');
        }

        DB::table('users')
            ->where('id', $sender->id)
            ->update([
                'balance' => $sender->balance - 100
            ]);

        DB::table('users')
            ->where('id', $receiver->id)
            ->update([
                'balance' => $receiver->balance + 100
            ]);
    });

<a name="debugging"></a>
## デバッグ

クエリの作成中に`dd`メソッドと`dump`メソッドを使用して、現在のクエリバインディングとSQLをダンプできます。`dd`メソッドはデバッグ情報を表示してから、リクエストの実行を停止します。`dump`メソッドはデバッグ情報を表示しますが、リクエストの実行を継続できます。

    DB::table('users')->where('votes', '>', 100)->dd();

    DB::table('users')->where('votes', '>', 100)->dump();

`dumpRawSql`メソッドと`ddRawSql`メソッドをクエリに対して呼び出すと、すべてのパラメータバインディングを適切に置換した状態で、クエリのSQLをダンプできます。。

    DB::table('users')->where('votes', '>', 100)->dumpRawSql();

    DB::table('users')->where('votes', '>', 100)->ddRawSql();
