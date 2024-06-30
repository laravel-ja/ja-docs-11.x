# イベント

- [イントロダクション](#introduction)
- [イベントとリスナの生成](#generating-events-and-listeners)
- [イベントとリスナの登録](#registering-events-and-listeners)
    - [イベント追跡](#event-discovery)
    - [イベントの手作業登録](#manually-registering-events)
    - [クロージャリスナ](#closure-listeners)
- [イベント定義](#defining-events)
- [リスナ定義](#defining-listeners)
- [キュー投入するイベントリスナ](#queued-event-listeners)
    - [キューの手作業操作](#manually-interacting-with-the-queue)
    - [キュー投入するイベントリスナとデータベーストランザクション](#queued-event-listeners-and-database-transactions)
    - [失敗したジョブの処理](#handling-failed-jobs)
- [イベント発行](#dispatching-events)
    - [データベーストランザクション後のイベント発行](#dispatching-events-after-database-transactions)
- [イベントサブスクライバ](#event-subscribers)
    - [イベントサブスクライバの記述](#writing-event-subscribers)
    - [イベントサブスクライバの登録](#registering-event-subscribers)
- [テスト](#testing)
    - [イベントサブセットのFake](#faking-a-subset-of-events)
    - [イベントFakeのスコープ](#scoped-event-fakes)

<a name="introduction"></a>
## イントロダクション

Laravelのイベントは、単純なオブザーバーパターンの実装を提供し、アプリケーション内で発生するさまざまなイベントをサブスクライブしてリッスンできるようにします。イベントクラスは通常、`app/Events`ディレクトリに保存し、リスナは`app/Listeners`に保存します。Artisanコンソールコマンドを使用してイベントとリスナを生成すると、これらのディレクトリが作成されるため、アプリケーションにこれらのディレクトリが表示されていなくても心配ありません。

１つのイベントに、相互に依存しない複数のリスナを含めることができるため、イベントは、アプリケーションのさまざまな側面を分離するための優れた方法として機能します。たとえば、注文が発送されるたびにユーザーにSlack通知を送信したい場合があります。注文処理コードをSlack通知コードに結合する代わりに、リスナが受信してSlack通知をディスパッチするために使用できる`App\Events\OrderShipped`イベントを発生させることができます。

<a name="generating-events-and-listeners"></a>
## イベントとリスナの生成

イベントとリスナを素早く生成するには、`make:event`と`make:listener`のArtisanコマンドを使います。

```shell
php artisan make:event PodcastProcessed

php artisan make:listener SendPodcastNotification --event=PodcastProcessed
```

使いやすいように、引数を指定せずに`make:event`と`make:listener` Artisanコマンドを呼び出すこともできます。その場合、Laravelは自動的にクラス名、リスナを作成する場合はクラス名と、そのリスナがリッスンするイベントのクラス名を求めるプロンプトを表示します。

```shell
php artisan make:event

php artisan make:listener
```

<a name="registering-events-and-listeners"></a>
## イベントとリスナの登録

<a name="event-discovery"></a>
### イベント追跡

Laravelはデフォルトで、アプリケーションの`Listeners`ディレクトリをスキャンして、イベントリスナを自動的に見つけて登録します。`handle`または`__invoke`で始まるリスナクラスのメソッドが見つかると、Laravelはそれらのメソッドを、メソッドのシグネチャでタイプヒントしてあるイベントのイベントリスナとして登録します。

    use App\Events\PodcastProcessed;

    class SendPodcastNotification
    {
        /**
         * 指定イベントの処理
         */
        public function handle(PodcastProcessed $event): void
        {
            // ...
        }
    }

リスナを別のディレクトリや複数のディレクトリへ保存する場合は、アプリケーションの`bootstrap/app.php`ファイルで`withEvents`メソッドを使用し、それらのディレクトリをスキャンするようにLaravelに指示してください。

    ->withEvents(discover: [
        __DIR__.'/../app/Domain/Orders/Listeners',
    ])

`event:list`コマンドは、アプリケーションに登録したすべてのリスナをリストアップするために使用します。

```shell
php artisan event:list
```

<a name="event-discovery-in-production"></a>
#### 実機でのイベント追跡

アプリケーションを高速化するために、`optimize`または`event:cache` Artisanコマンドを使用して、アプリケーションのすべてのリスナのマニフェストをキャッシュする必要があります。通常、このコマンドはアプリケーションの[デプロイプロセス](/docs/{{version}}/deployment#optimization)の一部として実行する必要があります。このマニフェストは、イベント登録処理を高速化するためにフレームワークが使用します。イベントキャッシュを破棄するには、`event:clear`コマンドを使用します。

<a name="manually-registering-events"></a>
### イベントの手作業登録

`Event`ファサードを使用すると、アプリケーションの`AppServiceProvider`の`boot`メソッド内で、イベントとそれに対応するリスナを手作業で登録できます。

    use App\Domain\Orders\Events\PodcastProcessed;
    use App\Domain\Orders\Listeners\SendPodcastNotification;
    use Illuminate\Support\Facades\Event;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Event::listen(
            PodcastProcessed::class,
            SendPodcastNotification::class,
        );
    }

`event:list`コマンドは、アプリケーションに登録しているすべてのリスナをリストアップするために使用します。

```shell
php artisan event:list
```

<a name="closure-listeners"></a>
### クロージャリスナ

通常、リスナはクラスとして定義しますが、アプリケーションの`AppServiceProvider`の`boot`メソッドで、手作業でクロージャベースのイベントリスナを登録することもできます。

    use App\Events\PodcastProcessed;
    use Illuminate\Support\Facades\Event;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Event::listen(function (PodcastProcessed $event) {
            // ...
        });
    }

<a name="queuable-anonymous-event-listeners"></a>
#### Queueable匿名イベントリスナ

クロージャベースのイベントリスナを登録するとき、リスナのクロージャを`Illuminate\Events\queueable`関数でラップして、[キュー](/docs/{{version}}/queues)を使用してリスナを実行するように、Laravelへ指示できます。

    use App\Events\PodcastProcessed;
    use function Illuminate\Events\queueable;
    use Illuminate\Support\Facades\Event;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Event::listen(queueable(function (PodcastProcessed $event) {
            // ...
        }));
    }

キュー投入ジョブと同様に、`onConnection`、`onQueue`、`delay`メソッドを使用して、キュー投入するリスナの実行をカスタマイズできます。

    Event::listen(queueable(function (PodcastProcessed $event) {
        // ...
    })->onConnection('redis')->onQueue('podcasts')->delay(now()->addSeconds(10)));

キューに投入した匿名リスナの失敗を処理したい場合は、`queueable`リスナを定義するときに`catch`メソッドにクロージャを指定できます。このクロージャは、リスナの失敗の原因となったイベントインスタンスと`Throwable`インスタンスを受け取ります。

    use App\Events\PodcastProcessed;
    use function Illuminate\Events\queueable;
    use Illuminate\Support\Facades\Event;
    use Throwable;

    Event::listen(queueable(function (PodcastProcessed $event) {
        // ...
    })->catch(function (PodcastProcessed $event, Throwable $e) {
        // キュー投入したリスナは失敗した…
    }));

<a name="wildcard-event-listeners"></a>
#### ワイルドカードイベントリスナ

ワイルドカードパラメータとして、`*`文字を使用してリスナを登録することもでき、同じリスナで複数のイベントをキャッチできます。ワイルドカードリスナは、最初の引数にイベント名を受け取り、２番目の引数にイベントデータ配列全体を受け取ります。

    Event::listen('event.*', function (string $eventName, array $data) {
        // ...
    });

<a name="defining-events"></a>
## イベント定義

イベントクラスは、基本的に、イベントに関連する情報を保持するデータコンテナです。たとえば、`App\Events\OrderShipped`イベントが[Eloquent ORM](/docs/{{version}}/eloquent)オブジェクトを受け取るとします。

    <?php

    namespace App\Events;

    use App\Models\Order;
    use Illuminate\Broadcasting\InteractsWithSockets;
    use Illuminate\Foundation\Events\Dispatchable;
    use Illuminate\Queue\SerializesModels;

    class OrderShipped
    {
        use Dispatchable, InteractsWithSockets, SerializesModels;

        /**
         * 新しいイベントインスタンスの生成
         */
        public function __construct(
            public Order $order,
        ) {}
    }

ご覧のとおり、このイベントクラスにはロジックが含まれていません。購読した`App\Models\Order`インスタンスのコンテナです。イベントで使用される`SerializesModels`トレイトは、[キュー投入するリスナ](#queued-event-listeners)を利用する場合など、イベントオブジェクトがPHPの`serialize`関数を使用してシリアル化される場合、Eloquentモデルを適切にシリアル化します。

<a name="defining-listeners"></a>
## リスナ定義

次に、例題のイベントのリスナを見てみましょう。イベントリスナは`handle`メソッドで、イベントインスタンスを受け取ります。`make:listener` Artisanコマンドは、`--event`オプションを指定して呼び出すと、自動的に適切なイベントクラスをインポートし、`handle`メソッド内でイベントをタイプヒントします。`handle`メソッド内では、イベントに応答するために必要なアクションを実行できます。

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;

    class SendShipmentNotification
    {
        /**
         * イベントリスナの生成
         */
        public function __construct()
        {
            // ...
        }

        /**
         * イベントの処理
         */
        public function handle(OrderShipped $event): void
        {
            // $event->orderを使用して注文にアクセス
        }
    }

> [!NOTE]
> イベントリスナは、コンストラクタに必要な依存関係をタイプヒントすることもできます。すべてのイベントリスナはLaravel[サービスコンテナ](/docs/{{version}}/container)を介して依存解決されるため、依存関係は自動的に注入されます。

<a name="stopping-the-propagation-of-an-event"></a>
#### イベント伝播の停止

場合によっては、他のリスナへのイベント伝播を停止したいことがあります。これを行うには、リスナの`handle`メソッドから`false`を返します。

<a name="queued-event-listeners"></a>
## キュー投入するイベントリスナ

リスナをキューに投入することは、リスナが電子メールの送信やHTTPリクエストの作成などの遅いタスクを実行する場合に役立ちます。キューに入れられたリスナを使用する前に、必ず[キューを設定](/docs/{{version}}/queues)して、サーバまたはローカル開発環境でキューワーカを起動してください。

リスナをキュー投入するように指定するには、リスナクラスへ`ShouldQueue`インターフェイスを追加します。`make:listener` Artisanコマンドが生成したリスナは、あらかじめこのインターフェイスを現在の名前空間へインポートしているので、すぐに使用できます。

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;
    use Illuminate\Contracts\Queue\ShouldQueue;

    class SendShipmentNotification implements ShouldQueue
    {
        // ...
    }

これだけです！これで、このリスナによって処理されるイベントがディスパッチされると、リスナはLaravelの[キューシステム](/docs/{{version}}/queues)を使用してイベントディスパッチャによって自動的にキューへ投入されます。リスナがキューによって実行されたときに例外が投げられない場合、キュー投入済みジョブは、処理が終了した後で自動的に削除されます。

<a name="customizing-the-queue-connection-queue-name"></a>
#### キュー接続と名前、遅延のカスタマイズ

イベントリスナのキュー接続、キュー名、またはキュー遅延時間をカスタマイズする場合は、リスナクラスで`$connection`、`$queue`、`$delay`プロパティを定義できます。

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;
    use Illuminate\Contracts\Queue\ShouldQueue;

    class SendShipmentNotification implements ShouldQueue
    {
        /**
         * ジョブの送信先となる接続名
         *
         * @var string|null
         */
        public $connection = 'sqs';

        /**
         * ジョブの送信先となるキュー名
         *
         * @var string|null
         */
        public $queue = 'listeners';

        /**
         * ジョブを処理するまでの時間(秒)
         *
         * @var int
         */
        public $delay = 60;
    }

実行時にリスナのキュー接続、キュー名、遅延時間を定義したい場合は、リスナに`viaConnection`、`viaQueue`、`withDelay`メソッドを定義します。

    /**
     * リスナのキュー接続の名前の取得
     */
    public function viaConnection(): string
    {
        return 'sqs';
    }

    /**
     * リスナのキュー名を取得
     */
    public function viaQueue(): string
    {
        return 'listeners';
    }

    /**
     * ジョブを開始するまでの秒数を取得
     */
    public function withDelay(OrderShipped $event): int
    {
        return $event->highPriority ? 0 : 60;
    }

<a name="conditionally-queueing-listeners"></a>
#### 条件付き投入リスナ

場合によっては、実行時にのみ使用可能なデータに基づいて、リスナをキュー投入する必要があるかどうかを判断する必要が起きるでしょう。このために、`shouldQueue`メソッドをリスナに追加して、リスナをキュー投入する必要があるかどうかを判断できます。`shouldQueue`メソッドが`false`を返す場合、リスナはキュー投入されません。

    <?php

    namespace App\Listeners;

    use App\Events\OrderCreated;
    use Illuminate\Contracts\Queue\ShouldQueue;

    class RewardGiftCard implements ShouldQueue
    {
        /**
         * ギフトカードを顧客へ提供
         */
        public function handle(OrderCreated $event): void
        {
            // ...
        }

        /**
         * リスナをキューへ投入するかを決定
         */
        public function shouldQueue(OrderCreated $event): bool
        {
            return $event->order->subtotal >= 5000;
        }
    }

<a name="manually-interacting-with-the-queue"></a>
### キューの手作業操作

リスナの基になるキュージョブの`delete`メソッドと`release`メソッドへ手作業でアクセスする必要がある場合は、`Illuminate\Queue\InteractsWithQueue`トレイトを使用してアクセスできます。このトレイトは、生成したリスナにはデフォルトでインポートされ、以下のメソッドへのアクセスを提供します。

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Queue\InteractsWithQueue;

    class SendShipmentNotification implements ShouldQueue
    {
        use InteractsWithQueue;

        /**
         * イベントの処理
         */
        public function handle(OrderShipped $event): void
        {
            if (true) {
                $this->release(30);
            }
        }
    }

<a name="queued-event-listeners-and-database-transactions"></a>
### キュー投入するイベントリスナとデータベーストランザクション

キュー投入したリスナがデータベーストランザクション内でディスパッチされると、データベーストランザクションがコミットされる前にキューによって処理される場合があります。これが発生した場合、データベーストランザクション中にモデルまたはデータベースレコードに加えた更新は、データベースにまだ反映されていない可能性があります。さらに、トランザクション内で作成されたモデルまたはデータベースレコードは、データベースに存在しない可能性があります。リスナがこれらのモデルに依存している場合、キューに入れられたリスナをディスパッチするジョブの処理時に予期しないエラーが発生する可能性があります。

キュー接続の`after_commit`設定オプションが`false`に設定されている場合でも、リスナクラスで`ShouldHandleEventsAfterCommit`インターフェイスを実装することにより、開いているすべてのデータベーストランザクションがコミットされた後に、特定のキューに入れられたリスナをディスパッチする必要があることを示すことができます。

    <?php

    namespace App\Listeners;

    use Illuminate\Contracts\Events\ShouldHandleEventsAfterCommit;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Queue\InteractsWithQueue;

    class SendShipmentNotification implements ShouldQueue, ShouldHandleEventsAfterCommit
    {
        use InteractsWithQueue;
    }

> [!NOTE]
> こうした問題の回避方法の詳細は、[キュー投入されるジョブとデータベーストランザクション](/docs/{{version}}/queues#jobs-and-database-transactions)に関するドキュメントを確認してください。

<a name="handling-failed-jobs"></a>
### 失敗したジョブの処理

キュー投入したイベントリスナが失敗する場合があります。キュー投入したリスナがキューワーカーによって定義された最大試行回数を超えると、リスナ上の`failed`メソッドが呼び出されます。`failed`メソッドは、失敗の原因となったイベントインスタンスと`Throwable`を受け取ります。

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Queue\InteractsWithQueue;
    use Throwable;

    class SendShipmentNotification implements ShouldQueue
    {
        use InteractsWithQueue;

        /**
         * イベントの処理
         */
        public function handle(OrderShipped $event): void
        {
            // ...
        }

        /**
         * ジョブの失敗を処理
         */
        public function failed(OrderShipped $event, Throwable $exception): void
        {
            // ...
        }
    }

<a name="specifying-queued-listener-maximum-attempts"></a>
#### キュー投入したリスナの最大試行回数の指定

キュー投入したリスナの１つでエラーが発生した場合、リスナが無期限に再試行し続けることを皆さんも望まないでしょう。そのため、Laravelはリスナを試行できる回数または期間を指定するさまざまな方法を提供しています。

リスナクラスで`$trys`プロパティを定義して、リスナが失敗したと見なされるまでに試行できる回数を指定できます。

    <?php

    namespace App\Listeners;

    use App\Events\OrderShipped;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Queue\InteractsWithQueue;

    class SendShipmentNotification implements ShouldQueue
    {
        use InteractsWithQueue;

        /**
         * キュー投入したリスナが試行される回数
         *
         * @var int
         */
        public $tries = 5;
    }

リスナが失敗するまでに試行できる回数を定義する代わりに、リスナをそれ以上試行しない時間を定義することもできます。これにより、リスナは特定の時間枠内で何度でも試行します。リスナの試行最長時間を定義するには、リスナクラスに`retryUntil`メソッドを追加します。このメソッドは`DateTime`インスタンスを返す必要があります:

    use DateTime;

    /**
     * リスナタイムアウト時間を決定
     */
    public function retryUntil(): DateTime
    {
        return now()->addMinutes(5);
    }

<a name="dispatching-events"></a>
## イベント発行

イベントをディスパッチするには、イベントで静的な`dispatch`メソッドを呼び出します。このメソッドは`Illuminate\Foundation\Events\Dispatchable`トレイトにより、イベントで使用可能になります。`dispatch`メソッドに渡された引数はすべて、イベントのコンストラクタへ渡されます。

    <?php

    namespace App\Http\Controllers;

    use App\Events\OrderShipped;
    use App\Http\Controllers\Controller;
    use App\Models\Order;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;

    class OrderShipmentController extends Controller
    {
        /**
         * 指定注文を発送
         */
        public function store(Request $request): RedirectResponse
        {
            $order = Order::findOrFail($request->order_id);

            // 注文出荷ロジック…

            OrderShipped::dispatch($order);

            return redirect('/orders');
        }
    }

条件付きでイベントをディスパッチしたい場合は、`dispatchIf`か`dispatchUnless`メソッドが使用できます。

    OrderShipped::dispatchIf($condition, $order);

    OrderShipped::dispatchUnless($condition, $order);

> [!NOTE]
> テストの際、あるイベントが実際にリスナを起動することなくディスパッチされたことをアサートできると役立ちます。Laravelに[組み込み済みのテストヘルパ](#testing)は、これを簡単に実現します。

<a name="dispatching-events-after-database-transactions"></a>
### データベーストランザクション後のイベント発行

アクティブなデータベーストランザクションをコミットした後にのみ、イベントを発行するようにLaravelへ指示したい場合があると思います。そのためには、イベントクラスで`ShouldDispatchAfterCommit`インターフェイスを実装します。

このインターフェイスは、現在のデータベーストランザクションがコミットされるまで、イベントを発行しないようにLaravelへ指示するものです。トランザクションが失敗した場合は、イベントを破棄します。イベントを発行した時にデータベーストランザクションが進行中でなければ、イベントを直ちに発行します。

    <?php

    namespace App\Events;

    use App\Models\Order;
    use Illuminate\Broadcasting\InteractsWithSockets;
    use Illuminate\Contracts\Events\ShouldDispatchAfterCommit;
    use Illuminate\Foundation\Events\Dispatchable;
    use Illuminate\Queue\SerializesModels;

    class OrderShipped implements ShouldDispatchAfterCommit
    {
        use Dispatchable, InteractsWithSockets, SerializesModels;

        /**
         * 新しいイベントインスタンスの生成
         */
        public function __construct(
            public Order $order,
        ) {}
    }

<a name="event-subscribers"></a>
## イベントサブスクライバ

<a name="writing-event-subscribers"></a>
### イベントサブスクライバの記述

イベントサブスクライバは、サブスクライバクラス自体から複数のイベントを購読できるクラスであり、単一のクラス内で複数のイベントハンドラを定義できます。サブスクライバは、イベントディスパッチャーインスタンスを渡す`subscribe`メソッドを定義する必要があります。特定のディスパッチャ上の`listen`メソッドを呼び出して、イベントリスナを登録します。

    <?php

    namespace App\Listeners;

    use Illuminate\Auth\Events\Login;
    use Illuminate\Auth\Events\Logout;
    use Illuminate\Events\Dispatcher;

    class UserEventSubscriber
    {
        /**
         * ユーザーログインイベントの処理
         */
        public function handleUserLogin(Login $event): void {}

        /**
         * ユーザーログアウトイベントの処理
         */
        public function handleUserLogout(Logout $event): void {}

        /**
         * サブスクライバのリスナを登録
         */
        public function subscribe(Dispatcher $events): void
        {
            $events->listen(
                Login::class,
                [UserEventSubscriber::class, 'handleUserLogin']
            );

            $events->listen(
                Logout::class,
                [UserEventSubscriber::class, 'handleUserLogout']
            );
        }
    }

イベントリスナのメソッドがサブスクライバ自身の中で定義されている場合は、サブスクライバの`subscribe`メソッドからメソッド名とイベントの配列を返す方が便利でしょう。Laravelはイベントリスナを登録する際に、サブスクライバのクラス名を自動的に決定します。

    <?php

    namespace App\Listeners;

    use Illuminate\Auth\Events\Login;
    use Illuminate\Auth\Events\Logout;
    use Illuminate\Events\Dispatcher;

    class UserEventSubscriber
    {
        /**
         * ユーザーログインイベントの処理
         */
        public function handleUserLogin(Login $event): void {}

        /**
         * ユーザーログアウトイベントの処理
         */
        public function handleUserLogout(Logout $event): void {}

        /**
         * サブスクライバのリスナを登録
         *
         * @return array<string, string>
         */
        public function subscribe(Dispatcher $events): array
        {
            return [
                Login::class => 'handleUserLogin',
                Logout::class => 'handleUserLogout',
            ];
        }
    }

<a name="registering-event-subscribers"></a>
### イベントサブスクライバの登録

サブスクライバを書き終えたら、イベントディスパッチャで登録します。サブスクライバを登録するには、`Event`ファサードの`subscribe`メソッドを使用します。通常、これはアプリケーションの`AppServiceProvider`の`boot`メソッド内で行います。

    <?php

    namespace App\Providers;

    use App\Listeners\UserEventSubscriber;
    use Illuminate\Support\Facades\Event;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * アプリケーションの全サービスの初期起動処理
         */
        public function boot(): void
        {
            Event::subscribe(UserEventSubscriber::class);
        }
    }

<a name="testing"></a>
## Testing

イベントをディスパッチするコードをテストする場合、イベントのリスナを実際に実行しないように、Laravelへ指示したい場合があるでしょう。リスナのコードは、対応するイベントをディスパッチするコードとは別に、直接テストすることができるからです。もちろん、リスナ自体をテストするには、リスナインスタンスをインスタンス化し、テスト内で直接`handle`メソッドを呼び出せます。

`Event`ファサードの`fake`メソッドを使用し、リスナを実行しないでテスト対象のコードを実行し、`assertDispatched`、`assertNotDispatched`、`assertNothingDispatched`メソッドを使用してアプリケーションがどのイベントをディスパッチするかをアサートできます。

```php tab=Pest
<?php

use App\Events\OrderFailedToShip;
use App\Events\OrderShipped;
use Illuminate\Support\Facades\Event;

test('orders can be shipped', function () {
    Event::fake();

    // 注文の発送処理…

    // 一つのイベントをディスパッチするのをアサート
    Event::assertDispatched(OrderShipped::class);

    // 一つのイベントを２回ディスパッチするのをアサート
    Event::assertDispatched(OrderShipped::class, 2);

    // あるイベントをディスパッチしないことをアサート
    Event::assertNotDispatched(OrderFailedToShip::class);

    // イベントを全くディスパッチしないことをアサート
    Event::assertNothingDispatched();
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Events\OrderFailedToShip;
use App\Events\OrderShipped;
use Illuminate\Support\Facades\Event;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * 注文発送のテスト
     */
    public function test_orders_can_be_shipped(): void
    {
        Event::fake();

        // 注文の発送処理…

        // 一つのイベントをディスパッチするのをアサート
        Event::assertDispatched(OrderShipped::class);

        // 一つのイベントを２回ディスパッチするのをアサート
        Event::assertDispatched(OrderShipped::class, 2);

        // あるイベントをディスパッチしないことをアサート
        Event::assertNotDispatched(OrderFailedToShip::class);

        // イベントを全くディスパッチしないことをアサート
        Event::assertNothingDispatched();
    }
}
```

クロージャを`assertDispatched`や`assertNotDispatched`メソッドに渡すと、指定したその「真理値テスト」に合格するイベントが、ディスパッチされたことをアサートできます。指定真理値テストにパスするイベントが最低１つディスパッチされた場合、アサートは成功します。

    Event::assertDispatched(function (OrderShipped $event) use ($order) {
        return $event->order->id === $order->id;
    });

イベントリスナが指定イベントをリッスンしていることを単純にアサートしたい場合は、`assertListening`メソッドを使用してください。

    Event::assertListening(
        OrderShipped::class,
        SendShipmentNotification::class
    );

> [!WARNING]
> `Event::fake()`を呼び出した後は、イベントリスナが実行されることはありません。したがって、テストがイベントに依存するモデルファクトリを使用している場合、例えば、モデルの`creating`イベント中にUUIDを作成する場合、ファクトリを使用した**後**に、`Event::fake()`を呼び出す必要があります。

<a name="faking-a-subset-of-events"></a>
### イベントサブセットのFake

特定のイベントセットに対してのみイベントリスナをFakeしたい場合は、`fake`または`fakeFor`メソッドへそれらを渡してください。

```php tab=Pest
test('orders can be processed', function () {
    Event::fake([
        OrderCreated::class,
    ]);

    $order = Order::factory()->create();

    Event::assertDispatched(OrderCreated::class);

    // 他のイベントは普段通りディスパッチする
    $order->update([...]);
});
```

```php tab=PHPUnit
/**
 * 注文発送のテスト
 */
public function test_orders_can_be_processed(): void
{
    Event::fake([
        OrderCreated::class,
    ]);

    $order = Order::factory()->create();

    Event::assertDispatched(OrderCreated::class);

    // 他のイベントは普段通りディスパッチする
    $order->update([...]);
}
```

`except`メソッドを使用すると、指定イベント以外のすべてのイベントをFakeできます。

    Event::fake()->except([
        OrderCreated::class,
    ]);

<a name="scoped-event-fakes"></a>
### イベントFakeのスコープ

テストの一部分だけでイベントリスナをFakeしたい場合は、`fakeFor`メソッドを使用します。

```php tab=Pest
<?php

use App\Events\OrderCreated;
use App\Models\Order;
use Illuminate\Support\Facades\Event;

test('orders can be processed', function () {
    $order = Event::fakeFor(function () {
        $order = Order::factory()->create();

        Event::assertDispatched(OrderCreated::class);

        return $order;
    });

    // イベントを通常通りディスパッチし、オブザーバが実行される
    $order->update([...]);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Events\OrderCreated;
use App\Models\Order;
use Illuminate\Support\Facades\Event;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * 注文処理のテスト
     */
    public function test_orders_can_be_processed(): void
    {
        $order = Event::fakeFor(function () {
            $order = Order::factory()->create();

            Event::assertDispatched(OrderCreated::class);

            return $order;
        });

        // イベントを通常通りディスパッチし、オブザーバが実行される
        $order->update([...])
    }
}
```
