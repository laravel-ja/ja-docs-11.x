# Artisanコンソール

- [イントロダクション](#introduction)
    - [Tinker（REPL）](#tinker)
- [コマンド記述](#writing-commands)
    - [コマンド生成](#generating-commands)
    - [コマンド構造](#command-structure)
    - [クロージャコマンド](#closure-commands)
    - [単一インスタンスコマンド](#isolatable-commands)
- [入力期待値の定義](#defining-input-expectations)
    - [引数](#arguments)
    - [オプション](#options)
    - [入力配列](#input-arrays)
    - [入力の説明](#input-descriptions)
    - [入力不足のプロンプト](#prompting-for-missing-input)
- [コマンドI／O](#command-io)
    - [入力の取得](#retrieving-input)
    - [入力のプロンプト](#prompting-for-input)
    - [出力の書き込み](#writing-output)
- [コマンド登録](#registering-commands)
- [プログラムからのコマンド実行](#programmatically-executing-commands)
    - [他コマンドからのコマンド呼び出し](#calling-commands-from-other-commands)
- [シグナルの処理](#signal-handling)
- [スタブのカスタマイズ](#stub-customization)
- [イベント](#events)

<a name="introduction"></a>
## イントロダクション

ArtisanはLaravelが用意しているコマンドラインインターフェイスです。Artisanは、アプリケーションのルートに`artisan`スクリプトとして存在し、アプリケーションの構築に役立つコマンドを多数提供しています。使用可能なすべてのArtisanコマンドのリストを表示するには、`list`コマンドを使用してください。

```shell
php artisan list
```

すべてのコマンドには、コマンドで使用可能な引数とオプションを表示および説明する「ヘルプ」画面も含まれています。ヘルプ画面を表示するには、コマンド名の前に「help」を付けます。

```shell
php artisan help migrate
```

<a name="laravel-sail"></a>
#### Laravel Sail

ローカル開発環境として[Laravel Sail](/docs/{{version}}/sail)を使用している場合は、必ず`sail`コマンドラインを使用してArtisanコマンドを呼び出してください。Sailは、アプリケーションのDockerコンテナ内でArtisanコマンドを実行します。

```shell
./vendor/bin/sail artisan list
```

<a name="tinker"></a>
### Tinker（REPL）

Laravel Tinkerは、[PsySH](https://github.com/bobthecow/psysh)パッケージを搭載したLaravelフレームワークの強力なREPLです。

<a name="installation"></a>
#### インストール

すべてのLaravelアプリケーションにはデフォルトでTinkerが含まれています。ただし、以前にアプリケーションからTinkerを削除した場合は、Composerを使用してTinkerをインストールできます。

```shell
composer require laravel/tinker
```

> [!NOTE]
> Laravelアプリケーションを操作するときに、ホットリロード、複数行のコード編集、自動補完をお求めですか？[Tinkerwell](https://tinkerwell.app)をチェックしてみてください！

<a name="usage"></a>
#### 使用法

Tinkerを使用すると、Eloquentモデル、ジョブ、イベントなどを含む、コマンドラインでLaravelアプリケーション全体を操作できます。Tinker環境に入るには、`tinker`Artisanコマンドを実行します。

```shell
php artisan tinker
```

`vendor:publish`コマンドを使用してTinkerの設定ファイルをリソース公開できます。

```shell
php artisan vendor:publish --provider="Laravel\Tinker\TinkerServiceProvider"
```

> [!WARNING]
> `dispatch`ヘルパ関数と`Dispatchable`クラスの`dispatch`メソッドは、ジョブをキューに投入するのにガベージコレクションへ依存しています。したがって、Tinkerを使う場合は、`Bus::dispatch`または`Queue::push`を使用してジョブをディスパッチする必要があります。

<a name="command-allow-list"></a>
#### コマンド許可リスト

Tinkerは「許可」リストを利用して、シェル内で実行できるArtisanコマンドを決定します。デフォルトでは、`clear-compiled`、`down`、`env`、`inspire`、`migrate`、`migrate:install`、`up`、`optimize`コマンドを実行できます。より多くのコマンドを許可したい場合は、それらを`tinker.php`設定ファイルの`commands`配列に追加してください。

    'commands' => [
        // App\Console\Commands\ExampleCommand::class,
    ],

<a name="classes-that-should-not-be-aliased"></a>
#### エイリアスするべきではないクラス

通常、Tinkerでクラスを操作すると、Tinkerはクラスへ自動的にエイリアスを付けます。ただし、一部のクラスをエイリアスしないことをお勧めします。これは、`tinker.php`設定ファイルの`dont_alias`配列にクラスをリストすることで実現できます。

    'dont_alias' => [
        App\Models\User::class,
    ],

<a name="writing-commands"></a>
## コマンド記述

Artisanが提供するコマンドに加え、独自のカスタムコマンドを作成することもできます。コマンドは通常、`app/Console/Commands`ディレクトリに保存します。ただし、Composerでコマンドをロードできる限り、独自の保存場所を自由に選択できます。

<a name="generating-commands"></a>
### コマンド生成

新しいコマンドを作成するには、`make:command` Artisanコマンドを使用します。このコマンドは、`app/Console/Commands`ディレクトリに新しいコマンドクラスを作成します。このディレクトリがアプリケーションに存在しなくても心配しないでください。`make:command`　Artisanコマンドを初めて実行したとき、このディレクトリを作成します。

```shell
php artisan make:command SendEmails
```

<a name="command-structure"></a>
### コマンド構造

コマンドを生成した後に、クラスの`signature`プロパティと`description`プロパティに適切な値を定義する必要があります。これらのプロパティは、`list`画面にコマンドを表示するときに使用されます。`signature`プロパティを使用すると、[コマンドの入力期待値](#defining-input-expectations)を定義することもできます。コマンドの実行時に`handle`メソッドが呼び出されます。このメソッドにコマンドロジックを配置できます。

コマンドの例を見てみましょう。コマンドの`handle`メソッドを介して必要な依存関係の注入を要求できることに注意してください。Laravel[サービスコンテナ](/docs/{{version}}/container)は、このメソッドの引数でタイプヒントされているすべての依存関係を自動的に注入します。

    <?php

    namespace App\Console\Commands;

    use App\Models\User;
    use App\Support\DripEmailer;
    use Illuminate\Console\Command;

    class SendEmails extends Command
    {
        /**
         * コンソールコマンドの名前と使い方
         *
         * @var string
         */
        protected $signature = 'mail:send {user}';

        /**
         * コンソールコマンドの説明
         *
         * @var string
         */
        protected $description = 'Send a marketing email to a user';

        /**
         * consoleコマンドの実行
         */
        public function handle(DripEmailer $drip): void
        {
            $drip->send(User::find($this->argument('user')));
        }
    }

> [!NOTE]
> コードの再利用性を上げるには、コンソールコマンドを軽くし、アプリケーションサービスに任せてタスクを実行することをお勧めします。上記の例では、電子メールを送信する「手間のかかる作業」を行うためにサービスクラスを挿入していることに注意してください。

<a name="exit-codes"></a>
#### 終了コード

`handle`メソッドが何も返さず、コマンドが正常に実行された場合、コマンドは`0`の終了コードで終了し、成功終了を示します。しかし、オプションとして`handle`メソッドは、コマンドの終了コードを指定するために、整数を返すこともできます。

    $this->error('Something went wrong.');

    return 1;

コマンド内のどのメソッドからでもコマンドを「失敗」させたい場合は、`fail`メソッドを利用してください。`fail`メソッドはコマンドの実行を即座に終了し、終了コード`1`を返します。

    $this->fail('Something went wrong.');

<a name="closure-commands"></a>
### クロージャコマンド

クロージャベースのコマンドは、コンソールコマンドをクラスとして定義する代替として提供しています。ルートクロージャがコントローラの代わりであるのと同様に、コマンドクロージャはコマンドクラスの代わりであると考えてください。

`routes/console.php`ファイルではHTTPルートを定義しませんが、アプリケーションのコンソールベースのエントリポイント(ルート)を定義しています。`Artisan::command`メソッドを使用して、このファイル内でクロージャベースのコンソールコマンドをすべて定義できます。`command`メソッドは２つ引数を取り、[コマンド使用定義](#defining-input-expectations)と、コマンドの引数とオプションを受け取るクロージャです。

    Artisan::command('mail:send {user}', function (string $user) {
        $this->info("Sending email to: {$user}!");
    });

クロージャは基になるコマンドインスタンスにバインドされているため、通常は完全なコマンドクラスでアクセスできるすべてのヘルパメソッドに完全にアクセスできます。

<a name="type-hinting-dependencies"></a>
#### タイプヒントの依存関係

コマンドの引数とオプションを受け取ることに加えて、コマンドクロージャは、[サービスコンテナ](/docs/{{version}}/container)により解決したい追加の依存関係をタイプヒントすることもできます。

    use App\Models\User;
    use App\Support\DripEmailer;

    Artisan::command('mail:send {user}', function (DripEmailer $drip, string $user) {
        $drip->send(User::find($user));
    });

<a name="closure-command-descriptions"></a>
#### クロージャコマンドの説明

クロージャベースのコマンドを定義するときは、`purpose`メソッドを使用してコマンドに説明を追加できます。この説明は、`php artisan list`または`php artisan help`コマンドを実行すると表示されます。

    Artisan::command('mail:send {user}', function (string $user) {
        // ...
    })->purpose('Send a marketing email to a user');

<a name="isolatable-commands"></a>
### 単一インスタンスコマンド

> [!WARNING]
> この機能を利用するには、アプリケーションで`memcached`、`redis`、`dynamodb`、`database`、`file`、`array`キャッシュドライバをアプリケーションのデフォルトキャッシュドライバとして使用する必要があります。さらに、すべてのサーバから同一のセントルキャッシュサーバと通信する必要があります。

同時に実行できるコマンドのインスタンスが１つだけであることを保証したい場合があります。これを実現するには、コマンドクラスで`Illuminate\Contracts\Console\Isolatable`インターフェイスを実装します。

    <?php

    namespace App\Console\Commands;

    use Illuminate\Console\Command;
    use Illuminate\Contracts\Console\Isolatable;

    class SendEmails extends Command implements Isolatable
    {
        // ...
    }

コマンドを`Isolatable`とマークすると、Laravel は自動的にコマンドに`--isolated`オプションを追加します。このオプションでコマンドを呼び出すと、Laravelはそのコマンドの他のインスタンスが既に実行されていないことを確認します。Laravelは、アプリケーションのデフォルトキャッシュドライバを使用してアトミックロックの取得を試みることで、この機能を実現します。コマンドの他のインスタンスが実行されている場合、コマンドは実行されませんが、コマンドは正常終了ステータスコードで終了します。

```shell
php artisan mail:send 1 --isolated
```

コマンドが実行できなかった場合に返される終了ステータスコードを指定したい場合は、`isolated`オプションで希望するステータスコードを指定できます。

```shell
php artisan mail:send 1 --isolated=12
```

<a name="lock-id"></a>
#### Lock ID

デフォルトでLaravelは、コマンド名を使用し、アプリケーションのキャッシュでアトミック・ロックを取得するために使用する文字列キーを生成します。しかし、Artisanコマンドクラスへ`isolatableId`メソッドを定義すれば、このキーをカスタマイズでき、コマンドの引数やオプションをキーへ統合できます。

```php
/**
 * コマンドの一意的なIDを取得
 */
public function isolatableId(): string
{
    return $this->argument('user');
}
```

<a name="lock-expiration-time"></a>
#### ロックの有効時間

デフォルトでは、単一コマンドのロックは、コマンド終了後に期限切れとなります。あるいは、コマンドが中断されて終了できなかった場合は、ロックは１時間後に期限切れになります。しかし、コマンドで`isolationLockExpiresAt`メソッドを定義すれば、ロックの有効期限を調整できます。

```php
use DateTimeInterface;
use DateInterval;

/**
 * 単一コマンドのロック期限を決定
 */
public function isolationLockExpiresAt(): DateTimeInterface|DateInterval
{
    return now()->addMinutes(5);
}
```

<a name="defining-input-expectations"></a>
## 入力期待値の定義

コンソールコマンドを作成する場合、引数またはオプションを介してユーザーからの入力を収集するのが一般的です。Laravelを使用すると、コマンドの`signature`プロパティを使用して、ユーザーへ期待する入力を定義するのが非常に便利になります。`signature`プロパティを使用すると、コマンドの名前、引数、およびオプションを表現力豊かなルートのように単一の構文で定義できます。

<a name="arguments"></a>
### 引数

ユーザーが指定できるすべての引数とオプションは、中括弧で囲います。次の例では、コマンドは１つの必須引数、`user`を定義します。

    /**
     * コンソールコマンドの名前と使い方
     *
     * @var string
     */
    protected $signature = 'mail:send {user}';

引数をオプションにしたり、引数のデフォルト値を定義したりもできます。

    // オプションの引数
    'mail:send {user?}'

    // オプションの引数とデフォルト値
    'mail:send {user=foo}'

<a name="options"></a>
### オプション

引数のようにオプションは、ユーザー入力の別の形態です。オプションは、コマンドラインで指定する場合、接頭辞として2つのハイフン(`--`)を付けます。オプションには、値を受け取るオプションと受け取らないオプションの２種類があります。値を受け取らないオプションは、論理「スイッチ」として機能します。このタイプのオプションの例を見てみましょう。

    /**
     * コンソールコマンドの名前と使い方
     *
     * @var string
     */
    protected $signature = 'mail:send {user} {--queue}';

この例では、Artisanコマンドを呼び出すときに`--queue`スイッチを指定しています。`--queue`スイッチが渡されると、オプションの値は`true`になります。それ以外の場合、値は`false`になります。

```shell
php artisan mail:send 1 --queue
```

<a name="options-with-values"></a>
#### 値を取るオプション

次に、値を期待するオプションを見てみましょう。ユーザーがオプションの値を指定する必要がある場合は、オプション名の末尾に「=」記号を付ける必要があります。

    /**
     * コンソールコマンドの名前と使い方
     *
     * @var string
     */
    protected $signature = 'mail:send {user} {--queue=}';

この例で、ユーザーはオプションとして値を渡すことができます。コマンドの起動時にオプションが指定されていない場合、その値は「null」になります。

```shell
php artisan mail:send 1 --queue=default
```

オプション名の後にデフォルト値を指定することにより、オプションにデフォルト値を割り当てられます。ユーザーからオプション値が指定されない場合は、デフォルト値を使用します。

    'mail:send {user} {--queue=default}'

<a name="option-shortcuts"></a>
#### オプションのショートカット

オプションを定義するときにショートカットを割り当てるには、オプション名の前にショートカットを指定し、`|`文字を区切り文字として使用してショートカットと完全なオプション名を分けます。

    'mail:send {user} {--Q|queue}'

ターミナルでコマンドを起動する場合、オプションのショートカットの先頭にハイフンを１つ付け、オプションの値を指定する際は`=`文字をつけないでください。

```shell
php artisan mail:send 1 -Qdefault
```

<a name="input-arrays"></a>
### 入力配列

複数の入力値を期待する引数またはオプションを定義する場合は、`*`文字を使用できます。まず、そのような引数を指定する例を見てみましょう。

    'mail:send {user*}'

このメソッドを呼び出すときは、コマンドラインへ順番に、`user`引数を渡せます。たとえば、次のコマンドは、`user`の値に`1`と`2`の値を配列に設定します。

```shell
php artisan mail:send 1 2
```

この`*`文字をオプションの引数定義と組み合わせ、引数を０個以上許可することができます。

    'mail:send {user?*}'

<a name="option-arrays"></a>
#### オプション配列

複数の入力値を期待するオプションを定義する場合、コマンドに渡す各オプション値には、オプション名のプレフィックスを付ける必要があります。

    'mail:send {--id=*}'

このコマンドは、複数の`--id`引数を渡し、起動できます。

```shell
php artisan mail:send --id=1 --id=2
```

<a name="input-descriptions"></a>
### 入力の説明

コロンを使用して引数名と説明を区切ることにより、入力引数とオプションに説明を割り当てることができます。コマンドを定義するために少し余分なスペースが必要な場合は、定義を自由に複数の行に分けてください。

    /**
     * コンソールコマンドの名前と使い方
     *
     * @var string
     */
    protected $signature = 'mail:send
                            {user : The ID of the user}
                            {--queue : Whether the job should be queued}';

<a name="prompting-for-missing-input"></a>
### 入力不足のプロンプト

コマンドが必須の引数を持っている場合に、その引数が指定されないとエラーメッセージを表示します。もしくは、`PromptsForMissingInput`インターフェイスを実装し、必要な引数がない場合に自動でユーザーにプロンプトを表示するように、コマンドを設定してください。

    <?php

    namespace App\Console\Commands;

    use Illuminate\Console\Command;
    use Illuminate\Contracts\Console\PromptsForMissingInput;

    class SendEmails extends Command implements PromptsForMissingInput
    {
        /**
         * コンソールコマンドの名前と使い方
         *
         * @var string
         */
        protected $signature = 'mail:send {user}';

        // ...
    }

Laravelが必要な引数をユーザーから収集する必要がある場合、引数名か説明のどちらかを使用し、インテリジェントに質問のフレーズを作り、自動的にユーザーへ尋ねます。必要な引数を収集するために使用する質問をカスタマイズしたい場合は、引数名をキーにした質問の配列を返す、`promptForMissingArgumentsUsing`メソッドを実装してください。

    /**
     * リターンする質問を使い、足りない入力引数をプロンプトする
     *
     * @return array<string, string>
     */
    protected function promptForMissingArgumentsUsing(): array
    {
        return [
            'user' => 'Which user ID should receive the mail?',
        ];
    }

質問とプレースホルダを含むタプルを使用して、プレースホルダ・テキストを提供することもできます。

    return [
        'user' => ['Which user ID should receive the mail?', 'E.g. 123'],
    ];

プロンプトを完全にコントロールしたい場合は、ユーザーへプロンプトを表示し、その答えを返すクロージャを指定してください。

    use App\Models\User;
    use function Laravel\Prompts\search;

    // ...

    return [
        'user' => fn () => search(
            label: 'Search for a user:',
            placeholder: 'E.g. Taylor Otwell',
            options: fn ($value) => strlen($value) > 0
                ? User::where('name', 'like', "%{$value}%")->pluck('name', 'id')->all()
                : []
        ),
    ];

> [!NOTE]
包括的な[Laravel Prompts](/docs/{{version}}/prompts)ドキュメントに、利用可能なプロンプトとその使用方法に関する追加情報を用意してあります。

[オプション](#options)の選択や入力をユーザーに促したい場合は、コマンドの`handle`メソッドにプロンプトを含めてください。引数が足りない場合にのみプロンプトを出したい場合は、`afterPromptingForMissingArguments`メソッドを実装してください：

    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Output\OutputInterface;
    use function Laravel\Prompts\confirm;

    // ...

    /**
     * 足りない引数をユーザーへ促した後に実行するアクション
     */
    protected function afterPromptingForMissingArguments(InputInterface $input, OutputInterface $output): void
    {
        $input->setOption('queue', confirm(
            label: 'Would you like to queue the mail?',
            default: $this->option('queue')
        ));
    }

<a name="command-io"></a>
## コマンドI／O

<a name="retrieving-input"></a>
### 入力の取得

コマンドの実行中に、コマンドが受け入れた引数とオプションの値にアクセスする必要があるでしょう。これには、`argument`メソッドと`option`メソッドを使用します。引数またはオプションが存在しない場合、`null`が返されます。

    /**
     * consoleコマンドの実行
     */
    public function handle(): void
    {
        $userId = $this->argument('user');
    }

すべての引数を`array`として取得する必要がある場合は、`arguments`メソッドを呼び出します。

    $arguments = $this->arguments();

オプションは、`option`メソッドを使用して引数と同じように簡単に取得できます。すべてのオプションを配列として取得するには、`options`メソッドを呼び出します。

    // 特定のオプションを取得
    $queueName = $this->option('queue');

    // すべてのオプションを配列として取得します
    $options = $this->options();

<a name="prompting-for-input"></a>
### 入力のプロンプト

> [!NOTE]
> [Laravel Prompts](/docs/{{version}}/prompts) は、美しくユーザーフレンドリーなUIをコマンドラインアプリケーションに追加するためのPHPパッケージで、プレースホルダテキストやバリデーションなどのブラウザライクな機能を備えています。

出力の表示に加えて、コマンドの実行中にユーザーへ入力を提供するように依頼することもできます。`ask`メソッドはユーザーへ、指定した質問をプロンプ​​トとして表示し、入力を受け取り、ユーザー入力をコマンドに戻します。

    /**
     * consoleコマンドの実行
     */
    public function handle(): void
    {
        $name = $this->ask('What is your name?');

        // ...
    }

`ask`メソッドは、第２引数もオプションで引数に取ります。この第２引数には、ユーザー入力がない場合に返すデフォルト値を指定します。

    $name = $this->ask('What is your name?', 'Taylor');

`secret`メソッドは`ask`に似ていますが、ユーザーがコンソールに入力するときに、ユーザーの入力は表示されません。この方法は、パスワードなどの機密情報をリクエストするときに役立ちます。

    $password = $this->secret('What is the password?');

<a name="asking-for-confirmation"></a>
#### 確認を求める

単純な「はい」か「いいえ」の確認をユーザーに求める必要がある場合は、「confirm」メソッドを使用します。デフォルトでは、このメソッドは`false`を返します。ただし、ユーザーがプロンプトに応答して「y」または「yes」を入力すると、メソッドは「true」を返します。

    if ($this->confirm('Do you wish to continue?')) {
        // ...
    }

必要に応じて、`confirm`メソッドの２番目の引数として`true`を渡すことにより、確認プロンプトがデフォルトで`true`を返すように指定できます。

    if ($this->confirm('Do you wish to continue?', true)) {
        // ...
    }

<a name="auto-completion"></a>
#### オートコンプリート

`anticipate`メソッドを使用して、可能な選択肢のオートコンプリートを提供できます。オートコンプリートのヒントに関係なく、ユーザーは引き続き任意の回答ができます。

    $name = $this->anticipate('What is your name?', ['Taylor', 'Dayle']);

もしくは`anticipate`メソッドの２番目の引数としてクロージャを渡すこともできます。クロージャは、ユーザーが入力文字を入力するたびに呼び出されます。クロージャは、これまでのユーザーの入力を含む文字列パラメータを受け取り、オートコンプリートのオプションの配列を返す必要があります。

    $name = $this->anticipate('What is your address?', function (string $input) {
        // オートコンプリートオプションを返す…
    });

<a name="multiple-choice-questions"></a>
#### 複数の選択肢の質問

質問をするときにユーザーに事前定義された選択肢のセットを提供する必要がある場合は、`choice`メソッドを使用します。メソッドに3番目の引数としてインデックスを渡すことにより、オプションが選択されていない場合に返されるデフォルト値の配列インデックスを設定できます。

    $name = $this->choice(
        'What is your name?',
        ['Taylor', 'Dayle'],
        $defaultIndex
    );

さらに、`choice`メソッドは、有効な応答を選択するための最大試行回数と、複数の選択が許可されるかどうかを決定するために、オプションとして４番目と５番目の引数を取ります。

    $name = $this->choice(
        'What is your name?',
        ['Taylor', 'Dayle'],
        $defaultIndex,
        $maxAttempts = null,
        $allowMultipleSelections = false
    );

<a name="writing-output"></a>
### 出力の書き込み

コンソールに出力を送信するには、`line`、`info`、`comment`、`question`、`warn`、`error`メソッドを使用できます。これらの各メソッドは、目的に応じて適切なANSIカラーを使用します。たとえば、一般的な情報をユーザーに表示してみましょう。通常、`info`メソッドはコンソールに緑色のテキストを表示します。

    /**
     * consoleコマンドの実行
     */
    public function handle(): void
    {
        // ...

        $this->info('The command was successful!');
    }

エラーメッセージを表示するには、`error`メソッドを使用します。エラーメッセージのテキストは通常​​、赤で表示します。

    $this->error('Something went wrong!');

`line`メソッドを使用して、色のないプレーンなテキストを表示できます。

    $this->line('Display this on the screen');

`newLine`メソッドを使用して空白行を表示できます。

    // 空白行を1行書く
    $this->newLine();

    // 空白行を３行書く
    $this->newLine(3);

<a name="tables"></a>
#### テーブル

`table`メソッドを使用すると、データの複数の行／列を簡単に正しくフォーマットできます。
テーブルの列名とデータを入力するだけで、Laravelがテーブルの適切な幅と高さを自動的に計算します。

    use App\Models\User;

    $this->table(
        ['Name', 'Email'],
        User::all(['name', 'email'])->toArray()
    );

<a name="progress-bars"></a>
#### プログレスバー

長時間実行されるタスクの場合、タスクの完了度をユーザーに通知する進行状況バーを表示すると便利です。`withProgressBar`メソッドを使用するとLaravelは、進行状況バーを表示し、指定した反復可能値を反復するごとにその進捗を進めます。

    use App\Models\User;

    $users = $this->withProgressBar(User::all(), function (User $user) {
        $this->performTask($user);
    });

場合によっては、プログレスバーの進め方を手作業で制御する必要があります。最初に、プロセスが繰り返すステップの総数を定義します。次に、各アイテムを処理した後、プログレスバーを進めます。

    $users = App\Models\User::all();

    $bar = $this->output->createProgressBar(count($users));

    $bar->start();

    foreach ($users as $user) {
        $this->performTask($user);

        $bar->advance();
    }

    $bar->finish();

> [!NOTE]
> より高度なオプションについては、[Symfonyのプログレスバーコンポーネントのドキュメント](https://symfony.com/doc/7.0/components/console/helpers/progressbar.html)をご覧ください。

<a name="registering-commands"></a>
## コマンド登録

Laravelはデフォルトで、自動的に`app/Console/Commands`ディレクトリ内の全てのコマンドを登録します。しかし、アプリケーションの`bootstrap/app.php`ファイル内の`withCommands`メソッドを使用すれば、他のディレクトリをスキャンし、Artisanコマンドを探すようLaravelに指示できます。

    ->withCommands([
        __DIR__.'/../app/Domain/Orders/Commands',
    ])

必要であれば、`withCommands`メソッドへコマンドのクラス名を与え、手作業でコマンドを登録することもできます。

    use App\Domain\Orders\Commands\SendEmails;

    ->withCommands([
        SendEmails::class,
    ])

 Artisanが起動すると、アプリケーション内のすべてのコマンドが[サービスコンテナ](/docs/{{version}}/container)によって解決され、Artisanに登録されます。

<a name="programmatically-executing-commands"></a>
## プログラムからのコマンド実行

CLIの外部でArtisanコマンドを実行したい場合があります。たとえば、ルートまたはコントローラからArtisanコマンドを実行したい場合があります。これを実現するには、`Artisan`ファサードの`call`メソッドを使用します。`call`メソッドは、最初の引数としてコマンドの名前またはクラス名のいずれかを受け入れ、２番目の引数としてコマンドパラメータの配列を取ります。終了コードが返されます:

    use Illuminate\Support\Facades\Artisan;

    Route::post('/user/{user}/mail', function (string $user) {
        $exitCode = Artisan::call('mail:send', [
            'user' => $user, '--queue' => 'default'
        ]);

        // ...
    });

または、Artisanコマンド全体を文字列として`call`メソッドに渡すこともできます。

    Artisan::call('mail:send 1 --queue=default');

<a name="passing-array-values"></a>
#### 配列値の受け渡し

コマンドで配列を受け入れるオプションを定義している場合は、値の配列をそのオプションに渡すことができます。

    use Illuminate\Support\Facades\Artisan;

    Route::post('/mail', function () {
        $exitCode = Artisan::call('mail:send', [
            '--id' => [5, 13]
        ]);
    });

<a name="passing-boolean-values"></a>
#### 論理値値の受け渡し

`migrate:refresh`コマンドの`--force`フラグなど、文字列値を受け入れないオプションの値を指定する必要がある場合は、そのオプションの値として`true`または`false`を渡してください。

    $exitCode = Artisan::call('migrate:refresh', [
        '--force' => true,
    ]);

<a name="queueing-artisan-commands"></a>
#### Artisanコマンドのキュー投入

`Artisan`ファサードで`queue`メソッドを使用すると、Artisanコマンドをキューに投入し、[キューワーカー](/docs/{{version}}/queues)によりバックグラウンド処理することもできます。この方法を使用する前に、確実にキューを設定し、キューリスナを実行してください。

    use Illuminate\Support\Facades\Artisan;

    Route::post('/user/{user}/mail', function (string $user) {
        Artisan::queue('mail:send', [
            'user' => $user, '--queue' => 'default'
        ]);

        // ...
    });

`onConnection`および`onQueue`メソッドを使用して、Artisanコマンドをディスパッチする接続やキューを指定できます。

    Artisan::queue('mail:send', [
        'user' => 1, '--queue' => 'default'
    ])->onConnection('redis')->onQueue('commands');

<a name="calling-commands-from-other-commands"></a>
### 他コマンドからのコマンド呼び出し

Artisanコマンドから他のコマンドを呼び出したい場合があります。`call`メソッドを使用してこれを行うことができます。この`call`メソッドは、コマンド名とコマンド引数/オプションの配列を受け入れます。

    /**
     * consoleコマンドの実行
     */
    public function handle(): void
    {
        $this->call('mail:send', [
            'user' => 1, '--queue' => 'default'
        ]);

        // ...
    }

別のコンソールコマンドを呼び出してその出力をすべて抑制したい場合は、`callSilently`メソッドを使用できます。`callSilently`メソッドは`call`メソッドと同じ使い方です。

    $this->callSilently('mail:send', [
        'user' => 1, '--queue' => 'default'
    ]);

<a name="signal-handling"></a>
## シグナルの処理

ご存知のように、オペレーティングシステムでは、実行中のプロセスにシグナルを送ることができます。例えば、`SIGTERM`シグナルはオペレーティングシステムがプログラムを終了するように要求する方法です。もし、Artisanコンソールコマンドでシグナルをリッスンし、発生時にコードを実行したい場合は、`trap`メソッドを使用してください。

    /**
     * consoleコマンドの実行
     */
    public function handle(): void
    {
        $this->trap(SIGTERM, fn () => $this->shouldKeepRunning = false);

        while ($this->shouldKeepRunning) {
            // ...
        }
    }

一度に複数のシグナルをリッスンするには、`trap`メソッドへシグナルの配列を指定してください。

    $this->trap([SIGTERM, SIGQUIT], function (int $signal) {
        $this->shouldKeepRunning = false;

        dump($signal); // SIGTERM / SIGQUIT
    });

<a name="stub-customization"></a>
## スタブのカスタマイズ

Artisanコンソールの`make`コマンドは、コントローラ、ジョブ、マイグレーション、テストなどのさまざまなクラスを作成するために使用されます。これらのクラスは、入力に基づいた値が挿入される「スタブ」ファイルを使用して生成されます。ただし、Artisanにより生成されるファイルへ小さな変更を加えることをおすすめします。このためには、`stub:publish`コマンドを使用して、最も一般的なスタブをアプリケーションにリソース公開し、カスタマイズできるようにします。

```shell
php artisan stub:publish
```

リソース公開するスタブは、アプリケーションのルートの`stubs`ディレクトリ内へ設置します。これらのスタブに加えた変更は、Artisanの`make`コマンドを使用して対応するクラスを生成するときに反映されます。

<a name="events"></a>
## イベント

Artisanは、コマンドの実行時に、`Illuminate\Console\Events\ArtisanStarting`、`Illuminate\Console\Events\CommandStarting`、および`Illuminate\Console\Events\CommandFinished`の3つのイベントをディスパッチします。`ArtisanStarting`イベントは、Artisanが実行を開始するとすぐにディスパッチされます。次に、コマンドが実行される直前に`CommandStarting`イベントがディスパッチされます。最後に、コマンドの実行が終了すると、`CommandFinished`イベントがディスパッチされます。
