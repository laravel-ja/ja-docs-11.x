# 多言語化

- [イントロダクション](#introduction)
    - [言語ファイルのリソース公開](#publishing-the-language-files)
    - [ロケールの設定](#configuring-the-locale)
    - [言語の複数形](#pluralization-language)
- [翻訳文字列の定義](#defining-translation-strings)
    - [短縮キーの使用](#using-short-keys)
    - [翻訳文字列をキーとして使用](#using-translation-strings-as-keys)
- [翻訳文字列の取得](#retrieving-translation-strings)
    - [翻訳文字列中のパラメータの置換](#replacing-parameters-in-translation-strings)
    - [複数形](#pluralization)
- [パッケージ言語ファイルのオーバーライド](#overriding-package-language-files)

<a name="introduction"></a>
## イントロダクション

> [!NOTE]
> デフォルトでは、Laravelアプリケーションのスケルトンは、`lang`ディレクトリを用意していません。Laravelの言語ファイルをカスタマイズしたい場合は、`lang:publish` Artisanコマンドでリソース公開できます。

Laravelの多言語機能は、さまざまな言語の文字列を取得する便利な方法を提供し、アプリケーション内で複数の言語を簡単にサポートできるようにしています。

Laravelは翻訳文字列を管理する２つの方法を提供しています。最初の方法は、言語文字列をアプリケーションの`lang`ディレクトリ内のファイルへ格納する方法です。このディレクトリ内には、アプリケーションがサポートする、各言語のサブディレクトリを用意できます。これは、バリデーションエラーメッセージのような、Laravel組み込み機能の翻訳文字列を管理するためにLaravelが使用する方法です。

    /lang
        /en
            messages.php
        /es
            messages.php

もしくは、翻訳文字列を`lang`ディレクトリ下のJSON ファイルで定義することもできます。この方法を取る場合、アプリケーションがサポートする各言語ごとに、このディレクトリの中に対応するJSONファイルを用意します。この方法は、翻訳可能な文字列が大量にあるアプリケーションに推奨します。

    /lang
        en.json
        es.json

このドキュメントでは、翻訳文字列を管理する各アプローチについて説明します。

<a name="publishing-the-language-files"></a>
### 言語ファイルのリソース公開

Laravelアプリケーションのスケルトンに、デフォルトで`lang`ディレクトリを用意していません。Laravelの言語ファイルをカスタマイズしたり、独自の言語ファイルを作成したい場合は、`lang:publish` Artisanコマンドを使用して、`lang`ディレクトリをスカフォールドする必要があります。`lang:publish`コマンドは、アプリケーションに`lang`ディレクトリを作成し、Laravelが使用する言語ファイルのデフォルトセットをリソース公開します。

```shell
php artisan lang:publish
```

<a name="configuring-the-locale"></a>
### ロケールの設定

The default language for your application is stored in the `config/app.php` configuration file's `locale` configuration option, which is typically set using the `APP_LOCALE` environment variable. You are free to modify this value to suit the needs of your application.

You may also configure a "fallback language", which will be used when the default language does not contain a given translation string. Like the default language, the fallback language is also configured in the `config/app.php` configuration file, and its value is typically set using the `APP_FALLBACK_LOCALE` environment variable.

`App`ファサードが提供する`setLocale`メソッドを使用して、単一のHTTPリクエストの間のデフォルト言語を実行時に変更できます。

    use Illuminate\Support\Facades\App;

    Route::get('/greeting/{locale}', function (string $locale) {
        if (! in_array($locale, ['en', 'es', 'fr'])) {
            abort(400);
        }

        App::setLocale($locale);

        // ...
    });

<a name="determining-the-current-locale"></a>
#### 現在のロケールの決定

`App`ファサードの`currentLocale`メソッドと`isLocale`メソッドを使用して、現在のロケールを判定したり、ロケールが指定する値であるかどうかを確認したりできます。

    use Illuminate\Support\Facades\App;

    $locale = App::currentLocale();

    if (App::isLocale('en')) {
        // ...
    }

<a name="pluralization-language"></a>
### 言語の複数形

Laravelの「複数形化機能（Pluralizer）」は、Eloquentやフレームワークの他の部分で単数形の文字列を複数形の文字列に変換するために使用していますが、英語以外の言語を使用するように指示できます。これは、アプリケーションのサービスプロバイダの`boot`メソッドの中で`useLanguage`メソッドを呼び出して、実現します。現在サポートしている言語は、フランス語（`french`）、ノルウェー語ーブークモール（`norwegian-bokmal`）、ポルトガル語（`portuguese`）、スペイン語（`spanish`）、トルコ語（`turkish`）です。

    use Illuminate\Support\Pluralizer;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Pluralizer::useLanguage('spanish');

        // ...
    }

> [!WARNING]
> 言語の複数形化をカスタマイズする場合、Eloquentモデルの[テーブル名](/docs/{{version}}/eloquent#table-names)は、明示的に定義する必要があります。

<a name="defining-translation-strings"></a>
## 翻訳文字列の定義

<a name="using-short-keys"></a>
### 短縮キーの使用

通常、翻訳文字列は`lang`ディレクトリにあるファイルに格納します。このディレクトリの中に、アプリケーションがサポートする各言語のサブディレクトリがあるはずです。これは、バリデーションエラーメッセージのような、Laravelの組み込み機能の翻訳文字列を管理するために、Laravelが採用している方法です。

    /lang
        /en
            messages.php
        /es
            messages.php

すべての言語ファイルは、キー付き文字列の配列を返します。例えば:

    <?php

    // lang/en/messages.php

    return [
        'welcome' => 'Welcome to our application!',
    ];

> [!WARNING]
> 地域によって異なる言語の場合、ISO15897に従って言語ディレクトリに名前を付ける必要があります。たとえば、「en-gb」ではなく「en_GB」をイギリス英語に使用する必要があります。

<a name="using-translation-strings-as-keys"></a>
### 翻訳文字列をキーとして使用

翻訳可能な文字列が多数あるアプリケーションの場合、「短縮キー」ですべての文字列を定義すると、ビューでキーを参照するときに混乱する可能性があり、アプリケーションがサポートするすべての翻訳文字列のキーを継続的に作成するのは面倒です。

このため、Laravelは文字列の「デフォルト」翻訳をキーとして、翻訳文字列を定義するサポートも提供しています。翻訳文字列をキーとして使用する言語ファイルは、JSONファイルとして`lang`ディレクトリに保存します。例えば、アプリケーションにスペイン語の翻訳がある場合、`lang/es.json`ファイルを作成する必要があります。

```json
{
    "I love programming.": "Me encanta programar."
}
```

#### キー／ファイルの競合

他の翻訳ファイル名と衝突するような翻訳文字列のキーを定義してはいけません。例えば、`nl/action.php`ファイルは存在するが`nl.json`ファイルは存在しないのに、`__('Action')`を"NL"ロケールへ翻訳すると、トランスレータは結果として`nl/action.php`の内容全体を返します。

<a name="retrieving-translation-strings"></a>
## 翻訳文字列の取得

`__`ヘルパ関数を使い、言語ファイルから翻訳文字列を取得できます。「短いキー」を使い、翻訳文字列を定義している場合は、キーを含むファイルとキー自身を「ドット」構文で、`__`関数に渡す必要があります。例として、`lang/en/messages.php`という言語ファイルから`welcome`という翻訳文字列を取得してみましょう。

    echo __('messages.welcome');

指定する翻訳文字列が存在しない場合、`__`関数は翻訳文字列キーを返します。したがって、上記の例を使用すると、翻訳文字列が存在しない場合、`__`関数は`messages.welcome`を返します。

[デフォルトの翻訳文字列を翻訳キー](#using-translation-strings-as-keys)として使用している場合は、文字列のデフォルトの翻訳を`__`関数に渡す必要があります。

    echo __('I love programming.');

この場合も、翻訳文字列が存在しない場合、`__`関数は指定する翻訳文字列キーを返します。

[Bladeテンプレートエンジン](/docs/{{version}}/Blade)を使用している場合は、`{{}}`エコー構文を使用して翻訳文字列を表示できます。

    {{ __('messages.welcome') }}

<a name="replacing-parameters-in-translation-strings"></a>
### 翻訳文字列中のパラメータの置換

必要に応じて、翻訳文字列にプレースホルダーを定義できます。すべてのプレースホルダーには接頭辞`:`が付いています。たとえば、プレースホルダー名を使用してウェルカムメッセージを定義できます。

    'welcome' => 'Welcome, :name',

翻訳文字列を取得するときにプレースホルダーを置き換えるには、置換の配列を２番目の引数として`__`関数に渡します。

    echo __('messages.welcome', ['name' => 'dayle']);

プレースホルダーにすべて大文字が含まれている場合、または最初の文字のみが大文字になっている場合、変換された値はそれに応じて大文字になります。

    'welcome' => 'Welcome, :NAME', // Welcome, DAYLE
    'goodbye' => 'Goodbye, :Name', // Goodbye, Dayle

<a name="object-replacement-formatting"></a>
#### オブジェクト置換フォーマット

翻訳用のプレースホルダーとしてオブジェクトを指定する場合、 オブジェクトの`__toString`メソッドを呼び出します。`__toString`](https://www.php.net/manual/ja/language.oop5.magic.php#object.tostring)メソッドは、PHP組み込みの「マジックメソッド」の一種です。しかし、サードパーティのライブラリに含まれるクラスとやり取りする場合など、指定するクラスの`__toString`メソッドを制御できないこともあります。

このような場合のため、Laravelでは特定タイプのオブジェクトに対する、カスタムフォーマットハンドラを登録できます。これを行うには、トランスレータの`stringable`メソッドを呼び出す必要があります。`stringable`メソッドは、フォーマットに対応するオブジェクトタイプをタイプヒントする必要があります。通常、`stringable`メソッドはアプリケーションの`AppServiceProvider`クラスの、`boot`メソッド内で呼び出します。

    use Illuminate\Support\Facades\Lang;
    use Money\Money;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Lang::stringable(function (Money $money) {
            return $money->formatTo('en_GB');
        });
    }

<a name="pluralization"></a>
### 複数形

別々の言語には複数形のさまざまな複雑なルールがあるため、複数形化は複雑な問題です。ただし、Laravelは、定義した複数形化ルールに基づいて文字列を異なる方法で翻訳する手助けができます。`|`文字を使用すると、文字列の単数形と複数形を区別できます。

    'apples' => 'There is one apple|There are many apples',

もちろん、[翻訳文字列をキー](#using-translation-strings-as-keys)として使用する場合でも、複数形化をサポートしています。

```json
{
    "There is one apple|There are many apples": "Hay una manzana|Hay muchas manzanas"
}
```

複数の値の範囲の変換文字列を指定する、より複雑な複数形化ルールを作成することもできます。

    'apples' => '{0} There are none|[1,19] There are some|[20,*] There are many',

複数形化オプションのある翻訳文字列を定義した後、`trans_choice`関数を使用して、指定する「個数」に合った行を取得できます。以下の例では、個数が１より大きいため、複数形の翻訳文字列が返されます。

    echo trans_choice('messages.apples', 10);

複数形化文字列でプレースホルダー属性を定義することもできます。これらのプレースホルダーは、`trans_choice`関数の３番目の引数として配列を渡すことで置き換えられます。

    'minutes_ago' => '{1} :value minute ago|[2,*] :value minutes ago',

    echo trans_choice('time.minutes_ago', 5, ['value' => 5]);

`trans_choice`関数に渡した整数値を表示したい場合は、組み込みの`:count`プレースホルダーを使用できます。

    'apples' => '{0} There are none|{1} There is one|[2,*] There are :count',

<a name="overriding-package-language-files"></a>
## パッケージ言語ファイルのオーバーライド

パッケージは、独自の言語ファイルを用意している場合があります。調整するため、パッケージのコアファイルを変更する代わりに、`lang/vendor/{package}/{locale}`ディレクトリへファイルを配置し、上書きできます。

例えば、`skyrim/hearthfire`パッケージの`messages.php`にある、英語の翻訳文字列を上書きする必要がある場合、言語ファイルを`lang/vendor/hearthfire/en/messages.php`へ置く必要があります。このファイルは、オーバーライドしたい翻訳文字列のみを定義します。オーバーライドしない翻訳文字列は、パッケージの元の言語ファイルから読み込まれます。
