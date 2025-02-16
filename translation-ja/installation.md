# インストール

- [Laravelとの出会い](#meet-laravel)
    - [なぜLaravelなのか？](#why-laravel)
- [Laravelアプリの生成](#creating-a-laravel-project)
    - [PHPとLaravelインストーラのインストール](#installing-php)
    - [アプリケーションの生成](#creating-an-application)
- [初期設定](#initial-configuration)
    - [環境ベースの設定](#environment-based-configuration)
    - [データベースとマイグレーション](#databases-and-migrations)
    - [ディレクトリ設定](#directory-configuration)
- [Herd使用のローカルインストール](#local-installation-using-herd)
    - [macOSでのHerd](#herd-on-macos)
    - [WindowsでのHerd](#herd-on-windows)
- [Sailで使用するDockerのインストール](#docker-installation-using-sail)
    - [macOSでのSail](#sail-on-macos)
    - [WindowsでのSail](#sail-on-windows)
    - [LinuxでのSail](#sail-on-linux)
    - [Sailサービスの選択](#choosing-your-sail-services)
- [IDEサポート](#ide-support)
- [次のステップ](#next-steps)
    - [Laravelフルスタックフレームワーク](#laravel-the-fullstack-framework)
    - [Laravel APIバックエンド](#laravel-the-api-backend)

<a name="meet-laravel"></a>
## Laravelとの出会い

Laravelは、表現力豊かでエレガントな構文を備えたWebアプリケーションフレームワークです。Webフレームワークは、アプリケーションを作成するための構造と開始点を提供します。これにより、細部に気を配りながら、すばらしいものの作成に集中できます。

Laravelは、すばらしい開発者エクスペリエンスの提供に努めています。同時に完全な依存注入、表現力豊かなデータベース抽象化レイヤー、キューとジョブのスケジュール、ユニットと統合テストなど、強力な機能もLaravelは提供しています。

PHP Webフレームワークをはじめて使用する場合も、長年の経験を持っている場合でも、Laravelは一緒に成長できるフレームワークです。私たちは皆さんがWeb開発者として最初の一歩を踏み出すのを支援したり、専門知識を次のレベルに引き上げる後押しをしたりしています。あなたが何を作り上げるのか楽しみにしています。

> [!NOTE]
> Laravelは初めてですか？[Laravel Bootcamp](https://bootcamp.laravel.com)では、Laravelのフレームワークを実際に体験しながら、Laravelアプリケーションを構築できます。

<a name="why-laravel"></a>
### なぜLaravelなのか？

Webアプリケーションを構築するときに利用できるさまざまなツールとフレームワークがあります。そうした状況でも、Laravelは最新のフルスタックWebアプリケーションを構築するために最良の選択であると私たちは信じています。

#### 前進するフレームワーク

私たちはLaravelを「進歩的な」フレームワークと呼んでいます。つまり、Laravelはあなたと一緒に成長するという意味です。もしあなたが、Web開発の最初の一歩を踏み出したばかりの方であれば、Laravelの膨大なドキュメント、ガイド、および[ビデオチュートリアル](https://laracasts.com)のライブラリが、圧倒されず骨子を学ぶのに役立つでしょう。

開発の上級者でしたら、Laravelの[依存注入](/docs/{{version}}/container)、[単体テスト](/docs/{{version}}/testing)、[キュー](/docs/{{version}}/queues)、[リアルタイムイベント](/docs/{{version}}/broadcasting)など堅牢なツールが役立つでしょう。Laravelは、プロフェッショナルなWebアプリケーションを構築するため調整してあり、エンタープライズにおける作業負荷を処理する準備ができています。

#### スケーラブルなフレームワーク

Laravelは素晴らしくスケーラブルです。PHPのスケーリングに適した基本の性質と、Redisなど高速な分散キャッシュシステムに対するLaravelの組み込み済みサポートにより、Laravelを使用した水平スケーリングは簡単です。実際、Laravelアプリケーションは、月あたり数億のリクエストを処理するよう簡単に拡張できます。

極端なスケーリングが必要ですか？ [Laravel Vapor](https://vapor.laravel.com)のようなプラットフォームを使用すると、AWSの最新のサーバレステクノロジーでほぼ無制限の規模でLaravelアプリケーションを実行できます。

#### コミュニティによるフレームワーク

LaravelはPHPエコシステムで最高のパッケージを組み合わせ、もっとも堅牢で開発者に優しいフレームワークとして使用できるように提供しています。さらに、世界中の何千人もの才能ある開発者が[フレームワークに貢献](https://github.com/laravel/framework)しています。多分あなたもLaravelの貢献者になるかもしれませんね。

<a name="creating-a-laravel-project"></a>
## Laravelアプリの生成

<a name="installing-php"></a>
### PHPとLaravelインストーラのインストール

最初のLaravelアプリケーションを作成する前に、ローカルマシンに[PHP](https://php.net)、[Composer](https://getcomposer.org)、[Laravelインストーラ](https://github.com/laravel/installer)がインストール済みであることを確認してください。さらに、アプリケーションのフロントエンドリソースをコンパイルできるように、[NodeとNPM](https://nodejs.org)か[Bun](https://bun.sh/)もインストールしてください。

ローカルマシンにPHPとComposerをインストールしていない場合は、以下のコマンドでPHP、Composer、LaravelインストーラをmacOS、Windows、Linuxへインストールできます。

```shell tab=macOS
/bin/bash -c "$(curl -fsSL https://php.new/install/mac/8.4)"
```

```shell tab=Windows PowerShell
# 管理者として実行する
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://php.new/install/windows/8.4'))
```

```shell tab=Linux
/bin/bash -c "$(curl -fsSL https://php.new/install/linux/8.4)"
```

上記のコマンドを実行した後、ターミナルセッションを再起動してください。PHP、Composer、Laravelインストーラをインストールした後に、`php.new`によりアップデートするには、ターミナルでコマンドを再実行してください。

すでにPHPとComposerがインストール済みの場合は、Composer経由でLaravelインストーラをインストールしてください。

```shell
composer global require laravel/installer
```

> [!NOTE]
> 全機能を備えたグラフィカルなPHPのインストールと管理については、[Laravel Herd](#local-installation-using-herd)をチェックしてください。

<a name="creating-an-application"></a>
### アプリケーションの生成

PHP、Composer、Laravelインストーラをインストールしたら、新しいLaravelアプリケーションを作成する準備ができました。Laravelインストーラは、希望するテストフレームワーク、データベース、スターターキットを選択するように促します。

```nothing
laravel new example-app
```

アプリケーションを作成したら、`dev` Composerスクリプトを使って、Laravelのローカル開発サーバ、キューワーカ、Vite開発サーバを起動できます。

```nothing
cd example-app
npm install && npm run build
composer run dev
```

開発サーバを起動すると、アプリケーションはウェブブラウザで[http://localhost:8000](http://localhost:8000)からアクセスできるようになります。これで、[Laravelエコシステムへの次のステップを開始する](#next-steps)準備が整いました。もちろん、[データベースを設定する](#databases-and-migrations)ことも必要でしょう。

> [!NOTE]
> Laravelアプリケーションを開発する際に、有利なスタートダッシュを切りたければ、[スターターキット](/docs/{{version}}/starter-kits)の１つを使用することを検討してください。Laravelのスターターキットは、新しいLaravelアプリケーションのために、バックエンドとフロントエンド側の認証のスカフォールドを提供します。

<a name="initial-configuration"></a>
## 初期設定

Laravelフレームワークのすべての設定ファイルは、`config`ディレクトリへ格納しています。各オプションはコメントによりドキュメント化されていますので、自由にファイルに目を通して、利用可能なオプションに慣れてください。

Laravelは初期設定で動き、追加の設定はほぼ必要ありません。すぐに開発を始めることができます！しかし、`config/app.php`ファイルとそのコメントの確認をお勧めします。このファイルは`url`や`locale`など、アプリケーションに応じて変更したいであろうオプションを含んでいます。。

<a name="environment-based-configuration"></a>
### 環境ベースの設定

アプリケーションをローカルマシンで実行するか、本番のWebサーバで実行するかにより、Laravelの設定オプション値の多くは異なる可能性があるため、多くの重要な設定値は、アプリケーションのルートに存在する`.env`ファイルを使用して定義します。

`.env`ファイルはアプリケーションのソース管理下へコミットすべきではありません。なぜなら、アプリケーションを使用する開発者やサーバごとに、異なる環境設定が必要になる可能性があるからです。さらに、侵入者がソース管理リポジトリにアクセスした場合、機密情報が公開されてしまうため、セキュリティ上のリスクとなります。

> [!NOTE]
> `.env`ファイルと環境ベースによる設定の詳細は、完全な[設定のドキュメント](/docs/{{version}}/configuration#environment-configuration)をチェックしてください。

<a name="databases-and-migrations"></a>
### データベースとマイグレーション

Laravelアプリケーションを作成したら、おそらくデータベースにデータを保存したいと思うでしょう。アプリケーションの`.env`設定ファイルはデフォルトで、LaravelがSQLiteデータベースとやり取りする指定をしています。

アプリケーション作成時に、Laravelはあなたのために`database/database.sqlite`ファイルを作成し、アプリケーションのデータベーステーブルを作成するために必要なマイグレーションを実行しました。

MySQLやPostgreSQLなど別のデータベースドライバを使用したい場合は、`.env`設定ファイルを更新して、適切なデータベースを使用できるようにしてください。例えば、MySQLを使いたい場合は、`.env`設定ファイルの`DB_*`変数を以下のように更新します。

```ini
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```

SQLite以外のデータベースを使用する場合は、データベースを作成し、アプリケーションの[データベースマイグレーション](/docs/{{version}}/migrations)を実行する必要があります。

```shell
php artisan migrate
```

> [!NOTE]
> macOSやWindowsで開発していて、MySQL、PostgreSQL、Redisをローカルにインストールする必要がある場合は、[Herd Pro](https://herd.laravel.com/#plans)の使用を検討してください。

<a name="directory-configuration"></a>
### ディレクトリ設定

Laravelは常に、Webサーバで設定した「Webディレクトリ」のルートから提供されるべきです。「Webディレクトリ」のサブディレクトリからLaravelアプリケーションを提供しようとしないでください。そうすると、アプリケーション内に存在する機密ファイルが公開されてしまう可能性があります。

<a name="local-installation-using-herd"></a>
## Herd使用のローカルインストール

[Laravel Herd](https://herd.laravel.com)は、macOSとWindowsのための、超高速でネイティブなLaravelとPHPの開発環境です。Herdには、PHPやNginxなど、Laravel開発を始めるために必要なものがすべて含まれています。

Herdをインストールすれば、Laravelを使う開発を始める準備ができます。Herdは、`php`、`composer`、`laravel`、`expose`、`node`、`npm`、`nvm`用のコマンドラインツールを用意してあります。

> [!NOTE]
> [Herd Pro](https://herd.laravel.com/#plans)は、ローカルのMySQL、Postgres、Redisデータベースを作成・管理する機能や、ローカルのメール閲覧、ログ監視など、Herdをさらに強力な機能で強化します。

<a name="herd-on-macos"></a>
### macOSでのHerd

macOSで開発する場合は、[Herd website](https://herd.laravel.com)からHerdインストーラをダウンロードできます。このインストーラは自動的に最新バージョンのPHPをダウンロードし、バックグラウンドで常に[Nginx](https://www.nginx.com/)を実行するようにMacを設定します。

macOS向けHerdは、[dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq)を使用し、「parked」ディレクトリをサポートします。パークしたディレクトリにあるLaravelアプリケーションは、自動的にHerdが提供します。デフォルトでHerdは、`~/Herd`にパークするディレクトリを作成し、そのディレクトリ名を使用して`.test`ドメイン上のこのディレクトリ内の任意のLaravelアプリケーションへアクセスできます。

Herdのインストール後、最も速く新しいLaravelアプリケーションを作成する方法は、Herdにバンドル済みのLaravel CLIを使用します。

```nothing
cd ~/Herd
laravel new my-app
cd my-app
herd open
```

もちろん、システムトレイにあるHerdメニューから開くことができるHerdのUIで、いつでもパークしたディレクトリやその他のPHP設定を管理することができます。

Herdについての詳細は、[Herd documentation](https://herd.laravel.com/docs)をご覧ください。

<a name="herd-on-windows"></a>
### WindowsでのHerd

[Herd website](https://herd.laravel.com/windows)から、Windowsインストーラをダウンロードできます。インストールし終えたら、Herdを起動してインストール処理を完了し、Herd UIにアクセスできます。

Herdのシステムトレイアイコンを左クリックで、Herd UIへアクセスできます。右クリックするとクイックメニューが開き、日常的に必要なすべてのツールにアクセスできます。

インストール中、Herdはホームディレクトリの`%USERPROFILE%\Herd`に、「parked」ディレクトリを作成します。パークしたディレクトリにあるLaravelアプリケーションは、自動的にHerdが提供しますので、ディレクトリ名`.test`ドメインで、Laravelアプリケーションへアクセスできます。

Herdインストール後、最も速く新しいLaravelアプリケーションを作成する方法は、Herdにバンドル済みのLaravel CLIを使用することです。開始するには、Powershellを開き、以下のコマンドを実行します：

```nothing
cd ~\Herd
laravel new my-app
cd my-app
herd open
```

Herdの詳細は、[Windows向けHerdドキュメント](https://herd.laravel.com/docs/windows)をご覧ください。

<a name="docker-installation-using-sail"></a>
## Sailで使用するDockerのインストール

皆さんの好みのオペレーティングシステムが何であれ、できるだけ簡単にLaravelを始められるようにしたいと考えています。そのため、ローカルマシンでLaravelアプリケーションを開発・実行するための様々なオプションが用意されています。これらのオプションは後ほど検討していただけますが、Laravelでは[Sail](/docs/{{version}}/sail)という、[Docker](https://www.docker.com)を使用してLaravelアプリケーションを実行する組み込みソリューションを提供しています。

Dockerは、ローカルマシンにインストールしているソフトウェアや構成に干渉しない、小型で軽量の「コンテナ」でアプリケーションとサービスを実行するためのツールです。これはつまり、パーソナルマシン上のWebサーバやデータベースなどの複雑な開発ツールの構成や準備について心配する必要はないことを意味します。開発を開始するには、[Docker Desktop](https://www.docker.com/products/docker-desktop)をインストールするだけです。

Laravel Sailは、LaravelのデフォルトのDocker構成と、操作するための軽量のコマンドラインインターフェイスです。 Sailは、Dockerの経験がなくても、PHP、MySQL、Redisを使用してLaravelアプリケーションを構築するために良い出発点を提供しています。

> [!NOTE]
> すでにDockerのエキスパートですか？ご心配なく！Laravelが提供する`docker-compose.yml`ファイルを使用して、Sailに関するすべてをカスタマイズできます。

<a name="sail-on-macos"></a>
### macOSでのSail

Macで開発していて、[Docker Desktop](https://www.docker.com/products/docker-desktop)がすでにインストールされているならば、簡単なターミナルコマンドを使用して新しいLaravelアプリケーションを作成できます。たとえば、「example-app」という名前のディレクトリに新しいLaravelアプリケーションを作成するには、ターミナルで以下のコマンドを実行します。

```shell
curl -s "https://laravel.build/example-app" | bash
```

もちろん、このURLの"example-app"は好きなように変更できます。ただし、アプリケーション名は英数字とハイフン、アンダーバーだけで構成してください。Laravelアプリケーションのディレクトリは、コマンドを実行したディレクトリ内に作成されます。

Sailのインストールは、Sailのアプリケーションコンテナをローカルマシン上に構築するため、数分かかる場合があります。

アプリケーションを作成したら、アプリケーションディレクトリに移動してLaravel Sailを起動してください。Laravel Sailは、LaravelのデフォルトのDocker構成を操作するためのシンプルなコマンドラインインターフェイスを提供しています。

```shell
cd example-app

./vendor/bin/sail up
```

アプリケーションのDockerコンテナが起動したら、アプリケーションの[データベースマイグレーション](/docs/{{version}}/migrations)を実行します。

```shell
./vendor/bin/sail artisan migrate
```

これで、http://localhostにより、ウェブブラウザからアプリケーションへアクセスできます。

> [!NOTE]
> Laravel Sailの詳細は、[完全なドキュメント](/docs/{{version}}/sail)で確認してください。

<a name="sail-on-windows"></a>
### WindowsでのSail

Windowsマシンに新しいLaravelアプリケーションを作成する前に、必ず[Docker Desktop](https://www.docker.com/products/docker-desktop)をインストールしてください。次に、Windows Subsystem for Linux 2（WSL2）がインストールされ、有効になっていることを確認する必要があります。 WSLを使用すると、Linuxバイナリ実行可能ファイルをWindows 10でネイティブに実行できます。WSL2をインストールして有効にする方法については、Microsoftの[開発者環境ドキュメント](https://docs.microsoft.com/en-us/windows/wsl/install-win10)を参照してください。

> [!NOTE]
> WSL2をインストールして有効にした後、Dockerデスクトップが[WSL2バックエンドを使用するように構成されている](https://docs.docker.com/docker-for-windows/wsl/)ことを確認する必要があります。

これで、最初のLaravelアプリケーションを作成する準備が整いました。[Windowsターミナル](https://www.microsoft.com/en-us/p/windows-terminal/9n0dx20hk701?rtc=1&activetab=pivot:overviewtab)を起動し、WSL2 Linuxオペレーティングシステムの新しいターミナルセッションを開始します。次に、簡単なターミナルコマンドを使用して新しいLaravelアプリケーションを作成してみましょう。たとえば、"example-app"という名前のディレクトリに新しいLaravelアプリケーションを作成するには、ターミナルで以下のコマンドを実行します。

```shell
curl -s https://laravel.build/example-app | bash
```

もちろん、このURLの"example-app"は好きなように変更できます。ただし、アプリケーション名は英数字とハイフン、アンダーバーだけで構成してください。Laravelアプリケーションのディレクトリは、コマンドを実行したディレクトリ内に作成されます。

Sailのインストールは、Sailのアプリケーションコンテナをローカルマシン上に構築するため、数分かかる場合があります。

アプリケーションを作成したら、アプリケーションディレクトリに移動してLaravel Sailを起動してください。Laravel Sailは、LaravelのデフォルトのDocker構成を操作するためのシンプルなコマンドラインインターフェイスを提供しています。

```shell
cd example-app

./vendor/bin/sail up
```

アプリケーションのDockerコンテナが起動したら、アプリケーションの[データベースマイグレーション](/docs/{{version}}/migrations)を実行します。

```shell
./vendor/bin/sail artisan migrate
```

これで、http://localhostにより、ウェブブラウザからアプリケーションへアクセスできます。

> [!NOTE]
> Laravel Sailの詳細は、[完全なドキュメント](/docs/{{version}}/sail)で確認してください。

#### WSL2内での開発

もちろん、WSL2インストール内で作成されたLaravelアプリケーションファイルを変更する必要があります。これを実現するには、Microsoftの[Visual Studio Code](https://code.visualstudio.com)エディターと[リモート開発](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)用のファーストパーティ拡張機能を使用することをお勧めします。

これらのツールをインストールしたら、Windowsターミナルを使用してアプリケーションのルートディレクトリから `code .`コマンドを実行することで、任意のLaravelアプリケーションを開けます。

<a name="sail-on-linux"></a>
### LinuxでのSail

Linuxで開発しており、[Docker Compose](https://docs.docker.com/compose/install/)を既にインストールしている場合は、簡単なターミナルコマンドで新しいLaravelアプリケーションを作成できます。

まず、Docker Desktop for Linuxを使用している場合は、以下のコマンドを実行してください。Docker Desktop for Linuxを使用していない場合は、このステップを飛ばしてください。

```shell
docker context use default
```

次に、"example-app "ディレクトリへ新しいLaravelアプリケーションを作成するために、ターミナルで以下のコマンドを実行します。

```shell
curl -s https://laravel.build/example-app | bash
```

もちろん、このURLの"example-app"は好きなように変更できます。ただし、アプリケーション名は英数字とハイフン、アンダーバーだけで構成してください。Laravelアプリケーションのディレクトリは、コマンドを実行したディレクトリ内に作成されます。

Sailのインストールは、Sailのアプリケーションコンテナをローカルマシン上に構築するため、数分かかる場合があります。

アプリケーションを作成したら、アプリケーションディレクトリに移動してLaravel Sailを起動してください。Laravel Sailは、LaravelのデフォルトのDocker構成を操作するためのシンプルなコマンドラインインターフェイスを提供しています。

```shell
cd example-app

./vendor/bin/sail up
```

アプリケーションのDockerコンテナが起動したら、アプリケーションの[データベースマイグレーション](/docs/{{version}}/migrations)を実行します。

```shell
./vendor/bin/sail artisan migrate
```

これで、http://localhostにより、ウェブブラウザからアプリケーションへアクセスできます。

> [!NOTE]
> Laravel Sailの詳細は、[完全なドキュメント](/docs/{{version}}/sail)で確認してください。

<a name="choosing-your-sail-services"></a>
### Sailサービスの選択

Sailで新しいLaravelアプリケーションを作成する際に、`with`というクエリ文字列変数を使って、新しいアプリケーションの`docker-compose.yml`ファイルで設定するサービスを選択できます。利用可能なサービスは、`mysql`、`pgsql`、`mariadb`、`redis`、`memcached`、`valkey`、`meilisearch`、`typesence`、`minio`、`selenium`、`mailpit`です。

```shell
curl -s "https://laravel.build/example-app?with=mysql,redis" | bash
```

設定したいサービスを指定しない場合は、`mysql`、`redis`、`meilisearch`、`mailpit`、`selenium`のデフォルトのスタックが設定されます。

URLへ`devcontainer`パラメータを追加し、デフォルトの[Devcontainer](/docs/{{version}}/sail#using-devcontainers)をインストールするよう、Sailに指示できます。

```shell
curl -s "https://laravel.build/example-app?with=mysql,redis&devcontainer" | bash
```

<a name="ide-support"></a>
## IDEサポート

Laravelアプリケーションを開発するときに、どのようなコードエディタを使用するかは自由ですが、[PhpStorm](https://www.jetbrains.com/phpstorm/laravel/)は、[Laravel Pint](https://www.jetbrains.com/help/phpstorm/using-laravel-pint.html)を含むLaravelとそのエコシステムを幅広くサポートしています。

さらに、コミュニティがメンテナンスしている[Laravel Idea](https://laravel-idea.com/) PhpStormプラグインは、コード生成、Eloquent構文補完、バリデーションルール補完など、IDEに役立つ様々な拡張機能を提供しています。

<a name="next-steps"></a>
## 次のステップ

Laravelアプリケーションを設定し終えて、次に何を学ぶべきか迷っているかもしれません。まず、以下のドキュメントを読み、Laravelの仕組みを理解することを強く推奨いたします。

<div class="content-list" markdown="1">

- [リクエストのライフサイクル](/docs/{{version}}/lifecycle)
- [設定](/docs/{{version}}/configuration)
- [ディレクトリ構成](/docs/{{version}}/structure)
- [フロントエンド](/docs/{{version}}/frontend)
- [サービスコンテナ](/docs/{{version}}/container)
- [ファサード](/docs/{{version}}/facades)

</div>

Laravelをどのように使用するかにより、旅の次の行き先も決まります。Laravelを使用するにはさまざまな方法があります。以下では、フレームワークの２つの主要なユースケースについて説明します。

> [!NOTE]
> Laravelは初めてですか？[Laravel Bootcamp](https://bootcamp.laravel.com)では、Laravelのフレームワークを実際に体験しながら、Laravelアプリケーションを構築できます。

<a name="laravel-the-fullstack-framework"></a>
### Laravelフルスタックフレームワーク

Laravelは、フルスタックフレームワークとして機能させることができます。「フルスタック」フレームワークとは、Laravelを使用して、アプリケーションへのリクエストをルーティングし、[Bladeテンプレート](/docs/{{version}}/blade)や[Inertia](https://inertiajs.com)などのシングルページアプリケーションハイブリッド技術でフロントエンドをレンダすることを意味します。これは、Laravelフレームワークの最も一般的な使用方法であり、私たちの意見では、Laravelを使用する最も生産的な方法です。

この方法でLaravelの使用を計画している場合は、[フロントエンド開発](/docs/{{version}}/frontend)、[ルーティング](/docs/{{version}}/routing)、[ビュー](/docs/{{version}}/views)、[Eloquent ORM](/docs/{{version}}/eloquent)についてのドキュメントをチェックすると良いかも知れません。さらに、[Livewire](https://livewire.laravel.com)や[Inertia](https://inertiajs.com)といったコミュニティパッケージについても学ぶことに興味があるかもしれません。これらのパッケージにより、Laravelをフルスタックフレームワークとして使用しながら、シングルページのJavaScriptアプリケーションが提供するUIの、利点をたくさん享受できます。

Laravelをフルスタックフレームワークとして使用している場合、[Vite](/docs/{{version}}/vite)を使用してアプリケーションのCSSとJavaScriptをコンパイルする方法を学ぶのも強く推奨します。

> [!NOTE]
> アプリケーションの構築をすぐに始めたい場合は、公式の[アプリケーションスターターキット](/docs/{{version}}/starter-kits)の１つをチェックしてください。

<a name="laravel-the-api-backend"></a>
### Laravel APIバックエンド

Laravelは、JavaScriptシングルページアプリケーションまたはモバイルアプリケーションへのAPIバックエンドとしても機能させることもあります。たとえば、[Next.js](https://nextjs.org)アプリケーションのAPIバックエンドとしてLaravelを使用できます。こうした使い方では、Laravelでアプリケーションに[認証](/docs/{{version}}/sanctum)とデータの保存/取得を提供すると同時に、キュー、メール、通知などのLaravelの強力なサービスを利用できます。

この方法でLaravelの使用を計画している場合は、[ルーティング](/docs/{{version}}/routing)、[Laravel Sanctum](/docs/{{version}}/sanctum)、[Eloquent ORM](/docs/{{version}}/eloquent)に関するドキュメントを確認することをお勧めします。

> [!NOTE]
> LaravelのバックエンドとNext.jsのフロントエンドのスカフォールドから始める必要がありますか？Laravel Breezeは、[APIスタック](/docs/{{version}}/starter-kits#breeze-and-next)と[Next.jsフロントエンド実装](https://github.com/laravel/breeze-next)を提供しているため、すぐに開始できます。
