# コンテキスト

- [イントロダクション](#introduction)
    - [動作の仕組み](#how-it-works)
- [コンテキストのチャプタ](#capturing-context)
    - [スタック](#stacks)
- [コンテキストの取得](#retrieving-context)
    - [アイテムの存在判定](#determining-item-existence)
- [コンテキストの削除](#removing-context)
- [隠しコンテキスト](#hidden-context)
- [イベント](#events)
    - [分離](#dehydrating)
    - [合成](#hydrated)

<a name="introduction"></a>
## イントロダクション

Laravelの「コンテキスト」機能により、アプリケーション内で実行されているリクエスト、ジョブ、コマンド実行に渡る情報をキャプチャ、取得、共有ができます。このキャプチャした情報は、アプリケーションが書き込むログにも含まれ、ログエントリが書き込まれる前に発生していた周辺のコード実行履歴をより深く理解することができ、分散システム全体の実行フローをトレースできるようにします。

<a name="how-it-works"></a>
### 動作の仕組み

Laravelのコンテキスト機能を理解する最良の方法は、組み込まれているログ機能を使い、実際に見てみることです。開始するには、`Context`ファサードを使い、[コンテキストへ情報を追加](#capturing-context)してください。この例では、[ミドルウェア](/docs/{{version}}/middleware)を使い、リクエストURLと一意なトレースIDを受信リクエストごとにコンテキストへ追加してみます。

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Str;
use Symfony\Component\HttpFoundation\Response;

class AddContext
{
    /**
     * 受信リクエストの処理
     */
    public function handle(Request $request, Closure $next): Response
    {
        Context::add('url', $request->url());
        Context::add('trace_id', Str::uuid()->toString());

        return $next($request);
    }
}
```

コンテキストへ追加した情報は、リクエストを通して書き込まれる [ログエントリ](/docs/{{version}}/loggin)へメタデータとして、自動的に追加します。コンテキストをメタデータとして追加することにより、個々のログエントリへ渡す情報と、`Context`を介して共有する情報を区別できるようになります。例えば、次のようなログエントリを書くとします。

```php
Log::info('User authenticated.', ['auth_id' => Auth::id()]);
```

書き出すログは、ログエントリに渡した`auth_id`を含みますが、メタデータとしてコンテキストの`url`と`trace_id`もふくんでいます。

```
User authenticated. {"auth_id":27} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

コンテキストへ追加する情報は、キュー投入するジョブでも利用できるようになります。例えば、`ProcessPodcast`ジョブをコンテキストへ情報を追加した後で、キュー投入するとしましょう。

```php
// ミドルウェア中
Context::add('url', $request->url());
Context::add('trace_id', Str::uuid()->toString());

// コントローラ中
ProcessPodcast::dispatch($podcast);
```

ジョブを投入すると、コンテキストに現在格納しているすべての情報をキャプチャし、ジョブと共有します。取り込こんだ情報は、ジョブの実行中に現在のコンテキストへ戻します。では、ジョブの処理メソッドがログへの書き込みであったとしましょう。

```php
class ProcessPodcast implements ShouldQueue
{
    use Queueable;

    // ...

    /**
     * ジョブの実行
     */
    public function handle(): void
    {
        Log::info('Processing podcast.', [
            'podcast_id' => $this->podcast->id,
        ]);

        // ...
    }
}
```

結果のログエントリは、ジョブ投入元のリクエストの間に、コンテキストへ追加した情報を含みます。

```
Processing podcast. {"podcast_id":95} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

Laravelのコンテキストの機能と関連して、組み込みログへ焦点を当てましたが、以下のドキュメントでは、コンテキストによってHTTPリクエスト／キュー投入ジョブの境界を越えて情報を共有する方法や、ログエントリーと一緒に書き込まない[隠しコンテキストデータ](#hidden-context)を追加する方法まで説明します。

<a name="capturing-context"></a>
## コンテキストのチャプタ

`Context`ファサードの`add`メソッドを使用して、現在のコンテキストへ情報を格納できます。

```php
use Illuminate\Support\Facades\Context;

Context::add('key', 'value');
```

一度に複数のアイテムを追加するには、`add`メソッドに連想配列を渡してください。

```php
Context::add([
    'first_key' => 'value',
    'second_key' => 'value',
]);
```

`add`メソッドは、同じキーを持つ既存の値を上書きします。そのキーがまだ存在しない場合にのみコンテキストへ情報を追加したい場合は、`addIf`メソッドを使用してください。

```php
Context::add('key', 'first');

Context::get('key');
// "first"

Context::addIf('key', 'second');

Context::get('key');
// "first"
```

<a name="conditional-context"></a>
#### 条件付きコンテキスト

`when`メソッドは、指定する条件に基づき、コンテキストへデータを追加するために使用します。指定条件を`true`と評価した場合、`when`メソッドに渡す最初のクロージャを呼び出し、`false`と評価した場合は、２番目のクロージャを呼び出します。

```php
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Context;

Context::when(
    Auth::user()->isAdmin(),
    fn ($context) => $context->add('permissions', Auth::user()->permissions),
    fn ($context) => $context->add('permissions', []),
);
```

<a name="stacks"></a>
### スタック

Contextには「スタック」を作成する機能があります。「スタック」は追加した順番で保存したデータのリストです。スタックへ情報を追加するには、`push`メソッドを呼び出します。

```php
use Illuminate\Support\Facades\Context;

Context::push('breadcrumbs', 'first_value');

Context::push('breadcrumbs', 'second_value', 'third_value');

Context::get('breadcrumbs');
// [
//     'first_value',
//     'second_value',
//     'third_value',
// ]
```

スタックは、アプリケーション全体で起きているイベントのような、リクエストに関する履歴情報を取得するのに便利です。たとえば、イベントリスナを作成して、クエリが実行されるたびにスタックへプッシュし、クエリSQLと期間をタプルとして取得できます。

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Facades\DB;

DB::listen(function ($event) {
    Context::push('queries', [$event->time, $event->sql]);
});
```

スタックに値があるかは、`stackContains`と`hiddenStackContains`メソッドで調べられる。

```php
if (Context::stackContains('breadcrumbs', 'first_value')) {
    //
}

if (Context::hiddenStackContains('secrets', 'first_value')) {
    //
}
```

`stackContains`と`hiddenStackContains`メソッドは、第２引数へクロージャも受け付ける。

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Str;

return Context::stackContains('breadcrumbs', function ($value) {
    return Str::startsWith($value, 'query_');
});
```

<a name="retrieving-context"></a>
## コンテキストの取得

コンテキストから情報を取得するには、`Context`ファサードの`get`メソッドを使用します。

```php
use Illuminate\Support\Facades\Context;

$value = Context::get('key');
```

`only`メソッドは、コンテキスト内の情報のサブセットを取得するために使用します。

```php
$data = Context::only(['first_key', 'second_key']);
```

`pull`メソッドは、コンテキストから情報を取得し、すぐにコンテキストから削除するために使用します。

```php
$value = Context::pull('key');
```

コンテキストが格納しているすべての情報を取得したい場合は、`all` メソッドを呼び出します。

```php
$data = Context::all();
```

<a name="determining-item-existence"></a>
### アイテムの存在判定

指定するキーに対応する値をコンテキストに格納しているかを調べるには、`has`メソッドを使用します。

```php
use Illuminate\Support\Facades\Context;

if (Context::has('key')) {
    // ...
}
```

`has`メソッドは、格納している値に関係なく存在しているなら`true`を返します。そのため、例えば`null`値を持つキーでも存在しているとみなします。

```php
Context::add('key', null);

Context::has('key');
// true
```

<a name="removing-context"></a>
## コンテキストの削除

`forget`メソッドを使うと、現在のコンテキストからキーとその値を削除できます。

```php
use Illuminate\Support\Facades\Context;

Context::add(['first_key' => 1, 'second_key' => 2]);

Context::forget('first_key');

Context::all();

// ['second_key' => 2]
```

`forget`メソッドに配列を指定すれば、複数のキーを一度に削除できます。

```php
Context::forget(['first_key', 'second_key']);
```

<a name="hidden-context"></a>
## 隠しコンテキスト

コンテキストは「隠し」データを保存する機能を提供しています。この隠し情報はログに追加せず、上記で説明したデータ検索方法ではアクセスできません。コンテキストは、隠しコンテキスト情報を操作するために別のメソッドを提供しています。

```php
use Illuminate\Support\Facades\Context;

Context::addHidden('key', 'value');

Context::getHidden('key');
// 'value'

Context::get('key');
// null
```

"hidden"メソッドは、前述の非隠しメソッドの機能を反映したものです。

```php
Context::addHidden(/* ... */);
Context::addHiddenIf(/* ... */);
Context::pushHidden(/* ... */);
Context::getHidden(/* ... */);
Context::pullHidden(/* ... */);
Context::onlyHidden(/* ... */);
Context::allHidden(/* ... */);
Context::hasHidden(/* ... */);
Context::forgetHidden(/* ... */);
```

<a name="events"></a>
## イベント

コンテキストは、コンテキストの合成と分離プロセスをフックできる、２つのイベントをディスパッチします。

これらのイベントがどのように使うかを説明するため、アプリケーションのミドルウェアで、HTTPリクエストの`Accept-Language`ヘッダに基づいて、`app.locale`設定値を設定する事を考えてみましょう。コンテキストのイベントにより、リクエスト中にこの値を取得し、キューへリストアすることができ、キューに送信する通知が正しい`app.locale`値を持つようにできます。これを実現するには、コンテキストのイベントと [隠し](#hidden-context)データが使えます。

<a name="dehydrating"></a>
### 分離

ジョブをキューへディスパッチするたびに、コンテキストのデータは「分離（dehtdrated）」され、ジョブのペイロードと一緒に取り込まれます。`Context::dehydrating`メソッドでは、分離処理中に呼び出すクロージャを登録できます。このクロージャの中で、キュー投入したジョブと共有するデータへ変更を加えることができます。

通常、アプリケーションの`AppServiceProvider`クラスの`boot`メソッド内で、`dehydrating` コールバックを登録すべきでしょう。

```php
use Illuminate\Log\Context\Repository;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\Context;

/**
 * アプリケーションの全サービスの初期起動処理
 */
public function boot(): void
{
    Context::dehydrating(function (Repository $context) {
        $context->addHidden('locale', Config::get('app.locale'));
    });
}
```

> [!NOTE]  
> `dehydrating`コールバック内で、`Context`ファサードを使用してはいけません。コールバックへ渡されたリポジトリにのみ、変更を加えるようにしてください。

<a name="hydrated"></a>
### 合成

キュー投入したジョブがキュー上で実行を開始するたび、そのジョブで共有されていたコンテキストは、現在のコンテキストへ「合成（hydrated）」します。`Context::hydrated`メソッドを使用すると、合成のプロセスで呼び出すクロージャを登録できます。

通常、アプリケーションの`AppServiceProvider`クラスの`boot`メソッド内で、`hydrated`コールバックを登録する必要があります。

```php
use Illuminate\Log\Context\Repository;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\Context;

/**
 * アプリケーションの全サービスの初期起動処理
 */
public function boot(): void
{
    Context::hydrated(function (Repository $context) {
        if ($context->hasHidden('locale')) {
            Config::set('app.locale', $context->getHidden('locale'));
        }
    });
}
```

> [!NOTE]  
> `Hydrated`コールバック内では`Context`ファサードを使用せず、代わりにコールバックへ渡すリポジトリにのみ変更を加えてください。
