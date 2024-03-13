# レート制限

- [イントロダクション](#introduction)
    - [キャッシュ設定](#cache-configuration)
- [基本の使い方](#basic-usage)
    - [試行回数の手作業増加](#manually-incrementing-attempts)
    - [試行のクリア](#clearing-attempts)

<a name="introduction"></a>
## イントロダクション

Laravelには、簡単に使用できるレート制限の抽象化機能があり、アプリケーションの[cache](/docs/{{version}}/cache)と連携して、指定した時間帯のアクションを制限する簡単な方法を提供します。

> [!NOTE]
> 受信HTTPリクエストのレートを制限したい場合は、[レート制限ミドルウェアのドキュメント](/docs/{{version}}/routing#rate-limiting)を参照してください。

<a name="cache-configuration"></a>
### キャッシュ設定

通常、レート制限では、アプリケーションの`cache`設定ファイルの`default`キーで定義されているデフォルトのアプリケーションキャッシュを使用します。しかし、アプリケーションの`cache`設定ファイル内で`limiter`キーを定義すれば、レート制限で使用するキャッシュドライバを指定できます。

    'default' => env('CACHE_STORE', 'database'),

    'limiter' => 'redis',

<a name="basic-usage"></a>
## 基本の使い方

`Illuminate\Support\Facades\RateLimiter`ファサードを使って、レート制限を操作できます。レート制限が提供する最もシンプルなメソッドは`attempt`メソッドで、これは指定されたコールバックを指定された秒数でレート制限するものです。

`attempt`メソッドは、コールバックで利用できる残り試行回数がない場合は、`false`を返し、そうでない場合には、`attempt`メソッドはコールバックの結果、または`true`を返します。attempt`メソッドが受け付ける最初の引数は、レート制限の「キー」で、レート制限をかけるアクションを表す任意の文字列を指定します。

    use Illuminate\Support\Facades\RateLimiter;

    $executed = RateLimiter::attempt(
        'send-message:'.$user->id,
        $perMinute = 5,
        function() {
            // メッセージ送信…
        }
    );

    if (! $executed) {
      return 'Too many messages sent!';
    }

必要であれば、`attempt`メソッドに第４引数を与えられます。これは「減衰率」、または試行が利用可能になるまでのリセット秒数です。例として、上記のサンプルコードを修正して、２分ごとに５回の試行するようにしてみます。

    $executed = RateLimiter::attempt(
        'send-message:'.$user->id,
        $perTwoMinutes = 5,
        function() {
            // メッセージ送信…
        },
        $decayRate = 120,
    );

<a name="manually-incrementing-attempts"></a>
### 試行回数の手作業増加

レート制限を手作業で操作する場合は、他にもさまざまな方法があります。たとえば、`tooManyAttempts`メソッドを呼び出して、指定したレート制限キーが１分間に許可した最大試行回数を超えたかを判断できます。

    use Illuminate\Support\Facades\RateLimiter;

    if (RateLimiter::tooManyAttempts('send-message:'.$user->id, $perMinute = 5)) {
        return 'Too many attempts!';
    }

    RateLimiter::increment('send-message:'.$user->id);

    // メッセージ送信処理…

他にも、`remaining`メソッドを使って、指定キーの残りの試行回数を取得することも可能です。指定キーに再試行回数が残っている場合は、`increment`メソッドを呼び出して総試行回数を増やせます。

    use Illuminate\Support\Facades\RateLimiter;

    if (RateLimiter::remaining('send-message:'.$user->id, $perMinute = 5)) {
        RateLimiter::increment('send-message:'.$user->id);

        // メッセージ送信処理…
    }

指定するレートリミッターキーの値を１以上増やしたい場合は、`increment`メソッドへ希望する量を指定してください。

    RateLimiter::increment('send-message:'.$user->id, amount: 5);

<a name="determining-limiter-availability"></a>
#### 使用可能時間の判断

キーの試行回数がなくなると、`availableIn`メソッドは試行回数が増えるまでの残り秒数を返します。

    use Illuminate\Support\Facades\RateLimiter;

    if (RateLimiter::tooManyAttempts('send-message:'.$user->id, $perMinute = 5)) {
        $seconds = RateLimiter::availableIn('send-message:'.$user->id);

        return 'You may try again in '.$seconds.' seconds.';
    }

    RateLimiter::increment('send-message:'.$user->id);

    // メッセージ送信処理…

<a name="clearing-attempts"></a>
### 試行のクリア

`clear`メソッドを使って、任意のレート制限キーの試行回数をリセットできます。例えば、指定メッセージが受信者によって読まれたときに試行回数をリセットできます。

    use App\Models\Message;
    use Illuminate\Support\Facades\RateLimiter;

    /**
     * メッセージを既読としてマーク
     */
    public function read(Message $message): Message
    {
        $message->markAsRead();

        RateLimiter::clear('send-message:'.$message->user_id);

        return $message;
    }
