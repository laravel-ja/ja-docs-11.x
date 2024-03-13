# モック

- [イントロダクション](#introduction)
- [オブジェクトのモック](#mocking-objects)
- [ファサードのモック](#mocking-facades)
    - [ファサードのスパイ](#facade-spies)
- [時間操作](#interacting-with-time)

<a name="introduction"></a>
## イントロダクション

Laravelアプリケーションをテストするとき、アプリケーションの一部分を「モック」し、特定のテストを行う間は実際のコードを実行したくない場合があります。たとえば、イベントをディスパッチするコントローラをテストする場合、テスト中に実際に実行されないように、イベントリスナをモックすることができます。これにより、イベントリスナはそれ自身のテストケースでテストできるため、イベントリスナの実行について気を取られずに、コントローラのHTTPレスポンスのみをテストできます。

Laravelは最初からイベント、ジョブ、その他のファサードをモックするための便利な方法を提供しています。これらのヘルパは主にMockeryの便利なレイヤーを提供するため、複雑なMockeryメソッド呼び出しを手作業で行う必要はありません。

<a name="mocking-objects"></a>
## オブジェクトのモック

Laravelの[サービスコンテナ](/docs/{{version}}/container)を介してアプリケーションに注入されるオブジェクトをモックする場合、モックしたインスタンスを`instance`結合としてコンテナに結合する必要があります。これにより、オブジェクト自体を構築する代わりに、オブジェクトのモックインスタンスを使用するようコンテナへ指示できます。

```php tab=Pest
use App\Service;
use Mockery;
use Mockery\MockInterface;

test('something can be mocked', function () {
    $this->instance(
        Service::class,
        Mockery::mock(Service::class, function (MockInterface $mock) {
            $mock->shouldReceive('process')->once();
        })
    );
});
```

```php tab=PHPUnit
use App\Service;
use Mockery;
use Mockery\MockInterface;

public function test_something_can_be_mocked(): void
{
    $this->instance(
        Service::class,
        Mockery::mock(Service::class, function (MockInterface $mock) {
            $mock->shouldReceive('process')->once();
        })
    );
}
```

これをより便利にするために、Laravelの基本テストケースクラスが提供する`mock`メソッドを使用できます。たとえば、以下の例は上記の例と同じです。

    use App\Service;
    use Mockery\MockInterface;

    $mock = $this->mock(Service::class, function (MockInterface $mock) {
        $mock->shouldReceive('process')->once();
    });

オブジェクトのいくつかのメソッドをモックするだけでよい場合は、`partialMock`メソッドを使用できます。モックされていないメソッドは、通常どおり呼び出されたときに実行されます。

    use App\Service;
    use Mockery\MockInterface;

    $mock = $this->partialMock(Service::class, function (MockInterface $mock) {
        $mock->shouldReceive('process')->once();
    });

同様に、オブジェクトを[スパイ](http://docs.mockery.io/en/latest/reference/spies.html)したい場合のため、Laravelの基本テストケースクラスは、`Mockery::spy`メソッドの便利なラッパーとして`spy`メソッドを提供しています。スパイはモックに似ています。ただし、スパイはスパイとテスト対象のコードとの間のやり取りを一度記録するため、コードの実行後にアサーションを作成できます。

    use App\Service;

    $spy = $this->spy(Service::class);

    // …

    $spy->shouldHaveReceived('process');

<a name="mocking-facades"></a>
## ファサードのモック

従来の静的メソッド呼び出しとは異なり、[ファサード](/docs/{{version}}/facades)（[リアルタイムファサード](/docs/{{version}}/facades#real-time-facades)を含む）もモックできます。これにより、従来の静的メソッドに比べて大きな利点が得られ、従来の依存注入を使用した場合と同じくテストが簡単になります。テスト時、コントローラの１つで発生するLaravelファサードへの呼び出しをモックしたい場合がよくあるでしょう。例として、次のコントローラアクションについて考えてみます。

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Support\Facades\Cache;

    class UserController extends Controller
    {
        /**
         * アプリケーションのすべてのユーザーのリストを取得
         */
        public function index(): array
        {
            $value = Cache::get('key');

            return [
                // ...
            ];
        }
    }

`shouldReceive`メソッドを使用して`Cache`ファサードへの呼び出しをモックできます。これにより、[Mockery](https://github.com/padraic/mockery)モックのインスタンスが返されます。ファサードは実際にはLaravel[サービスコンテナ](/docs/{{version}}/container)によって依存解決および管理されるため、通常の静的クラスよりもはるかにテストがやりやすいのです。たとえば、`Cache`ファサードの`get`メソッドの呼び出しをモックしてみましょう。

```php tab=Pest
<?php

use Illuminate\Support\Facades\Cache;

test('get index', function () {
    Cache::shouldReceive('get')
                ->once()
                ->with('key')
                ->andReturn('value');

    $response = $this->get('/users');

    // …
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Support\Facades\Cache;
use Tests\TestCase;

class UserControllerTest extends TestCase
{
    public function test_get_index(): void
    {
        Cache::shouldReceive('get')
                    ->once()
                    ->with('key')
                    ->andReturn('value');

        $response = $this->get('/users');

        // ...
    }
}
```

> [!WARNING]  
> `Request`ファサードをモックしないでください。代わりに、テストの実行時に、`get`や`post`などの[HTTPテストメソッド](/docs/{{version}}/http-tests)に必要な入力を渡します。同様に、`Config`ファサードをモックする代わりに、テストでは`Config::set`メソッドを呼び出してください。

<a name="facade-spies"></a>
### ファサードのスパイ

ファサードで[スパイ](http://docs.mockery.io/en/latest/reference/spies.html)したい場合は、対応するファサードで`spy`メソッドを呼び出します。スパイはモックに似ています。ただし、スパイはスパイとテスト対象のコードとの間のやり取りを一時的に記録しているため、コードの実行後にアサーションを作成できます。

```php tab=Pest
<?php

use Illuminate\Support\Facades\Cache;

test('values are be stored in cache', function () {
    Cache::spy();

    $response = $this->get('/');

    $response->assertStatus(200);

    Cache::shouldHaveReceived('put')->once()->with('name', 'Taylor', 10);
});
```

```php tab=PHPUnit
use Illuminate\Support\Facades\Cache;

public function test_values_are_be_stored_in_cache(): void
{
    Cache::spy();

    $response = $this->get('/');

    $response->assertStatus(200);

    Cache::shouldHaveReceived('put')->once()->with('name', 'Taylor', 10);
}
```

<a name="interacting-with-time"></a>
## 時間操作

テスト時、`now`や`Illuminate\Support\Carbon::now()`のようなヘルパが返す時間を変更したいことはよくあります。幸いなことに、Laravelのベース機能テストクラスは現在時間を操作するヘルパを用意しています。

```php tab=Pest
test('time can be manipulated', function () {
    // Travel into the future...
    $this->travel(5)->milliseconds();
    $this->travel(5)->seconds();
    $this->travel(5)->minutes();
    $this->travel(5)->hours();
    $this->travel(5)->days();
    $this->travel(5)->weeks();
    $this->travel(5)->years();

    // Travel into the past...
    $this->travel(-5)->hours();

    // Travel to an explicit time...
    $this->travelTo(now()->subHours(6));

    // Return back to the present time...
    $this->travelBack();
});
```

```php tab=PHPUnit
public function test_time_can_be_manipulated(): void
{
    // Travel into the future...
    $this->travel(5)->milliseconds();
    $this->travel(5)->seconds();
    $this->travel(5)->minutes();
    $this->travel(5)->hours();
    $this->travel(5)->days();
    $this->travel(5)->weeks();
    $this->travel(5)->years();

    // Travel into the past...
    $this->travel(-5)->hours();

    // Travel to an explicit time...
    $this->travelTo(now()->subHours(6));

    // Return back to the present time...
    $this->travelBack();
}
```

また、さまざまな時間移動のメソッドへ、クロージャを提供することもできます。クロージャは、指定された時刻で時間を止めたまま起動します。クロージャが実行されると、時間は通常通り再開されます。

    $this->travel(5)->days(function () {
        // ５日後の将来で、何かをテストする…
    });

    $this->travelTo(now()->subDays(10), function () {
        // 指定した時間で、何かをテストする…
    });

`freezeTime`メソッドは、現在の時刻を停止するために使用します。同様に、`freezeSecond`メソッドは、現在の時刻で止めますが、現在秒の先頭で止めます。

    use Illuminate\Support\Carbon;

    // 時刻を止め、クロージャ実行後は通常どおりに再開する
    $this->freezeTime(function (Carbon $time) {
        // ...
    });

    // 現在秒で時刻を止め、クロージャ実行後は通常通りに再開する
    $this->freezeSecond(function (Carbon $time) {
        // ...
    })

ご想像の通り、上記の方法はすべて、ディスカッションフォーラムで非アクティブな投稿をロックするなど、時間に敏感なアプリケーションの動作をテストするのに主に役立ちます。

```php tab=Pest
use App\Models\Thread;

test('forum threads lock after one week of inactivity', function () {
    $thread = Thread::factory()->create();

    $this->travel(1)->week();

    expect($thread->isLockedByInactivity())->toBeTrue();
});
```

```php tab=PHPUnit
use App\Models\Thread;

public function test_forum_threads_lock_after_one_week_of_inactivity()
{
    $thread = Thread::factory()->create();

    $this->travel(1)->week();

    $this->assertTrue($thread->isLockedByInactivity());
}
```
