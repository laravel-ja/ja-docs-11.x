# コンソールテスト

- [イントロダクション](#introduction)
- [実行成功／失敗のアサート](#success-failure-expectations)
- [入力/出力の期待値](#input-output-expectations)
- [コンソールイベント](#console-events)

<a name="introduction"></a>
## イントロダクション

Laravelは、HTTPテストを簡素化することに加え、アプリケーションの[カスタムコンソールコマンド](/docs/{{version}}/artisan)をテストするためのシンプルなAPIも提供します。

<a name="success-failure-expectations"></a>
## 実行成功／失敗のアサート

手始めに、Artisanコマンドの終了コードをアサートする方法を見てみましょう。それには、`artisan`メソッドを使用して、テストからArtisanコマンドを呼び出します。それから、`assertExitCode`メソッドを使用して、指定した終了コードでコマンドが完了することをアサートします。

```php tab=Pest
test('console command', function () {
    $this->artisan('inspire')->assertExitCode(0);
});
```

```php tab=PHPUnit
/**
 * コンソールコマンドのテスト
 */
public function test_console_command(): void
{
    $this->artisan('inspire')->assertExitCode(0);
}
```

コマンドが指定コードで終了しないことをアサートするには、`assertNotExitCode`メソッドを使用します。

    $this->artisan('inspire')->assertNotExitCode(1);

もちろん、すべてのターミナルコマンドは通常、成功した場合に`0`のステータスコードで終了し、成功しなかった場合には０以外の終了コードで終了します。したがって、使い勝手が良いように`assertSuccessful`と`assertFailed`のアサーションを利用し、コマンドが成功の終了コードで終了した／終了しなかったことをアサートできます。

    $this->artisan('inspire')->assertSuccessful();

    $this->artisan('inspire')->assertFailed();

<a name="input-output-expectations"></a>
## 入力/出力の期待値

Laravelで`expectsQuestion`メソッドを使用すれば、コンソールコマンドのユーザー入力を簡単に「モック」できます。さらに、終了コードを`assertExitCode`メソッドで、コンソールコマンドに期待する出力を`expectsOutput`メソッドでそれぞれ指定することもできます。

    Artisan::command('question', function () {
        $name = $this->ask('What is your name?');

        $language = $this->choice('Which language do you prefer?', [
            'PHP',
            'Ruby',
            'Python',
        ]);

        $this->line('Your name is '.$name.' and you prefer '.$language.'.');
    });

このコマンドは、`expectsQuestion`、`expectsOutput`、`doesntExpectOutput`、`expectsOutputToContain`、`doesntExpectOutputToContain`、`assertExitCode`メソッドを利用する以下の例でテストできます。

```php tab=Pest
test('console command', function () {
    $this->artisan('question')
         ->expectsQuestion('What is your name?', 'Taylor Otwell')
         ->expectsQuestion('Which language do you prefer?', 'PHP')
         ->expectsOutput('Your name is Taylor Otwell and you prefer PHP.')
         ->doesntExpectOutput('Your name is Taylor Otwell and you prefer Ruby.')
         ->expectsOutputToContain('Taylor Otwell')
         ->doesntExpectOutputToContain('you prefer Ruby')
         ->assertExitCode(0);
});
```

```php tab=PHPUnit
/**
 * コンソールコマンドのテスト
 */
public function test_console_command(): void
{
    $this->artisan('question')
         ->expectsQuestion('What is your name?', 'Taylor Otwell')
         ->expectsQuestion('Which language do you prefer?', 'PHP')
         ->expectsOutput('Your name is Taylor Otwell and you prefer PHP.')
         ->doesntExpectOutput('Your name is Taylor Otwell and you prefer Ruby.')
         ->expectsOutputToContain('Taylor Otwell')
         ->doesntExpectOutputToContain('you prefer Ruby')
         ->assertExitCode(0);
}
```

<a name="confirmation-expectations"></a>
#### 確認の期待

「はい」または「いいえ」の回答の形式で確認を期待するコマンドを作成する場合は、`expectsConfirmation`メソッドを使用できます。

    $this->artisan('module:import')
        ->expectsConfirmation('Do you really wish to run this command?', 'no')
        ->assertExitCode(1);

<a name="table-expectations"></a>
#### テーブルの期待

コマンドがArtisanの`table`メソッドを使用して情報のテーブルを表示する場合、テーブル全体の出力期待値を書き込むのは面倒な場合があります。代わりに、`expectsTable`メソッドを使用できます。このメソッドは、テーブルのヘッダを最初の引数として受け入れ、テーブルのデータを２番目の引数として受け入れます。

    $this->artisan('users:all')
        ->expectsTable([
            'ID',
            'Email',
        ], [
            [1, 'taylor@example.com'],
            [2, 'abigail@example.com'],
        ]);

<a name="console-events"></a>
## コンソールイベント

デフォルトでは、アプリケーションのテスト実行中は、`Illuminate\Console\Events\CommandStarting`と`Illuminate\Console\Events\CommandFinished`はデフォルトでディスパッチしません。しかし、テストクラスへ`Illuminate\Foundation\Testing\WithConsoleEvents`トレイトを追加すれば、これらのイベントを有効にできます：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\WithConsoleEvents;

uses(WithConsoleEvents::class);

// ...
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Foundation\Testing\WithConsoleEvents;
use Tests\TestCase;

class ConsoleEventTest extends TestCase
{
    use WithConsoleEvents;

    // ...
}
```
