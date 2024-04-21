# バリデーション

- [イントロダクション](#introduction)
- [クイックスタート](#validation-quickstart)
    - [ルート定義](#quick-defining-the-routes)
    - [コントローラ作成](#quick-creating-the-controller)
    - [バリデーションロジック作成](#quick-writing-the-validation-logic)
    - [バリデーションエラー表示](#quick-displaying-the-validation-errors)
    - [フォームの再取得](#repopulating-forms)
    - [オプションフィールドに対する注意](#a-note-on-optional-fields)
    - [バリデーションエラーのレスポンス形式](#validation-error-response-format)
- [フォームリクエストバリデーション](#form-request-validation)
    - [フォームリクエスト作成](#creating-form-requests)
    - [フォームリクエストの認可](#authorizing-form-requests)
    - [エラーメッセージのカスタマイズ](#customizing-the-error-messages)
    - [バリデーションのための入力準備](#preparing-input-for-validation)
- [バリデータの生成](#manually-creating-validators)
    - [自動リダイレクト](#automatic-redirection)
    - [名前付きエラーバッグ](#named-error-bags)
    - [エラーメッセージのカスタマイズ](#manual-customizing-the-error-messages)
    - [追加バリデーションの実行](#performing-additional-validation)
- [バリデーション済み入力の利用](#working-with-validated-input)
- [エラーメッセージ操作](#working-with-error-messages)
    - [言語ファイルでのカスタムメッセージの指定](#specifying-custom-messages-in-language-files)
    - [言語ファイルでの属性の指定](#specifying-attribute-in-language-files)
    - [言語ファイルでの値の指定](#specifying-values-replacements-in-language-files)
- [使用可能なバリデーションルール](#available-validation-rules)
- [条件付きの追加ルール](#conditionally-adding-rules)
- [配列のバリデーション](#validating-arrays)
    - [ネストした配列入力のバリデーション](#validating-nested-array-input)
    - [エラーメッセージインデックスとポジション](#error-message-indexes-and-positions)
- [ファイルのバリデーション](#validating-files)
- [パスワードのバリデーション](#validating-passwords)
- [カスタムバリデーションルール](#custom-validation-rules)
    - [ルールオブジェクトの使用](#using-rule-objects)
    - [クロージャの使用](#using-closures)
    - [暗黙のルール](#implicit-rules)

<a name="introduction"></a>
## イントロダクション

Laravelは、アプリケーションの受信データをバリデーションするために複数の異なるアプローチを提供します。すべての受信HTTPリクエストで使用可能な`validate`メソッドを使用するのがもっとも一般的です。しかし、バリデーションに対する他のアプローチについても説明していきます。

Laravelは、データに適用する便利で数多くのバリデーションルールを持っており、特定のデータベーステーブルで値が一意であるかどうかをバリデーションする機能も提供しています。Laravelのすべてのバリデーション機能に精通できるよう、各バリデーションルールを詳しく説明します。

<a name="validation-quickstart"></a>
## クイックスタート

Laravelの強力なバリデーション機能について学ぶため、フォームをバリデーションし、エラーメッセージをユーザーに表示する完全な例を見てみましょう。この高レベルの概要を読むことで、Laravelを使用して受信リクエストデータをバリデーションする一般的な方法を理解できます。

<a name="quick-defining-the-routes"></a>
### ルート定義

まず、`routes/web.php`ファイルに以下のルートを定義してあるとしましょう。

    use App\Http\Controllers\PostController;

    Route::get('/post/create', [PostController::class, 'create']);
    Route::post('/post', [PostController::class, 'store']);

`GET`のルートは新しいブログポストを作成するフォームをユーザーへ表示し、`POST`ルートで新しいブログポストをデータベースへ保存します。

<a name="quick-creating-the-controller"></a>
### コントローラ作成

次に、これらのルートへの受信リクエストを処理する単純なコントローラを見てみましょう。今のところ、`store`メソッドは空のままにしておきます。

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;
    use Illuminate\View\View;

    class PostController extends Controller
    {
        /**
         * 新ブログポスト作成フォームの表示
         */
        public function create(): View
        {
            return view('post.create');
        }

        /**
         * 新しいブログポストの保存
         */
        public function store(Request $request): RedirectResponse
        {
            // ブログポストのバリデーションと保存コード…

            $post = /** ... */

            return to_route('post.show', ['post' => $post->id]);
        }
    }

<a name="quick-writing-the-validation-logic"></a>
### バリデーションロジック

これで、新しいブログ投稿をバリデーションするロジックを`store`メソッドに入力する準備が整いました。これを行うには、`Illuminate\Http\Request`オブジェクトが提供する`validate`メソッドを使用します。バリデーションルールにパスすると、コードは正常に実行され続けます。しかし、バリデーションに失敗すると`Illuminate\Validation\ValidationException`例外を投げ、適切なエラーレスポンスを自動的にユーザーへ返送します。

伝統的なHTTPリクエスト処理中にバリデーションが失敗した場合、直前のURLへのリダイレクトレスポンスを生成します。受信リクエストがXHRリクエストの場合、[バリデーションエラーメッセージを含むJSONレスポンス](#validation-error-response-format)を返します。

`validate`メソッドをもっとよく理解するため、`store`メソッドに取り掛かりましょう。

    /**
     * 新ブログポストの保存
     */
    public function store(Request $request): RedirectResponse
    {
        $validated = $request->validate([
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
        ]);

        // ブログポストは有効

        return redirect('/posts');
    }

ご覧のとおり、バリデーションルールは`validate`メソッドへ渡します。心配いりません。利用可能なすべてのバリデーションルールは[文書化](#available-validation-rules)されています。この場合でもバリデーションが失敗したとき、適切な応答を自動的に生成します。バリデーションにパスすると、コントローラは正常に実行を継続します。

もしくは、バリデーションルールを`|`で区切る文字列の代わりに、ルールの配列で指定することもできます。

    $validatedData = $request->validate([
        'title' => ['required', 'unique:posts', 'max:255'],
        'body' => ['required'],
    ]);

さらに、`validateWithBag`メソッドを使用して、リクエストをバリデーションした結果のエラーメッセージを[名前付きエラーバッグ](#named-error-bags)内へ保存できます。

    $validatedData = $request->validateWithBag('post', [
        'title' => ['required', 'unique:posts', 'max:255'],
        'body' => ['required'],
    ]);

<a name="stopping-on-first-validation-failure"></a>
#### 最初のバリデーション失敗時に停止

最初のバリデーションに失敗したら、残りのバリデーションルールの判定を停止したい場合も、ときどき起きるでしょう。これには、`bail`ルールを使ってください。

    $request->validate([
        'title' => 'bail|required|unique:posts|max:255',
        'body' => 'required',
    ]);

この例の場合、`title`属性の`unique`ルールに失敗すると、`max`ルールをチェックしません。ルールは指定した順番にバリデートします。

<a name="a-note-on-nested-attributes"></a>
#### ネストした属性の注意点

受信HTTPリクエストに「ネストされた」フィールドデータが含まれている場合は、「ドット」構文を使用してバリデーションルールでこうしたフィールドを指定できます。

    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'author.name' => 'required',
        'author.description' => 'required',
    ]);

フィールド名にピリオドそのものが含まれている場合は、そのピリオドをバックスラッシュでエスケープし、「ドット」構文として解釈されることを明示的に防げます。

    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'v1\.0' => 'required',
    ]);

<a name="quick-displaying-the-validation-errors"></a>
### バリデーションエラー表示

では、受信リクエストフィールドが指定したバリデーションルールにパスしない場合はどうなるでしょうか。前述のように、Laravelはユーザーを直前の場所へ自動的にリダイレクトします。さらに、すべてのバリデーションエラーと[リクエスト入力](/docs/{{version}}/requests#retrieveing-old-input)は自動的に[セッションに一時保持](/docs/{{version}}/session#flash-data)保存されます。

`$errors`変数は、`web`ミドルウェアグループが提供する`Illuminate\View\Middleware\ShareErrorsFromSession`ミドルウェアにより、アプリケーションのすべてのビューで共有されます。このミドルウェアが適用されると、ビューで`$errors`変数は常に定義され、`$errors`変数が常に使用可能になり、安全・便利に使用できると想定できます。`$errors`変数は`Illuminate\Support\MessageBag`のインスタンスです。このオブジェクトの操作の詳細は、[ドキュメントを確認してください](#working-with-error-messages)。

この例では、バリデーションに失敗すると、エラーメッセージをビューで表示できるように、コントローラの`create`メソッドへリダイレクトされることになります。

```blade
<!-- /resources/views/post/create.blade.php -->

<h1>Create Post</h1>

@if ($errors->any())
    <div class="alert alert-danger">
        <ul>
            @foreach ($errors->all() as $error)
                <li>{{ $error }}</li>
            @endforeach
        </ul>
    </div>
@endif

<!-- Postフォームの作成 -->
```

<a name="quick-customizing-the-error-messages"></a>
#### エラーメッセージのカスタマイズ

Laravelの組み込み検証ルールには、それぞれエラーメッセージがあり、アプリケーションの`lang/en/validation.php`ファイル内に用意しています。アプリケーションに`lang`ディレクトリがない場合は、`lang:publish` Artisanコマンドにより、Laravelへ作成を指示してください。

`lang/en/validation.php`ファイル内に、各バリデーションルールの翻訳エントリーがあります。これらのメッセージは、アプリケーションのニーズに応じて自由に変更・修正してください。

さらに、このファイルを他の言語ディレクトリにコピーして、アプリケーションように言語用メッセージを翻訳することもできます。Laravelの多言語化の詳細については、完全な[多言語化のドキュメント](/docs/{{version}}/localization)をチェックしてみてください。

> [!WARNING]
> Laravelアプリケーションのスケルトンは、デフォルトで`lang`ディレクトリを用意していません。Laravelの言語ファイルをカスタマイズしたい場合は、`lang:publish` Artisanコマンドでリソース公開してください。

<a name="quick-xhr-requests-and-validation"></a>
#### XHRリクエストとバリデーション

この例では、従来のフォームを使用してデータをアプリケーションに送信しました。ただし、多くのアプリケーションは、JavaScriptを利用したフロントエンドから、XHRリクエストを受信します。XHRリクエスト中に`validate`メソッドを使用すると、Laravelはリダイレクト応答を生成しません。代わりに、Laravelは[すべてのバリデーションエラーを含むJSONレスポンス](#validation-error-response-format)を生成します。このJSONレスポンスは、422 HTTPステータスコードとともに送信されます。

<a name="the-at-error-directive"></a>
#### `@error`ディレクティブ

`@error`[Blade](/docs/{{version}}/brade)ディレクティブを使用して、特定の属性にバリデーションエラーメッセージが存在するかを簡単に判断できます。`@error`ディレクティブ内で、`$message`変数をエコーし​​てエラーメッセージを表示ができます。

```blade
<!-- /resources/views/post/create.blade.php -->

<label for="title">Post Title</label>

<input id="title"
    type="text"
    name="title"
    class="@error('title') is-invalid @enderror">

@error('title')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror
```

[名前付きエラーバッグ](#named-error-bags)を使用している場合は、エラーバッグの名前を `@error` ディレクティブの第2引数へ渡せます。

```blade
<input ... class="@error('title', 'post') is-invalid @enderror">
```

<a name="repopulating-forms"></a>
### フォームの再取得

Laravelがバリデーションエラーのためにリダイレクトレスポンスを生成するとき、フレームワークは自動的に[セッションへのリクエストのすべての入力を一時保持保存します](/docs/{{version}}/session#flash-data)。これにより、直後のリクエスト時、保存しておいた入力に簡単にアクセスし、ユーザーが送信しようとしたフォームを再入力・再表示できるようにします。

直前のリクエストから一時保持保存された入力を取得するには、`Illuminate\Http\Request`のインスタンスで`old`メソッドを呼び出します。`old`メソッドは、直前に一時保持保存した入力データを[セッション](/docs/{{version}}/session)から取り出します。

    $title = $request->old('title');

Laravelはグローバルな`old`ヘルパも提供しています。[Bladeテンプレート](/docs/{{version}}/Blade)内に古い入力を表示している場合は、`old`ヘルパを使用してフォームを再入力・再表示する方が便利です。指定されたフィールドに古い入力が存在しない場合、`null`が返されます。

```blade
<input type="text" name="title" value="{{ old('title') }}">
```

<a name="a-note-on-optional-fields"></a>
### オプションフィールドに対する注意

Laravelはデフォルトで、`TrimStrings`と`ConvertEmptyStringsToNull`ミドルウェアをアプリケーションのグローバルミドルウェアスタックに含めています。このため、バリデータに`null`値を無効と判定させたくない場合は、「オプション」のリクエストフィールドを`nullable`としてマークする必要があります。一例を御覧ください。

    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

上記の例の場合、`publish_at`フィールドが`null`か、有効な日付表現であることを指定しています。ルール定義に`nullable`が追加されないと、バリデータは`null`を無効な日付として判定します。

<a name="validation-error-response-format"></a>
### バリデーションエラーのレスポンス形式

受信HTTPリクエストがJSONレスポンスを期待しており、アプリケーションが`Illuminate\Validation\ValidationException`例外を投げる場合、Laravelは自動的にエラーメッセージをフォーマットして、`422 Unprocessable Entity` HTTPレスポンスを返します。

以下に、バリデーションエラーのJSONレスポンスフォーマット例を示します。ネストしたエラーのキーは、「ドット」記法で１次元化されることに注意してください。

```json
{
    "message": "The team name must be a string. (and 4 more errors)",
    "errors": {
        "team_name": [
            "The team name must be a string.",
            "The team name must be at least 1 characters."
        ],
        "authorization.role": [
            "The selected authorization.role is invalid."
        ],
        "users.0.email": [
            "The users.0.email field is required."
        ],
        "users.2.email": [
            "The users.2.email must be a valid email address."
        ]
    }
}
```

<a name="form-request-validation"></a>
## フォームリクエストバリデーション

<a name="creating-form-requests"></a>
### フォームリクエスト作成

より複雑なバリデーションシナリオの場合は、「フォームリクエスト」を作成することをお勧めします。フォームリクエストは、独自のバリデーションおよび認可ロジックをカプセル化するカスタムリクエストクラスです。フォームリクエストクラスを作成するには、`make:request` Artisan CLIコマンドを使用します。

```shell
php artisan make:request StorePostRequest
```

生成するフォームリクエストクラスは、`app/Http/Requests`ディレクトリに配置します。このディレクトリが存在しない場合は、`make:request`コマンドの実行時に作成します。Laravelにが生成する各フォームリクエストには、`authorize`と`rules`の２つのメソッドがあります。

ご想像のとおり、`authorize`メソッドは、現在認証されているユーザーがリクエストによって表されるアクションを実行できるかどうかを判断し、`rules`メソッドはリクエスト中のデータを検証するバリデーションルールを返します。

    /**
     * リクエストに適用するバリデーションルールを取得
     *
     * @return array<string, \Illuminate\Contracts\Validation\Rule|array|string>
     */
    public function rules(): array
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
        ];
    }

> [!NOTE]
> `rules`メソッドの引数で必要な依存関係をタイプヒントにより指定できます。それらはLaravel[サービスコンテナ](/docs/{{version}}/container)を介して自動的に依存解決されます。

では、どのようにバリデーションルールを実行するのでしょうか？必要なのはこのリクエストをコントローラのメソッドで、タイプヒントを使い指定することです。受信フォームリクエストは、コントローラメソッドを呼び出す前にバリデーションを行います。つまり、コントローラにバリデーションロジックを取っ散らかす必要はありません。

    /**
     * 新ブログポストの保存
     */
    public function store(StorePostRequest $request): RedirectResponse
    {
        // 送信されたリクエストは正しい

        // バリデーション済みデータの取得
        $validated = $request->validated();

        // バリデーション済み入力データの一部を取得
        $validated = $request->safe()->only(['name', 'email']);
        $validated = $request->safe()->except(['name', 'email']);

        // ブログ投稿の保存処理…

        return redirect('/posts');
    }

バリデーションが失敗した場合、リダイレクトレスポンスを生成し、ユーザーを直前のページへ送り返します。エラーもセッ​​ションへ一時保存し、表示できるようにします。リクエストがXHRリクエストの場合、422ステータスコードで、[バリデーションエラーのJSON表現を含むHTTPレスポンス](#validation-error-response-format)がユーザーに返されます。

> [!NOTE]
> Inertiaを搭載したLaravelのフロントエンドへ、リアルタイムのフォームリクエストバリデーションを追加する必要がありますか？[Laravel Precognition](/docs/{{version}}/precognition)をチェックしてください。

<a name="performing-additional-validation-on-form-requests"></a>
#### 追加バリデーションの実行

最初のバリデーションが完了した後に、追加のバリデーションを実行する必要がある場合があります。この場合、フォームリクエストの`after`メソッドを使用します。

`after`メソッドは、バリデーションが完了した後に呼び出すCallableやクロージャの配列を返す必要があります。指定Callablesは、`Illuminate\Validation\Validator`インスタンスを受け取るので、必要に応じ追加のエラーメッセージを表示できます。

    use Illuminate\Validation\Validator;

    /**
     * リクエストに対する「追加」バリデーションCallableの取得
     */
    public function after(): array
    {
        return [
            function (Validator $validator) {
                if ($this->somethingElseIsInvalid()) {
                    $validator->errors()->add(
                        'field',
                        'Something is wrong with this field!'
                    );
                }
            }
        ];
    }

前述のとおり、`after`メソッドが返す配列には、呼び出し可能なクラスも含まれます。そうしたクラスの`__invoke`メソッドは、`Illuminate\Validation\Validator`インスタンスを受け取ります。

```php
use App\Validation\ValidateShippingTime;
use App\Validation\ValidateUserStatus;
use Illuminate\Validation\Validator;

/**
 * リクエストに対する「追加」バリデーションCallableの取得
 */
public function after(): array
{
    return [
        new ValidateUserStatus,
        new ValidateShippingTime,
        function (Validator $validator) {
            //
        }
    ];
}
```

<a name="request-stopping-on-first-validation-rule-failure"></a>
#### バリデーションの最初の失敗で停止

リクエストクラスに`stopOnFirstFailure`プロパティを追加することで、バリデーションの失敗が起きてすぐに、すべての属性のバリデーションを停止する必要があることをバリデータへ指示できます。

    /**
     * バリデータが最初のルールの失敗で停​​止するかを指示
     *
     * @var bool
     */
    protected $stopOnFirstFailure = true;

<a name="customizing-the-redirect-location"></a>
#### リダイレクト先のカスタマイズ

前述のとおり、フォームリクエストのバリデーションに失敗した場合、ユーザーを元の場所に戻すためにリダイレクトレスポンスが生成されます。しかし、この動作は自由にカスタマイズ可能です。それには、フォームのリクエストで、`$redirect`プロパティを定義します。

    /**
     * バリデーション失敗時に、ユーザーをリダイレクトするURI
     *
     * @var string
     */
    protected $redirect = '/dashboard';

または、ユーザーを名前付きルートへリダイレクトする場合は、`$redirectRoute`プロパティを代わりに定義します。

    /**
     * バリデーション失敗時に、ユーザーをリダイレクトするルート
     *
     * @var string
     */
    protected $redirectRoute = 'dashboard';

<a name="authorizing-form-requests"></a>
### フォームリクエストの認可

フォームリクエストクラスには、`authorize`メソッドも含まれています。このメソッド内で、認証済みユーザーが特定のリソースを更新する権限を持っているかどうかを判別できます。たとえば、ユーザーが更新しようとしているブログコメントを実際に所有しているかどうかを判断できます。ほとんどの場合、次の方法で[認可ゲートとポリシー](/docs/{{version}}/authentication)を操作します。

    use App\Models\Comment;

    /**
     * ユーザーがこのリクエストの権限を持っているかを判断する
     */
    public function authorize(): bool
    {
        $comment = Comment::find($this->route('comment'));

        return $comment && $this->user()->can('update', $comment);
    }

全フォームリクエストはLaravelのベースリクエストクラスを拡張していますので、現在認証済みユーザーへアクセスする、`user`メソッドが使えます。また、上記例中の`route`メソッドの呼び出しにも、注目してください。たとえば`{comment}`パラメーターのような、呼び出しているルートで定義してあるURIパラメータにもアクセスできます。

    Route::post('/comment/{comment}');

したがって、アプリケーションが[ルートモデル結合](/docs/{{version}}/routing#route-model-binding)を利用している場合、解決済みモデルをリクエストのプロパティとしてアクセスすることで、コードをさらに簡潔にすることができます。

    return $this->user()->can('update', $this->comment);

`authorize`メソッドが`false`を返すと、403ステータスコードのHTTPレスポンスを自動的に返し、コントローラメソッドは実行しません。

アプリケーションの別の部分でリクエストの認可ロジックを処理する場合は、`authorize`メソッドを完全に削除し、`true`を返してください。

    /**
     * ユーザーがこのリクエストの権限を持っているかを判断する
     */
    public function authorize(): bool
    {
        return true;
    }

> [!NOTE]
> `authorize`メソッドの引数で、必要な依存をタイプヒントにより指定できます。それらはLaravelの[サービスコンテナ](/docs/{{version}}/container)により、自動的に依存解決されます。

<a name="customizing-the-error-messages"></a>
### エラーメッセージのカスタマイズ

フォームリクエストにより使用されているメッセージは`message`メソッドをオーバーライドすることによりカスタマイズできます。このメソッドから属性／ルールと対応するエラーメッセージのペアを配列で返してください。

    /**
     * 定義済みバリデーションルールのエラーメッセージ取得
     *
     * @return array<string, string>
     */
    public function messages(): array
    {
        return [
            'title.required' => 'A title is required',
            'body.required' => 'A message is required',
        ];
    }

<a name="customizing-the-validation-attributes"></a>
#### バリデーション属性のカスタマイズ

Laravelの組み込みバリデーションルールエラーメッセージの多くは、`:attribute`プレースホルダーを含んでいます。バリデーションメッセージの`:attribute`プレースホルダーをカスタム属性名に置き換えたい場合は、`attributes`メソッドをオーバーライドしてカスタム名を指定します。このメソッドは、属性と名前のペアの配列を返す必要があります。

    /**
     * バリデーションエラーのカスタム属性の取得
     *
     * @return array<string, string>
     */
    public function attributes(): array
    {
        return [
            'email' => 'email address',
        ];
    }

<a name="preparing-input-for-validation"></a>
### バリデーションのための入力準備

バリデーションルールを適用する前にリクエストからのデータを準備またはサニタイズする必要がある場合は、`prepareForValidation`メソッド使用します。

    use Illuminate\Support\Str;

    /**
     * バリーデーションのためにデータを準備
     */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'slug' => Str::slug($this->slug),
        ]);
    }

同様に、バリデーションが完了した後にリクエストデータをノーマライズする必要がある場合は、 `passedValidation` メソッドを使用します。

    /**
     * Handle a passed validation attempt.
     */
    protected function passedValidation(): void
    {
        $this->replace(['name' => 'Taylor']);
    }

<a name="manually-creating-validators"></a>
## バリデータの生成

リクエストの`validate`メソッドを使用したくない場合は、`Validator`[ファサード](/docs/{{version}}/facades)を使用し、バリデータインスタンスを生成する必要があります。 このファサードの`make`メソッドで、新しいバリデータインスタンスを生成します。

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Validator;

    class PostController extends Controller
    {
        /**
         * 新しいブログポストの保存
         */
        public function store(Request $request): RedirectResponse
        {
            $validator = Validator::make($request->all(), [
                'title' => 'required|unique:posts|max:255',
                'body' => 'required',
            ]);

            if ($validator->fails()) {
                return redirect('post/create')
                            ->withErrors($validator)
                            ->withInput();
            }

            // バリデーション済みデータの取得
            $validated = $validator->validated();

            // バリデーション済みデータの一部を取得
            $validated = $validator->safe()->only(['name', 'email']);
            $validated = $validator->safe()->except(['name', 'email']);

            // ブログポストの保存処理…

            return redirect('/posts');
        }
    }

`make`メソッドの第１引数は、バリデーションを行うデータです。第２引数はそのデータに適用するバリデーションルールの配列です。

リクエストのバリデーションが失敗したかを判断した後、`withErrors`メソッドを使用してエラーメッセージをセッションへ一時保持保存できます。この方法を使用すると、リダイレクト後に`$errors`変数がビューで自動的に共有されるため、メッセージをユーザーへ簡単に表示できます。`withErrors`メソッドは、バリデータ、`MessageBag`、またはPHPの`array`を引数に取ります。

#### 最初のバリデーション失敗時に停止

`stopOnFirstFailure`メソッドは、バリデーションが失敗したら、すぐにすべての属性のバリデーションを停止する必要があることをバリデータへ指示します。

    if ($validator->stopOnFirstFailure()->fails()) {
        // ...
    }

<a name="automatic-redirection"></a>
### 自動リダイレクト

バリデータインスタンスを手作業で作成するが、HTTPリクエストの`validate`メソッドによって提供される自動リダイレクトを利用したい場合は、既存のバリデータインスタンスで`validate`メソッドを呼び出してください。バリデーションが失敗した場合、ユーザーは自動的にリダイレクトされます。XHRリクエストの場合は、[JSONレスポンスが返されます](#validation-error-response-format)。

    Validator::make($request->all(), [
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ])->validate();

`validateWithBag`メソッドを使用しリクエストのバリデートを行った結果、失敗した場合に、エラーメッセージを[名前付きエラーバッグ](#named-error-bags)へ保存することもできます。

    Validator::make($request->all(), [
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ])->validateWithBag('post');

<a name="named-error-bags"></a>
### 名前付きエラーバッグ

1つのページに複数のフォームがある場合は、バリデーションエラーを含む`MessageBag`に名前を付けて、特定のフォームのエラーメッセージを取得可能にできます。これを実現するには、`withErrors`の２番目の引数として名前を渡します。

    return redirect('register')->withErrors($validator, 'login');

`$errors`変数を使い、名前を付けた`MessageBag`インスタンスへアクセスできます。

```blade
{{ $errors->login->first('email') }}
```

<a name="manual-customizing-the-error-messages"></a>
### エラーメッセージのカスタマイズ

必要に応じて、Laravelが提供するデフォルトのエラーメッセージの代わりにバリデータインスタンスが使用するカスタムエラーメッセージを指定できます。カスタムメッセージを指定する方法はいくつかあります。まず、カスタムメッセージを`validator::make`メソッドに３番目の引数として渡す方法です。

    $validator = Validator::make($input, $rules, $messages = [
        'required' => 'The :attribute field is required.',
    ]);

この例の、`:attribute`プレースホルダーは、バリデーション中のフィールドの名前に置き換えられます。バリデーションメッセージには他のプレースホルダーも利用できます。例をご覧ください。

    $messages = [
        'same' => 'The :attribute and :other must match.',
        'size' => 'The :attribute must be exactly :size.',
        'between' => 'The :attribute value :input is not between :min - :max.',
        'in' => 'The :attribute must be one of the following types: :values',
    ];

<a name="specifying-a-custom-message-for-a-given-attribute"></a>
#### 特定の属性に対するカスタムメッセージ指定

特定の属性に対してのみカスタムエラーメッセージを指定したい場合があります。「ドット」表記を使用してこれを行えます。最初に属性の名前を指定し、次にルールを指定します。

    $messages = [
        'email.required' => 'We need to know your email address!',
    ];

<a name="specifying-custom-attribute-values"></a>
#### カスタム属性値の指定

Laravelの組み込みエラーメッセージの多くは、バリデーション中のフィールドや属性の名前に置き換えられる`:attribute`プレースホルダーを含んでいます。特定のフィールドでこれらのプレースホルダーを置き換える値をカスタマイズするには、カスタム属性表示名の配列を4番目の引数として`Validator::make`メソッドに渡します。

    $validator = Validator::make($input, $rules, $messages, [
        'email' => 'email address',
    ]);

<a name="performing-additional-validation"></a>
### 追加バリデーションの実行

最初のバリデーションが完了した後に、追加のバリデーションを実行する必要がある場合があります。バリデータの`after`メソッドで実現可能です。`after`メソッドは、バリデーションが完了した後に呼び出すCallableやクロージャの配列を引数に取ります。指定Callablesは、`Illuminate\Validation\Validator`インスタンスを受け取るので、必要に応じ追加のエラーメッセージを表示できます。

    use Illuminate\Support\Facades\Validator;

    $validator = Validator::make(/* ... */);

    $validator->after(function ($validator) {
        if ($this->somethingElseIsInvalid()) {
            $validator->errors()->add(
                'field', 'Something is wrong with this field!'
            );
        }
    });

    if ($validator->fails()) {
        // ...
    }

前述のとおり、`after`メソッドは、呼び出し可能な配列を引数に取ります。「バリデーション後」のロジックが 実行可能なクラスへカプセル化されている場合は特に便利です。そのクラスは`__invoke`メソッドで、`Illuminate\Validation\Validator`インスタンスを受け取ります。

```php
use App\Validation\ValidateShippingTime;
use App\Validation\ValidateUserStatus;

$validator->after([
    new ValidateUserStatus,
    new ValidateShippingTime,
    function ($validator) {
        // ...
    },
]);
```

<a name="working-with-validated-input"></a>
## バリデーション済み入力の利用

フォームのリクエストや手作業で作成したバリデータのインスタンスを使って リクエストデータをバリデーションした後に、バリデーション済みの受信リクエストデータを取得したいと思われるかもしれません。これには方法がいくつかあります。まず、フォームのリクエストやバリデータインスタンスの`validated`メソッドを呼び出す方法です。このメソッドは、バリデーション済みデータの配列を返します。

    $validated = $request->validated();

    $validated = $validator->validated();

または、フォームリクエストやバリデータのインスタンスに対して、`safe`メソッドを呼び出すこともできます。このメソッドは、`Illuminate\Support\ValidatedInput`のインスタンスを返します。このオブジェクトは`only`、`except`、`all`メソッドを用意しており、バリデーション済みデータのサブセットや配列全体を取得できます。

    $validated = $request->safe()->only(['name', 'email']);

    $validated = $request->safe()->except(['name', 'email']);

    $validated = $request->safe()->all();

さらに、`Illuminate\Support\ValidatedInput`インスタンスをループで配列のようにアクセスすることもできます。

    // バリデーション済みデータをループ
    foreach ($request->safe() as $key => $value) {
        // ...
    }

    // バリデーション済みデータを配列としてアクセス
    $validated = $request->safe();

    $email = $validated['email'];

バリデーション済みデータへさらにフィールドを追加したい場合は、`merge`メソッドを呼び出します。

    $validated = $request->safe()->merge(['name' => 'Taylor Otwell']);

バリデーション済みデータを[コレクション](/docs/{{version}}/collections)インスタンスとして取得したい場合は、`collect`メソッドを呼び出します。

    $collection = $request->safe()->collect();

<a name="working-with-error-messages"></a>
## エラーメッセージの操作

`Validator`インスタンスの`errors`メソッドを呼び出すと、エラーメッセージを操作する便利なメソッドを数揃えた、`Illuminate\Support\MessageBag`インスタンスを受け取れます。自動的に作成され、すべてのビューで使用できる`$errors`変数も、`MessageBag`クラスのインスタンスです。

<a name="retrieving-the-first-error-message-for-a-field"></a>
#### 指定フィールドの最初のエラーメッセージ取得

指定したフィールドの最初のエラーメッセージを取得するには、`first`メソッドを使います。

    $errors = $validator->errors();

    echo $errors->first('email');

<a name="retrieving-all-error-messages-for-a-field"></a>
#### Retrieving All Error Messages for a Field

指定したフィールドの全エラーメッセージを配列で取得したい場合は、`get`メソッドを使います。

    foreach ($errors->get('email') as $message) {
        // ...
    }

配列形式のフィールドをバリデーションする場合は、`*`文字を使用し、各配列要素の全メッセージを取得できます。

    foreach ($errors->get('attachments.*') as $message) {
        // ...
    }

<a name="retrieving-all-error-messages-for-all-fields"></a>
#### 全フィールドの全エラーメッセージ取得

全フィールドの全メッセージの配列を取得したい場合は、`all`メソッドを使います。

    foreach ($errors->all() as $message) {
        // ...
    }

<a name="determining-if-messages-exist-for-a-field"></a>
#### 指定フィールドのメッセージ存在確認

`has`メソッドは、指定したフィールドのエラーメッセージが存在しているかを判定するために使います。

    if ($errors->has('email')) {
        // ...
    }

<a name="specifying-custom-messages-in-language-files"></a>
### 言語ファイルでのカスタムメッセージの指定

Laravelの組み込み検証ルールには、それぞれエラーメッセージがあり、アプリケーションの`lang/en/validation.php`ファイル内に用意しています。アプリケーションに`lang`ディレクトリがない場合は、`lang:publish` Artisanコマンドにより、Laravelへ作成を指示してください。

`lang/en/validation.php`ファイル内に、各バリデーションルールの翻訳エントリーがあります。これらのメッセージは、アプリケーションのニーズに応じて自由に変更・修正してください。

さらに、このファイルを他の言語ディレクトリにコピーして、アプリケーションように言語用メッセージを翻訳することもできます。Laravelの多言語化の詳細については、完全な[多言語化のドキュメント](/docs/{{version}}/localization)をチェックしてみてください。

> [!WARNING]
> Laravelアプリケーションのスケルトンは、デフォルトで`lang`ディレクトリを用意していません。Laravelの言語ファイルをカスタマイズしたい場合は、`lang:publish` Artisanコマンドでリソース公開してください。

<a name="custom-messages-for-specific-attributes"></a>
#### 特定の属性のカスタムメッセージ

アプリケーションのバリデーション言語ファイルでは、特定の属性とルールの組み合わせで使用するエラーメッセージをカスタマイズできます。これを行うには、アプリケーションの`lang/xx/validation.php`言語ファイルの`custom`配列に、カスタマイズメッセージを追加します。

    'custom' => [
        'email' => [
            'required' => 'We need to know your email address!',
            'max' => 'Your email address is too long!'
        ],
    ],

<a name="specifying-attribute-in-language-files"></a>
### 言語ファイルでの属性の指定

Laravelの組み込みエラーメッセージの多くは、`:attribute`プレースホルダーを含んでおり、バリデーション中のフィールドや属性の名前に置き換えます。もし、バリデーションメッセージの`:attribute`部分をカスタム値に置き換えたい場合は、言語ファイル`lang/xx/validation.php`の`attributes`配列でカスタム属性名を指定できます。

    'attributes' => [
        'email' => 'email address',
    ],

> [!WARNING]
> Laravelアプリケーションのスケルトンは、デフォルトで`lang`ディレクトリを用意していません。Laravelの言語ファイルをカスタマイズしたい場合は、`lang:publish` Artisanコマンドでリソース公開してください。

<a name="specifying-values-in-language-files"></a>
### 言語ファイルでの値の指定

Laravelの組み込みバリデーションルールエラーメッセージの一部は、リクエスト属性の現在の値に置き換えられる`:value`プレースホルダーを含んでいます。しかし、バリデーションメッセージの`:value`部分を値のカスタム表現へ置き換えたい場合があるでしょう。たとえば、`payment_type`の値が`cc`の場合にクレジットカード番号が必要であることを定義する次のルールについて考えてみます。

    Validator::make($request->all(), [
        'credit_card_number' => 'required_if:payment_type,cc'
    ]);

このバリデーションルールが通らない場合に、次のようなメッセージが表示されるとします。

```none
The credit card number field is required when payment type is cc.
```

支払タイプの値に`cc`と表示する代わりに、`lang/xx/validation.php`言語ファイルで`values`配列を定義し、よりユーザーフレンドリーな値表現を指定できます。

    'values' => [
        'payment_type' => [
            'cc' => 'credit card'
        ],
    ],

> [!WARNING]
> Laravelアプリケーションのスケルトンは、デフォルトで`lang`ディレクトリを用意していません。Laravelの言語ファイルをカスタマイズしたい場合は、`lang:publish` Artisanコマンドでリソース公開してください。

この値を定義したら、バリデーションルールは次のエラーメッセージを生成します。

```none
The credit card number field is required when payment type is credit card.
```

<a name="available-validation-rules"></a>
## 使用可能なバリデーションルール

使用可能な全バリデーションルールとその機能の一覧です。

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
</style>

<div class="collection-method-list" markdown="1">

[受け入れ](#rule-accepted)
[条件一致時受け入れ](#rule-accepted-if)
[アクティブなURL](#rule-active-url)
[（日付）より後](#rule-after)
[（日付）以降](#rule-after-or-equal)
[アルファベット](#rule-alpha)
[アルファベット記号](#rule-alpha-dash)
[アルファベット数字](#rule-alpha-num)
[配列](#rule-array)
[Ascii](#rule-ascii)
[継続終了](#rule-bail)
[（日付）より前](#rule-before)
[（日付）以前](#rule-before-or-equal)
[範囲](#rule-between)
[論理](#rule-boolean)
[確認](#rule-confirmed)
[現在のパスワード](#rule-current-password)
[日付](#rule-date)
[同一日付](#rule-date-equals)
[日付形式](#rule-date-format)
[小数点以下](#rule-decimal)
[拒否](#rule-declined)
[条件一致時拒否](#rule-declined-if)
[相違](#rule-different)
[桁指定数値](#rule-digits)
[桁範囲指定数値](#rule-digits-between)
[寸法(画像ファイル)](#rule-dimensions)
[別々](#rule-distinct)
[文字列非開始](#rule-doesnt-start-with)
[文字列非終了](#rule-doesnt-end-with)
[メールアドレス](#rule-email)
[文字列終了](#rule-ends-with)
[列挙型](#rule-enum)
[除外](#rule-exclude)
[条件一致時フィールド除外](#rule-exclude-if)
[条件不一致時フィールド除外](#rule-exclude-unless)
[存在時フィールド除外](#rule-exclude-with)
[不在時フィールド除外](#rule-exclude-without)
[存在（データベース）](#rule-exists)
[拡張子](#rule-extensions)
[ファイル](#rule-file)
[充満](#rule-filled)
[より大きい](#rule-gt)
[以上](#rule-gte)
[Hex Color](#rule-hex-color)
[画像（ファイル)](#rule-image)
[内包](#rule-in)
[配列内](#rule-in-array)
[整数](#rule-integer)
[IPアドレス](#rule-ip)
[JSON](#rule-json)
[より小さい](#rule-lt)
[以下](#rule-lte)
[リスト](#rule-list)
[小文字](#rule-lowercase)
[MACアドレス](#rule-mac)
[最大値](#rule-max)
[最大桁数](#rule-max-digits)
[MIMEタイプ](#rule-mimetypes)
[MIMEタイプ(ファイル拡張子)](#rule-mimes)
[最小値](#rule-min)
[最小桁数](#rule-min-digits)
[非入力](#rule-missing)
[条件一致時非入力](#rule-missing-if)
[条件不一致時非入力](#rule-missing-unless)
[存在時非入力](#rule-missing-with)
[全不在時非入力](#rule-missing-with-all)
[倍数値](#rule-multiple-of)
[非内包](#rule-not-in)
[正規表現不一致](#rule-not-regex)
[NULL許可](#rule-nullable)
[数値](#rule-numeric)
[存在](#rule-present)
[条件一致時存在](#rule-present-if)
[条件非一致時存在](#rule-present-unless)
[存在時存在](#rule-present-with)
[全存在時存在](#rule-present-with-all)
[禁止](#rule-prohibited)
[条件一致時禁止](#rule-prohibited-if)
[条件不一致時禁止](#rule-prohibited-unless)
[他フィールド禁止](#rule-prohibits)
[正規表現](#rule-regex)
[必須](#rule-required)
[指定フィールド値一致時必須](#rule-required-if)
[受け入れ時必須](#rule-required-if-accepted)
[受け入れ拒否時必須](#rule-required-if-declined)
[指定フィールド値非一致時必須](#rule-required-unless)
[指定フィールド存在時必須](#rule-required-with)
[全指定フィールド存在時必須](#rule-required-with-all)
[指定フィールド非存在時必須](#rule-required-without)
[全指定フィールド非存在時必須](#rule-required-without-all)
[配列かつキー包含必須](#rule-required-array-keys)
[同一](#rule-same)
[サイズ](#rule-size)
[存在時バリデート実行](#validating-when-present)
[文字列開始](#rule-starts-with)
[文字列](#rule-string)
[タイムゾーン](#rule-timezone)
[一意（データベース）](#rule-unique)
[大文字](#rule-uppercase)
[URL](#rule-url)
[ULID](#rule-ulid)
[UUID](#rule-uuid)

</div>

<a name="rule-accepted"></a>
#### accepted

フィールドが、`"yes"`、`"on"`、`1`、`"1"`、`true`、`"true"`であることをバリデートします。これは、「利用規約」の承認または同様のフィールドをバリデーションするのに役立ちます。

<a name="rule-accepted-if"></a>
#### accepted_if:他のフィールド,値,...

他のフィールドが指定した値と等しい場合、このフィールドは `"yes"`、`"on"`、`1`、`"1"`、`"true"`、`true`であることをバリデートします。これは、「利用規則」の了承や似たようなフィールドをバリデートするのに便利です。

<a name="rule-active-url"></a>
#### active_url

フィールドが、`dns_get_record` PHP関数により、有効なAかAAAAレコードであることをバリデートします。`dns_get_record`へ渡す前に、`parse_url` PHP関数により指定したURLのホスト名を切り出します。

<a name="rule-after"></a>
#### after:_日付_

フィールドは、指定された日付以降の値であることをバリデートします。日付を有効な`DateTime`インスタンスに変換するため、`strtotime`PHP関数に渡します。

    'start_date' => 'required|date|after:tomorrow'

`strtotime`により評価される日付文字列を渡す代わりに、その日付と比較する他のフィールドを指定することもできます。

    'finish_date' => 'required|date|after:start_date'

<a name="rule-after-or-equal"></a>
#### after\_or\_equal:_日付_

フィールドが指定した日付以降であることをバリデートします。詳細は[after](#rule-after)ルールを参照してください。

<a name="rule-alpha"></a>
#### alpha

フィールドは、[`\p{L}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、並びに[`\p{M}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=)に含まれる、Unicodeのアルファベット文字のみであることをバリデートします。

このバリデーションルールをASCII文字（`a-z`と`A-Z`）の範囲に限定したい場合は、`ascii`オプションを指定してください。

```php
'username' => 'alpha:ascii',
```

<a name="rule-alpha-dash"></a>
#### alpha_dash

フィールドは、[`\p{L}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、並びに[`\p{M}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=)、[`\p{N}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=)に含まれる、Unicodeのアルファベット文字と数字、およびASCIIのダッシュ(`-`)とASCIIの下線(`_`)であることをバリデートします。

このバリデーションルールをASCII文字（`a-z`と`A-Z`）の範囲に限定したい場合は、`ascii`オプションを指定してください。

```php
'username' => 'alpha_dash:ascii',
```

<a name="rule-alpha-num"></a>
#### alpha_num

フィールドは、[`\p{L}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AL%3A%5D&g=&i=)、並びに[`\p{M}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AM%3A%5D&g=&i=)、[`\p{N}`](https://util.unicode.org/UnicodeJsps/list-unicodeset.jsp?a=%5B%3AN%3A%5D&g=&i=)に含まれる、Unicodeのアルファベット文字と数字のみであることをバリデートします。

このバリデーションルールをASCII文字（`a-z`と`A-Z`）の範囲に限定したい場合は、`ascii`オプションを指定してください。

```php
'username' => 'alpha_num:ascii',
```

<a name="rule-array"></a>
#### array

フィールドがPHPの配列タイプであることをバリデートします。

`array`ルールに追加の値を指定する場合、入力配列の各キーは、ルールに指定した値のリスト内に存在する必要があります。次の例では、入力配列の`admin`キーは、`array`ルールにした値のリストに含まれていないので、無効です。

    use Illuminate\Support\Facades\Validator;

    $input = [
        'user' => [
            'name' => 'Taylor Otwell',
            'username' => 'taylorotwell',
            'admin' => true,
        ],
    ];

    Validator::make($input, [
        'user' => 'array:name,username',
    ]);

一般に、配列に存在を許すキーは、常に指定する必要があります。

<a name="rule-ascii"></a>
#### ascii

フィールドが全て７ビットのアスキー文字であることをバリデートします。

<a name="rule-bail"></a>
#### bail

フィールドでバリデーションにはじめて失敗した時点で、残りのルールのバリデーションを中止します。

`bail`ルールはバリデーションが失敗したときに、特定のフィールドのバリデーションのみを停止するだけで、一方の`stopOnFirstFailure`メソッドは、ひとつバリデーション失敗が発生したら、すべての属性のバリデーションを停止する必要があることをバリデータに指示します。

    if ($validator->stopOnFirstFailure()->fails()) {
        // ...
    }

<a name="rule-before"></a>
#### before:_日付_

フィールドは、指定された日付より前の値であることをバリデートします。日付を有効な`DateTime`インスタンスへ変換するために、PHPの`strtotime`関数へ渡します。さらに、[`after`](#rule-after)ルールと同様に、バリデーション中の別のフィールドの名前を`date`の値として指定できます。

<a name="rule-before-or-equal"></a>
#### before\_or\_equal:_日付_

フィールドは、指定された日付より前または同じ値であることをバリデートします。日付を有効な`DateTime`インスタンスへ変換するために、PHPの`strtotime`関数へ渡します。さらに、[`after`](#rule-after)ルールと同様に、バリデーション中の別のフィールドの名前を`date`の値として指定できます。

<a name="rule-between"></a>
#### between:_min_,_max_

フィールドが指定した*最小値*以上で、*最大値*以下のサイズであることをバリデートします。[`size`](#rule-size)ルールと同様の判定方法で、文字列、数値、配列、ファイルが評価されます。

<a name="rule-boolean"></a>
#### boolean

フィールドが論理値として有効であることをバリデートします。受け入れられる入力は、`true`、`false`、`1`、`0`、`"1"`、`"0"`です。

<a name="rule-confirmed"></a>
#### confirmed

フィールドが、`{field}_confirmation`フィールドと一致する必要があります。たとえば、バリデーション中のフィールドが「password」の場合、「password_confirmation」フィールドが入力に存在し一致している必要があります。

<a name="rule-current-password"></a>
#### current_password

フィールドが、認証されているユーザーのパスワードと一致することをバリデートします。ルールの最初のパラメータで、[認証ガード](/docs/{{version}}/authentication)を指定できます。

    'password' => 'current_password:api'

<a name="rule-date"></a>
#### date

パリデーションされる値はPHP関数の`strtotime`により有効であり、相対日付ではないことをバリデートします。

<a name="rule-date-equals"></a>
#### date_equals:_日付_

フィールドが、指定した日付と同じことをバリデートします。日付を有効な`DateTime`インスタンスに変換するために、PHPの`strtotime`関数へ渡します。

<a name="rule-date-format"></a>
#### date_format:_フォーマット_,…

バリデーションされる値が、指定する*フォーマット*定義のどれか一つと一致するかバリデートします。バリデーション時には`date`か`date_format`の*どちらか*を使用しなくてはならず、両方はできません。このバリデーションはPHPの[DateTime](https://www.php.net/manual/ja/class.datetime.php)クラスがサポートするフォーマットをすべてサポートしています。

<a name="rule-decimal"></a>
#### decimal:_min_,_max_

フィールドが数値であり、指定された小数点以下の桁数を含んでいることをバリデートします。

    // 小数点以下が2桁ピッタリの必要がある(9.99)
    'price' => 'decimal:2'

    // 小数点以下が２から４桁である必要がある
    'price' => 'decimal:2,4'

<a name="rule-declined"></a>
#### declined

The field under validation must be `"no"`, `"off"`, `0`, `"0"`, `false`, or `"false"`.

<a name="rule-declined-if"></a>
#### declined_if:他のフィールド,値,...

The field under validation must be `"no"`, `"off"`, `0`, `"0"`, `false`, or `"false"` if another field under validation is equal to a specified value.

<a name="rule-different"></a>
#### different:_フィールド_

フィールドが指定された*フィールド*と異なった値を指定されていることをバリデートします。

<a name="rule-digits"></a>
#### digits:_値_

フィールドが整数で、*値*の桁数であることをバリデートします。

<a name="rule-digits-between"></a>
#### digits_between:_最小値_,_最大値_

フィールドが整数で、桁数が*最小値*から*最大値*の間であることをバリデートします。

<a name="rule-dimensions"></a>
#### dimensions

バリデーション対象のファイルが、パラメータにより指定されたサイズに合致することをバリデートします。

    'avatar' => 'dimensions:min_width=100,min_height=200'

使用可能なパラメータは、*min\_width*、*max\_width*、*min\_height*、*max\_height*、*width*、*height*、*ratio*です。

*_ratio_*制約は、幅を高さで割ったものとして表す必要があります。これは、`3/2`のような分数または`1.5`のようなfloatのいずれかで指定します。

    'avatar' => 'dimensions:ratio=3/2'

このルールは多くの引数を要求するので、`Rule::dimensions`メソッドを使い、記述的にこのルールを構築してください。

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'avatar' => [
            'required',
            Rule::dimensions()->maxWidth(1000)->maxHeight(500)->ratio(3 / 2),
        ],
    ]);

<a name="rule-distinct"></a>
#### distinct

配列のバリデーション時、フィールドに重複した値がないことをバリデートします。

    'foo.*.id' => 'distinct'

distinctはデフォルトで緩い比較を使用します。厳密な比較を使用するには、検証ルール定義に`strict`パラメータを追加することができます。

    'foo.*.id' => 'distinct:strict'

バリデーションルールの引数に`ignore_case`を追加して、大文字と小文字の違いを無視するルールを加えられます。

    'foo.*.id' => 'distinct:ignore_case'

<a name="rule-doesnt-start-with"></a>
#### doesnt_start_with:_foo_,_bar_,...

フィールドの値が指定した値で開始しないことをバリデートします。

<a name="rule-doesnt-end-with"></a>
#### doesnt_end_with:_foo_,_bar_,...

フィールドの値が指定した値で終了しないことをバリデートします。

<a name="rule-email"></a>
#### email

バリデーション中のフィールドは、電子メールアドレスのフォーマットである必要があります。このバリデーションルールは、[`egulias/email-validator`](https://github.com/egulias/EmailValidator)パッケージを使用して電子メールアドレスをバリデーションします。デフォルトでは`RFCValidation`バリデータが適用されますが、他のバリデーションスタイルを適用することもできます。

    'email' => 'email:rfc,dns'

上記の例では、`RFCValidation`と`DNSCheckValidation`バリデーションを適用しています。適用可能なバリデーションスタイルは、次の通りです。

<div class="content-list" markdown="1">

- `rfc`: `RFCValidation`
- `strict`: `NoRFCWarningsValidation`
- `dns`: `DNSCheckValidation`
- `spoof`: `SpoofCheckValidation`
- `filter`: `FilterEmailValidation`
- `filter_unicode`: `FilterEmailValidation::unicode()`

</div>

PHPの`filter_var`関数を使用する`filter`バリデータは、Laravelに付属しており、Laravelバージョン5.8より前のLaravelのデフォルトの電子メールバリデーション動作でした。

> [!WARNING]
> `dns`および`spoof`バリデータには、PHPの`intl`拡張が必要です。

<a name="rule-ends-with"></a>
#### ends_with:_foo_,_bar_,...

フィールドの値が、指定された値で終わることをバリデートします。

<a name="rule-enum"></a>
#### enum

`Enum`ルールはクラスベースのルールで、対象のフィールドに有効なenumの値が含まれているかどうかをバリデートします。`Enum`ルールは、コンストラクタの引数に唯一、enumの名前を取ります。プリミティブ値のバリデーションを行う際は、`Enum`ルールに値に依存した（backed）Enumを指定しなければなりません。

    use App\Enums\ServerStatus;
    use Illuminate\Validation\Rule;

    $request->validate([
        'status' => [Rule::enum(ServerStatus::class)],
    ]);

`Enum`ルールの`only`メソッドと`except`メソッドを使用し、有効な列挙ケースを制限できます。

    Rule::enum(ServerStatus::class)
        ->only([ServerStatus::Pending, ServerStatus::Active]);

    Rule::enum(ServerStatus::class)
        ->except([ServerStatus::Pending, ServerStatus::Active]);

`when`メソッドは、`Enum`ルールを条件付きで変更するために使います。

```php
use Illuminate\Support\Facades\Auth;
use Illuminate\Validation\Rule;

Rule::enum(ServerStatus::class)
    ->when(
        Auth::user()->isAdmin(),
        fn ($rule) => $rule->only(...),
        fn ($rule) => $rule->only(...),
    );
```

<a name="rule-exclude"></a>
#### exclude

フィルードの値が`validate`と`validated`メソッドが返すリクエストデータから除外されていることをバリデートします。

<a name="rule-exclude-if"></a>
#### exclude_if:_他のフィールド_,_値_

*他のフィールド*が*値*と等しい場合、`validate`と`validated`メソッドが返すリクエストデータから、バリデーション指定下のフィールドが除外されます。

複雑な条件付き除外ロジックが必要な場合は、`Rule::excludeIf`メソッドが便利です。このメソッドは、論理値かクロージャを引数に取ります。クロージャを指定する場合、そのクロージャは、`true`または`false`を返し、バリデーション中のフィールドを除外するかしないかを示す必要があります。

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($request->all(), [
        'role_id' => Rule::excludeIf($request->user()->is_admin),
    ]);

    Validator::make($request->all(), [
        'role_id' => Rule::excludeIf(fn () => $request->user()->is_admin),
    ]);

<a name="rule-exclude-unless"></a>
#### exclude_unless:_他のフィールド_,_値_

*他のフィールド*が*値*と等しくない場合、`validate`と`validated`メソッドが返すリクエストデータから、バリデーション指定下のフィールドが除外されます。もし*値*が`null`（`exclude_unless:name,null`）の場合は、比較フィールドが`null`であるか、比較フィールドがリクエストデータに含まれていない限り、バリデーション指定下のフィールドは除外されます。

<a name="rule-exclude-with"></a>
#### exclude_with:_他のフィールド_

*他のフィールド*が存在する場合、`validate`と`validated`メソッドが返すリクエストデータから、バリデーション指定可のフィールドが除外されます。

<a name="rule-exclude-without"></a>
#### exclude_without:_他のフィールド_

*他のフィールド*が存在しない場合、`validate`と`validated`メソッドが返すリクエストデータから、バリデーション指定下のフィールドが除外されます。

<a name="rule-exists"></a>
#### exists:_テーブル_,_カラム_

フィールドは、指定のデータベーステーブルに存在する必要があります。

<a name="basic-usage-of-exists-rule"></a>
#### 基本的なExistsルールの使用法

    'state' => 'exists:states'

`column`オプションが指定されていない場合、フィールド名が使用されます。したがって、この場合、ルールは、`states`データベーステーブルに、リクエストの`state`属性値と一致する`state`カラム値を持つレコードが含まれていることをバリデーションします。

<a name="specifying-a-custom-column-name"></a>
#### カスタムカラム名の指定

データベーステーブル名の後に配置することで、バリデーションルールで使用するデータベースカラム名を明示的に指定できます。

    'state' => 'exists:states,abbreviation'

場合によっては、`exists`クエリに使用する特定のデータベース接続を指定する必要があります。これは、接続名をテーブル名の前に付けることで実現できます。

    'email' => 'exists:connection.staff,email'

テーブル名を直接指定する代わりに、Eloquentモデルを指定することもできます。

    'user_id' => 'exists:App\Models\User,id'

バリデーションルールで実行されるクエリをカスタマイズしたい場合は、ルールをスムーズに定義できる`Rule`クラスを使ってください。下の例では、`|`文字を区切りとして使用する代わりに、バリデーションルールを配列として指定しています。

    use Illuminate\Database\Query\Builder;
    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'email' => [
            'required',
            Rule::exists('staff')->where(function (Builder $query) {
                return $query->where('account_id', 1);
            }),
        ],
    ]);

`Rule::exists`メソッドが生成する、`exists`ルールで使用するデータベースカラム名は、`exists`メソッドの第２引数として明示的に指定できます。

    'state' => Rule::exists('states', 'abbreviation'),

<a name="rule-extensions"></a>
#### extensions:_foo_,_bar_,...

ファイルはリストする拡張子のいずれかに一致することをバリデートします。

    'photo' => ['required', 'extensions:jpg,png'],

> [!WARNING]
> ユーザーが割り当てた拡張子だけでファイルをバリデートしてはいけません。このルールは通常、[`mimes`](#rule-mimes)または[`mimetypes`](#rule-mimetypes)ルールと組み合わせて使用する必要があります。

<a name="rule-file"></a>
#### file

フィールドがアップロードに成功したファイルであることをバリデートします。

<a name="rule-filled"></a>
#### filled

フィールドが存在する場合、空でないことをバリデートします。

<a name="rule-gt"></a>
#### gt:_field_

フィールドが指定した*フィールド*か*値*より大きいことをバリデートします。２つのフィールドは同じタイプでなくてはなりません。文字列、数値、配列、ファイルは、[`size`](#rule-size)ルールと同じ規約により評価します。

<a name="rule-gte"></a>
#### gte:_field_

フィールドが指定した*フィールド*か*値*以上であることをバリデートします。２つのフィールドは同じタイプでなくてはなりません。文字列、数値、配列、ファイルは、[`size`](#rule-size)ルールと同じ規約により評価します。

<a name="rule-hex-color"></a>
#### hex_color

フィールドが、[16進数](https://developer.mozilla.org/en-US/docs/Web/CSS/hex-color)形式の有効な色値を含んでいることをバリデートします。

<a name="rule-image"></a>
#### image

ファイルは画像（jpg、jpeg、png、bmp、gif、svg、webp）であることをバリデートします。

<a name="rule-in"></a>
#### in:foo,bar...

フィールドが指定したリストの中の値に含まれていることをバリデートします。このルールを使用するために配列を`implode`する必要が多くなりますので、ルールを記述的に構築するには、`Rule::in`メソッドを使ってください。

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'zones' => [
            'required',
            Rule::in(['first-zone', 'second-zone']),
        ],
    ]);

`in`ルールと`array`ルールを組み合わせた場合、入力配列の各値は、`in`ルールに指定した値のリスト内に存在しなければなりません。次の例では，入力配列中の`LAS`空港コードは，`in`ルールへ指定した空港のリストに含まれていないため無効です。

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    $input = [
        'airports' => ['NYC', 'LAS'],
    ];

    Validator::make($input, [
        'airports' => [
            'required',
            'array',
        ],
        'airports.*' => Rule::in(['NYC', 'LIT']),
    ]);

<a name="rule-in-array"></a>
#### in_array:_他のフィールド_.*

フィールドが、*他のフィールド*の値のどれかであることをバリデートします。

<a name="rule-integer"></a>
#### integer

フィールドが整数値であることをバリデートします。

> [!WARNING]
> このバリデーションルールは、入力が「整数」変数タイプであるかを確認しません。入力がPHPの`FILTER_VALIDATE_INT`ルールで受け入れられるかバリデーションするだけです。入力を数値としてバリデーションする必要がある場合は、このルールを[`numeric`バリデーションルール](#rule-numeric)と組み合わせて使用​​してください。

<a name="rule-ip"></a>
#### ip

フィールドがIPアドレスの形式として正しいことをバリデートします。

<a name="ipv4"></a>
#### ipv4

フィールドがIPv4アドレスの形式として正しいことをバリデートします。

<a name="ipv6"></a>
#### ipv6

フィールドがIPv6アドレスの形式として正しいことをバリデートします。

<a name="rule-json"></a>
#### json

フィールドが有効なJSON文字列であることをバリデートします。

<a name="rule-lt"></a>
#### lt:_field_

フィールドが指定した*フィールド*より小さいことをバリデートします。２つのフィールドは同じタイプでなくてはなりません。文字列、数値、配列、ファイルは、[`size`](#rule-size)ルールと同じ規約により評価します。

<a name="rule-lte"></a>
#### lte:_field_

フィールドが指定した*フィールド*以下であることをバリデートします。２つのフィールドは同じタイプでなくてはなりません。文字列、数値、配列、ファイルは、[`size`](#rule-size)ルールと同じ規約により評価します。

<a name="rule-lowercase"></a>
#### lowercase

フィールドが小文字であることをバリデートします。

<a name="rule-list"></a>
#### list

フィールドが、リストの配列であることをバリデートします。配列は、キーが0から`count($array) - 1`までの連続した数値で構成されている場合、リストとみなします。

<a name="rule-mac"></a>
#### mac_address

フィールドがMACアドレスとして正しいことをバリデートします。

<a name="rule-max"></a>
#### max:_値_

フィールドが最大値として指定された*値*以下であることをバリデートします。[`size`](#rule-size)ルールと同様の判定方法で、文字列、数値、配列、ファイルが評価されます。

<a name="rule-max-digits"></a>
#### max_digits:_値_

整数フィールドが、最大*値*桁数以下であることをバリデートします。

<a name="rule-mimetypes"></a>
#### mimetypes:*text/plain*,...

フィールドが指定するMIMEタイプのどれかであることをバリデートします。

    'video' => 'mimetypes:video/avi,video/mpeg,video/quicktime'

アップロードしたファイルのMIMEタイプを判別するために、ファイルの内容が読み取られ、フレームワークはMIMEタイプを推測します。これは、クライアントが提供するMIMEタイプとは異なる場合があります。

<a name="rule-mimes"></a>
#### mimes:_foo_,_bar_,...

ファイルは、リストする拡張子のいずれかに対応するMIMEタイプを持っていることをバリデートします。

    'photo' => 'mimes:jpg,bmp,png'

拡張子を指定するだけでもよいのですが、このルールは実際には、ファイルの内容を読み取ってそのMIMEタイプを推測することにより、ファイルのMIMEタイプをバリデーションします。MIMEタイプとそれに対応する拡張子の完全なリストは、次の場所にあります。

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)

<a name="mime-types-and-extensions"></a>
#### MIMEタイプと拡張子

このバリデーションルールは、MIMEタイプとユーザーがファイルに割り当てた拡張子との一致をバリデートするものではありません。例えば、`mimes:png`バリデーションルールは、ファイル名が`photo.txt`であっても、有効なPNGコンテンツを含むファイルを有効なPNG画像とみなします。ユーザーがファイルへ割り当てた拡張子をバリデートしたい場合は、[`extensions`](#rule-extensions)ルールを使用してください。

<a name="rule-min"></a>
#### min:_値_

フィールドが最小値として指定された*値*以上であることをバリデートします。[`size`](#rule-size)ルールと同様の判定方法で、文字列、数値、配列、ファイルが評価されます。

<a name="rule-min-digits"></a>
#### min_digits:_値_

整数フィールドが、最小*値*桁数以上であることをバリデートします。

<a name="rule-multiple-of"></a>
#### multiple_of:_値_

フィールドが、*値*の倍数であることをバリデートします。

<a name="rule-missing"></a>
#### missing

フィールドが入力データ中に存在しないことをバリデートします。

 <a name="rule-missing-if"></a>
 #### missing_if:_他のフィールド_,_値_,...

 *他のフィールド*が*値*のどれかと等しい場合、フィールドが入力データ中に存在しないことをバリデートします。

 <a name="rule-missing-unless"></a>
 #### missing_unless:_他のフィールド_,_値_

*他のフィールド*が*値*の全てに一致しない場合、フィールドが入力データ中に存在しないことをバリデートします。

 <a name="rule-missing-with"></a>
 #### missing_with:_foo_,_bar_,...

指定したフィールドのどれかが存在する*場合のみ*、フィールドが入力データ中に存在しないことをバリデートします。

 <a name="rule-missing-with-all"></a>
 #### missing_with_all:_foo_,_bar_,...

 指定したフィールド全てが存在する*場合のみ*、フィールドが入力データ中に存在しないことをバリデートします。

<a name="rule-not-in"></a>
#### not_in:_foo_,_bar_,...

フィールドが指定された値のリスト中に含まれていないことをバリデートします。`Rule::notIn`メソッドのほうが、ルールの構成が読み書きしやすいでしょう。

    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'toppings' => [
            'required',
            Rule::notIn(['sprinkles', 'cherries']),
        ],
    ]);

<a name="rule-not-regex"></a>
#### not_regex:_正規表現_

フィールドが指定した正規表現と一致しないことをバリデートします。

Internally, this rule uses the PHP `preg_match` function. The pattern specified should obey the same formatting required by `preg_match` and thus also include valid delimiters. For example: `'email' => 'not_regex:/^.+$/i'`.

> [!WARNING]
> `regex`／`not_regex`パターンを使用するとき、特に正規表現に`|`文字が含まれている場合は、`|`区切り文字を使用する代わりに配列を使用してバリデーションルールを指定する必要があります。

<a name="rule-nullable"></a>
#### nullable

フィールドが`null`値であることを許容します。

<a name="rule-numeric"></a>
#### numeric

フィールドは[数値](https://www.php.net/manual/ja/function.is-numeric.php)であることをバリデートします。

<a name="rule-present"></a>
#### present

フィールドが入力データに存在していることをバリデートします。

<a name="rule-present-if"></a>
#### present_if:_他フィールド_,_値_,…

**他フィールド**が**値**のどれかに等しい場合、フィールドが存在することをバリデートします。

<a name="rule-present-unless"></a>
#### present_unless:_他フィールド_,_値_

**他フィールド**が**value**のいずれにも等しくない場合、フィールドが存在することをバリデートします。

<a name="rule-present-with"></a>
#### present_with:_foo_,_bar_,...

指定する他のフィールドのいずれかが存在する**場合のみ**、フィールドが存在することをバリデートします。

<a name="rule-present-with-all"></a>
#### present_with_all:_foo_,_bar_,...

指定する他のフィールドすべてがが存在する**場合のみ**、フィールドが存在することをバリデートします。

<a name="rule-prohibited"></a>
#### prohibited

フィールドが存在しないか、または空であることをバリデートします。フィールドが以下の条件のいずれかである場合、「空」であると判定します。

<div class="content-list" markdown="1">

- 値が`null`である。
- 値が空文字列である。
- 値が空配列であるか、空の`Countable`オブジェクトである。
- 値がパスのないアップロード済みファイルである。

</div>

<a name="rule-prohibited-if"></a>
#### prohibited_if:_他のフィールド_,_値_,…

*他のフィールド*が*値*のいずれかと等しい場合、フィールドが存在しないか、空であることをバリデートします。フィールドが以下の条件のいずれかである場合、「空」であると判定します。

<div class="content-list" markdown="1">

- 値が`null`である。
- 値が空文字列である。
- 値が空配列であるか、空の`Countable`オブジェクトである。
- 値がパスのないアップロード済みファイルである。

</div>

複雑な条件付き禁止（空か存在してない）ロジックが必要な場合は、`Rule::prohibitedIf`メソッドを利用してください。このメソッドは、論理値かクロージャを引数に取ります。クロージャを指定する場合、そのクロージャは`true`または`false`を返し、バリデーション中のフィールドを禁止するかしないかを示す必要があります。

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($request->all(), [
        'role_id' => Rule::prohibitedIf($request->user()->is_admin),
    ]);

    Validator::make($request->all(), [
        'role_id' => Rule::prohibitedIf(fn () => $request->user()->is_admin),
    ]);

<a name="rule-prohibited-unless"></a>
#### prohibited_unless:_他のフィールド_,_値_,…

*他のフィールド*が*値*の全てと等しくない場合、フィールドが存在しないか、空であることをバリデートします。フィールドが以下の条件のいずれかである場合、そのフィールドを「空」であると判定します。

<div class="content-list" markdown="1">

- 値が`null`である。
- 値が空文字列である。
- 値が空配列であるか、空の`Countable`オブジェクトである。
- 値がパスのないアップロード済みファイルである。

</div>

<a name="rule-prohibits"></a>
#### prohibits:_他のフィールド_,…

フィールドが存在し空でない場合、全ての*他のフィールド*は存在しないか、空であることをバリデートします。フィールドが以下の条件のいずれかである場合、そのフィールドを「空」であると判定します。

<div class="content-list" markdown="1">

- 値が`null`である。
- 値が空文字列である。
- 値が空配列であるか、空の`Countable`オブジェクトである。
- 値がパスのないアップロード済みファイルである。

</div>

<a name="rule-regex"></a>
#### regex:_正規表現_

フィールドが指定された正規表現にマッチすることをバリデートします。

Internally, this rule uses the PHP `preg_match` function. The pattern specified should obey the same formatting required by `preg_match` and thus also include valid delimiters. For example: `'email' => 'regex:/^.+@.+$/i'`.

> [!WARNING]
> `regex`／`not_regex`パターンを使用するとき、特に正規表現に`|`文字が含まれている場合は、`|`区切り文字を使用する代わりに、配列でルールを指定する必要があります。

<a name="rule-required"></a>
#### required

フィールドが入力データ中に存在し、かつ空でないことをバリデートします。フィールドが以下の条件のいずれかの場合、そのフィールドを「空」であると判定します。

<div class="content-list" markdown="1">

- 値が`null`である。
- 値が空文字列である。
- 値が空配列であるか、空の`Countable`オブジェクトである。
- 値がパスのないアップロード済みファイルである。

</div>

<a name="rule-required-if"></a>
#### required\_if:_他のフィールド_,_値_,...

*他のフィールド*が*値*のどれかと一致している場合、このフィールドが存在し、かつ空でないことをバリデートします。

`required_if`ルールのより複雑な条件を作成したい場合は、`Rule::requiredIf`メソッドを使用できます。このメソッドは、ブール値またはクロージャを受け入れます。クロージャが渡されると、クロージャは「true」または「false」を返し、バリデーション中のフィールドが必要かどうかを示します。

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($request->all(), [
        'role_id' => Rule::requiredIf($request->user()->is_admin),
    ]);

    Validator::make($request->all(), [
        'role_id' => Rule::requiredIf(fn () => $request->user()->is_admin),
    ]);

<a name="rule-required-if-accepted"></a>
#### required_if_accepted:_他のフィールド_,...

*他のフィールド*が`"yes"`、`"on"`、`1`、`"1"`、`true`、`"true"`と等しい場合、このフィールドが存在し、かつ空でないことをバリデートします。

<a name="rule-required-if-declined"></a>
#### required_if_declined:_anotherfield_,...

*他のフィールド*が`"no"`、`"off"`、`0`、`"0"`、`false`、`"false"`と等しい場合、このフィールドが存在し、かつ空でないことをバリデートします。

<a name="rule-required-unless"></a>
#### required\_unless:_他のフィールド_,_値_,...

*他のフィールド*が*値*のどれとも一致していない場合、このフィールドが存在し、かつ空でないことをバリデートします。これは*値*が`null`でない限り、*他のフィールド*はリクエストデータに存在しなければならないという意味でもあります。もし*値*が`null`（`required_unless:name,null`）の場合は、比較フィールドが`null`であるか、リクエストデータに比較フィールドが存在しない限り、バリデーション対象下のフィールドは必須です。

<a name="rule-required-with"></a>
#### required\_with:_foo_,_bar_,...

指定した他のフィールドが*一つでも存在している*場合、このフィールドが存在し、かつ空でないことをバリデートします。

<a name="rule-required-with-all"></a>
#### required\_with_all:_foo_,_bar_,...

指定した他のフィールドが*すべて存在している*場合、このフィールドが存在し、かつ空でないことをバリデートします。

<a name="rule-required-without"></a>
#### required\_without:_foo_,_bar_,...

指定した他のフィールドの*どれか一つでも存在していない*場合、このフィールドが存在し、かつ空でないことをバリデートします。

<a name="rule-required-without-all"></a>
#### required\_without_all:_foo_,_bar_,...

指定した他のフィールドが*すべて存在していない*場合、このフィールドが存在し、かつ空でないことをバリデートします。

<a name="rule-required-array-keys"></a>
#### required_array_keys:_foo_,_bar_,...

フィールドは配列であり、少なくとも指定したキーを含んでいることをバリデートします。

<a name="rule-same"></a>
#### same:_フィールド_

*フィールド*が、指定したフィールドと同じ値であることをバリデートします。

<a name="rule-size"></a>
#### size:_値_

フィールドは指定した*値*と同じサイズであることをバリデートします。文字列の場合、*値*は文字長です。数値項目の場合、*値*は整数値（属性に`numeric`か`integer`ルールを持っている必要があります）です。配列の場合、*値*は配列の個数(`count`)です。ファイルの場合、*値*はキロバイトのサイズです。

    // 文字列長が１２文字ちょうどであることをバリデートする
    'title' => 'size:12';

    // 指定された整数が１０であることをバリデートする
    'seats' => 'integer|size:10';

    // 配列にちょうど５要素あることをバリデートする
    'tags' => 'array|size:5';

    // アップロードしたファイルが５１２キロバイトぴったりであることをバリデートする
    'image' => 'file|size:512';

<a name="rule-starts-with"></a>
#### starts_with:_foo_,_bar_,...

フィールドが、指定した値のどれかで始まることをバリデートします。

<a name="rule-string"></a>
#### string

フィルードは文字列タイプであることをバリデートします。フィールドが`null`であることも許す場合は、そのフィールドに`nullable`ルールも指定してください。

<a name="rule-timezone"></a>
#### timezone

The field under validation must be a valid timezone identifier according to the `DateTimeZone::listIdentifiers` method.

The arguments [accepted by the `DateTimeZone::listIdentifiers` method](https://www.php.net/manual/en/datetimezone.listidentifiers.php) may also be provided to this validation rule:

    'timezone' => 'required|timezone:all';

    'timezone' => 'required|timezone:Africa';

    'timezone' => 'required|timezone:per_country,US';

<a name="rule-unique"></a>
#### unique:_テーブル_,_カラム_

フィールドが、指定されたデータベーステーブルに存在しないことをバリデートします。

**カスタムテーブル／カラム名の指定**

テーブル名を直接指定する代わりに、Eloquentモデルを指定することもできます。

    'email' => 'unique:App\Models\User,email_address'

`column`オプションは、フィールドの対応するデータベースカラムを指定するために使用します。`column`オプションが指定されていない場合、バリデーション中のフィールドの名前が使用されます。

    'email' => 'unique:users,email_address'

**カスタムデータベース接続の指定**

場合により、バリデータが行うデータベースクエリのカスタム接続を指定する必要があります。これには、接続名をテーブル名の前に追加します。

    'email' => 'unique:connection.users,email_address'

**指定されたIDのuniqueルールを無視する**

場合により、uniqueのバリデーション中に特定のIDを無視したいことがあります。たとえば、ユーザーの名前、メールアドレス、および場所を含む「プロファイルの更新」画面について考えてみます。メールアドレスが一意であることを確認したいでしょう。しかし、ユーザーが名前フィールドのみを変更し、メールフィールドは変更しない場合、ユーザーは当該電子メールアドレスの所有者であるため、バリデーションエラーが投げられるのは望ましくありません。

バリデータにユーザーIDを無視するように指示するには、ルールをスムーズに定義できる`Rule`クラスを使います。以下の例の場合、さらにルールを`|`文字を区切りとして使用する代わりに、バリデーションルールを配列として指定しています。

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    Validator::make($data, [
        'email' => [
            'required',
            Rule::unique('users')->ignore($user->id),
        ],
    ]);

> [!WARNING]
> ユーザーがコントロールするリクエストの入力を`ignore`メソッドへ、決して渡してはいけません。代わりに、Eloquentモデルインスタンスの自動増分IDやUUIDのような、生成されたユニークなIDだけを渡してください。そうしなければ、アプリケーションがSQLインジェクション攻撃に対し、脆弱になります。

モデルキーの値を`ignore`メソッドに渡す代わりに、モデルインスタンス全体を渡すこともできます。Laravelはモデルからキーを自動的に抽出します:

    Rule::unique('users')->ignore($user)

もし、テーブルの主キーとして`id`以外のカラム名を使用している場合は、`ignore`メソッドを呼び出す時に、カラムの名前を指定してください。

    Rule::unique('users')->ignore($user->id, 'user_id')

`unique`ルールはデフォルトで、バリデートしようとしている属性名と一致するカラムの同一性をチェックします。しかしながら、`unique`メソッドの第２引数として、異なったカラム名を渡すことも可能です。

    Rule::unique('users', 'email_address')->ignore($user->id)

**追加のWHERE節を付け加える**

`where`メソッドを使用してクエリをカスタマイズすることにより、追加のクエリ条件を指定できます。たとえば、`account_id`列の値が`1`の検索レコードのみ検索するクエリ条件で絞り込むクエリを追加してみます。

    'email' => Rule::unique('users')->where(fn (Builder $query) => $query->where('account_id', 1))

<a name="rule-uppercase"></a>
#### uppercase

フィールドが大文字であることをバリデートします。

<a name="rule-url"></a>
#### url

フィールドが有効なURLであることをバリデートします。

有効と見なすURLプロトコルを指定したい場合は、バリデーションルールのパラメータとして、そのプロトコルを渡すことができます：

```php
'url' => 'url:http,https',

'game' => 'url:minecraft,steam',
```

<a name="rule-ulid"></a>
#### ulid

フィールドが有効なULID（[Universally Unique Lexicographically Sortable Identifier](https://github.com/ulid/spec)）であることをバリデートします。

<a name="rule-uuid"></a>
#### uuid

フィールドが有効な、RFC 4122（バージョン1、3、4、5）universally unique identifier (UUID)であることをバリデートします。

<a name="conditionally-adding-rules"></a>
## 条件付きでルールを追加する

<a name="skipping-validation-when-fields-have-certain-values"></a>
#### フィールドが特定値を持つ場合にバリデーションを飛ばす

他のフィールドに指定値が入力されている場合は、バリデーションを飛ばしたい状況がときどき起きるでしょう。`exclude_if`バリデーションルールを使ってください。`appointment_date`と`doctor_name`フィールドは、`has_appointment`フィールドが`false`値の場合バリデートされません。

    use Illuminate\Support\Facades\Validator;

    $validator = Validator::make($data, [
        'has_appointment' => 'required|boolean',
        'appointment_date' => 'exclude_if:has_appointment,false|required|date',
        'doctor_name' => 'exclude_if:has_appointment,false|required|string',
    ]);

もしくは逆に`exclude_unless`ルールを使い、他のフィールドに指定値が入力されていない場合は、バリデーションを行わないことも可能です。

    $validator = Validator::make($data, [
        'has_appointment' => 'required|boolean',
        'appointment_date' => 'exclude_unless:has_appointment,true|required|date',
        'doctor_name' => 'exclude_unless:has_appointment,true|required|string',
    ]);

<a name="validating-when-present"></a>
#### 項目存在時のバリデーション

状況によっては、フィールドがバリデーション対象のデータに存在する場合に**のみ**、フィールドに対してバリデーションチェックを実行したい場合があります。これをすばやく実行するには、`sometimes`ルールをルールリストに追加します。

    $v = Validator::make($data, [
        'email' => 'sometimes|required|email',
    ]);

上の例では`email`フィールドが、`$data`配列の中に存在している場合のみバリデーションが実行されます。

> [!NOTE]
> フィールドが常に存在しているが、空であることをバリデートする場合は、[この追加フィールドに対する注意事項](#a-note-on-optional-fields)を確認してください。

<a name="complex-conditional-validation"></a>
#### 複雑な条件のバリデーション

ときどきもっと複雑な条件のロジックによりバリデーションルールを追加したい場合も起きます。たとえば他のフィールドが100より大きい場合のみ、指定したフィールドが入力されているかをバリデートしたいときなどです。もしくは２つのフィールドのどちらか一方が存在する場合は、両方共に値を指定する必要がある場合です。こうしたルールを付け加えるのも面倒ではありません。最初に`Validator`インスタンスを生成するのは、**固定ルール**の場合と同じです。

    use Illuminate\Support\Facades\Validator;

    $validator = Validator::make($request->all(), [
        'email' => 'required|email',
        'games' => 'required|numeric',
    ]);

私たちのWebアプリケーションがゲームコレクター向けであると仮定しましょう。ゲームコレクターがアプリケーションに登録し、100を超えるゲームを所有している場合は、なぜこれほど多くのゲームを所有しているのかを説明してもらいます。たとえば、ゲームの再販店を経営している場合や、ゲームの収集を楽しんでいる場合などです。この要件を条件付きで追加するには、`Validator`インスタンスの`sometimes`メソッドが使用できます。

    use Illuminate\Support\Fluent;

    $validator->sometimes('reason', 'required|max:500', function (Fluent $input) {
        return $input->games >= 100;
    });

`sometimes`メソッドに渡す最初の引数は、条件付きでバリデーションするフィールドの名前です。２番目の引数は、追加するルールのリストです。３番目の引数として渡すクロージャが`true`を返す場合、ルールが追加されます。この方法で、複雑な条件付きバリデーションを簡単に作成できます。複数のフィールドに条件付きバリデーションを一度に追加することもできます。

    $validator->sometimes(['reason', 'cost'], 'required', function (Fluent $input) {
        return $input->games >= 100;
    });

> [!NOTE]
> クロージャへ渡される`$input`パラメーターは、`Illuminate\Support\Fluent`のインスタンスであり、バリデーション下の入力とファイルへアクセスするために使用できます。

<a name="complex-conditional-array-validation"></a>
#### 複雑な条件の配列バリデーション

インデックスがわからない、入れ子になった同じ配列の中にある別のフィールドに基づいて、あるフィールドを検証したいことがあります。このような場合には、クロージャへ第２引数を渡してください。第２引数には、配列中のバリデーション対象の現アイテムが渡されます。

    $input = [
        'channels' => [
            [
                'type' => 'email',
                'address' => 'abigail@example.com',
            ],
            [
                'type' => 'url',
                'address' => 'https://example.com',
            ],
        ],
    ];

    $validator->sometimes('channels.*.address', 'email', function (Fluent $input, Fluent $item) {
        return $item->type === 'email';
    });

    $validator->sometimes('channels.*.address', 'url', function (Fluent $input, Fluent $item) {
        return $item->type !== 'email';
    });

クロージャへ渡される`$input`パラメータと同様に、`$item`パラメータは属性データが配列の場合は`Illuminate\Support\Fluent`のインスタンス、そうでない場合は文字列になります。

<a name="validating-arrays"></a>
## 配列のバリデーション

[`array`バリデーションルールのドキュメント](#rule-array)で説明したように、`array`ルールは、許可する配列キーのリストを受け入れます。配列内にその他のキーが存在する場合、バリデーションは失敗します。

    use Illuminate\Support\Facades\Validator;

    $input = [
        'user' => [
            'name' => 'Taylor Otwell',
            'username' => 'taylorotwell',
            'admin' => true,
        ],
    ];

    Validator::make($input, [
        'user' => 'array:name,username',
    ]);

一般的に、配列内に存在することが許される配列キーを常に指定する必要があります。そうしないと、バリデータの`validate`と`validated`メソッドは他のネストした配列バリデーションルールにより検証されていなくても、配列とそのすべてのキーを含むバリデーション済みデータをすべて返してしまうことになります。

<a name="validating-nested-array-input"></a>
### ネストした配列入力のバリデーション

ネストした配列ベースフォームの入力フィールドをバリデーションするのが、面倒である必要はありません。配列内の属性をバリデーションするには、「ドット記法」が使えます。たとえば、HTTPリクエストに`photos[profile]`フィールドが含まれている場合、次のように検証します。

    use Illuminate\Support\Facades\Validator;

    $validator = Validator::make($request->all(), [
        'photos.profile' => 'required|image',
    ]);

配列の各要素をバリデーションすることもできます。たとえば、特定の配列入力フィールドの各メールが一意であることをバリデーションするには、次のようにします。

    $validator = Validator::make($request->all(), [
        'person.*.email' => 'email|unique:users',
        'person.*.first_name' => 'required_with:person.*.last_name',
    ]);

同様に、[言語ファイルのカスタムバリデーションメッセージ](#custom-messages-for-specific-attributes)を指定するときに`*`文字を使用すると、配列ベースのフィールドに単一のバリデーションメッセージを簡単に使用できます。

    'custom' => [
        'person.*.email' => [
            'unique' => 'Each person must have a unique email address',
        ]
    ],

<a name="accessing-nested-array-data"></a>
#### ネストした配列データへのアクセス

バリデーションルールを属性に割り当てる際に、ネストした配列要素の値にアクセスする必要が起きることがあります。このような場合は、`Rule::forEach`メソッドを使用します。`forEach`メソッドは、バリデーション対象の配列属性の繰り返しごとに呼び出すクロージャを引数に取り、値と明示的で完全に展開された属性名を受け取ります。クロージャは、配列の要素に割り当てるルールの配列を返す必要があります。

    use App\Rules\HasPermission;
    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;

    $validator = Validator::make($request->all(), [
        'companies.*.id' => Rule::forEach(function (string|null $value, string $attribute) {
            return [
                Rule::exists(Company::class, 'id'),
                new HasPermission('manage-company', $value),
            ];
        }),
    ]);

<a name="error-message-indexes-and-positions"></a>
### エラーメッセージインデックスとポジション

配列のバリデーションを行うとき、失敗した特定項目のインデックスや位置をアプリケーションのエラーメッセージから参照したいことがあります。これを行うには、[カスタムバリデーションメッセージ] (#manual-customizing-the-error-messages)へ、`:index`（０始点）と`:position`（１始点）のプレースホルダを使ってください。

    use Illuminate\Support\Facades\Validator;

    $input = [
        'photos' => [
            [
                'name' => 'BeachVacation.jpg',
                'description' => 'A photo of my beach vacation!',
            ],
            [
                'name' => 'GrandCanyon.jpg',
                'description' => '',
            ],
        ],
    ];

    Validator::validate($input, [
        'photos.*.description' => 'required',
    ], [
        'photos.*.description.required' => 'Please describe photo #:position.',
    ]);

上記の例で、バリデーションは失敗し、*"Please describe photo #2"*がユーザーに表示されます。

必要であれば、`second-index`、`second-position`、`third-index`、`third-position`などを使って、さらに深くネストしたインデックスやポジションを参照できます。

    'photos.*.attributes.*.string' => 'Invalid attribute for photo #:second-position.',

<a name="validating-files"></a>
## ファイルのバリデーション

Laravelでは、アップロードされたファイルを検証するため、`mimes`、`image`、`min`、`max`など様々なバリデーションルールを提供しています。ファイルのバリデーションを行う際に、これらのルールを個別に指定することも可能ですが、便利なようにファイルバリデーションルールビルダも提供しています。

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rules\File;

    Validator::validate($input, [
        'attachment' => [
            'required',
            File::types(['mp3', 'wav'])
                ->min(1024)
                ->max(12 * 1024),
        ],
    ]);

アプリケーションがユーザーからアップロードされた画像を受け取る場合、`File`ルールの`image`コンストラクタメソッドを使用して、アップロードされるファイルが画像であることを指定することができます。さらに、`dimensions`ルールを使用して、画像の大きさを制限することもできます。

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rule;
    use Illuminate\Validation\Rules\File;

    Validator::validate($input, [
        'photo' => [
            'required',
            File::image()
                ->min(1024)
                ->max(12 * 1024)
                ->dimensions(Rule::dimensions()->maxWidth(1000)->maxHeight(500)),
        ],
    ]);

> [!NOTE]
> 画像サイズのバリデーションに関するより詳しい情報は、[dimensionsルールのドキュメント](#rule-dimensions)に記載しています。

<a name="validating-files-file-sizes"></a>
#### ファイルサイズ

使いやすいように、最小ファイルサイズと最大ファイルサイズは、ファイルサイズの単位を示すサフィックス付きの文字列として指定できます。`kb`、`mb`、`gb`、`tb`のサフィックスをサポートしています。

```php
File::image()
    ->min('1kb')
    ->max('10mb')
```

<a name="validating-files-file-types"></a>
#### ファイルタイプ

`types`メソッドを呼び出すときには拡張子を指定するだけですが、このメソッドは実際にファイルの内容を読み込んで、MIMEタイプを推測し、バリデーションします。MIME タイプとそれに対応する拡張子の完全なリストは、次の場所にあります。

[https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types](https://svn.apache.org/repos/asf/httpd/httpd/trunk/docs/conf/mime.types)

<a name="validating-passwords"></a>
## パスワードのバリデーション

パスワードが十分なレベルの複雑さがあることを確認するために、Laravelの`Password`ルールオブジェクトを使用します。

    use Illuminate\Support\Facades\Validator;
    use Illuminate\Validation\Rules\Password;

    $validator = Validator::make($request->all(), [
        'password' => ['required', 'confirmed', Password::min(8)],
    ]);

`Password`ルールオブジェクトを使用すると、文字・数字・記号を最低１文字必須にしたり、文字種を組み合わせたりのように、パスワードの指定をアプリケーションで使用する複雑さの要件に合うよう簡単にカスタマイズできます。

    // 最低８文字必要
    Password::min(8)

    // 最低１文字の文字が必要
    Password::min(8)->letters()

    // 最低大文字小文字が１文字ずつ必要
    Password::min(8)->mixedCase()

    // 最低一文字の数字が必要
    Password::min(8)->numbers()

    // 最低一文字の記号が必要
    Password::min(8)->symbols()

さらに、`uncompromised`メソッドを使って、公開されているパスワードのデータ漏洩によるパスワード漏洩がないことを確認することもできます。

    Password::min(8)->uncompromised()

内部的には、`Password`ルールオブジェクトは、[k-Anonymity](https://en.wikipedia.org/wiki/K-anonymity)モデルを使用して、ユーザーのプライバシーやセキュリティを犠牲にすることなく、パスワードが[haveibeenpwned.com](https://haveibeenpwned.com)サービスを介してリークされているかを判断しています。

デフォルトでは、データリークに少なくとも1回パスワードが表示されている場合は、侵害されたと見なします。`uncompromised`メソッドの最初の引数を使用してこのしきい値をカスタマイズできます。

    // 同一のデータリークにおいて、パスワードの出現回数が3回以下であることを確認
    Password::min(8)->uncompromised(3);

もちろん、上記の例ですべてのメソッドをチェーン化することができます。

    Password::min(8)
        ->letters()
        ->mixedCase()
        ->numbers()
        ->symbols()
        ->uncompromised()

<a name="defining-default-password-rules"></a>
#### デフォルトパスワードルールの定義

パスワードのデフォルトバリデーションルールをアプリケーションの一箇所で指定できると便利でしょう。クロージャを引数に取る`Password::defaults`メソッドを使用すると、これを簡単に実現できます。`defaults`メソッドへ渡すクロージャは、パスワードルールのデフォルト設定を返す必要があります。通常、`defaults`ルールはアプリケーションのサービスプロバイダの1つの`boot`メソッド内で呼び出すべきです。

```php
use Illuminate\Validation\Rules\Password;

/**
 * アプリケーションの全サービスの初期処理
 */
public function boot(): void
{
    Password::defaults(function () {
        $rule = Password::min(8);

        return $this->app->isProduction()
                    ? $rule->mixedCase()->uncompromised()
                    : $rule;
    });
}
```

そして、バリデーションで特定のパスワードへデフォルトルールを適用したい場合に、引数なしで`defaults`メソッドを呼び出します。

    'password' => ['required', Password::defaults()],

時には、デフォルトのパスワードバリデーションルールへ追加ルールを加えたい場合があります。このような場合には、`rules`メソッドを使用します。

    use App\Rules\ZxcvbnRule;

    Password::defaults(function () {
        $rule = Password::min(8)->rules([new ZxcvbnRule]);

        // ...
    });

<a name="custom-validation-rules"></a>
## カスタムバリデーションルール

<a name="using-rule-objects"></a>
### ルールオブジェクトの使用

Laravelは有用な数多くのバリデーションルールを提供しています。ただし、独自のものを指定することもできます。カスタムバリデーションルールを登録する１つの方法は、ルールオブジェクトを使用することです。新しいルールオブジェクトを生成するには、`make:rule`Artisanコマンドを使用できます。このコマンドを使用して、文字列が大文字であることを確認するルールを生成してみましょう。Laravelは新しいルールを`app/Rules`ディレクトリに配置します。このディレクトリが存在しない場合、Artisanコマンドを実行してルールを作成すると、Laravelがそのディレクトリを作成します。

```shell
php artisan make:rule Uppercase
```

ルールを作成したら、次はその動作を定義します。ルールオブジェクトには、`validate`というメソッドが1つあります。このメソッドは、属性名とその値、バリデーションに失敗したときに、エラーメッセージとともに呼び出されるコールバックを受け取ります。

    <?php

    namespace App\Rules;

    use Closure;
    use Illuminate\Contracts\Validation\ValidationRule;

    class Uppercase implements ValidationRule
    {
        /**
         * バリデーションルールの実行
         */
        public function validate(string $attribute, mixed $value, Closure $fail): void
        {
            if (strtoupper($value) !== $value) {
                $fail('The :attribute must be uppercase.');
            }
        }
    }

ルールを定義したら、他のバリデーションルールと一緒に、ルールオブジェクトのインスタンスをバリデータへ渡し、指定します。

    use App\Rules\Uppercase;

    $request->validate([
        'name' => ['required', 'string', new Uppercase],
    ]);

#### バリデーションメッセージの翻訳

`$fail`クロージャへ文字列のエラーメッセージを指定する代わりに、[翻訳文字列キー](/docs/{{version}}/localization)を指定し、Laravelへエラーメッセージの翻訳を指示できます。

    if (strtoupper($value) !== $value) {
        $fail('validation.uppercase')->translate();
    }

必要に応じ、プレースホルダを置き換えたり、`translate`メソッドの第１引数と第２引数として使用する言語を指定することもできます。

    $fail('validation.location')->translate([
        'value' => $this->value,
    ], 'fr')

#### 追加データへのアクセス

カスタムバリデーションルールクラスがバリデーション下の他のすべてのデータへアクセスする必要がある場合、そのルールクラスに`Illuminate\Contracts\Validation\DataAwareRule`インターフェイスを実装してください。このインターフェイスは、クラスへ`setData` メソッドの定義を要求します。このメソッドはLaravelにより自動的に（バリデーション処理前に）、バリデーション対象の全データで呼び出されます。

    <?php

    namespace App\Rules;

    use Illuminate\Contracts\Validation\DataAwareRule;
    use Illuminate\Contracts\Validation\ValidationRule;

    class Uppercase implements DataAwareRule, ValidationRule
    {
        /**
         * バリデーション下の全データ
         *
         * @var array<string, mixed>
         */
        protected $data = [];

        // ...

        /**
         * バリデーション下のデータをセット
         *
         * @param  array<string, mixed>  $data
         */
        public function setData(array $data): static
        {
            $this->data = $data;

            return $this;
        }
    }

または、バリデーションルールが、バリデーションを行うバリデータインスタンスへのアクセスを必要とする場合は、`ValidatorAwareRule`インターフェイスを実装してください。

    <?php

    namespace App\Rules;

    use Illuminate\Contracts\Validation\ValidationRule;
    use Illuminate\Contracts\Validation\ValidatorAwareRule;
    use Illuminate\Validation\Validator;

    class Uppercase implements ValidationRule, ValidatorAwareRule
    {
        /**
         * バリデータインスタンス
         *
         * @var \Illuminate\Validation\Validator
         */
        protected $validator;

        // ...

        /**
         * 現用バリデータのセット
         */
        public function setValidator(Validator $validator): static
        {
            $this->validator = $validator;

            return $this;
        }
    }

<a name="using-closures"></a>
### クロージャの使用

アプリケーション全体でカスタムルールの機能が１回だけ必要な場合は、ルールオブジェクトの代わりにクロージャを使用できます。クロージャは、属性の名前、属性の値、およびバリデーションが失敗した場合に呼び出す必要がある`$fail`コールバックを受け取ります。

    use Illuminate\Support\Facades\Validator;
    use Closure;

    $validator = Validator::make($request->all(), [
        'title' => [
            'required',
            'max:255',
            function (string $attribute, mixed $value, Closure $fail) {
                if ($value === 'foo') {
                    $fail("The {$attribute} is invalid.");
                }
            },
        ],
    ]);

<a name="implicit-rules"></a>
### 暗黙のルール

デフォルトでは、バリデーションされる属性が存在しないか、空の文字列が含まれている場合、カスタムルールを含む通常のバリデーションルールは実行されません。たとえば、[`unique`](#rule-unique)ルールは空の文字列に対して実行されません。

    use Illuminate\Support\Facades\Validator;

    $rules = ['name' => 'unique:users,name'];

    $input = ['name' => ''];

    Validator::make($input, $rules)->passes(); // true

属性が空の場合でもカスタムルールを実行するには、ルールがその属性を必要とすることを示す必要があります。簡単に新しい暗黙のルールオブジェクトを生成するには、`make:rule` Artisanコマンドに、`--implicit`オプションを付けて使用します。

```shell
php artisan make:rule Uppercase --implicit
```

> [!WARNING]
> 「暗黙の」ルールは、属性が必要であることを*暗黙的に*します。欠落している属性または空の属性を実際に無効にするかどうかは、あなた次第です。
