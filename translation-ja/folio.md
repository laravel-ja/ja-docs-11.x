# Laravel Folio

- [イントロダクション](#introduction)
- [インストール](#installation)
    - [ページパス／URI](#page-paths-uris)
    - [サブドメインのルート](#subdomain-routing)
- [ルートの生成](#creating-routes)
    - [ネストしたルート](#nested-routes)
    - [ルートインデックス](#index-routes)
- [ルートパラメータ](#route-parameters)
- [ルートモデル結合](#route-model-binding)
    - [モデルのソフトデリート](#soft-deleted-models)
- [レンダフック](#render-hooks)
- [名前付きルート](#named-routes)
- [ミドルウェア](#middleware)
- [ルートのキャッシュ](#route-caching)

<a name="introduction"></a>
## イントロダクション

[Laravel Folio](https://github.com/laravel/folio)（フォリオ）は、Laravelアプリケーションのルーティングを簡素化するために設計した、強力なページベースのルータです。Laravel Folioを使用すると、アプリケーションで`resources/views/pages`ディレクトリ内にBladeテンプレートを作成するのと同じくらい簡単にルートを生成できます。

例えば、`/greeting` URLでアクセスできるページを作成するには、アプリケーションの`resources/views/pages`ディレクトリに`greeting.blade.php`ファイルを作成します。

```php
<div>
    Hello World
</div>
```

<a name="installation"></a>
## インストール

Folioを利用開始するには、Composerパッケージマネージャを使い、プロジェクトにインストールします。

```bash
composer require laravel/folio
```

Folioインストール後、`folio:install` Artisanコマンドを実行すると、Folioのサービスプロバイダをアプリケーションへインストールします。このサービスプロバイダは、Folioがルート／ページを検索するディレクトリを登録します。

```bash
php artisan folio:install
```

<a name="page-paths-uris"></a>
### ページパス／URI

Folioはデフォルトで、アプリケーションの`resources/views/pages`ディレクトリのページを提供しますが、Folioサービスプロバイダの`boot`メソッドでこのディレクトリをカスタマイズできます。

同じLaravelアプリケーションの中で、複数のFolioパスを指定すると便利な場合があります。例えば、アプリケーションの"admin"エリア用にFolioページの別のディレクトリを用意し、それ以外のアプリケーションのページでは、別のディレクトリを使いたい場合などです。

そのために、`Folio::path`メソッドと`Folio::uri`メソッドを使います。`path`メソッドは HTTPリクエストをルーティングする際に、Folioがページをスキャンするディレクトリを登録し、`uri`メソッドはそのディレクトリのページの「ベースURI」を指定します。

```php
use Laravel\Folio\Folio;

Folio::path(resource_path('views/pages/guest'))->uri('/');

Folio::path(resource_path('views/pages/admin'))
    ->uri('/admin')
    ->middleware([
        '*' => [
            'auth',
            'verified',

            // ...
        ],
    ]);
```

<a name="subdomain-routing"></a>
### サブドメインのルート

また、受信リクエストのサブドメインに基づき、ページをルーティングすることもできます。例えば、`admin.example.com`からのリクエストを、他のFolioページとは異なるページディレクトリにルーティングしたいとします。この場合、`Folio::path`メソッドを呼び出した後に`domain`メソッドを呼び出します。

```php
use Laravel\Folio\Folio;

Folio::domain('admin.example.com')
    ->path(resource_path('views/pages/admin'));
```

`domain`メソッドでは、ドメインやサブドメインの一部をパラメータとして取り込むこともできます。これらのパラメータはページテンプレートへ注入します。

```php
use Laravel\Folio\Folio;

Folio::domain('{account}.example.com')
    ->path(resource_path('views/pages/admin'));
```

<a name="creating-routes"></a>
## ルートの生成

Folioにマウントしたディレクトリのいずれかに、Bladeテンプレートを配置することで、Folioルートを作成できます。Folioはデフォルトで、`resources/views/pages`ディレクトリをマウントしますが、Folio サービスプロバイダの`boot`メソッドでこれらのディレクトリをカスタマイズできます。

BladeテンプレートをFolioへマウントしたディレクトリに配置すると、ブラウザからすぐにアクセスできます。例えば、`pages/schedule.blade.php`に配置されたページには、ブラウザから`http://example.com/schedule`でアクセスできます。

すべてのFolioページ／ルートのリストを手軽に表示するには、`folio:list` Artisanコマンドを呼び出します。

```bash
php artisan folio:list
```

<a name="nested-routes"></a>
### ネストしたルート

Folioのディレクトリ内にディレクトリを１つ以上作成すれば、ネストしたルートを作成できます。例えば、`/user/profile`からアクセスできるページを作成するには、`pages/user`ディレクトリ内に`profile.blade.php`テンプレートを作成します。

```bash
php artisan folio:page user/profile

# pages/user/profile.blade.php → /user/profile
```

<a name="index-routes"></a>
### ルートインデックス

特定のページをディレクトリの「インデックス」にしたい場合があります。Folioディレクトリ内に、`index.blade.php`テンプレートを配置すると、そのディレクトリのルートへのリクエストは、すべてそのページにルーティングされます。

```bash
php artisan folio:page index
# pages/index.blade.php → /

php artisan folio:page users/index
# pages/users/index.blade.php → /users
```

<a name="route-parameters"></a>
## ルートパラメータ

多くの場合、リクエストのURLセグメントをページに注入し、それらをやりとりできるようにする必要があります。例えば、プロフィールを表示しているユーザーの"ID"にアクセスする必要があるかもしれません。これを実現するには、ページのファイル名のセグメントを角括弧で囲みます。

```bash
php artisan folio:page "users/[id]"

# pages/users/[id].blade.php → /users/1
```

キャプチャしたセグメントには、Bladeテンプレート内の変数としてアクセスできます

```html
<div>
    User {{ $id }}
</div>
```

複数のセグメントをキャプチャするには、カプセル化するセグメントの前に３つのドット（`...`）を付けます。

```bash
php artisan folio:page "users/[...ids]"

# pages/users/[...ids].blade.php → /users/1/2/3
```

複数のセグメントをキャプチャした場合、キャプチャしたセグメントは配列としてページに注入します。

```html
<ul>
    @foreach ($ids as $id)
        <li>User {{ $id }}</li>
    @endforeach
</ul>
```

<a name="route-model-binding"></a>
## ルートモデル結合

ページテンプレートのファイル名のワイルドカードセグメントが、アプリケーションのEloquentモデルに対応する場合、Folioは自動的にLaravelのルートモデルバインディング機能を利用し、依存解決したモデルインスタンスをページに注入しようとします。

```bash
php artisan folio:page "users/[User]"

# pages/users/[User].blade.php → /users/1
```

キャプチャしたモデルは、Bladeテンプレート内の変数としてアクセスすることができます。モデルの変数名は「キャメルケース」へ変換されます。

```html
<div>
    User {{ $user->id }}
</div>
```

#### キーのカスタマイズ

結合済みのEloquentモデルを`id`以外のカラムを使い、依存解決したい場合があります。それには、ページのファイル名でそのカラムを指定します。例えば、`[Post:slug].blade.php`というファイル名のページでは、`id`カラムの代わりに`slug`カラムを使い結合済みモデルを依存解決しようとします。

Windowsの場合は、`-`でモデル名とキーを区切ります。例：`[Post-slug].blade.php`

#### モデルの場所

Folioはデフォルトで、アプリケーションの`app/Models`ディレクトリ内でモデルを検索します。しかし必要であれば、テンプレートのファイル名に完全修飾したモデルクラス名を指定できます。

```bash
php artisan folio:page "users/[.App.Models.User]"

# pages/users/[.App.Models.User].blade.php → /users/1
```

<a name="soft-deleted-models"></a>
### モデルのソフトデリート

暗黙的なモデル結合を依存解決する際にデフォルトでは、ソフトデリート済みのモデルを取得しません。しかし、必要であれば、ページのテンプレート内で、`withTrashed`関数を呼び出し、ソフトデリート済みモデルを取得するようにFolioへ指示できます。

```php
<?php

use function Laravel\Folio\{withTrashed};

withTrashed();

?>

<div>
    User {{ $user->id }}
</div>
```

<a name="render-hooks"></a>
## レンダフック

Folioは受信リクエストに対するレスポンスとして、ページのBladeテンプレートのコンテンツをデフォルトで返します。しかし、ページのテンプレート内で`render`関数を呼び出せば、レスポンスをカスタマイズできます。

`render`関数は、Folioがレンダする`View`インスタンスを受け取るクロージャを引数に取ります。`View`インスタンスの受け取りに加え、追加のルートパラメータやモデルバインディングも、`render`クロージャへ渡します。

```php
<?php

use App\Models\Post;
use Illuminate\Support\Facades\Auth;
use Illuminate\View\View;

use function Laravel\Folio\render;

render(function (View $view, Post $post) {
    if (! Auth::user()->can('view', $post)) {
        return response('Unauthorized', 403);
    }

    return $view->with('photos', $post->author->photos);
}); ?>

<div>
    {{ $post->content }}
</div>

<div>
    This author has also taken {{ count($photos) }} photos.
</div>
```

<a name="named-routes"></a>
## 名前付きルート

`name`関数を使って、指定したページのルートの名前を指定することができます。

```php
<?php

use function Laravel\Folio\name;

name('users.index');
```

Laravelの名前付きルートと同様、`route`関数を使用して、名前を割り当てたFolioページへのURLを生成できます。

```php
<a href="{{ route('users.index') }}">
    All Users
</a>
```

ページにパラメータがある場合は、その値を`route`関数に渡すだけです。

```php
route('users.show', ['user' => $user]);
```

<a name="middleware"></a>
## ミドルウェア

ページのテンプレート内で、`middleware`関数を呼び出し、特定のページへミドルウェアを適用できます。

```php
<?php

use function Laravel\Folio\{middleware};

middleware(['auth', 'verified']);

?>

<div>
    Dashboard
</div>
```

または、ミドルウェアをページグループに割り当てるには、`Folio::path`メソッドを呼び出した後に、`middleware`メソッドをチェーンします。

ミドルウェアをどのページに適用するべきかを指定するために、ミドルウェアの配列を適用すべきページに対応するURLパターンを使ってキーにできます。ワイルドカード文字として、`*`を使用します。

```php
use Laravel\Folio\Folio;

Folio::path(resource_path('views/pages'))->middleware([
    'admin/*' => [
        'auth',
        'verified',

        // ...
    ],
]);
```

インラインの無名ミドルウェアを定義するためには、ミドルウェアの配列へクロージャを含めてください。

```php
use Closure;
use Illuminate\Http\Request;
use Laravel\Folio\Folio;

Folio::path(resource_path('views/pages'))->middleware([
    'admin/*' => [
        'auth',
        'verified',

        function (Request $request, Closure $next) {
            // ...

            return $next($request);
        },
    ],
]);
```

<a name="route-caching"></a>
## ルートのキャッシュ

Folioを使用する際は、常に[Laravelのルートキャッシュ機能](/docs/{{version}}/routing#route-caching)を利用する必要があります。Folioは`route:cache` Artisanコマンドをリッスンし、Folioのページ定義とルート名が適切にキャッシュされ、パフォーマンスが最大になるようにします。
