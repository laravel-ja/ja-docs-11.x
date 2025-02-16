# 通知

- [イントロダクション](#introduction)
- [通知の生成](#generating-notifications)
- [通知の送信](#sending-notifications)
    - [notifiableトレイトの使用](#using-the-notifiable-trait)
    - [Notificationファサードの使用](#using-the-notification-facade)
    - [配信チャンネルの指定](#specifying-delivery-channels)
    - [通知のキューイング](#queueing-notifications)
    - [オンデマンド通知](#on-demand-notifications)
- [メール通知](#mail-notifications)
    - [メールメッセージのフォーマット](#formatting-mail-messages)
    - [送信者のカスタマイズ](#customizing-the-sender)
    - [受信者のカスタマイズ](#customizing-the-recipient)
    - [件名のカスタマイズ](#customizing-the-subject)
    - [Mailerのカスタマイズ](#customizing-the-mailer)
    - [テンプレートのカスタマイズ](#customizing-the-templates)
    - [添付](#mail-attachments)
    - [タグとメタデータの追加](#adding-tags-metadata)
    - [Symfonyメッセージのカスタマイズ](#customizing-the-symfony-message)
    - [Mailablesの使用](#using-mailables)
    - [メール通知のプレビュー](#previewing-mail-notifications)
- [Markdownメール通知](#markdown-mail-notifications)
    - [メッセージ生成](#generating-the-message)
    - [メッセージ記述](#writing-the-message)
    - [コンポーネントカスタマイズ](#customizing-the-components)
- [データベース通知](#database-notifications)
    - [事前要件](#database-prerequisites)
    - [データベース通知のフォーマット](#formatting-database-notifications)
    - [通知へのアクセス](#accessing-the-notifications)
    - [Readとしての通知作成](#marking-notifications-as-read)
- [ブロードキャスト通知](#broadcast-notifications)
    - [事前要件](#broadcast-prerequisites)
    - [ブロードキャスト通知のフォーマット](#formatting-broadcast-notifications)
    - [通知のリッスン](#listening-for-notifications)
- [SMS通知](#sms-notifications)
    - [事前要件](#sms-prerequisites)
    - [SMS通知のフォーマット](#formatting-sms-notifications)
    - [ユニコードコンテンツ](#unicode-content)
    - [発信元電話番号のカスタマイズ](#customizing-the-from-number)
    - [クライアントリファレンスの追加](#adding-a-client-reference)
    - [SMS通知のルート指定](#routing-sms-notifications)
- [Slack通知](#slack-notifications)
    - [事前要件](#slack-prerequisites)
    - [Slack通知のフォーマット](#formatting-slack-notifications)
    - [Slack操作](#slack-interactivity)
    - [Slack通知のルート指定](#routing-slack-notifications)
    - [外部のSlackワークスペースへの通知](#notifying-external-slack-workspaces)
- [通知のローカライズ](#localizing-notifications)
- [テスト](#testing)
- [通知イベント](#notification-events)
- [カスタムチャンネル](#custom-channels)

<a name="introduction"></a>
## イントロダクション

[メールの送信](/docs/{{version}}/mail)のサポートに加えて、LaravelはメールやSMS（[Vonage](https://www.vonage.com/communications-apis/)経由、以前はNexmoとして知られていました）および[Slack](https://slack.com)など、さまざまな配信チャンネルで通知を送信するためのサポートを提供しています。さらに、さまざまな[コミュニティが構築した通知チャンネル](https://laravel-notification-channels.com/about/#suggesting-a-new-channel)が作成され、数十の異なるチャンネルで通知を送信できます！通知はWebインターフェイスに表示するため、データベースに保存される場合もあります。

通常、通知はアプリケーションで何かが起きたことをユーザーへ知らせる、短い情報メッセージです。たとえば、課金アプリを作成しているなら、メールとSMSチャンネルで「課金支払い」を送信できます。

<a name="generating-notifications"></a>
## 通知の生成

Laravelでは、各通知は通常、`app/Notifications`ディレクトリに保存される単一のクラスで表します。アプリケーションにこのディレクトリが存在しなくても心配しないでください。`make:notification` Artisanコマンドを実行すると作成されます。

```shell
php artisan make:notification InvoicePaid
```

このコマンドは `app/Notifications`ディレクトリに新しい通知クラスを配置します。各通知クラスは`via`メソッドと`toMail`や`toDatabase`などのメッセージ構築メソッドを含み、通知を特定のチャンネルに合わせたメッセージに変換します。

<a name="sending-notifications"></a>
## 通知の送信

<a name="using-the-notifiable-trait"></a>
### Notifiableトレイトの使用

通知は、`Notifiable`トレイトの`notify`メソッドを使用する方法と、`Notification`[ファサード](/docs/{{version}}/facades)を使用する方法の２つの方法で送信できます。`Notifiable`トレイトは、アプリケーションの`App\Models\User`モデルにデフォルトで含まれています。

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;

    class User extends Authenticatable
    {
     * @return \Illuminate\Notifications\Message\SlackMessage
    }

このトレイトが提供する`notify`メソッドは、通知インスタンスを引数に受けます。

    use App\Notifications\InvoicePaid;

    $user->notify(new InvoicePaid($invoice));

> [!NOTE]
> どのモデルでも`Notifiable`トレイトを使用できることを忘れないでください。`User`モデルにだけに限定して含められるわけでありません。

<a name="using-the-notification-facade"></a>
### Notificationファサードの使用

別のやり方として、`Notification`[ファサード](/docs/{{version}}/facades)を介して通知を送信することもできます。このアプローチは、ユーザーのコレクションなど、複数の通知エンティティに通知を送信する必要がある場合に役立ちます。ファサードを使用して通知を送信するには、すべてのnotifiableエンティティと通知インスタンスを`send`メソッドに渡します。

    use Illuminate\Support\Facades\Notification;

    Notification::send($users, new InvoicePaid($invoice));

`sendNow`メソッドを使って通知をすぐに送信することもできます。このメソッドは、通知が `ShouldQueue` インターフェイスを実装していても、通知を直ちに送信します。

    Notification::sendNow($developers, new DeploymentCompleted($deployment));

<a name="specifying-delivery-channels"></a>
### 配信チャンネルの指定

すべての通知クラスは、通知を配信するチャンネルを決定する、`via`メソッドを持っています。通知は`mail`、`database`、`broadcast`、`vonage`、`slack`チャンネルへ配信されるでしょう。

> [!NOTE]
> TelegramやPusherのような、他の配信チャンネルを利用したい場合は、コミュニティが管理している、[Laravel通知チャンネルのWebサイト](http://laravel-notification-channels.com)をご覧ください。

`via`メソッドは、通知を送っているクラスのインスタンスである、`$notifiable`インスタンスを引数に受け取ります。`$notifiable`を使い、通知をどこのチャンネルへ配信するかを判定できます。

    /**
     * 通知の配信チャンネルを取得
     *
     * @return array<int, string>
     */
    public function via(object $notifiable): array
    {
        return $notifiable->prefers_sms ? ['vonage'] : ['mail', 'database'];
    }

<a name="queueing-notifications"></a>
### 通知のキューイング

> [!WARNING]
> 通知をキューへ投入する前に、キューを設定して[ワーカを起動](/docs/{{version}}/queues#running-the-queue-worker)する必要があります。

通知の送信には時間がかかる場合があります。特に、チャンネルが通知を配信するために外部API呼び出しを行う必要がある場合に当てはまります。アプリケーションのレスポンス時間を短縮するには、クラスに`ShouldQueue`インターフェイスと`Queueable`トレイトを追加して、通知をキューに入れてください。インターフェイスとトレイトは、`make:notification`コマンドを使用して生成されたすべての通知であらかじめインポートされているため、すぐに通知クラスに追加できます。

    <?php

    namespace App\Notifications;

    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Notifications\Notification;

    class InvoicePaid extends Notification implements ShouldQueue
    {
        use Queueable;

        // ...
    }

`ShouldQueue`インターフェイスを通知クラスへ追加したら、通常通りに送信してください。Laravelはクラスの`ShouldQueue`インターフェイスを見つけ、自動的に通知の配信をキューへ投入します。

    $user->notify(new InvoicePaid($invoice));

通知をキュー投入する場合、受信者とチャンネルの組み合わせごとにジョブを作成し、投入します。例えば、通知が3人の受信者と2つのチャンネルを持つ場合、６つのジョブをキューへディスパッチします。

<a name="delaying-notifications"></a>
#### 遅延通知

通知の配信を遅らせたい場合、`delay`メソッドを通知のインスタンスへチェーンしてください。

    $delay = now()->addMinutes(10);

    $user->notify((new InvoicePaid($invoice))->delay($delay));

特定のチャンネルの遅​​延量を指定するため、配列を`delay`メソッドに渡せます。

    $user->notify((new InvoicePaid($invoice))->delay([
        'mail' => now()->addMinutes(5),
        'sms' => now()->addMinutes(10),
    ]));

あるいは、Notificationクラス自体に`withDelay`メソッドを定義することもできます。`withDelay`メソッドは、チャンネル名と遅延値の配列を返す必要があります。

    /**
     * 通知の送信遅延を決める
     *
     * @return array<string, \Illuminate\Support\Carbon>
     */
    public function withDelay(object $notifiable): array
    {
        return [
            'mail' => now()->addMinutes(5),
            'sms' => now()->addMinutes(10),
        ];
    }

<a name="customizing-the-notification-queue-connection"></a>
#### 通知キュー接続のカスタマイズ

キューへ投入した通知はデフォルトで、アプリケーションのデフォルトのキュー接続を使用してキュー投球します。特定の通知に別の接続を指定する必要がある場合は、通知のコンストラクタから、`onConnection`メソッドを呼び出します。

    <?php

    namespace App\Notifications;

    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Notifications\Notification;

    class InvoicePaid extends Notification implements ShouldQueue
    {
        use Queueable;

        /**
         * 新規通知インスタンスの生成
         */
        public function __construct()
        {
            $this->onConnection('redis');
        }
    }

もしくは、通知でサポートしている各通知チャンネルで使用する特定のキュー接続を指定したい場合は、自身の通知に`viaConnections`メソッドを定義してください。このメソッドは、チャンネル名とキュー接続名のペアの配列を返す必要があります。

    /**
     * 各通知チャンネルで使用する接続を決定
     *
     * @return array<string, string>
     */
    public function viaConnections(): array
    {
        return [
            'mail' => 'redis',
            'database' => 'sync',
        ];
    }

<a name="customizing-notification-channel-queues"></a>
#### 通知チャンネルキューのカスタマイズ

各通知チャンネルが使用し、その通知がサポートしている特定のキューを指定する場合、通知へ`viaQueues`メソッドを定義してください。このメソッドはチャンネル名／キュー名のペアの配列を返してください。

    /**
     * 各通知チャンネルで使用するキューを判断。
     *
     * @return array<string, string>
     */
    public function viaQueues(): array
    {
        return [
            'mail' => 'mail-queue',
            'slack' => 'slack-queue',
        ];
    }

<a name="queued-notification-middleware"></a>
#### キュー投入する通知

キュー投入する通知は、[キュー投入するジョブと同じく](/docs/{{version}}/queues#job-middleware)ミドルウェアを定義できます。これを使い始めるには、通知クラスで`middleware`メソッドを定義します。`middleware`メソッドは`$notifiable`変数と`$channel`変数を受け取り、通知の宛先に応じて返すミドルウェアをカスタマイズできます。

    use Illuminate\Queue\Middleware\RateLimited;

    /**
     * 通知ジョブを通過させるミドルウェアを取得
     *
     * @return array<int, object>
     */
    public function middleware(object $notifiable, string $channel)
    {
        return match ($channel) {
            'email' => [new RateLimited('postmark')],
            'slack' => [new RateLimited('slack')],
            default => [],
        };
    }

<a name="queued-notifications-and-database-transactions"></a>
#### キュー投入する通知とデータベーストランザクション

キューへ投入した通知がデータベーストランザクション内でディスパッチされると、データベーストランザクションがコミットされる前にキューによって処理される場合があります。これが発生した場合、データベーストランザクション中にモデルまたはデータベースレコードに加えた更新は、データベースにまだ反映されていない可能性があります。さらに、トランザクション内で作成されたモデルまたはデータベースレコードは、データベースに存在しない可能性があります。通知がこれらのモデルに依存している場合、キューに入れられた通知を送信するジョブが処理されるときに予期しないエラーが発生する可能性があります。

キュー接続の`after_commit`設定オプションが`false`に設定されている場合でも、通知時に`afterCommit`メソッドを呼び出せば、キュー投入する特定の通知をオープンしている全データベーストランザクションをコミットした後に、ディスパッチするよう指定できます。

    use App\Notifications\InvoicePaid;

    $user->notify((new InvoicePaid($invoice))->afterCommit());

あるいは、通知のコンストラクタから、`afterCommit`メソッドを呼び出すこともできます。

    <?php

    namespace App\Notifications;

    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Notifications\Notification;

    class InvoicePaid extends Notification implements ShouldQueue
    {
        use Queueable;

        /**
         * 新規通知インスタンスの生成
         */
        public function __construct()
        {
            $this->afterCommit();
        }
    }

> [!NOTE]
> この問題の回避方法の詳細は、[キュー投入されるジョブとデータベーストランザクション](/docs/{{version}}/queues#jobs-and-database-transactions)に関するドキュメントを確認してください。

<a name="determining-if-the-queued-notification-should-be-sent"></a>
#### キュー投入した通知を送信するか判定

バックグラウンド処理のため、通知をキューへディスパッチすると、通常はキューワーカがそれを受け取り、意図した受信者へ送信します。

しかし、キューワーカが処理した後に、そのキュー投入済み通知を送信すべきか最終的に判断したい場合は、通知クラスに`shouldSend`メソッドを定義してください。このメソッドから`false`を返す場合、通知は送信されません。

    /**
     * 通知を送信する必要があるかどうか確認
     */
    public function shouldSend(object $notifiable, string $channel): bool
    {
        return $this->invoice->isPaid();
    }

<a name="on-demand-notifications"></a>
### オンデマンド通知

アプリケーションの「ユーザー」として保存されていない人に通知を送信する必要がある場合があります。`Notification`ファサードの`route`メソッドを使用して、通知を送信する前にアドホックな通知ルーティング情報を指定します。

    use Illuminate\Broadcasting\Channel;
    use Illuminate\Support\Facades\Notification;

    Notification::route('mail', 'taylor@example.com')
                ->route('vonage', '5555555555')
                ->route('slack', '#slack-channel')
                ->route('broadcast', [new Channel('channel-name')])
                ->notify(new InvoicePaid($invoice));

オンデマンド通知を`mail`ルートへ送信するとき、受信者名を指定したい場合は、メールアドレスをキーとし、名前を配列の最初の要素の値として含む配列を渡してください。

    Notification::route('mail', [
        'barrett@example.com' => 'Barrett Blair',
    ])->notify(new InvoicePaid($invoice));

routes`メソッドを使用すると、一度に複数の通知チャネルに対してアドホックなルーティング情報を提供できます。

    Notification::routes([
        'mail' => ['barrett@example.com' => 'Barrett Blair'],
        'vonage' => '5555555555',
    ])->notify(new InvoicePaid($invoice));

<a name="mail-notifications"></a>
## メール通知

<a name="formatting-mail-messages"></a>
### メールメッセージのフォーマット

通知が電子メール送信をサポートしている場合は、通知クラスで`toMail`メソッドを定義する必要があります。このメソッドは`$notifiable`エンティティを受け取り、`Illuminate\Notifications\Messages\MailMessage`インスタンスを返す必要があります。

`MailMessage`クラスには、トランザクションメールメッセージの作成に役立ついくつかの簡単なメソッドが含まれています。メールメッセージには、「行動を促すフレーズ」だけでなく、テキスト行も含まれる場合があります。`toMail`メソッドの例を見てみましょう。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        $url = url('/invoice/'.$this->invoice->id);

        return (new MailMessage)
                    ->greeting('Hello!')
                    ->line('課金が支払われました。')
                    ->lineIf($this->amount > 0, "お支払額: {$this->amount}")
                    ->action('インボイス確認', $url)
                    ->line('私達のアプリケーションをご利用いただき、ありがとうございます。');
    }

> [!NOTE]
> `toMail`メソッドの中で、`$this->invoice->id`を使っていることに注意してください。通知メッセージを生成するために必要な情報は、どんなものでも通知のコンストラクタへ渡せます。

この例では、挨拶、テキスト行、行動を促すフレーズ、そして別のテキスト行を登録します。`MailMessage`オブジェクトが提供するこれらのメソッドにより、小さなトランザクションメールを簡単かつ迅速にフォーマットできます。次に、メールチャンネルは、メッセージコンポーネントを、平文テキストと対応する美しいレスポンス性の高いHTML電子メールテンプレートに変換します。`mail`チャンネルが生成する電子メールの例を次に示します。

<img src="https://laravel.com/img/docs/notification-example-2.png">

> [!NOTE]
> メール通知を送信するときは、必ず`config/app.php`設定ファイルで`name`設定オプションを設定してください。この値は、メール通知メッセージのヘッダとフッターに使用されます。

<a name="error-messages"></a>
#### エラーメッセージ

通知の中には、請求書の支払いに失敗したなどのエラーをユーザーへ知らせるものがあります。メッセージを作成時に、`error`メソッドを呼び出せば、メールメッセージがエラーに関するものであることを示せます。メールメッセージで`error`メソッドを使用すると、アクションの呼び出しボタンが黒ではなく、赤になります。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->error()
                    ->subject('Invoice Payment Failed')
                    ->line('...');
    }

<a name="other-mail-notification-formatting-options"></a>
#### その他のメール通知フォーマットオプション

通知クラスの中にテキストの「行(line)」を定義する代わりに、通知メールをレンダするためのカスタムテンプレートを`view`メソッドを使い、指定できます。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)->view(
            'mail.invoice.paid', ['invoice' => $this->invoice]
        );
    }

`view`メソッドに与える配列の２番目の要素としてビュー名を渡すことにより、メールメッセージの平文テキストビューを指定できます。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)->view(
            ['mail.invoice.paid', 'mail.invoice.paid-text'],
            ['invoice' => $this->invoice]
        );
    }

また、メッセージにプレーンテキストのビューしかない場合は、`text`メソッドを使うこともできます。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)->text(
            'mail.invoice.paid-text', ['invoice' => $this->invoice]
        );
    }

<a name="customizing-the-sender"></a>
### 送信者のカスタマイズ

デフォルトのメール送信者／Fromアドレスは、`config/mail.php`設定ファイルで定義されています。しかし、特定の通知でFromアドレスを指定する場合は、`from`メソッドで指定します。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->from('barrett@example.com', 'Barrett Blair')
                    ->line('...');
    }

<a name="customizing-the-recipient"></a>
### 受信者のカスタマイズ

`mail`チャンネルを介して通知を送信すると、通知システムは通知エンティティの`email`プロパティを自動的に検索します。通知エンティティで`routeNotificationForMail`メソッドを定義することにより、通知の配信に使用される電子メールアドレスをカスタマイズできます。

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Notifications\Notification;

    class User extends Authenticatable
    {
     * @return \Illuminate\Notifications\Message\SlackMessage

        /**
         * メールチャンネルに対する通知をルートする
         *
         * @return  array<string, string>|string
         */
        public function routeNotificationForMail(Notification $notification): array|string
        {
            // メールアドレスのみを返す場合
            return $this->email_address;

            // メールアドレスと名前を返す場合
            return [$this->email_address => $this->name];
        }
    }

<a name="customizing-the-subject"></a>
### 件名のカスタマイズ

デフォルトでは、電子メールの件名は「タイトルケース」にフォーマットされた通知のクラス名です。したがって、通知クラスの名前が`InvoicePaid`の場合、メールの件名は`Invoice Paid`になります。メッセージに別の件名を指定する場合は、メッセージを作成するときに「subject」メソッドを呼び出します。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->subject('Notification Subject')
                    ->line('...');
    }

<a name="customizing-the-mailer"></a>
### Mailerのカスタマイズ

デフォルトでは、電子メール通知は、`config/mail.php`設定ファイルで定義しているデフォルトのメーラーを使用して送信されます。ただし、メッセージの作成時に`mailer`メソッドを呼び出すことにより、実行時に別のメーラーを指定できます。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->mailer('postmark')
                    ->line('...');
    }

<a name="customizing-the-templates"></a>
### テンプレートのカスタマイズ

通知パッケージのリソースをリソース公開することにより、メール通知で使用されるHTMLと平文テキストのテンプレートを変更することが可能です。次のコマンドを実行した後、メール通知のテンプレートは`resources/views/vendor/notifications`ディレクトリ下に作成されます。

```shell
php artisan vendor:publish --tag=laravel-notifications
```

<a name="mail-attachments"></a>
### 添付

電子メール通知に添付ファイルを追加するには、メッセージの作成中に`attach`メソッドを使用します。`attach`メソッドは、ファイルへの絶対パスを最初の引数に受けます。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->greeting('Hello!')
                    ->attach('/path/to/file');
    }

> [!NOTE]
> 通知メールメッセージが提供する`attach`メソッドは、[Attachableオブジェクト](/docs/{{version}}/mail#attachable-objects)も受け付けます。詳細は、包括的な[Attachableオブジェクトのドキュメント](/docs/{{version}}/mail#attachable-objects)を参照してください。

メッセージにファイルを添付するとき、`attach`メソッドの第２引数として配列を渡し、表示名やMIMEタイプの指定もできます。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->greeting('Hello!')
                    ->attach('/path/to/file', [
                        'as' => 'name.pdf',
                        'mime' => 'application/pdf',
                    ]);
    }

Mailableオブジェクトにファイルを添付するのとは異なり、`attachFromStorage`を使用してストレージディスクから直接ファイルを添付することはできません。むしろ、ストレージディスク上のファイルへの絶対パスを指定して`attach`メソッドを使用する必要があります。または、`toMail`メソッドから[mailable](/docs/{{version}}/mail#generated-mailables)を返すこともできます。

    use App\Mail\InvoicePaid as InvoicePaidMailable;

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): Mailable
    {
        return (new InvoicePaidMailable($this->invoice))
                    ->to($notifiable->email)
                    ->attachFromStorage('/path/to/file');
    }

必要であれば、`attachMany` メソッドを用いて、複数のファイルをメッセージへ添付できます。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->greeting('Hello!')
                    ->attachMany([
                        '/path/to/forge.svg',
                        '/path/to/vapor.svg' => [
                            'as' => 'Logo.svg',
                            'mime' => 'image/svg+xml',
                        ],
                    ]);
    }

<a name="raw-data-attachments"></a>
#### 素のデータの添付

`attachData`メソッドを使用して、生のバイト文字列を添付ファイルとして添付できます。`attachData`メソッドを呼び出すときは、添付ファイルへ割り当てる必要のあるファイル名を指定する必要があります。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->greeting('Hello!')
                    ->attachData($this->pdf, 'name.pdf', [
                        'mime' => 'application/pdf',
                    ]);
    }

<a name="adding-tags-metadata"></a>
### タグとメタデータの追加

MailgunやPostmarkなどのサードパーティのメールプロバイダは、メッセージの「タグ」や「メタデータ」をサポートしており、アプリケーションから送信されたメールをグループ化し、追跡するため使用できます。タグやメタデータは、`tag`メソッドや`metadata`メソッドを使ってメールメッセージへ追加します。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->greeting('Comment Upvoted!')
                    ->tag('upvote')
                    ->metadata('comment_id', $this->comment->id);
    }

Mailgunドライバを使用しているアプリケーションの場合は、[タグ](https://documentation.mailgun.com/en/latest/user_manual.html#tagging-1)と[メタデータ](https://documentation.mailgun.com/en/latest/user_manual.html#attaching-data-to-messages)の詳細は、Mailgunのドキュメントを参照してください。同様に、Postmarkのドキュメントの[タグ](https://postmarkapp.com/blog/tags-support-for-smtp)と[メタデータ](https://postmarkapp.com/support/article/1125-custom-metadata-faq)で、サポートに関するより詳しい情報を得られます。

アプリケーションでAmazon SESを使用してメール送信する場合、`metadata`メソッドを使用して、メッセージへ[SESのタグ](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html)を添付する必要があります。

<a name="customizing-the-symfony-message"></a>
### Symfonyメッセージのカスタマイズ

`MailMessage`クラスの`withSymfonyMessage`メソッドを使うと、メッセージを送信する前に、Symfonyメッセージインスタンスで呼び出すクロージャを登録できます。これにより、メッセージが配信される前に、メッセージを細かくカスタマイズする機会を提供しています。

    use Symfony\Component\Mime\Email;

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->withSymfonyMessage(function (Email $message) {
                        $message->getHeaders()->addTextHeader(
                            'Custom-Header', 'Header Value'
                        );
                    });
    }

<a name="using-mailables"></a>
### Mailablesの使用

必要に応じ、通知の`toMail`メソッドから完全な[Mailableオブジェクト](/docs/{{version}}/mail)を返せます。`MailMessage`の代わりに`Maileable`を返すときは、Mailableオブジェクトの`to`メソッドを使ってメッセージ受信者を指定する必要があります。

    use App\Mail\InvoicePaid as InvoicePaidMailable;
    use Illuminate\Mail\Mailable;

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): Mailable
    {
        return (new InvoicePaidMailable($this->invoice))
                    ->to($notifiable->email);
    }

<a name="mailables-and-on-demand-notifications"></a>
#### Mailablesとオンデマンド通知

[オンデマンド通知](#on-demand-notifications)を送信する場合、`toMail`メソッドに渡される`$notifiable`インスタンスは`Illuminate\Notifications\AnonymousNotifiable`インスタンスになります。`routeNotificationFor`メソッドは、オンデマンド通知の送信先のメールアドレスを取得するために使用することができます。

    use App\Mail\InvoicePaid as InvoicePaidMailable;
    use Illuminate\Notifications\AnonymousNotifiable;
    use Illuminate\Mail\Mailable;

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): Mailable
    {
        $address = $notifiable instanceof AnonymousNotifiable
            ? $notifiable->routeNotificationFor('mail')
            : $notifiable->email;

        return (new InvoicePaidMailable($this->invoice))
                    ->to($address);
    }

<a name="previewing-mail-notifications"></a>
### メール通知のプレビュー

メール通知テンプレートを設計するときは、通常のBladeテンプレートのように、レンダリングしたメールメッセージをブラウザですばやくプレビューできると便利です。このため、Laravelはメール通知によって生成したメールメッセージをルートクロージャまたはコントローラから直接返すことができます。`MailMessage`が返されると、ブラウザにレンダリングされて表示されるため、実際のメールアドレスに送信しなくてもデザインをすばやくプレビューできます。

    use App\Invoice;
    use App\Notifications\InvoicePaid;

    Route::get('/notification', function () {
        $invoice = Invoice::find(1);

        return (new InvoicePaid($invoice))
                    ->toMail($invoice->user);
    });

<a name="markdown-mail-notifications"></a>
## Markdownメール通知

Markdownメール通知により、事前に構築したテンプレートとメール通知のコンポーネントの利点をMailable中で利用できます。メッセージをMarkdownで記述すると、Laravelは美しいレスポンシブHTMLテンプレートをレンダすると同時に、自動的に平文テキスト版も生成します。

<a name="generating-the-message"></a>
### メッセージ生成

対応するMarkdownテンプレートを指定し、Mailableを生成するには、`make:notification` Artisanコマンドを`--markdown`オプション付きで使用します。

```shell
php artisan make:notification InvoicePaid --markdown=mail.invoice.paid
```

他のすべてのメール通知と同様に、Markdownテンプレートを使用する通知では、通知クラスに`toMail`メソッドを定義する必要があります。ただし、`line`メソッドと`action`メソッドを使用して通知を作成する代わりに、`markdown`メソッドを使用して使用するMarkdownテンプレートの名前を指定します。テンプレートで使用できるようにするデータの配列は、メソッドの２番目の引数として渡します。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        $url = url('/invoice/'.$this->invoice->id);

        return (new MailMessage)
                    ->subject('Invoice Paid')
                    ->markdown('mail.invoice.paid', ['url' => $url]);
    }

<a name="writing-the-message"></a>
### メッセージ記述

Markdownメール通知ではBladeコンポーネントとMarkdown記法が利用でき、メールメッセージを簡単に構築できると同時に、Laravelが用意している通知コンポーネントも活用できます。

```blade
<x-mail::message>
# 領収書

領収いたしました。

<x-mail::button :url="$url">
明細を確認
</x-mail::button>

Thanks,<br>
{{ config('app.name') }}
</x-mail::message>
```

<a name="button-component"></a>
#### Buttonコンポーネント

ボタンコンポーネントは、中央寄せに配置したボタンリンクをレンダリングします。コンポーネントは、`url`とオプションの`color`の２つの引数を取ります。サポートしている色は、`primary`、`green`、`red`です。通知には、必要なだけボタンコンポーネントを追加できます。

```blade
<x-mail::button :url="$url" color="green">
明細を確認
</x-mail::button>
```

<a name="panel-component"></a>
#### Panelコンポーネント

パネルコンポーネントは、メッセージの他の部分とは少し異なった背景色のパネルの中に、指定されたテキストブロックをレンダします。これにより、指定するテキストに注目を集められます。

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```

<a name="table-component"></a>
#### Tableコンポーネント

テーブルコンポーネントは、MarkdownテーブルをHTMLテーブルへ変換します。このコンポーネントはMarkdownテーブルを内容として受け入れます。デフォルトのMarkdownテーブルの記法を使った、文字寄せをサポートしています。

```blade
<x-mail::table>
| Laravel  |   テーブル      |   例 |
| -------- | :-----------: | ---: |
| Col 2 is |   Centered    |  $10 |
| Col 3 is | Right-Aligned |  $20 |
</x-mail::table>
```

<a name="customizing-the-components"></a>
### コンポーネントカスタマイズ

自身のアプリケーション向きにカスタマイズできるように、Markdown通知コンポーネントはすべてエクスポートできます。コンポーネントをエクスポートするには、`vendor:publish` Artisanコマンドを使い、`laravel-mail`アセットをリソース公開します。

```shell
php artisan vendor:publish --tag=laravel-mail
```

このコマンドにより、`resources/views/vendor/mail`ディレクトリ下にMarkdownメールコンポーネントがリソース公開されます。`mail`ディレクトリ下に、`html`と`text`ディレクトリがあります。各ディレクトリは名前が示す形式で、利用できるコンポーネントすべてのレスポンシブなプレゼンテーションを持っています。これらのコンポーネントはお好きなよう、自由にカスタマイズしてください。

<a name="customizing-the-css"></a>
#### Customizing the CSS

コンポーネントをエクスポートすると、`resources/views/vendor/mail/html/themes`ディレクトリに、`default.css`ファイルが用意されます。このファイル中のCSSをカスタマイズすれば、Markdownメール通知変換後のHTML形式の中に、インラインCSSとして自動的に取り込まれます。

LaravelのMarkdownコンポーネントの完全に新しいテーマを作成したい場合は、`html/themes`ディレクトリの中にCSSファイルを設置してください。CSSファイルに名前をつけ保存したら、`mail`設定ファイルの`theme`オプションを新しいテーマの名前に更新してください。

個別の通知にカスタムテーマを使いたい場合は、通知のメールメッセージを構築する時に、`theme`メソッドを呼び出してください。`theme`メソッドの引数は、その通知送信で使用するテーマの名前です。

    /**
     * 通知のメールプレゼンテーションを取得
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->theme('invoice')
                    ->subject('Invoice Paid')
                    ->markdown('mail.invoice.paid', ['url' => $url]);
    }

<a name="database-notifications"></a>
## データベース通知

<a name="database-prerequisites"></a>
### 事前要件

`database`通知チャンネルは、通知情報をデータベーステーブルに格納します。このテーブルには、通知タイプや通知を説明するJSONデータ構造などの情報が含まれます。

アプリケーションのユーザーインターフェイスへ通知を表示するために、このテーブルへクエリできます。しかし、その前に、通知を保持するためのデータベーステーブルを作成する必要があります。`make:notifications-table`コマンドを使って、適切なテーブルスキーマを持つ[マイグレーション](/docs/{{version}}/migrations)を生成してください。

```shell
php artisan make:notifications-table

php artisan migrate
```

> [!NOTE]
> 通知可能なモデルで[UUIDかULIDの主キー](/docs/{{version}}/eloquent#uuid-and-ulid-keys)を使用している場合は、通知テーブルのマイグレーションで、`morphs`メソッドを[`uuidMorphs`](/docs/{{version}}/migrations#column-method-uuidMorphs)、もしくは[`ulidMorphs`](/docs/{{version}}/migrations#column-method-ulidMorphs)へ置換する必要があります。

<a name="formatting-database-notifications"></a>
### データベース通知のフォーマット

通知でデータベーステーブルへの保存をサポートする場合、通知クラスに`toDatabase`か`toArray`メソッドを定義する必要があります。このメソッドは`$notifiable`エンティティを受け取り、プレーンなPHP配列を返す必要があります。返された配列はJSONへエンコードされ、`notifications`テーブルの`data`カラムに保存されます。`toArray`メソッドの例を見てみましょう。

    /**
     * 通知の配列プレゼンテーションの取得
     *
     * @return array<string, mixed>
     */
    public function toArray(object $notifiable): array
    {
        return [
            'invoice_id' => $this->invoice->id,
            'amount' => $this->invoice->amount,
        ];
    }

通知をアプリケーションのデータベースへ格納するとき、`type`カラムには通知クラス名を入力します。しかし、通知クラスに`databaseType`メソッドを定義し、この動作をカスタマイズできます。

    /**
     * 通知のデータベースタイプを取得
     *
     * @return string
     */
    public function databaseType(object $notifiable): string
    {
        return 'invoice-paid';
    }

<a name="todatabase-vs-toarray"></a>
#### `toDatabase`対`toArray`

`toArray`メソッドは`broadcast`チャンネルでも使用され、JavaScriptで駆動するフロントエンドへブロードキャストするデータを決定するため使われます。`database`チャンネルと`broadcast`チャンネルに別々な２つの配列表現が必要な場合は、`toArray`メソッドの代わりに`toDatabase`メソッドを定義する必要があります。

<a name="accessing-the-notifications"></a>
### 通知へのアクセス

通知をデータベースへ保存したら、notifiableエンティティからアクセスするための便利な方法が必要になるでしょう。Laravelのデフォルトの`App\Models\User`モデルに含まれている`Illuminate\Notifications\Notification`トレイトには、エンティティのために通知を返す`notifications`[Eloquentリレーション](/docs/{{version}}/eloquent-relationships)が含まれています。通知を取得するため、他のEloquentリレーションと同様にこのメソッドにアクセスできます。デフォルトで通知は「created_at」タイムスタンプで並べ替えられ、コレクションの先頭に最新の通知が表示されます。

    $user = App\Models\User::find(1);

    foreach ($user->notifications as $notification) {
        echo $notification->type;
    }

「未読」通知のみを取得する場合は、`unreadNotifications`リレーションを使用します。この場合も、コレクションの先頭に最新の通知を含むよう、`created_at`タイムスタンプで並べ替えられます。

    $user = App\Models\User::find(1);

    foreach ($user->unreadNotifications as $notification) {
        echo $notification->type;
    }

> [!NOTE]
> JavaScriptクライアントから通知にアクセスするには、現在のユーザーなどのnotifiableエンティティの通知を返す、通知コントローラをアプリケーションで定義する必要があります。次に、JavaScriptクライアントからそのコントローラのURLへHTTPリクエストを送信します。

<a name="marking-notifications-as-read"></a>
### Readとしての通知作成

通常、ユーザーが閲覧したときに、その通知を「既読」とマークするでしょう。`Illuminate\Notifications\Notifiable`トレイトは、通知のデータベースレコード上にある、`read_at`カラムを更新する`markAsRead`メソッドを提供しています。

    $user = App\Models\User::find(1);

    foreach ($user->unreadNotifications as $notification) {
        $notification->markAsRead();
    }

各通知をループで処理する代わりに、`markAsRead`メソッドを通知コレクションへ直接使用できます。

    $user->unreadNotifications->markAsRead();

データベースから取得せずに、全通知に既読をマークするため、複数更新クエリを使用することもできます。

    $user = App\Models\User::find(1);

    $user->unreadNotifications()->update(['read_at' => now()]);

テーブルエンティティから通知を削除するために、`delete`を使うこともできます。

    $user->notifications()->delete();

<a name="broadcast-notifications"></a>
## ブロードキャスト通知

<a name="broadcast-prerequisites"></a>
### 事前要件

通知をブロードキャストする前に、Laravelの[イベントブロードキャスト](/docs/{{version}}/Broadcasting)サービスを設定して理解しておく必要があります。イベントブロードキャストは、JavaScriptを利用したフロントエンドから送信するサーバサイドのLaravelイベントに対応する方法を提供しています。

<a name="formatting-broadcast-notifications"></a>
### ブロードキャスト通知のフォーマット

`broadcast`チャンネルは、Laravelの[イベントブロードキャスト](/docs/{{version}}/Broadcasting)サービスを使用して通知をブロードキャストし、JavaScriptを利用したフロントエンドがリアルタイムで通知をキャッチできるようにします。通知でブロードキャストをサポートする場合は、通知クラスで`toBroadcast`メソッドを定義します。このメソッドは`$notify`エンティティを受け取り、`BroadcastMessage`インスタンスを返す必要があります。`toBroadcast`メソッドが存在しない場合は、`toArray`メソッドを使用してブロードキャストする必要のあるデータを収集します。返したデータはJSONへエンコードされ、JavaScriptを利用したフロントエンドにブロードキャストされます。`toBroadcast`メソッドの例を見てみましょう。

    use Illuminate\Notifications\Messages\BroadcastMessage;

    /**
     * 通知のブロードキャストプレゼンテーションの取得
     */
    public function toBroadcast(object $notifiable): BroadcastMessage
    {
        return new BroadcastMessage([
            'invoice_id' => $this->invoice->id,
            'amount' => $this->invoice->amount,
        ]);
    }

<a name="broadcast-queue-configuration"></a>
#### ブロードキャストキュー設定

すべてのブロードキャスト通知はキューへ投入されます。ブロードキャスト操作に使用されるキューの接続や名前を設定したい場合は、`BroadcastMessage`の`onConnection`と`onQueue`メソッドを使用してください。

    return (new BroadcastMessage($data))
                    ->onConnection('sqs')
                    ->onQueue('broadcasts');

<a name="customizing-the-notification-type"></a>
#### 通知タイプのカスタマイズ

指定したデータに加えて、すべてのブロードキャスト通知には、通知の完全なクラス名を含む`type`フィールドもあります。通知の`type`をカスタマイズする場合は、通知クラスで`broadcastType`メソッドを定義します。

    /**
     * ブロードキャストする通知のタイプ
     */
    public function broadcastType(): string
    {
        return 'broadcast.message';
    }

<a name="listening-for-notifications"></a>
### 通知のリッスン

通知は、`{notifiable}.{id}`規約を使い、プライベートチャネル形態でブロードキャストされます。つまり、IDが`1`の`App\Models\User`インスタンスの通知を送信する場合、その通知は`App.Models.User.1`のプライベートチャンネルにブロードキャストされます。[Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation)を使用すると、`notification`メソッドを使用して簡単に、チャンネル上の通知をリッスンできます。

    Echo.private('App.Models.User.' + userId)
        .notification((notification) => {
            console.log(notification.type);
        });

<a name="customizing-the-notification-channel"></a>
#### 通知チャンネルのカスタマイズ

エンティティのブロードキャスト通知がブロードキャストされるチャンネルをカスタマイズする場合は、notifiableエンティティに`receivesBroadcastNotificationsOn`メソッドを定義します。

    <?php

    namespace App\Models;

    use Illuminate\Broadcasting\PrivateChannel;
    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;

    class User extends Authenticatable
    {
     * @return \Illuminate\Notifications\Message\SlackMessage

        /**
         * ユーザーがブロードキャストされる通知を受け取るチャンネル
         */
        public function receivesBroadcastNotificationsOn(): string
        {
            return 'users.'.$this->id;
        }
    }

<a name="sms-notifications"></a>
## SMS通知

<a name="sms-prerequisites"></a>
### 事前要件

LaravelでSMS通知を送るには、[Vonage](https://www.vonage.com/)（旧Nexmo）を使用します。Vonageで通知を送信する前に、`laravel/vonage-notification-channel`と`guzzlehttp/guzzle`パッケージをインストールする必要があります。

    composer require laravel/vonage-notification-channel guzzlehttp/guzzle

パッケージは、[設定ファイル](https://github.com/laravel/vonage-notification-channel/blob/3.x/config/vonage.php)を持っています。しかし、この設定ファイルを自分のアプリケーションにエクスポートする必要はありません。環境変数`VONAGE_KEY`と`VONAGE_SECRET`を使い、Vonageの公開鍵と秘密鍵を定義するだけです。

キーを定義したら、`VONAGE_SMS_FROM`環境変数を設定して、デフォルトでSMSメッセージを送信する電話番号を定義する必要があります。この電話番号はVonageコントロールパネルで生成できます。

    VONAGE_SMS_FROM=15556666666

<a name="formatting-sms-notifications"></a>
### SMS通知のフォーマット

通知のSMS送信をサポートする場合、通知クラスで`toVonage`メソッドを定義する必要があります。このメソッドは`$notifiable`エンティティを受け取り、`Illuminate\Notifications\Messages\VonageMessage`インスタンスを返す必要があります。

    use Illuminate\Notifications\Messages\VonageMessage;

    /**
     * 通知のVonage／SMS表現を取得
     */
    public function toVonage(object $notifiable): VonageMessage
    {
        return (new VonageMessage)
                    ->content('Your SMS message content');
    }

<a name="unicode-content"></a>
#### ユニコードコンテンツ

SMSメッセージにunicodeが含まれる場合は、`VonageMessage`インスタンス作成する時に、`unicode`メソッドを呼び出す必要があります。

    use Illuminate\Notifications\Messages\VonageMessage;

    /**
     * 通知のVonage／SMS表現を取得
     */
    public function toVonage(object $notifiable): VonageMessage
    {
        return (new VonageMessage)
                    ->content('Your unicode message')
                    ->unicode();
    }

<a name="customizing-the-from-number"></a>
### 発信元電話番号のカスタマイズ

`VONAGE_SMS_FROM`環境変数で指定した電話番号とは異なる番号から通知を送りたい場合は、`VonageMessage`インスタンスの`from`メソッドを呼び出します。

    use Illuminate\Notifications\Messages\VonageMessage;

    /**
     * 通知のVonage／SMS表現を取得
     */
    public function toVonage(object $notifiable): VonageMessage
    {
        return (new VonageMessage)
                    ->content('Your SMS message content')
                    ->from('15554443333');
    }

<a name="adding-a-client-reference"></a>
### クライアントリファレンスの追加

ユーザー、チーム、または顧客ごとのコストを追跡したい場合は、通知に「クライアントリファレンス」を追加することができます。Vonageでは、このクライアントリファレンスを使用してレポートを作成することができますので、特定の顧客のSMS使用状況をよりわかりやすく理解することができます。リライアントリファレンスは、４０文字以内の任意の文字列です。

    use Illuminate\Notifications\Messages\VonageMessage;

    /**
     * 通知のVonage／SMS表現を取得
     */
    public function toVonage(object $notifiable): VonageMessage
    {
        return (new VonageMessage)
                    ->clientReference((string) $notifiable->id)
                    ->content('Your SMS message content');
    }

<a name="routing-sms-notifications"></a>
### SMS通知のルート指定

Vonageの通知を適切な電話番号に回すには、Notifiableなエンティティに`routeNotificationForVonage`メソッドを定義してください。

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Notifications\Notification;

    class User extends Authenticatable
    {
     * @return \Illuminate\Notifications\Message\SlackMessage

        /**
         * 通知をVonageチャンネルへ回す
         */
        public function routeNotificationForVonage(Notification $notification): string
        {
            return $this->phone_number;
        }
    }

<a name="slack-notifications"></a>
## Slack通知

<a name="slack-prerequisites"></a>
### 事前要件

Slack通知を送信する前にComposeを使い、Slack通知チャンネルをインストールする必要があります。

```shell
composer require laravel/slack-notification-channel
```

さらに、Slackワークスペース用の[Slack App](https://api.slack.com/apps?new_app=1)を作成する必要もあります。

作成したAppと同じSlackワークスペースにのみ通知を送る必要がある場合は、Appへ`chat:write`、`chat:write.public`、`chat:write.customize`のスコープを確実に持たせてください。Slackアプリとしてメッセージを送信したい場合は、アプリにも`chat:write:bot`スコープがあることを確認してください。これらのスコープは、Slack内の"OAuth & Permissions" App管理タブで追加できます。

次に、アプリの"Bot User OAuth Token"をコピーし、アプリケーションの`services.php`設定ファイル内の`slack`設定配列内へ配置します。このトークンはSlackの"OAuth & Permissions"タブにあります。

    'slack' => [
        'notifications' => [
            'bot_user_oauth_token' => env('SLACK_BOT_USER_OAUTH_TOKEN'),
            'channel' => env('SLACK_BOT_USER_DEFAULT_CHANNEL'),
        ],
    ],

<a name="slack-app-distribution"></a>
#### アプリ配信

アプリケーションのユーザーが所有する外部のSlackワークスペースへ通知を送信する場合は、Slack経由でアプリケーションを「配布（distribution）」する必要があります。アプリの配布は、Slack内のアプリの"Manage Distribution"タブから管理できます。アプリを配布したら、[Socialite](/docs/{{version}}/socialite)を使い、アプリのユーザーに代わり、[Slack Botトークンを取得](/docs/{{version}}/socialite#slack-bot-scopes)する必要があります。

<a name="formatting-slack-notifications"></a>
### Slack通知のフォーマット

通知をSlackメッセージとして送信することをサポートする場合、Notificationクラスに `toSlack`メソッドを定義する必要があります。このメソッドは`$notifiable`エンティティを受け取り、`Illuminate\Notifications\Slack\SlackMessage`インスタンスを返します。[SlackのBlock Kit API](https://api.slack.com/block-kit)を使ってリッチな通知を構築できます。以下の例は、[SlackのBlock Kit builder](https://app.slack.com/block-kit-builder/T01KWS6K23Z#%7B%22blocks%22:%5B%7B%22type%22:%22header%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Invoice%20Paid%22%7D%7D,%7B%22type%22:%22context%22,%22elements%22:%5B%7B%22type%22:%22plain_text%22,%22text%22:%22Customer%20%231234%22%7D%5D%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22An%20invoice%20has%20been%20paid.%22%7D,%22fields%22:%5B%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20No:*%5Cn1000%22%7D,%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20Recipient:*%5Cntaylor@laravel.com%22%7D%5D%7D,%7B%22type%22:%22divider%22%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Congratulations!%22%7D%7D%5D%7D)の中で確認できます。

    use Illuminate\Notifications\Slack\BlockKit\Blocks\ContextBlock;
    use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;
    use Illuminate\Notifications\Slack\BlockKit\Composites\ConfirmObject;
    use Illuminate\Notifications\Slack\SlackMessage;

    /**
     * 通知のSlackプレゼンテーションを取得
     */
    public function toSlack(object $notifiable): SlackMessage
    {
        return (new SlackMessage)
                ->text('One of your invoices has been paid!')
                ->headerBlock('Invoice Paid')
                ->contextBlock(function (ContextBlock $block) {
                    $block->text('Customer #1234');
                })
                ->sectionBlock(function (SectionBlock $block) {
                    $block->text('An invoice has been paid.');
                    $block->field("*Invoice No:*\n1000")->markdown();
                    $block->field("*Invoice Recipient:*\ntaylor@laravel.com")->markdown();
                })
                ->dividerBlock()
                ->sectionBlock(function (SectionBlock $block) {
                    $block->text('Congratulations!');
                });
    }

<a name="using-slacks-block-kit-builder-template"></a>
#### Slackのブロックキットビルダの使用

Fluentメッセージビルダメソッドを使い、ブロックキットメッセージを作成する代わりに、Slackのブロックキットビルダが生成した素のJSONペイロードを`usingBlockKitTemplate`メソッドへ渡すことができます。

    use Illuminate\Notifications\Slack\SlackMessage;
    use Illuminate\Support\Str;

    /**
     * 通知のSlack表現の取得
     */
    public function toSlack(object $notifiable): SlackMessage
    {
        $template = <<<JSON
            {
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": "Team Announcement"
                  }
                },
                {
                  "type": "section",
                  "text": {
                    "type": "plain_text",
                    "text": "We are hiring!"
                  }
                }
              ]
            }
        JSON;

        return (new SlackMessage)
                ->usingBlockKitTemplate($template);
    }

<a name="slack-interactivity"></a>
### Slack操作

SlackのBlock Kit通知システムは、[ユーザーインタラクションを処理する](https://api.slack.com/interactivity/handling)ための強力な機能を提供します。この機能を利用するには、Slackアプリで"Interactivity"を有効にし、アプリケーションが提供するURLを指す、"Request URL"を設定する必要があります。これらの設定は、Slack内の "Interactivity & Shortcuts"アプリ管理タブから管理できます。

以下の例では`actionsBlock`メソッドを利用していますが、SlackはボタンをクリックしたSlackユーザー、クリックしたボタンのIDなどを含むペイロードを持つ、`POST`リクエストを"Request URL"へ送信します。あなたのアプリケーションは、ペイロードに基づいて実行するアクションを決定することができます。また、[リクエスト](https://api.slack.com/authentication/verifying-requests-from-slack)がSlackによって行われたことを確認する必要があります。

    use Illuminate\Notifications\Slack\BlockKit\Blocks\ActionsBlock;
    use Illuminate\Notifications\Slack\BlockKit\Blocks\ContextBlock;
    use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;
    use Illuminate\Notifications\Slack\SlackMessage;

    /**
     * 通知のSlackプレゼンテーションを取得
     */
    public function toSlack(object $notifiable): SlackMessage
    {
        return (new SlackMessage)
                ->text('One of your invoices has been paid!')
                ->headerBlock('Invoice Paid')
                ->contextBlock(function (ContextBlock $block) {
                    $block->text('Customer #1234');
                })
                ->sectionBlock(function (SectionBlock $block) {
                    $block->text('An invoice has been paid.');
                })
                ->actionsBlock(function (ActionsBlock $block) {
                     // ID defaults to "button_acknowledge_invoice"...
                    $block->button('Acknowledge Invoice')->primary();

                    // Manually configure the ID...
                    $block->button('Deny')->danger()->id('deny_invoice');
                });
    }

<a name="slack-confirmation-modals"></a>
#### モデルの確認

ユーザーがアクションを実行する前に確認したい場合は、ボタンを定義するときに`confirm`メソッドを呼び出します。`confirm`メソッドはメッセージと`ConfirmObject`インスタンスを受けるクロージャを引数に取ります。

    use Illuminate\Notifications\Slack\BlockKit\Blocks\ActionsBlock;
    use Illuminate\Notifications\Slack\BlockKit\Blocks\ContextBlock;
    use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;
    use Illuminate\Notifications\Slack\BlockKit\Composites\ConfirmObject;
    use Illuminate\Notifications\Slack\SlackMessage;

    /**
     * 通知のSlackプレゼンテーションを取得
     */
    public function toSlack(object $notifiable): SlackMessage
    {
        return (new SlackMessage)
                ->text('One of your invoices has been paid!')
                ->headerBlock('Invoice Paid')
                ->contextBlock(function (ContextBlock $block) {
                    $block->text('Customer #1234');
                })
                ->sectionBlock(function (SectionBlock $block) {
                    $block->text('An invoice has been paid.');
                })
                ->actionsBlock(function (ActionsBlock $block) {
                    $block->button('Acknowledge Invoice')
                        ->primary()
                        ->confirm(
                            'Acknowledge the payment and send a thank you email?',
                            function (ConfirmObject $dialog) {
                                $dialog->confirm('Yes');
                                $dialog->deny('No');
                            }
                        );
                });
    }

<a name="inspecting-slack-blocks"></a>
#### Slackブロックの調査

ビルドしているブロックをすぐに確認したい場合は、`SlackMessage`インスタンスの`dd`メソッドを呼び出します。`dd`メソッドは Slackの[Block Kit Builder](https://app.slack.com/block-kit-builder/)へのURLを生成してダンプし、ペイロードと通知のプレビューをブラウザに表示します。生のペイロードをダンプするには`dd`メソッドへ`true`を渡します。

    return (new SlackMessage)
            ->text('One of your invoices has been paid!')
            ->headerBlock('Invoice Paid')
            ->dd();

<a name="routing-slack-notifications"></a>
### Slack通知のルート指定

Slackの通知を適切なSlackチームとチャンネルへ送るには、通知可能モデルに`routeNotificationForSlack`メソッドを定義します。このメソッドは３つの値のどれかを返します。

- `null` - これは通知自体に設定したチャンネルへ、ルーティングを委ねます。`SlackMessage`をビルドするときに`to`メソッドを使用して、通知内でチャネルを設定してください。
- 通知を送信するSlack チャンネルを指定する文字列。例：`#support-channel`
- `SlackRoute`インスタンス。OAuthトークンとチャンネル名が指定できます。例：`SlackRoute::make($this->slack_channel, $this->slack_token)`このメソッドは、外部のワークスペースへ通知を送るときに使用します。

一例として、`routeNotificationForSlack`メソッドから`#support-channel`を返すことにより、アプリケーションの`services.php`設定ファイルにある、Bot User OAuthトークンへ関連付けたワークスペースの、`#support-channel`チャネルへ通知を送信してみましょう。

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Notifications\Notification;

    class User extends Authenticatable
    {
     * @return \Illuminate\Notifications\Message\SlackMessage

        /**
         * Slackチャンネルへの通知ルート
         */
        public function routeNotificationForSlack(Notification $notification): mixed
        {
            return '#support-channel';
        }
    }

<a name="notifying-external-slack-workspaces"></a>
### 外部のSlackワークスペースへの通知

> [!NOTE]
> 外部Slackワークスペースへ通知を送信する前に、Slackアプリを[配布](#slack-app-distribution)する必要があります。

もちろん、アプリケーションのユーザーが所有するSlackワークスペースへ、通知を送りたいことも多いでしょう。そのためには、まずユーザーのSlack OAuthトークンを取得する必要があります。嬉しいことに、[Laravel Socialite](/docs/{{version}}/socialite)にはSlackドライバが含まれており、アプリケーションのユーザーをSlackで簡単に認証し、[ボットトークンを取得](/docs/{{version}}/socialite#slack-bot-scopes)できます。

ボットトークンを取得し、アプリケーションのデータベースへ保存したら、`SlackRoute::make`メソッドを使用して、ユーザーのワークスペースへ通知をルーティングできます。さらに、あなたのアプリケーションでは、通知をどのチャンネルに送るかをユーザーが指定できるようにする必要があるでしょう：

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Notifications\Notification;
    use Illuminate\Notifications\Slack\SlackRoute;

    class User extends Authenticatable
    {
     * @return \Illuminate\Notifications\Message\SlackMessage

        /**
         * Slackチャンネルへの通知ルート
         */
        public function routeNotificationForSlack(Notification $notification): mixed
        {
            return SlackRoute::make($this->slack_channel, $this->slack_token);
        }
    }

<a name="localizing-notifications"></a>
## 通知のローカライズ

Laravelを使用すると、HTTPリクエストの現在のロケール以外のロケールで通知を送信でき、通知をキュー投入する場合でもこのロケールを記憶しています。

このために、`Illuminate\Notifications\Notification`クラスは目的の言語を指定するための`locale`メソッドを提供しています。通知が評価されると、アプリケーションはこのロケールに変更され、評価が完了すると前のロケールに戻ります。

    $user->notify((new InvoicePaid($invoice))->locale('es'));

通知可能な複数のエンティティをローカライズするのも、`Notification`ファサードにより可能です。

    Notification::locale('es')->send(
        $users, new InvoicePaid($invoice)
    );

<a name="user-preferred-locales"></a>
### ユーザー希望のローケル

ユーザーの希望するローケルをアプリケーションで保存しておくことは良くあります。notifiableモデルで`HasLocalePreference`契約を実装すると、通知送信時にこの保存してあるローケルを使用するように、Laravelへ指示できます。

    use Illuminate\Contracts\Translation\HasLocalePreference;

    class User extends Model implements HasLocalePreference
    {
        /**
         * ユーザーの希望するローケルの取得
         */
        public function preferredLocale(): string
        {
            return $this->locale;
        }
    }

このインターフェイスを実装すると、そのモデルに対しmailableや通知を送信する時に、Laravelは自動的に好みのローケルを使用します。そのため、このインターフェイスを使用する場合、`locale`メソッドを呼び出す必要はありません。

    $user->notify(new InvoicePaid($invoice));

<a name="testing"></a>
## テスト

`Notification`ファサードの`fake`メソッドを使用すれば、通知を実際に送信しなくてすみます。通常、通知の送信は、実際にテストしているコードとは無関係です。ほとんどの場合、Laravelが指定された通知を送信するように指示されたことを単純にアサートすれば十分です。

`Notification`ファサードの`fake`メソッドを呼び出したあとに、ユーザーへ通知を送る指示したことをアサートし、その通知が受け取ったデータを調べることもできます。

```php tab=Pest
<?php

use App\Notifications\OrderShipped;
use Illuminate\Support\Facades\Notification;

test('orders can be shipped', function () {
    Notification::fake();

    // 注文発送の実行…

    // 通知されないことをアサート
    Notification::assertNothingSent();

    // 一つの通知が送信されることをアサート
    Notification::assertSentTo(
        [$user], OrderShipped::class
    );

    // 一つの通知が送信されないことをアサート
    Notification::assertNotSentTo(
        [$user], AnotherNotification::class
    );

    // 指定した数の通知が送信されることをアサート
    Notification::assertCount(3);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Notifications\OrderShipped;
use Illuminate\Support\Facades\Notification;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_orders_can_be_shipped(): void
    {
        Notification::fake();

        // Perform order shipping...

        // 通知されないことをアサート
        Notification::assertNothingSent();

        // 一つの通知が送信されることをアサート
        Notification::assertSentTo(
            [$user], OrderShipped::class
        );

        // 一つの通知が送信されないことをアサート
        Notification::assertNotSentTo(
            [$user], AnotherNotification::class
        );

        // 指定した数の通知が送信されることをアサート
        Notification::assertCount(3);
    }
}
```

指定「真偽値テスト」にパスした通知が送信されたことをアサートするため、`assertSentTo`または`assertNotSentTo`メソッドへクロージャを渡せます。指定真偽値テストにパスする通知が最低１つ送信された場合、そのアサートはパスします。

    Notification::assertSentTo(
        $user,
        function (OrderShipped $notification, array $channels) use ($order) {
            return $notification->order->id === $order->id;
        }
    );

<a name="on-demand-notifications"></a>
#### オンデマンド通知

テストするコードが、[オンデマンド通知](#on-demand-notifications)を送信する場合、`assertSentOnDemand`メソッドでオンデマンド通知を送信したことをテストできます。

    Notification::assertSentOnDemand(OrderShipped::class);

`assertSentOnDemand`メソッドの第２引数にクロージャを渡すことで、オンデマンド通知が正しい「ルート」アドレスに送信されたかを判断できます。

    Notification::assertSentOnDemand(
        OrderShipped::class,
        function (OrderShipped $notification, array $channels, object $notifiable) use ($user) {
            return $notifiable->routes['mail'] === $user->email;
        }
    );

<a name="notification-events"></a>
## 通知イベント

<a name="notification-sending-event"></a>
#### 通知送信前イベント

通知を送信すると、通知システムが`Illuminate\Notifications\Events\NotificationSending`イベントをディスパッチします。これは"Notifiable"エンティティと、通知インスタンス自身を含んでいます。アプリケーション内へ、このイベントの[イベントリスナ](/docs/{{version}}/events)を作成できます。

    use Illuminate\Notifications\Events\NotificationSending;

    class CheckNotificationStatus
    {
        /**
         * 指定イベントの処理
         */
        public function handle(NotificationSending $event): void
        {
            // ...
        }
    }

`NotificationSending`イベントのイベントリスナが、その`handle`メソッドから`false`を返した場合、通知は送信されません。

    /**
     * 指定イベントの処理
     */
    public function handle(NotificationSending $event): bool
    {
        return false;
    }

イベントリスナの中では、イベントの`notifiable`、`notification`、`channel`プロパティへアクセスし、通知先や通知自体の詳細を調べられます。

    /**
     * 指定イベントの処理
     */
    public function handle(NotificationSending $event): void
    {
        // $event->channel
        // $event->notifiable
        // $event->notification
    }

<a name="notification-sent-event"></a>
#### 通知送信後イベント

通知を送信すると、通知システムが`Illuminate\Notifications\Events\NotificationSent`[イベント](/docs/{{version}}/events)をディスパッチします。これは、"Notifiable"エンティティと通知インスタンス自身を含んでいます。アプリケーション内へ、このイベントの[イベントリスナ](/docs/{{version}}/events)を作成できます。

    use Illuminate\Notifications\Events\NotificationSent;

    class LogNotification
    {
        /**
         * 指定イベントの処理
         */
        public function handle(NotificationSent $event): void
        {
            // ...
        }
    }

イベントリスナ内では、イベントの `notifiable`、`notification`、`channel`、`response`プロパティにアクセスして、通知先や通知自体の詳細を知ることができます。

    /**
     * 指定イベントの処理
     */
    public function handle(NotificationSent $event): void
    {
        // $event->channel
        // $event->notifiable
        // $event->notification
        // $event->response
    }

<a name="custom-channels"></a>
## カスタムチャンネル

Laravelには通知チャンネルがいくつか付属していますが、他のチャンネルを介して通知を配信する独自​​のドライバを作成することもできます。Laravelではこれをシンプルに実現できます。作成開始するには、`send`メソッドを含むクラスを定義します。このメソッドは、`$notifying`と`$notification`の2つの引数を受け取る必要があります。

`send`メソッド内で、通知メソッドを呼び出して、チャンネルが理解できるメッセージオブジェクトを取得し、必要に応じて通知を`$notifiable`インスタンスに送信します。

    <?php

    namespace App\Notifications;

    use Illuminate\Notifications\Notification;

    class VoiceChannel
    {
        /**
         * 指定された通知の送信
         */
        public function send(object $notifiable, Notification $notification): void
        {
            $message = $notification->toVoice($notifiable);

            // 通知を$notifiableインスタンスへ送信する…
        }
    }

通知チャンネルクラスを定義したら、任意の通知の`via`メソッドからクラス名を返せます。この例では、通知の`toVoice`メソッドは、ボイスメッセージを表すために選択するどんなオブジェクトも返せます。たとえば、次のメッセージを表すために独自の`VoiceMessage`クラスを定義できます。

    <?php

    namespace App\Notifications;

    use App\Notifications\Messages\VoiceMessage;
    use App\Notifications\VoiceChannel;
    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Notifications\Notification;

    class InvoicePaid extends Notification
    {
        use Queueable;

        /**
         * 通知チャンネルの取得
         */
        public function via(object $notifiable): string
        {
            return VoiceChannel::class;
        }

        /**
         * 通知の音声プレゼンテーションを取得
         */
        public function toVoice(object $notifiable): VoiceMessage
        {
            // ...
        }
    }
