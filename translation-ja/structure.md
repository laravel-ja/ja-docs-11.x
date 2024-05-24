# ディレクトリ構造

- [イントロダクション](#introduction)
- [プロジェクトディレクトリ](#the-root-directory)
    - [appディレクトリ](#the-root-app-directory)
    - [bootstrapディレクトリ](#the-bootstrap-directory)
    - [configディレクトリ](#the-config-directory)
    - [databaseディレクトリ](#the-database-directory)
    - [publicディレクトリ](#the-public-directory)
    - [resourcesディレクトリ](#the-resources-directory)
    - [routesディレクトリ](#the-routes-directory)
    - [storageディレクトリ](#the-storage-directory)
    - [testsディレクトリ](#the-tests-directory)
    - [vendorディレクトリ](#the-vendor-directory)
- [Appディレクトリ](#the-app-directory)
    - [Broadcastingディレクトリ](#the-broadcasting-directory)
    - [Consoleディレクトリ](#the-console-directory)
    - [Eventsディレクトリ](#the-events-directory)
    - [Exceptionsディレクトリ](#the-exceptions-directory)
    - [Httpディレクトリ](#the-http-directory)
    - [Jobsディレクトリ](#the-jobs-directory)
    - [Listenersディレクトリ](#the-listeners-directory)
    - [Mailディレクトリ](#the-mail-directory)
    - [Modelsディレクトリ](#the-models-directory)
    - [Notificationsディレクトリ](#the-notifications-directory)
    - [Policiesディレクトリ](#the-policies-directory)
    - [Providersディレクトリ](#the-providers-directory)
    - [Rulesディレクトリ](#the-rules-directory)

<a name="introduction"></a>
## イントロダクション

Laravelのデフォルトアプリケーション構造はアプリケーションの大小にかかわらず、素晴らしいスタートを切ってもらえることを意図しています。アプリケーションは皆さんのお好みに応じ、自由に体系立ててください。クラスがComposerによりオートローディングできるならば、Laravelはクラスをどこに配置するか強制することはまずありません。

> [!NOTE]
> Laravelは初めてですか？[Laravel Bootcamp](https://bootcamp.laravel.com)では、Laravelのフレームワークを実際に体験しながら、Laravelアプリケーションを構築することができます。

<a name="the-root-directory"></a>
## プロジェクトディレクトリ

<a name="the-root-app-directory"></a>
#### appディレクトリ

`app`ディレクトリは、アプリケーションのコアコードを配置します。このフォルダの詳細は、この後に説明します。しかし、アプリケーションのほとんど全部のクラスは、このディレクトリの中に設定されることを覚えておいてください。

<a name="the-bootstrap-directory"></a>
#### bootstrapディレクトリ

`bootstrap`ディレクトリには、フレームワークを初期起動処理する`app.php`ファイルを設置しています。このディレクトリには、ルートやサービスのキャッシュファイルなど、パフォーマンスを最適化するためのフレームワークで生成されたファイルを含む`cache`ディレクトリも含んでいます。

<a name="the-config-directory"></a>
#### configディレクトリ

`config`ディレクトリは名前が示す通り、アプリケーションの全設定ファイルを設置しています。全ファイルに目を通し、設定可能なオプションに慣れ親しんでおくのは良い考えでしょう。

<a name="the-database-directory"></a>
#### databaseディレクトリ

`database`フォルダはデータベースのマイグレーションとモデルファクトリ、初期値設定（シーディング）を配置しています。ご希望であれば、このディレクトリをSQLiteデータベースの設置場所としても利用できます。

<a name="the-public-directory"></a>
#### publicディレクトリ

`public`ディレクトリには、アプリケーションへの全リクエストの入り口となり、オートローディングを設定する`index.php`ファイルがあります。また、このディレクトリにはアセット（画像、JavaScript、CSSなど）を配置します。

<a name="the-resources-directory"></a>
#### resourcesディレクトリ

`resources`ディレクトリには、[ビュー](/docs/{{version}}/views)や、CSS、JavaScriptなどのコンパイルしていない、素のアセットを格納します。

<a name="the-routes-directory"></a>
#### routesディレクトリ

`routes`ディレクトリに、アプリケーションのすべてのルート定義が含めます。`web.php`と`console.php`の2つのルートファイルが、デフォルトでLaravelに用意してあります。

`web.php`ファイルは、Laravelが`web`ミドルウェアグループに配置するルートを定義し、セッションの状態、CSRF保護、クッキーの暗号化を提供します。もしあなたのアプリケーションがステートレスでRESTfulなAPIを提供しないのであれば、ほとんどの場合、すべてのルートは、この`web.php`ファイルで定義することでしょう。

`console.php`ファイルは、クロージャベースのコンソールコマンドをすべて定義する場所です。各クロージャはコマンドインスタンスと結合されるため、各コマンドのIOメソッドを操作する簡単なアプローチが可能です。このファイルはHTTPルートを定義しませんが、アプリケーションへのコンソールベースのエントリポイント(ルート)を定義しています。また、`console.php`ファイル中でタスクを[スケジュール](/docs/{{version}}/scheduling)することもできます。

APIルート（`api.php`）とブロードキャストチャンネル（`channels.php`）のルートファイルをオプションとして、追加インストールすることもできます。

`api.php`ファイルはステートレスを意図したルートを含んでいるので、これらのルートを通してアプリケーションに入るリクエストは[トークンを使って](/docs/{{version}}/sanctum)認証されることを意図しており、セッション状態にアクセスすることはありません。

`channels.php`ファイルは、アプリケーションがサポートするすべての[イベントブロードキャスト](/docs/{{version}}/broadcasting)チャンネルを登録できる場所です。

<a name="the-storage-directory"></a>
#### storageディレクトリ

`storage`ディレクトリには、ログ、コンパイル済みBladeテンプレート、ファイルベースのセッション、ファイルキャッシュ、およびフレームワークが作成したその他のファイルが含まれます。このディレクトリは、`app`、`framework`、`logs`ディレクトリに分離されています。`app`ディレクトリは、アプリケーションが作成したファイルを保存するために使用できます。`framework`ディレクトリは、フレームワークが作成したファイルとキャッシュを保存するために使用します。最後に、`logs`ディレクトリにはアプリケーションのログファイルを保存しています。

`storage/app/public`ディレクトリは、プロファイルアバターなど、一般にアクセス可能である必要のあるユーザー生成ファイルを保存するために使用します。このディレクトリを指すシンボリックリンクを`public/storage`に作成する必要があります。`php artisan storage:link` Artisanコマンドを使用してリンクを作成できます。

<a name="the-tests-directory"></a>
#### testsディレクトリ

`tests`ディレクトリには自動テストを格納します。[Pest](https://pestphp.com)あるいは[PHPUnit](https://phpunit.de/)のユニットテストや機能テストの例をあらかじめ用意してあります。各テストクラスの末尾には、`Test`という単語をつけなければなりません。テストを実行するには、`/vendor/bin/pest`あるいは`/vendor/bin/phpunit`コマンドを使用します。また、テスト結果をより詳細に美しく表示したい場合は、`php artisan test`Artisanコマンドを使ってテストを実行することもできます。

<a name="the-vendor-directory"></a>
#### vendorディレクトリ

`vendor`ディレクトリには、[Composer](https://getcomposer.org)による依存パッケージが配置されます。

<a name="the-app-directory"></a>
## appディレクトリ

アプリケーションの主要な部分は、`app`ディレクトリ内に配置します。このディレクトリはデフォルトで、`App`名前空間のもとに置かれており、[PSR-4オートローディング規約](https://www.php-fig.org/psr/psr-4/)を使い、Composerがオートロードしています。

`app`ディレクトリはデフォルトで、 `Http`、`Models`、`Providers`ディレクトリを含んでいます。しかし、make Artisanコマンドを使用してクラスを生成していくうちに、appディレクトリ内に他の様々なディレクトリが生成されていくでしょう。例えば、`make:command` Artisanコマンドを実行してコマンドクラスを生成するまで、`app/Console`ディレクトリは存在しません。

`Console`ディレクトリと、`Http`ディレクトリについては、以下のそれぞれのセクションで詳しく説明しますが、`Console`ディレクトリと`Http`ディレクトリはアプリケーションのコアとなるAPIを提供するものだと考えてください。HTTPプロトコルとCLIは、どちらもアプリケーションとやりとりする仕組みですが、実際にはアプリケーションのロジックを含みません。言い換えると、これらはアプリケーションへ命令をかける２つの方法です。`Console`ディレクトリにはすべてのArtisanコマンドを格納し、`Http`ディレクトリにはコントローラ、ミドルウェア、リクエストを格納します。

> [!NOTE]
> Artisanコマンドにより、`app`ディレクトリ下にたくさんのクラスが生成されます。使用可能なコマンドを確認するには、`php artisan list make`コマンドをターミナルで実行してください。

<a name="the-broadcasting-directory"></a>
#### Broadcastingディレクトリ

`Broadcasting`ディレクトリは、アプリケーションの全ブロードキャストチャンネルクラスで構成します。これらのクラスは、`make:channel`コマンドで生成されます。このディレクトリはデフォルトでは存在しませんが、最初にチャンネルを生成したときに作成されます。チャンネルについての詳細は、[イベントブロードキャスト](/docs/{{version}}/broadcasting)のドキュメントで確認してください。

<a name="the-console-directory"></a>
#### Consoleディレクトリ

`Console`ディレクトリには、アプリケーションのすべてのカスタムArtisanコマンドを格納します。これらのコマンドは、`make:command`コマンドを使用し生成できます。

<a name="the-events-directory"></a>
#### Eventsディレクトリ

このディレクトリはデフォルトで存在していません。`event:generate`か`make:event` Artisanコマンド実行時に作成されます。`Events`ディレクトリは、[イベントクラス](/docs/{{version}}/events)を設置する場所です。イベントは特定のアクションが起きたことをアプリケーションの別の部分へ知らせるために使われ、柔軟性と分離性を提供しています。

<a name="the-exceptions-directory"></a>
#### Exceptionsディレクトリ

`Exceptions`ディレクトリは、アプリケーションのカスタム例外をすべて格納します。これらの例外は、`make:exception`コマンドを使い生成できます。

<a name="the-http-directory"></a>
#### Httpディレクトリ

`Http`ディレクトリはコントローラ、ミドルウェア、フォームリクエストを設置します。アプリケーションへのリクエストを処理するロジックは、ほぼすべてこのディレクトリ内に設置します。

<a name="the-jobs-directory"></a>
#### Jobsディレクトリ

このディレクトリはデフォルトで存在していません。`make:job` Artisanコマンドを実行すると作成されます。`Jobs`ディレクトリはアプリケーションの[キュー投入可能なジョブ](/docs/{{version}}/queues)を置いておく場所です。`Jobs`はアプリケーションによりキューに投入されるか、もしくは現在のリクエストサイクル中に同期的に実行されます。現在のリクエストサイクル中に同期的に実行するジョブは、[コマンドパターン](https://en.wikipedia.org/wiki/Command_pattern)を実装しているため、時に「コマンド」と呼ばれることがあります。

<a name="the-listeners-directory"></a>
#### Listenersディレクトリ

このディレクトリはデフォルトで存在していません。`event:generate`か`make:listener` Artisanコマンドを実行すると、作成されます。`Listeners`ディレクトリには、[events](/docs/{{version}}/events)イベントを処理するクラスを設置します。イベントリスナはイベントインスタンスを受け取り、発行されたイベントへ対応するロジックを実行します。たとえば、`UserRegistered`（ユーザー登録）イベントは、`SendWelcomeEmail`（ウェルカムメール送信）リスナにより処理されることになるでしょう。

<a name="the-mail-directory"></a>
#### Mailディレクトリ

このディレクトリはデフォルトでは存在していませんが、`make:mail`　Artisanコマンドを実行すると作成されます。`Mail`ディレクトリには、アプリケーションから送信されるすべての[メールを表すクラス](/docs/{{version}}/mail)を設置します。メールオブジェクトを使用すると、メールを作成するすべてのロジックを、`Mail::send`メソッドを使用して送信できる単一の単純なクラスにカプセル化できます。

<a name="the-models-directory"></a>
#### Modelsディレクトリ

`Models`ディレクトリには、すべての[Eloquentモデルクラス](/docs/{{version}}/eloquent)を設置します。Laravelが提供するEloquent　ORMは、データベースを操作するための美しくシンプルなActiveRecordの実装を提供しています。各データベーステーブルには、そのテーブル操作に使う対応する「モデル」があります。モデルを使用し、テーブル内のデータをクエリしたり、テーブルに新しいレコードを挿入したりできます。

<a name="the-notifications-directory"></a>
#### Notificationsディレクトリ

このディレクトリはデフォルトでは存在しませんが、`make:notification` Artisanコマンドを実行すると作成されます。`Notifications`ディレクトリには、アプリケーション内で発生するイベントに関する簡単な通知など、アプリケーションが送信するすべての「トランザクション」的な[通知](/docs/{{version}}/notifications)を設置します。Laravelの通知機能は、電子メール、Slack、SMSなどのさまざまなドライバを介して通知を送信したり、データベースに保存したりすることを抽象化しています。

<a name="the-policies-directory"></a>
#### Policiesディレクトリ

このディレクトリはデフォルトでは存在しませんが、`make:policy` Artisanコマンドを実行すると作成されます。`Policies`ディレクトリには、アプリケーションの[認可ポリシークラス](/docs/{{version}}/authentication)を設置します。ポリシーは、ユーザーがリソースに対して特定のアクションを実行できるかどうかを判断するために使用されます。

<a name="the-providers-directory"></a>
#### Providersディレクトリ

`Providers`ディレクトリは、アプリケーションの全[サービスプロバイダ](/docs/{{version}}/providers)により構成します。サービスプロバイダは、サービスをコンテナと結合、イベントの登録、もしくはアプリケーションへやってくるリクエストを処理するために必要な用意をするタスクを実行するなど、アプリケーションの事前準備を行います。

新しいLaravelアプリケーションには、このディレクトリにあらかじめ`AppServiceProvider`を用意しています。必要に応じて、このディレクトリへ自身のプロバイダーを自由に追加してください。。

<a name="the-rules-directory"></a>
#### Rulesディレクトリ

このディレクトリは、デフォルトでは存在していません。`make:rule` Artisanコマンドを実行すると、作成されます。`Rules`ディレクトリは、アプリケーションが使用するバリデーションルールオブジェクトで構成します。ルールは複雑なバリデーションロジックをシンプルなオブジェクトへカプセル化するために使用します。詳細は、[バリデーションのドキュメント](/docs/{{version}}/validation)で確認してください。
