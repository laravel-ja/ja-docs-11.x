# 並列処理

- [イントロダクション](#introduction)
- [タスクの同時実行](#running-concurrent-tasks)
- [タスクの遅延実行](#deferring-concurrent-tasks)

<a name="introduction"></a>
## イントロダクション

> [!WARNING]
> Laravelの`Concurrency`ファサードは、現在ベータであり、コミュニティーのフィードバッグを集めています。

時には、互いに依存していない複数の低速タスクを実行する必要があるでしょう。多くの場合、タスクを同時実行することで大幅なパフォーマンス改善が実現できます。Laravelの`Concurrency`ファサードは、クロージャを同時実行するためのシンプルで便利なAPIを提供します。

<a name="how-it-works"></a>
#### いかに動作するか

Laravelは指定クロージャをシリアライズし、隠しArtisan CLIコマンドへディスパッチすることで並行処理を実現します。Artisan CLIコマンドはクロージャのシリアライズを解除し、自身のPHPプロセス内でそれを呼び出します。クロージャが呼び出された後、結果の値はシリアライズされて親プロセスに戻されます。

`Concurrency`ファサードは、3つのドライバをサポートします。`process`(デフォルト)、`fork`、`sync`です。

`fork`ドライバは、デフォルトの`process`ドライバに比べパフォーマンスが向上しますが、PHPのCLI コンテキストでのみ使用できます。`fork`ドライバを使用する前に、`spatie/fork`パッケージをインストールする必要があります。

```bash
composer require spatie/fork
```

`sync`ドライバは主にテスト中で使用し、すべての並列処理を無効にして、親プロセス内で与えられたクロージャを順番に実行したいときに便利です。

<a name="running-concurrent-tasks"></a>
## タスクの同時実行

タスクを並列実行するには、`Concurrency`ファサードの`run`メソッドを呼び出します。`run`メソッドには、PHPの子プロセスで同時に実行するクロージャの配列を渡します。

```php
use Illuminate\Support\Facades\Concurrency;
use Illuminate\Support\Facades\DB;

[$userCount, $orderCount] = Concurrency::run([
    fn () => DB::table('users')->count(),
    fn () => DB::table('orders')->count(),
]);
```

指定のドライバを使用するには、`driver`メソッドを使用します。

```php
$results = Concurrency::driver('fork')->run(...);
```

または、デフォルトの並列実行ドライバを変更するため、`config:publish` Artisanコマンドで`concurrency`設定ファイルをリソース公開し、ファイル内の`default`オプションを更新する必要があります。

```bash
php artisan config:publish concurrency
```

<a name="deferring-concurrent-tasks"></a>
## タスクの遅延実行

クロージャの配列を並列実行したいが、それらのクロージャが返す結果には興味がない場合は、`defer`メソッドの使用を検討すべきでしょう。`defer`メソッドを呼び出すと、指定されたクロージャはすぐには実行されません。代わりに、LaravelはHTTPレスポンスがユーザーに送信された後に、クロージャを同時に実行します。

```php
use App\Services\Metrics;
use Illuminate\Support\Facades\Concurrency;

Concurrency::defer([
    fn () => Metrics::report('users'),
    fn () => Metrics::report('orders'),
]);
```
