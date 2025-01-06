# パスワードリセット

- [イントロダクション](#introduction)
    - [モデルの準備](#model-preparation)
    - [データベース準備](#database-preparation)
    - [信頼するホストの設定](#configuring-trusted-hosts)
- [ルート](#routing)
    - [パスワードリセットリンクの要求](#requesting-the-password-reset-link)
    - [パスワードのリセット](#resetting-the-password)
- [期限切れトークンの削除](#deleting-expired-tokens)
- [カスタマイズ](#password-customization)

<a name="introduction"></a>
## イントロダクション

ほとんどのWebアプリケーションは、ユーザーが忘れたパスワードをリセットする方法を提供します。Laravelでは、構築するすべてのアプリケーションでこれを手作業で再実装する必要はなく、パスワードリセットリンクを送信してパスワードを安全にリセットするための便利なサービスを提供しています。

> [!NOTE]
> さっそく始めたいですか？Laravel[アプリケーションスターターキット](/docs/{{version}}/starter-kits)を新しいLaravelアプリケーションにインストールしてください。Laravelのスターターキットは、忘れたパスワードのリセットを含む、認証システム全体のスカフォールドの面倒を見ています。

<a name="model-preparation"></a>
### モデルの準備

Laravelのパスワードリセット機能を使用する前に、アプリケーションの`App\Models\User`モデルで`Illuminate\Notifications\Notifiable`トレイトを使用する必要があります。通常、このトレイトは、新しいLaravelアプリケーションで作成されるデフォルトの`App\Models\User`モデルに最初から含まれています。

次に、`App\Models\User`モデルが`Illuminate\Contracts\Auth\CanResetPassword`コントラクトを実装していることを確認します。フレームワークに含まれている`App\Models\User`モデルは、最初からこのインターフェイスを実装しており、`Illuminate\Auth\Passwords\CanResetPassword`トレイトを使用して、インターフェイスの実装に必要なメソッドを持っています。

<a name="database-preparation"></a>
### データベース準備

アプリケーションのパスワードリセットトークンを保存するテーブルを作成する必要があります。通常、これはLaravelのデフォルトの`0001_01_000000_create_users_table.php`データベースマイグレーションに含まれています。

<a name="configuring-trusted-hosts"></a>
### 信頼するホストの設定

デフォルトでは、LaravelはHTTPリクエストの`host`ヘッダの内容に関係なく受信したすべてのリクエストにレスポンスします。さらに、Webリクエスト中にアプリケーションへの絶対URLを生成するときに、`host`ヘッダの値を使用します。

通常、NginxやApacheなどのウェブは、指定したホスト名に一致するリクエストのみをアプリケーションへ送信するように設定する必要があります。しかし、ウェブを直接カスタマイズする権限がなく、特定のホスト名だけを応答するようにLaravelへ指示する必要がある場合は、アプリケーションの`bootstrap/app.php`ファイルで`trustHosts`ミドルウェアメソッドを使用することで可能です。これは、アプリケーションがパスワードリセット機能を提供する場合に特に重要です。

このミドルウェアのメソッドの詳細は、[`TrustHosts`ミドルウェアのドキュメント](/docs/{{version}}/requests#configuring-trusted-hosts)を参照してください。

<a name="routing"></a>
## ルート

ユーザーがパスワードをリセットできるようにするためのサポートを適切に実装するには、ルートをいくつか定義する必要があります。最初に、ユーザーが自分の電子メールアドレスを介してパスワードリセットリンクをリクエストできるようにするためのルートのペアが必要になります。２つ目は、ユーザーが電子メールで送られてきたパスワードリセットリンクにアクセスしてパスワードリセットフォームに記入した後、実際にパスワードをリセットするためのルートが必要になります。

<a name="requesting-the-password-reset-link"></a>
### パスワードリセットリンクの要求

<a name="the-password-reset-link-request-form"></a>
#### パスワードリセットリンクリクエストフォーム

まず、パスワードリセットリンクをリクエストするために必要なルートを定義します。手始めに、パスワードリセットリンクリクエストフォームを使用してビューを返すルートを定義します。

    Route::get('/forgot-password', function () {
        return view('auth.forgot-password');
    })->middleware('guest')->name('password.request');

このルートによって返されるビューには、`email`フィールドを含むフォームが必要です。これにより、ユーザーは特定の電子メールアドレスのパスワードリセットリンクをリクエストできます。

<a name="password-reset-link-handling-the-form-submission"></a>
#### フォーム送信処理

次に、「パスワードを忘れた」ビューからのフォーム送信リクエストを処理するルートを定義します。このルートは、電子メールアドレスを検証し、対応するユーザーにパスワードリセットリクエストを送信する責任があります。

    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Password;

    Route::post('/forgot-password', function (Request $request) {
        $request->validate(['email' => 'required|email']);

        $status = Password::sendResetLink(
            $request->only('email')
        );

        return $status === Password::ResetLinkSent
                    ? back()->with(['status' => __($status)])
                    : back()->withErrors(['email' => __($status)]);
    })->middleware('guest')->name('password.email');

先に進む前に、このルートをさらに詳しく調べてみましょう。最初に、リクエストの`email`属性が検証されます。次に、Laravelの組み込みの「パスワードブローカ」(`Password`ファサードが返す)を使用して、パスワードリセットリンクをユーザーに送信します。パスワードブローカは、指定するフィールド(この場合はメールアドレス)でユーザーを取得し、Laravelの組み込み[通知システム](/docs/{{version}}/notifications)を介してユーザーにパスワードリセットリンクを送信します。

`sendResetLink`メソッドは、"status"スラッグを返します。このステータスは、Laravelの[多言語化](/docs/{{version}}/localization)ヘルパを使って翻訳でき、リクエストのステータスに関するユーザーフレンドリーなメッセージをユーザーへ表示可能にします。パスワードリセットステータスの翻訳は、アプリケーションの`lang/{lang}/passwords.php`言語ファイルで決まります。ステータスのスラッグに指定できる各項目は、`passwords`言語ファイル内にあります。

> [!NOTE]
> Laravelアプリケーションのスケルトンは、デフォルトでは`lang`ディレクトリを含んでいません。Laravelの言語ファイルをカスタマイズしたい場合は、`lang:publish` Artisanコマンドでリソース公開できます。

`Password`ファサードの`sendResetLink`メソッドを呼び出すときに、Laravelがアプリケーションのデータベースからユーザーレコードを取得する方法をどのように知っているのか疑問に思われるかもしれません。Laravelパスワードブローカは、認証システムの「ユーザープロバイダ」を利用してデータベースレコードを取得します。パスワードブローカが使用するユーザープロバイダは、`config/auth.php`設定ファイルの`passwords`設定配列内で設定します。カスタムユーザープロバイダの作成の詳細については、[認証ドキュメント](/docs/{{version}}/authentication#adding-custom-user-providers)を参照してください。

> [!NOTE]
> パスワードのリセットを手作業で実装する場合は、ビューの内容とルートを自分で定義する必要があります。必要なすべての認証および検証ロジックを含むスカフォールドが必要な場合は、[Laravelアプリケーションスターターキット](/docs/{{version}}/starter-kits)を確認してください。

<a name="resetting-the-password"></a>
### パスワードのリセット

<a name="the-password-reset-form"></a>
#### パスワードリセットフォーム

次に、電子メールで送信されたパスワードリセットリンクをユーザーがクリックして新しいパスワードを入力したときに、実際にパスワードをリセットするために必要なルートを定義します。まず、ユーザーがパスワードのリセットリンクをクリックしたときに表示されるパスワードのリセットフォームを表示するルートを定義しましょう。このルートは、後でパスワードリセットリクエストを確認するために使用する`token`パラメータを受け取ります。

    Route::get('/reset-password/{token}', function (string $token) {
        return view('auth.reset-password', ['token' => $token]);
    })->middleware('guest')->name('password.reset');

このルートが返すビューにより、`email`フィールド、`password`フィールド、`password_confirmation`フィールド、および非表示の`token`フィールドを含むフォームを表示します。これにはルートが受け取る秘密の`$token`の値が含まれている必要があります。

<a name="password-reset-handling-the-form-submission"></a>
#### フォーム送信処理

もちろん、パスワードリセットフォームの送信を実際に処理するためルートを定義する必要もあります。このルートは、受信リクエストのバリデーションとデータベース内のユーザーのパスワードの更新を担当します。

    use App\Models\User;
    use Illuminate\Auth\Events\PasswordReset;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Hash;
    use Illuminate\Support\Facades\Password;
    use Illuminate\Support\Str;

    Route::post('/reset-password', function (Request $request) {
        $request->validate([
            'token' => 'required',
            'email' => 'required|email',
            'password' => 'required|min:8|confirmed',
        ]);

        $status = Password::reset(
            $request->only('email', 'password', 'password_confirmation', 'token'),
            function (User $user, string $password) {
                $user->forceFill([
                    'password' => Hash::make($password)
                ])->setRememberToken(Str::random(60));

                $user->save();

                event(new PasswordReset($user));
            }
        );

        return $status === Password::PasswordReset
                    ? redirect()->route('login')->with('status', __($status))
                    : back()->withErrors(['email' => [__($status)]]);
    })->middleware('guest')->name('password.update');

先に進む前に、このルートをさらに詳しく調べてみましょう。最初に、リクエストの`token`、`email`、および`password`属性がバリデーションされます。次に、Laravelの組み込みの「パスワードブローカ」(`Password`ファサードが返す)を使用して、パスワードリセットリクエストの資格情報を検証します。

パスワードブローカに与えられたトークン、電子メールアドレス、およびパスワードが有効である場合、`reset`メソッドに渡されたクロージャが呼び出されます。ユーザーインスタンスとパスワードリセットフォームに提供された平文テキストのパスワードを受け取るこのクロージャ内で、データベース内のユーザーのパスワードを更新します。

`reset`メソッドは、「ステータス」スラグを返します。このステータスは、リクエストのステータスに関するユーザーフレンドリーなメッセージをユーザーに表示するために、Laravelの [多言語化](/docs/{{version}}/localization)ヘルパを使用して翻訳できます。パスワードリセットステータスの翻訳は、アプリケーションの`lang/{lang}/passwords.php`言語ファイルにより行います。`passwords`言語ファイルの中に、ステータスのスラグがとり得る値のエントリを配置しています。アプリケーションに`lang` ディレクトリがない場合は、`lang:publish` Artisan コマンドを使用して作成できます。

先に進む前に、`Password`ファサードの`reset`メソッドを呼び出すときに、Laravelがアプリケーションのデータベースからユーザーレコードを取得する方法をどのように知っているのか疑問に思われるかもしれません。Laravelパスワードブローカは、認証システムの「ユーザープロバイダ」を利用してデータベースレコードを取得します。パスワードブローカが使用するユーザープロバイダは、`config/auth.php`設定ファイルの`passwords`設定配列内で設定しています。カスタムユーザープロバイダの作成の詳細については、[認証ドキュメント](/docs/{{version}}/authentication#adding-custom-user-providers)を参照してください。

<a name="deleting-expired-tokens"></a>
## 期限切れトークンの削除

期限が切れたパスワードリセットトークンは、データベース内にまだ存在します。しかし、これらのレコードは、`auth:clear-resets` Artisanコマンドで簡単に削除できます。

```shell
php artisan auth:clear-resets
```

この処理を自動化したい場合は、アプリケーションの[スケジューラ](/docs/{{version}}/scheduling)への、当コマンド追加を検討してください。

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('auth:clear-resets')->everyFifteenMinutes();

<a name="password-customization"></a>
## カスタマイズ

<a name="reset-link-customization"></a>
#### リセットリンクのカスタマイズ

`ResetPassword`通知クラスが提供する`createUrlUsing`メソッドを使用して、パスワードリセットリンクのURLをカスタマイズできます。このメソッドは、通知を受け取るユーザーインスタンスとパスワードリセットリンクトークンを受け取るクロージャを引数に取ります。通常、このメソッドは`App\Providers\AppServiceProvider`サービスプロバイダの`boot`メソッドから呼び出します。

    use App\Models\User;
    use Illuminate\Auth\Notifications\ResetPassword;

    /**
     * アプリケーションの全サービスの初期起動処理
     */
    public function boot(): void
    {
        ResetPassword::createUrlUsing(function (User $user, string $token) {
            return 'https://example.com/reset-password?token='.$token;
        });
    }

<a name="reset-email-customization"></a>
#### リセットメールカスタマイズ

パスワードリセットリンクをユーザーに送信するために使用する通知クラスは簡単に変更できます。それには、`App\Models\User`モデルの`sendPasswordResetNotification`メソッドをオーバーライドします。このメソッド内で、自分で作成した[通知クラス](/docs/{{version}}/notifications)を使用して通知を送信できます。パスワードリセット`$token`は、メソッドが受け取る最初の引数です。この`$token`を使用して、パスワードリセットURLを作成し、ユーザーに通知を送信します。

    use App\Notifications\ResetPasswordNotification;

    /**
     * パスワードリセット通知をユーザーに送信
     *
     * @param  string  $token
     */
    public function sendPasswordResetNotification($token): void
    {
        $url = 'https://example.com/reset-password?token='.$token;

        $this->notify(new ResetPasswordNotification($url));
    }
