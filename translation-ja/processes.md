# プロセス

- [イントロダクション](#introduction)
- [プロセスの起動](#invoking-processes)
    - [プロセスのオプション](#process-options)
    - [プロセス出力](#process-output)
    - [パイプライン](#process-pipelines)
- [非同期プロセス](#asynchronous-processes)
    - [プロセスIDとシグナル](#process-ids-and-signals)
    - [非同期プロセス出力](#asynchronous-process-output)
- [同時実行プロセス](#concurrent-processes)
    - [名前付きプールアクセス](#naming-pool-processes)
    - [プールプロセスIDとシグナル](#pool-process-ids-and-signals)
- [テスト](#testing)
    - [プロセスのFake](#faking-processes)
    - [特定プロセスのFake](#faking-specific-processes)
    - [プロセス順序のFake](#faking-process-sequences)
    - [非同期プロセスライフサイクルのFake](#faking-asynchronous-process-lifecycles)
    - [利用可能なアサート](#available-assertions)
    - [未指定プロセスの防止](#preventing-stray-processes)

<a name="introduction"></a>
## イントロダクション

Laravelは、[Symfonyプロセスコンポーネント](https://symfony.com/doc/7.0/components/process.html)の周りに表現力豊かで最小限のAPIを提供し、Laravelアプリケーションから外部プロセスを便利に呼び出せるようにしています。Laravelのプロセス機能は、最も一般的なユースケースと素晴らしい開発者体験に傾注しています。

<a name="invoking-processes"></a>
## プロセスの起動

プロセスを起動するには、`Process`ファサードが提供している`run`メソッドと`start`メソッドを利用します。`run` メソッドはプロセスを呼び出して、プロセスの実行が終了するのを待ちます。一方の`start`メソッドは非同期プロセスの実行に使用します。このドキュメントでは、両方のアプローチを検討します。まず、基本的な同期プロセスを起動し、その結果を確認する方法を調べましょう。

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

return $result->output();
```

もちろん、`run`メソッドが返す、`Illuminate\Contracts\Process\ProcessResult`インスタンスは、プロセスの結果を調べるために使用できる多くの有用なメソッドを用意しています。

```php
$result = Process::run('ls -la');

$result->successful();
$result->failed();
$result->exitCode();
$result->output();
$result->errorOutput();
```

<a name="throwing-exceptions"></a>
#### 例外を投げる

プロセス結果が出て、終了コードが０より大きい（つまり失敗を表している）場合に、`Illuminate\Process\Exceptions\ProcessFailedException`のインスタンスを投げたい場合は、`throw`と`throwIf`メソッドを使用してください。プロセスが失敗しなかった場合、プロセス結果のインスタンスを返します。

```php
$result = Process::run('ls -la')->throw();

$result = Process::run('ls -la')->throwIf($condition);
```

<a name="process-options"></a>
### プロセスのオプション

もちろん、プロセス起動の前に、そのプロセスの動作をカスタマイズする必要がある場合もあるでしょう。幸福なことに、Laravelでは、作業ディレクトリ、タイムアウト、環境変数など、様々なプロセス機能を調整できます。

<a name="working-directory-path"></a>
#### 作業ディレクトリパス

プロセスの作業ディレクトリを指定するには、`path`メソッドを使用します。このメソッドを呼び出さない場合、プロセスは現在実行中のPHPスクリプトの作業ディレクトリを引き継ぎます。

```php
$result = Process::path(__DIR__)->run('ls -la');
```

<a name="input"></a>
#### 入力

入力は、`input`メソッドを使い、プロセスの「標準入力」から行えます。

```php
$result = Process::input('Hello World')->run('cat');
```

<a name="timeouts"></a>
#### タイムアウト

デフォルトでプロセスは６０秒以上実行されると、`Illuminate\Process\Exceptions\ProcessTimedOutException`のインスタンスを投げます。しかし、`timeout`メソッドでこの動作をカスタマイズできます。

```php
$result = Process::timeout(120)->run('bash import.sh');
```

もしくは、プロセスのタイムアウトを完全に無効にしたい場合は、`forever`メソッドを呼びだしてください。

```php
$result = Process::forever()->run('bash import.sh');
```

`idleTimeout`メソッドを使用すると、出力をまったく返さずにプロセスを実行できる最大秒数を指定できます。

```php
$result = Process::timeout(60)->idleTimeout(30)->run('bash import.sh');
```

<a name="environment-variables"></a>
#### 環境変数

環境変数は、`env`メソッドでプロセスへ指定できます。起動したプロセスはシステムで定義したすべての環境変数を継承します。

```php
$result = Process::forever()
    ->env(['IMPORT_PATH' => __DIR__])
    ->run('bash import.sh');
```

呼び出したプロセスから継承した環境変数を削除したい場合は、その環境変数に`false`値を指定してください。

```php
$result = Process::forever()
    ->env(['LOAD_PATH' => false])
    ->run('bash import.sh');
```

<a name="tty-mode"></a>
#### TTYモード

`tty`メソッドを使用すると、プロセスのTTYモードを有効にできます。TTYモードはプロセスの入出力をプログラムの入出力と接続し、プロセスがVimやNanoのようなエディタをプロセスとして開くことができるようにします。

```php
Process::forever()->tty()->run('vim');
```

<a name="process-output"></a>
### プロセス出力

前の説明の通り、プロセスの出力には、プロセス結果の`output`(stdout)と`errorOutput`(stderr)メソッドを使用してアクセスできます。

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

echo $result->output();
echo $result->errorOutput();
```

しかし、`run`メソッドの第２引数へクロージャを渡すと、リアルタイムに出力を収集することもできます。クロージャは２つの引数を取ります。出力の「タイプ」（`stdout`または`stderr`）と出力文字列そのものです。

```php
$result = Process::run('ls -la', function (string $type, string $output) {
    echo $output;
});
```

Laravelには、`seeInOutput`と`seeInErrorOutput`メソッドもあり、指定文字列がプロセスの出力に含まれているかを判断できる、便利な方法を提供しています。

```php
if (Process::run('ls -la')->seeInOutput('laravel')) {
    // …
}
```

<a name="disabling-process-output"></a>
#### プロセス出力の無効化

もし、あなたのプロセスが興味のない出力を大量に書き出すものであるなら、出力の取得を完全に無効化し、メモリを節約できます。これを行うには、プロセスをビルドするときに、`quietly`メソッドを呼び出してください。

```php
use Illuminate\Support\Facades\Process;

$result = Process::quietly()->run('bash import.sh');
```

<a name="process-pipelines"></a>
### パイプライン

あるプロセスの出力を別のプロセスの入力にしたい場合があると思います。これはよく、あるプロセスの出力を別のプロセスへ"パイプ"すると呼んでいます。`Process`ファサードが提供する、`pipe`メソッドで、これを簡単に実現できます。`pipe`メソッドは、パイプラインで接続したプロセスを同時に実行し、パイプラインの最後のプロセスの結果を返します。

```php
use Illuminate\Process\Pipe;
use Illuminate\Support\Facades\Process;

$result = Process::pipe(function (Pipe $pipe) {
    $pipe->command('cat example.txt');
    $pipe->command('grep -i "laravel"');
});

if ($result->successful()) {
    // …
}
```

パイプラインを構成する個々のプロセスをカスタマイズする必要がない場合は、`pipe`メソッドにコマンド文字列の配列を渡すだけです。

```php
$result = Process::pipe([
    'cat example.txt',
    'grep -i "laravel"',
]);
```

`pipe`メソッドの第２引数へクロージャを渡すと、プロセスの出力をリアルタイムに収集できます。クロージャは２引数を取ります。出力の「タイプ」（`stdout`か`stderr`）と、出力文字列そのものです：

```php
$result = Process::pipe(function (Pipe $pipe) {
    $pipe->command('cat example.txt');
    $pipe->command('grep -i "laravel"');
}, function (string $type, string $output) {
    echo $output;
});
```

Laravelでは、パイプライン内の各プロセスに、`as`メソッドで文字列キーを割り当てることもできます。このキーは、`pipe`メソッドへ指定する出力クロージャにも渡され、出力がどのプロセスに属しているかを判定できます：

```php
$result = Process::pipe(function (Pipe $pipe) {
    $pipe->as('first')->command('cat example.txt');
    $pipe->as('second')->command('grep -i "laravel"');
})->start(function (string $type, string $output, string $key) {
    // …
});
```

<a name="asynchronous-processes"></a>
## 非同期プロセス

`run`メソッドは同期的にプロセスを起動する一方で、`start`メソッドは非同期的にプロセスを起動するため使用します。これにより、プロセスをバックグラウンドで実行している間、アプリケーションは他のタスクを実行し続けられます。プロセスを起動したら、`running`メソッドを使用して、プロセスがまだ実行されているかを判定できます。

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    // …
}

$result = $process->wait();
```

お気づきでしょうが、`wait`メソッドを呼び出し、プロセスの実行終了を待ち、プロセスの実行結果インスタンスを取得できます。

```php
$process = Process::timeout(120)->start('bash import.sh');

// ...

$result = $process->wait();
```

<a name="process-ids-and-signals"></a>
### プロセスIDとシグナル

`id`メソッドを使えば、オペレーティングシステムが割り当てた、実行中のプロセスのプロセスIDを取得できます。

```php
$process = Process::start('bash import.sh');

return $process->id();
```

実行中のプロセスへ、「シグナル」を送るには、`signal`メソッドを使用します。定義済みのシグナル定数の一覧は、[PHPドキュメント](https://www.php.net/manual/ja/pcntl.constants.php)に記載されています。

```php
$process->signal(SIGUSR2);
```

<a name="asynchronous-process-output"></a>
### 非同期プロセス出力

非同期プロセスの実行中は、`output`メソッドと`errorOutput`メソッドを使用して、現在の出力全部へアクセスできます。しかし、 `latestOutput`と`latestErrorOutput`を使用すれば、最後に出力を取得した後に発生したプロセス出力にアクセスできます。

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    echo $process->latestOutput();
    echo $process->latestErrorOutput();

    sleep(1);
}
```

`run`メソッドと同様、`start`メソッドの第２引数にクロージャを渡せば、非同期プロセスからリアルタイムに出力を収集できます。クロージャは２つの引数を受け取ります。出力の「タイプ」（`stdout`または`stderr`）と出力文字列そのものです。

```php
$process = Process::start('bash import.sh', function (string $type, string $output) {
    echo $output;
});

$result = $process->wait();
```

 プロセスが終了するまで待つ代わりに、`waitUntil`メソッドを使用すると、プロセスの出力に基づき待ちをやめられます。Laravelは、`waitUntil`メソッドで指定するクロージャが`true`を返すと、プロセスの終了を待つのをやめます。

```php
$process = Process::start('bash import.sh');

$process->waitUntil(function (string $type, string $output) {
    return $output === 'Ready...';
});
```

<a name="concurrent-processes"></a>
## 同時実行プロセス

またLaravelでは、同時実行の非同期プロセスのプールを簡単に管理でき、多くのタスクを簡単に同時実行できます。使用するには、`pool`メソッドを呼び出してください。これは、`Illuminate\Process\Pool`のインスタンスを受け取るクロージャを引数に取ります。

このクロージャの中で、プールに所属するプロセスを定義してください。プロセスプールを`start`メソッドで開始すれば、`running`メソッドにより実行中のプロセスの[コレクション](/docs/{{version}}/collections)にアクセスできるようになります。

```php
use Illuminate\Process\Pool;
use Illuminate\Support\Facades\Process;

$pool = Process::pool(function (Pool $pool) {
    $pool->path(__DIR__)->command('bash import-1.sh');
    $pool->path(__DIR__)->command('bash import-2.sh');
    $pool->path(__DIR__)->command('bash import-3.sh');
})->start(function (string $type, string $output, int $key) {
    // …
});

while ($pool->running()->isNotEmpty()) {
    // …
}

$results = $pool->wait();
```

ご覧のように、プール内のすべてのプロセスの実行が終了するのを待ち、その結果を`wait`メソッドで取得できます。`wait`メソッドは、プール内の各プロセスのプロセス結果のインスタンスへキーでアクセスできる、配列アクセス可能なオブジェクトを返します。

```php
$results = $pool->wait();

echo $results[0]->output();
```

もしくは便利な方法として、`concurrently`メソッドを使用し、非同期プロセスプールを起動し、その結果をすぐに待つこともできます。これは、PHPの配列分解機能と組み合わせると、特に表現力豊かな構文を利用できます。

```php
[$first, $second, $third] = Process::concurrently(function (Pool $pool) {
    $pool->path(__DIR__)->command('ls -la');
    $pool->path(app_path())->command('ls -la');
    $pool->path(storage_path())->command('ls -la');
});

echo $first->output();
```

<a name="naming-pool-processes"></a>
### 名前付きプールアクセス

プロセスプールの結果に数値キーによりアクセスするのは表現力が乏しいので、Laravelでは`as`メソッドを使用し、プール内の各プロセスに文字列キーを割り当てできます。このキーは、`start`メソッドへ指定するクロージャにも渡され、出力がどのプロセスに属しているかを判断できます。

```php
$pool = Process::pool(function (Pool $pool) {
    $pool->as('first')->command('bash import-1.sh');
    $pool->as('second')->command('bash import-2.sh');
    $pool->as('third')->command('bash import-3.sh');
})->start(function (string $type, string $output, string $key) {
    // …
});

$results = $pool->wait();

return $results['first']->output();
```

<a name="pool-process-ids-and-signals"></a>
### プールプロセスIDとシグナル

プロセスプールの`running`メソッドは、プール内で呼び出したすべてのプロセスのコレクションを提供するため、プール内のプロセスIDで簡単にアクセスできます。

```php
$processIds = $pool->running()->each->id();
```

また、便利なように、プロセスプール上で`signal`メソッドを呼び出すと、そのプール内のすべてのプロセスへシグナルを送ることができます。

```php
$pool->signal(SIGUSR2);
```

<a name="testing"></a>
## テスト

Laravelの多くのサービスでは、テストを簡単かつ表現豊かに書くための機能を提供していますが、Laravelプロセスサービスも例外ではありません。`Process`ファサードの`fake`メソッドを使用すると、プロセスが呼び出されたときに、Laravelへスタブ／ダミーの結果を返すように指示できます。

<a name="faking-processes"></a>
### プロセスのFake

LaravelのプロセスFake機能を調べるため、プロセスを起動するあるルートを想像してみましょう。

```php
use Illuminate\Support\Facades\Process;
use Illuminate\Support\Facades\Route;

Route::get('/import', function () {
    Process::run('bash import.sh');

    return 'Import complete!';
});
```

このルートをテストする場合、引数なしで`Process`ファサードの`fake`メソッドを呼び出すことで、起動したすべてのプロセスに対し、プロセス実行成功のFakeな結果を返すように、Laravelへ指示できます。さらに、与えられたプロセスが「実行済み(run)」であることを[アサート](#available-assertions)することもできます。

```php tab=Pest
<?php

use Illuminate\Process\PendingProcess;
use Illuminate\Contracts\Process\ProcessResult;
use Illuminate\Support\Facades\Process;

test('process is invoked', function () {
    Process::fake();

    $response = $this->get('/import');

    // シンプルなプロセスのアサート
    Process::assertRan('bash import.sh');

    // もしくは、プロセス設定を調べる
    Process::assertRan(function (PendingProcess $process, ProcessResult $result) {
        return $process->command === 'bash import.sh' &&
               $process->timeout === 60;
    });
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Process\PendingProcess;
use Illuminate\Contracts\Process\ProcessResult;
use Illuminate\Support\Facades\Process;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_process_is_invoked(): void
    {
        Process::fake();

        $response = $this->get('/import');

        // シンプルなプロセスのアサート
        Process::assertRan('bash import.sh');

        // もしくは、プロセス設定を調べる
        Process::assertRan(function (PendingProcess $process, ProcessResult $result) {
            return $process->command === 'bash import.sh' &&
                   $process->timeout === 60;
        });
    }
}
```

説明してきたように、`Process`ファサードの`fake`メソッドを呼び出すと、Laravelは常にプロセス実行成功の結果を返すように指示し、出力は返しません。しかし、`Process`ファサードの`result`メソッドを使用すると、Fakeしたプロセスの出力と終了コードを簡単に指定できます。

```php
Process::fake([
    '*' => Process::result(
        output: 'Test output',
        errorOutput: 'Test error output',
        exitCode: 1,
    ),
]);
```

<a name="faking-specific-processes"></a>
### 特定プロセスのFake

前の例でお気づきかもしれませんが、`Process`ファサードの`fake`メソッドへ配列を渡すことで、プロセスごとに異なるFakeの結果を指定できます。

配列のキーは、フェイクしたいコマンドのパターンと、それに関連する結果を表してください。 `*`文字は、ワイルドカード文字として使用できます。Fakeしないプロセスコマンドは、実際に呼び出されます。`Process`ファサードの`result`メソッドを使用して、これらのコマンドのスタブ／フェイク結果を作成できます。

```php
Process::fake([
    'cat *' => Process::result(
        output: 'Test "cat" output',
    ),
    'ls *' => Process::result(
        output: 'Test "ls" output',
    ),
]);
```

Fakeしたプロセスの終了コードや、エラー出力をカスタマイズする必要がない場合は、Fakeプロセスの結果を単純な文字列として指定する方法が便利です。

```php
Process::fake([
    'cat *' => 'Test "cat" output',
    'ls *' => 'Test "ls" output',
]);
```

<a name="faking-process-sequences"></a>
### プロセス順序のFake

テスト対象のコードが同じコマンドにより複数のプロセスを呼び出す場合、各プロセスの呼び出しに対し異なるFakeプロセス結果を割り当てたい場合もあるでしょう。`Process`ファサードの`sequence`メソッドで、これを実現できます。

```php
Process::fake([
    'ls *' => Process::sequence()
        ->push(Process::result('First invocation'))
        ->push(Process::result('Second invocation')),
]);
```

<a name="faking-asynchronous-process-lifecycles"></a>
### 非同期プロセスライフサイクルのFake

ここまでは主に、`run`メソッドを使用して、同期的に起動するプロセスのフェイクについて説明してきました。しかし、もし`start`を使って呼び出す非同期処理とやりとりするコードをテストしようとするなら、Fakeの処理を記述するためにもっと洗練されたアプローチが必要になるでしょう。

例として、非同期処理を操作する、以下のようなルートを想像してください。

```php
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Route;

Route::get('/import', function () {
    $process = Process::start('bash import.sh');

    while ($process->running()) {
        Log::info($process->latestOutput());
        Log::info($process->latestErrorOutput());
    }

    return 'Done';
});
```

この処理を適切にフェイクするためには、`running`メソッドが、何回`true`を返すか記述できるようにする必要があります。さらに、複数行の出力を順番に返すように指定したい場合もあります。これを実現するには、`Process`ファサードの`describe`メソッドを使用します。

```php
Process::fake([
    'bash import.sh' => Process::describe()
        ->output('First line of standard output')
        ->errorOutput('First line of error output')
        ->output('Second line of standard output')
        ->exitCode(0)
        ->iterations(3),
]);
```

上の例を掘り下げてみましょう。`output`メソッドと`errorOutput`メソッドを使用して、順番に出力される行を複数指定しています。`exitCode`メソッドを使い、Fakeプロセスの最終的な終了コードを指定しています。最後に、`iterations`メソッドを使用して、`running`メソッドが`true`を何回返すか、指定しています。

<a name="available-assertions"></a>
### 利用可能なアサート

[以前に説明した](#faking-processes)ように、Laravelは機能テスト用にいくつかのプロセスのアサートを提供しています。以下は、それぞれのアサートについての説明です。

<a name="assert-process-ran"></a>
#### assertRan

指定したプロセスが、起動されたことをアサートします。

```php
use Illuminate\Support\Facades\Process;

Process::assertRan('ls -la');
```

`assertRan`メソッドは、クロージャを引数に取り、プロセスのインスタンスとプロセス結果を受け取りますので、プロセスの設定オプションを調べられます。このクロージャが、`true`を返した場合、アサートは「パス」となります。

```php
Process::assertRan(fn ($process, $result) =>
    $process->command === 'ls -la' &&
    $process->path === __DIR__ &&
    $process->timeout === 60
);
```

`assertRan`クロージャに渡す、`$process`は`Illuminate\Process\PendingProcess`インスタンスであり、`$result`は`Illuminate\Contracts\Process\ProcessResult`インスタンスです。

<a name="assert-process-didnt-run"></a>
#### assertDidntRun

指定したプロセスが、起動されなかったことをアサートします。

```php
use Illuminate\Support\Facades\Process;

Process::assertDidntRun('ls -la');
```

`assertRan`メソッドと同様に、`assertDidntRun`メソッドもクロージャを引数に取ります。クロージャはプロセスのインスタンスとプロセス結果を受け取りますので、プロセスの設定オプションを調べられます。このクロージャが、`true`を返すと、アサートは「失敗」します。

```php
Process::assertDidntRun(fn (PendingProcess $process, ProcessResult $result) =>
    $process->command === 'ls -la'
);
```

<a name="assert-process-ran-times"></a>
#### assertRanTimes

指定したプロセスが、指定回数だけ起動されたことをバリデートします。

```php
use Illuminate\Support\Facades\Process;

Process::assertRanTimes('ls -la', times: 3);
```

`assertRanTimes`メソッドも、クロージャを引数に取り、プロセスのインスタンスとプロセス結果を受け取りますので、プロセスの設定オプションを調べられます。このクロージャが、`true` を返し、プロセスが指定した回数だけ起動された場合、アサーションは「パス」となります。

```php
Process::assertRanTimes(function (PendingProcess $process, ProcessResult $result) {
    return $process->command === 'ls -la';
}, times: 3);
```

<a name="preventing-stray-processes"></a>
### 未指定プロセスの防止

もし、個別のテストや、テストスイート全体を通して、呼び出されたすべてのプロセスがFakeされていることを確認したい場合は、 `preventStrayProcesses`メソッドを呼び出してください。このメソッドを呼び出すと、対応するFake結果を持たないプロセスは、実際のプロセスを開始するのではなく、例外を投げます。

    use Illuminate\Support\Facades\Process;

    Process::preventStrayProcesses();

    Process::fake([
        'ls *' => 'Test output...',
    ]);

    // Fakeレスポンスが返る
    Process::run('ls -la');

    // 例外が投げられる
    Process::run('bash import.sh');
