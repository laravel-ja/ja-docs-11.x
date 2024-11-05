# MongoDB

- [イントロダクション](#introduction)
- [インストール](#installation)
    - [MongoDBドライバ](#mongodb-driver)
    - [MongoDBサーバの開始](#starting-a-mongodb-server)
    - [Laravel MongoDBパッケージのインストール](#install-the-laravel-mongodb-package)
- [設定](#configuration)
- [機能](#features)

<a name="introduction"></a>
## イントロダクション

[MongoDB](https://www.mongodb.com/resources/products/fundamentals/why-use-mongodb)は、最も人気のあるNoSQLドキュメント指向データベースの1つで、高い書き込み負荷（分析やIoTに有用）と高可用性（自動フェイルオーバーでレプリカセットを簡単に設定できる）で利用されています。また、水平スケーラビリティのためにデータベースを簡単にシャードでき、集計、テキスト検索、地理空間クエリを実行するための強力なクエリ言語を持っています。

SQLデータベースのように行や列のテーブルへデータを格納するのではなく、MongoDBデータベースの各レコードは、データのバイナリ表現であるBSONで記述されたドキュメントです。アプリケーションはこの情報をJSON形式で取り出せできます。ドキュメント、配列、埋め込みドキュメント、バイナリデータなど、さまざまなデータタイプをサポートしています。

LaravelでMongoDBを使用する前に、Composer経由で`mongodb/laravel-mongodb`パッケージをインストールして使用することをお勧めします。`laravel-mongodb`パッケージはMongoDBにより公式にメンテナンスされています。MongoDBはMongoDBドライバにより、PHPでネイティブにサポートされていますが、[Laravel MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/)パッケージはEloquentや他のLaravelの機能とのより豊かな統合を提供します。

```shell
composer require mongodb/laravel-mongodb
```

<a name="installation"></a>
## インストール

<a name="mongodb-driver"></a>
### MongoDBドライバ

MongoDBデータベースに接続するには、`mongodb` PHP拡張が必要です。[Laravel Herd](https://herd.laravel.com)を使いローカルで開発している場合や、`php.new`を使いPHPをインストールしている場合は、すでにこの拡張モジュールがインストールされています。しかし、手作業で拡張をインストールする必要がある場合は、PECL経由でインストールできます。

```shell
pecl install mongodb
```

MongoDB PHP拡張モジュールのインストールの詳細は、[MongoDB PHP拡張モジュールのインストール手順](https://www.php.net/manual/ja/mongodb.installation.php) を参照ください。

<a name="starting-a-mongodb-server"></a>
### MongoDBサーバの開始

MongoDBコミュニティサーバは、ローカルでMongoDBを実行するために使用でき、Windows、macOS、Linux、またはDockerコンテナとしてインストールできます。MongoDB のインストール方法は、[公式MongoDBコミュニティインストールガイド](https://docs.mongodb.com/manual/administration/install-community/)を参照してください。

MongoDBサーバの接続文字列は、`.env`ファイルで設定します。

```ini
MONGODB_URI="mongodb://localhost:27017"
MONGODB_DATABASE="laravel_app"
```

クラウドでMongoDBをホスティングするには、[MongoDB Atlas](https://www.mongodb.com/cloud/atlas)の使用を検討してください。
アプリケーションからローカルで、MongoDB Atlasクラスタへアクセスするには、プロジェクトのIPアクセスリストに[クラスタのネットワーク設定の中で自分のIPアドレスを追加する](https://www.mongodb.com/docs/atlas/security/add-ip-address-to-list/)必要があります。

MongoDB Atlasの接続文字列は、`.env`ファイルで設定することもできます。

```ini
MONGODB_URI="mongodb+srv://<username>:<password>@<cluster>.mongodb.net/<dbname>?retryWrites=true&w=majority"
MONGODB_DATABASE="laravel_app"
```

<a name="install-the-laravel-mongodb-package"></a>
### Laravel MongoDBパッケージのインストール

最後に、Composerを使ってLaravel MongoDBパッケージをインストールします。

```shell
composer require mongodb/laravel-mongodb
```

> [!NOTE]
> `mongodb` PHP拡張モジュールがインストールされていないと、このパッケージのインストールは失敗します。PHPの設定はCLIとWebサーバで異なることがあるので、両方の設定で拡張モジュールが有効になっていることを確認してください。

<a name="configuration"></a>
## 設定

アプリケーションの`config/database.php`設定ファイルで、MongoDB接続を設定できます。このファイルで、`mongodb`ドライバを使う`mongodb`接続を追加します。

```php
'connections' => [
    'mongodb' => [
        'driver' => 'mongodb',
        'dsn' => env('MONGODB_URI', 'mongodb://localhost:27017'),
        'database' => env('MONGODB_DATABASE', 'laravel_app'),
    ],
],
```

<a name="features"></a>
## 機能

設定を完了したら、`mongodb`パッケージとデータベース接続をアプリケーションで使い、様々な強力な機能を活用することができます。

- [Eloquentを使用し](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/eloquent-models/)、モデルをMongoDBのコレクションに格納することができます。Eloquentの標準機能に加えて、Laravel MongoDBパッケージは埋め込みリレーションシップなどの追加機能を提供します。このパッケージはMongoDBドライバへの直接アクセスも提供し、素のクエリや集約パイプラインのような操作を実行するために使用できます。
 - クエリビルダを使って[複雑なクエリを書く](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/query-builder/)。
 - `mongodb`[キャッシュドライバ](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/) は、TTLインデックスなどのMongoDBの機能を使用し、期限切れのキャッシュエントリを自動的に消去するように最適化されています。
 - `mongodb`キュー・ドライバを使った、[キュー投入するジョブのディスパッチと処理](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/)。
 - [Flysystem用GridFSアダプタ](https://flysystem.thephpleague.com/docs/adapter/gridfs/)を介して、[GridFSにファイルを保存する](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/gridfs/)。
 - データベース接続やEloquentを使うサードパーティ製パッケージのほとんどは、MongoDBと一緒に使うことができます。

MongoDBとLaravelの使い方を引き続き学ぶには、MongoDBの[クイックスタートガイド](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/quick-start/)を参照してください。
