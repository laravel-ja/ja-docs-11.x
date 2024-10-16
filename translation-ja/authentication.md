# 認証

- [イントロダクション](#introduction)
    - [スターターキット](#starter-kits)
    - [データベースの検討](#introduction-database-considerations)
    - [エコシステム概要](#ecosystem-overview)
- [認証クイックスタート](#authentication-quickstart)
    - [スターターキットのインストール](#install-a-starter-kit)
    - [認証済みユーザー取得](#retrieving-the-authenticated-user)
    - [ルートの保護](#protecting-routes)
    - [ログイン回数制限](#login-throttling)
- [ユーザーを手作業で認証する](#authenticating-users)
    - [持続ログイン](#remembering-users)
    - [他の認証方法](#other-authentication-methods)
- [HTTP基本認証](#http-basic-authentication)
    - [ステートレスなHTTP基本認証](#stateless-http-basic-authentication)
- [ログアウト](#logging-out)
    - [他のデバイスでのセッションの無効化](#invalidating-sessions-on-other-devices)
- [パスワードの確認](#password-confirmation)
    - [設定](#password-confirmation-configuration)
    - [ルーティング](#password-confirmation-routing)
    - [パスワード確認済みルートの保護](#password-confirmation-protecting-routes)
- [カスタムガードの追加](#adding-custom-guards)
    - [クロージャリクエストガード](#closure-request-guards)
- [カスタムユーザープロバイダの追加](#adding-custom-user-providers)
    - [ユーザープロバイダ契約](#the-user-provider-contract)
    - [Authenticatable契約](#the-authenticatable-contract)
- [パスワードの自動再ハッシュ](#automatic-password-rehashing)
- [ソーシャル認証](/docs/{{version}}/socialite)
- [イベント](#events)

<a name="introduction"></a>
## イントロダクション

多くのWebアプリケーションは、ユーザーがアプリケーションで認証して「ログイン」する手段を提供しています。この機能をWebアプリケーションに実装することは、複雑でリスクを伴う可能性のある作業になりえます。このためLaravelは、認証を迅速、安全、かつ簡単に実装するために必要なツールを提供するよう努めています。

Laravelの認証機能は、基本的に「ガード」と「プロバイダ」で構成されています。ガードは、リクエストごとにユーザーを認証する方法を定義します。たとえば、Laravelには、セッションストレージとクッキーを使用して状態を維持する「セッション」ガードを用意しています。

プロバイダは、永続ストレージからユーザーを取得する方法を定義します。Laravelは[Eloquent](/docs/{{version}}/eloquent)とデータベースクエリビルダを使用してユーザーを取得するためのサポートを用意しています。ただし、アプリケーションの必要性に応じて、追加のプロバイダを自由に定義できます。

アプリケーションの認証設定ファイルは`config/auth.php`にあります。このファイルは、Laravelの認証サービスの動作を微調整できるように、十分なコメントの付いたオプションがいくつか含まれています。

> [!NOTE]
> ガードとプロバイダを「役割」や「許可」と混同しないでください。権限を介したユーザーアクションの認可の詳細については、[認可](/docs/{{version}}/authorization)のドキュメントを参照してください。

<a name="starter-kits"></a>
### スターターキット

さっそく始めたいですか？新しいLaravelアプリケーションに[Laravelアプリケーションスターターキット](/docs/{{version}}/starter-kits)をインストールしてください。データベースをマイグレーションした後に、ブラウザで`/register`やアプリケーションに割り当てた他のURLへアクセスしてください。スターターキットは、認証システム全体のスカフォールドの面倒を見ます。

**最終的にLaravelアプリケーションでスターターキットを使用しないことを選択した場合でも、[Laravel Breeze](/docs/{{version}}/starter-kits#laravel-breeze)スターターキットをインストールすることは、Laravelのすべての認証機能を実際のLaravelプロジェクトに実装する方法を学ぶ素晴らしい機会になるでしょう。**　Laravel Breezeは認証コントローラ、ルート、ビューを作成するため、これらのファイル内のコードを調べて、Laravelの認証機能を実装する方法を学べます。

<a name="introduction-database-considerations"></a>
### データベースの検討

デフォルトでLaravelは`app/Models`ディレクトリに、`App\Models\User` [Eloquentモデル](/docs/{{version}}/eloquent)を用意します。このモデルは、デフォルトのEloquent認証ドライバで使用します。アプリケーションがEloquentを使用していない場合は、Laravelクエリビルダを使用する`database`認証プロバイダを使用できます。

`App\Models\User`モデルのデータベーススキーマを構築するときは、パスワード列の長さを確実に６０文字以上にしてください。もちろん、新しいLaravelアプリケーションに含まれている`users`テーブルのマイグレーションでは、あらかじめこの長さを超えるカラムが作成されます。

さらに、`users`（または同等の）テーブルに、１００文字のnullableな文字列`remember_token`カラムを確実に含めてください。このカラムは、アプリケーションへログインするときに「持続ログイン（remember me）」オプションを選択したユーザーのトークンを格納するために使用します。繰り返しますが、新しいLaravelアプリケーションに含まれているデフォルトの`users`テーブルのマイグレーションには、すでにこのカラムが含まれています。

<a name="ecosystem-overview"></a>
### エコシステム概要

Laravelは認証に関連する複数のパッケージを提供しています。先へ進む前に、Laravelの一般的な認証エコシステムを確認し、各パッケージの意図と目的について説明します。

まず、認証がどのように機能するかを確認しましょう。Webブラウザを使用する場合、ユーザーはログインフォームを介してユーザー名とパスワードを入力します。これらのログイン情報が正しい場合、アプリケーションは認証したユーザーに関する情報をユーザーの[セッション](/docs/{{version}}/session)に保存します。ブラウザに発行されたクッキーにはセッションIDが含まれているため、アプリケーションへの後続のリクエストでユーザーを正しいセッションに関連付けることができます。セッションクッキーを受信すると、アプリケーションはセッションIDに基づいてセッションデータを取得し、認証情報がセッションに保存されていることに注意して、ユーザーを「認証済み」と見なします。

リモートサービスがAPIへアクセスするために認証する必要がある場合はWebブラウザがないため、通常クッキーは認証に使用されません。代わりにリモートサービスはリクエストごとにAPIトークンをAPIに送信します。アプリケーションは有効なAPIトークンのテーブルに対して受信したトークンを検証し、そのAPIトークンに関連付けられたユーザーによって実行されたものとしてそのリクエストを「認証」します。

<a name="laravels-built-in-browser-authentication-services"></a>
#### Laravelの組み込みブラウザ認証サービス

Laravelは、通常`Auth`もしくは`Session`ファサードを介してアクセスされる組み込みの認証およびセッションサービスを持っています。これらの機能は、Webブラウザから開始された要求に対してクッキーベースの認証を提供します。これらは、ユーザーのログイン情報を確認し、ユーザーを認証できるようにするメソッドを提供しています。さらに、これらのサービスは適切な認証データをユーザーのセッションへ自動的に保存し、ユーザーのセッションクッキーを発行します。これらのサービスの使用方法に関する説明は、このドキュメントに含まれています。

**アプリケーションスターターキット**

このドキュメントで説明しているように、これらの認証サービスを手作業で操作して、アプリケーション独自の認証レイヤーを構築できます。ただし、より素早く開始できるように、認証レイヤー全体の堅牢で最新のスカフォールドを提供する[無料パッケージ](/docs/{{version}}/starter-kits)をリリースしました。こうしたパッケージは、[Laravel Breeze](/docs/{{version}}/starter-kits#laravel-breeze)、[Laravel Jetstream](/docs/{{version}}/starter-kits#laravel-jetstream)、[Laravel Fortify](/docs/{{version}}/fortify)です。

*Laravel Breeze*は、ログイン、登録、パスワードのリセット、電子メールの確認、パスワードの確認など、Laravelの全認証機能のシンプルで最小限の実装です。Laravel Breezeのビューレイヤーは、[Tailwind CSS](https://tailwindcss.com)によりスタイル設定したシンプルな[Bladeテンプレート](/docs/{{version}}/blade)で構成しています。使い始めるには、Laravelの[アプリケーションスターターキット](/docs/{{version}}/starter-kits)のドキュメントを確認してください。

*Laravel Fortify*は、Laravelのヘッドレス認証バックエンドであり、クッキーベースの認証や２要素認証、メールアドレス確認や他の機能など、このドキュメントにある多くの機能を実装しています。FortifyはLaravel Jetstreamの認証バックエンドを提供していますし、Laravelで認証する必要があるSPAの認証を提供するために、[Laravel Sanctum](/docs/{{version}}/sanctum)と組み合わせて独立して使用できます。

*[Laravel Jetstream](https://jetstream.laravel.com)（[和文](/jetstream/1.0/ja/introduction.html)）*は、Laravel Fortifyの認証サービスを利用および公開し、美しくモダンなUIに[Tailwind CSS](https://tailwindcss.com)を使用している[Livewire](https://livewire.laravel.com)や[Inertia](https://inertiajs.com)を使っている、堅牢なアプリケーションのスターターキットです。Laravel Jetstreamにはオプションサポートとして、２要素認証、チームサポート、ブラウザセッション管理、プロファイル管理、およびAPIトークン認証を提供する[Laravel Sanctum](/docs/{{version}}/sanctum)との組み込み統合が含まれています。LaravelのAPI認証手法について以降で説明します。

<a name="laravels-api-authentication-services"></a>
#### Laravel API認証サービス

Laravelは、APIトークンの管理とリクエストのAPIトークンによる認証を支援する２つのオプションパッケージ、[Passport](/docs/{{version}}/passport)と[Sanctum](/docs/{{version}}/sanctum)を提供しています。これらのライブラリとLaravelの組み込みのクッキーベースの認証ライブラリは、相互に排他的ではないことに注意してください。これらのライブラリは主にAPIトークン認証に重点を置いていますが、組み込みの認証サービスはクッキーベースのブラウザ認証に重点を置いています。多くのアプリケーションは、Laravelの組み込みクッキーベースの認証サービスとLaravelのAPI認証パッケージの両方を使用します。

**Passport**

PassportはOAuth2認証プロバイダであり、さまざまなタイプのトークンを発行できる多くのOAuth2「許可タイプ」を提供します。一般にAPI認証は、堅牢で複雑なパッケージです。しかし、ほとんどのアプリケーションは、OAuth2仕様によって提供される複雑な機能を必要としないため、ユーザーと開発者の両者を混乱させる可能性があります。さらに、開発者は、PassportなどのOAuth2認証プロバイダを使用してSPAアプリケーションまたはモバイルアプリケーションを認証する方法で、長い間混乱してきました。

**Sanctum**

OAuth2の複雑さと開発者の混乱に対応するため、ウェブブラウザからのファーストパーティのウェブリクエストとトークンによるAPIリクエストの両方を処理できる、よりシンプルで合理化された認証パッケージの構築に着手しました。この目標は、[Laravel Sanctum](/docs/{{version}}/sanctum)のリリースにより実現しました。これは、APIに加えてファーストパーティのWeb UIを提供するアプリケーションや、バックエンドLaravelアプリケーションとは別に存在するシングルページアプリケーション（SPA）や、モバイルクライアントを提供するアプリケーションに推奨される認証パッケージと考えてください。

Laravel Sanctumは、アプリケーションの認証プロセス全体を管理できるWeb／APIハイブリッド認証パッケージです。これが可能なのは、Sanctumベースのアプリケーションがリクエストを受信すると、Sanctumは最初に認証済みセッションを参照するセッションクッキーがリクエストに含まれているかどうかを判断するからです。Sanctumは、すでに説明したLaravelの組み込み認証サービスを呼び出すことでこれを実現します。リクエストがセッションクッキーを介して認証されていない場合、SanctumはリクエストのAPIトークンを検査します。APIトークンが存在する場合、Sanctumはそのトークンを使用してリクエストを認証します。このプロセスの詳細については、Sanctumの「[仕組み](/docs/{{version}}/sanctum#how-it-works)」のドキュメントを参照してください。

Laravel Sanctumは、[Laravel Jetstream](https://jetstream.laravel.com)（[和文](/jetstream/1.0/ja/introduction.html)）アプリケーションスターターキットに含めることを私たちが選択したAPIパッケージです。なぜなら、これはWebアプリケーションの認証ニーズの大部分に最適であると考えているためです。

<a name="summary-choosing-your-stack"></a>
#### 要約とスタックの選択

要約すると、アプリケーションがブラウザを使用してアクセスされる、モノリシックなLaravelアプリケーションを構築している場合は、アプリケーションにLaravelの組み込み認証サービスを使用してください。

次に、アプリケーションがサードパーティによって使用されるAPIを提供している場合は、[Passport](/docs/{{version}}/passport)または[Sanctum](/docs/{{version}}/sanctum)のどちらかを選んでください。アプリケーションにAPIトークン認証を提供します。一般に、Sanctumは、API認証、SPA認証、さらに「スコープ(socpe)」や「能力(ability)」のサポートを含むモバイル認証のためのシンプルで完全なソリューションであるため、使える場合はこちらを推奨します。

Laravelをバックエンドで利用するシングルページアプリケーション（SPA）を構築している場合は、[Laravel Sanctum](/docs/{{version}}/sanctum)を使用するべきでしょう。Sanctumを使用する場合は、[独自のバックエンド認証ルートを手作業で実装する](#authenticating-users)か、[Laravel Fortify](/docs/{{version}}/fortify)をユーザー登録、パスワードのリセット、メールの検証などの機能のためのルートとコントローラを提供するヘッドレス認証バックエンドサービスとして利用する必要があります。

パスポートは、アプリケーションがOAuth2仕様によって提供されるすべての機能を絶対に必要とする場合に選択できます。

また、すぐに始めたい場合はLaravelの人気のある認証スタックと組み込みの認証サービス、LaravelSanctumをあらかじめ使用し、新しいLaravelアプリケーションをすばやく開始する方法として、[Laravel Breeze](/docs/{{version}}/starter-kits#laravel-breeze)をおすすめします。

<a name="authentication-quickstart"></a>
## 認証クイックスタート

> [!WARNING]
> ドキュメントのこの部分では、[Laravelアプリケーションスターターキット](/docs/{{version}}/starter-kits)を介したユーザーの認証について説明します。これには、すばやく開始するのに便利なＵＩスカフォールドが含まれています。Laravelの認証システムを直接統合したい場合は、[ユーザーの手作業認証](#authenticating-users)に関するドキュメントを確認してください。

<a name="install-a-starter-kit"></a>
### スターターキットのインストール

まず、[Laravelアプリケーションスターターキットをインストールする](/docs/{{version}}/starter-kits)必要があります。現在のスターターキットであるLaravel BreezeとLaravel Jetstreamは、認証を新しいLaravelアプリケーションへ組み込むために美しく設計された開始点を提供します。

Laravel Breezeは、ログイン、ユーザー登録、パスワードリセット、メールの確認、パスワードの確認など、Laravelのすべての認証機能の最小限でシンプルな実装です。Laravel Breezeのビューレイヤーは、[Tailwind CSS](https://tailwindcss.com)を用いスタイル設定されたシンプルな[Bladeテンプレート](/docs/{{version}}/blade)で構成しています。さらに、Breezeは[Livewire](https://livewire.laravel.com)または[Inertia](https://inertiajs.com)に基づくスカフォールドオプションがあり、InertiaベースのスカフォールドにはVueまたはReactを使用できます。

[Laravel Jetstream](https://jetstream.laravel.com)は、[Livewire](https://livewire.laravel.com)か[InertiaとVue](https://inertiajs.com)を使用する、アプリケーションのスカフォールドのサポートを含む、より堅牢なアプリケーションスターターキットです。さらに、Jetstreamはオプションとして２要素認証、チーム、プロファイル管理、ブラウザセッション管理、[Laravel Sanctum](/docs/{{version}}/sanctum)を介するAPIサポート、アカウント削除などのサポートを備えています。

<a name="retrieving-the-authenticated-user"></a>
### 認証済みユーザー取得

認証スターターキットをインストールし、ユーザーがアプリケーションへ登録・認証できるようにした後、多くの場合は現在認証済みのユーザーを操作する必要があります。受信リクエストの処理時、`Auth`ファサードの`user`メソッドを使用して認証済みユーザーにアクセスできます。

    use Illuminate\Support\Facades\Auth;

    // 現在認証しているユーザーを取得
    $user = Auth::user();

    // 現在認証しているユーザーのIDを取得
    $id = Auth::id();

別の方法として、ユーザーが一度認証されると、`Illuminate\Http\Request`インスタンスを介してその認証済みユーザーへアクセスできます。タイプヒントクラスは、コントローラメソッドに自動的に依存注入されることを忘れないでください。`Illuminate\Http\Request`オブジェクトをタイプヒントすることで、リクエストの`user`メソッドを介しアプリケーションのどのコントローラメソッドからも、認証済みユーザーへ便利にアクセスできます。

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;

    class FlightController extends Controller
    {
        /**
         * 既存のフライトの情報を更新
         */
        public function update(Request $request): RedirectResponse
        {
            $user = $request->user();

            // ...

            return redirect('/flights');
        }
    }

<a name="determining-if-the-current-user-is-authenticated"></a>
#### 現在のユーザーが認証済みか判定

受信HTTPリクエストを送ったユーザーが認証されているかを判断するには、`Auth`ファサードで`check`メソッドを使用します。ユーザーが認証されている場合、このメソッドは`true`を返します。

    use Illuminate\Support\Facades\Auth;

    if (Auth::check()) {
        // ユーザーはログイン済み
    }

> [!NOTE]
> `check`メソッドを使用してユーザーが認証されているかどうかを判定できますが、通常はミドルウェアを使用して、ユーザーに特定のルート/コントローラへのアクセスを許す前に、ユーザーが認証されていることを確認します。これについて詳しくは、[ルートの保護](/docs/{{version}}/authentication#protecting-routes)に関するドキュメントをご覧ください。

<a name="protecting-routes"></a>
### ルートの保護

[ルートミドルウェア](/docs/{{version}}/middleware)を使うと、認証済みユーザーだけが指定したルートへアクセスできるようにすることができます。Laravelには`auth`ミドルウェアが同梱されていますが、これは`Illuminate\Auth\Middleware\Authenticate`クラスの [ミドルウェアエイリアス](/docs/{{version}}/middleware#middleware-aliases)です。このミドルウェアはあらかじめLaravel内部でエイリアスしているので、必要なのはルート定義へこのミドルウェアを指定するだけです。

    Route::get('/flights', function () {
        // 認証済みユーザーのみがこのルートにアクセス可能
    })->middleware('auth');

<a name="redirecting-unauthenticated-users"></a>
#### 認証されていないユーザーのリダイレクト

`auth`ミドルウェアが未認証のユーザーを検出すると、そのユーザーを`login`[名前付きルート](/docs/{{version}}/routing#named-routes)へリダイレクトします。アプリケーションの`bootstrap/app.php`ファイル中の、`redirectGuestsTo`メソッドを使用して、この動作を変更できます。

    use Illuminate\Http\Request;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->redirectGuestsTo('/login');

        // クロージャ使用の場合
        $middleware->redirectGuestsTo(fn (Request $request) => route('login'));
    })

<a name="specifying-a-guard"></a>
#### ガードの指定

`auth`ミドルウェアをルートへ接続するときに、ユーザーの認証に使用する「ガード」を指定することもできます。指定するガードは、`auth.php`設定ファイルの`guards`配列内のキーの1つに対応している必要があります。

    Route::get('/flights', function () {
        // 認証済みユーザーのみがこのルートにアクセス可能
    })->middleware('auth:admin');

<a name="login-throttling"></a>
### ログイン回数制限

Laravel BreezeまたはLaravel Jetstream [スターターキット](/docs/{{version}}/starter-kits)を使用している場合、ログインの試行へレート制限が自動的に適用されます。デフォルトでは、ユーザーが数回試行した後に正しい資格情報を提供できなかった場合、ユーザーは１分間ログインできません。ログインの回数制限は、ユーザーのユーザー名／メールアドレスとIPアドレスの組み合わせごとに制限します。

> [!NOTE]
> アプリケーション内の他のルートのレート制限を希望する場合は、[レート制限のドキュメント](/docs/{{version}}/routing#rate-limiting)を確認してください。

<a name="authenticating-users"></a>
## ユーザーを手作業で認証する

Laravelの[アプリケーションスターターキット](/docs/{{version}}/starter-kits)に含まれている認証スカフォールドを必ず使用する必要はありません。このスカフォールドを使用しないことを選択した場合は、Laravel認証クラスを直接使用してユーザー認証を管理する必要があります。心配ありません。簡単です！

`Auth` [ファサード](/docs/{{version}}/facades)を介してLaravelの認証サービスにアクセスできるように、クラスの最初で`Auth`ファサードを必ずインポートする必要があります。次に、`attempt`メソッドを見てみましょう。`attempt`メソッドは通常、アプリケーションの「ログイン」フォームからの認証試行を処理するために使用されます。認証が成功した場合は、ユーザーの[セッション](/docs/{{version}}/session)を再生成して、[セッション固定攻撃](https://en.wikipedia.org/wiki/Session_fixation)を防ぐ必要があります。

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\Request;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Support\Facades\Auth;

    class LoginController extends Controller
    {
        /**
         * 認証の試行を処理
         */
        public function authenticate(Request $request): RedirectResponse
        {
            $credentials = $request->validate([
                'email' => ['required', 'email'],
                'password' => ['required'],
            ]);

            if (Auth::attempt($credentials)) {
                $request->session()->regenerate();

                return redirect()->intended('dashboard');
            }

            return back()->withErrors([
                'email' => 'The provided credentials do not match our records.',
            ])->onlyInput('email');
        }
    }

`attempt`メソッドは、最初の引数にキー／値ペアの配列を受け入れます。配列の値はデータベーステーブルでユーザーを見つけるために使用します。したがって、上例でユーザーは、`email`カラムの値により取得されます。ユーザーが見つかった場合、データベースに保存しているハッシュ化したパスワードと、配列でメソッドに渡した`password`値を比較します。フレームワークはデータベース内のハッシュ化したパスワードと比較する前に値を自動的にハッシュするため、受信リクエストの`password`値はハッシュしないでください。２つのハッシュ化されたパスワードが一致した場合、ユーザーの認証セッションを開始します。

Laravelの認証サービスは、認証ガードの「プロバイダ」設定に基づいてデータベースからユーザーを取得することを忘れないでください。デフォルトの `config/auth.php`設定ファイルでは、Eloquentユーザープロバイダが指定されており、ユーザーを取得するときに`App\Models\User`モデルを使用するように指示されています。アプリケーションの必要に基づいて、設定ファイル内のこうした値を変更できます。

認証が成功した場合、`attempt`メソッドは`true`を返します。それ以外の場合は、`false`が返します。

Laravelのリダイレクタが提供する`intended`メソッドは、認証ミドルウェアによってインターセプトされる前に、アクセスしようとしたURLへユーザーをリダイレクトします。目的の行き先が使用できない場合のために、このメソッドにはフォールバックURIが指定できます。

<a name="specifying-additional-conditions"></a>
#### 追加条件の指定

必要に応じて、ユーザーのメールアドレスとパスワードに加え、認証クエリにクエリ条件を追加することもできます。これのためには、`attempt`メソッドへ渡す配列にクエリ条件を追加するだけです。たとえば、ユーザーが「アクティブ（active）」としてマークされていることを確認できます。

    if (Auth::attempt(['email' => $email, 'password' => $password, 'active' => 1])) {
        // 認証に成功した
    }

クエリ条件が複雑な場合は、配列へクロージャを渡します。このクロージャはアプリケーションの要件に基づき、クエリをカスタマイズ出来るようにクエリのインスタンスと一緒に起動されます。

    use Illuminate\Database\Eloquent\Builder;

    if (Auth::attempt([
        'email' => $email,
        'password' => $password,
        fn (Builder $query) => $query->has('activeSubscription'),
    ])) {
        // 認証に成功した
    }

> [!WARNING]
> これらの例では、`email`は必須オプションではなく、単に例として使用しています。データベーステーブルの"username"に対応するカラム名を使用する必要があります。

クロージャを第２引数として受ける、`attemptWhen`メソッドは、ユーザーを実際に認証する前に、潜在的なユーザーに対してより広範な審査を行うために使用します。クロージャは潜在的なユーザーを引数に取り、そのユーザーを認証するかを示すため、`true`または`false`を返さなければなりません。

    if (Auth::attemptWhen([
        'email' => $email,
        'password' => $password,
    ], function (User $user) {
        return $user->isNotBanned();
    })) {
        // 認証に成功した
    }

<a name="accessing-specific-guard-instances"></a>
#### 特定のガードインスタンスへのアクセス

`Auth`ファサードの`guard`メソッドを介して、ユーザーを認証するときに利用するガードインスタンスを指定できます。これにより、完全に分離した認証可能なモデルやユーザーテーブルを使用して、アプリケーションの個別の部分での認証を管理できます。

`guard`メソッドへ渡たすガード名は、`auth.php`設定ファイルで設定するガードの１つに対応している必要があります。

    if (Auth::guard('admin')->attempt($credentials)) {
        // ...
    }

<a name="remembering-users"></a>
### 持続ログイン

多くのWebアプリケーションでは、ログインフォームに「継続ログイン（remember me）」チェックボックスがあります。アプリケーションで「継続ログイン」機能を提供したい場合は、`attempt`メソッドの2番目の引数としてブール値を渡すことができます。

この値が`true`の場合、Laravelはユーザーを無期限に、または手作業でログアウトするまで認証されたままにします。 `users`テーブルには、「継続ログイン（remember me）」トークンを格納するために使用される文字列、`remember_token`カラムを含める必要があります。新しいLaravelアプリケーションに含まれている `users`テーブルのマイグレーションには、すでにこのカラムが含まれています。

    use Illuminate\Support\Facades\Auth;

    if (Auth::attempt(['email' => $email, 'password' => $password], $remember)) {
        // $rememberがtrueならユーザーは持続ログインされる
    }

アプリケーションで、"remember me"機能を提供している場合は、`viaRemember`メソッドを使用して、現在認証されているユーザーが"remember me"クッキーで認証されているかを判断できます。

    use Illuminate\Support\Facades\Auth;

    if (Auth::viaRemember()) {
        // ...
    }

<a name="other-authentication-methods"></a>
### 他の認証方法

<a name="authenticate-a-user-instance"></a>
#### ユーザーインスタンスの認証

既存のユーザーインスタンスを現在認証されているユーザーとして設定する必要がある場合は、ユーザーインスタンスを`Auth`ファサードの`login`メソッドに渡すことができます。指定されたユーザーインスタンスは、`Illuminate\Contracts\Auth\Authenticatable`[契約](/docs/{{version}}/contracts)の実装である必要があります。Laravelに含まれている`App\Models\User`モデルは前もってこのインターフェイスを実装しています。この認証方法は、ユーザーがアプリケーションに登録した直後など、すでに有効なユーザーインスタンスがある場合に役立ちます。

    use Illuminate\Support\Facades\Auth;

    Auth::login($user);

`login`メソッドの２番目の引数へ論理値を渡たせます。この値は、認証されたセッションに「継続ログイン（remember me）」機能が必要かどうかを示します。これはセッションが無期限に、またはユーザーがアプリケーションから自分でログアウトするまで認証され続けることを意味します。

    Auth::login($user, $remember = true);

必要に応じて、`login`メソッドを呼び出す前に認証ガードを指定できます。

    Auth::guard('admin')->login($user);

<a name="authenticate-a-user-by-id"></a>
#### IDによるユーザー認証

データベースレコードの主キーを使用してユーザーを認証するには、`loginUsingId`メソッドを使用できます。このメソッドは、認証するユーザーの主キーを引数に取ります。

    Auth::loginUsingId(1);

`loginUsingId`メソッドの`remember`引数として論理値を渡すことができます。この値は、認証されたセッションに「継続ログイン（remember me）」機能が必要かどうかを示します。これはセッションが無期限に、またはユーザーがアプリケーションから自分でログアウトするまで認証され続けることを意味します。

    Auth::loginUsingId(1, remember: true);

<a name="authenticate-a-user-once"></a>
#### ユーザーを１回認証する

`once`メソッドを使用して、そのリクエスト１回だけアプリケーションでユーザーを認証できます。このメソッドを呼び出すときは、セッションやクッキーを使用しません。

    if (Auth::once($credentials)) {
        // ...
    }

<a name="http-basic-authentication"></a>
## HTTP基本認証

[HTTP基本認証](https://en.wikipedia.org/wiki/Basic_access_authentication)は、専用の「ログイン」ページを設定せずに、アプリケーションのユーザーを認証する簡単な方法を提供します。使用するには、`auth.basic`[ミドルウェア](/docs/{{version}}/middleware)をルートに指定します。`auth.basic`ミドルウェアはLaravelフレームワークに含まれているため、定義する必要はありません。

    Route::get('/profile', function () {
        // 認証済みユーザーのみがこのルートにアクセス可能
    })->middleware('auth.basic');

このミドルウェアがルートに接続されると、ブラウザでそのルートへアクセスするときにログイン情報の入力が自動的に求められます。デフォルトでは、`auth.basic`ミドルウェアは、`users`データベーステーブルの`email`カラムがユーザーの"username"であると想定します。

<a name="a-note-on-fastcgi"></a>
#### FastCGIに関する注記

PHPFastCGIとApacheを使用してLaravelアプリケーションを提供している場合、HTTP基本認証は正しく機能しない可能性があります。これらの問題を修正するために、アプリケーションの`.htaccess`ファイルに次の行を追加できます。

```apache
RewriteCond %{HTTP:Authorization} ^(.+)$
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```

<a name="stateless-http-basic-authentication"></a>
### ステートレスなHTTP基本認証

セッションでユーザー識別子クッキーを設定せずにHTTP基本認証を使用することもできます。これは主に、HTTP認証を使用してアプリケーションのAPIへの要求を認証することを選択した場合に役立ちます。これを実現するには、`onceBasic`メソッドを呼び出す[ミドルウェアを定義](/docs/{{version}}/middleware)します。`onceBasic`メソッドによってレスポンスが返されない場合、そのリクエストはさらにアプリケーションに渡されます。

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Auth;
    use Symfony\Component\HttpFoundation\Response;

    class AuthenticateOnceWithBasicAuth
    {
        /**
         * 受信リクエストの処理
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            return Auth::onceBasic() ?: $next($request);
        }

    }

Next, attach the middleware to a route:

    Route::get('/api/user', function () {
        // 認証済みユーザーのみがこのルートにアクセス可能
    })->middleware(AuthenticateOnceWithBasicAuth::class);

<a name="logging-out"></a>
## ログアウト

アプリケーションからユーザーを手作業でログアウトするには、`Auth`ファサードが提供する`logout`メソッドを使用できます。これにより、ユーザーのセッションから認証情報が削除され、以降のリクエストが認証されなくなります。

`logout`メソッドを呼び出すことに加えて、ユーザーのセッションを無効にして、[CSRFトークン](/docs/{{version}}/csrf)を再生成することを推奨します。ユーザーをログアウトした後、通常はユーザーをアプリケーションのルートにリダイレクトします。

    use Illuminate\Http\Request;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Support\Facades\Auth;

    /**
     * ユーザーをアプリケーションからログアウトさせる
     */
    public function logout(Request $request): RedirectResponse
    {
        Auth::logout();

        $request->session()->invalidate();

        $request->session()->regenerateToken();

        return redirect('/');
    }

<a name="invalidating-sessions-on-other-devices"></a>
### 他のデバイスでのセッションの無効化

Laravelは、現在のデバイスのセッションを無効にすることなく、他のデバイスでアクティブなそのユーザーのセッションを無効にして「ログアウト」するためのメカニズムも提供しています。この機能は通常、ユーザーがパスワードを変更または更新していて、現在のデバイスを認証したまま他のデバイスのセッションを無効にしたい状況で使用します。

これを使用する前に、セッション認証を受け取るルートへ`Illuminate\Session\Middleware\AuthenticateSession`ミドルウェアを確実に指定する必要があります。通常、このミドルウェアをルートグループ定義に置いて、アプリケーションの大半のルートに適用できるようにします。`AuthenticateSession`ミドルウェアはデフォルトで、`auth.session`[ミドルウェアエイリアス](/docs/{{version}}/middleware#middleware-aliases)を使ってルートへ指定します。

    Route::middleware(['auth', 'auth.session'])->group(function () {
        Route::get('/', function () {
            // ...
        });
    });

次に、`Auth`ファサードが提供する`logoutOtherDevices`メソッドを使用します。このメソッドは、アプリケーションが入力フォームから受け入れた、ユーザーの現在のパスワードを確認する必要があります。

    use Illuminate\Support\Facades\Auth;

    Auth::logoutOtherDevices($currentPassword);

`logoutOtherDevices`メソッドが呼び出されると、そのユーザーの他のセッションは完全に無効になります。つまり、以前に認証されたすべてのガードから「ログアウト」されます。

<a name="password-confirmation"></a>
## パスワードの確認

アプリケーションの構築中、あるアクションを実行させる前、またはユーザーをアプリケーションの機密領域へリダイレクトする前に、ユーザーへパスワードを確認してもらうアクションの発生する場面がしばし起こるでしょう。Laravelはこの行程を簡単にするためのミドルウェアが組み込まれています。この機能を実装するには、２つのルートを定義する必要があります。１つはユーザーにパスワードの確認を求めるビューを表示するルート、もう１つはパスワードの有効性を確認後、ユーザーを目的の行き先にリダイレクトするルートです。

> [!NOTE]
> 次のドキュメントでは、Laravelのパスワード確認機能を直接統合する方法について説明しています。ただし、もっと早く始めたい場合は、[Laravelアプリケーションスターターキット](/docs/{{version}}/starter-kits)にこの機能のサポートが含まれています！

<a name="password-confirmation-configuration"></a>
### 設定

パスワードを確認したら、ユーザーはその後３時間パスワードを再確認するように求められることはありません。ただし、アプリケーションの`config/auth.php`設定ファイル内の`password_timeout`設定値の値を変更することにより、ユーザーがパスワードの再入力を求められるまでの時間を設定できます。

<a name="password-confirmation-routing"></a>
### ルーティング

<a name="the-password-confirmation-form"></a>
#### パスワード確認フォーム

まず、ユーザーにパスワードの確認を要求するビューを表示するルートを定義します。

    Route::get('/confirm-password', function () {
        return view('auth.confirm-password');
    })->middleware('auth')->name('password.confirm');

ご想像のとおり、このルートによって返されるビューには、`password`フィールドを含むフォームが必要です。さらに、ユーザーがアプリケーションの保護された領域に入ろうとしているため、パスワードを確認する必要があることを説明する、お好みなテキストをビュー内に含めてください。

<a name="confirming-the-password"></a>
#### パスワードの確認

次に、「パスワードの確認」ビューからのフォーム要求を処理するルートを定義します。このルートはパスワードを検証し、ユーザーを目的の行き先にリダイレクトする責任を負っています。

    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Hash;
    use Illuminate\Support\Facades\Redirect;

    Route::post('/confirm-password', function (Request $request) {
        if (! Hash::check($request->password, $request->user()->password)) {
            return back()->withErrors([
                'password' => ['The provided password does not match our records.']
            ]);
        }

        $request->session()->passwordConfirmed();

        return redirect()->intended();
    })->middleware(['auth', 'throttle:6,1']);

先へ進む前に、このルートをさらに詳しく調べてみましょう。まず、リクエストの`password`フィールドが、認証済みユーザーのパスワードと実際に一致するか判定されます。パスワードが有効な場合、ユーザーがパスワードを確認したことをLaravelのセッションに通知する必要があります。`passwordConfirmed`メソッドは、ユーザーのセッションにタイムスタンプを設定します。このタイムスタンプを使用して、ユーザーが最後にパスワードを確認した日時をLaravelは判別できます。最後に、ユーザーを目的の行き先にリダイレクトします。

<a name="password-confirmation-protecting-routes"></a>
### ルートの保護

最近パスワード確認を済ませたことが必要なアクションを実行するすべてのルートに、`password.confirm`ミドルウェアが割り当てられていることを確認する必要があります。このミドルウェアはLaravelのデフォルトインストールに含まれており、ユーザーがパスワードを確認した後にその場所へリダイレクトされるように、ユーザーの意図した行き先をセッションに自動的に保存します。ユーザーの意図した行き先をセッションに保存した後、ミドルウェアはユーザーを`password.confirm`[名前付きルート](/docs/{{version}}/routing#named-routes)にリダイレクトします。

    Route::get('/settings', function () {
        // ...
    })->middleware(['password.confirm']);

    Route::post('/settings', function () {
        // ...
    })->middleware(['password.confirm']);

<a name="adding-custom-guards"></a>
## カスタムガードの追加

独自の認証ガードを定義するには、`Auth`ファサードの`extend`メソッドを使用します。`extend`メソッドの呼び出しは、[サービスプロバイダ](/docs/{{version}}/providers)内に置く必要があります。Laravelはあらかじめ`AppServiceProvider`を同梱しているので、そのプロバイダ内にコードを配置することができます。

    <?php

    namespace App\Providers;

    use App\Services\Auth\JwtGuard;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\Facades\Auth;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        // ...

        /**
         * アプリケーションの全サービスの初期起動処理
         */
        public function boot(): void
        {
            Auth::extend('jwt', function (Application $app, string $name, array $config) {
                // Illuminate\Contracts\Auth\Guardインスタンスを返す…

                return new JwtGuard(Auth::createUserProvider($config['provider']));
            });
        }
    }

上記の例でわかるように、`extend`メソッドへ渡たすコールバックは、`Illuminate\Contracts\Auth\Guard`の実装を返す必要があります。このインターフェイスには、カスタムガードを定義するために実装する必要のあるメソッドがいくつか含まれています。カスタムガードを定義したら、`auth.php`設定ファイルの`guards`設定でそのガードを参照します。

    'guards' => [
        'api' => [
            'driver' => 'jwt',
            'provider' => 'users',
        ],
    ],

<a name="closure-request-guards"></a>
### クロージャリクエストガード

カスタムHTTPリクエストベースの認証システムを実装する最も簡単な方法は、`Auth::viaRequest`メソッドを使用することです。この方法は、単一クロージャを使用して認証プロセスをすばやく定義できます。

使用を開始するには、アプリケーションの`AppServiceProvider`の`boot`メソッド内で、`Auth::viaRequest`メソッドを呼び出します。`viaRequest`メソッドの最初の引数は、認証ドライバの名前を指定します。この名前は、カスタムガードを表す任意の文字列を指定できます。このメソッドの第２引数には、HTTPリクエストを受け取ってユーザーインスタンスを返すか、認証に失敗した場合は`null`を返すクロージャを指定します。

    use App\Models\User;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Auth;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        Auth::viaRequest('custom-token', function (Request $request) {
            return User::where('token', (string) $request->token)->first();
        });
    }

カスタム認証ドライバを定義したら、`auth.php`設定ファイルの`guards`設定でドライバとして指定します。

    'guards' => [
        'api' => [
            'driver' => 'custom-token',
        ],
    ],

最後に、認証ミドルウェアをルートへ割り当てるときに、ガードを参照してください。

    Route::middleware('auth:api')->group(function () {
        // ...
    });

<a name="adding-custom-user-providers"></a>
## カスタムユーザープロバイダの追加

従来のリレーショナルデータベースを使用してユーザーを保存していない場合は、独自の認証ユーザープロバイダでLaravelを拡張する必要があります。`Auth`ファサードで`provider`メソッドを使用して、カスタムユーザープロバイダを定義します。ユーザープロバイダリゾルバは、`Illuminate\Contracts\Auth\UserProvider`の実装を返す必要があります。

    <?php

    namespace App\Providers;

    use App\Extensions\MongoUserProvider;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\Facades\Auth;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        // ...

        /**
         * アプリケーションの全サービスの初期起動処理
         */
        public function boot(): void
        {
            Auth::provider('mongo', function (Application $app, array $config) {
                // Illuminate\Contracts\Auth\UserProviderのインスタンスを返す…

                return new MongoUserProvider($app->make('mongo.connection'));
            });
        }
    }

`provider`メソッドを使用してプロバイダを登録した後、`auth.php`設定ファイルで新しいユーザープロバイダに切り替えてください。まず、新しいドライバを使用する`provider`を定義します。

    'providers' => [
        'users' => [
            'driver' => 'mongo',
        ],
    ],

最後に、`guards`設定でこのプロバイダを参照します。

    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],
    ],

<a name="the-user-provider-contract"></a>
### ユーザープロバイダ契約

`Illuminate\Contracts\Auth\UserProvider`の実装は、MySQLやMongoDBなどの永続ストレージシステムから`Illuminate\Contracts\Auth\Authenticatable`実装をフェッチする責務を持っています。これら２つのインターフェイスにより、ユーザーデータの保存方法や、認証されたユーザーを表すために使用されるクラスのタイプに関係なく、Laravel認証メカニズムは機能し続けられます。

`Illuminate\Contracts\Auth\UserProvider`コントラクトを見てみましょう。

    <?php

    namespace Illuminate\Contracts\Auth;

    interface UserProvider
    {
        public function retrieveById($identifier);
        public function retrieveByToken($identifier, $token);
        public function updateRememberToken(Authenticatable $user, $token);
        public function retrieveByCredentials(array $credentials);
        public function validateCredentials(Authenticatable $user, array $credentials);
        public function rehashPasswordIfRequired(Authenticatable $user, array $credentials, bool $force = false);
    }

`retrieveById`関数は通常、MySQLデータベースからの自動増分IDなど、ユーザーを表すキーを受け取ります。IDに一致する`Authenticatable`実装は、このメソッドにより取得され、返えされる必要があります。

`retrieveByToken`関数は、一意の`$identifier`と"remember me" `$token`によってユーザーを取得します。通常、これは`remember_token`のようなデータベースカラムに格納されます。前のメソッドと同様に、トークン値が一致する`Authenticatable`の実装をこのメソッドは返す必要があります。

`updateRememberToken`メソッドは、`$user`インスタンスの`remember_token`を新しい`$token`で更新します。「remember me」認証の試行が成功したとき、またはユーザーがログアウトしているときに、新しいトークンをユーザーに割り当てます。

`retrieveByCredentials`メソッドは、アプリケーションで認証を試みるときに、`Auth::attempt`メソッドに渡されたログイン情報の配列を受け取ります。次に、メソッドはこれらのログイン情報に一致するユーザーを永続ストレージへ「クエリ」する必要があります。通常、このメソッドは、`$credentials['username']`の値と一致する"username"を持つユーザーレコードを検索する"WHERE"条件でクエリを実行します。このメソッドは、`Authenticatable`の実装を返す必要があります。**この方法では、パスワードの検証や認証を試みるべきではありません。**

`validateCredentials`メソッドは、指定された`$user`を`$credentials`と比較してユーザーを認証する必要があります。たとえば、このメソッドは通常、`Hash::check`メソッドを使用して、`$user->getAuthPassword()`の値を`$credentials['password']`の値と比較します。このメソッドは、パスワードが有効かどうかを示す`true`か`false`を返す必要があります。

`rehashPasswordIfRequired`メソッドは、指定された`$user`のパスワードが必須かつサポートされている場合、そのパスワードを再ハッシュする必要があります。例えば、このメソッドは通常`$credentials['password']`の値を再ハッシュする必要があるかを判定するために`Hash::needsRehash`メソッドを使用するでしょう。パスワードをリハッシュする必要がある場合、このメソッドは、`Hash::make`メソッドを使用してパスワードをリハッシュし、永続ストレージ内のユーザーのレコードを更新する必要があります。

<a name="the-authenticatable-contract"></a>
### Authenticatable契約

`UserProvider`の各メソッドについて説明したので、`Authenticatable`契約を見てみましょう。ユーザープロバイダは、`retrieveById`、`retrieveByToken`、`retrieveByCredentials`メソッドからこのインターフェイスの実装を返す必要があることに注意してください。

    <?php

    namespace Illuminate\Contracts\Auth;

    interface Authenticatable
    {
        public function getAuthIdentifierName();
        public function getAuthIdentifier();
        public function getAuthPasswordName();
        public function getAuthPassword();
        public function getRememberToken();
        public function setRememberToken($value);
        public function getRememberTokenName();
    }

このインターフェイスはシンプルです。`getAuthIdentifierName`メソッドはユーザーの「主キー」カラムの名前を返し、`getAuthIdentifier`メソッドはユーザーの「主キー」を返します。MySQLバックエンドを使用している場合は、ユーザーレコードに割り当てられている自動インクリメントの主キーになるでしょう。`getAuthPasswordName`メソッドは、ユーザーのパスワードカラムの名前を返します。`getAuthPassword`メソッドは、ユーザーのハッシュ化済みパスワードを返します。

このインターフェイスにより、使用しているORMまたはストレージ抽象化レイヤーに関係なく、認証システムは任意の「ユーザー」クラスと連携できます。デフォルトでLaravelは`app/Models`ディレクトリに、このインターフェイスを実装する`App\Models\User`クラスを持っています。

<a name="automatic-password-rehashing"></a>
## パスワードの自動再ハッシュ

Laravelのデフォルトのパスワードハッシュアルゴリズムはbcryptです。bcryptハッシュの「ストレッチング（ワークファクター）」は、アプリケーションの`config/hashing.php`設定ファイル、または`BCRYPT_ROUNDS`環境変数で調整できます。

通常、bcryptのワークファクターは、ＣＰＵ／ＧＰＵの処理能力が上がるにつれ増やし、時間をかけていく必要があります。アプリケーションのbcryptワークファクターを上げると、Laravelのスターターキットや、`attempt`メソッドにより[手作業でユーザーを認証する](#authenticating-users)時に、Laravelはユーザーのパスワードをスムーズかつ自動的にリハッシュします。

通常、パスワードの自動再ハッシュは、アプリケーションを混乱させることはありません。しかし、`hashing`設定ファイルを公開することで、この動作を無効にすることができます：

```shell
php artisan config:publish hashing
```

設定ファイルを公開したら、`rehash_on_login`設定値を`false`へ設定してください。

```php
'rehash_on_login' => false,
```

<a name="events"></a>
## イベント

Laravelは、認証プロセス中に様々な[イベント](/docs/{{version}}/events)をディスパッチします。以下のイベントに対して、[リスナを定義](/docs/{{version}}/events)できます.

<div class="overflow-auto">

| イベント名 |
| --- |
| `Illuminate\Auth\Events\Registered` |
| `Illuminate\Auth\Events\Attempting` |
| `Illuminate\Auth\Events\Authenticated` |
| `Illuminate\Auth\Events\Login` |
| `Illuminate\Auth\Events\Failed` |
| `Illuminate\Auth\Events\Validated` |
| `Illuminate\Auth\Events\Verified` |
| `Illuminate\Auth\Events\Logout` |
| `Illuminate\Auth\Events\CurrentDeviceLogout` |
| `Illuminate\Auth\Events\OtherDeviceLogout` |
| `Illuminate\Auth\Events\Lockout` |
| `Illuminate\Auth\Events\PasswordReset` |

</div>
