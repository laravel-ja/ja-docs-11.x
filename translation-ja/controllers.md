# コントローラ

- [イントロダクション](#introduction)
- [コントローラを書く](#writing-controllers)
    - [基本のコントローラ](#basic-controllers)
    - [シングルアクションコントローラ](#single-action-controllers)
- [コントローラミドルウェア](#controller-middleware)
- [リソースコントローラ](#resource-controllers)
    - [部分的なリソースルート](#restful-partial-resource-routes)
    - [ネストしたリソース](#restful-nested-resources)
    - [リソースルートの命名](#restful-naming-resource-routes)
    - [リソースルートパラメータの命名](#restful-naming-resource-route-parameters)
    - [リソースルートのスコープ](#restful-scoping-resource-routes)
    - [リソースURIのローカライズ](#restful-localizing-resource-uris)
    - [リソースコントローラへのルート追加](#restful-supplementing-resource-controllers)
    - [シングルトンリソースコントローラ](#singleton-resource-controllers)
- [依存注入とコントローラ](#dependency-injection-and-controllers)

<a name="introduction"></a>
## イントロダクション

すべてのリクエスト処理ロジックをルートファイルのクロージャとして定義する代わりに、「コントローラ」クラスを使用してこの動作を整理することを推奨します。コントローラにより、関係するリクエスト処理ロジックを単一のクラスにグループ化できます。たとえば、`UserController`クラスは、ユーザーの表示、作成、更新、削除など、ユーザーに関連するすべての受信リクエストを処理するでしょう。コントローラはデフォルトで、`app/Http/Controllers`ディレクトリに保存します。

<a name="writing-controllers"></a>
## コントローラを書く

<a name="basic-controllers"></a>
### 基本のコントローラ

新しいコントローラを簡単に生成するには、`make:controller` Artisanコマンドを実行します。アプリケーションのすべてのコントローラは、デフォルトで`app/Http/Controllers`ディレクトリへ設置します。

```shell
php artisan make:controller UserController
```

基本的なコントローラの例を見てみましょう。コントローラは、HTTPリクエストに応答するパブリックメソッドをいくつでも持てます。

    <?php

    namespace App\Http\Controllers;

    use App\Models\User;
    use Illuminate\View\View;

    class UserController extends Controller
    {
        /**
         * 指定ユーザーのプロファイルを表示
         */
        public function show(string $id): View
        {
            return view('user.profile', [
                'user' => User::findOrFail($id)
            ]);
        }
    }

コントローラクラスとメソッドが書けたら、コントローラメソッドへのルートを以下のように定義できます。

    use App\Http\Controllers\UserController;

    Route::get('/user/{id}', [UserController::class, 'show']);

受信リクエストが指定したルートURIに一致すると、`App\Http\Controllers\UserController`クラスの`show`メソッドが呼び出され、ルートパラメータがメソッドに渡されます。

> [!NOTE]
> コントローラはベースクラスを継承する**必要はありません**。しかし、コントローラ全体で共有すべきメソッドを含む基底クラスを継承しておくと便利な場合があります。

<a name="single-action-controllers"></a>
### シングルアクションコントローラ

コントローラのアクションがとくに複雑な場合は、コントローラクラス全体をその単一のアクション専用にするのが便利です。これを利用するには、コントローラ内で単一の`__invoke`メソッドを定義します。

    <?php

    namespace App\Http\Controllers;

    class ProvisionServer extends Controller
    {
        /**
         * 新しいWebサーバをプロビジョニング
         */
        public function __invoke()
        {
            // ...
        }
    }

シングルアクションコントローラのルートを登録する場合、コントローラ方式を指定する必要はありません。代わりに、コントローラの名前をルーターに渡すだけです。

    use App\Http\Controllers\ProvisionServer;

    Route::post('/server', ProvisionServer::class);

`make:controller` Artisanコマンドで`--invokable`オプションを指定すると、`__invoke`メソッドを含んだコントローラを生成できます。

```shell
php artisan make:controller ProvisionServer --invokable
```

> [!NOTE]
> [stubのリソース公開](/docs/{{version}}/artisan#stub-customization)を使用し、コントローラのスタブをカスタマイズできます。

<a name="controller-middleware"></a>
## コントローラミドルウェア

[ミドルウェア](/docs/{{version}}/middleware)はルートファイルの中で、コントローラのルートに対して指定します。

    Route::get('profile', [UserController::class, 'show'])->middleware('auth');

あるいは、コントローラクラス内でミドルウェアを指定する便利な方法もあります。それには、コントローラでstaticな`middleware`メソッドを持つ、`HasMiddleware`インターフェイスを実装する必要があります。このメソッドから、コントローラのアクションに適用するミドルウェアの配列を返します。

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Routing\Controllers\HasMiddleware;
    use Illuminate\Routing\Controllers\Middleware;

    class UserController extends Controller implements HasMiddleware
    {
        /**
         * コントローラへ指定するミドルウェアを取得
         */
        public static function middleware(): array
        {
            return [
                'auth',
                new Middleware('log', only: ['index']),
                new Middleware('subscribed', except: ['store']),
            ];
        }

        // ...
    }

また、コントローラのミドルウェアをクロージャとして定義することもできます。これは、ミドルウェアクラス全体を書かなくても、インラインミドルウェアを定義できる便利な方法です。

    use Closure;
    use Illuminate\Http\Request;

    /**
     * コントローラへ指定するミドルウェアを取得
     */
    public static function middleware(): array
    {
        return [
            function (Request $request, Closure $next) {
                return $next($request);
            },
        ];
    }

<a name="resource-controllers"></a>
## リソースコントローラ

アプリケーション内の各Eloquentモデルを「リソース」と考える場合、通常、アプリケーション内の各リソースに対して同じ一連のアクションを実行します。たとえば、アプリケーションに`Photo`モデルと`Movie`モデルが含まれているとします。ユーザーはこれらのリソースを作成、読み取り、更新、または削除できるでしょう。

このようなコモン・ケースのため、Laravelリソースルーティングは、通常の作成、読み取り、更新、および削除("CRUD")ルートを１行のコードでコントローラに割り当てます。使用するには、`make:controller` Artisanコマンドへ`--resource`オプションを指定すると、こうしたアクションを処理するコントローラをすばやく作成できます。

```shell
php artisan make:controller PhotoController --resource
```

このコマンドは、`app/Http/Controllers/PhotoController.php`にコントローラを生成します。コントローラには、そのまま使用可能な各リソース操作のメソッドを用意してあります。次に、コントローラを指すリソースルートを登録しましょう。

    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class);

この一つのルート宣言で、リソースに対するさまざまなアクションを処理するための複数のルートを定義しています。生成したコントローラには、これらのアクションごとにスタブしたメソッドがすでに含まれています。`route:list` Artisanコマンドを実行すると、いつでもアプリケーションのルートの概要をすばやく確認できます。

配列を`resources`メソッドに渡すことで、一度に多くのリソースコントローラを登録することもできます。

    Route::resources([
        'photos' => PhotoController::class,
        'posts' => PostController::class,
    ]);

<a name="actions-handled-by-resource-controller"></a>
#### リソースコントローラにより処理されるアクション

| 動詞      | URI                    | アクション | ルート名       |
| --------- | ---------------------- | ---------- | -------------- |
| GET       | `/photos`              | index      | photos.index   |
| GET       | `/photos/create`       | create     | photos.create  |
| POST      | `/photos`              | store      | photos.store   |
| GET       | `/photos/{photo}`      | show       | photos.show    |
| GET       | `/photos/{photo}/edit` | edit       | photos.edit    |
| PUT/PATCH | `/photos/{photo}`      | update     | photos.update  |
| DELETE    | `/photos/{photo}`      | destroy    | photos.destroy |

<a name="customizing-missing-model-behavior"></a>
#### 見つからないモデルの動作のカスタマイズ

暗黙的にバインドしたリソースモデルが見つからない場合、通常404のHTTPレスポンスが生成されます。ただし、リソースルートを定義するときに`missing`メソッドを呼び出すことでこの動作をカスタマイズすることができます。`missing`メソッドは、暗黙的にバインドされたモデルがリソースのルートに対して見つからない場合に呼び出すクロージャを引数に取ります。

    use App\Http\Controllers\PhotoController;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Redirect;

    Route::resource('photos', PhotoController::class)
            ->missing(function (Request $request) {
                return Redirect::route('photos.index');
            });

<a name="soft-deleted-models"></a>
#### ソフトデリートしたモデル

通常、暗黙のモデルバインディングでは、[ソフトデリート](/docs/{{version}}/eloquent#soft-deleting)したモデルを取得せず、代わりに404 HTTPレスポンスを返します。しかし、リソースルートを定義するときに、`withTrashed`メソッドを呼び出せば、ソフトデリート済みモデルの操作を許可するようにフレームワークへ指示できます。

    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class)->withTrashed();

引数なしで`withTrashed`を呼び出すと、`show`、`edit`、`update`リソースルートでのソフトデリート操作を許可します。`withTrashed`メソッドへ配列を渡し、これらのルートのサブセットを指定することもできます。

    Route::resource('photos', PhotoController::class)->withTrashed(['show']);

<a name="specifying-the-resource-model"></a>
#### リソースモデルの指定

[ルートモデル結合](/docs/{{version}}/routing#route-model-binding)を使用していて、リソースコントローラのメソッドでモデルインスタンスをタイプヒントしたい場合は、コントローラを生成するときのオプションに`--model`を使用します。

```shell
php artisan make:controller PhotoController --model=Photo --resource
```

<a name="generating-form-requests"></a>
#### フォームリクエストの生成

リソースコントローラの生成時に、`--requests`オプションを指定すると、コントローラの保存と更新メソッド用に[フォームリクエストクラス](/docs/{{version}}/validation#form-request-validation)を生成するようにArtisanへ指定できます。

```shell
php artisan make:controller PhotoController --model=Photo --resource --requests
```

<a name="restful-partial-resource-routes"></a>
### 部分的なリソースルート

リソースルートの宣言時に、デフォルトアクション全部を指定する代わりに、ルートで処理するアクションの一部を指定可能です。

    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class)->only([
        'index', 'show'
    ]);

    Route::resource('photos', PhotoController::class)->except([
        'create', 'store', 'update', 'destroy'
    ]);

<a name="api-resource-routes"></a>
#### APIリソースルート

APIに使用するリソースルートを宣言する場合、`create`や`edit`のようなHTMLテンプレートを提供するルートを除外したいことがよく起こります。そのため、これらの２ルートを自動的に除外する、`apiResource`メソッドが使用できます。

    use App\Http\Controllers\PhotoController;

    Route::apiResource('photos', PhotoController::class);

`apiResources`メソッドに配列として渡すことで、一度に複数のAPIリソースコントローラを登録できます。

    use App\Http\Controllers\PhotoController;
    use App\Http\Controllers\PostController;

    Route::apiResources([
        'photos' => PhotoController::class,
        'posts' => PostController::class,
    ]);

`create`や`edit`メソッドを含まないAPIリソースコントローラを素早く生成するには、`make:controller`コマンドを実行する際、`--api`スイッチを使用してください。

```shell
php artisan make:controller PhotoController --api
```

<a name="restful-nested-resources"></a>
### ネストしたリソース

ネストしたリソースへのルートを定義したい場合もあるでしょう。たとえば、写真リソースは、写真へ投稿された複数のコメントを持っているかもしれません。リソースコントローラをネストするには、ルート宣言で「ドット」表記を使用します。

    use App\Http\Controllers\PhotoCommentController;

    Route::resource('photos.comments', PhotoCommentController::class);

このルートにより次のようなURLでアクセスする、ネストしたリソースが定義できます。

    /photos/{photo}/comments/{comment}

<a name="scoping-nested-resources"></a>
#### ネストしたリソースのスコープ

Laravelの[暗黙的なモデル結合](/docs/{{version}}/routing#implicit-model-binding-scoping)機能は、リソース解決する子モデルが親モデルに属することを確認するように、ネストした結合を自動的にスコープできます。ネストしたリソースを定義するときに`scoped`メソッドを使用することにより、自動スコープを有効にしたり、子リソースを取得するフィールドをLaravelに指示したりできます。この実現方法の詳細は、[リソースルートのスコープ](#restful-scoping-resource-routes)に関するドキュメントを参照してください。

<a name="shallow-nesting"></a>
#### Shallowネスト

子のIDがすでに一意な識別子になってる場合、親子両方のIDをURIに含める必要はまったくありません。主キーの自動増分のように、一意の識別子をURIセグメント中でモデルを識別するために使用しているのなら、「shallow（浅い）ネスト」を使用できます。

    use App\Http\Controllers\CommentController;

    Route::resource('photos.comments', CommentController::class)->shallow();

このルート定義は、以下のルートを定義します。

| 動詞      | URI                               | アクション | ルート名               |
| --------- | --------------------------------- | ---------- | ---------------------- |
| GET       | `/photos/{photo}/comments`        | index      | photos.comments.index  |
| GET       | `/photos/{photo}/comments/create` | create     | photos.comments.create |
| POST      | `/photos/{photo}/comments`        | store      | photos.comments.store  |
| GET       | `/comments/{comment}`             | show       | comments.show          |
| GET       | `/comments/{comment}/edit`        | edit       | comments.edit          |
| PUT/PATCH | `/comments/{comment}`             | update     | comments.update        |
| DELETE    | `/comments/{comment}`             | destroy    | comments.destroy       |

<a name="restful-naming-resource-routes"></a>
### リソースルートの命名

すべてのリソースコントローラアクションにはデフォルトのルート名があります。ただし、`names`配列に指定したいルート名を渡すことで、この名前を上書きできます。

    use App\Http\Controllers\PhotoController;

    Route::resource('photos', PhotoController::class)->names([
        'create' => 'photos.build'
    ]);

<a name="restful-naming-resource-route-parameters"></a>
### リソースルートパラメータの命名

`Route::resource`はデフォルトで、リソース名の「単数形」バージョンに基づいて、リソースルートのルートパラメータを作成します。`parameters`メソッドを使用して、リソースごとにこれを簡単にオーバーライドできます。`parameters`メソッドに渡す配列は、リソース名とパラメーター名の連想配列である必要があります。

    use App\Http\Controllers\AdminUserController;

    Route::resource('users', AdminUserController::class)->parameters([
        'users' => 'admin_user'
    ]);

上記の例では、リソースの`show`ルートに対して以下のURIが生成されます。

    /users/{admin_user}

<a name="restful-scoping-resource-routes"></a>
### リソースルートのスコープ

Laravelの[スコープ付き暗黙モデル結合](/docs/{{version}}/routing#implicit-model-binding-scoping)機能は、解決する子モデルが親モデルに属することを確認するように、ネストした結合を自動的にスコープできます。ネストしたリソースを定義するときに`scoped`メソッドを使用することで、自動スコープを有効にし、以下のように子リソースを取得するフィールドをLaravelに指示できます。

    use App\Http\Controllers\PhotoCommentController;

    Route::resource('photos.comments', PhotoCommentController::class)->scoped([
        'comment' => 'slug',
    ]);

このルートは、以下のようなURIでアクセスする、スコープ付きのネストしたリソースを登録します。

    /photos/{photo}/comments/{comment:slug}

ネストしたルートパラメーターとしてカスタムキー付き暗黙的結合を使用する場合、親からネストしているモデルを取得するために、Laravelはクエリのスコープを自動的に設定し、親のリレーション名を推測する規則を使用します。この場合、`Photo`モデルには、`Comment`モデルを取得するために使用できる`comments`(ルートパラメータ名の複数形)という名前のリレーションがあると想定します。

<a name="restful-localizing-resource-uris"></a>
### リソースURIのローカライズ

`Route::resource`はデフォルトで、英語の動詞とその複数形の規則を使用してリソースURIを作成します。もし `create`と`edit`アクションの動詞をローカライズする必要がある場合は、`Route::resourceVerbs`メソッドを使用してください。これは、アプリケーションの`AppProviders`内の、`boot`メソッドの先頭で行ってください。

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Route::resourceVerbs([
            'create' => 'crear',
            'edit' => 'editar',
        ]);
    }

Laravelの複数形化機能は、[皆さんがニーズに基づいて設定した複数の異なる言語](/docs/{{version}}/localization#pluralization-language)をサポートしています。言語の動詞と複数形化をカスタマイズしたら、`Route::resource('publicacion', PublicacionController::class)`のようなリソースルート登録で、以下のURIが生成されるようになります。

    /publicacion/crear

    /publicacion/{publicaciones}/editar

<a name="restful-supplementing-resource-controllers"></a>
### リソースコントローラへのルート追加

リソースルートのデフォルトセットを超えてリソースコントローラにルートを追加する必要がある場合は、`Route::resource`メソッドを呼び出す前にそれらのルートを定義する必要があります。そうしないと、`resource`メソッドで定義されたルートが、意図せずに補足ルートよりも優先される可能性があります。

    use App\Http\Controller\PhotoController;

    Route::get('/photos/popular', [PhotoController::class, 'popular']);
    Route::resource('photos', PhotoController::class);

> [!NOTE]
> コントローラの責務を限定することを思い出してください。典型的なリソースアクションから外れたメソッドが繰り返して必要になっているようであれば、コントローラを２つに分け、小さなコントローラにすることを考えましょう。

<a name="singleton-resource-controllers"></a>
### シングルトンリソースコントローラ

アプリケーションが単一インスタンスのリソースだけ持つことが時々あります。例えば、ユーザーの「プロフィール」は編集や更新が可能ですが、ユーザーが複数の「プロフィール」を持つことはありません。同様に、画像は１つの「サムネイル」だけを持つことができます。こうしたリソースは、「シングルトンリソース」と呼ばれ、リソースのインスタンスは１つしか存在しないことを意味します。このようなシナリオでは、「シングルトン」リソースコントローラを登録してください。

```php
use App\Http\Controllers\ProfileController;
use Illuminate\Support\Facades\Route;

Route::singleton('profile', ProfileController::class);
```

上記のシングルトンリソース定義により、以下のルートを登録します。このように、シングルトンリソースでは「作成」ルートを登録しません。また、リソースのインスタンスが１つしか存在しないため、登録するルートは識別子を受け付けません。

| 動詞      | URI             | アクション | ルート名       |
| --------- | --------------- | ---------- | -------------- |
| GET       | `/profile`      | show       | profile.show   |
| GET       | `/profile/edit` | edit       | profile.edit   |
| PUT/PATCH | `/profile`      | update     | profile.update |

シングルトン・リソースは、標準リソースの中に入れ子にすることもできます。

```php
Route::singleton('photos.thumbnail', ThumbnailController::class);
```

この例では、`photos`リソースはすべての[標準リソースルート](#actions-handled-by-resource-controller)を受け取ります。しかし、`thumbnail`リソースは、以下のルートを持つシングルトンリソースとなります。

| 動詞      | URI                              | アクション | ルート名                |
| --------- | -------------------------------- | ---------- | ----------------------- |
| GET       | `/photos/{photo}/thumbnail`      | show       | photos.thumbnail.show   |
| GET       | `/photos/{photo}/thumbnail/edit` | edit       | photos.thumbnail.edit   |
| PUT/PATCH | `/photos/{photo}/thumbnail`      | update     | photos.thumbnail.update |

<a name="creatable-singleton-resources"></a>
#### シングルトンリソースの作成

シングルトンリソースを作成、保存するルートを定義したい場合も時にはあります。これを行うには、シングルトンリソースルートを登録する際、`creatable`メソッドを呼び出してください。

```php
Route::singleton('photos.thumbnail', ThumbnailController::class)->creatable();
```

この例では、以下のルートを登録します。ご覧の通り、作成可能なシングルトンリソースには、`DELETE`ルートも登録します。

| 動詞      | URI                                | アクション | ルート名                 |
| --------- | ---------------------------------- | ---------- | ------------------------ |
| GET       | `/photos/{photo}/thumbnail/create` | create     | photos.thumbnail.create  |
| POST      | `/photos/{photo}/thumbnail`        | store      | photos.thumbnail.store   |
| GET       | `/photos/{photo}/thumbnail`        | show       | photos.thumbnail.show    |
| GET       | `/photos/{photo}/thumbnail/edit`   | edit       | photos.thumbnail.edit    |
| PUT/PATCH | `/photos/{photo}/thumbnail`        | update     | photos.thumbnail.update  |
| DELETE    | `/photos/{photo}/thumbnail`        | destroy    | photos.thumbnail.destroy |

Laravelにシングルトンリソースの`DELETE`ルートを登録し、作成／保存ルートは登録したくない場合は、`destroyable`メソッドを利用します。

```php
Route::singleton(...)->destroyable();
```

<a name="api-singleton-resources"></a>
#### APIのシングルトンリソース

`apiSingleton`メソッドを使用すると、API経由で操作するシングルトンリソースを登録できます。

```php
Route::apiSingleton('profile', ProfileController::class);
```

もちろん、APIシングルトンリソースは、`creatable`にすることも可能で、この場合はそのリソースに対する、`store`と`destroy`ルートを登録します。

```php
Route::apiSingleton('photos.thumbnail', ProfileController::class)->creatable();
```

<a name="dependency-injection-and-controllers"></a>
## 依存注入とコントローラ

<a name="constructor-injection"></a>
#### コンストラクターインジェクション

全コントローラの依存を解決するために、Laravelの[サービスコンテナ](/docs/{{version}}/container)が使用されます。これにより、コントローラが必要な依存をコンストラクターにタイプヒントで指定できるのです。依存クラスは自動的に解決され、コントローラへインスタンスが注入されます。

    <?php

    namespace App\Http\Controllers;

    use App\Repositories\UserRepository;

    class UserController extends Controller
    {
        /**
         * 新しいコントローラインスタンスの生成
         */
        public function __construct(
            protected UserRepository $users,
        ) {}
    }

<a name="method-injection"></a>
#### メソッドインジェクション

コンストラクターによる注入に加え、コントローラのメソッドでもタイプヒントにより依存を指定することもできます。メソッドインジェクションの典型的なユースケースは、コントローラメソッドへ`Illuminate\Http\Request`インスタンスを注入する場合です。

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;

    class UserController extends Controller
    {
        /**
         * 新ユーザーの保存
         */
        public function store(Request $request): RedirectResponse
        {
            $name = $request->name;

            // Store the user...

            return redirect('/users');
        }
    }

コントローラメソッドへルートパラメーターによる入力値が渡される場合も、依存定義の後に続けてルート引数を指定します。たとえば以下のようにルートが定義されていれば：

    use App\Http\Controllers\UserController;

    Route::put('/user/{id}', [UserController::class, 'update']);

下記のように`Illuminate\Http\Request`をタイプヒントで指定しつつ、コントローラメソッドで定義している`id`パラメータにアクセスできます。

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;

    class UserController extends Controller
    {
        /**
         * 指定ユーザーの更新
         */
        public function update(Request $request, string $id): RedirectResponse
        {
            // Update the user...

            return redirect('/users');
        }
    }
