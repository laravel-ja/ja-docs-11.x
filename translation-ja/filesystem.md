# ファイルストレージ

- [イントロダクション](#introduction)
- [設定](#configuration)
    - [ローカルドライバ](#the-local-driver)
    - [公開ディスク](#the-public-disk)
    - [ドライバの動作要件](#driver-prerequisites)
    - [スコープ付きと読み取り専用ファイルシステム](#scoped-and-read-only-filesystems)
    - [Amazon S3互換ファイルシステム](#amazon-s3-compatible-filesystems)
- [ディスクインスタンスの取得](#obtaining-disk-instances)
    - [オンデマンドディスク](#on-demand-disks)
- [ファイルの取得](#retrieving-files)
    - [ファイルのダウンロード](#downloading-files)
    - [ファイルのURL](#file-urls)
    - [一時的なURL](#temporary-urls)
    - [ファイルメタデータ](#file-metadata)
- [ファイルの保存](#storing-files)
    - [ファイルの前後への追加](#prepending-appending-to-files)
    - [ファイルのコピーと移動](#copying-moving-files)
    - [自動ストリーミング](#automatic-streaming)
    - [ファイルのアップロード](#file-uploads)
    - [ファイルの可視性](#file-visibility)
- [ファイルの削除](#deleting-files)
- [ディレクトリ](#directories)
- [テスト](#testing)
- [カスタムファイルシステム](#custom-filesystems)

<a name="introduction"></a>
## イントロダクション

Laravelは、Frankde Jongeによる素晴らしい[Flysystem](https://github.com/thephpleague/flysystem) PHPパッケージのおかげで、ファイルシステムの強力な抽象化を提供しています。Laravel Flysystem統合は、ローカルファイルシステム、SFTP、およびAmazonS3を操作するためのシンプルなドライバを提供します。さらに良いことに、APIは各システムで同じままであるため、ローカル開発マシンと本番サーバの間でこれらのストレージオプションを切り替えるのは驚くほど簡単です。

<a name="configuration"></a>
## 設定

Laravelのファイルシステム設定ファイルは`config/filesystems.php`にあります。このファイル内で、すべてのファイルシステム「ディスク」を設定できます。各ディスクは、特定のストレージドライバとストレージの場所を表します。サポートしている各ドライバの設定例を設定ファイルに用意しているので、ストレージ設定と資格情報を反映するように設定を変更してください。

`local`ドライバは、Laravelアプリケーションを実行しているサーバでローカルに保存されているファイルを操作し、`s3`ドライバはAmazonのS3クラウドストレージサービスへの書き込みに使用します。

> [!NOTE]
> 必要な数のディスクを構成でき、同じドライバを使用する複数のディスクを使用することもできます。

<a name="the-local-driver"></a>
### ローカルドライバ

`local`ドライバを使用する場合、すべてのファイル操作は、`filesystems`設定ファイルで定義した`root`ディレクトリからの相対位置です。デフォルトでは、この値は`storage/app`ディレクトリに設定されています。したがって、次のメソッドは`storage/app/example.txt`に書き込みます。

    use Illuminate\Support\Facades\Storage;

    Storage::disk('local')->put('example.txt', 'Contents');

<a name="the-public-disk"></a>
### 公開ディスク

アプリケーションの`filesystems`設定ファイルに含まれている`public`ディスクは、パブリックに公開してアクセスできるようにするファイルを対象としています。デフォルトでは、`public`ディスクは`local`ドライバを使用し、そのファイルを`storage/app/public`に保存します。

これらのファイルにWebからアクセスできるようにするには、`storage/app/public`ソースディレクトリから`public/storage`ターゲットディレクトリへのシンボリックリンクを作成する必要があります。このフォルダ規約を利用すると、[Envoyer](https://envoyer.io)のようなダウンタイムゼロのデプロイメントシステムを使用する場合に、パブリックにアクセス可能なファイルを1つのディレクトリに保持し、デプロイメント間で簡単に共有できます。

シンボリックリンクを作成するには、`storage:link` Artisanコマンドを使用できます。

```shell
php artisan storage:link
```

ファイルを保存し、シンボリックリンクを作成したら、`asset`ヘルパを使用してファイルへのURLを作成できます。

    echo asset('storage/file.txt');

`filesystems`設定ファイルで追加のシンボリックリンクを設定できます。`storage:link`コマンドを実行すると、設定された各リンクが作成されます。

    'links' => [
        public_path('storage') => storage_path('app/public'),
        public_path('images') => storage_path('app/images'),
    ],

設定したシンボリックリンクを破棄するには、`storage:unlink` コマンドを使用します。

```shell
php artisan storage:unlink
```

<a name="driver-prerequisites"></a>
### ドライバの動作要件

<a name="s3-driver-configuration"></a>
#### S3ドライバ設定

S3ドライバを使用する前に、Composerパッケージマネージャを使用し、Flysystem S3パッケージをインストールする必要があります。

```shell
composer require league/flysystem-aws-s3-v3 "^3.0" --with-all-dependencies
```

S3ディスク設定配列は、`config/filesystems.php` 設定ファイルにあります。通常は、`config/filesystems.php`設定ファイルから参照されている以下の環境変数を使い、S3の情報と認証情報を設定します。

```
AWS_ACCESS_KEY_ID=<your-key-id>
AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=<your-bucket-name>
AWS_USE_PATH_STYLE_ENDPOINT=false
```

わかりやすいように、これらの環境変数はAWS CLIで使用されている命名規則と一致させています。

<a name="ftp-driver-configuration"></a>
#### FTPドライバの設定

FTPドライバを使用する前に、Composerパッケージマネージャを使用し、Flysystem FTPパッケージをインストールする必要があります。

```shell
composer require league/flysystem-ftp "^3.0"
```

LaravelのFlysystemの統合は、FTPでもうまく動作しますが、フレームワークのデフォルトの`config/filesystems.php`設定ファイルにサンプル設定は、用意していません。FTPファイルシステムを設定する必要がある場合は、以下の設定例を使用してください。

    'ftp' => [
        'driver' => 'ftp',
        'host' => env('FTP_HOST'),
        'username' => env('FTP_USERNAME'),
        'password' => env('FTP_PASSWORD'),

        // オプションのFTP設定
        // 'port' => env('FTP_PORT', 21),
        // 'root' => env('FTP_ROOT'),
        // 'passive' => true,
        // 'ssl' => true,
        // 'timeout' => 30,
    ],

<a name="sftp-driver-configuration"></a>
#### SFTPドライバの設定

SFTPドライバを使用する前に、Composerパッケージマネージャを使用して Flysystem SFTPパッケージをインストールする必要があります。

```shell
composer require league/flysystem-sftp-v3 "^3.0"
```

LaravelのFlysystemの統合は、SFTPでもうまく動作しますが、フレームワークのデフォルトの`config/filesystems.php`設定ファイルにサンプル設定は、用意していません。SFTPファイルシステムを設定する必要がある場合は、以下の設定例を使用してください：

    'sftp' => [
        'driver' => 'sftp',
        'host' => env('SFTP_HOST'),

        // 基本認証の設定
        'username' => env('SFTP_USERNAME'),
        'password' => env('SFTP_PASSWORD'),

        // 暗号化パスワードを使用するSSHキーベースの認証の設定
        'privateKey' => env('SFTP_PRIVATE_KEY'),
        'passphrase' => env('SFTP_PASSPHRASE'),

        // Settings for file / directory permissions...
        'visibility' => 'private', // `private` = 0600, `public` = 0644
        'directory_visibility' => 'private', // `private` = 0700, `public` = 0755

        // オプションのSFTP設定
        // 'hostFingerprint' => env('SFTP_HOST_FINGERPRINT'),
        // 'maxTries' => 4,
        // 'passphrase' => env('SFTP_PASSPHRASE'),
        // 'port' => env('SFTP_PORT', 22),
        // 'root' => env('SFTP_ROOT', ''),
        // 'timeout' => 30,
        // 'useAgent' => true,
    ],

<a name="scoped-and-read-only-filesystems"></a>
### スコープ付きと読み取り専用ファイルシステム

スコープ付きディスクを使用すると、すべてのパスに自動的に指定したパスプレフィックスが付くファイルシステムを定義できます。スコープ付きファイルシステムディスクを作成する前に、Composerパッケージマネージャを使用して、Flysystemパッケージを追加でインストールする必要があります。

```shell
composer require league/flysystem-path-prefixing "^3.0"
```

`scoped`ドライバを利用するディスクを定義し、既存のファイルシステムディスクをパススコープするインスタンスを作成します。例えば、既存の`s3`ディスクを特定のパスプレフィックスにスコープするディスクを作成すれば、スコープしたディスクを使用するすべてのファイル操作で、指定したプレフィックスを使用します。

```php
's3-videos' => [
    'driver' => 'scoped',
    'disk' => 's3',
    'prefix' => 'path/to/videos',
],
```

「読み取り専用(Read-only)」ディスクで、書き込み操作を許可しないファイルシステムディスクを作成できます。`read-only`設定オプションを使用する前に、Composerパッケージマネージャ経由で、Flysystemパッケージを追加インストールする必要があります。

```shell
composer require league/flysystem-read-only "^3.0"
```

次に、ディスクの設定配列に、`read-only`設定オプションを含めてください。

```php
's3-videos' => [
    'driver' => 's3',
    // ...
    'read-only' => true,
],
```

<a name="amazon-s3-compatible-filesystems"></a>
### Amazon S3互換ファイルシステム

アプリケーションの`filesystems`設定ファイルには、デフォルトで`s3`のディスク設定がしてあります。このディスクを使用して[Amazon S3](https://aws.amazon.com/s3/)を操作するだけでなく、[MinIO](https://github.com/minio/minio)、[DigitalOcean Spaces](https://www.digitalocean.com/products/spaces/)、[Akamai/Linode Object Storage](https://www.linode.com/products/object-storage/)、[Vultr Object Storage](https://www.vultr.com/products/object-storage/)、[Hetzner Cloud Storage](https://www.hetzner.com/storage/object-storage/)など、S3互換のファイルストレージサービスを操作することもできます。

通常、ディスクの認証情報を使用予定のサービス認証情報へ合わせて更新した後に、`endpoint`設定オプションの値を更新するだけで済みます。このオプションの値は通常、`AWS_ENDPOINT`環境変数で定義されています。

    'endpoint' => env('AWS_ENDPOINT', 'https://minio:9000'),

<a name="minio"></a>
#### MinIO

LaravelのFlysystemインテグレーションでMinIOを使用する際に、適切なURLを生成するには、`AWS_URL`環境変数を定義し、アプリケーションのローカルURLと一致させ、URLパスにバケット名を含める必要があります。

```ini
AWS_URL=http://localhost:9000/local
```

> [!WARNING]
> `endpoint`がクライアントからアクセスできない場合、`temporaryUrl`メソッドを使って一時ストレージのURLを生成しても、MinIOを使用しているときには機能しないでしょう。

<a name="obtaining-disk-instances"></a>
## ディスクインスタンスの取得

`Storage`ファサードは、設定済みのディスクと対話するために使用できます。たとえば、ファサードで`put`メソッドを使用して、アバターをデフォルトのディスクに保存できます。最初に`disk`メソッドを呼び出さずに`Storage`ファサードのメソッドを呼び出すと、メソッドは自動的にデフォルトのディスクに渡されます。

    use Illuminate\Support\Facades\Storage;

    Storage::put('avatars/1', $content);

アプリケーションが複数のディスクを操作する場合は、`Storage`ファサードで`disk`メソッドを使用し、特定のディスク上のファイルを操作できます。

    Storage::disk('s3')->put('avatars/1', $content);

<a name="on-demand-disks"></a>
### オンデマンドディスク

時には、アプリケーションの`filesystems`設定ファイルに実際にその構成が存在しなくても、指定する構成を使用して実行時にディスクを作成したい場合があります。これを実現するため、`Storage`ファサードの`build`メソッドへ設定配列を渡せます。

```php
use Illuminate\Support\Facades\Storage;

$disk = Storage::build([
    'driver' => 'local',
    'root' => '/path/to/root',
]);

$disk->put('image.jpg', $content);
```

<a name="retrieving-files"></a>
## ファイルの取得

`get`メソッドを使用して、ファイルの内容を取得できます。ファイルの素の文字列の内容は、メソッドによって返されます。すべてのファイルパスは、ディスクの「ルート」の場所を基準にして指定する必要があることに注意してください。

    $contents = Storage::get('file.jpg');

取得するファイルがJSONを含んでいる場合、`json`メソッドを使用してファイルを取得し、その内容をデコードできます。

    $orders = Storage::json('orders.json');

`exists`メソッドを使用して、ファイルがディスクに存在するかどうかを判定できます。

    if (Storage::disk('s3')->exists('file.jpg')) {
        // ...
    }

`missing`メソッドを使用して、ファイルがディスク存在していないことを判定できます。

    if (Storage::disk('s3')->missing('file.jpg')) {
        // ...
    }

<a name="downloading-files"></a>
### ファイルのダウンロード

`download`メソッドを使用して、ユーザーのブラウザに指定したパスでファイルをダウンロードするように強制するレスポンスを生成できます。`download`メソッドは、メソッドの２番目の引数としてファイル名を受け入れます。これにより、ユーザーがファイルをダウンロードするときに表示されるファイル名が決まります。最後に、HTTPヘッダの配列をメソッドの３番目の引数として渡すことができます。

    return Storage::download('file.jpg');

    return Storage::download('file.jpg', $name, $headers);

<a name="file-urls"></a>
### ファイルのURL

`url`メソッドを使用し、特定のファイルのURLを取得できます。`local`ドライバを使用している場合、これは通常、指定されたパスの前に`/storage`を追加し、ファイルへの相対URLを返します。`s3`ドライバを使用している場合は、完全修飾リモートURLが返されます。

    use Illuminate\Support\Facades\Storage;

    $url = Storage::url('file.jpg');

`local`ドライバを使用する場合、パブリックにアクセス可能である必要があるすべてのファイルは、`storage/app/public`ディレクトリに配置する必要があります。さらに、`storage/app/public`ディレクトリを指す`public/storage`に[シンボリックリンクを作成](#the-public-disk)する必要があります。

> [!WARNING]
> `local`ドライバを使用する場合、`url`の戻り値はURLエンコードされません。このため、常に有効なURLを作成する名前を使用してファイルを保存することをお勧めします。

<a name="url-host-customization"></a>
#### URLホストのカスタマイズ

`Storage`ファサードを使用して生成したURLのホストを変更したい場合は、ディスクの設定配列へ`url`オプションを追加または変更してください。

    'public' => [
        'driver' => 'local',
        'root' => storage_path('app/public'),
        'url' => env('APP_URL').'/storage',
        'visibility' => 'public',
        'throw' => false,
    ],

<a name="temporary-urls"></a>
### 一時的なURL

`temporaryUrl`メソッドを使用すると、`local`と`s3`ドライバを使用して保存されたファイルへの一時URLを作成できます。このメソッドは、パスと、URLの有効期限を指定する`DateTime`インスタンスを受け入れます。

    use Illuminate\Support\Facades\Storage;

    $url = Storage::temporaryUrl(
        'file.jpg', now()->addMinutes(5)
    );

<a name="enabling-local-temporary-urls"></a>
#### ローカル一時URLの有効化

一時URLのサポートを`local`ドライバへ導入する前に、アプリケーションの開発を開始した場合は、ローカルの一時URLを有効にする必要があります。そのために、`config/filesystems.php`設定ファイル内の`local`ディスクの設定配列に、`serve`オプションを追加してください。

```php
'local' => [
    'driver' => 'local',
    'root' => storage_path('app/private'),
    'serve' => true, // [tl! add]
    'throw' => false,
],
```

<a name="s3-request-parameters"></a>
#### S3リクエストパラメータ

追加の[S3リクエストパラメータ](https://docs.aws.amazon.com/AmazonS3/latest/API/RESTObjectGET.html#RESTObjectGET-requests)を指定する必要がある場合は、リクエストパラメータの配列を`temporaryUrl`メソッドの引数の３番目として渡すことができます。

    $url = Storage::temporaryUrl(
        'file.jpg',
        now()->addMinutes(5),
        [
            'ResponseContentType' => 'application/octet-stream',
            'ResponseContentDisposition' => 'attachment; filename=file2.jpg',
        ]
    );

<a name="customizing-temporary-urls"></a>
#### 一時URLのカスタマイズ

特定するストレージディスクに対する一時的なURLの生成方法をカスタマイズする必要がある場合、 `buildTemporaryUrlsUsing` メソッドを使用してください。例えば、一時的なURLを通常サポートしていないディスクに保存されているファイルをダウンロードできるコントローラがある場合、これは便利です。通常、このメソッドはサービスプロバイダの`boot`メソッドから呼び出します。

    <?php

    namespace App\Providers;

    use DateTime;
    use Illuminate\Support\Facades\Storage;
    use Illuminate\Support\Facades\URL;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * 全アプリケーションサービスの初期起動処理
         */
        public function boot(): void
        {
            Storage::disk('local')->buildTemporaryUrlsUsing(
                function (string $path, DateTime $expiration, array $options) {
                    return URL::temporarySignedRoute(
                        'files.download',
                        $expiration,
                        array_merge($options, ['path' => $path])
                    );
                }
            );
        }
    }

<a name="temporary-upload-urls"></a>
#### 一時的なアップロードURL

> [!WARNING]
> 一時的なアップロードURLの生成機能は、`s3`ドライバのみサポートしています。

クライアントサイドのアプリケーションから、直接ファイルをアップロードするために使用する一時的なURLを生成する必要がある場合は、`temporaryUploadUrl`メソッドを使用します。このメソッドには、パスとURLの有効期限を指定する`DateTime`インスタンスを指定します。`temporaryUploadUrl`メソッドからは、アップロードURLとアップロードリクエストに含めるべきヘッダを連想配列で返します。

    use Illuminate\Support\Facades\Storage;

    ['url' => $url, 'headers' => $headers] = Storage::temporaryUploadUrl(
        'file.jpg', now()->addMinutes(5)
    );

このメソッドは、主にクライアントサイドのアプリケーションがAmazon S3などのクラウドストレージシステムに直接ファイルをアップロードする必要があるサーバレス環境で役立つでしょう。

<a name="file-metadata"></a>
### ファイルメタデータ

Laravelは、ファイルの読み取りと書き込みに加えて、ファイル自体に関する情報も提供できます。たとえば、`size`メソッドを使用して、ファイルのサイズをバイト単位で取得できます。

    use Illuminate\Support\Facades\Storage;

    $size = Storage::size('file.jpg');

`lastModified`メソッドは、ファイルが最後に変更されたときのUNIXタイムスタンプを返します。

    $time = Storage::lastModified('file.jpg');

指定ファイルのMIMEタイプは、`mimeType`メソッドで取得できます。

    $mime = Storage::mimeType('file.jpg');

<a name="file-paths"></a>
#### ファイルパス

`path`メソッドを使用して、特定のファイルのパスを取得できます。`local`ドライバを使用している場合、これはファイルへの絶対パスを返します。`s3`ドライバを使用している場合、このメソッドはS3バケット内のファイルへの相対パスを返します。

    use Illuminate\Support\Facades\Storage;

    $path = Storage::path('file.jpg');

<a name="storing-files"></a>
## ファイルの保存

`put`メソッドは、ファイルの内容をディスクに保存するために使用します。PHPの`resource`を`put`メソッドに渡すこともできます。このメソッドは、Flysystemの基盤となるストリームサポートを使用します。すべてのファイルパスは、ディスク用に設定された「ルート」の場所を基準にして指定する必要があることに注意してください。

    use Illuminate\Support\Facades\Storage;

    Storage::put('file.jpg', $contents);

    Storage::put('file.jpg', $resource);

<a name="failed-writes"></a>
#### 書き込みの失敗

`put`メソッド、もしくは他の「書き込み」操作でディスクへファイルを書き込めない場合は、`false`が返ります。

    if (! Storage::put('file.jpg', $contents)) {
        // ファイルがディスクへ書き込めなかった
    }

必要であれば、ファイルシステムのディスクの設定配列で、`throw`オプションを定義できます。このオプションを`true`に定義すると、`put`のような「書き込み」メソッドの書き込み失敗時に、`League\Flysystem\UnableToWriteFile`インスタンスを投げます。

    'public' => [
        'driver' => 'local',
        // ...
        'throw' => true,
    ],

<a name="prepending-appending-to-files"></a>
### ファイルの前後への追加

`prepend`および`append`メソッドを使用すると、ファイルの最初または最後に書き込むことができます。

    Storage::prepend('file.log', 'Prepended Text');

    Storage::append('file.log', 'Appended Text');

<a name="copying-moving-files"></a>
### ファイルのコピーと移動

`copy`メソッドを使用して、既存のファイルをディスク上の新しい場所にコピーできます。また、`move`メソッドを使用して、既存のファイルの名前を変更したり、新しい場所に移動したりできます。

    Storage::copy('old/file.jpg', 'new/file.jpg');

    Storage::move('old/file.jpg', 'new/file.jpg');

<a name="automatic-streaming"></a>
### 自動ストリーミング

ファイルをストレージにストリーミングすると、メモリ使用量が大幅に削減されます。Laravelに特定のファイルの保存場所へのストリーミングを自動的に管理させたい場合は、`putFile`または`putFileAs`メソッドを使用します。このメソッドは、`Illuminate\Http\File`または`Illuminate\Http\UploadedFile`インスタンスのいずれかを引数に取り、ファイルを目的の場所へ自動的にストリーミングします。

    use Illuminate\Http\File;
    use Illuminate\Support\Facades\Storage;

    // ファイル名の一意のIDを自動的に生成
    $path = Storage::putFile('photos', new File('/path/to/photo'));

    // ァイル名を手作業で指定
    $path = Storage::putFileAs('photos', new File('/path/to/photo'), 'photo.jpg');

`putFile`メソッドには注意すべき重要な点がいくつかあります。ファイル名ではなく、ディレクトリ名のみを指定することに注意してください。デフォルトでは、`putFile`メソッドはファイル名として働く一意のIDを生成します。ファイルの拡張子は、ファイルのMIMEタイプを調べることによって決定されます。ファイルへのパスは`putFile`メソッドが返すため、生成されたファイル名を含むパスをデータベースへ保存できます。

`putFile`メソッドと`putFileAs`メソッドは、保存するファイルの「可視性」を指定する引数も取ります。これは、AmazonS3などのクラウドディスクにファイルを保存していて、生成されたURLを介してファイルへパブリックアクセスできるようにする場合、特に便利です。

    Storage::putFile('photos', new File('/path/to/photo'), 'public');

<a name="file-uploads"></a>
### ファイルのアップロード

Webアプリケーションでは、ファイルを保存するための最も一般的な使用例の1つは、写真やドキュメントなどのユーザーがアップロードしたファイルを保存することです。Laravelを使用すると、アップロードされたファイルインスタンスに`store`メソッドを使用し、ファイルを非常に簡単に保存できます。アップロード済みファイルを保存するパスを指定して`store`メソッドを呼び出します。

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Http\Request;

    class UserAvatarController extends Controller
    {
        /**
         * ユーザーのアバターを更新
         */
        public function update(Request $request): string
        {
            $path = $request->file('avatar')->store('avatars');

            return $path;
        }
    }

この例で注意すべき重要なことがあります。ファイル名ではなく、ディレクトリ名のみを指定したことに注意してください。デフォルトでは、`store`メソッドはファイル名として機能する一意のIDを生成します。ファイルの拡張子は、ファイルのMIMEタイプを調べることによって決定されます。ファイルへのパスは`store`メソッドによって返されるため、生成されたファイル名を含むパスをデータベースに保存できます。

`Storage`ファサードで`putFile`メソッドを呼び出して、上記の例と同じファイルストレージ操作を実行することもできます。

    $path = Storage::putFile('avatars', $request->file('avatar'));

<a name="specifying-a-file-name"></a>
#### ファイル名の指定

保存されたファイルにファイル名を自動的に割り当てたくない場合は、引数としてパス、ファイル名、および(オプションの)ディスクを受け取る`storeAs`メソッドを使用します。

    $path = $request->file('avatar')->storeAs(
        'avatars', $request->user()->id
    );

`Storage`ファサードで`putFileAs`メソッドを使用することもできます。これにより、上記の例と同じファイルストレージ操作が実行されます。

    $path = Storage::putFileAs(
        'avatars', $request->file('avatar'), $request->user()->id
    );

> [!WARNING]
> 印刷できない無効なUnicode文字はファイルパスから自動的に削除されます。したがって、Laravelのファイルストレージメソッドに渡す前に、ファイルパスをサニタイズすることをお勧めします。ファイルパスは、`League\Flysystem\WhitespacePathNormalizer::normalizePath`メソッドを使用して正規化されます。

<a name="specifying-a-disk"></a>
#### ディスクの指定

デフォルトでは、このアップロード済みファイルの`store`メソッドはデフォルトのディスクを使用します。別のディスクを指定する場合は、ディスク名を２番目の引数として`store`メソッドに渡します。

    $path = $request->file('avatar')->store(
        'avatars/'.$request->user()->id, 's3'
    );

`storeAs`メソッドを使用している場合は、ディスク名を３番目の引数としてメソッドに渡すことができます。

    $path = $request->file('avatar')->storeAs(
        'avatars',
        $request->user()->id,
        's3'
    );

<a name="other-uploaded-file-information"></a>
#### アップロード済みファイルのその他の情報

アップロードされたファイルの元の名前と拡張子を取得したい場合は、`getClientOriginalName`と`getClientOriginalExtension`メソッドを使って取得します。

    $file = $request->file('avatar');

    $name = $file->getClientOriginalName();
    $extension = $file->getClientOriginalExtension();

ただし，悪意のあるユーザーによりファイル名や拡張子が改竄される可能性があるため，`getClientOriginalName`と`getClientOriginalExtension`メソッドは安全であると考えられないことに注意してください。そのため，アップロードされたファイルの名前と拡張子を取得するには，通常，`hashName`メソッドと`extension`メソッドを使用するべきです。

    $file = $request->file('avatar');

    $name = $file->hashName(); // ユニークでランダムな名前を生成する
    $extension = $file->extension(); // ファイルのMIMEタイプに基づき拡張子を決める

<a name="file-visibility"></a>
### ファイルの可視性

LaravelのFlysystem統合では、「可視性」は複数のプラットフォームにわたるファイル権限の抽象化です。ファイルは`public`または`private`として宣言できます。ファイルが`public`と宣言されている場合、そのファイルは一般的に他のユーザーがアクセスできる必要があることを示しています。たとえば、S3ドライバを使用する場合、`public`ファイルのURLを取得できます。

`put`メソッドを介してファイルを書き込むとき、可視性を設定できます。

    use Illuminate\Support\Facades\Storage;

    Storage::put('file.jpg', $contents, 'public');

ファイルがすでに保存されている場合、その可視性は、`getVisibility`および`setVisibility`メソッドにより取得および設定できます。

    $visibility = Storage::getVisibility('file.jpg');

    Storage::setVisibility('file.jpg', 'public');

アップロード済みファイルを操作するときは、`storePublicly`メソッドと`storePubliclyAs`メソッドを使用して、アップロード済みファイルを`public`の可視性で保存できます。

    $path = $request->file('avatar')->storePublicly('avatars', 's3');

    $path = $request->file('avatar')->storePubliclyAs(
        'avatars',
        $request->user()->id,
        's3'
    );

<a name="local-files-and-visibility"></a>
#### ローカルファイルと可視性

`local`ドライバを使用する場合、`public`の[可視性](#file-visibility)は、ディレクトリの`0755`パーミッションとファイルの`0644`パーミッションに変換されます。アプリケーションの`filesystems`設定ファイルでパーミッションマッピングを変更できます。

    'local' => [
        'driver' => 'local',
        'root' => storage_path('app'),
        'permissions' => [
            'file' => [
                'public' => 0644,
                'private' => 0600,
            ],
            'dir' => [
                'public' => 0755,
                'private' => 0700,
            ],
        ],
        'throw' => false,
    ],

<a name="deleting-files"></a>
## ファイルの削除

`delete`メソッドは、削除する単一のファイル名またはファイルの配列を受け入れます。

    use Illuminate\Support\Facades\Storage;

    Storage::delete('file.jpg');

    Storage::delete(['file.jpg', 'file2.jpg']);

必要に応じて、ファイルを削除するディスクを指定できます。

    use Illuminate\Support\Facades\Storage;

    Storage::disk('s3')->delete('path/file.jpg');

<a name="directories"></a>
## ディレクトリ

<a name="get-all-files-within-a-directory"></a>
#### ディレクトリ内のすべてのファイルを取得

`files`メソッドは、指定されたディレクトリ内のすべてのファイルの配列を返します。すべてのサブディレクトリを含む、特定のディレクトリ内のすべてのファイルのリストを取得する場合は、`allFiles`メソッドを使用できます。

    use Illuminate\Support\Facades\Storage;

    $files = Storage::files($directory);

    $files = Storage::allFiles($directory);

<a name="get-all-directories-within-a-directory"></a>
#### ディレクトリ内のすべてのディレクトリを取得

`directories`メソッドは、指定されたディレクトリ内のすべてのディレクトリの配列を返します。さらに、`allDirectories`メソッドを使用して、特定のディレクトリ内のすべてのディレクトリとそのすべてのサブディレクトリのリストを取得できます。

    $directories = Storage::directories($directory);

    $directories = Storage::allDirectories($directory);

<a name="create-a-directory"></a>
#### ディレクトリを作成する

`makeDirectory`メソッドは、必要なサブディレクトリを含む、指定したディレクトリを作成します。

    Storage::makeDirectory($directory);

<a name="delete-a-directory"></a>
#### ディレクトリを削除する

最後に、`deleteDirectory`メソッドを使用して、ディレクトリとそのすべてのファイルを削除できます。

    Storage::deleteDirectory($directory);

<a name="testing"></a>
## テスト

`Storage`ファサードの`fake`メソッドを使うと、簡単に偽のディスクを生成できます。これを`Illuminate\Http\UploadedFile`クラスのファイル生成ユーティリティと組み合わせると、ファイルのアップロードのテストが非常に簡単になります。例えば、以下のようにです。

```php tab=Pest
<?php

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

test('albums can be uploaded', function () {
    Storage::fake('photos');

    $response = $this->json('POST', '/photos', [
        UploadedFile::fake()->image('photo1.jpg'),
        UploadedFile::fake()->image('photo2.jpg')
    ]);

    // 一つ以上のファイルを保存することをアサート
    Storage::disk('photos')->assertExists('photo1.jpg');
    Storage::disk('photos')->assertExists(['photo1.jpg', 'photo2.jpg']);

    // ファイルを保存しないことをアサート
    Storage::disk('photos')->assertMissing('missing.jpg');
    Storage::disk('photos')->assertMissing(['missing.jpg', 'non-existing.jpg']);

    // 指定ディレクトリ内のファイル数が、期待値と一致することをアサート
    Storage::disk('photos')->assertCount('/wallpapers', 2);

    // 指定ディレクトリが空であることをアサート
    Storage::disk('photos')->assertDirectoryEmpty('/wallpapers');
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_albums_can_be_uploaded(): void
    {
        Storage::fake('photos');

        $response = $this->json('POST', '/photos', [
            UploadedFile::fake()->image('photo1.jpg'),
            UploadedFile::fake()->image('photo2.jpg')
        ]);

        // 一つ以上のファイルを保存することをアサート
        Storage::disk('photos')->assertExists('photo1.jpg');
        Storage::disk('photos')->assertExists(['photo1.jpg', 'photo2.jpg']);

        // ファイルを保存しないことをアサート
        Storage::disk('photos')->assertMissing('missing.jpg');
        Storage::disk('photos')->assertMissing(['missing.jpg', 'non-existing.jpg']);

        // 指定ディレクトリ内のファイル数が、期待値と一致することをアサート
        Storage::disk('photos')->assertCount('/wallpapers', 2);

        // 指定ディレクトリが空であることをアサート
        Storage::disk('photos')->assertDirectoryEmpty('/wallpapers');
    }
}
```

`fake`メソッドはデフォルトで、テンポラリディレクトリのファイルをすべて削除します。もしこれらのファイルを残しておきたい場合は、代わりに"persistentFake"メソッドを使用してください。ファイルアップロードのテストに関するより詳しい情報は、[ファイルアップロードに関するHTTPテストのドキュメント](/docs/{{version}}/http-tests#testing-file-uploads)を参照してください。

> [!WARNING]
> `image`メソッドには、[GD拡張](https://www.php.net/manual/ja/book.image.php)が必要です。

<a name="custom-filesystems"></a>
## カスタムファイルシステム

LaravelのFlysystem統合は、最初からすぐに使える「ドライバ」をいくつかサポートしています。ただし、Flysystemはこれらに限定されず、他の多くのストレージシステム用のアダプターを備えています。Laravelアプリケーションでこれらの追加アダプターの１つを使用する場合は、カスタムドライバを作成できます。

カスタムファイルシステムを定義するには、Flysystemアダプターが必要です。コミュニティが管理するDropboxアダプターをプロジェクトに追加してみましょう。

```shell
composer require spatie/flysystem-dropbox
```

次に、アプリケーションの[サービスプロバイダ](/docs/{{version}}/provider)の1つの`boot`メソッド内にドライバを登録します。これには、`Storage`ファサードの`extend`メソッドを使用します。

    <?php

    namespace App\Providers;

    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Filesystem\FilesystemAdapter;
    use Illuminate\Support\Facades\Storage;
    use Illuminate\Support\ServiceProvider;
    use League\Flysystem\Filesystem;
    use Spatie\Dropbox\Client as DropboxClient;
    use Spatie\FlysystemDropbox\DropboxAdapter;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * 全アプリケーションサービスの登録
         */
        public function register(): void
        {
            // ...
        }

        /**
         * 全アプリケーションサービスの初期起動処理
         */
        public function boot(): void
        {
            Storage::extend('dropbox', function (Application $app, array $config) {
                $adapter = new DropboxAdapter(new DropboxClient(
                    $config['authorization_token']
                ));

                return new FilesystemAdapter(
                    new Filesystem($adapter, $config),
                    $adapter,
                    $config
                );
            });
        }
    }

`extend`メソッドの第１引数はドライバ名前で、第２引数は変数`$app`と`$config`を受け取るクロージャです。このクロージャは`Illuminate\Filesystem\FilesystemAdapter`のインスタンスを返さなければなりません。変数`$config`には、指定したディスクの`config/filesystems.php`で定義している値が格納されます。

拡張機能のサービスプロバイダを作成・登録したら、`config/filesystems.php`設定ファイルで`dropbox`ドライバを使用できます。
