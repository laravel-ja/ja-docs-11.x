# Prompts

- [イントロダクション](#introduction)
- [インストール](#installation)
- [利用可能なプロンプト](#available-prompts)
    - [テキスト](#text)
    - [テキストエリア](#textarea)
    - [パスワード](#password)
    - [確認](#confirm)
    - [選択](#select)
    - [複数選択](#multiselect)
    - [候補](#suggest)
    - [検索](#search)
    - [マルチ検索](#multisearch)
    - [一時停止](#pause)
- [バリデーション前の入力変換](#transforming-input-before-validation)
- [フォーム](#forms)
- [情報メッセージ](#informational-messages)
- [テーブル](#tables)
- [スピン](#spin)
- [プログレスバー](#progress)
- [Clearing the Terminal](#clear)
- [ターミナルの考察](#terminal-considerations)
- [未サポートの環境とフォールバック](#fallbacks)

<a name="introduction"></a>
## イントロダクション

[Laravel Prompts](https://github.com/laravel/prompts)は、美しくユーザーフレンドリーなUIをコマンドラインアプリケーションに追加するためのPHPパッケージで、プレースホルダテキストやバリデーションなどのブラウザにあるような機能を備えています。

<img src="https://laravel.com/img/docs/prompts-example.png">

Laravel Promptsは、[Artisanコンソールコマンド](/docs/{{version}}/artisan#writing-commands)でユーザー入力を受けるために最適ですが、コマンドラインのPHPプロジェクトでも使用できます。

> [!NOTE]
> Laravel PromptsはmacOS、Linux、WindowsのWSLをサポートしています。詳しくは、[未サポートの環境とフォールバック](#fallbacks)のドキュメントをご覧ください。

<a name="インストール"></a>
## インストール

Laravelの最新リリースには、はじめからLaravel Promptsを用意してあります。

Composerパッケージマネージャを使用して、他のPHPプロジェクトにLaravel Promptsをインストールすることもできます。

```shell
composer require laravel/prompts
```

<a name="available-prompts"></a>
## 利用可能なプロンプト

<a name="text"></a>
### テキスト

`text`関数は、指定した質問でユーザーに入力を促し、回答を受け、それを返します。

```php
use function Laravel\Prompts\text;

$name = text('What is your name?');
```

プレースホルダテキストとデフォルト値、ヒントの情報も指定できます。

```php
$name = text(
    label: 'What is your name?',
    placeholder: 'E.g. Taylor Otwell',
    default: $user?->name,
    hint: 'This will be displayed on your profile.'
);
```

<a name="text-required"></a>
#### 必須値

入力値が必須の場合は、`required`引数を渡してください。

```php
$name = text(
    label: 'What is your name?',
    required: true
);
```

バリデーション・メッセージをカスタマイズしたい場合は、文字列を渡すこともできます。

```php
$name = text(
    label: 'What is your name?',
    required: 'Your name is required.'
);
```

<a name="text-validation"></a>
#### 追加のバリデーション

最後に、追加のバリデーションロジックを実行したい場合は、`validate`引数にクロージャを渡します。

```php
$name = text(
    label: 'What is your name?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

クロージャは入力された値を受け取り、エラーメッセージを返すか、バリデーションに合格した場合は、`null`を返します。

あるいは、Laravelの[バリデーション](/docs/{{version}}/validation)を活用することもできます。そのためには、`validate`引数に属性名と必要なバリデーションルールを含む配列を指定します。

```php
$name = text(
    label: 'What is your name?',
    validate: ['name' => 'required|max:255|unique:users']
);
```

<a name="textarea"></a>
### テキストエリア

`textarea`関数は、指定した質問をユーザーに促し、複数行のtextareaで入力を受け付け、それを返します。

```php
use function Laravel\Prompts\textarea;

$story = textarea('Tell me a story.');
```

プレースホルダ・テキスト、デフォルト値、情報のヒントを含めることもできます。

```php
$story = textarea(
    label: 'Tell me a story.',
    placeholder: 'This is a story about...',
    hint: 'This will be displayed on your profile.'
);
```

<a name="textarea-required"></a>
#### 必須値

入力値が必要な場合は、`required`引数を渡してください。

```php
$story = textarea(
    label: 'Tell me a story.',
    required: true
);
```

バリデーションメッセージをカスタマイズしたい場合は、文字列を渡すこともできます。

```php
$story = textarea(
    label: 'Tell me a story.',
    required: 'A story is required.'
);
```

<a name="textarea-validation"></a>
#### 追加のバリデーション

最後に、追加のバリデーションロジックを実行したい場合は、`validate`引数にクロージャを渡します。

```php
$story = textarea(
    label: 'Tell me a story.',
    validate: fn (string $value) => match (true) {
        strlen($value) < 250 => 'The story must be at least 250 characters.',
        strlen($value) > 10000 => 'The story must not exceed 10,000 characters.',
        default => null
    }
);
```

このクロージャは入力値を受け取り、エラーメッセージを返すか、バリデーションをパスした場合は、`null`を返します。

あるいは、Laravelの[バリデータ](/docs/{{version}}/validation)を利用することもできます。それには、`validate`引数へ属性名と必要なバリデーションルールを含む配列を指定します。

```php
$story = textarea(
    label: 'Tell me a story.',
    validate: ['story' => 'required|max:10000']
);
```

<a name="password"></a>
### パスワード

`password`関数は`text`関数と似ていますが、コンソールで入力されるユーザー入力をマスクします。これはパスワードのような機密情報を要求するときに便利です：

```php
use function Laravel\Prompts\password;

$password = password('What is your password?');
```

プレースホルダテキストと情報のヒントも含められます。

```php
$password = password(
    label: 'What is your password?',
    placeholder: 'password',
    hint: 'Minimum 8 characters.'
);
```

<a name="password-required"></a>
#### 必須値

入力値が必須の場合は、`required`引数を渡してください。

```php
$password = password(
    label: 'What is your password?',
    required: true
);
```

バリデーション・メッセージをカスタマイズしたい場合は、文字列を渡すこともできます。

```php
$password = password(
    label: 'What is your password?',
    required: 'The password is required.'
);
```

<a name="password-validation"></a>
#### 追加のバリデーション

最後に、追加のバリデーションロジックを実行したい場合は、`validate`引数にクロージャを渡します。

```php
$password = password(
    label: 'What is your password?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 8 => 'The password must be at least 8 characters.',
        default => null
    }
);
```

クロージャは入力された値を受け取り、エラーメッセージを返すか、バリデーションに合格した場合は、`null`を返します。

あるいは、Laravelの[バリデーション](/docs/{{version}}/validation)を活用することもできます。そのためには、`validate`引数に属性名と必要なバリデーションルールを含む配列を指定します。

```php
$password = password(
    label: 'What is your password?',
    validate: ['password' => 'min:8']
);
```

<a name="confirm"></a>
### 確認

ユーザーに "yes／no"の確認を求める必要がある場合は、`confirm`関数を使います。ユーザーは矢印キーを使うか、`y`または`n`を押して回答を選択します。この関数は`true`または`false`を返します。

```php
use function Laravel\Prompts\confirm;

$confirmed = confirm('Do you accept the terms?');
```

また、デフォルト値、「はい」／「いいえ」ラベルをカスタマイズする文言、情報ヒントも含められます。

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    default: false,
    yes: 'I accept',
    no: 'I decline',
    hint: 'The terms must be accepted to continue.'
);
```

<a name="confirm-required"></a>
#### 必須の"Yes"

必要であれば、`required`引数を渡し、ユーザーに"Yes"を選択させることもできます。

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    required: true
);
```

バリデーション・メッセージをカスタマイズしたい場合は、文字列を渡すこともできます。

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    required: 'You must accept the terms to continue.'
);
```

<a name="select"></a>
### 選択

ユーザーにあらかじめ定義された選択肢から選ばせる必要がある場合は、`select`関数を使います。

```php
use function Laravel\Prompts\select;

$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner']
);
```

デフォルト選択肢と情報のヒントも指定できます。

```php
$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner'],
    default: 'Owner',
    hint: 'The role may be changed at any time.'
);
```

`options`引数に連想配列を渡し、選択値の代わりにキーを返すこともできます。

```php
$role = select(
    label: 'What role should the user have?',
    options: [
        'member' => 'Member',
        'contributor' => 'Contributor',
        'owner' => 'Owner',
    ],
    default: 'owner'
);
```

５選択肢を超えると、選択肢がリスト表示されます。`scroll`引数を渡し、カスタマイズ可能です。

```php
$role = select(
    label: 'Which category would you like to assign?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```

<a name="select-validation"></a>
#### 追加のバリデーション

他のプロンプト関数とは異なり、他に何も選択できなくなるため、`select`関数は`required`引数を受けません。しかし、選択肢を提示したいが、選択されないようにする必要がある場合は、`validate`引数にクロージャを渡してください。

```php
$role = select(
    label: 'What role should the user have?',
    options: [
        'member' => 'Member',
        'contributor' => 'Contributor',
        'owner' => 'Owner',
    ],
    validate: fn (string $value) =>
        $value === 'owner' && User::where('role', 'owner')->exists()
            ? 'An owner already exists.'
            : null
);
```

`options`引数が連想配列の場合、クロージャは選択されたキーを受け取ります。連想配列でない場合は選択された値を受け取ります。クロージャはエラーメッセージを返すか、バリデーションに成功した場合は`null`を返してください。

<a name="multiselect"></a>
### 複数選択

ユーザーが複数のオプションを選択できるようにする必要がある場合は、`multiselect`関数を使用してください。

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete']
);
```

デフォルト選択肢と情報のヒントも指定できます。

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete'],
    default: ['Read', 'Create'],
    hint: 'Permissions may be updated at any time.'
);
```

`options`引数に連想配列を渡し、選択値の代わりにキーを返させることもできます。

```php
$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: [
        'read' => 'Read',
        'create' => 'Create',
        'update' => 'Update',
        'delete' => 'Delete',
    ],
    default: ['read', 'create']
);
```

５選択肢を超えると、選択肢がリスト表示されます。`scroll`引数を渡し、カスタマイズ可能です。

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```

<a name="multiselect-required"></a>
#### 必須値

デフォルトで、ユーザーは０個以上の選択肢を選択できます。`required`引数を渡せば、代わりに１つ以上の選択肢を強制できます。

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: true
);
```

バリデーションメッセージをカスタマイズしたい場合は、`required`引数に文字列を指定します。

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: 'You must select at least one category'
);
```

<a name="multiselect-validation"></a>
#### 追加のバリデーション

選択肢を提示するが、その選択肢が選択されないようにする必要がある場合は、`validate`引数にクロージャを渡してください。

```php
$permissions = multiselect(
    label: 'What permissions should the user have?',
    options: [
        'read' => 'Read',
        'create' => 'Create',
        'update' => 'Update',
        'delete' => 'Delete',
    ],
    validate: fn (array $values) => ! in_array('read', $values)
        ? 'All users require the read permission.'
        : null
);
```

`options`引数が連想配列の場合、クロージャは選択されたキーを受け取ります。連想配列でない場合は選択された値を受け取ります。クロージャはエラーメッセージを返すか、バリデーションに成功した場合は`null`を返してください。

<a name="suggest"></a>
### 候補

`suggest`関数を使用すると、可能性のある選択肢を自動補完できます。ユーザーは自動補完のヒントに関係なく、任意の答えを入力することもできます。

```php
use function Laravel\Prompts\suggest;

$name = suggest('What is your name?', ['Taylor', 'Dayle']);
```

あるいは、`suggest`関数の第２引数にクロージャを渡すこともできます。クロージャはユーザーが入力文字をタイプするたびに呼び出されます。クロージャは文字列パラメータにこれまでのユーザー入力を受け取り、オートコンプリート用の選択肢配列を返す必要があります。

```php
$name = suggest(
    label: 'What is your name?',
    options: fn ($value) => collect(['Taylor', 'Dayle'])
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
)
```

プレースホルダテキストとデフォルト値、ヒントの情報も指定できます。

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    placeholder: 'E.g. Taylor',
    default: $user?->name,
    hint: 'This will be displayed on your profile.'
);
```

<a name="suggest-required"></a>
#### 必須値

入力値が必須の場合は、`required`引数を渡してください。

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: true
);
```

バリデーション・メッセージをカスタマイズしたい場合は、文字列を渡すこともできます。

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: 'Your name is required.'
);
```

<a name="suggest-validation"></a>
#### 追加のバリデーション

最後に、追加のバリデーションロジックを実行したい場合は、`validate`引数にクロージャを渡します。

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

クロージャは入力された値を受け取り、エラーメッセージを返すか、バリデーションに合格した場合は、`null`を返します。

あるいは、Laravelの[バリデーション](/docs/{{version}}/validation)を活用することもできます。そのためには、`validate`引数に属性名と必要なバリデーションルールを含む配列を指定します。

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    validate: ['name' => 'required|min:3|max:255']
);
```

<a name="search"></a>
### 検索

ユーザーが選択できる選択肢がたくさんある場合、`search`機能を使えば、矢印キーを使って選択肢を選択する前に、検索クエリを入力して結果を絞り込むことができます。

```php
use function Laravel\Prompts\search;

$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

クロージャは、ユーザーがこれまでに入力したテキストを受け取り、 選択肢の配列を返さなければなりません。連想配列を返す場合は選択された選択肢のキーが返され、 そうでない場合はその値が代わりに返されます。

プレースホルダテキストと情報のヒントも含められます。

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    placeholder: 'E.g. Taylor Otwell',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    hint: 'The user will receive an email immediately.'
);
```

５選択肢を超えると、選択肢がリスト表示されます。`scroll`引数を渡し、カスタマイズ可能です。

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    scroll: 10
);
```

<a name="search-validation"></a>
#### 追加のバリデーション

追加のバリデーションロジックを実行したい場合は、`validate`引数にクロージャを渡します。

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    validate: function (int|string $value) {
        $user = User::findOrFail($value);

        if ($user->opted_out) {
            return 'This user has opted-out of receiving mail.';
        }
    }
);
```

`options`引数が連想配列の場合、クロージャは選択されたキーを受け取ります。連想配列でない場合は選択された値を受け取ります。クロージャはエラーメッセージを返すか、バリデーションに成功した場合は`null`を返してください。

<a name="multisearch"></a>
### マルチ検索

検索可能なオプションがたくさんあり、ユーザーが複数のアイテムを選択できるようにする必要がある場合、`multisearch`関数で、ユーザーが矢印キーとスペースバーを使ってオプションを選択する前に、検索クエリを入力してもらい、結果を絞り込めます。

```php
use function Laravel\Prompts\multisearch;

$ids = multisearch(
    'Search for the users that should receive the mail',
    fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

クロージャは、ユーザーがそれまでにタイプしたテキストを受け取り、オプションの配列を返さなければなりません。クロージャから連想配列を返す場合は、選択済みオプションのキーを返します。それ以外の場合は、代わりに値を返します。

プレースホルダテキストと情報のヒントも含められます。

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    placeholder: 'E.g. Taylor Otwell',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    hint: 'The user will receive an email immediately.'
);
```

リストをスクロールし始める前に、最大５選択肢表示します。`scroll`引数を指定し、カスタマイズできます。

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    scroll: 10
);
```

<a name="multisearch-required"></a>
#### 必須値

デフォルトで、ユーザーは０個以上の選択肢を選択できます。`required`引数を渡せば、代わりに１つ以上の選択肢を強制できます。

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    required: true
);
```

バリデーションメッセージをカスタマイズする場合は、`required`引数へ文字列を指定してください。

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    required: 'You must select at least one user.'
);
```

<a name="multisearch-validation"></a>
#### 追加のバリデーション

追加のバリデーションロジックを実行したい場合は、`validate`引数にクロージャを渡します。

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    validate: function (array $values) {
        $optedOut = User::whereLike('name', '%a%')->findMany($values);

        if ($optedOut->isNotEmpty()) {
            return $optedOut->pluck('name')->join(', ', ', and ').' have opted out.';
        }
    }
);
```

`options`クロージャが連想配列を返す場合、クロージャは選択済みのキーを受け取ります。そうでない場合は、選択済みの値を受け取ります。クロージャはエラーメッセージを返すか、バリデーションに合格した場合は`null`を返します。

<a name="pause"></a>
### 一時停止

`pause`関数は、ユーザーへ情報テキストを表示し、Enter/Returnキーが押されるのを待つことにより、ユーザーの続行の意思を確認するために使用します。

```php
use function Laravel\Prompts\pause;

pause('Press ENTER to continue.');
```

<a name="transforming-input-before-validation"></a>
## バリデーション前の入力変換

ときに、バリデーションを行う前にプロンプト入力を変換したい場合があることでしょう。たとえば、入力済み文字列から空白を取り除きたい場合などです。これを実現するために、プロンプト関数の多くは`transform`引数を用意しています。

```php
$name = text(
    label: 'What is your name?',
    transform: fn (string $value) => trim($value),
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

<a name="forms"></a>
## フォーム

追加のアクションを実行する前に、情報を収集するため複数のプロンプトを順番に表示することがよくあります。`form`関数でユーザーに埋めてもらうプロンプトをグループ化して作成できます。

```php
use function Laravel\Prompts\form;

$responses = form()
    ->text('What is your name?', required: true)
    ->password('What is your password?', validate: ['password' => 'min:8'])
    ->confirm('Do you accept the terms?')
    ->submit();
```

`submit`メソッドは、フォームのプロンプトから受け取る、全レスポンスを含む数値添字配列を返します。しかし、`name`引数で各プロンプトの名前を指定することも可能です。名前を指定した場合、その名前に対応するプロンプトのレスポンスへアクセスできます。

```php
use App\Models\User;
use function Laravel\Prompts\form;

$responses = form()
    ->text('What is your name?', required: true, name: 'name')
    ->password(
        label: 'What is your password?',
        validate: ['password' => 'min:8'],
        name: 'password'
    )
    ->confirm('Do you accept the terms?')
    ->submit();

User::create([
    'name' => $responses['name'],
    'password' => $responses['password'],
]);
```

`form`機能を使用する一番の利点は、ユーザーが`CTRL + U`を使用してフォーム内の前のプロンプトに戻ることができることです。これにより、ユーザーはフォーム全体をキャンセルして再開することなく、間違いを直したり、選択を変更したりできます。

フォームのプロンプトをより細かくコントロールする必要がある場合は、プロンプト関数を直接呼び出す代わりに、`add`メソッドを呼び出してください。`add`メソッドには、ユーザーが過去に入力したすべてのレスポンスを渡します。

```php
use function Laravel\Prompts\form;
use function Laravel\Prompts\outro;

$responses = form()
    ->text('What is your name?', required: true, name: 'name')
    ->add(function ($responses) {
        return text("How old are you, {$responses['name']}?");
    }, name: 'age')
    ->submit();

outro("Your name is {$responses['name']} and you are {$responses['age']} years old.");
```

<a name="informational-messages"></a>
## 情報メッセージ

`note`、`info`、`warning`、`error`、`alert`関数は、情報メッセージを表示するために使用します。

```php
use function Laravel\Prompts\info;

info('Package installed successfully.');
```

<a name="tables"></a>
## テーブル

`table`関数を使うと、複数の行や列のデータを簡単に表示できます。指定する必要があるのは、カラム名とテーブルのデータだけです。

```php
use function Laravel\Prompts\table;

table(
    headers: ['Name', 'Email'],
    rows: User::all(['name', 'email'])->toArray()
);
```

<a name="spin"></a>
## スピン

`spin`関数は、指定したコールバックを実行している間、オプションのメッセージとともにスピナーを表示します。これは進行中の処理を示す役割を果たし、完了するとコールバックの結果を返します。

```php
use function Laravel\Prompts\spin;

$response = spin(
    message: 'Fetching response...',
    callback: fn () => Http::get('http://example.com')
);
```

> [!WARNING]
> `spin`関数でスピナーをアニメーションするために、`pcntl` PHP拡張モジュールが必要です。この拡張モジュールが利用できない場合は、代わりに静的なスピナーが表示されます。

<a name="progress"></a>
## プログレスバー

実行に長時間かかるタスクの場合、タスクの完了度をユーザーに知らせるプログレスバーを表示すると便利です。`progress`関数を使用すると、Laravelはプログレスバーを表示し、指定する反復可能な値を繰り返し処理するごとに進捗を進めます。

```php
use function Laravel\Prompts\progress;

$users = progress(
    label: 'Updating users',
    steps: User::all(),
    callback: fn ($user) => $this->performTask($user)
);
```

`progress`関数はマップ関数のように動作し、コールバックの各繰り返しの戻り値を含む配列を返します。

このコールバックは、`Laravel\Prompts\Progress`インスタンスも受け取り可能で、繰り返しごとにラベルとヒントを修正できます。

```php
$users = progress(
    label: 'Updating users',
    steps: User::all(),
    callback: function ($user, $progress) {
        $progress
            ->label("Updating {$user->name}")
            ->hint("Created on {$user->created_at}");

        return $this->performTask($user);
    },
    hint: 'This may take some time.'
);
```

プログレス・バーの進め方を手作業でコントロールする必要がある場合があります。まず、プロセスが反復処理するステップの総数を定義します。そして、各アイテムを処理した後に`advance`メソッドでプログレスバーを進めます。

```php
$progress = progress(label: 'Updating users', steps: 10);

$users = User::all();

$progress->start();

foreach ($users as $user) {
    $this->performTask($user);

    $progress->advance();
}

$progress->finish();
```

<a name="clear"></a>
## ターミナルのクリア

`clear`関数はユーザーのターミナルを消去するために使用します。

```
use function Laravel\Prompts\clear;

clear();
```

<a name="terminal-considerations"></a>
## ターミナルの考察

<a name="terminal-width"></a>
#### ターミナルの横幅

ラベルやオプション、バリデーションメッセージの長さが、ユーザーの端末の「列」の文字数を超える場合、自動的に切り捨てます。ユーザーが狭い端末を使う可能性がある場合は、これらの文字列の長さを最小限にする検討をしてください。一般的に安全な最大長は、８０文字の端末をサポートする場合、７４文字です。

<a name="terminal-height"></a>
#### ターミナルの高さ

`scroll`引数を受け入れるプロンプトの場合、設定済み値は、検証メッセージ用のスペースを含めて、ユーザーの端末の高さに合わせて自動的に縮小されます。

<a name="fallbacks"></a>
## 未サポートの環境とフォールバック

Laravel PromptsはmacOS、Linux、WindowsのWSLをサポートしています。Windows版のPHPの制限により、現在のところWSL以外のWindowsでは、Laravel Promptsを使用できません。

このため、Laravel Promptsは[Symfony Console Question Helper](https://symfony.com/doc/7.0/components/console/helpers/questionhelper.html)のような代替実装のフォールバックをサポートしています。

> [!NOTE]
> LaravelフレームワークでLaravel Promptsを使用する場合、各プロンプトのフォールバックが設定済みで、未サポートの環境では自動的に有効になります。

<a name="fallback-conditions"></a>
#### フォールバックの条件

Laravelを使用していない場合や、フォールバック動作をカスタマイズする必要がある場合は、`Prompt`クラスの`fallbackWhen`へブール値を渡してください。

```php
use Laravel\Prompts\Prompt;

Prompt::fallbackWhen(
    ! $input->isInteractive() || windows_os() || app()->runningUnitTests()
);
```

<a name="fallback-behavior"></a>
#### フォールバックの振る舞い

Laravelを使用していない場合や、フォールバックの動作をカスタマイズする必要がある場合は、各プロンプトクラスの`fallbackUsing`静的メソッドへクロージャを渡してください。

```php
use Laravel\Prompts\TextPrompt;
use Symfony\Component\Console\Question\Question;
use Symfony\Component\Console\Style\SymfonyStyle;

TextPrompt::fallbackUsing(function (TextPrompt $prompt) use ($input, $output) {
    $question = (new Question($prompt->label, $prompt->default ?: null))
        ->setValidator(function ($answer) use ($prompt) {
            if ($prompt->required && $answer === null) {
                throw new \RuntimeException(
                    is_string($prompt->required) ? $prompt->required : 'Required.'
                );
            }

            if ($prompt->validate) {
                $error = ($prompt->validate)($answer ?? '');

                if ($error) {
                    throw new \RuntimeException($error);
                }
            }

            return $answer;
        });

    return (new SymfonyStyle($input, $output))
        ->askQuestion($question);
});
```

フォールバックは、プロンプトクラスごとに個別に設定する必要があります。クロージャはプロンプトクラスのインスタンスを受け取り、 プロンプトの適切な型を返す必要があります。
