# テスト: テストの準備

- [イントロダクション](#introduction)
- [環境](#environment)
- [テストの作成](#creating-tests)
- [テストの実行](#running-tests)
    - [テストを並列で実行](#running-tests-in-parallel)
    - [テストカバレージのレポート](#reporting-test-coverage)
    - [テストのプロファイル](#profiling-tests)

<a name="introduction"></a>
## イントロダクション

Laravel is built with testing in mind. In fact, support for testing with [Pest](https://pestphp.com) and [PHPUnit](https://phpunit.de) is included out of the box and a `phpunit.xml` file is already set up for your application. The framework also ships with convenient helper methods that allow you to expressively test your applications.

デフォルトでは、アプリケーションの`tests`ディレクトリには、`Feature`と`Unit`の２つのディレクトリを用意しています。単体テストは、コードの非常に小さな孤立した部分に焦点を当てたテストです。実際、ほとんどの単体テストはおそらく単一のメソッドに焦点を合わせています。「ユニット」テストディレクトリ内のテストはLaravelアプリケーションを起動しないため、アプリケーションのデータベースやその他のフレームワークサービスにアクセスできません。

機能テストでは、複数のオブジェクトが相互作用する方法や、JSONエンドポイントへの完全なHTTPリクエストなど、コードの広い部分をテストします。**一般的に、ほとんどのテストは機能テストである必要があります。これらのタイプのテストは、システム全体が意図したとおりに機能しているという信頼性を一番提供します。**

An `ExampleTest.php` file is provided in both the `Feature` and `Unit` test directories. After installing a new Laravel application, execute the `vendor/bin/pest`, `vendor/bin/phpunit`, or `php artisan test` commands to run your tests.

<a name="environment"></a>
## 環境

When running tests, Laravel will automatically set the [configuration environment](/docs/{{version}}/configuration#environment-configuration) to `testing` because of the environment variables defined in the `phpunit.xml` file. Laravel also automatically configures the session and cache to the `array` driver so that no session or cache data will be persisted while testing.

必要に応じて、他のテスト環境設定値を自由に定義できます。`testing`環境変数はアプリケーションの`phpunit.xml`ファイルで設定していますが、テストを実行する前は必ず`config:clear` Artisanコマンドを使用して設定のキャッシュをクリアしてください。

<a name="the-env-testing-environment-file"></a>
#### `.env.testing`環境ファイル

In addition, you may create a `.env.testing` file in the root of your project. This file will be used instead of the `.env` file when running Pest and PHPUnit tests or executing Artisan commands with the `--env=testing` option.

<a name="creating-tests"></a>
## テストの作成

新しいテストケースを作成するには、`make:test` Artisanコマンドを使用します。デフォルトでは、テストは`tests/Feature`ディレクトリへ配置されます。

```shell
php artisan make:test UserTest
```

`tests/Unit`ディレクトリ内にテストを作成したい場合は、`make:test`コマンドを実行するときに`--unit`オプションを使用します。

```shell
php artisan make:test UserTest --unit
```

> [!NOTE]
> [stubのリソース公開](/docs/{{version}}/artisan#stub-customization) を使って、Testスタブをカスタマイズできます。

Once the test has been generated, you may define test as you normally would using Pest or PHPUnit. To run your tests, execute the `vendor/bin/pest`, `vendor/bin/phpunit`, or `php artisan test` command from your terminal:

```php tab=Pest
<?php

test('basic', function () {
    expect(true)->toBeTrue();
});
```

```php tab=PHPUnit
<?php

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic test example.
     */
    public function test_basic_test(): void
    {
        $this->assertTrue(true);
    }
}
```

> [!WARNING]
> テストクラスに独自の`setUp`メソッドを定義する場合は、親のクラスの`parent::setUp()`／`parent::tearDown()`を確実に呼び出してください。通常、`parent::setUp()`は自分の `setUp`メソッドの最初で、`parent::tearDown()`は自分の`tearDown`メソッドの最後で呼び出します。

<a name="running-tests"></a>
## テストの実行

As mentioned previously, once you've written tests, you may run them using `pest` or `phpunit`:

```shell tab=Pest
./vendor/bin/pest
```

```shell tab=PHPUnit
./vendor/bin/phpunit
```

In addition to the `pest` or `phpunit` commands, you may use the `test` Artisan command to run your tests. The Artisan test runner provides verbose test reports in order to ease development and debugging:

```shell
php artisan test
```

Any arguments that can be passed to the `pest` or `phpunit` commands may also be passed to the Artisan `test` command:

```shell
php artisan test --testsuite=Feature --stop-on-failure
```

<a name="running-tests-in-parallel"></a>
### テストを並列で実行

By default, Laravel and Pest / PHPUnit execute your tests sequentially within a single process. However, you may greatly reduce the amount of time it takes to run your tests by running tests simultaneously across multiple processes. To get started, you should install the `brianium/paratest` Composer package as a "dev" dependency. Then, include the `--parallel` option when executing the `test` Artisan command:

```shell
composer require brianium/paratest --dev

php artisan test --parallel
```

デフォルトで、Laravelはマシンで利用可能なCPUコアの数だけ、プロセスを作成します。しかし、`--processes`オプションでプロセス数を調整できます。

```shell
php artisan test --parallel --processes=4
```

> [!WARNING]
> When running tests in parallel, some Pest / PHPUnit options (such as `--do-not-cache-result`) may not be available.

<a name="parallel-testing-and-databases"></a>
#### 並列テストとデータベース

プライマリデータベース接続を設定さえすれば、Laravelは、テストを実行する並列プロセスごとに、テストデータベースの作成とマイグレーションを自動的に処理します。テストデータベースは、プロセスごとに一意のプロセストークンをサフィックス化します。たとえば、２つの並列テストプロセスがある場合、Laravelは`your_db_test_1`と`your_db_test_2`テストデータベースを作成して使用します。

デフォルトでは、テストデータベースは`test` Artisanコマンドへの呼び出し間で持続的に保持され、後続の`test`呼び出しで再使用できます。ただし、`--recreate-databases`オプションを使用してそれらを再作成できます。

```shell
php artisan test --parallel --recreate-databases
```

<a name="parallel-testing-hooks"></a>
#### 並列テストフック

アプリケーションのテストでときどき、複数のテストプロセスにより安全に使用される特定のリソースを準備する必要があるかもしれません。

`ParallelTesting`ファサードを使用して、プロセスやテストケースの`setUp`と`tearDown`で実行するコードを指定できます。指定するクロージャは、プロセストークンと現在のテストケースを含む`$token`と`$testcase`変数を受け取ります。

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\Artisan;
    use Illuminate\Support\Facades\ParallelTesting;
    use Illuminate\Support\ServiceProvider;
    use PHPUnit\Framework\TestCase;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * 全アプリケーションサービスの初期起動処理
         */
        public function boot(): void
        {
            ParallelTesting::setUpProcess(function (int $token) {
                // ...
            });

            ParallelTesting::setUpTestCase(function (int $token, TestCase $testCase) {
                // ...
            });

            // テストデータベースが作成されたときに実行される
            ParallelTesting::setUpTestDatabase(function (string $database, int $token) {
                Artisan::call('db:seed');
            });

            ParallelTesting::tearDownTestCase(function (int $token, TestCase $testCase) {
                // ...
            });

            ParallelTesting::tearDownProcess(function (int $token) {
                // ...
            });
        }
    }

<a name="accessing-the-parallel-testing-token"></a>
#### 並列テストトークンへのアクセス

アプリケーションのテストコードの他の場所から、現在の並列プロセスの「トークン」にアクセスしたい場合は、`token`メソッドを使用します。このトークンは個々のテストプロセスのための一意な文字列の識別子であり、並列テストプロセス間でリソースを分割するために使用できます。たとえば、Laravelは各並行テストプロセスで作成するテストデータベースの末尾に、このトークンを自動的に付加します。

    $token = ParallelTesting::token();

<a name="reporting-test-coverage"></a>
### テストカバレージのレポート

> [!WARNING]
> この機能を使用するには、[Xdebug](https://xdebug.org)、または[PCOV](https://pecl.php.net/package/pcov)が必要です。

アプリケーションのテストを実行する際に、テストケースが実際にアプリケーションコードをどの程度カバーしているかどうか、また、テストを実行する際にどれだけのアプリケーションコードが使用されているかを確認したいと思うことでしょう。これを行うには、`test`コマンドを実行するときに、`--coverage`オプションを指定します。

```shell
php artisan test --coverage
```

<a name="enforcing-a-minimum-coverage-threshold"></a>
#### 最低限度のカバーレージ基準の強制

`min`オプションを使用すると、アプリケーションのテストカバレッジの最小値を定義できます。この閾値を満たさない場合、テストスイートは失敗します。

```shell
php artisan test --coverage --min=80.3
```

<a name="profiling-tests"></a>
### テストのプロファイル

Artisanテストランナは、アプリケーションの最も遅いテストをリストアップする、便利なメカニズムも用意しています。`test`コマンドへ`--profile`オプションを付けて起動すると、最も遅い１０テストのリストが表示され、テストスイートを高速化するため、どのテストを改善できるかを簡単に調査できます。

```shell
php artisan test --profile
```
