# Bladeテンプレート

- [イントロダクション](#introduction)
    - [LivewireでBladeを強化する](#supercharging-blade-with-livewire)
- [データの表示](#displaying-data)
    - [HTMLエンティティエンコーディング](#html-entity-encoding)
    - [BladeとJavaScriptフレームワーク](#blade-and-javascript-frameworks)
- [Bladeディレクティブ](#blade-directives)
    - [If文](#if-statements)
    - [Switch文](#switch-statements)
    - [繰り返し](#loops)
    - [ループ変数](#the-loop-variable)
    - [条件クラス](#conditional-classes)
    - [その他の属性](#additional-attributes)
    - [サブビューの読み込み](#including-subviews)
    - [`@once`ディレクティブ](#the-once-directive)
    - [生PHP](#raw-php)
    - [コメント](#comments)
- [コンポーネント](#components)
    - [コンポーネントのレンダ](#rendering-components)
    - [コンポーネントへのデータ渡し](#passing-data-to-components)
    - [コンポーネント属性](#component-attributes)
    - [予約語](#reserved-keywords)
    - [スロット](#slots)
    - [インラインコンポーネントビュー](#inline-component-views)
    - [動的コンポーネント](#dynamic-components)
    - [コンポーネントの手作業登録](#manually-registering-components)
- [匿名コンポーネント](#anonymous-components)
    - [匿名インデックスコンポーネント](#anonymous-index-components)
    - [データプロパティ／属性](#data-properties-attributes)
    - [親データへのアクセス](#accessing-parent-data)
    - [匿名コンポーネントのパス](#anonymous-component-paths)
- [レイアウト構築](#building-layouts)
    - [コンポーネントを使用したレイアウト](#layouts-using-components)
    - [テンプレートの継承を使用したレイアウト](#layouts-using-template-inheritance)
- [フォーム](#forms)
    - [CSRFフィールド](#csrf-field)
    - [メソッドフィールド](#method-field)
    - [バリデーションエラー](#validation-errors)
- [スタック](#stacks)
- [サービス注入](#service-injection)
- [インラインBladeテンプレートのレンダ](#rendering-inline-blade-templates)
- [Bladeフラグメントのレンダ](#rendering-blade-fragments)
- [Bladeの拡張](#extending-blade)
    - [カスタムEchoハンドラ](#custom-echo-handlers)
    - [カスタムIf文](#custom-if-statements)

<a name="introduction"></a>
## イントロダクション

Bladeは、Laravelに含まれているシンプルでありながら強力なテンプレートエンジンです。一部のPHPテンプレートエンジンとは異なり、BladeはテンプレートでプレーンなPHPコードの使用を制限しません。実際、すべてのBladeテンプレートはプレーンなPHPコードにコンパイルされ、変更されるまでキャッシュされます。つまり、Bladeはアプリケーションに実質的にオーバーヘッドをかけません。Bladeテンプレートファイルは`.blade.php`ファイル拡張子を使用し、通常は`resources/views`ディレクトリに保存します。

Bladeビューは、グローバルな`view`ヘルパを使用してルートまたはコントローラから返します。もちろん、[views](/docs/{{version}}/views)のドキュメントに記載されているように、データを`view`ヘルパの２番目の引数を使用してBladeビューに渡せます。

    Route::get('/', function () {
        return view('greeting', ['name' => 'Finn']);
    });

<a name="supercharging-blade-with-livewire"></a>
### LivewireでBladeを強化する

Bladeテンプレートを次のレベルに引き上げ、ダイナミックなインターフェイスを簡単に構築したくありませんか？[Laravel Livewire](https://livewire.laravel.com)をチェックしてください。Livewireは、ReactやVueのようなフロントエンドフレームワークにのみ可能な動的機能で拡張したBladeコンポーネントを書けるようになります。多くのJavaScriptフレームワークの複雑さ、クライアントサイドレンダリング、構築ステップなしで、モダンなリアクティブフロントエンドを構築する素晴らしいアプローチです。

<a name="displaying-data"></a>
## データの表示

変数を中括弧で囲むことにより、Bladeビューに渡すデータを表示できます。たとえば、以下のルートがあるとします。

    Route::get('/', function () {
        return view('welcome', ['name' => 'Samantha']);
    });

次のように `name`変数の内容を表示できます。

```blade
Hello, {{ $name }}.
```

> [!NOTE]
> Bladeの`{{ }}`エコー文は、XSS攻撃を防ぐために、PHPの`htmlspecialchars`関数を通して自動的に送信します。

ビューに渡した変数の内容を表示するに留まりません。PHP関数の結果をエコーすることもできます。実際、Bladeエコーステートメント内に任意のPHPコードを入れることができます。

```blade
The current UNIX timestamp is {{ time() }}.
```

<a name="html-entity-encoding"></a>
### HTMLエンティティエンコーディング

デフォルトのBlade(およびLaravel`e`関数)はHTMLエンティティをダブルエンコードします。ダブルエンコーディングを無効にする場合は、`AppServiceProvider`の`boot`メソッドから`Blade::withoutDoubleEncoding`メソッドを呼び出します。

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\Blade;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * 全アプリケーションサービスの初期起動処理
         */
        public function boot(): void
        {
            Blade::withoutDoubleEncoding();
        }
    }

<a name="displaying-unescaped-data"></a>
#### エスケープしないデータの表示

デフォルトのBlade `{{ }}`文は、XSS攻撃を防ぐために、PHPの`htmlspecialchars`関数を通して自動的に送信します。データをエスケープしたくない場合は、次の構文を使用します。

```blade
Hello, {!! $name !!}.
```

> [!WARNING]
> アプリケーションのユーザーによって提供されるコンテンツをエコーするときは十分に注意してください。ユーザーが入力したデータを表示するときはXSS攻撃を防ぐために、通常はエスケープする二重中括弧構文を使用してください。

<a name="blade-and-javascript-frameworks"></a>
### BladeとJavaScriptフレームワーク

多くのJavaScriptフレームワークでも「中括弧」を使用して、特定の式をブラウザに表示することを示しているため、`@`記号を使用して式をそのままにしておくように、Bladeレンダリングエンジンへ通知できます。例を確認してください。

```blade
<h1>Laravel</h1>

Hello, @{{ name }}.
```

この例の`@`記号はBladeが削除します。そして、`{{ name }}`式はBladeエンジンによって変更されないので、JavaScriptフレームワークでレンダリングできます。

`@`記号は、Bladeディレクティブをエスケープするためにも使用できます。

```blade
{{-- Bladeテンプレート --}}
@@if()

<!-- HTML出力 -->
@if()
```

<a name="rendering-json"></a>
#### JSONのレンダ

JavaScript変数を初期化するために、配列をJSONとしてレンダリングする目的でビューに配列を渡す場合があります。一例を確認してください。

```blade
<script>
    var app = <?php echo json_encode($array); ?>;
</script>
```

自分で`json_encode`を呼び出す代わりに、`Illuminate\Support\Js::from`メソッドディレクティブが使えます。`from`メソッドは、PHPの`json_encode`関数と同じ引数を受け入れますが、取得結果のJSONはHTMLクオートの中へ含められるよう適切にエスケープされていることを保証します。`from`メソッドは、与えたオブジェクトや配列を有効なJavaScriptオブジェクトに変換する`JSON.parse` JavaScript文を文字列として返します。

```blade
<script>
    var app = {{ Illuminate\Support\Js::from($array) }};
</script>
```

最新バージョンのLaravelアプリケーション・スケルトンには、`Js`ファサードが含まれており、Bladeテンプレート内でこの機能に簡単にアクセスできるようになっています。

```blade
<script>
    var app = {{ Js::from($array) }};
</script>
```

> [!WARNING]
> 既存の変数をJSONとしてレンダするには、`Js::from`メソッドのみ使用してください。Bladeのテンプレートは正規表現に基づいているため、複雑な表現をディレクティブに渡そうとすると、予期せぬ失敗を引き起こす可能性があります。

<a name="the-at-verbatim-directive"></a>
#### `@verbatim`ディレクティブ

テンプレートの大部分でJavaScript変数を表示している場合は、HTMLを`@verbatim`ディレクティブでラップして、各Bladeエコーステートメントの前に`@`記号を付ける必要がないようにできます。

```blade
@verbatim
    <div class="container">
        Hello, {{ name }}.
    </div>
@endverbatim
```

<a name="blade-directives"></a>
## Bladeディレクティブ

テンプレートの継承とデータの表示に加えて、Bladeは条件ステートメントやループなど一般的なPHP制御構造への便利なショートカットも提供しています。こうしたショートカットは、PHP制御構造を操作するための非常にクリーンで簡潔な方法を提供する一方で、慣れ親しんだ同等のPHP構文も生かしています。

<a name="if-statements"></a>
### If文

`@if`、`@elseif`、`@else`、`@endif`ディレクティブを使用して`if`ステートメントを作成できます。これらのディレクティブは、対応するPHPの構文と同じように機能します。

```blade
@if (count($records) === 1)
    １レコードあります。
@elseif (count($records) > 1)
    複数レコードあります。
@else
    レコードがありません。
@endif
```

使いやすいように、Bladeは`@unless`ディレクティブも提供しています。

```blade
@unless (Auth::check())
    あなたはサインインしていません。
@endunless
```

すでに説明した条件付きディレクティブに加えて、`@isset`および`@empty`ディレクティブをそれぞれのPHP関数の便利なショートカットとして使用できます。

```blade
@isset($records)
    // $recordsが定義済みで、NULLではない…
@endisset

@empty($records)
    // $recordsは「空」だ…
@endempty
```

<a name="authentication-directives"></a>
#### 認証ディレクティブ

`@auth`および`@guest`ディレクティブを使用すると、現在のユーザーが[認証済み](/docs/{{version}}/authentication)であるか、ゲストであるかを簡潔に判断できます。

```blade
@auth
    // ユーザーは認証済み…
@endauth

@guest
    // ユーザーは認証されていない…
@endguest
```

必要に応じて、`@auth`および`@guest`ディレクティブを使用するときにチェックする必要がある認証ガードを指定できます。

```blade
@auth('admin')
    // ユーザーは認証済み…
@endauth

@guest('admin')
    // ユーザーは認証されていない…
@endguest
```

<a name="environment-directives"></a>
#### 環境ディレクティブ

`@production`ディレクティブを使用して、アプリケーションが本番環境で実行されているかを確認できます。

```blade
@production
    // Production限定コンテンツ…
@endproduction
```

または、`@env`ディレクティブを使用して、アプリケーションが特定の環境で実行されているかどうかを判断できます。

```blade
@env('staging')
    // アプリケーションは"staging"で動作している…
@endenv

@env(['staging', 'production'])
    // アプリケーションは"staging"か"production"で動作している…
@endenv
```

<a name="section-directives"></a>
#### セクションディレクティブ

`@hasSection`ディレクティブを使用して、テンプレート継承セクションにコンテンツがあるかどうかを判断できます。

```blade
@hasSection('navigation')
    <div class="pull-right">
        @yield('navigation')
    </div>

    <div class="clearfix"></div>
@endif
```

`sectionMissing`ディレクティブを使用して、セクションにコンテンツがないかどうかを判断できます。

```blade
@sectionMissing('navigation')
    <div class="pull-right">
        @include('default-navigation')
    </div>
@endif
```

<a name="session-directives"></a>
#### セッションディレクティブ

`@session`ディレクティブは、[セッション](/docs/{{version}}/session)の値が存在するかを判定するために使用します。セッションの値が存在すする場合、`@session`ディレクティブと`@endsession`ディレクティブ内のテンプレートの内容が評価されます。`@session`ディレクティブの内容の中で、セッションの値を表示するために、`$value`変数をechoできます。

```blade
@session('status')
    <div class="p-4 bg-green-100">
        {{ $value }}
    </div>
@endsession
```

<a name="switch-statements"></a>
### Switch文

Switchステートメントは、`@switch`、`@case`、`@break`、`@default`、`@endswitch`ディレクティブを使用して作成できます。

```blade
@switch($i)
    @case(1)
        最初のケース…
        @break

    @case(2)
        ２番めのケース…
        @break

    @default
        デフォルトのケース…
@endswitch
```

<a name="loops"></a>
### 繰り返し

条件文に加えて、BladeはPHPのループ構造を操作するための簡単なディレクティブを提供します。繰り返しますが、これらの各ディレクティブは、対応するPHPと同じように機能します。

```blade
@for ($i = 0; $i < 10; $i++)
    現在の値は、{{ $i }}
@endfor

@foreach ($users as $user)
    <p>このユーザーは：{{ $user->id }}</p>
@endforeach

@forelse ($users as $user)
    <li>{{ $user->name }}</li>
@empty
    <p>ユーザーはいません。</p>
@endforelse

@while (true)
    <p>無限ループ中です。</p>
@endwhile
```

> [!NOTE]
> `foreach`ループの反復処理中に、[ループ変数](#the-loop-variable) を使い、ループの最初の反復処理や最後の反復処理というような、反復に関する役立つ情報を取得可能です。

ループを使用する場合は、`@continue`および`@break`ディレクティブを使用して、現在の反復をスキップするか、ループを終了することもできます。

```blade
@foreach ($users as $user)
    @if ($user->type == 1)
        @continue
    @endif

    <li>{{ $user->name }}</li>

    @if ($user->number == 5)
        @break
    @endif
@endforeach
```

ディレクティブ宣言に継続条件または中断条件を含めることもできます。

```blade
@foreach ($users as $user)
    @continue($user->type == 1)

    <li>{{ $user->name }}</li>

    @break($user->number == 5)
@endforeach
```

<a name="the-loop-variable"></a>
### ループ変数

`foreach`ループの反復処理中、ループの内部では`$loop`変数を利用できます。この変数により、現在のループのインデックスや、ループの最初の反復なのか最後の反復なのか、といった便利な情報にアクセスすることができます。

```blade
@foreach ($users as $user)
    @if ($loop->first)
        これが最初の繰り返しです。
    @endif

    @if ($loop->last)
        これが最後の繰り返しです。
    @endif

    <p>このユーザーは：{{ $user->id }}</p>
@endforeach
```

ネストしたループ内にいる場合は、`parent`プロパティを介して親ループの`$loop`変数にアクセスできます。

```blade
@foreach ($users as $user)
    @foreach ($user->posts as $post)
        @if ($loop->parent->first)
            これは親ループの最初の繰り返しです。
        @endif
    @endforeach
@endforeach
```

`$loop`変数は、他にもいろいろと便利なプロパティを持っています。

| プロパティ         | 説明                                       |
| ------------------ | ------------------------------------------ |
| `$loop->index`     | 現在の反復のインデックス（初期値０）       |
| `$loop->iteration` | 現在の反復数（初期値１）。                 |
| `$loop->remaining` | 反復の残数。                               |
| `$loop->count`     | 反復している配列の総アイテム数             |
| `$loop->first`     | ループの最初の繰り返しか判定               |
| `$loop->last`      | ループの最後の繰り返しか判定               |
| `$loop->even`      | 今回が偶数回目の繰り返しか判定             |
| `$loop->odd`       | 今回が奇数回目の繰り返しか判定             |
| `$loop->depth`     | 現在のループのネストレベル                 |
| `$loop->parent`    | ループがネストしている場合、親のループ変数 |

<a name="conditional-classes"></a>
### 条件付きクラスとスタイル

`@class`ディレクティブは、CSSのクラス文字列を条件付きでコンパイルします。このディレクティブは、クラスの配列を受け取ります。配列のキーには、追加したいクラスが入り、値は論理値です。配列のキーが数字の場合は、レンダリングするクラスリストへ常に取り込みます。

```blade
@php
    $isActive = false;
    $hasError = true;
@endphp

<span @class([
    'p-4',
    'font-bold' => $isActive,
    'text-gray-500' => ! $isActive,
    'bg-red' => $hasError,
])></span>

<span class="p-4 text-gray-500 bg-red"></span>
```

同様に、`@style`ディレクティブは、条件付きでHTML要素へインラインのCSSスタイルを追加するために使用します。

```blade
@php
    $isActive = true;
@endphp

<span @style([
    'background-color: red',
    'font-weight: bold' => $isActive,
])></span>

<span style="background-color: red; font-weight: bold;"></span>
```

<a name="additional-attributes"></a>
### その他の属性

使いやすくするため、指定したHTMLのチェックボックス入力が"checked"であることを表す、`@checked`ディレクティブも使用できます。このディレクティブは、指定条件が`true`と評価された場合、`checked`をechoします。

```blade
<input type="checkbox"
        name="active"
        value="active"
        @checked(old('active', $user->active)) />
```

同様に、`@selected`ディレクティブは、特定のセレクトオプションが"selected"であることを表すために使用します。

```blade
<select name="version">
    @foreach ($product->versions as $version)
        <option value="{{ $version }}" @selected(old('version') == $version)>
            {{ $version }}
        </option>
    @endforeach
</select>
```

さらに、`@disabled`ディレクティブは、指定要素が"disabled"であることを表すために使用します。

```blade
<button type="submit" @disabled($errors->isNotEmpty())>Submit</button>
```

さらに、`@readonly` ディレクティブは、指定した要素が"readonly"であるべきかを示すために使用します。

```blade
<input type="email"
        name="email"
        value="email@laravel.com"
        @readonly($user->isNotAdmin()) />
```

加えて、`@required`ディレクティブは、指定要素が"required"であることを表すために使用します。

```blade
<input type="text"
        name="title"
        value="title"
        @required($user->isAdmin()) />
```

<a name="including-subviews"></a>
### サブビューの読み込み

> [!NOTE]
> `@include`ディレクティブは自由に使用できますが、Blade[コンポーネント](#components)は同様の機能を提供しつつ、データや属性のバインドなど`@include`ディレクティブに比べていくつかの利点があります。

Bladeの`@include`ディレクティブを使用すると、別のビュー内からBladeビューを読み込めます。親ビューで使用できるすべての変数は、読み込むビューで使用できます。

```blade
<div>
    @include('shared.errors')

    <form>
        <!-- フォームのコンテンツ -->
    </form>
</div>
```

読み込むビューは親ビューで使用可能なすべてのデータを継承しますが、読み込むビューで使用可能にする必要がある追加データの配列を渡すこともできます。

```blade
@include('view.name', ['status' => 'complete'])
```

存在しないビューを`@include`しようとすると、Laravelはエラーを投げます。存在する場合と存在しない場合があるビューを読み込む場合は、`@includeIf`ディレクティブを使用する必要があります。

```blade
@includeIf('view.name', ['status' => 'complete'])
```

指定する論理式が`true`か`false`と評価された場合に、ビューを`@include`したい場合は、`@includeWhen`および`@includeUnless`ディレクティブを使用できます。

```blade
@includeWhen($boolean, 'view.name', ['status' => 'complete'])

@includeUnless($boolean, 'view.name', ['status' => 'complete'])
```

指定するビュー配列中、存在する最初のビューを含めるには、`includeFirst`ディレクティブを使用します。

```blade
@includeFirst(['custom.admin', 'admin'], ['status' => 'complete'])
```

> [!WARNING]
> Bladeビューで`__DIR__`と`__FILE__`定数は使用しないでください。これらは、キャッシュされコンパイルされたビューの場所を参照するためです。

<a name="rendering-views-for-collections"></a>
#### Rendering Views for Collections

ループと読み込みをBladeの`@each`ディレクティブで1行に組み合わせられます。

```blade
@each('view.name', $jobs, 'job')
```

`@each`ディレクティブの最初の引数は、配列またはコレクション内の各要素に対してレンダするビューです。２番目の引数は、反復する配列またはコレクションであり、３番目の引数は、ビュー内で現在の反復要素に割り当てる変数名です。したがって、たとえば`jobs`配列を反復処理する場合、通常はビュー内で`job`変数として各ジョブにアクセスしたいと思います。現在の反復の配列キーは、ビュー内で`key`変数として使用できます。

`@each`ディレクティブに４番目の引数を渡すこともできます。この引数は、指定された配列が空の場合にレンダするビューを指定します。

```blade
@each('view.name', $jobs, 'job', 'view.empty')
```

> [!WARNING]
> `@each`を使いレンダするビューは、親ビューから変数を継承しません。子ビューでそうした変数が必要な場合は、代わりに`@foreach`と`@include`ディレクティブを使用する必要があります。

<a name="the-once-directive"></a>
### `@once`ディレクティブ

`@once`ディレクティブを使用すると、レンダリングサイクルごとに１回だけ評価されるテンプレートの部分を定義できます。これは、[stacks](#stacks)を使用してJavaScriptの特定の部分をページのヘッダへ入れ込むのに役立つでしょう。たとえば、ループ内で特定の[コンポーネント](#components)をレンダする場合に、コンポーネントが最初にレンダされるときにのみ、あるJavaScriptをヘッダに入れ込みたい場合があるとしましょう。

```blade
@once
    @push('scripts')
        <script>
            // カスタムJavaScript…
        </script>
    @endpush
@endonce
```

`@once`ディレクティブは、`@push`や`@prepend`ディレクティブと一緒に使われることが多いため、便利なように`@pushOnce`と`@prependOnce`ディレクティブを用意しました。

```blade
@pushOnce('scripts')
    <script>
        // カスタムJavaScript…
    </script>
@endPushOnce
```

<a name="raw-php"></a>
### 生PHP

状況によっては、PHPコードをビューに埋め込めると便利です。Blade`@php`ディレクティブを使用して、テンプレート内でプレーンPHPのブロックを定義し、実行できます。

```blade
@php
    $counter = 1;
@endphp
```

もしくは、クラスをインポートするためだけにPHPを使用する場合は、`@use`ディレクティブを使用してください。

```blade
@use('App\Models\Flight')
```

インポートしたクラスのエイリアスを指定するために、`@use`ディレクティブに第２引数を指定できます。

```php
@use('App\Models\Flight', 'FlightModel')
```

<a name="comments"></a>
### コメント

また、Bladeでは、ビューにコメントを定義することができます。ただし、HTMLのコメントとは異なり、Bladeのコメントは、アプリケーションが返すHTMLには含まれません。

```blade
{{-- このコメントはHTMLのなかに存在しない --}}
```

<a name="components"></a>
## コンポーネント

コンポーネントとスロットは、セクション、レイアウト、インクルードと同様の利便性を提供します。ただし、コンポーネントとスロットのメンタルモデルが理解しやすいと感じる人もいるでしょう。コンポーネントを作成するには、クラスベースのコンポーネントと匿名コンポーネントの2つのアプローチがあります。

クラスベースのコンポーネントを作成するには、`make:component` Artisanコマンドを使用できます。コンポーネントの使用方法を説明するために、単純な「アラート（`Alert`）」コンポーネントを作成します。`make:component`コマンドは、コンポーネントを`app/View/Components`ディレクトリに配置します。

```shell
php artisan make:component Alert
```

`make:component`コマンドは、コンポーネントのビューテンプレートも作成します。ビューは`resources/views/components`ディレクトリに配置されます。独自のアプリケーション用のコンポーネントを作成する場合、コンポーネントは`app/View/Components`ディレクトリと`resources/views/components`ディレクトリ内で自動的に検出するため、通常それ以上のコンポーネント登録は必要ありません。

サブディレクトリ内にコンポーネントを作成することもできます。

```shell
php artisan make:component Forms/Input
```

上記のコマンドは、`app/View/Components/Forms`ディレクトリに`Input`コンポーネントを作成し、ビューは`resources/views/components/forms`ディレクトリに配置します。

匿名コンポーネント(Bladeテンプレートのみでクラスを持たないコンポーネント)を作成したい場合は、`make:component`コマンドを実行するとき、`--view`フラグを使用します。

```shell
php artisan make:component forms.input --view
```

上記のコマンドは、`resources/views/components/forms/input.blade.php`へBladeファイルを作成します。このファイルは`<x-forms.input />`により、コンポーネントとしてレンダできます。

<a name="manually-registering-package-components"></a>
#### パッケージコンポーネントの手作業登録

独自のアプリケーション用コンポーネントを作成する場合、コンポーネントを`app/View/Components`ディレクトリと`resources/views/components`ディレクトリ内で自動的に検出します。

ただし、Bladeコンポーネントを利用するパッケージを構築する場合は、コンポーネントクラスとそのHTMLタグエイリアスを手作業で登録する必要があります。通常、コンポーネントはパッケージのサービスプロバイダの`boot`メソッドに登録する必要があります。

    use Illuminate\Support\Facades\Blade;

    /**
     * パッケージの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Blade::component('package-alert', Alert::class);
    }

コンポーネントを登録すると、タグエイリアスを使用してレンダリングできます。

```blade
<x-package-alert/>
```

または、規約により、`componentNamespace`メソッドを使用してコンポーネントクラスを自動ロードすることもできます。たとえば、`Nightshade`パッケージには、`Package\Views\Components`名前空間内にある`Calendar`と`ColorPicker`コンポーネントが含まれているとしましょう。

    use Illuminate\Support\Facades\Blade;

    /**
     * パッケージの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Blade::componentNamespace('Nightshade\Views\Components', 'nightshade');
    }

これにより、`package-name::`構文を使用して、ベンダーの名前空間でパッケージコンポーネントが使用できるようになります。

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Bladeは、コンポーネント名のパスカルケースを使い、コンポーネントにリンクしているクラスを自動的に検出します。サブディレクトリもサポートしており、「ドット」表記を使用します。

<a name="rendering-components"></a>
### コンポーネントのレンダ

コンポーネントを表示するために、Bladeテンプレート１つの中でBladeコンポーネントタグを使用できます。Bladeコンポーネントタグは、文字列`x-`で始まり、その後にコンポーネントクラスのケバブケース名を続けます。

```blade
<x-alert/>

<x-user-profile/>
```

コンポーネントクラスが`app/View/Components`ディレクトリ内のより深い場所にネストしている場合は、`.`文字を使用してディレクトリのネストを表せます。たとえば、コンポーネントが`app/View/Components/Inputs/Button.php`にあるとしたら、以下のようにレンダリングします。

```blade
<x-inputs.button/>
```

もし、コンポーネントを条件付きでレンダしたい場合は、コンポーネントクラスで`shouldRender`メソッドを定義してください。`shouldRender`メソッドが`false`を返した場合、コンポーネントをレンダしません。

    use Illuminate\Support\Str;

    /**
     * コンポーネントをレンダするか
     */
    public function shouldRender(): bool
    {
        return Str::length($this->message) > 0;
    }

<a name="passing-data-to-components"></a>
### コンポーネントへのデータ渡し

HTML属性を使用してBladeコンポーネントへデータを渡せます。ハードコードするプリミティブ値は、単純なHTML属性文字列を使用してコンポーネントに渡すことができます。PHPの式と変数は、接頭辞として`:`文字を使用する属性を介してコンポーネントに渡す必要があります。

```blade
<x-alert type="error" :message="$message"/>
```

コンポーネントの全データ属性は、そのクラスコンストラクターで定義する必要があります。コンポーネントのすべてのパブリックプロパティは、コンポーネントのビューで自動的に使用可能になります。コンポーネントの`render`メソッドからビューにデータを渡す必要はありません。

    <?php

    namespace App\View\Components;

    use Illuminate\View\Component;
    use Illuminate\View\View;

    class Alert extends Component
    {
        /**
         * コンポーネントインスタンスを作成
         */
        public function __construct(
            public string $type,
            public string $message,
        ) {}

        /**
         * コンポーネントを表すビュー／コンテンツを取得
         */
        public function render(): View
        {
            return view('components.alert');
        }
    }

コンポーネントをレンダするときに、変数を名前でエコーすることにより、コンポーネントのパブリック変数の内容を表示できます。

```blade
<div class="alert alert-{{ $type }}">
    {{ $message }}
</div>
```

<a name="casing"></a>
#### ケース形式

コンポーネントコンストラクタの引数はキャメルケース（`camelCase`）を使用して指定する必要がありますが、HTML属性で引数名を参照する場合はケバブケース（`kebab-case`）を使用する必要があります。たとえば、次のようにコンポーネントコンストラクタに渡します。

    /**
     * コンポーネントインスタンスの作成
     */
    public function __construct(
        public string $alertType,
    ) {}

`$alertType`引数は、次のようにコンポーネントに提供できます。

```blade
<x-alert alert-type="danger" />
```

<a name="short-attribute-syntax"></a>
#### 属性の短縮構文

コンポーネントに属性を渡す場合、「短縮」構文も使用できます。属性名と対応する変数名が一致することが多いため、これは便利な方法です。

```blade
{{-- 短縮形 --}}
<x-profile :$userId :$name />

{{-- 同じ動作をする記法 --}}
<x-profile :user-id="$userId" :name="$name" />
```

<a name="escaping-attribute-rendering"></a>
#### 属性レンダのエスケープ

alpine.jsなどのJavaScriptフレームワークもコロンプレフィックスで属性を指定するため、属性がPHP式ではないことをBladeへ知らせるために二重のコロン(`::`)プレフィックスを使用してください。たとえば、以下のコンポーネントが存在しているとしましょう。

```blade
<x-button ::class="{ danger: isDeleting }">
    Submit
</x-button>
```

Bladeにより、以下のHTMLとしてレンダされます。

```blade
<button :class="{ danger: isDeleting }">
    Submit
</button>
```

<a name="component-methods"></a>
#### コンポーネントメソッド

コンポーネントテンプレートで使用できるパブリック変数に加えて、コンポーネントの任意のパブリックメソッドを呼び出すこともできます。たとえば、`isSelected`メソッドを持つコンポーネントを想像してみてください。

    /**
     * 指定したオプションが現在選択されているオプションであるか判別
     */
    public function isSelected(string $option): bool
    {
        return $option === $this->selected;
    }

メソッドの名前に一致する変数を呼び出すことにより、コンポーネントテンプレートでこのメソッドを実行できます。

```blade
<option {{ $isSelected($value) ? 'selected' : '' }} value="{{ $value }}">
    {{ $label }}
</option>
```

<a name="using-attributes-slots-within-component-class"></a>
#### コンポーネントクラス内の属性とスロットへのアクセス

Bladeコンポーネントを使用すると、クラスのrenderメソッド内のコンポーネント名、属性、およびスロットにアクセスすることもできます。ただし、このデータにアクセスするには、コンポーネントの`render`メソッドからクロージャを返す必要があります。クロージャは、唯一の引数として`$data`配列を取ります。この配列はコンポーネントに関する情報を提供するいくつかの要素が含んでいます。

    use Closure;

    /**
     * コンポーネントを表すビュー／コンテンツを取得
     */
    public function render(): Closure
    {
        return function (array $data) {
            // $data['componentName'];
            // $data['attributes'];
            // $data['slot'];

            return '<div>Components content</div>';
        };
    }

`componentName`は、HTMLタグで`x-`プレフィックスの後に使用されている名前と同じです。したがって、`<x-alert/>`の`componentName`は`alert`になります。`attributes`要素には、HTMLタグに存在していたすべての属性が含まれます。`slot`要素は、コンポーネントのスロットの内容を含む`Illuminate\Support\HtmlString`インスタンスです。

クロージャは文字列を返す必要があります。返された文字列が既存のビューに対応している場合、そのビューがレンダリングされます。それ以外の場合、返される文字列はインラインBladeビューとして評価されます。

<a name="additional-dependencies"></a>
#### 追加の依存関係

コンポーネントがLaravelの[サービスコンテナ](/docs/{{version}}/container)からの依存を必要とする場合、コンポーネントのデータ属性の前にそれらをリストすると、コンテナによって自動的に注入されます。

```php
use App\Services\AlertCreator;

/**
 * Create the component instance.
 */
public function __construct(
    public AlertCreator $creator,
    public string $type,
    public string $message,
) {}
```

<a name="hiding-attributes-and-methods"></a>
#### 非表示属性/メソッド

パブリックメソッドまたはプロパティがコンポーネントテンプレートへ変数として公開されないようにする場合は、コンポーネントの`$except`配列プロパティへ追加してください。

    <?php

    namespace App\View\Components;

    use Illuminate\View\Component;

    class Alert extends Component
    {
        /**
         * コンポーネントテンプレートに公開してはいけないプロパティ／メソッド。
         *
         * @var array
         */
        protected $except = ['type'];

        /**
         * コンポーネントインスタンスを作成
         */
        public function __construct(
            public string $type,
        ) {}
    }

<a name="component-attributes"></a>
### コンポーネント属性

データ属性をコンポーネントへ渡す方法は、すでに説明しました。ただ、コンポーネントが機能するためには、不必要な`class`などの追加のHTML属性データを指定する必要が起きることもあります。通常これらの追加の属性は、コンポーネントテンプレートのルート要素に渡します。たとえば、次のように`alert`コンポーネントをレンダリングしたいとします。

```blade
<x-alert type="error" :message="$message" class="mt-4"/>
```

コンポーネントのコンストラクタの一部ではないすべての属性は、コンポーネントの「属性バッグ」に自動的に追加されます。この属性バッグは、`$attributes`変数を通じてコンポーネントで自動的に使用可能になります。この変数をエコーすることにより、すべての属性をコンポーネント内でレンダリングできます。

```blade
<div {{ $attributes }}>
    <!-- Component content -->
</div>
```

> [!WARNING]
> コンポーネントタグ内での`@env`などのディレクティブの使用は、現時点でサポートしていません。たとえば、`<x-alert :live="@env('production')"/>`はコンパイルしません。

<a name="default-merged-attributes"></a>
#### デフォルト/マージした属性

属性のデフォルト値を指定したり、コンポーネントの属性の一部へ追加の値をマージしたりする必要のある場合があります。これを実現するには、属性バッグの`merge`メソッドを使用できます。このメソッドは、コンポーネントに常に適用する必要のあるデフォルトのCSSクラスのセットを定義する場合、特に便利です。

```blade
<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

このコンポーネントが以下のように使用されると仮定しましょう。

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

コンポーネントが最終的にレンダするHTMLは、以下のようになります。

```blade
<div class="alert alert-error mb-4">
    <!-- $message変数の中身 -->
</div>
```

<a name="conditionally-merge-classes"></a>
#### 条件付きマージクラス

特定の条件が`true`である場合に、クラスをマージしたい場合があります。これは`class`メソッドで実行でき、クラスの配列を引数に取ります。配列のキーには追加したいクラスを含み、値に論理式を指定します。配列要素に数字キーがある場合は、レンダするクラスリストへ常に含まれます。

```blade
<div {{ $attributes->class(['p-4', 'bg-red' => $hasError]) }}>
    {{ $message }}
</div>
```

他の属性をコンポーネントにマージする必要がある場合は、`merge`メソッドを`class`メソッドへチェーンできます。

```blade
<button {{ $attributes->class(['p-4'])->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

> [!NOTE]
> マージした属性を受け取るべきではない他のHTML要素にクラスを条件付きでコンパイルする必要がある場合は、[`@class`ディレクティブ](#consitional-classes)を使用できます。

<a name="non-class-attribute-merging"></a>
#### 非クラス属性のマージ

`class`属性ではない属性をマージする場合、`merge`メソッドへ指定する値は属性の「デフォルト」値と見なします。ただし、`class`属性とは異なり、こうした属性は挿入した属性値とマージしません。代わりに上書きします。たとえば、`button`コンポーネントの実装は次のようになるでしょう。

```blade
<button {{ $attributes->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

カスタム`type`を指定してボタンコンポーネントをレンダすると、このコンポーネントを使用するときにそれが指定されます。タイプが指定されない場合、`button`タイプを使用します。

```blade
<x-button type="submit">
    Submit
</x-button>
```

この例でレンダリングされる、`button`コンポーネントのHTMLは以下のようになります。

```blade
<button type="submit">
    Submit
</button>
```

`class`以外の属性にデフォルト値と挿入する値を結合させたい場合は、`prepends`メソッドを使用します。この例では、`data-controller`属性は常に`profile-controller`で始まり、追加で挿入する`data-controller`値はデフォルト値の後に配置されます。

```blade
<div {{ $attributes->merge(['data-controller' => $attributes->prepends('profile-controller')]) }}>
    {{ $slot }}
</div>
```

<a name="filtering-attributes"></a>
#### 属性の取得とフィルタリング

`filter`メソッドを使用して属性をフィルタリングできます。このメソッドは、属性を属性バッグへ保持する場合に`true`を返す必要があるクロージャを引数に取ります。

```blade
{{ $attributes->filter(fn (string $value, string $key) => $key == 'foo') }}
```

キーが特定の文字列で始まるすべての属性を取得するために、`whereStartsWith`メソッドを使用できます。

```blade
{{ $attributes->whereStartsWith('wire:model') }}
```

逆に、`whereDoesntStartWith`メソッドは、指定する文字列で始まるすべての属性を除外するために使用します。

```blade
{{ $attributes->whereDoesntStartWith('wire:model') }}
```

`first`メソッドを使用して、特定の属性バッグの最初の属性をレンダできます。

```blade
{{ $attributes->whereStartsWith('wire:model')->first() }}
```

属性がコンポーネントに存在するか確認する場合は、`has`メソッドを使用できます。このメソッドは、属性名をその唯一の引数に取り、属性が存在していることを示す論理値を返します。

```blade
@if ($attributes->has('class'))
    <div>Class attribute is present</div>
@endif
```

配列を`has`メソッドへ渡した場合、このメソッドは与えた属性がすべてコンポーネントに存在するかを判定します。

```blade
@if ($attributes->has(['name', 'class']))
    <div>All of the attributes are present</div>
@endif
```

`hasAny`メソッドは、指定属性のいずれかがコンポーネントに存在するかを判定するために使用します。

```blade
@if ($attributes->hasAny(['href', ':href', 'v-bind:href']))
    <div>One of the attributes is present</div>
@endif
```

`get`メソッドを使用して特定の属性の値を取得できます。

```blade
{{ $attributes->get('class') }}
```

<a name="reserved-keywords"></a>
### 予約語

コンポーネントをレンダするBladeの内部使用のため、いくつかのキーワードを予約しています。以下のキーワードは、コンポーネント内のパブリックプロパティまたはメソッド名として定できません。

<div class="content-list" markdown="1">

- `data`
- `render`
- `resolveView`
- `shouldRender`
- `view`
- `withAttributes`
- `withName`

</div>

<a name="slots"></a>
### スロット

多くの場合、「スロット」を利用して追加のコンテンツをコンポーネントに渡す必要があることでしょう。コンポーネントスロットは、`$slot`変数をエコーしレンダします。この概念を学習するために、`alert`コンポーネントに次のマークアップがあると過程してみましょう。

```blade
<!-- /resources/views/components/alert.blade.php -->

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

コンポーネントにコンテンツを挿入することにより、そのコンテンツを`slot`に渡せます。

```blade
<x-alert>
    <strong>おっと！</strong> なにかおかしいようです！
</x-alert>
```

コンポーネント内の異なる場所へ、複数の異なるスロットをレンダする必要のある場合があります。「タイトル（"title"）」スロットへ挿入できるようにするため、アラートコンポーネントを変更してみましょう。

```blade
<!-- /resources/views/components/alert.blade.php -->

<span class="alert-title">{{ $title }}</span>

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

`x-slot`タグを使用して、名前付きスロットのコンテンツを定義できます。明示的に`x-slot`タグ内にないコンテンツは、`$slot`変数のコンポーネントに渡されます。

```xml
<x-alert>
    <x-slot:title>
        Server Error
    </x-slot>

    <strong>おっと！</strong> なにかおかしいようです！
</x-alert>
```

そのスロットにコンテンツが含まれているかを判断するには、`isEmpty`メソッドを呼び出します。

```blade
<span class="alert-title">{{ $title }}</span>

<div class="alert alert-danger">
    @if ($slot->isEmpty())
        ここはスロットが空の場合のデフォルトコンテンツ.
    @else
        {{ $slot }}
    @endif
</div>
```

さらに、`hasActualContent`メソッドを使用して、スロットにHTMLコメントではない「本当」のコンテンツが含まれているかを判定できます。

```blade
@if ($slot->hasActualContent())
    コメントではないコンテンツを持つスコープ。
@endif
```

<a name="scoped-slots"></a>
#### スコープ付きスロット

VueのようなJavaScriptフレームワークを使用している方は「スコープ付きスロット」に慣れていると思います。これは、スロットの中でコンポーネントのデータやメソッドへアクセスできる機構です。コンポーネントにパブリックメソッドまたはプロパティを定義し、`$component`変数を使いスロット内のコンポーネントにアクセスすることで、Laravelと同様の動作を実現できます。この例では、`x-alert`コンポーネントのコンポーネントクラスにパブリック`formatAlert`メソッドが定義されていると想定しています。

```blade
<x-alert>
    <x-slot:title>
        {{ $component->formatAlert('Server Error') }}
    </x-slot>

    <strong>おっと！</strong> なにかおかしいようです！
</x-alert>
```

<a name="slot-attributes"></a>
#### スロット属性

ブレードコンポーネントと同様に、CSSクラス名などのスロットに追加の[属性](#component-attributes)を割り当てることができます。

```xml
<x-card class="shadow-sm">
    <x-slot:heading class="font-bold">
        Heading
    </x-slot>

    Content

    <x-slot:footer class="text-sm">
        Footer
    </x-slot>
</x-card>
```

スロット属性を操作するため、スロットの変数の`attributes`プロパティへアクセスできます。属性を操作する方法の詳細は、[コンポーネント属性](#component-attributes)のドキュメントを参照してください。

```blade
@props([
    'heading',
    'footer',
])

<div {{ $attributes->class(['border']) }}>
    <h1 {{ $heading->attributes->class(['text-lg']) }}>
        {{ $heading }}
    </h1>

    {{ $slot }}

    <footer {{ $footer->attributes->class(['text-gray-700']) }}>
        {{ $footer }}
    </footer>
</div>
```

<a name="inline-component-views"></a>
### インラインコンポーネントビュー

非常に小さいコンポーネントの場合、コンポーネントクラスとビューテンプレートの両方を管理するのは面倒だと感じるかもしれません。そのため、コンポーネントのマークアップを`render`メソッドから直接返すことができます。

    /**
     * コンポーネントを表すビュー／コンテンツを取得
     */
    public function render(): string
    {
        return <<<'blade'
            <div class="alert alert-danger">
                {{ $slot }}
            </div>
        blade;
    }

<a name="generating-inline-view-components"></a>
#### インラインビューコンポーネントの生成

インラインビューをレンダするコンポーネントを作成するには、`make:component`コマンドを実行するときに`inline`オプションを使用します。

```shell
php artisan make:component Alert --inline
```

<a name="dynamic-components"></a>
### 動的コンポーネント

時には、あるコンポーネントをレンダする必要があっても、実行時までどのコンポーネントをレンダすべきか分からないことがあります。この場合、Laravelの組み込みコンポーネントである`dynamic-component`を使用すると、実行時の値や変数に基づいてコンポーネントをレンダできます。

```blade
// $componentName = "secondary-button";

<x-dynamic-component :component="$componentName" class="mt-4" />
```

<a name="manually-registering-components"></a>
### コンポーネントの手作業登録

> [!WARNING]
> コンポーネントの手作業登録に関する以下のドキュメントは、主にビューコンポーネントを含むLaravelのパッケージを作成している開発者に当てはまるものです。こうしたパッケージを書いていない場合は、コンポーネントに関する以下のドキュメントは、あなたに関係しないでしょう。

独自のアプリケーション用コンポーネントを作成する場合、コンポーネントを`app/View/Components`ディレクトリと`resources/views/components`ディレクトリ内で自動的に検出します。

しかし、Bladeコンポーネントを利用するパッケージを構築する場合や、コンポーネントをデフォルト外のディレクトリに配置する場合は、コンポーネントクラスとそのHTMLタグエイリアスを手作業で登録し、Laravelがコンポーネントを探す場所を認識できるようにする必要があります。通常、パッケージのサービスプロバイダの`boot`メソッドでコンポーネントを登録する必要があります。

    use Illuminate\Support\Facades\Blade;
    use VendorPackage\View\Components\AlertComponent;

    /**
     * パッケージの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Blade::component('package-alert', AlertComponent::class);
    }

コンポーネントを登録すると、タグエイリアスを使用してレンダリングできます。

```blade
<x-package-alert/>
```

#### パッケージコンポーネントの自動ロード

または、規約により、`componentNamespace`メソッドを使用してコンポーネントクラスを自動ロードすることもできます。たとえば、`Nightshade`パッケージには、`Package\Views\Components`名前空間内にある`Calendar`と`ColorPicker`コンポーネントが含まれているとしましょう。

    use Illuminate\Support\Facades\Blade;

    /**
     * パッケージの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Blade::componentNamespace('Nightshade\Views\Components', 'nightshade');
    }

これにより、`package-name::`構文を使用して、ベンダーの名前空間でパッケージコンポーネントが使用できるようになります。

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Bladeは、コンポーネント名のパスカルケースを使い、コンポーネントにリンクしているクラスを自動的に検出します。サブディレクトリもサポートしており、「ドット」表記を使用します。

<a name="anonymous-components"></a>
## 匿名コンポーネント

インラインコンポーネントと同様に、匿名コンポーネントは、単一のファイルを介してコンポーネントを管理するためのメカニズムを提供します。ただし、匿名コンポーネントは単一のビューファイルを利用し、関連するクラスはありません。匿名コンポーネントを定義するには、`resources/views/components`ディレクトリ内にBladeテンプレートを配置するだけで済みます。たとえば、`resources/views/components/alert.blade.php`でコンポーネントを定義すると、以下のようにレンダできます。

```blade
<x-alert/>
```

`.`文字を使用して、コンポーネントが`components`ディレクトリ内のより深い場所にネストしていることを示せます。たとえば、コンポーネントが`resources/views/components/input/button.blade.php`で定義されていると仮定すると、次のようにレンダリングできます。

```blade
<x-inputs.button/>
```

<a name="anonymous-index-components"></a>
### 匿名インデックスコンポーネント

あるコンポーネントが多数のBladeテンプレートから構成されている場合、そのコンポーネントのテンプレートを１つのディレクトリにまとめたいことがあります。例えば、以下のようなディレクトリ構造を持つ「アコーディオン」コンポーネントがあるとします。

```none
/resources/views/components/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

このディレクトリ構造を使用すると、アコーディオン・コンポーネントとそのアイテムを以下のようにレンダできます。

```blade
<x-accordion>
    <x-accordion.item>
        ...
    </x-accordion.item>
</x-accordion>
```

しかし、`x-accordion`でアコーディオンコンポーネントをレンダするには、他のアコーディオン関連のテンプレートと一緒に`accordion`ディレクトリ内に入れ子にするのではなく、"index"アコーディオン・コンポーネントテンプレートを`resources/views/components`ディレクトリ内に配置する必要がありました。

Bladeでは幸い、コンポーネントのテンプレートディレクトリ内に `index.blade.php` ファイルを配置することができます。index.blade.php`のテンプレートがコンポーネントに存在する場合、そのテンプレートはコンポーネントの「ルート」ノードとしてレンダされます。そこで、上記の例で示したのと同じBladeの構文を引き続き使用することができますが、ディレクトリ構造を次のように調整します。

```none
/resources/views/components/accordion/index.blade.php
/resources/views/components/accordion/item.blade.php
```

<a name="data-properties-attributes"></a>
### データプロパティ／属性

匿名コンポーネントには関連付けたクラスがないため、どのデータを変数としてコンポーネントに渡す必要があり、どの属性をコンポーネントの[属性バッグ](#component-attributes)に配置する必要があるか、どう区別するか疑問に思われるかもしれません。

コンポーネントのBladeテンプレートの上部にある`@props`ディレクティブを使用して、データ変数と見なすべき属性を指定できます。コンポーネント上の他のすべての属性は、コンポーネントの属性バッグを介して利用します。データ変数にデフォルト値を指定する場合は、変数の名前を配列キーに指定し、デフォルト値を配列値として指定します。

```blade
<!-- /resources/views/components/alert.blade.php -->

@props(['type' => 'info', 'message'])

<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

上記のコンポーネント定義をすると、そのコンポーネントは次のようにレンダできます。

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

<a name="accessing-parent-data"></a>
### 親データへのアクセス

子コンポーネントの中にある親コンポーネントのデータにアクセスしたい場合があります。このような場合は、`@aware`ディレクティブを使用します。例えば、親の`<x-menu>`と子の`<x-menu.item>`で構成される複雑なメニューコンポーネントを作っていると想像してください。

```blade
<x-menu color="purple">
    <x-menu.item>...</x-menu.item>
    <x-menu.item>...</x-menu.item>
</x-menu>
```

`<x-menu>`コンポーネントは、以下のような実装になるでしょう。

```blade
<!-- /resources/views/components/menu/index.blade.php -->

@props(['color' => 'gray'])

<ul {{ $attributes->merge(['class' => 'bg-'.$color.'-200']) }}>
    {{ $slot }}
</ul>
```

`color`プロップは親(`<x-menu>`)にしか渡されていないため、`<x-menu.item>`の中では利用できません。しかし、`@aware`ディレクティブを使用すれば、`<x-menu.item>`内でも利用可能になります。

```blade
<!-- /resources/views/components/menu/item.blade.php -->

@aware(['color' => 'gray'])

<li {{ $attributes->merge(['class' => 'text-'.$color.'-800']) }}>
    {{ $slot }}
</li>
```

> [!WARNING]
> `@aware`ディレクティブは、HTML属性によって親コンポーネントに明示的に渡されていない親データにはアクセスできません。親コンポーネントに明示的に渡されていないデフォルトの`@props`値は、`@aware`ディレクティブではアクセスすることができません。

<a name="anonymous-component-paths"></a>
### 匿名コンポーネントのパス

前で説明したように、匿名コンポーネントは通常、`resources/views/components`ディレクトリに、Bladeテンプレートを配置することにより定義します。しかし、時にはデフォルトのパスに加えて、他の匿名コンポーネントパスをLaravelへ登録したい場合が起こるでしょう。

`anonymousComponentPath`メソッドは、最初の引数に匿名コンポーネントの場所の「パス」、第２番引数としてコンポーネントを配置するための「名前空間」をオプションで取ります。通常、このメソッドはアプリケーションの[サービスプロバイダ](/docs/{{version}}/providers)の`boot`メソッドから呼び出す必要があります。

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Blade::anonymousComponentPath(__DIR__.'/../components');
    }

上記の例のように、コンポーネントパスへプレフィックスを指定せず登録した場合、対応するプレフィックスを指定していないBladeコンポーネントと同様にレンダーします。例えば、上記で登録したパスに`panel.blade.php`コンポーネントが存在する場合、以下のようにレンダーされるでしょう。

```blade
<x-panel />
```

プレフィックスの「名前空間」は、`anonymousComponentPath`メソッドの第２引数として与えます。

    Blade::anonymousComponentPath(__DIR__.'/../components', 'dashboard');

プレフィックスを指定すると、その「名前空間」内のコンポーネントは、コンポーネントがレンダーされるとき、そのコンポーネントの名前空間にプレフィックスを付けレンダーされます。

```blade
<x-dashboard::panel />
```

<a name="building-layouts"></a>
## レイアウト構築

<a name="layouts-using-components"></a>
### コンポーネント使用の構築

ほとんどのWebアプリケーションは、さまざまなページ間で同じ一般的なレイアウトを共有しています。私たちが作成するすべてのビューでレイアウト全体のHTMLを繰り返さなければならない場合、アプリケーションを維持するのは非常に面倒になると懸念できるでしょう。幸いに、このレイアウトを単一の[Bladeコンポーネント](#コンポーネント)として定義し、アプリケーション全体で使用できるため、便利です。

<a name="defining-the-layout-component"></a>
#### レイアウトコンポーネントの定義

例として、「TODO」リストアプリケーションを構築していると想像してください。次のような`layout`コンポーネントを定義できるでしょう。

```blade
<!-- resources/views/components/layout.blade.php -->

<html>
    <head>
        <title>{{ $title ?? 'Todo Manager' }}</title>
    </head>
    <body>
        <h1>Todos</h1>
        <hr/>
        {{ $slot }}
    </body>
</html>
```

<a name="applying-the-layout-component"></a>
#### レイアウトコンポーネントの適用

`layout`コンポーネントを定義したら、コンポーネントを利用するBladeビューを作成できます。この例では、タスクリストを表示する簡単なビューを定義します。

```blade
<!-- resources/views/tasks.blade.php -->

<x-layout>
    @foreach ($tasks as $task)
        {{ $task }}
    @endforeach
</x-layout>
```

コンポーネントへ挿入されるコンテンツは、`layout`コンポーネント内のデフォルト`$slot`変数として提供されることを覚えてください。お気づきかもしれませんが、`layout`も提供されるのであれば、`$title`スロットを尊重します。それ以外の場合は、デフォルトのタイトルが表示されます。[コンポーネントのドキュメント](#components)で説明している標準スロット構文を使用して、タスクリストビューからカスタムタイトルを挿入できます。

```blade
<!-- resources/views/tasks.blade.php -->

<x-layout>
    <x-slot:title>
        カスタムタイトル
    </x-slot>

    @foreach ($tasks as $task)
        {{ $task }}
    @endforeach
</x-layout>
```

レイアウトとタスクリストのビューを定義したので、ルートから`task`ビューを返す必要があります。

    use App\Models\Task;

    Route::get('/tasks', function () {
        return view('tasks', ['tasks' => Task::all()]);
    });

<a name="layouts-using-template-inheritance"></a>
### テンプレート継承を使用するレイアウト

<a name="defining-a-layout"></a>
#### レイアウトの定義

レイアウトは、「テンプレート継承」を介して作成することもできます。これは、[コンポーネント](#components)の導入前にアプリケーションを構築するための主な方法でした。

始めるにあたり、簡単な例を見てみましょう。まず、ページレイアウトを調べます。ほとんどのWebアプリケーションはさまざまなページ間で同じ一般的なレイアウトを共有するため、このレイアウトを単一のBladeビューとして定義でき、便利です。

```blade
<!-- resources/views/layouts/app.blade.php -->

<html>
    <head>
        <title>App Name - @yield('title')</title>
    </head>
    <body>
        @section('sidebar')
            This is the master sidebar.
        @show

        <div class="container">
            @yield('content')
        </div>
    </body>
</html>
```

ご覧のとおり、このファイルには典型的なHTMLマークアップが含まれています。ただし、`@section`と`@yield`ディレクティブに注意してください。名前が暗示するように、`@section`ディレクティブは、コンテンツのセクションを定義しますが、`@yield`ディレクティブは指定するセクションの内容を表示するために使用します。

アプリケーションのレイアウトを定義したので、レイアウトを継承する子ページを定義しましょう。

<a name="extending-a-layout"></a>
#### レイアウトの拡張

子ビューを定義するときは、`@extends` Bladeディレクティブを使用して、子ビューが「継承する」のレイアウトを指定します。Bladeレイアウトを拡張するビューは、`@section`ディレクティブを使用してレイアウトのセクションにコンテンツを挿入できます。上記の例で見たように、これらのセクションの内容は`@yield`を使用してレイアウトへ表示できます。

```blade
<!-- resources/views/child.blade.php -->

@extends('layouts.app')

@section('title', 'Page Title')

@section('sidebar')
    @@parent

    <p>これはマスターサイドバーに追加される</p>
@endsection

@section('content')
    <p>これは本文の内容</p>
@endsection
```

この例では、`sidebar`のセクションは、`@parent`ディレクティブを利用して、(上書きするのではなく)、レイアウトのサイドバーにコンテンツを追加しています。`@parent`ディレクティブは、ビューをレンダするときにレイアウトの内容へ置き換えられます。

> [!NOTE]
> 前の例とは反対に、この「サイドバー」のセクションは`@show`の代わりに`@endection`で終わります。`@endection`ディレクティブはセクションを定義するだけですが、一方の`@show`は定義し、そのセクションを**すぐに挿入**します。

`@yield`ディレクティブは、デフォルト値を２番目の引数に取ります。この値は、生成するセクションが未定義の場合にレンダされます。

```blade
@yield('content', 'Default content')
```

<a name="forms"></a>
## フォーム

<a name="csrf-field"></a>
### CSRFフィールド

アプリケーションでHTMLフォームを定義したときは、[CSRF保護](/docs/{{version}}/csrf)ミドルウェアがリクエストをバリデートできるように、フォームへ隠しCSRFトークンフィールドを含める必要があります。`@csrf` Bladeディレクティブを使用してトークンフィールドを生成できます。

```blade
<form method="POST" action="/profile">
    @csrf

    ...
</form>
```

<a name="method-field"></a>
### Methodフィールド

HTMLフォームは`put`、`patch`、または`delete`リクエストを作ることができないので、これらのHTTP動詞を偽装するために`_Method`隠しフィールドを追加する必要があります。`@method`Bladeディレクティブは、皆さんのためこのフィールドを作成します。

```blade
<form action="/foo/bar" method="POST">
    @method('PUT')

    ...
</form>
```

<a name="validation-errors"></a>
### バリデーションエラー

指定する属性に[バリデーションエラーメッセージ](/docs/{{version}}/validation#quick-displaying-the-validation-errors)が存在するかをすばやく確認するために`@error`ディレクティブを使用できます。`@error`ディレクティブ内では、エラーメッセージを表示するため、`$message`変数をエコーできます。

```blade
<!-- /resources/views/post/create.blade.php -->

<label for="title">Post Title</label>

<input id="title"
    type="text"
    class="@error('title') is-invalid @enderror">

@error('title')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror
```

`@error`ディレクティブは"if"文へコンパイルされるため、属性のエラーがない場合にコンテンツをレンダしたい場合は、`@else`ディレクティブを使用できます。

```blade
<!-- /resources/views/auth.blade.php -->

<label for="email">Email address</label>

<input id="email"
    type="email"
    class="@error('email') is-invalid @else is-valid @enderror">
```

ページが複数のフォームを含んでいる場合にエラーメッセージを取得するため、[特定のエラーバッグの名前](/docs/{{version}}/validation#named-error-bags)を第２引数へ渡せます。

```blade
<!-- /resources/views/auth.blade.php -->

<label for="email">Email address</label>

<input id="email"
    type="email"
    class="@error('email', 'login') is-invalid @enderror">

@error('email', 'login')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror
```

<a name="stacks"></a>
## スタック

Bladeを使用すると、別のビューまたはレイアウトのどこか別の場所でレンダできる名前付きスタックに入れ込めます。これは、子ビューに必要なJavaScriptライブラリを指定する場合、とくに便利です。

```blade
@push('scripts')
    <script src="/example.js"></script>
@endpush
```

もし、指定した論理式が、`true`と評価された場合にコンテンツを`@push`したいのであれば、`@pushIf`ディレクティブを使用します。

```blade
@pushIf($shouldPush, 'scripts')
    <script src="/example.js"></script>
@endPushIf
```

必要な回数だけスタックに入れ込めます。スタックしたコンテンツを完全にレンダするには、スタックの名前を`@stack`ディレクティブに渡します。

```blade
<head>
    <!-- HEADのコンテンツ -->

    @stack('scripts')
</head>
```

スタックの先頭にコンテンツを追加する場合は、`@prepend`ディレクティブを使用します。

```blade
@push('scripts')
    これは２番めになる
@endpush

// Later...

@prepend('scripts')
    これが最初になる
@endprepend
```

<a name="service-injection"></a>
## サービス注入

`@inject`ディレクティブは、Laravel[サービスコンテナ](/docs/{{version}}/container)からサービスを取得するために使用します。`@inject`に渡す最初の引数は、サービスが配置される変数の名前であり、２番目の引数は解決したいサービスのクラスかインターフェイス名です。

```blade
@inject('metrics', 'App\Services\MetricsService')

<div>
    Monthly Revenue: {{ $metrics->monthlyRevenue() }}.
</div>
```

<a name="rendering-inline-blade-templates"></a>
## インラインBladeテンプレートのレンダ

素のBladeテンプレート文字列を有効なHTMLに変換する必要が起きるかもしれません。このような場合、`Blade`ファサードが提供する`render`メソッドを使用します。`render`メソッドはBladeテンプレート文字列と、テンプレートに提供するデータの配列をオプションで受け取ります。

```php
use Illuminate\Support\Facades\Blade;

return Blade::render('Hello, {{ $name }}', ['name' => 'Julian Bashir']);
```

LaravelはインラインのBladeテンプレートを`storage/framework/views`ディレクトリに書き込むことによってレンダします。Bladeテンプレートをレンダリングした後、Laravelにこれらの一時ファイルを削除させたい場合は、このメソッドに`deleteCachedView`引数を指定してください。

```php
return Blade::render(
    'Hello, {{ $name }}',
    ['name' => 'Julian Bashir'],
    deleteCachedView: true
);
```

<a name="rendering-blade-fragments"></a>
## Bladeフラグメントのレンダ

[Turbo](https://turbo.hotwired.dev/)や[htmx](https://htmx.org/)などのフロントエンドフレームワークを使用する場合は、HTTPレスポンスの中からBladeテンプレートの一部のみを返す必要が起きる場合があります。Bladeの「フラグメント(fragment)」は、まさにそれを可能にします。まず、Bladeテンプレートの一部を`@fragment`ディレクティブと`@endfragment`ディレクティブの中に配置します。

```blade
@fragment('user-list')
    <ul>
        @foreach ($users as $user)
            <li>{{ $user->name }}</li>
        @endforeach
    </ul>
@endfragment
```

その後、このテンプレートを利用するビューをレンダする際に、`fragment`メソッドを呼び出し、指定したフラグメントのみを送信HTTPレスポンスへ含めるように指示します。

```php
return view('dashboard', ['users' => $users])->fragment('user-list');
```

`fragmentIf`メソッドを使用すると、指定条件に基づき、ビューの断片を条件付きで返せます。そうでなければ、ビュー全体を返します。

```php
return view('dashboard', ['users' => $users])
    ->fragmentIf($request->hasHeader('HX-Request'), 'user-list');
```

`fragments`と`fragmentsIf`メソッドを使用すると、レスポンスへ複数のビューフラグメントを返せます。フラグメントは一つに連結します。

```php
view('dashboard', ['users' => $users])
    ->fragments(['user-list', 'comment-list']);

view('dashboard', ['users' => $users])
    ->fragmentsIf(
        $request->hasHeader('HX-Request'),
        ['user-list', 'comment-list']
    );
```

<a name="extending-blade"></a>
## Bladeの拡張

Bladeでは、`directive`メソッドを使用して独自のカスタムディレクティブを定義できます。Bladeコンパイラは、カスタムディレクティブを検出すると、ディレクティブに含まれる式を使用して、提供したコールバックを呼び出します。

次の例は、指定した`$var`をフォーマットする`@datetime($var)`ディレクティブを作成します。これは`DateTime`インスタンスである必要があります。

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\Blade;
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
         * 全アプリケーションサービスの初期起動処理
         */
        public function boot(): void
        {
            Blade::directive('datetime', function (string $expression) {
                return "<?php echo ($expression)->format('m/d/Y H:i'); ?>";
            });
        }
    }

ご覧のとおり、ディレクティブに渡される式に`format`メソッドをチェーンします。したがって、この例でディレクティブが生成する最終的なPHPは、以下のよ​​うになります。

    <?php echo ($var)->format('m/d/Y H:i'); ?>

> [!WARNING]
> Bladeディレクティブのロジックを更新した後は、キャッシュ済みのBladeビューをすべて削除する必要があります。キャッシュ済みBladeビューは、`view:clear` Artisanコマンドを使用して削除できます。

<a name="custom-echo-handlers"></a>
### カスタムEchoハンドラ

Bladeを使ってオブジェクトを「エコー」しようとすると、そのオブジェクトの`__toString`メソッドが呼び出されます。[`__toString`](https://www.php.net/manual/ja/language.oop5.magic.php#object.tostring)メソッドは、PHPが組み込んでいる一つの「マジックメソッド」です。しかし、操作するクラスがサードパーティのライブラリへ属している場合など、特定のクラスで`__toString`メソッドを制御できない場合が起こりえます。

このような場合、Bladeで、その特定のタイプのオブジェクト用に、カスタムEchoハンドラを登録できます。Bladeの`stringable`メソッドを呼び出してください。`stringable`メソッドは、クロージャを引数に取ります。このクロージャは、自分がレンダリングを担当するオブジェクトのタイプをタイプヒントで指定する必要があります。一般的には、アプリケーションの`AppServiceProvider`クラスの`boot`メソッド内で、`stringable`メソッドを呼び出します。

    use Illuminate\Support\Facades\Blade;
    use Money\Money;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Blade::stringable(function (Money $money) {
            return $money->formatTo('en_GB');
        });
    }

カスタムEchoハンドラを定義したら、Bladeテンプレート内のオブジェクトを簡単にエコーできます。

```blade
Cost: {{ $money }}
```

<a name="custom-if-statements"></a>
### カスタムIf文

カスタムディレクティブのプログラミングは、単純なカスタム条件ステートメントを定義するとき、必要以上の複雑さになる場合があります。そのため、Bladeには`Blade::if`メソッドが用意されており、クロージャを使用してカスタム条件付きディレクティブをすばやく定義できます。たとえば、アプリケーションのデフォルト「ディスク」の設定をチェックするカスタム条件を定義しましょう。これは、`AppServiceProvider`の`boot`メソッドで行います。

    use Illuminate\Support\Facades\Blade;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Blade::if('disk', function (string $value) {
            return config('filesystems.default') === $value;
        });
    }

カスタム条件を定義すると、テンプレート内で使用できます。

```blade
@disk('local')
    <!-- アプリケーションはローカルディスクを使用している -->
@elsedisk('s3')
    <!-- アプリケーションはs3ディスクを使用している -->
@else
    <!-- アプリケーションは他のディスクを使用している -->
@enddisk

@unlessdisk('local')
    <!-- アプリケーションはローカルディスクを使用していない -->
@enddisk
```
