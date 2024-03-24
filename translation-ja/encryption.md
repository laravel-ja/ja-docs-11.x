# 暗号化

- [イントロダクション](#introduction)
- [設定](#configuration)
    - [優しい暗号化キーのローテーション](#gracefully-rotating-encryption-keys)
- [エンクリプタの使用](#using-the-encrypter)

<a name="introduction"></a>
## イントロダクション

Laravelの暗号化サービスは、AES-256およびAES-128暗号化を使用してOpenSSLを介してテキストを暗号化および復号化するためのシンプルで便利なインターフェイスを提供します。Laravelの暗号化された値はすべて、メッセージ認証コード(MAC)を使用して署名されているため、暗号化された値の基になる値を変更したり改ざんしたりすることはできません。

<a name="configuration"></a>
## 設定

Laravelの暗号化を使用する前に、`config/app.php`設定ファイルで`key`設定オプションを設定する必要があります。この設定値は、`APP_KEY`環境変数が反映されます。`php artisan key:generate`コマンドを使用してこの変数の値を生成する必要があります。これは、`key:generate`コマンドがPHPの安全なランダムバイトジェネレーターを使用して、アプリケーションの暗号的に安全なキーを構築するためです。通常、`APP_KEY`環境変数の値は、[Laravelのインストール](/docs/{{version}}/installation)中に生成されます。

<a name="gracefully-rotating-encryption-keys"></a>
### 優しい暗号化キーのローテーション

アプリケーションの暗号化キーを変更すると、認証済みユーザーセッションをアプリケーションから全てログアウトします。これは、セッションクッキーを含む全てのクッキーがLaravelによって暗号化されているためです。さらに、以前の暗号化キーで暗号化されたデータを復号することもできなくなります。

Laravelはこの問題を軽減するため.、アプリケーションの`APP_PREVIOUS_KEYS`環境変数へ、以前の暗号化キーをリストアップしておくことができます。この変数に以前の暗号化キーをカンマ区切りで列挙してください。

```ini
APP_KEY="base64:J63qRTDLub5NuZvP+kb8YIorGS6qFYHKVo6u7179stY="
APP_PREVIOUS_KEYS="base64:2nLsGFGzyoae2ax3EF2Lyq/hH6QghBGLIq5uL+Gp8/w="
```

この環境変数を設定すると、Laravelは値を暗号化する時、常に「現在の」暗号化キーを使用します。しかし、値を復号化する場合、Laravelはまず現在のキーを試し、復号化に失敗すると、いずれかのキーで復号化できるまで、Laravelは以前のキーをすべて試します。

このやさしい復号化のアプローチにより、暗号化キーをローテーションしても、ユーザーはアプリケーションを中断することなく使い続けることができます。

<a name="using-the-encrypter"></a>
## エンクリプタの使用

<a name="encrypting-a-value"></a>
#### 値の暗号化

`Crypt`ファサードが提供する`encryptString`メソッドを使用して値を暗号化できます。暗号化された値はすべて、OpenSSLとAES-256-CBC暗号を使用して暗号化されます。さらに、暗号化されたすべての値は、メッセージ認証コード(MAC)で署名されます。統合されたメッセージ認証コードは、悪意のあるユーザーにより改ざんされた値の復号化を防ぎます。

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Crypt;

    class DigitalOceanTokenController extends Controller
    {
        /**
         * ユーザーのDigitalOceanAPIトークンを保存
         */
        public function store(Request $request): RedirectResponse
        {
            $request->user()->fill([
                'token' => Crypt::encryptString($request->token),
            ])->save();

            return redirect('/secrets');
        }
    }

<a name="decrypting-a-value"></a>
#### 値の復号

`Crypt`ファサードが提供する`decryptString`メソッドを使用して値を復号化できます。メッセージ認証コードが無効な場合など、値を適切に復号化できない場合、`Illuminate\Contracts\Encryption\DecryptException`を投げます。

    use Illuminate\Contracts\Encryption\DecryptException;
    use Illuminate\Support\Facades\Crypt;

    try {
        $decrypted = Crypt::decryptString($encryptedValue);
    } catch (DecryptException $e) {
        // ...
    }
