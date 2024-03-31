# Laravel Pennant

- [イントロダクション](#introduction)
- [インストール](#installation)
- [設定](#configuration)
- [機能の定義](#defining-features)
    - [クラスベースの機能](#class-based-features)
- [機能のチェック](#checking-features)
    - [条件付き実行](#conditional-execution)
    - [ `HasFeatures`トレイト](#the-has-features-trait)
    - [Bladeディレクティブ](#blade-directive)
    - [ミドルウェア](#middleware)
    - [メモリ内キャッシュ](#in-memory-cache)
- [スコープ](#scope)
    - [スコープの指定](#specifying-the-scope)
    - [デフォルトスコープ](#default-scope)
    - [NULL許可のスコープ](#nullable-scope)
    - [スコープの識別子](#identifying-scope)
    - [スコープのシリアライズ](#serializing-scope)
- [機能のリッチな値](#rich-feature-values)
- [複数の機能の取得](#retrieving-multiple-features)
- [Eagerロード](#eager-loading)
- [値の更新](#updating-values)
    - [バルク更新](#bulk-updates)
    - [機能の削除](#purging-features)
- [テスト](#testing)
- [カスタム機能ドライバの追加](#adding-custom-pennant-drivers)
    - [ドライバの実装](#implementing-the-driver)
    - [ドライバの登録](#registering-the-driver)
- [イベント](#events)

<a name="introduction"></a>
## イントロダクション

[Laravel Pennant](https://github.com/laravel/pennant)（ペナント：三角旗）は無駄がない、シンプルで軽量な機能フラグパッケージです。機能フラグを使うことで、新しいアプリケーションの機能を躊躇なく段階的にロールアウトしたり、新しいインターフェイスデザインをＡ／Ｂテストしたり、トランクベースの開発戦略を推奨したり、その他多くのことができるようになります。

<a name="installation"></a>
## インストール

まず、Composerパッケージマネージャを使って、プロジェクトにPennantをインストールします。

```shell
composer require laravel/pennant
```

次に、`vendor:publish` Artisanコマンドを使用し、Pennantの設定ファイルとマイグレーションファイルをリソース公開する必要があります。

```shell
php artisan vendor:publish --provider="Laravel\Pennant\PennantServiceProvider"
```

最後に、アプリケーションのデータベースマイグレーションを実行してください。これにより、Pennantが`database`ドライバを動かすために使う、`features`テーブルが作成されます。

```shell
php artisan migrate
```

<a name="configuration"></a>
## 設定

Pennantのリソースを公開すると、その設定ファイルを`config/pennant.php`へ保存します。この設定ファイルでPennantがデフォルトとして使用する、算出済みの機能フラグ値を保存するストレージメカニズムを指定します。

Pennantは、算出済み機能フラグの値をメモリ内の配列へ格納する、`array`ドライバをサポートしています。もしくは、算出済み機能フラグ値を、リレーショナルデータベースに永続的に保存する、`database`ドライバも使用できます。(これはPennantで使用する、デフォルト保存メカニズムです。)

<a name="defining-features"></a>
## 機能の定義

機能を定義するには、`Feature`ファサードが提供する、`define`メソッドを使用します。機能の名前と、その機能の初期値を決定するため呼び出す、クロージャを指定する必要があります。

通常、機能は`Feature`ファサードを使用し、サービスプロバイダで定義します。クロージャは、機能チェックのための「スコープ」を引数に取ります。最も一般的なのは、現在認証しているユーザーをスコープにすることでしょう。この例では、アプリケーションのユーザーへ、新しいAPIを段階的に提供する機能を定義しています。

```php
<?php

namespace App\Providers;

use App\Models\User;
use Illuminate\Support\Lottery;
use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * 全アプリケーションサービスの初期起動処理
     */
    public function boot(): void
    {
        Feature::define('new-api', fn (User $user) => match (true) {
            $user->isInternalTeamMember() => true,
            $user->isHighTrafficCustomer() => false,
            default => Lottery::odds(1 / 100),
        });
    }
}
```

ご覧の通り、この機能では以下のようなルールを設けています。

- チーム内メンバーは全員、新しいAPIを使用できる。
- トラフィック量が多い顧客は、新しいAPIを使用できない。
- それ以外の場合、１／１００の確率で、ランダムに機能をアクティブにする。

最初に`new-api`機能を指定したユーザーに対してチェックしたら、クロージャの実行結果をストレージドライバへ保存します。次回、同じユーザーに対しこの機能をチェックするとき、値はストレージから取り出し、クロージャを呼び出しません。

使いやすいように、機能定義が抽選（lottery）を返すだけの場合は、クロージャを完全に省略できます。

    Feature::define('site-redesign', Lottery::odds(1, 1000));

<a name="class-based-features"></a>
### クラスベースの機能

Pennantでは、クラスベースで機能を定義することもできます。クロージャベースの機能定義とは異なり、クラスベースの機能は、サービスプロバイダに登録する必要がありません。クラスベースの機能を生成するには、`pennant:feature` Artisanコマンドを実行します。デフォルトで、機能クラスはアプリケーションの`app/Features`ディレクトリへ配置します。

```shell
php artisan pennant:feature NewApi
```

機能クラスを書く場合、`resolve`メソッドのみ定義する必要があります。このメソッドは、 指定したスコープに対する機能の初期値を解決するために呼び出されます。この場合も、スコープは通常、現在認証しているユーザーでしょう。

```php
<?php

namespace App\Features;

use Illuminate\Support\Lottery;

class NewApi
{
    /**
     * 機能の初期値を決める
     */
    public function resolve(User $user): mixed
    {
        return match (true) {
            $user->isInternalTeamMember() => true,
            $user->isHighTrafficCustomer() => false,
            default => Lottery::odds(1 / 100),
        };
    }
}
```

> [!NOTE] Feature classes are resolved via the [container](/docs/{{version}}/container), so you may inject dependencies into the feature class's constructor when needed.

#### 機能の保存名のカスタマイズ

デフォルトで、Pennantは機能クラスの完全修飾クラス名を保存します。保存する機能名をアプリケーションの内部構造から切り離したい場合は、機能クラスで`$name`プロパティを指定してください。このプロパティの値をクラス名の代わりに格納します。

```php
<?php

namespace App\Features;

class NewApi
{
    /**
     * 機能の保存名
     *
     * @var string
     */
    public $name = 'new-api';

    // ...
}
```

<a name="checking-features"></a>
## 機能のチェック

ある機能がアクティブであるかを判断するには、`Feature`ファサードの`active`メソッドを使用します。デフォルトで機能は、現在認証しているユーザーを対象にチェックします。

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Feature;

class PodcastController
{
    /**
     * リソースリストの表示
     */
    public function index(Request $request): Response
    {
        return Feature::active('new-api')
                ? $this->resolveNewApiResponse($request)
                : $this->resolveLegacyApiResponse($request);
    }

    // ...
}
```

デフォルトで機能は、現在認証しているユーザーに対してチェックしますが、別のユーザーや[スコープ](#scope)に対してチェックすることも簡単にできます。これを行うには、`Feature`ファサードの`for`メソッドを使用します。

```php
return Feature::for($user)->active('new-api')
        ? $this->resolveNewApiResponse($request)
        : $this->resolveLegacyApiResponse($request);
```

Pennantはさらに、機能がアクティブかを判断するのに役立つ、便利なメソッドをいくつか用意しています。

```php
// 指定機能がすべてアクティブであることを判断
Feature::allAreActive(['new-api', 'site-redesign']);

// 指定機能のうち、どれかがアクティブであることを判断
Feature::someAreActive(['new-api', 'site-redesign']);

// 特定機能がアクティブではないことを判断
Feature::inactive('new-api');

// 指定機能が全部アクティブではないことを判断
Feature::allAreInactive(['new-api', 'site-redesign']);

// 指摘機能のうち、どれかがアクティブでないことを判断
Feature::someAreInactive(['new-api', 'site-redesign']);
```

> [!NOTE]
> PennantをHTTPコンテキスト外で使う場合、例えばArtisanコマンドや、キュー投入したジョブでは、機能のスコープを通常[明示的に指定](#specifying-the-scope)する必要があります。あるいは、認証済みHTTPコンテキストと、認証されていないコンテキストの両方を考慮した、[デフォルトスコープ](#default-scope)を定義することもできます。

<a name="checking-class-based-features"></a>
#### クラスベース機能のチェック

クラスベースの機能の場合、機能をチェックするときにクラス名を指定する必要があります。

```php
<?php

namespace App\Http\Controllers;

use App\Features\NewApi;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Feature;

class PodcastController
{
    /**
     * リソースリストの表示
     */
    public function index(Request $request): Response
    {
        return Feature::active(NewApi::class)
                ? $this->resolveNewApiResponse($request)
                : $this->resolveLegacyApiResponse($request);
    }

    // ...
}
```

<a name="conditional-execution"></a>
### 条件付き実行

`when`メソッドは、機能がアクティブなときに、スムーズに指定クロージャを実行するために使います。また、2つ目のクロージャを指定し、機能が非アクティブの場合に実行させることもできます。

    <?php

    namespace App\Http\Controllers;

    use App\Features\NewApi;
    use Illuminate\Http\Request;
    use Illuminate\Http\Response;
    use Laravel\Pennant\Feature;

    class PodcastController
    {
        /**
         * リソースリストの表示
         */
        public function index(Request $request): Response
        {
            return Feature::when(NewApi::class,
                fn () => $this->resolveNewApiResponse($request),
                fn () => $this->resolveLegacyApiResponse($request),
            );
        }

        // ...
    }

`unless`メソッドは、`when`メソッドと逆の働きをし、その機能が非アクティブな場合、最初のクロージャを実行します。

    return Feature::unless(NewApi::class,
        fn () => $this->resolveLegacyApiResponse($request),
        fn () => $this->resolveNewApiResponse($request),
    );

<a name="the-has-features-trait"></a>
### `HasFeatures`トレイト

Pennantの`HasFeatures`トレイトは、アプリケーションの`User`モデル(あるいは、機能を持つ他のモデル)に追加し、モデルから直接機能をスムーズにチェックする、便利な手法を提供しています。

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Pennant\Concerns\HasFeatures;

class User extends Authenticatable
{
    use HasFeatures;

    // ...
}
```

このトレイトをモデルへ追加すれば、`features`メソッドを呼び出すことで、簡単に機能を確認できます。

```php
if ($user->features()->active('new-api')) {
    // ...
}
```

もちろん、`features`メソッドは、機能を操作する他の多くの便利なメソッドへのアクセスも提供します。

```php
// 値
$value = $user->features()->value('purchase-button')
$values = $user->features()->values(['new-api', 'purchase-button']);

// 状態
$user->features()->active('new-api');
$user->features()->allAreActive(['new-api', 'server-api']);
$user->features()->someAreActive(['new-api', 'server-api']);

$user->features()->inactive('new-api');
$user->features()->allAreInactive(['new-api', 'server-api']);
$user->features()->someAreInactive(['new-api', 'server-api']);

// 条件付き実行
$user->features()->when('new-api',
    fn () => /* ... */,
    fn () => /* ... */,
);

$user->features()->unless('new-api',
    fn () => /* ... */,
    fn () => /* ... */,
);
```

<a name="blade-directive"></a>
### Bladeディレクティブ

Blade内でも機能をシームレスにチェックするため、Pennantは`@feature`ディレクティブを提供します。

```blade
@feature('site-redesign')
    <!-- 'site-redesign'はアクティブ -->
@else
    <!-- 'site-redesign'は非アクティブ -->
@endfeature
```

<a name="middleware"></a>
### ミドルウェア

Pennantは、[ミドルウェア](/docs/{{version}}/middleware)も用意しています。このミドルウェアは、ルートが呼び出される前に、現在認証されているユーザーがその機能にアクセスできることを確認するために使います。ミドルウェアをルートに割り当て、ルートにアクセスするために必要な機能を指定ができます。指定した機能のどれかが、現在認証されているユーザーにとって無効である場合、ルートは`400 Bad Request` HTTPレスポンスを返します。静的メソッド`using`には複数の機能を渡せます。

```php
use Illuminate\Support\Facades\Route;
use Laravel\Pennant\Middleware\EnsureFeaturesAreActive;

Route::get('/api/servers', function () {
    // ...
})->middleware(EnsureFeaturesAreActive::using('new-api', 'servers-api'));
```

<a name="customizing-the-response"></a>
#### レスポンスのカスタマイズ

リスト中の機能が非アクティブのときにミドルウェアが返すレスポンスをカスタマイズしたい場合は、`EnsureFeaturesAreActive`ミドルウェアが提供する、`whenInactive`メソッドを利用してください。通常、このメソッドはアプリケーションのサービスプロバイダの`boot`メソッド内で呼び出す必要があります。

```php
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Middleware\EnsureFeaturesAreActive;

/**
 * 全アプリケーションサービスの初期起動処理
 */
public function boot(): void
{
    EnsureFeaturesAreActive::whenInactive(
        function (Request $request, array $features) {
            return new Response(status: 403);
        }
    );

    // ...
}
```

<a name="in-memory-cache"></a>
### メモリ内キャッシュ

機能をチェックすると、Pennantはその結果のメモリ内キャッシュを作成します。`database`ドライバを使っている場合、これは同じ機能フラグを一つのリクエストで再チェックしても、追加のデータベースクエリが発生しないことを意味します。これはまた、その機能がリクエストの間、一貫した結果を持つことを保証します。

メモリ内のキャッシュを手作業で消去する必要がある場合は、`Feature`ファサードが提供する`flushCache`メソッドを使用してください。

    Feature::flushCache();

<a name="scope"></a>
## スコープ

<a name="specifying-the-scope"></a>
### スコープの指定

説明してきたように、機能は通常、現在認証しているユーザーに対してチェックされます。しかし、これは必ずしもあなたのニーズに合うとは限りません。そのため、`Feature`ファサードの`for`メソッドでは、ある機能をチェックする対象のスコープを指定できます。

```php
return Feature::for($user)->active('new-api')
        ? $this->resolveNewApiResponse($request)
        : $this->resolveLegacyApiResponse($request);
```

もちろん、機能のスコープは、「ユーザー」に限定されません。新しい課金システムを構築しており、個々のユーザーではなく、チーム全体にロールアウトしていると想像してみてください。たぶん、古いチームには、新しいチームよりもゆっくりとロールアウトしたいと思うでしょう。この機能解決のクロージャは、次のようなものになります。

```php
use App\Models\Team;
use Carbon\Carbon;
use Illuminate\Support\Lottery;
use Laravel\Pennant\Feature;

Feature::define('billing-v2', function (Team $team) {
    if ($team->created_at->isAfter(new Carbon('1st Jan, 2023'))) {
        return true;
    }

    if ($team->created_at->isAfter(new Carbon('1st Jan, 2019'))) {
        return Lottery::odds(1 / 100);
    }

    return Lottery::odds(1 / 1000);
});
```

定義したクロージャは、`User`を想定しておらず、代わりに`Team`モデルを想定していることにお気づきでしょう。この機能がユーザーのチームに対してアクティブかを判断するには、`Feature`ファサードの`for`メソッドへ、チームを渡す必要があります。

```php
if (Feature::for($user->team)->active('billing-v2')) {
    return redirect()->to('/billing/v2');
}

// ...
```

<a name="default-scope"></a>
### デフォルトスコープ

Pennant が機能をチェックするのに使うデフォルトのスコープをカスタマイズすることも可能です。たとえば、すべての機能をユーザーではなく、現在認証しているユーザーのチームに対してチェックするとします。機能をチェックするたびに、`Feature::for($user->team)`を毎回呼び出す代わりに、チームをデフォルトのスコープとして指定できます。一般に、これはアプリケーションのいずれかのサービスプロバイダで行う必要があります。

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Auth;
use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * 全アプリケーションサービスの初期起動処理
     */
    public function boot(): void
    {
        Feature::resolveScopeUsing(fn ($driver) => Auth::user()?->team);

        // ...
    }
}
```

これにより、`for`メソッドで明示的にスコープを指定しなかった場合、機能チェックでは現在認証しているユーザーのチームをデフォルトのスコープとして使用するようになりました。

```php
Feature::active('billing-v2');

// 上記コードが以下と同じ動作をするようになった

Feature::for($user->team)->active('billing-v2');
```

<a name="nullable-scope"></a>
### NULL許可のスコープ

もし、ある機能をチェックするときに指定したスコープで、nullable型または`null`を含むunion型により、機能の定義で`null`をサポートしていない場合、Pennantは自動的にその機能の結果値として`false`を返します。

したがって、機能に渡すスコープが`null`になる可能性があり、その機能の値リゾルバを呼び出したい場合は、機能の定義でこれを考慮する必要があります。`null`スコープは、Artisan コマンド、キュー投入したジョブ、もしくは未認証のルート内で機能をチェックする場合に発生する可能性があります。これらのコンテキストでは通常、認証済みユーザーが存在しないため、デフォルトのスコープは `null` になります。

もし、常に[機能のスコープを明示的に指定](#specifying-the-scope)しないのであれば、スコープのタイプを"nullable"にし、機能定義のロジック内で`null` スコープ値を確実に処理してください。

```php
use App\Models\User;
use Illuminate\Support\Lottery;
use Laravel\Pennant\Feature;

Feature::define('new-api', fn (User $user) => match (true) {// [tl! remove]
Feature::define('new-api', fn (User|null $user) => match (true) {// [tl! add]
    $user === null => true,// [tl! add]
    $user->isInternalTeamMember() => true,
    $user->isHighTrafficCustomer() => false,
    default => Lottery::odds(1 / 100),
});
```

<a name="identifying-scope"></a>
### スコープの識別子

Pennant我用意している`array`と`database`ストレージドライバは、すべてのPHPデータ型とEloquentモデルのスコープ識別子を適切に保存する方法を知っています。しかし、サードパーティのPennantドライバを使用する場合、そのドライバはEloquentモデルやその他のカスタム型に対する識別子を正しく格納する方法を知らない可能性があります。

こうした観点から、Pennantは、アプリケーションの中でPennantのスコープとして使用するオブジェクトへ、`FeatureScopeable`契約を実装することにより、スコープの値を保存用にフォーマットできるようにしました。

例えば、1つのアプリケーションで2つの異なる機能ドライバを使用しているとします。組み込みの`database`ドライバと、サードパーティの"Flag Rocket"ドライバです。"Flag Rocket"ドライバはEloquentモデルを適切に保存する方法を知りません。代わりに、`FlagRocketUser`インスタンスを必要とします。`FeatureScopeable`が定義する`toFeatureIdentifier`を実装し、アプリケーションで使用する各ドライバに提供する保存可能なスコープの値をカスタマイズできます。

```php
<?php

namespace App\Models;

use FlagRocket\FlagRocketUser;
use Illuminate\Database\Eloquent\Model;
use Laravel\Pennant\Contracts\FeatureScopeable;

class User extends Model implements FeatureScopeable
{
    /**
     * オブジェクトを指定されたドライバの機能スコープ識別子にキャスト
     */
    public function toFeatureIdentifier(string $driver): mixed
    {
        return match($driver) {
            'database' => $this,
            'flag-rocket' => FlagRocketUser::fromId($this->flag_rocket_id),
        };
    }
}
```

<a name="serializing-scope"></a>
### スコープのシリアライズ

PennantはEloquentモデルに関連付けた機能を格納するとき、デフォルトで完全修飾クラス名を使います。[Eloquentモーフィックマップ](/docs/{{version}}/eloquent-relationships#custom-polymorphic-types)を使っている場合は、Pennantでもモーフィックマップを使い、保存した機能をアプリケーションの構造から切り離せます。

これを行なうには、サービスプロバイダでEloquentモーフマップを定義した後に、`Feature`ファサードの`useMorphMap`メソッドを呼び出します。

```php
use Illuminate\Database\Eloquent\Relations\Relation;
use Laravel\Pennant\Feature;

Relation::enforceMorphMap([
    'post' => 'App\Models\Post',
    'video' => 'App\Models\Video',
]);

Feature::useMorphMap();
```

<a name="rich-feature-values"></a>
## 機能のリッチな値

ここまで、機能は主にバイナリ状態、つまり「アクティブ」か「非アクティブ」かで取り扱ってきましたが、Pennantはリッチな値も格納できます。

例えば、アプリケーションの「Buy now」ボタンに3つの新しい色をテストする場合を考えてみましょう。機能定義から`true`や`false`を返す代わりに、文字列を返せます。

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn (User $user) => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

`value`メソッドを使用すると、`purchase-button`機能の値を取得できます。

```php
$color = Feature::value('purchase-button');
```

また、Pennantが用意しているBladeディレクティブでは簡単に、機能の現在の値に基づいて、条件付きでコンテンツをレンダできます。

```blade
@feature('purchase-button', 'blue-sapphire')
    <!-- 'blue-sapphire' is active -->
@elsefeature('purchase-button', 'seafoam-green')
    <!-- 'seafoam-green' is active -->
@elsefeature('purchase-button', 'tart-orange')
    <!-- 'tart-orange' is active -->
@endfeature
```

> [!NOTE] When using rich values, it is important to know that a feature is considered "active" when it has any value other than `false`.

[条件付き`when`](#conditional-execution)メソッドを呼び出すと、その機能のリッチな値が最初のクロージャに提供されます。

    Feature::when('purchase-button',
        fn ($color) => /* ... */,
        fn () => /* ... */,
    );

同様に、条件付きの`unless`メソッドを呼び出すと、その機能のリッチな値がオプションの２番目のクロージャへ渡されます。

    Feature::unless('purchase-button',
        fn () => /* ... */,
        fn ($color) => /* ... */,
    );

<a name="retrieving-multiple-features"></a>
## 複数の機能の取得

`value`メソッドで、指定スコープに対する複数の機能を取得できます。

```php
Feature::values(['billing-v2', 'purchase-button']);

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
// ]
```

もしくは、`all`メソッドを使用し、指定スコープに定義されているすべての機能の値を取得することもできます。

```php
Feature::all();

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
//     'site-redesign' => true,
// ]
```

しかし、クラスベースの機能は動的に登録されますので、明示的にチェックするまでPennantにはわかりません。つまり、アプリケーションのクラスベースの機能は、現在のリクエストでチェックしていない場合、`all`メソッドが返す結果に現れない可能性があります。

もし、 `all` メソッドを使うときに、特徴クラスが常に含まれるよう確実にしたい場合は、Pennantの機能発見機構を使ってください。最初に、アプリケーションのサービスプロバイダの一つの中で、`discover`メソッドを呼び出します。

    <?php

    namespace App\Providers;

    use Illuminate\Support\ServiceProvider;
    use Laravel\Pennant\Feature;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * 全アプリケーションサービスの初期起動処理
         */
        public function boot(): void
        {
            Feature::discover();

            // ...
        }
    }

`discover`メソッドは、アプリケーションの`app/Features`ディレクトリにあるすべての機能クラスを登録します。`all`メソッドは、現在のリクエストでチェック済みかに関係なく、これらのクラスを結果に含めます。

```php
Feature::all();

// [
//     'App\Features\NewApi' => true,
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
//     'site-redesign' => true,
// ]
```

<a name="eager-loading"></a>
## Eagerロード

Pennantは、一つのリクエストで解決したすべての機能のメモリ内キャッシュを保持しますが、それでも性能上の問題が発生する可能性があります。これを軽減するために、Pennantは機能の値をEagerロードする機能を提供しています。

これを理解するため、機能がアクティブであるかをループの中でチェックしていると考えてください。

```php
use Laravel\Pennant\Feature;

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

データベースドライバを使用していると仮定すると、このコードはループ内のすべてのユーザーに対してデータベースクエリを実行することになり、潜在的に数百のクエリを実行することになります。しかし、Pennantの`load`メソッドを使えば、ユーザーやスコープのコレクションの値をEagerロードでき、この潜在的なパフォーマンスのボトルネックを取り除けます。

```php
Feature::for($users)->load(['notifications-beta']);

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

機能の値がまだロードされていないときだけロードするには、`loadMissing`メソッドを使用します。

```php
Feature::for($users)->loadMissing([
    'new-api',
    'purchase-button',
    'notifications-beta',
]);
```

<a name="updating-values"></a>
## 値の更新

機能の値を初めて解決するとき、裏で動作しているドライバはその結果をストレージへ保存します。これは、リクエスト間で一貫したユーザー体験を保証するために、多くの場合必要です。しかし、時には、保存している機能の値を手作業で更新したい場合もあるでしょう。

このために、`activate`と`deactivate`メソッドを使用して、機能の"on"と"off"を切り替えます。

```php
use Laravel\Pennant\Feature;

// デフォルトのスコープの機能を有効にする
Feature::activate('new-api');

// 指定のスコープの機能を無効にする
Feature::for($user->team)->deactivate('billing-v2');
```

または、`activate`メソッドへ第２引数を指定し、手作業で機能へリッチな値を設定することも可能です。

```php
Feature::activate('purchase-button', 'seafoam-green');
```

Pennantへ、ある機能の保存値を消去するように指示するには、`forget`メソッドを使用します。その機能を再びチェックするとき、Pennantはその機能の定義により、値を解決します。

```php
Feature::forget('purchase-button');
```

<a name="bulk-updates"></a>
### バルク更新

保存している機能の値を一括で更新するには、`activateForEveryone`メソッドと`deactivateForEveryone`メソッドを使用します。

例えば、あなたが`new-api`機能の安定性に自信を持ち、チェックアウトフローに最適な`'purchase-button'`の色を見つけたとします。それに応じて、全ユーザーの機能値を更新できます。

```php
use Laravel\Pennant\Feature;

Feature::activateForEveryone('new-api');

Feature::activateForEveryone('purchase-button', 'seafoam-green');
```

もしくは、全ユーザーに対し、その機能を非アクティブにすることもできます。

```php
Feature::deactivateForEveryone('new-api');
```

> [!NOTE] This will only update the resolved feature values that have been stored by Pennant's storage driver. You will also need to update the feature definition in your application.

<a name="purging-features"></a>
### 機能の削除

時には、ストレージから機能全体を取り除くのが、有用な場合があります。これは、アプリケーションからその機能を削除した場合や、その機能の定義に調整を加え、全ユーザーにロールアウトする状況で、典型的に必要になります。

ある機能に対して保存されているすべての値を削除するには、`purge`メソッドを使用します。

```php
// １機能の削除
Feature::purge('new-api');

// 複数機能の削除
Feature::purge(['new-api', 'purchase-button']);
```

もしストレージから、*全*機能を削除したい場合は、引数なしで`purge`メソッドを呼び出します。

```php
Feature::purge();
```

アプリケーションのデプロイパイプラインの一貫として、機能を削除できると便利であるため、Pennantは指定した機能をストレージから削除する、`pennant:purge` Artisanコマンドを用意しています。

```sh
php artisan pennant:purge new-api

php artisan pennant:purge new-api purchase-button
```

また、リスト内で指定した機能を*除く*すべての機能を削除することも可能です。例えば、"new-api"と"purchase-button"機能を保存したまま、他のすべてのフィーチャーを削除したいとします。これを行うには、`--except`オプションへこれらの機能名を渡します。

```sh
php artisan pennant:purge --except=new-api --except=purchase-button
```

使いやすいように、`pennant:purge`コマンドは、`--except-registered`フラグもサポートしています。このフラグは、サービスプロバイダで明示的に登録している機能以外のすべての機能を削除することを意味します。

```sh
php artisan pennant:purge --except-registered
```

<a name="testing"></a>
## テスト

機能フラグを操作するコードをテストする場合、機能フラグの返り値をテストでコントロールする最も簡単な方法は、その機能を単純に再定義することです。たとえば、アプリケーションのサービスプロバイダに次のような機能が定義されているとしましょう。

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn () => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

テストの中で、機能の戻り値を変更するには、テストの最初にその機能を再定義します。以下のテストは、サービスプロバイダに`Arr::random()`の実装が残っていても、常にパスします。

```php tab=Pest
use Laravel\Pennant\Feature;

test('it can control feature values', function () {
    Feature::define('purchase-button', 'seafoam-green');

    expect(Feature::value('purchase-button'))->toBe('seafoam-green');
});
```

```php tab=PHPUnit
use Laravel\Pennant\Feature;

public function test_it_can_control_feature_values()
{
    Feature::define('purchase-button', 'seafoam-green');

    $this->assertSame('seafoam-green', Feature::value('purchase-button'));
}
```

クラスベースの機能でも、同様のアプローチが可能です。

```php tab=Pest
use Laravel\Pennant\Feature;

test('it can control feature values', function () {
    Feature::define(NewApi::class, true);

    expect(Feature::value(NewApi::class))->toBeTrue();
});
```

```php tab=PHPUnit
use App\Features\NewApi;
use Laravel\Pennant\Feature;

public function test_it_can_control_feature_values()
{
    Feature::define(NewApi::class, true);

    $this->assertTrue(Feature::value(NewApi::class));
}
```

もし機能が、`Lottery`インスタンスを返すのであれば、便利な[利用できるテストヘルパ](/docs/{{version}}/helpers#testing-lotteries)があります。

<a name="store-configuration"></a>
#### 保存域の設定

アプリケーションの`phpunit.xml`ファイルで、`PENNANT_STORE`環境変数を定義すれば、テスト中にPennantが使用する保存域を設定できます。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit colors="true">
    <!-- ... -->
    <php>
        <env name="PENNANT_STORE" value="array"/>
        <!-- ... -->
    </php>
</phpunit>
```

<a name="adding-custom-pennant-drivers"></a>
## カスタム機能ドライバの追加

<a name="implementing-the-driver"></a>
#### ドライバの実装

もし、Pennantの既存ストレージドライバが、どれもあなたのアプリケーションのニーズに合わない場合は、独自のストレージドライバを書いてください。カスタムドライバは、`Laravel\Pennant\Contracts\Driver`インターフェイスを実装する必要があります。

```php
<?php

namespace App\Extensions;

use Laravel\Pennant\Contracts\Driver;

class RedisFeatureDriver implements Driver
{
    public function define(string $feature, callable $resolver): void {}
    public function defined(): array {}
    public function getAll(array $features): array {}
    public function get(string $feature, mixed $scope): mixed {}
    public function set(string $feature, mixed $scope, mixed $value): void {}
    public function setForAllScopes(string $feature, mixed $value): void {}
    public function delete(string $feature, mixed $scope): void {}
    public function purge(array|null $features): void {}
}
```

あとは、Redis接続を使う、これらのメソッドを実装するだけです。それぞれのメソッドの実装例は、[Pennantのソースコード](https://github.com/laravel/pennant/blob/1.x/src/Drivers/DatabaseDriver.php)にある、`Laravel\Pennant\Drivers\DatabaseDriver`を見てください。

> [!NOTE]
> Laravelは、拡張機能を格納するディレクトリを用意していません。好きな場所に自由に配置できます。この例では、`RedisFeatureDriver`を格納するために、`Extensions`ディレクトリを作成しました。

<a name="registering-the-driver"></a>
#### ドライバの登録

ドライバを実装したら、Laravelに登録します。Pennantへドライバを追加するには、`Feature`ファサードが提供する`extend`メソッドをしようします。`extend`メソッドは、アプリケーションの[サービスプロバイダ](/docs/{{version}}/providers の`boot`メソッドから呼び出す必要があります。

```php
<?php

namespace App\Providers;

use App\Extensions\RedisFeatureDriver;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

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
     * 全アプリケーションサービスの初期起動処理
     */
    public function boot(): void
    {
        Feature::extend('redis', function (Application $app) {
            return new RedisFeatureDriver($app->make('redis'), $app->make('events'), []);
        });
    }
}
```

ドライバを登録したら、アプリケーションの`config/pennant.php`設定ファイルで、`redis`ドライバが利用できるようになります。

    'stores' => [

        'redis' => [
            'driver' => 'redis',
            'connection' => null,
        ],

        // ...

    ],

<a name="events"></a>
## イベント

Pennantは、アプリケーション全体の機能フラグを追跡するときに便利な、さまざまなイベントを発行します。

### `Laravel\Pennant\Events\FeatureRetrieved`

このイベントは、[機能をチェックする](#checking-features)たびディスパッチします。このイベントは、アプリケーション全体で機能フラグの使用状況に対するメトリクスを作成し、追跡するのに便利でしょう。

### `Laravel\Pennant\Events\FeatureResolved`

このイベントは、機能の値を特定のスコープで初めて解決したときにディスパッチします。

### `Laravel\Pennant\Events\UnknownFeatureResolved`

このイベントは、未知の機能を特定のスコープで初めて解決したときにディスパッチします。このイベントをリッスンしておくと、機能フラグを削除したつもりが、誤ってアプリケーション全体にその機能への参照を残してしまった場合に便利です。

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Log;
use Laravel\Pennant\Events\UnknownFeatureResolved;

class AppServiceProvider extends ServiceProvider
{
    /**
     * 全アプリケーションサービスの初期起動処理
     */
    public function boot(): void
    {
        Event::listen(function (UnknownFeatureResolved $event) {
            Log::error("Resolving unknown feature [{$event->feature}].");
        });
    }
}
```

### `Laravel\Pennant\Events\DynamicallyRegisteringFeatureClass`

このイベントは、リクエスト中で[クラスベースの機能](#class-based-features)を初めて動的にチェックしたときにディスパッチします。

### `Laravel\Pennant\Events\UnexpectedNullScopeEncountered`

このイベントは、[nullをサポートしない](#nullable-scope)機能定義へ、`null`スコープを渡したときにディスパッチします。

この状況をスムーズに処理し、その機能は`false`を返します。しかし、この機能のデフォルトのスムーズな動作を外したければ、アプリケーションの`AppServiceProvider`の`boot`メソッドでこのイベントのリスナを登録してください。

```php
use Illuminate\Support\Facades\Log;
use Laravel\Pennant\Events\UnexpectedNullScopeEncountered;

/**
 * アプリケーションの全サービスの初期起動処理
 */
public function boot(): void
{
    Event::listen(UnexpectedNullScopeEncountered::class, fn () => abort(500));
}

```

### `Laravel\Pennant\Events\FeatureUpdated`

このイベントは、通常`activate`または`deactivate`を呼び出すことにより、スコープの機能を更新したときにディスパッチします。

### `Laravel\Pennant\Events\FeatureUpdatedForAllScopes`

このイベントは、通常`activateForEveryone`または`deactivateForEveryone`を呼び出すことで、すべてのスコープの機能を更新したときにディスパッチします。

### `Laravel\Pennant\Events\FeatureDeleted`

このイベントは、スコープの機能を削除するときにディスパッチします。

### `Laravel\Pennant\Events\FeaturesPurged`

このイベントは、特定の機能をパージするときにディスパッチします。

### `Laravel\Pennant\Events\AllFeaturesPurged`

このイベントは、すべての機能をパージするときにディスパッチします。
