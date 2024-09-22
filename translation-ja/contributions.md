# 貢献ガイド

- [バグレポート](#bug-reports)
- [質問のサポート](#support-questions)
- [コア開発の議論](#core-development-discussion)
- [どのブランチ？](#which-branch)
- [アセットのコンパイル](#compiled-assets)
- [セキュリティ脆弱性](#security-vulnerabilities)
- [コーディングスタイル](#coding-style)
    - [PHPDoc](#phpdoc)
    - [StyleCI](#styleci)
- [行動規範](#code-of-conduct)

<a name="bug-reports"></a>
## バグレポート

より積極的に援助して頂きたいため、Laravelではただのバグレポートでなく、プルリクエストしてくれることを強く推奨しています。プルリクエストは、”ready for review"（"draft"状態ではない）とマークされている場合にのみレビューされ、新しい機能のすべてのテストを通します。"draft"状態で残された残りの非アクティブプルリクエストは、数日後に閉じられます。

バグレポートを提出する場合にはその問題をタイトルに含め、明確に内容を記述してください。できる限り関連する情報や、その問題をデモするコードも含めてください。バグレポートの目的はあなた自身、そして他の人でも、簡単にバグが再現でき修正されるようにすることです。

バグレポートは、同じ問題を持つ他の人があなたと協力して解決してくれることを期待して作成することを忘れないでください。バグレポートに自動で何らかの活動が表示されたり、他の人がその問題の解決に飛びついたりすることを期待しないでください。バグレポートを作成することは、あなた自身や他の人が問題を解決する道を歩み始める手助けになります。もしあなたが協力したければ、[issueトラッカーに記載しているバグ](https://github.com/issues?q=is%3Aopen+is%3Aissue+label%3Abug+user%3Alaravel)を修正することで協力できます。LaravelのすべてのIssueを表示するには、GitHubで認証する必要があります。

Laravelを使っているときに、DocBlock、PHPStan、IDEの警告に気づいた場合は、GitHub issueを作成しないでください。代わりに、問題を解決するための、プルリクエストを提出してください。

LaravelのソースコードはGitHubで管理され、各Laravelプロジェクトのリポジトリが存在しています。

<div class="content-list" markdown="1">

- [Laravelアプリケーション](https://github.com/laravel/laravel)
- [Laravelアートワーク](https://github.com/laravel/art)
- [Laravel Breeze](https://github.com/laravel/breeze)
- [Laravelドキュメント](https://github.com/laravel/docs)
- [Laravel Dusk](https://github.com/laravel/dusk)
- [Laravel Cashier Stripe](https://github.com/laravel/cashier)
- [Laravel Cashier Paddle](https://github.com/laravel/cashier-paddle)
- [Laravel Echo](https://github.com/laravel/echo)
- [Laravel Envoy](https://github.com/laravel/envoy)
- [Laravel Folio](https://github.com/laravel/folio)
- [Laravelフレームワーク](https://github.com/laravel/framework)
- [Laravel Homestead](https://github.com/laravel/homestead) ([構築スクリプト](https://github.com/laravel/settler))
- [Laravel Horizon](https://github.com/laravel/horizon)
- [Laravel Jetstream](https://github.com/laravel/jetstream)
- [Laravel Passport](https://github.com/laravel/passport)
- [Laravel Pennant](https://github.com/laravel/pennant)
- [Laravel Pint](https://github.com/laravel/pint)
- [Laravel Prompts](https://github.com/laravel/prompts)
- [Laravel Reverb](https://github.com/laravel/reverb)
- [Laravel Sail](https://github.com/laravel/sail)
- [Laravel Sanctum](https://github.com/laravel/sanctum)
- [Laravel Scout](https://github.com/laravel/scout)
- [Laravel Socialite](https://github.com/laravel/socialite)
- [Laravel Telescope](https://github.com/laravel/telescope)
- [Laravel Webサイト](https://github.com/laravel/laravel.com)

</div>

<a name="support-questions"></a>
## 質問のサポート

LaravelのGitHubイシュートラッカーは、Laravelのヘルプやサポートの提供を目的としていません。代わりに以下のチャンネルを利用してください。

<div class="content-list" markdown="1">

- [GitHub Discussions](https://github.com/laravel/framework/discussions)
- [Laracastsフォーラム](https://laracasts.com/discuss)
- [Laravel.ioフォーラム](https://laravel.io/forum)
- [StackOverflow](https://stackoverflow.com/questions/tagged/laravel)
- [Discord](https://discord.gg/laravel)
- [Larachat](https://larachat.co)
- [IRC](https://web.libera.chat/?nick=artisan&channels=#laravel)

</div>

<a name="core-development-discussion"></a>
## コア開発の議論

新機能や、現存のLaravelの振る舞いについて改善を提言したい場合は、Laravelフレームワークリポジトリの[GitHub discussion board](https://github.com/laravel/framework/discussions)へお願いします。新機能を提言する場合は自発的に、それを完動させるのに必要な、コードを最低限でも実装してください。

バグ、新機能、既存機能の実装についてのざっくばらんな議論は、[Laravel Discord server](https://discord.gg/laravel)の`#internals`チャンネルで行っています。LaravelのメンテナーであるTaylor Otwellは、通常ウイークエンドの午前８時から５時まで（America/Chicago標準時、UTC-6:00）接続しています。他の時間帯では、ときどき接続しています。

<a name="which-branch"></a>
## どのブランチ？

**すべての**バグフィックスは、バグフィックスをサポートする最新バージョン (現在は `11.x`) へ送るべきです。次期リリースにのみ存在する機能を修正する場合を除き、バグ修正を**決して**、`master`ブランチに送ってはいけません。

現在のリリースと**完全な下位互換性**がある**マイナー**な機能は、最新の安定版ブランチ（現在は`11.x`）へ送ってください。

**メジャー**な新機能や互換性のない変更を含む機能は、常に次のリリースに含まれる`master`ブランチへ送ってください。

<a name="compiled-assets"></a>
## アセットのコンパイル

`laravel/laravel`リポジトリの`resources/css`や`resources/js`下のほとんどのファイルのように、コンパイル済みファイルに影響を及ぼすファイルへ変更を行う場合、コンパイル済みファイルをコミットしないでください。大きなファイルサイズであるため、メンテナは実際レビューできません。悪意のあるコードをLaravelへ紛れ込ませる方法を提供してしまいます。これを防御的に防ぐため、すべてのコンパイル済みファイルはLaravelメンテナが生成しコミットします。

<a name="security-vulnerabilities"></a>
## セキュリティ脆弱性

Laravelにセキュリティー脆弱性を見つけたときは、メールで[Taylor Otwell(taylorotwell@laravel.com)](mailto:taylor@laravel.com)に連絡してください。全セキュリティー脆弱性は、速やかに対応されるでしょう。

<a name="coding-style"></a>
## コーディングスタイル

Laravelは[PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md)コーディング規約と[PSR-4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md)オートローディング規約に準拠しています。

<a name="phpdoc"></a>
### PHPDoc

次に正しいLaravelのドキュメントブロックの例を示します。`@param`属性に続け２スペース、引数タイプ、２スペース、最後に変数名となっていることに注意してください。

    /**
     * Register a binding with the container.
     *
     * @param  string|array  $abstract
     * @param  \Closure|string|null  $concrete
     * @param  bool  $shared
     * @return void
     *
     * @throws \Exception
     */
    public function bind($abstract, $concrete = null, $shared = false)
    {
        // ...
    }

ネイティブ型の使用により、`@param`属性や`@return`属性が冗長になる場合は、それらを削除することができます。

    /**
     * Execute the job.
     */
    public function handle(AudioProcessor $processor): void
    {
        //
    }

ただし、ネイティブ型が汎用型の場合は、`@param`属性または`@return`属性を用いて汎用型を指定してください。

    /**
     * Get the attachments for the message.
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromStorage('/path/to/file'),
        ];
    }

<a name="styleci"></a>
### StyleCI

コードのスタイルが完璧でなくても心配ありません。プルリクエストがマージされた後で、[StyleCI](https://styleci.io/)が自動的にスタイルを修正し、Laravelリポジトリへマージします。これによりコードスタイルではなく、貢献の内容へ集中できます。

<a name="code-of-conduct"></a>
## 行動規範

Laravelの行動規範はRubyの行動規範を基にしています。行動規範の違反はTaylor Otwell(taylor@laravel.com)へ報告してください。

<div class="content-list" markdown="1">

- 参加者は反対意見に寛容であること。
- 参加者は個人攻撃や個人的な発言の誹謗に陥らぬように言動に気をつけてください。
- 相手の言動を解釈する時、参加者は常に良い意図だと仮定してください。
- 嫌がらせと考えるのがふさわしい振る舞いは、寛容に扱いません。

</div>
