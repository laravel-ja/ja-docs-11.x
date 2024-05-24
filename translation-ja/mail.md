# メール

- [イントロダクション](#introduction)
    - [設定](#configuration)
    - [ドライバ事前設定](#driver-prerequisites)
    - [フェイルオーバー設定](#failover-configuration)
    - [Round Robin Configuration](#round-robin-configuration)
- [Mailableの生成](#generating-mailables)
- [Mailableの記述](#writing-mailables)
    - [Senderの設定](#configuring-the-sender)
    - [ビューの設定](#configuring-the-view)
    - [ビューデータ](#view-data)
    - [添付](#attachments)
    - [インライン添付](#inline-attachments)
    - [Attachableオブジェクト](#attachable-objects)
    - [ヘッダ](#headers)
    - [タグとメタデータ](#tags-and-metadata)
    - [Symfonyメッセージのカスタマイズ](#customizing-the-symfony-message)
- [Markdown Mailable](#markdown-mailables)
    - [Markdown Mailableの生成](#generating-markdown-mailables)
    - [Markdownメッセージの記述](#writing-markdown-messages)
    - [コンポーネントのカスタマイズ](#customizing-the-components)
- [メール送信](#sending-mail)
    - [メールのキュー投入](#queueing-mail)
- [Mailableのレンダ](#rendering-mailables)
    - [ブラウザによるMailableのプレビュー](#previewing-mailables-in-the-browser)
- [Mailableの多言語化](#localizing-mailables)
- [テスト](#testing-mailables)
    - [Mailable内容のテスト](#testing-mailable-content)
    - [Mailable送信のテスト](#testing-mailable-sending)
- [メールとローカル開発](#mail-and-local-development)
- [イベント](#events)
- [カスタムトランスポート](#custom-transports)
    - [Symfonyトランスポートの追加](#additional-symfony-transports)

<a name="introduction"></a>
## イントロダクション

メール送信は複雑であってはいけません。Laravelは、人気のある[Symfony Mailer](https://symfony.com/doc/7.0/mailer.html)コンポーネントを利用した、クリーンでシンプルなメールAPIを提供します。LaravelとSymfony Mailerは、SMTP、Mailgun、Postmark、Resend、Amazon SES、および`sendmail`経由でメールを送信するためのドライバを提供し、ローカルまたはクラウドベースのお好みのサービスを通して、メールの送信をすぐに始められます。

<a name="configuration"></a>
### 設定

Laravelのメールサービスは、アプリケーションの`config/mail.php`設定ファイルを介して設定できます。このファイル内で設定された各メーラーには、独自の設定と独自の「トランスポート」があり、アプリケーションがさまざまな電子メールサービスを使用して特定の電子メールメッセージを送信できるようにします。たとえば、アプリケーションでPostmarkを使用してトランザクションメールを送信し、AmazonSESを使用して一括メールを送信するなどです。

`mail`設定ファイル内に、`mailers`設定配列があります。この配列には、Laravelがサポートしている主要なメールドライバ／トランスポートごとのサンプル設定エントリが含まれています。その中で`default`設定値は、アプリケーションが電子メールメッセージを送信する必要があるときにデフォルトで使用するメーラーを決定します。

<a name="driver-prerequisites"></a>
### ドライバ／トランスポートの前提条件

Mailgun、Postmark、Resend、MailerSendなどのAPIベースドライバは、SMTPサーバを経由してメールを送信するよりもシンプルで高速です。可能であれば、こうしたドライバのいずれかを使用することをお勧めします。

<a name="mailgun-driver"></a>
#### Mailgunドライバ

Mailgunドライバを使用する場合は、Composerで、SymfonyのMailgun Mailerトランスポートをインストールします。

```shell
composer require symfony/mailgun-mailer symfony/http-client
```

次に、アプリケーションの`config/mail.php`設定ファイルの`default`オプションを`mailgun`へ設定し、次の設定配列を`mailers`設定配列へ追加します：

    'mailgun' => [
        'transport' => 'mailgun',
        // 'client' => [
        //     'timeout' => 5,
        // ],
    ],

アプリケーションのデフォルトメーラーを設定したら、`config/services.php`設定ファイルへ以下のオプションを追加します。

    'mailgun' => [
        'domain' => env('MAILGUN_DOMAIN'),
        'secret' => env('MAILGUN_SECRET'),
        'endpoint' => env('MAILGUN_ENDPOINT', 'api.mailgun.net'),
        'scheme' => 'https',
    ],

米国の[Mailgunリージョン](https://documentation.mailgun.com/en/latest/api-intro.html#mailgun-regions)を使用していない場合は、`services`設定ファイルでリージョンのエンドポイントを定義できます。

    'mailgun' => [
        'domain' => env('MAILGUN_DOMAIN'),
        'secret' => env('MAILGUN_SECRET'),
        'endpoint' => env('MAILGUN_ENDPOINT', 'api.eu.mailgun.net'),
        'scheme' => 'https',
    ],

<a name="postmark-driver"></a>
#### Postmarkドライバ

[Postmark](https://postmarkapp.com/)ドライバを使用する場合は、Composerを使い、SymfonyのPostmark Mailerトランスポートをインストールします。

```shell
composer require symfony/postmark-mailer symfony/http-client
```

次に、アプリケーションの`config/mail.php`設定ファイルの`default`オプションを`postmark`へ設定します。アプリケーションのデフォルトメーラーを設定したら、`config/services.php`設定ファイルへ以下のオプションを確実に含めてください。

    'postmark' => [
        'token' => env('POSTMARK_TOKEN'),
    ],

特定のメーラで使用する必要があるPostmarkメッセージストリームを指定したい場合は、`message_stream_id`設定オプションをメーラの設定配列に追加してください。この設定配列は、アプリケーションの`config/mail.php`設定ファイルにあります。

    'postmark' => [
        'transport' => 'postmark',
        'message_stream_id' => env('POSTMARK_MESSAGE_STREAM_ID'),
        // 'client' => [
        //     'timeout' => 5,
        // ],
    ],

この方法で、メッセージストリームが異なる複数のPostmarkメーラを設定することもできます。

<a name="resend-driver"></a>
#### Resendドライバ

[Resend](https://resend.com/)ドライバを使用する場合は、ComposerでResendのPHP SDKをインストールしてください。

```shell
composer require resend/resend-php
```

次に、アプリケーションの`config/mail.php`設定ファイルの`default`オプションを`resend`に設定します。アプリケーションのデフォルトメーラを設定したら、`config/services.php`設定ファイルに以下のオプションを確実に含めてください。

    'resend' => [
        'key' => env('RESEND_KEY'),
    ],

<a name="ses-driver"></a>
#### SESドライバ

Amazon SESドライバを使用するには、最初にAmazon AWS SDK for PHPをインストールする必要があります。このライブラリは、Composerパッケージマネージャを使用し、インストールできます。

```shell
composer require aws/aws-sdk-php
```

次に、`config/mail.php`設定ファイルの`default`オプションを`ses`に設定し、`config/services.php`設定ファイルに以下のオプションがあることを確認してください。

    'ses' => [
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    ],

AWSの[一時的な認証情報](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html)をセッショントークン経由で利用するには、アプリケーションのSES設定へ`token`キーを追加します。

    'ses' => [
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
        'token' => env('AWS_SESSION_TOKEN'),
    ],

SESの[サブスクリプション管理機能](https://docs.aws.amazon.com/ses/latest/dg/sending-email-subscription-management.html)を操作するため、メールメッセージの[`headers`](#headers)メソッドが返す配列の中で、`X-Ses-List-Management-Options`ヘッダを返してください。

```php
/**
 * メッセージヘッダの取得
 */
public function headers(): Headers
{
    return new Headers(
        text: [
            'X-Ses-List-Management-Options' => 'contactListName=MyContactList;topicName=MyTopic',
        ],
    );
}
```

Laravelがメール送信時に、AWS SDKの`SendEmail`メソッドへ渡す、[追加オプション](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-sesv2-2019-09-27.html#sendemail)を定義したい場合は、`ses`設定に`options`配列を定義します。

    'ses' => [
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
        'options' => [
            'ConfigurationSetName' => 'MyConfigurationSet',
            'EmailTags' => [
                ['Name' => 'foo', 'Value' => 'bar'],
            ],
        ],
    ],

<a name="mailersend-driver"></a>
#### MailerSendドライバ

トランザクションメールとSMSサービスの[MailerSend](https://www.mailersend.com/)は、Laravel用の独自APIベースのメールドライバを保守しています。ドライバを含むパッケージは、Composerパッケージマネージャ経由でインストールできます。

```shell
composer require mailersend/laravel-driver
```

パッケージをインストールしたら、アプリケーションの`.env`ファイルへ`MAILERSEND_API_KEY`環境変数を追加します。さらに、`MAIL_MAILER`環境変数を`mailersend`と定義する必要があります。

```shell
MAIL_MAILER=mailersend
MAIL_FROM_ADDRESS=app@yourdomain.com
MAIL_FROM_NAME="App Name"

MAILERSEND_API_KEY=your-api-key
```

最後に、アプリケーションの`config/mail.php`設定ファイルの、`mailers`配列へMailerSendを追加します。

```php
'mailersend' => [
    'transport' => 'mailersend',
],
```

ホストしたテンプレートの使用方法など、MailerSendの詳細は、[MailerSendドライバのドキュメント](https://github.com/mailersend/mailersend-laravel-driver#usage)を参照してください。

<a name="failover-configuration"></a>
### フェイルオーバー設定

アプリケーションのメールを送信するように設定した外部サービスがダウンすることがあります。このような場合には、プライマリ配信ドライバがダウンした場合に使用する、1つ以上のバックアップメール配信設定を定義できると便利です。

これを実現するには、アプリケーションの`mail`設定ファイルで、`failover`トランスポートを使用するメーラーを定義する必要があります。アプリケーションの`failover`メーラー設定配列に、配送に使う設定済みのメーラーを選択する順序を指定する、`mailers`配列を含める必要があります。

    'mailers' => [
        'failover' => [
            'transport' => 'failover',
            'mailers' => [
                'postmark',
                'mailgun',
                'sendmail',
            ],
        ],

        // ...
    ],

フェイルオーバーメーラーを定義したら、アプリケーションの`mail`設定ファイル内の`default`設定キーの値に、その名前を指定して、このメーラーをアプリケーションが使用するデフォルトメーラーとして設定する必要があります。

    'default' => env('MAIL_MAILER', 'failover'),

<a name="round-robin-configuration"></a>
### ラウンドロビン設定

`roundrobin`トランスポートで、メール送信の作業負荷を複数のメーラーへ分散できます。まず、アプリケーションの`mail`設定ファイル内で、`roundrobin`トランスポートを使用するメーラーを定義します。アプリケーションの`roundrobin`メーラーの設定配列では、どの設定済みメーラーを送信に使用するかを指定する`mailers`配列を含める必要があります。

    'mailers' => [
        'roundrobin' => [
            'transport' => 'roundrobin',
            'mailers' => [
                'ses',
                'postmark',
            ],
        ],

        // ...
    ],

ラウンドロビンメーラーを定義したら、アプリケーションの`mail`設定ファイル内の`default`設定キーの値に、その名前を指定し、このメーラーをアプリケーションで使用するデフォルトメーラーとして設定する必要があります。

    'default' => env('MAIL_MAILER', 'roundrobin'),

ラウンドロビントランスポートは、設定済みのメーラーのリストの中からランダムにメーラーを選択し、その後のメールに対し、次に利用可能なメーラーに切り替えます。*[高可用性](https://en.wikipedia.org/wiki/High_availability)*を達成するのに役立つ、`failover`トランスポートとは対照的に、`roundrobin`トランスポートは、*[負荷分散](https://en.wikipedia.org/wiki/Load_balancing_(computing))*を提供します。

<a name="generating-mailables"></a>
## Mailableの生成

Laravelアプリケーションを構築する場合、アプリケーションが送信する各タイプの電子メールは"Mailable"クラスとして表します。これらのクラスは`app/Mail`ディレクトリに保存されます。アプリケーションにこのディレクトリが存在しなくても心配ありません。`make:mail`　Artisanコマンドを使用して最初のメール可能なクラスを作成するときに、生成されます。

```shell
php artisan make:mail OrderShipped
```

<a name="writing-mailables"></a>
## Mailableの記述

Mailableクラスを生成したら、その中身を調べるために開いてみましょう。Mailableクラスの設定は、`envelope`、`content`、`attachments`などのメソッドで行います。

`envelope`メソッドは、メッセージのサブジェクトと、時折り受信者を定義する、`Illuminate\Mail\Mailables\Envelope`オブジェクトを返します。`content`メソッドは、メッセージの内容を生成するために使用する[Bladeテンプレート](/docs/{{version}}/blade)を定義する、`Illuminate\Mail\Mailables\Content`オブジェクトを返します。

<a name="configuring-the-sender"></a>
### Senderの設定

<a name="using-the-envelope"></a>
#### Envelopeの使用

まず、メール送信者の設定を調べてみましょう。つまり、「誰から送られた」メールかということです。送信者の設定は、２つの方法があります。まず、メッセージのEnvelope（封筒）に"from"アドレスを指定する方法です。

    use Illuminate\Mail\Mailables\Address;
    use Illuminate\Mail\Mailables\Envelope;

    /**
     * メッセージEnvelopeを取得
     */
    public function envelope(): Envelope
    {
        return new Envelope(
            from: new Address('jeffrey@example.com', 'Jeffrey Way'),
            subject: 'Order Shipped',
        );
    }

お望みであれば、`replyTo`アドレスも指定できます。

    return new Envelope(
        from: new Address('jeffrey@example.com', 'Jeffrey Way'),
        replyTo: [
            new Address('taylor@example.com', 'Taylor Otwell'),
        ],
        subject: 'Order Shipped',
    );

<a name="using-a-global-from-address"></a>
#### グローバル`from`アドレスの使用

ただし、アプリケーションがすべての電子メールに同じ「送信者」アドレスを使用している場合、生成する各Mailableクラスへ`from`を追加するのは面倒です。代わりに、`config/mail.php`設定ファイルでグローバルな「送信者」アドレスを指定できます。このアドレスは、Mailableクラス内で「送信者」アドレスを指定しない場合に使用します。

    'from' => [
        'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
        'name' => env('MAIL_FROM_NAME', 'Example'),
    ],

また、`config/mail.php`設定ファイル内でグローバルな"reply_to"アドレスも定義できます。

    'reply_to' => ['address' => 'example@example.com', 'name' => 'App Name'],

<a name="configuring-the-view"></a>
### ビューの設定

Mailableクラスの`content`メソッド内で`view`、つまりメールのコンテンツをレンダリングするときどのテンプレートを使用するかを定義します。各メールは通常、[Bladeテンプレート](/docs/{{version}}/blade)を使用してコンテンツをレンダするので、メールのHTML構築にBladeテンプレート・エンジンのパワーと利便性をフル活用できます。

    /**
     * メッセージ内容の定義を取得
     */
    public function content(): Content
    {
        return new Content(
            view: 'mail.orders.shipped',
        );
    }

> [!NOTE]
> すべてのメールテンプレートを格納するために`resources/views/emails`ディレクトリを作成することを推奨します。ただし、`resources/views`ディレクトリ内ならば好きな場所へ自由に配置できます。

<a name="plain-text-emails"></a>
#### 平文テキストの電子メール

平文テキスト版のメールを定義したい場合は、メッセージの`Content`定義を作成するときに、平文テキストのテンプレートを指定してください。`view`パラメータと同様、`text`パラメータにはメールの内容をレンダするために使用するテンプレートの名前を指定します。HTMLバージョンと平文テキストバージョンの両方を自由に定義できます。

    /**
     * メッセージ内容の定義を取得
     */
    public function content(): Content
    {
        return new Content(
            view: 'mail.orders.shipped',
            text: 'mail.orders.shipped-text'
        );
    }

明確にするために、`html`パラメータを`view`パラメータの別名として使用できます。

    return new Content(
        html: 'mail.orders.shipped',
        text: 'mail.orders.shipped-text'
    );

<a name="view-data"></a>
### ビューデータ

<a name="via-public-properties"></a>
#### Publicなプロパティ経由

通常、電子メールのHTMLをレンダするときに使用するデータをビューへ渡す必要があります。ビューでデータを利用できるようにする方法は２つあります。まず、Mailableクラスで定義したパブリックプロパティは、自動的にビューで使用できるようになります。したがって、たとえばMailableクラスのコンストラクタにデータを渡し、そのデータをクラスで定義したパブリックプロパティに設定できます。

    <?php

    namespace App\Mail;

    use App\Models\Order;
    use Illuminate\Bus\Queueable;
    use Illuminate\Mail\Mailable;
    use Illuminate\Mail\Mailables\Content;
    use Illuminate\Queue\SerializesModels;

    class OrderShipped extends Mailable
    {
        use Queueable, SerializesModels;

        /**
         * 新しいメッセージインスタンスの生成
         */
        public function __construct(
            public Order $order,
        ) {}

        /**
         * メッセージ内容の定義を取得
         */
        public function content(): Content
        {
            return new Content(
                view: 'mail.orders.shipped',
            );
        }
    }

データをパブリックプロパティへ設定すると、ビューで自動的に利用できるようになるため、Bladeテンプレートの他のデータにアクセスするのと同じようにアクセスできます。

    <div>
        Price: {{ $order->price }}
    </div>

<a name="via-the-with-parameter"></a>
#### `with`パラメータ経由

もし、テンプレートへ送る前にメールのデータフォーマットをカスタマイズしたい場合は、`Content`定義の`with`パラメータを使用して、手作業でデータをビューに渡すこともできます。しかし、このデータを`protected`または`private`プロパティにセットすることで、データが自動的にテンプレートで利用されないようにする必要があります。

    <?php

    namespace App\Mail;

    use App\Models\Order;
    use Illuminate\Bus\Queueable;
    use Illuminate\Mail\Mailable;
    use Illuminate\Mail\Mailables\Content;
    use Illuminate\Queue\SerializesModels;

    class OrderShipped extends Mailable
    {
        use Queueable, SerializesModels;

        /**
         * 新しいメッセージインスタンスの生成
         */
        public function __construct(
            protected Order $order,
        ) {}

        /**
         * メッセージ内容の定義を取得
         */
        public function content(): Content
        {
            return new Content(
                view: 'mail.orders.shipped',
                with: [
                    'orderName' => $this->order->name,
                    'orderPrice' => $this->order->price,
                ],
            );
        }
    }

データが`with`メソッドに渡されると、ビューで自動的に利用できるようになるため、Bladeテンプレートの他のデータにアクセスするのと同じようにアクセスできます。

    <div>
        Price: {{ $orderPrice }}
    </div>

<a name="attachments"></a>
### 添付

メールに添付ファイルを追加するには、メッセージの`attachments`メソッドから返す配列に添付ファイルを追加します。最初に、`Attachment`クラスが提供する、`fromPath`メソッドでファイルパスを指定して、添付ファイルを追加します。

    use Illuminate\Mail\Mailables\Attachment;

    /**
     * メッセージの添付を取得
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromPath('/path/to/file'),
        ];
    }

メッセージへファイルを添付するときに、`as`と`withMime`メソッドを使い、添付ファイルの表示名とMIMEタイプを指定することもできます。

    /**
     * メッセージの添付を取得
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromPath('/path/to/file')
                    ->as('name.pdf')
                    ->withMime('application/pdf'),
        ];
    }

<a name="attaching-files-from-disk"></a>
#### ディスクからファイルを添付

[ファイルシステムディスク](/docs/{{version}}/filesystem)のいずれかにファイルを保存している場合、`fromStorage`添付メソッドを使用してメールへ添付できます。

    /**
     * メッセージの添付を取得
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromStorage('/path/to/file'),
        ];
    }

もちろん、添付ファイル名とMIMEタイプも指定できます。

    /**
     * メッセージの添付を取得
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromStorage('/path/to/file')
                    ->as('name.pdf')
                    ->withMime('application/pdf'),
        ];
    }

デフォルトディスク以外のストレージディスクを指定する必要がある場合は、`fromStorageDisk`メソッドを使用してください。

    /**
     * メッセージの添付を取得
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromStorageDisk('s3', '/path/to/file')
                    ->as('name.pdf')
                    ->withMime('application/pdf'),
        ];
    }

<a name="raw-data-attachments"></a>
#### 素のデータの添付ファイル

`fromData`添付メソッドを使用すると、生のバイト列を添付ファイルにできます。例えば、メモリ上でPDFを生成し、それをディスクへ一旦書き込まずにメールへ添付したい場合は、このメソッドを使用します。`fromData`メソッドは、添付ファイルに割り当てるべき名前と同時に、生のデータバイトを解決するクロージャを受け取ります。

    /**
     * メッセージの添付を取得
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromData(fn () => $this->pdf, 'Report.pdf')
                    ->withMime('application/pdf'),
        ];
    }

<a name="inline-attachments"></a>
### インライン添付

インライン画像をメールに埋め込むのは、通常面倒です。ただし、Laravelはメールに画像を添付する便利な方法を提供しています。インライン画像を埋め込むには、メールテンプレート内の`$message`変数で`embed`メソッドを使用します。Laravelは自動的に`$message`変数をすべてのメールテンプレートで利用できるようにするので、手作業で渡すことを心配する必要はありません。

```blade
<body>
    Here is an image:

    <img src="{{ $message->embed($pathToImage) }}">
</body>
```

> [!WARNING]
> 平文ンテキストメッセージはインライン添付ファイルを利用しないため、`$message`変数は平文テキストメッセージテンプレートでは使用できません。

<a name="embedding-raw-data-attachments"></a>
#### 素のデータの添付ファイルへの埋め込み

電子メールテンプレートに埋め込みたい素の画像データ文字列がすでにある場合は、`$message`変数で`embedData`メソッドを呼び出すことができます。`embedData`メソッドを呼び出すときは、埋め込み画像に割り当てる必要のあるファイル名を指定する必要があります。

```blade
<body>
    Here is an image from raw data:

    <img src="{{ $message->embedData($data, 'example-image.jpg') }}">
</body>
```

<a name="attachable-objects"></a>
### Attachableオブジェクト

単純な文字列のパスを介してメッセージへファイルを添付すれば十分なことがある一方で、多くの場合、アプリケーション内の添付可能(Attachable)なエンティティはクラスによって表されます。例えば、アプリケーションがメッセージに写真を添付している場合、アプリケーションはその写真を表す`Photo`モデルを用意することもできます。その場合、`Photo`モデルを`attach`メソッドに渡せれば、便利ですよね？添付可能なAttachableオブジェクトを使用すれば、それが実行できます。

この機能を利用するには、メッセージへ添付できるオブジェクトに、`Illuminate\Contracts\Mail\Attachable`インターフェイスを実装します。このインターフェイスは、そのクラスが`Illuminate\Mail\Attachment`インスタンスを返す`toMailAttachment`メソッドを定義するよう指示します:

    <?php

    namespace App\Models;

    use Illuminate\Contracts\Mail\Attachable;
    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Mail\Attachment;

    class Photo extends Model implements Attachable
    {
        /**
         * モデルの添付可能な形式を取得
         */
        public function toMailAttachment(): Attachment
        {
            return Attachment::fromPath('/path/to/file');
        }
    }

 添付可能なオブジェクトを定義したら、メールメッセージを作成する際に`attachments`メソッドにより、そのオブジェクトのインスタンスを返してください。

    /**
     * メッセージの添付を取得
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [$this->photo];
    }

もちろん、添付ファイルデータは、Amazon S3などのリモートファイルストレージサービスに保存されている場合もあるでしょう。そのため、Laravelでは、アプリケーションの[ファイルシステム・ディスク](/docs/{{version}}/filesystem)のいずれかに保存しているデータから、添付ファイルのインスタンスを生成することも可能です。

    // デフォルトデスクから、添付ファイルを作成する
    return Attachment::fromStorage($this->path);

    // 特定のディスクから、添付ファイルを作成する
    return Attachment::fromStorageDisk('backblaze', $this->path);

さらに、メモリ上にあるデータを介して添付ファイルインスタンスを作成することもできます。これを行うには、`fromData`メソッドへクロージャを指定します。クロージャは、添付ファイルを表す生データを返す必要があります。

    return Attachment::fromData(fn () => $this->content, 'Photo Name');

Laravelは、添付ファイルをカスタマイズするために使用できる追加メソッドも提供しています。例えば、`as`や`withMime`メソッドを使用して、ファイル名やMIMEタイプをカスタマイズできます。

    return Attachment::fromPath('/path/to/file')
            ->as('Photo Name')
            ->withMime('image/jpeg');

<a name="headers"></a>
### ヘッダ

時には、送信するメッセージへ追加のヘッダを付ける必要が起きるかもしれません。例えば、カスタム`Message-Id`や、その他の任意のテキストヘッダを設定する必要があるかもしれません。

これを行うには、Mailableで`headers`メソッドを定義します。`headers`メソッドは、`Illuminate\Mail\Mailables\Headers`インスタンスを返す必要があります。このクラスは `messageId`、`references`、`text`を引数に取ります。もちろん、特定のメッセージに必要なパラメータだけを渡すこともできます。

    use Illuminate\Mail\Mailables\Headers;

    /**
     * メッセージヘッダの取得
     */
    public function headers(): Headers
    {
        return new Headers(
            messageId: 'custom-message-id@example.com',
            references: ['previous-message@example.com'],
            text: [
                'X-Custom-Header' => 'Custom Value',
            ],
        );
    }

<a name="tags-and-metadata"></a>
### タグとメタデータ

MailgunやPostmarkなどのサードパーティのメールプロバイダは、メッセージの「タグ」や「メタデータ」をサポートしており、アプリケーションが送信したメールをグループ化し、追跡しするために使用できます。タグやメタデータは、`Envelope`定義により、メールメッセージへ追加します。

    use Illuminate\Mail\Mailables\Envelope;

    /**
     * メッセージEnvelopeを取得
     *
     * @return \Illuminate\Mail\Mailables\Envelope
     */
    public function envelope(): Envelope
    {
        return new Envelope(
            subject: 'Order Shipped',
            tags: ['shipment'],
            metadata: [
                'order_id' => $this->order->id,
            ],
        );
    }

アプリケーションでMailgunドライバを使用している場合、[タグ](https://documentation.mailgun.com/en/latest/user_manual.html#tagging-1)と[メタデータ](https://documentation.mailgun.com/en/latest/user_manual.html#attaching-data-to-messages)の詳細は、Mailgunのドキュメントを参照してください。同様に、Postmarkのドキュメントも、[タグ](https://postmarkapp.com/blog/tags-support-for-smtp)と[メタデータ](https://postmarkapp.com/support/article/1125-custom-metadata-faq)のサポートについて、詳しい情報を得るために参照できます。

アプリケーションがAmazon SESを使用してメールを送信している場合、`metadata`メソッドを使用して、メッセージへ[SES 「タグ」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html)を添付する必要があります。

<a name="customizing-the-symfony-message"></a>
### Symfonyメッセージのカスタマイズ

Laravelのメール機能は、Symfony Mailerによって提供されています。Laravelでは、メッセージを送信する前に、Symfonyのメッセージインスタンスで呼び出されるカスタムコールバックを登録できます。これにより、メッセージ送信前に、そのメッセージを深くカスタマイズするチャンスが得られます。これを利用するには、`Envelope`定義で`using`パラメータを定義します。

    use Illuminate\Mail\Mailables\Envelope;
    use Symfony\Component\Mime\Email;

    /**
     * メッセージEnvelopeを取得
     */
    public function envelope(): Envelope
    {
        return new Envelope(
            subject: 'Order Shipped',
            using: [
                function (Email $message) {
                    // ...
                },
            ]
        );
    }

<a name="markdown-mailables"></a>
## Markdown Mailable

Markdown Mailableメッセージを使用すると、Mailableで[メール通知](/docs/{{version}}/notifications#mail-notifications)の事前に作成されたテンプレートとコンポーネントを利用できます。メッセージはMarkdownで記述されているため、Laravelはメッセージの美しくレスポンシブなHTMLテンプレートをレンダすると同時に、平文テキスト版も自動的に生成できます。

<a name="generating-markdown-mailables"></a>
### Markdown Mailableの生成

対応するMarkdownテンプレートを使用してMailableファイルを生成するには、`make:mail` Artisanコマンドの`--markdown`オプションを使用します。

```shell
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
```

次に、Mailableの`Content`定義をその`content`メソッド内で設定するときに、`view`パラメータの代わりに、`markdown`パラメータを使用します。

    use Illuminate\Mail\Mailables\Content;

    /**
     * メッセージ内容の定義を取得
     */
    public function content(): Content
    {
        return new Content(
            markdown: 'mail.orders.shipped',
            with: [
                'url' => $this->orderUrl,
            ],
        );
    }

<a name="writing-markdown-messages"></a>
### Markdownメッセージの記述

Markdown Mailableは、BladeコンポーネントとMarkdown構文の組み合わせを使用して、Laravelに組み込まれた電子メールUIコンポーネントを活用しながら、メールメッセージを簡単に作成できるようにします。

```blade
<x-mail::message>
# 発送

注文を発送しました。

<x-mail::button :url="$url">
注文の確認
</x-mail::button>

Thanks,<br>
{{ config('app.name') }}
</x-mail::message>
```

> [!NOTE]
> Markdownメールを書くときに余分なインデントを使用しないでください。Markdown標準に従って、Markdownパーサーはインデントされたコンテンツをコードブロックとしてレンダリングします。

<a name="button-component"></a>
#### ボタンコンポーネント

ボタンコンポーネントは、中央に配置されたボタンリンクをレンダします。コンポーネントは、`url`とオプションの`color`の２つの引数を取ります。サポートしている色は、`primary`、`success`、`error`です。メッセージには、必要なだけのボタンコンポーネントを追加できます。

```blade
<x-mail::button :url="$url" color="success">
注文の確認
</x-mail::button>
```

<a name="panel-component"></a>
#### パネルコンポーネント

パネルコンポーネントは、メッセージの残りの部分とはわずかに異なる背景色を持つパネルで、指定するテキストのブロックをレンダします。これにより、特定のテキストブロックに注意を引くことができます。

```blade
<x-mail::panel>
ここはパネルの本文。
</x-mail::panel>
```

<a name="table-component"></a>
#### テーブルコンポーネント

テーブルコンポーネントを使用すると、MarkdownテーブルをHTMLテーブルに変換できます。コンポーネントは、そのコンテンツとしてMarkdownテーブルを受け入れます。テーブルの列の配置は、デフォルトのMarkdownテーブルの配置構文をサポートします。

```blade
<x-mail::table>
| Laravel  |   テーブル    |   例 |
| -------- | :-----------: | ---: |
| Col 2 is |   Centered    |  $10 |
| Col 3 is | Right-Aligned |  $20 |
</x-mail::table>
```

<a name="customizing-the-components"></a>
### コンポーネントのカスタマイズ

カスタマイズするために、すべてのMarkdownメールコンポーネントを自身のアプリケーションへエクスポートできます。コンポーネントをエクスポートするには、`vendor:publish`　Artisanコマンドを使用して`laravel-mail`アセットタグでリソース公開します。

```shell
php artisan vendor:publish --tag=laravel-mail
```

このコマンドは、Markdownメールコンポーネントを`resources/views/vendor/mail`ディレクトリへリソース公開します。`mail`ディレクトリには`html`ディレクトリと`text`ディレクトリが含まれ、それぞれに利用可能なすべてのコンポーネントのそれぞれの表現が含まれます。これらのコンポーネントは自由にカスタマイズできます。

<a name="customizing-the-css"></a>
#### CSSのカスタマイズ

コンポーネントをエクスポートした後、`resources/views/vendor/mail/html/themes`ディレクトリには`default.css`ファイルが含まれます。このファイルのCSSをカスタマイズすると、MarkdownメールメッセージのHTML表現内でスタイルが自動的にインラインCSSスタイルに変換されます。

LaravelのMarkdownコンポーネント用にまったく新しいテーマを作成したい場合は、CSSファイルを`html/themes`ディレクトリに配置できます。CSSファイルに名前を付けて保存した後、アプリケーションの`config/mail.php`設定ファイルの`theme`オプションを更新して、新しいテーマの名前と一致させます。

個々のMailableのテーマをカスタマイズするには、Mailableクラスの`$theme`プロパティを、そのMailableを送信するときに使用するテーマの名前に設定します。

<a name="sending-mail"></a>
## メール送信

メッセージを送信するには、`Mail`[ファサード](/docs/{{version}}/facades)で`to`メソッドを使用します。`to`メソッドは、電子メールアドレス、ユーザーインスタンス、またはユーザーのコレクションを受け入れます。オブジェクトまたはオブジェクトのコレクションを渡すと、メーラーは電子メールの受信者を決定するときに自動的に`email`および`name`プロパティを使用するため、これらの属性がオブジェクトで使用可能であることを確認してください。受信者を指定したら、Mailableクラスのインスタンスを`send`メソッドに渡すことができます。

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Mail\OrderShipped;
    use App\Models\Order;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Mail;

    class OrderShipmentController extends Controller
    {
        /**
         * 指定注文を発送
         */
        public function store(Request $request): RedirectResponse
        {
            $order = Order::findOrFail($request->order_id);

            // 注文の発送処理…

            Mail::to($request->user())->send(new OrderShipped($order));

            return redirect('/orders');
        }
    }

メッセージを送信するときに「宛先」の受信者を指定するだけに限定されません。それぞれのメソッドをチェーン化することで、「to」、「cc」、「bcc」の受信者を自由に設定できます。

    Mail::to($request->user())
        ->cc($moreUsers)
        ->bcc($evenMoreUsers)
        ->send(new OrderShipped($order));

<a name="looping-over-recipients"></a>
#### 受信者をループする

場合によっては、受信者/電子メールアドレスの配列を反復処理して、受信者のリストにMailableファイルを送信する必要が起きるでしょう。しかし、`to`メソッドはメールアドレスをMailable受信者のリストに追加するため、ループを繰り返すたびに、以前のすべての受信者に別のメールが送信されます。したがって、受信者ごとにMailableインスタンスを常に再作成する必要があります。

    foreach (['taylor@example.com', 'dries@example.com'] as $recipient) {
        Mail::to($recipient)->send(new OrderShipped($order));
    }

<a name="sending-mail-via-a-specific-mailer"></a>
#### 特定のメーラーを介してメールを送信

デフォルトでは、Laravelはアプリケーションの`mail`設定ファイルで`default`メーラーとして設定されたメーラーを使用してメールを送信します。しかし、`mailer`メソッドを使用して、特定のメーラー設定を使用してメッセージを送信することができます。

    Mail::mailer('postmark')
            ->to($request->user())
            ->send(new OrderShipped($order));

<a name="queueing-mail"></a>
### メールのキュー投入

<a name="queueing-a-mail-message"></a>
#### メールメッセージのキュー投入

電子メールメッセージの送信はアプリケーションのレスポンス時間に悪影響を与える可能性があるため、多くの開発者はバックグラウンド送信のために電子メールメッセージをキューに投入することを選択します。Laravelは組み込みの[統一キューAPI](/docs/{{version}}/queues)を使用してこれを簡単にしています。メールメッセージをキューに投入するには、メッセージの受信者を指定した後、`Mail`ファサードで`queue`メソッドを使用します。

    Mail::to($request->user())
        ->cc($moreUsers)
        ->bcc($evenMoreUsers)
        ->queue(new OrderShipped($order));

このメソッドは、メッセージがバックグラウンドで送信されるように、ジョブをキューへ自動的に投入処理します。この機能を使用する前に、[キューを設定](/docs/{{version}}/queues)しておく必要があります。

<a name="delayed-message-queueing"></a>
#### 遅延メッセージキュー

キューに投入した電子メールメッセージの配信を遅らせたい場合は、`later`メソッドを使用します。`later`メソッドは最初の引数にメッセージの送信時期を示す`DateTime`インスタンスを取ります。

    Mail::to($request->user())
        ->cc($moreUsers)
        ->bcc($evenMoreUsers)
        ->later(now()->addMinutes(10), new OrderShipped($order));

<a name="pushing-to-specific-queues"></a>
#### 特定のキューへの投入

`make:mail`コマンドを使用して生成したすべてのMailableクラスは`Illuminate\Bus\Queueable`トレイトを利用するため、任意のMailableクラスインスタンスで`onQueue`メソッドと`onConnection`メソッドを呼び出して、メッセージに使う接続とキュー名を指定できます。

    $message = (new OrderShipped($order))
                    ->onConnection('sqs')
                    ->onQueue('emails');

    Mail::to($request->user())
        ->cc($moreUsers)
        ->bcc($evenMoreUsers)
        ->queue($message);

<a name="queueing-by-default"></a>
#### デフォルトでのキュー投入

常にキューに入れたいMailableクラスがある場合は、クラスに`ShouldQueue`契約を実装できます。これで、メール送信時に`send`メソッドを呼び出しても、この契約を実装しているため、Mailableはキューへ投入されます。

    use Illuminate\Contracts\Queue\ShouldQueue;

    class OrderShipped extends Mailable implements ShouldQueue
    {
        // ...
    }

<a name="queued-mailables-and-database-transactions"></a>
#### Mailableのキュー投入とデータベーストランザクション

キュー投入されたMailableがデータベーストランザクション内でディスパッチされると、データベーストランザクションがコミットされる前にキューによって処理される場合があります。これが発生した場合、データベーストランザクション中にモデルまたはデータベースレコードに加えた更新は、データベースにまだ反映されていない可能性があります。さらに、トランザクション内で作成したモデルまたはデータベースレコードは、データベースに存在しない可能性があります。Mailableがこれらのモデルに依存している場合、キューに入れられたMailableを送信するジョブが処理されるときに予期しないエラーが発生する可能性があります。

キュー接続の`after_commit`設定オプションが`false`に設定されている場合でも、メールメッセージ送信時に`afterCommit`メソッドを呼び出せば、キュー投入する特定のMailableが、オープンしているすべてのデータベーストランザクションのコミット後に、ディスパッチすると示せます。

    Mail::to($request->user())->send(
        (new OrderShipped($order))->afterCommit()
    );

あるいは、Mailableのコンストラクタで、`afterCommit`メソッドを呼び出すこともできます。

    <?php

    namespace App\Mail;

    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Mail\Mailable;
    use Illuminate\Queue\SerializesModels;

    class OrderShipped extends Mailable implements ShouldQueue
    {
        use Queueable, SerializesModels;

        /**
         * 新しいメッセージインスタンスの生成
         */
        public function __construct()
        {
            $this->afterCommit();
        }
    }

> [!NOTE]
> これらの問題の回避方法の詳細は、[キュー投入したジョブとデータベーストランザクション](/docs/{{version}}/queues#jobs-and-database-transactions)に関するドキュメントを確認してください。

<a name="rendering-mailables"></a>
## Mailableのレンダ

MailableのHTMLコンテンツを送信せずにキャプチャしたい場合があります。これを行うには、Mailableの`render`メソッドを呼び出してください。このメソッドは、Mailableファイルの評価済みHTMLコンテンツを文字列として返します。

    use App\Mail\InvoicePaid;
    use App\Models\Invoice;

    $invoice = Invoice::find(1);

    return (new InvoicePaid($invoice))->render();

<a name="previewing-mailables-in-the-browser"></a>
### ブラウザによるMailableのプレビュー

Mailableのテンプレートを設計するときは、通常のBladeテンプレートのように、レンダ済みMailableをブラウザですばやくプレビューできると便利です。このためLaravelでは、ルートクロージャまたはコントローラから直接Mailableを返せます。Mailableが返されると、ブラウザでレンダされ表示されるため、実際の電子メールアドレスへ送信しなくても、デザインをすばやくプレビューできます。

    Route::get('/mailable', function () {
        $invoice = App\Models\Invoice::find(1);

        return new App\Mail\InvoicePaid($invoice);
    });

<a name="localizing-mailables"></a>
## Mailableの多言語化

Laravelを使用すると、リクエストの現在のロケール以外のロケールでMailableファイルを送信でき、メールがキュー投入される場合でもこのロケールを記憶しています。

これを実現するために、`Mail`ファサードは目的の言語を設定するための`locale`メソッドを提供します。Mailableのテンプレートが評価されると、アプリケーションはこのロケールに変更され、評価が完了すると前のロケールに戻ります。

    Mail::to($request->user())->locale('es')->send(
        new OrderShipped($order)
    );

<a name="user-preferred-locales"></a>
### ユーザー優先ロケール

場合によっては、アプリケーションは各ユーザーの優先ロケールを保存しています。１つ以上のモデルに`HasLocalePreference`コントラクトを実装することで、メールを送信するときにこの保存されたロケールを使用するようにLaravelに指示できます。

    use Illuminate\Contracts\Translation\HasLocalePreference;

    class User extends Model implements HasLocalePreference
    {
        /**
         * ユーザーの希望するロケールを取得
         */
        public function preferredLocale(): string
        {
            return $this->locale;
        }
    }

このインターフェイスを実装すると、LaravelはMailableと通知をモデルへ送信するとき、自動的に優先ロケールを使用します。したがって、このインターフェイスを使用する場合、`locale`メソッドを呼び出す必要はありません。

    Mail::to($request->user())->send(new OrderShipped($order));

<a name="testing-mailables"></a>
## Testing

<a name="testing-mailable-content"></a>
### Mailable内容のテスト

Laravelは、Mailableの構造を調べる数多くのメソッドを提供しています。さらに、期待するコンテンツがMailableに含まれているかをテストするために便利なメソッドをいくらか用意しています。これらのメソッドは以下の通りです。`assertSeeInHtml`、`assertDontSeeInHtml`、`assertSeeInOrderInHtml`、`assertSeeInText`、`assertDontSeeInText`、`assertSeeInOrderInText`、`assertHasAttachment`、`assertHasAttachedData`、`assertHasAttachmentFromStorage`、`assertHasAttachmentFromStorageDisk`

ご想像のとおり、"HTML"アサートは、MailableのHTMLバージョンに特定の文字列が含まれていることを宣言し、"text"アサートは、Mailableの平文テキストバージョンに特定の文字列が含まれていることを宣言します。

```php tab=Pest
use App\Mail\InvoicePaid;
use App\Models\User;

test('mailable content', function () {
    $user = User::factory()->create();

    $mailable = new InvoicePaid($user);

    $mailable->assertFrom('jeffrey@example.com');
    $mailable->assertTo('taylor@example.com');
    $mailable->assertHasCc('abigail@example.com');
    $mailable->assertHasBcc('victoria@example.com');
    $mailable->assertHasReplyTo('tyler@example.com');
    $mailable->assertHasSubject('Invoice Paid');
    $mailable->assertHasTag('example-tag');
    $mailable->assertHasMetadata('key', 'value');

    $mailable->assertSeeInHtml($user->email);
    $mailable->assertSeeInHtml('Invoice Paid');
    $mailable->assertSeeInOrderInHtml(['Invoice Paid', 'Thanks']);

    $mailable->assertSeeInText($user->email);
    $mailable->assertSeeInOrderInText(['Invoice Paid', 'Thanks']);

    $mailable->assertHasAttachment('/path/to/file');
    $mailable->assertHasAttachment(Attachment::fromPath('/path/to/file'));
    $mailable->assertHasAttachedData($pdfData, 'name.pdf', ['mime' => 'application/pdf']);
    $mailable->assertHasAttachmentFromStorage('/path/to/file', 'name.pdf', ['mime' => 'application/pdf']);
    $mailable->assertHasAttachmentFromStorageDisk('s3', '/path/to/file', 'name.pdf', ['mime' => 'application/pdf']);
});
```

```php tab=PHPUnit
use App\Mail\InvoicePaid;
use App\Models\User;

public function test_mailable_content(): void
{
    $user = User::factory()->create();

    $mailable = new InvoicePaid($user);

    $mailable->assertFrom('jeffrey@example.com');
    $mailable->assertTo('taylor@example.com');
    $mailable->assertHasCc('abigail@example.com');
    $mailable->assertHasBcc('victoria@example.com');
    $mailable->assertHasReplyTo('tyler@example.com');
    $mailable->assertHasSubject('Invoice Paid');
    $mailable->assertHasTag('example-tag');
    $mailable->assertHasMetadata('key', 'value');

    $mailable->assertSeeInHtml($user->email);
    $mailable->assertSeeInHtml('Invoice Paid');
    $mailable->assertSeeInOrderInHtml(['Invoice Paid', 'Thanks']);

    $mailable->assertSeeInText($user->email);
    $mailable->assertSeeInOrderInText(['Invoice Paid', 'Thanks']);

    $mailable->assertHasAttachment('/path/to/file');
    $mailable->assertHasAttachment(Attachment::fromPath('/path/to/file'));
    $mailable->assertHasAttachedData($pdfData, 'name.pdf', ['mime' => 'application/pdf']);
    $mailable->assertHasAttachmentFromStorage('/path/to/file', 'name.pdf', ['mime' => 'application/pdf']);
    $mailable->assertHasAttachmentFromStorageDisk('s3', '/path/to/file', 'name.pdf', ['mime' => 'application/pdf']);
}
```

<a name="testing-mailable-sending"></a>
### Mailable送信のテスト

指定Mailableが特定のユーザーに「送信」されたことをアサートするテストとは別に、Mailalbeの内容をテストすることをお勧めします。通常、Mailableの内容は、テストしているコードには関係ないため、LaravelがMailableを送信するように指示されたことを単純に主張するだけで十分です。

`Mail`ファサードの`fake`メソッドを使用して、メールが実際には送信されないようにできます。`Mail`ファサードの`fake`メソッドを呼び出した後、メールソフトがユーザーに送信するよう指示されたことをアサートし、受信されたMailableのデータを検査することも可能です。

```php tab=Pest
<?php

use App\Mail\OrderShipped;
use Illuminate\Support\Facades\Mail;

test('orders can be shipped', function () {
    Mail::fake();

    // 注文の発送処理

    // Mailableを送らないことをアサート
    Mail::assertNothingSent();

    // Mailableを一つ送信したことをアサート
    Mail::assertSent(OrderShipped::class);

    // Mailableを一つ２回送信したことをアサート
    Mail::assertSent(OrderShipped::class, 2);

    // あるMailableを送信しないことをアサート
    Mail::assertNotSent(AnotherMailable::class);

    // Mailableを合計３回送信することをアサート
    Mail::assertSentCount(3);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Mail\OrderShipped;
use Illuminate\Support\Facades\Mail;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_orders_can_be_shipped(): void
    {
        Mail::fake();

        // 注文の発送処理

        // Mailableを送らないことをアサート
        Mail::assertNothingSent();

        // Mailableを一つ送信したことをアサート
        Mail::assertSent(OrderShipped::class);

        // Mailableを一つ２回送信したことをアサート
        Mail::assertSent(OrderShipped::class, 2);

        // あるMailableを送信しないことをアサート
        Mail::assertNotSent(AnotherMailable::class);

        // Mailableを合計３回送信することをアサート
        Mail::assertSentCount(3);
    }
}
```

バックグラウンドで配送するためMailableをキューに投入する場合は、`assertSent`の代わりに`assertQueued`メソッドを使用してください。

    Mail::assertQueued(OrderShipped::class);
    Mail::assertNotQueued(OrderShipped::class);
    Mail::assertNothingQueued();
    Mail::assertQueuedCount(3);

クロージャを`assertSent`、`assertNotSent`、`assertQueued`、`assertNotQueued`メソッドへ渡すと、指定した「真理値テスト」にパスするMailableが送信されたことをアサートできます。指定した真理値テストにパスするMailableを少なくとも１つ送信した場合、アサーションをパスします。

    Mail::assertSent(function (OrderShipped $mail) use ($order) {
        return $mail->order->id === $order->id;
    });

`Mail`ファサードのアサートメソッドを呼び出すときに、指定するクロージャが引数に受けるMailableインスタンスは、Mailableを調べるための有用なメソッドを提供しています。

    Mail::assertSent(OrderShipped::class, function (OrderShipped $mail) use ($user) {
        return $mail->hasTo($user->email) &&
               $mail->hasCc('...') &&
               $mail->hasBcc('...') &&
               $mail->hasReplyTo('...') &&
               $mail->hasFrom('...') &&
               $mail->hasSubject('...');
    });

また、Mailableインスタンスには、Mailableに添付されたファイルを調べるための有用なメソッドを用意しています。

    use Illuminate\Mail\Mailables\Attachment;

    Mail::assertSent(OrderShipped::class, function (OrderShipped $mail) {
        return $mail->hasAttachment(
            Attachment::fromPath('/path/to/file')
                    ->as('name.pdf')
                    ->withMime('application/pdf')
        );
    });

    Mail::assertSent(OrderShipped::class, function (OrderShipped $mail) {
        return $mail->hasAttachment(
            Attachment::fromStorageDisk('s3', '/path/to/file')
        );
    });

    Mail::assertSent(OrderShipped::class, function (OrderShipped $mail) use ($pdfData) {
        return $mail->hasAttachment(
            Attachment::fromData(fn () => $pdfData, 'name.pdf')
        );
    });

メールが送信されていないことをアサートするためのメソッドが２つあることにお気づきでしょうか。`assertNotSent`と`assertNotQueued`です。時には、メールが送信されなかったこと、**または**キュー投入されなかったことをアサートしたい場合があります。これを行うには、`assertNothingOutgoing`と`assertNotOutgoing`メソッドを使用します。

    Mail::assertNothingOutgoing();

    Mail::assertNotOutgoing(function (OrderShipped $mail) use ($order) {
        return $mail->order->id === $order->id;
    });

<a name="mail-and-local-development"></a>
## メールとローカル開発

電子メールを送信するアプリケーションを開発する場合、実際の電子メールアドレスに電子メールを送信したくない場合があります。Laravelは、ローカル開発中に実際の電子メールの送信を「無効にする」ためのいくつかの方法を提供します。

<a name="log-driver"></a>
#### Logドライバ

メールを送信する代わりに、`log`メールドライバは検査のためにすべてのメールメッセージをログファイルに書き込みます。通常、このドライバはローカル開発中にのみ使用されます。環境ごとのアプリケーションの設定の詳細については、[設定ドキュメント](/docs/{{version}}/configuration#environment-configuration)を確認してください。

<a name="mailtrap"></a>
#### HELO／Mailtrap／Mailpit

もしくは、[HELO](https://usehelo.com)や[Mailtrap](https://mailtrap.io)などのサービスと`smtp`ドライバを使用して、メールメッセージを「ダミー」メールボックスに送信可能です。本当の電子メールクライアントでそれらを表示できます。このアプローチには、Mailtrapのメッセージビューアで最終的な電子メールを実際に調べられるという利点があります。

[Laravel Sail](/docs/{{version}}/sale)を使用している場合は、[Mailpit](https://github.com/axllent/mailpit)を使用してメッセージをプレビューできます。Sailの実行中は、`http://localhost:8025`でMailpitインターフェイスにアクセスできます。

<a name="using-a-global-to-address"></a>
#### グローバルな`to`アドレスの使用

最後に、`Mail`ファサードが提供する`alwaysTo`メソッドを呼び出し、グローバルな「宛先」アドレスを指定する方法です。通常、このメソッドは、アプリケーションのサービスプロバイダの`boot`メソッドから呼び出すべきでしょう。

    use Illuminate\Support\Facades\Mail;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        if ($this->app->environment('local')) {
            Mail::alwaysTo('taylor@example.com');
        }
    }

<a name="events"></a>
## イベント

Laravelはメール送信時に２つのイベントをディスパッチします。`MessageSending`イベントはメッセージを送信する前にディスパッチし、`MessageSent`イベントはメッセージを送信した後にディスパッチします。これらのイベントはメールがキューに入っている時点でなく、*送信*した時にディスパッチされることを覚えておいてください。アプリケーションの中で、これらのイベントの[イベントリスナ](/docs/{{version}}/events)を作成できます。

    use Illuminate\Mail\Events\MessageSending;
    // use Illuminate\Mail\Events\MessageSent;

    class LogMessage
    {
        /**
         * 指定イベントの処理
         */
        public function handle(MessageSending $event): void
        {
            // ...
        }
    }

<a name="custom-transports"></a>
## カスタムトランスポート

Laravelは様々なメールトランスポートを用意していますが、Laravelが予めサポートしていない他のサービスを使いメールを配信するため、独自のトランスポートを書きたい場合があり得ます。取り掛かるには、`Symfony\Component\Mailer\Transport\AbstractTransport`クラスを継承するクラスを定義します。次に、トランスポートで`doSend`と`__toString()`メソッドを実装します。

    use MailchimpTransactional\ApiClient;
    use Symfony\Component\Mailer\SentMessage;
    use Symfony\Component\Mailer\Transport\AbstractTransport;
    use Symfony\Component\Mime\Address;
    use Symfony\Component\Mime\MessageConverter;

    class MailchimpTransport extends AbstractTransport
    {
        /**
         * 新しいMailchimpトランスポートインスタンスの生成
         */
        public function __construct(
            protected ApiClient $client,
        ) {
            parent::__construct();
        }

        /**
         * {@inheritDoc}
         */
        protected function doSend(SentMessage $message): void
        {
            $email = MessageConverter::toEmail($message->getOriginalMessage());

            $this->client->messages->send(['message' => [
                'from_email' => $email->getFrom(),
                'to' => collect($email->getTo())->map(function (Address $email) {
                    return ['email' => $email->getAddress(), 'type' => 'to'];
                })->all(),
                'subject' => $email->getSubject(),
                'text' => $email->getTextBody(),
            ]]);
        }

        /**
         * トランスポートの文字列表現の取得
         */
        public function __toString(): string
        {
            return 'mailchimp';
        }
    }

カスタムトランスポートを定義したら、`Mail`ファサードが提供する`extend`メソッドで登録します。一般的には、アプリケーションの`AppServiceProvider`サービスプロバイダの`boot`メソッド内で行います。`extend`メソッドへ渡されるクロージャへ、`$config`引数が渡されます。この引数には、アプリケーションの`config/mail.php`設定ファイルで定義してあるメーラーの設定配列が含まれています。

    use App\Mail\MailchimpTransport;
    use Illuminate\Support\Facades\Mail;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Mail::extend('mailchimp', function (array $config = []) {
            return new MailchimpTransport(/* ... */);
        });
    }

カスタムトランスポートを定義し、登録すると、アプリケーションの`config/mail.php`設定ファイル内に、新しいトランスポートを利用するメーラー定義を作成できます。

    'mailchimp' => [
        'transport' => 'mailchimp',
        // ...
    ],

<a name="additional-symfony-transports"></a>
### Symfonyトランスポートの追加

Laravelは、MailgunやPostmarkのように、Symfonyがメンテナンスしている既存のメールトランスポートをサポートしています。しかし、Laravelを拡張して、Symfonyが保守する追加のトランスポートを追加サポートしたい場合があるでしょう。Composerを使い、必要なSymfonyメーラーをインストールし、Laravelでそのトランスポートを登録することで、これが実現できます。例として、"Brevo"（以前の"Sendinblue"） Symfonyメーラーをインストールし、登録してみましょう。

```none
composer require symfony/brevo-mailer symfony/http-client
```

Sendinblueメーラーパッケージをインストールしたら、アプリケーションの`Brevo`設定ファイルへ、Brevo API認証情報のエントリを追加します。

    'brevo' => [
        'key' => 'your-api-key',
    ],

次に、`Mail`ファサードの`extend`メソッドを使用して、Laravelへこのトランスポートを登録します。一般的に、これはサービスプロバイダの`boot`メソッド内で行う必要があります。

    use Illuminate\Support\Facades\Mail;
    use Symfony\Component\Mailer\Bridge\Brevo\Transport\BrevoTransportFactory;
    use Symfony\Component\Mailer\Transport\Dsn;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Mail::extend('brevo', function () {
            return (new BrevoTransportFactory)->create(
                new Dsn(
                    'brevo+api',
                    'default',
                    config('services.brevo.key')
                )
            );
        });
    }

トランスポートを登録したら、アプリケーションの`config/mail.php`設定ファイル内に、その新しいトランスポートを利用するメーラー定義を作成します。

    'brevo' => [
        'transport' => 'brevo',
        // ...
    ],
