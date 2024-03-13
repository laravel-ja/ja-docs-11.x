# Laravel Valet

- [イントロダクション](#introduction)
- [インストール](#installation)
    - [Valetのアップグレード](#upgrading-valet)
- [サイト動作](#serving-sites)
    - ["Park"コマンド](#the-park-command)
    - ["Link"コマンド](#the-link-command)
    - [TLSによる安全なサイト](#securing-sites)
    - [デフォルトサイトの提供](#serving-a-default-site)
    - [サイトごとのPHPバージョン](#per-site-php-versions)
- [サイト共有](#sharing-sites)
    - [ローカルネットワーク上のサイト共有](#sharing-sites-on-your-local-network)
- [サイト限定環境変数](#site-specific-environment-variables)
- [プロキシサーバ](#proxying-services)
- [カスタムValetドライバ](#custom-valet-drivers)
    - [ローカルドライバ](#local-drivers)
- [その他のValetコマンド](#other-valet-commands)
- [Valetのディレクトリとファイル](#valet-directories-and-files)
    - [ディスクアクセス](#disk-access)

<a name="introduction"></a>
## イントロダクション

> [!NOTE]
> macOS上でLaravelアプリケーションをより簡単に開発する方法をお探しですか？[Laravel Herd](https://herd.laravel.com)をチェックしてください。Valet、PHP、Composerなど、Laravel開発を始めるために必要なものをHerdはすべて用意します。

[Laravel Valet](https://github.com/laravel/valet)（バレット：従者）は、macOSミニマリスト向けのLaravel開発環境です。Larave lValetは、マシンの起動時に常にバックグラウンドで[Nginx](https://www.nginx.com/)を実行するようにMacを設定します。次に、[DnsMasq](https://en.wikipedia.org/wiki/Dnsmasq)を使用して、Valetは`*.test`ドメイン上のすべてのリクエストをプロキシし、ローカルマシンにインストールしているサイトへ転送します。

言い換えれば、Valetは、約７MBのRAMを使用する非常に高速なLaravel開発環境です。Valetは、[Sail](/docs/{{version}}/sale)や[Homestead](/docs/{{version}}/homestead)の完全な代替ではありませんが、極端に速度を好むとかRAMの量が限られているマシンで作業しているなど、柔軟な開発環境の基本が必要な場合は優れた代替手段になるでしょう。

Valetは以下をサポートしていますが、これらに限定されません。

<style>
    #valet-support > ul {
        column-count: 3; -moz-column-count: 3; -webkit-column-count: 3;
        line-height: 1.9;
    }
</style>

<div id="valet-support" markdown="1">

- [Laravel](https://laravel.com)
- [Bedrock](https://roots.io/bedrock/)
- [CakePHP 3](https://cakephp.org)
- [ConcreteCMS](https://www.concretecms.com/)
- [Contao](https://contao.org/en/)
- [Craft](https://craftcms.com)
- [Drupal](https://www.drupal.org/)
- [ExpressionEngine](https://www.expressionengine.com/)
- [Jigsaw](https://jigsaw.tighten.co)
- [Joomla](https://www.joomla.org/)
- [Katana](https://github.com/themsaid/katana)
- [Kirby](https://getkirby.com/)
- [Magento](https://magento.com/)
- [OctoberCMS](https://octobercms.com/)
- [Sculpin](https://sculpin.io/)
- [Slim](https://www.slimframework.com)
- [Statamic](https://statamic.com)
- Static HTML
- [Symfony](https://symfony.com)
- [WordPress](https://wordpress.org)
- [Zend](https://framework.zend.com)

</div>

独自の[カスタムドライバ](#custom-valet-drivers)でValetを拡張できます。

<a name="installation"></a>
## インストール

> [!WARNING]
> ValetにはmacOSと[Homebrew](https://brew.sh/)が必要です。インストールする前に、ApacheやNginxなどの他のプログラムがローカルマシンのポート80をバインドしていないことを確認する必要があります。

開始するには、最初に`update`コマンドを使用してHomebrewが最新であることを確認する必要があります。

```shell
brew update
```

次に、Homebrewを使用してPHPをインストールする必要があります。

```shell
brew install php
```

PHPをインストールしたら、[Composerパッケージマネージャ](https://getcomposer.org)をインストールする準備が整います。さらに、`$HOME/.composer/vendor/bin`ディレクトリがシステムの「PATH」にあることを確認する必要があります。Composerをインストールできたら、Laravel ValetをグローバルComposerパッケージとしてインストールできます。

```shell
composer global require laravel/valet
```

最後に、Valetの`install`コマンドを実行します。これにより、ValetとDnsMasqが設定およびインストールされます。さらに、Valetが依存しているデーモンが、システムの起動時に起動するように設定されます。

```shell
valet install
```

Valetをインストールしたら、`pingfoobar.test`などのコマンドを使用してターミナルの`*.test`ドメインにpingを実行してみてください。Valetが正しくインストールされている場合、このドメインが`127.0.0.1`でレスポンスしているのがわかります。

マシンが起動するたび、Valetは自動的に必要なサービスを開始します。

<a name="php-versions"></a>
#### PHPバージョン

> [!NOTE]
> グローバルにPHPバージョンを変更する代わりに、`isolate` [コマンド](#per-site-php-versions)を使い、サイトごとのPHPバージョンを使用するようにValetへ指示できます。

Valetでは、`valet use php@version`コマンドを使用してPHPのバージョンを切り替えることができます。Valetは、指定するPHPバージョンがまだインストールされていない場合、Homebrewを介してインストールします。

```shell
valet use php@8.2

valet use php
```

プロジェクトのルートに`.valetrc`ファイルを作成することもできます。`.valetrc`ファイルには、サイトで使用するPHPバージョンを指定する必要があります。

```shell
php=php@8.2
```

このファイルを作成したら、`valet use`コマンドを実行してください。コマンドはファイルを読み、サイトで優先するPHPバージョンを決めます。

> [!WARNING]
> 複数のPHPバージョンをインストールしている場合でも、Valetは一度に一つのPHPバージョンのみを提供します。

<a name="database"></a>
#### データベース

アプリケーションにデータベースが必要な場合は、[DBngin](https://dbngin.com)を確認してください。DBnginは、MySQL、PostgreSQL、およびRedisを含む無料のオールインワンデータベース管理ツールを提供します。DBnginをインストールした後、`root`ユーザー名とパスワードに空の文字列を使用して、`127.0.0.1`でデータベースに接続できます。

<a name="resetting-your-installation"></a>
#### インストレーションのリセット

Valetインストレーションが正しく動作せずに問題が起きた時は、`composer global require laravel/valet`の後に、`valet install`を実行してください。これによりインストール済みのValetがリセットされ、さまざまな問題が解決されます。稀にValetを「ハードリセット」する必要がある場合もあり、その場合は`valet install`の前に`valet uninstall --force`を実行してください。

<a name="upgrading-valet"></a>
### Valetのアップグレード

ターミナルで`composer global require laravel/valet`コマンドを実行すると、Valetのインストールを更新できます。アップグレード後、`valet install`コマンドを実行して、Valetが必要に応じて設定ファイルへ追加のアップグレードを行うことを推奨します。

<a name="upgrading-to-valet-4"></a>
#### Valet4へのアップグレード

Valet3からValet4へアップグレードするときは、Valetのインストールを以下の手順に従い、正しくアップグレードしてください。

<div class="content-list" markdown="1">

- サイトのPHP バージョンをカスタマイズするため、`.valetphprc`ファイルを追加している場合は、各`.valetphprc`ファイルを`.valetrc`へリネームしてください。次に、`.valetrc`ファイルの既存の内容の前に、`php=`を追加します。
- 新しいドライバシステムの名前空間、拡張子、タイプヒント、戻り値タイプヒントが一致するように、カスタムドライバを更新してください。例としてValetの[SampleValetDriver](https://github.com/laravel/valet/blob/d7787c025e60abc24a5195dc7d4c5c6f2d984339/cli/stubs/SampleValetDriver.php)を参考にするとよいでしょう。
- PHP7.1～7.4をサイトで使用しているなら、主に使用するバージョンでなくてもPHP8.0以上のバージョンを確実にHomebrewでインストールしてください。Valetが自身のスクリプトの実行に、このバージョンを使用します。

</div>

<a name="serving-sites"></a>
## サイト動作

Valetがインストールされると、Laravelアプリケーションの提供を開始する準備が整います。Valetは、アプリケーションの提供に役立つ２つのコマンド`park`と`link`を提供しています。

<a name="the-park-command"></a>
### `park`コマンド

`park`コマンドは、アプリケーションを含むマシン上のディレクトリを登録します。ディレクトリがValetで「パーク」されると、そのディレクトリ内のすべてのディレクトリにWebブラウザの`http://<directory-name>.test`からアクセスできるようになります。

```shell
cd ~/Sites

valet park
```

これだけです。これで、「パークした」ディレクトリ内に作成したアプリケーションはすべて、「http://<directory-name>.test」規則を使用して自動的に提供されます。したがって、パーキングディレクトリに「laravel」という名前のディレクトリが含まれている場合、そのディレクトリ内のアプリケーションには``http://laravel.test`からアクセスできます。付け加えて、Valetではワイルドカードのサブドメイン（`http://foo.laravel.test`）を使ったアクセスも自動的に可能です。

<a name="the-link-command"></a>
### `link`コマンド

`link`コマンドを使用してLaravelアプリケーションを提供することもできます。このコマンドは、ディレクトリ全体ではなく、ディレクトリ内の単一のサイトにサービスを提供する場合に役立ちます。

```shell
cd ~/Sites/laravel

valet link
```

`link`コマンドを使用してアプリケーションがValetにリンクされると、そのディレクトリ名を使用してアプリケーションにアクセスできます。したがって、上記の例でリンクされたサイトは、`http://laravel.test`でアクセスできます。付け加えて、Valetではワイルドカードのサブドメイン（`http://foo.laravel.test`）を使ったアクセスも自動的に可能です。

別のホスト名でアプリケーションを提供する場合は、ホスト名を`link`コマンドへ渡せます。たとえば、以下のコマンドを実行して、アプリケーションを`http://application.test`で利用できます。

```shell
cd ~/Sites/laravel

valet link application
```

もちろん、`link`コマンドを使用してサブドメイン上のアプリケーションを提供することもできます。

```shell
valet link api.application
```

`links`コマンドを実行して、リンクされているすべてのディレクトリのリストを表示できます。

```shell
valet links
```

`unlink`コマンドは、サイトへのシンボリックリンクを破棄できます。

```shell
cd ~/Sites/laravel

valet unlink
```

<a name="securing-sites"></a>
### TLSによるサイト保護

デフォルトでは、ValetはHTTP経由でサイトへサービスを提供します。ただし、HTTP/2を使用して暗号化されたTLSを介してサイトにサービスを提供する場合は、`secure`コマンドを使用できます。たとえば、サイトが`laravel.test`ドメインでValetによって提供されている場合は、次のコマンドを実行してサイトを保護する必要があります。

```shell
valet secure laravel
```

サイトを「保護解除」し、プレーンHTTPを介したトラフィックの提供に戻すには、`unsecure`コマンドを使用します。`secure`コマンドと同様に、このコマンドはセキュリティで保護しなくするホスト名を指定します。

```shell
valet unsecure laravel
```

<a name="serving-a-default-site"></a>
### デフォルトサイトの提供

時には、未知の`test`ドメインを訪問したときに、`404`の代わりに「デフォルト」サイトを提供するよう、Valetを設定したいことがあるかもしれません。これを実現するには、デフォルトサイトとして機能するサイトへのパスを含む `~/.config/valet/config.json`設定ファイルへ`default`オプションを追加します。

    "default": "/Users/Sally/Sites/example-site",

<a name="per-site-php-versions"></a>
### サイトごとのPHPバージョン

Valetはサイトにサービスを提供するため、デフォルトでグローバルなPHPインストールを使用します。しかし、さまざまなサイトで複数のPHPバージョンをサポートする必要がある場合は、`isolate`コマンドを使用して、特定のサイトが使用するPHPバージョンを指定してください。`isolate`コマンドは、現在の作業ディレクトリにあるサイトに対し、Valetが指定したPHPバージョンを使用するように設定します。

```shell
cd ~/Sites/example-site

valet isolate php@8.0
```

サイト名がそれを含むディレクトリ名と一致しない場合は、`--site`オプションを使い、サイト名を指定します。

```shell
valet isolate php@8.0 --site="site-name"
```

便利なように、`valet php`、`composer`、`which-php`コマンドはプロキシされ、サイトのPHPバージョンに応じて適切な PHP CLIやツールが呼び出されます。

```shell
valet php
valet composer
valet which-php
```

個別のサイトとPHPバージョンの全一覧を表示するには、`isolated`コマンドを実行してください。

```shell
valet isolated
```

サイトのPHPをValetのグローバルインストールしたPHPバージョンに戻すには、サイトのルートディレクトリで、`unisolate`コマンドを実行します。

```shell
valet unisolate
```

<a name="sharing-sites"></a>
## サイト共有

Valetは、ローカルサイトを世界へ公開し共有するコマンドも用意しており、モバイルデバイスでサイトをテストしたり、チームメンバーやクライアントと共有したりする簡単な方法を提供しています。

はじめからValetは、ngrokやExposeを使ったサイトの共有に対応しています。サイトを共有する前に、`share-tool`コマンドを使用し、`ngrok`または`expose`を指定して、Valetの設定を更新する必要があります。

```shell
valet share-tool ngrok
```

選択したツールが、Homebrew（ngrokの場合）やComposer（Exposeの場合）を介して、まだインストールされていない場合、Valetは自動的にインストールするように促します。もちろん、どちらのツールでも、サイトの共有を開始する前に、ngrokもしくはExposeのアカウントを認証する必要があります。

サイトを共有するには、ターミナルでサイトのディレクトリに移動し、Valetの`share`コマンドを実行します。パブリックからアクセス可能なURLがクリップボードにコピーされますので、ブラウザに直接貼り付けたり、チームと共有したりできます。

```shell
cd ~/Sites/laravel

valet share
```

To stop sharing your site, you may press `Control + C`.

> [!WARNING]
> カスタムDNSサーバ（`1.1.1.1`など）を使用している場合、ngrok共有が正しく動作しないかもしれません。このような場合は、Macのシステム設定を開き、ネットワーク設定へ行き、詳細設定を開き、DNSタブを開き、最初のDNSサーバとして`127.0.0.1`を追加します。

<a name="sharing-sites-via-ngrok"></a>
#### Ngrokを使うサイト共有

ngrokを使用してサイトを共有するには、[ngrokアカウントの作成](https://dashboard.ngrok.com/signup)と[認証トークンの設定](https://dashboard.ngrok.com/get-started/your-authtoken)が必要です。認証トークンを取得したら、そのトークンを使ってValetの設定を更新できます。

```shell
valet set-ngrok-token YOUR_TOKEN_HERE
```

> [!NOTE]
> `valet share --region=eu`のように、共有コマンドに追加のngrokパラメータを渡せます。詳しくは、[ngrokのドキュメント](https://ngrok.com/docs)を参照してください。

<a name="sharing-sites-via-expose"></a>
#### Exposeを使うサイト共有

Exposeを使ってサイトを共有するには、[Exposeアカウントの作成](https://expose.dev/register)と[認証トークンによるExposeへの認証](https://expose.dev/docs/getting-started/getting-your-token)が必要です。

Exposeがサポートしている追加のコマンドラインパラメータに関する情報は、[Exposeのドキュメント](https://expose.dev/docs)を参照してください。

<a name="sharing-sites-on-your-local-network"></a>
### ローカルネットワークでのサイト共有

Valetは、開発マシンがインターネットからのセキュリティリスクにさらされないように、デフォルトで内部の`127.0.0.1`インターフェイスへの受信トラフィックを制限します。

ローカルネットワーク上の他のデバイスが自分のマシンのIPアドレス（例:`192.168.1.1.10/application.test`）を介して自分のマシン上のValetサイトにアクセスできるようにしたい場合は、そのサイトの適切なNginx設定ファイルを手作業で編集して`listen`ディレクティブの制限を取り除く必要があります。ポート80と443の`listen`ディレクティブのプレフィックス`127.0.0.0.1:`を削除する必要があります。

プロジェクトで`valet secure`を実行していない場合は、`/usr/local/etc/nginx/valet/valet.conf`ファイルを編集し、HTTPSではないサイトへのネットワークアクセスを開けます。あるサイトに対し`valet secure`を実行することで、HTTPSにてプロジェクトサイトを動かしている場合は、`~/.config/valet/Nginx/app-name.test`ファイルを編集する必要があります。

Nginx設定を更新したら、設定の変更を反映するために`valet restart`コマンドを実行してください。

<a name="site-specific-environment-variables"></a>
## サイト限定環境変数

他のフレームワークを使用する一部のアプリケーションは、サーバ環境変数に依存する場合がありますが、それらの変数をプロジェクト内で設定する方法を提供しません。Valetを使用すると、プロジェクトのルート内に`.valet-env.php`ファイルを追加することにより、サイト固有の環境変数を設定できます。このファイルは、配列で指定する各サイトのグローバル`$_SERVER`配列へ追加するサイト／環境変数のペアの配列を返す必要があります。

    <?php

    return [
        // laravel.testサイトの$_SERVER['key']を"value"へ設定
        'laravel' => [
            'key' => 'value',
        ],

        // すべてのサイトで$_SERVER['key']を"value"へ設定
        '*' => [
            'key' => 'value',
        ],
    ];

<a name="proxying-services"></a>
## プロキシサーバ

時にローカルマシンの他のサービスへValetドメインをプロキシ動作させたいこともあるでしょう。たとえば、Valetを実行する一方で、たまにDockerにより別のサイトを実行する必要がある場合です。しかし、ValetとDockerは同時に８０ポートを両方でバインドできません。

これを解決するには、`proxy`コマンドを使いプロキシを生成してください。たとえば、`http://elasticsearch.test`からのトラフィックをすべて`http://127.0.0.1:9200`へ仲介するには、以下のとおりです。

```shell
# Proxy over HTTP...
valet proxy elasticsearch http://127.0.0.1:9200

# Proxy over TLS + HTTP/2...
valet proxy elasticsearch http://127.0.0.1:9200 --secure
```

`unproxy`コマンドでプロキシを削除できます。

```shell
valet unproxy elasticsearch
```

`proxies`コマンドを使用して、プロキシするすべてのサイト設定を一覧表示できます。

```shell
valet proxies
```

<a name="custom-valet-drivers"></a>
## カスタムValetドライバ

独自のValet「ドライバ」を作成して、ValetがネイティブにサポートしていないフレームワークやCMSで実行するPHPアプリケーションを提供できます。Valetをインストールすると、`SampleValetDriver.php`ファイルを含む`〜/.config/valet/Drivers`ディレクトリが作成されます。このファイルには、カスタムドライバの作成方法を示すサンプルドライバ実装が含まれています。ドライバを作成するには、`serves`、`isStaticFile`、および`frontControllerPath`の３つのメソッドを実装するだけです。

全３メソッドは`$sitePath`、`$siteName`、`$uri`を引数で受け取ります。`$sitePath`は、`/Users/Lisa/Sites/my-project`のように、サイトプロジェクトへのフルパスです。`$siteName`は"ホスト" / "サイト名"記法のドメイン(`my-project`)です。`$uri`はやって来たリクエストのURI(`/foo/bar`)です。

カスタムValetドライバが完成したら、`FrameworkValetDriver.php`命名規則を使用して`〜/.config/valet/Drivers`ディレクトリに配置します。たとえば、WordPress用のカスタムバレットドライバを作成している場合、ファイル名は`WordPressValetDriver.php`である必要があります。

カスタムValetドライバで実装する各メソッドのサンプルコードを見ていきましょう。

<a name="the-serves-method"></a>
#### `serves`メソッド

`serves`メソッドは、そのドライバがやって来たリクエストを処理すべき場合に、`true`を返してください。それ以外の場合は`false`を返してください。そのためには、メソッドの中で、渡された`$sitePath`の内容が、動作させようとするプロジェクトタイプを含んでいるかを判定します。

たとえば、`WordPressValetDriver`を書いていると仮定してみましょう。`serves`メソッドは次のようになります。

    /**
     * このドライバでリクエストを処理するか決める
     */
    public function serves(string $sitePath, string $siteName, string $uri): bool
    {
        return is_dir($sitePath.'/wp-admin');
    }

<a name="the-isstaticfile-method"></a>
#### `isStaticFile`メソッド

`isStaticFile`はリクエストが画像やスタイルシートのような「静的」なファイルであるかを判定します。ファイルが静的なものであれば、そのファイルが存在するディスク上のフルパスを返します。リクエストが静的ファイルでない場合は、`false`を返します。

    /**
     * リクエストが静的なファイルであるかを判定する
     *
     * @return string|false
     */
    public function isStaticFile(string $sitePath, string $siteName, string $uri)
    {
        if (file_exists($staticFilePath = $sitePath.'/public/'.$uri)) {
            return $staticFilePath;
        }

        return false;
    }

> [!WARNING]
> `isStaticFile`メソッドは、リクエストのURIが`/`ではなく、`serves`メソッドで`true`が返された場合のみ呼びだされます。

<a name="the-frontcontrollerpath-method"></a>
#### `frontControllerPath`メソッド

`frontControllerPath`メソッドは、アプリケーションの「フロントコントローラ」への完全修飾パスを返す必要があります。これは通常、「index.php」ファイルまたは同等のものです。

    /**
     * アプリケーションのフロントコントローラへの絶対パスの取得
     */
    public function frontControllerPath(string $sitePath, string $siteName, string $uri): string
    {
        return $sitePath.'/public/index.php';
    }

<a name="local-drivers"></a>
### ローカルドライバ

単一のアプリケーション用にカスタムValetドライバを定義する場合は、アプリケーションのルートディレクトリに`LocalValetDriver.php`ファイルを作成します。カスタムドライバは、基本の`ValetDriver`クラスを拡張するか、`LaravelValetDriver`などの既存のアプリケーション固有のドライバを拡張する場合があります。

    use Valet\Drivers\LaravelValetDriver;

    class LocalValetDriver extends LaravelValetDriver
    {
        /**
         * リクエストに対し、このドライバを動作させるかを決める
         */
        public function serves(string $sitePath, string $siteName, string $uri): bool
        {
            return true;
        }

        /**
         * アプリケーションのフロントコントローラに対する完全な解決済みパスを取得する
         */
        public function frontControllerPath(string $sitePath, string $siteName, string $uri): string
        {
            return $sitePath.'/public_html/index.php';
        }
    }

<a name="other-valet-commands"></a>
## その他のValetコマンド

<div class="overflow-auto">

| コマンド                  | 説明                                                                                                                                                        |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `valet list`              | 全Valetコマンドの一覧を表示します。                                                                                                                         |
| `valet diagnose`          | Valetのデバッグを支援するための診断結果を出力します。                                                                                                       |
| `valet directory-listing` | ディレクトリ一覧の動作を決定します。デフォルトは "off "で、ディレクトリに対して404ページを表示します。                                                      |
| `valet forget`            | "park"された（サイト検索の親ディレクトリとして登録された）ディレクトリでこのコマンドを実行し、サイト検索対象のディレクトリリストから外します。              |
| `valet log`               | Valetサービスにより書き込まれたログリストを表示します。                                                                                                     |
| `valet paths`             | "park"されたすべてのパスを表示します。                                                                                                                      |
| `valet restart`           | Valetデーモンをリスタートします。                                                                                                                           |
| `valet start`             | Valetデーモンをスタートします。                                                                                                                             |
| `valet stop`              | Valetデーモンを停止します。                                                                                                                                 |
| `valet trust`             | Valetコマンド実行でパスワード入力をしなくて済むように、BrewとValetへsudoersファイルを追加します。                                                           |
| `valet uninstall`         | Valetをアンインストールします。手作業で削除する場合のインストラクションを表示します。`--force`パラメータを指定した場合は、Valetすべてを強制的に削除します。 |

</div>

<a name="valet-directories-and-files"></a>
## Valetのディレクトリとファイル

Valet環境の問題のトラブルシューティングを行う際に、次のディレクトリとファイルの情報が役立つでしょう。

#### `~/.config/valet`

Valetのすべての設定が含まれます。定期的にこのディレクトリをバックアップすることを推奨します。

#### `~/.config/valet/dnsmasq.d/`

このディレクトリは、DNSMasq設定を保存しています。

#### `~/.config/valet/Drivers/`

このディレクトリは、Valetのドライバを保存しています。ドライバは、特定のフレームワーク／CMSをどのように提供するかを決めています。

#### `~/.config/valet/Nginx/`

このディレクトリは、ValetのNginxサイト設定をすべて保存しています。ディレクトリ内のファイルは、`install`、`secure`コマンドを実行すると再構築されます。

#### `~/.config/valet/Sites/`

このディレクトリには、[リンクしたプロジェクト](#the-link-command)のすべてのシンボリックリンクが保存されます。

#### `~/.config/valet/config.json`

このファイルは、Valetのマスター設定ファイルです。

#### `~/.config/valet/valet.sock`

このファイルは、ValetのNginxインストールが使用するPHP-FPMソケットです。これは、PHPが正しく実行されている場合にのみ存在します。

#### `~/.config/valet/Log/fpm-php.www.log`

このファイルは、PHPエラーのユーザーログです。

#### `~/.config/valet/Log/nginx-error.log`

このファイルは、Nginxエラーのユーザーログです。

#### `/usr/local/var/log/php-fpm.log`

このファイルは、PHP-FPMエラーのシステムログです。

#### `/usr/local/var/log/nginx`

このディレクトリは、Nginxアクセスログとエラーログを保存します。

#### `/usr/local/etc/php/X.X/conf.d`

このディレクトリには、さまざまなPHP設定設定用の`*.ini`ファイルが含まれています。

#### `/usr/local/etc/php/X.X/php-fpm.d/valet-fpm.conf`

このファイルは、PHP-FPMプール設定ファイルです。

#### `~/.composer/vendor/laravel/valet/cli/stubs/secure.valet.conf`

このファイルは、サイトのSSL証明書を構築するために使用するデフォルトのNginx設定です。

<a name="disk-access"></a>
### ディスクアクセス

macOS 10.14以降、[一部のファイルやディレクトリへのアクセスがデフォルトで制限されます](https://manuals.info.apple.com/MANUALS/1000/MA1902/en_US/apple-platform-security-guide.pdf)。この制限には、デスクトップ、ドキュメント、およびダウンロードの各ディレクトリが含まれます。さらに、ネットワークボリュームとリムーバブルボリュームへのアクセスも制限されています。したがって、Valetでは、サイトフォルダをこれらの保護された場所の外に配置することを推奨します。

しかし、これらのロケーションの中からサイトを提供したい場合、Nginxに「フルディスクアクセス」を与える必要があります。そうしないと、特に静的アセットを提供する際に、Nginxからサーバエラーやその他の予測不可能な動作が発生する可能性があります。通常macOSは自動的に、Nginxへこれらの場所へのフルアクセスを許可するよう促します。もしくは、`システム環境設定` > `セキュリティとプライバシー` > `プライバシー`から、`ディスクへのフルアクセス`を選択することもできます。次に、メインウィンドウにある `nginx` エントリを有効にします。
