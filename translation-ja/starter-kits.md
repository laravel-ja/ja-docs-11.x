# スターターキット

- [イントロダクション](#introduction)
- [Laravel Breeze](#laravel-breeze)
    - [インストール](#laravel-breeze-installation)
    - [BreezeとBlade](#breeze-and-blade)
    - [BreezeとLivewire](#breeze-and-livewire)
    - [BreezeとReact／Vue](#breeze-and-inertia)
    - [BreezeとNext.js／API](#breeze-and-next)
- [Laravel Jetstream](#laravel-jetstream)

<a name="introduction"></a>
## イントロダクション

新しいLaravelアプリケーションの構築をすぐに取りかかれるようするため、認証とアプリケーションのスターターキットを提供しています。これらのキットはアプリケーションのユーザーを登録および認証するために必要なルート、コントローラ、ビューを自動的にスカフォールドします。

皆さんがこうしたスターターキットを使用してくれるのは大歓迎ですが、これらは必須でありません。Laravelの真新しいコピーをインストールするだけで、自分自身のアプリケーションを自由にゼロから構築できます。いずれにせよ、みなさんが素晴らしいものを作り上げるのはわかっています！

<a name="laravel-breeze"></a>
## Laravel Breeze

[Laravel Breeze](https://github.com/laravel/breeze)は、ログイン、ユーザー登録、パスワードリセット、メール確認、パスワード確認など、すべての[認証機能](/docs/{{version}}/authentication)を最小かつシンプルにLaravelへ実装したものです。さらに、Breezeには、ユーザーが名前、電子メールアドレス、パスワードを更新できるシンプルな「プロファイル」ページが含まれています。

Laravel Breezeのデフォルトのビュー層は、[Tailwind CSS](https://tailwindcss.com)でスタイリングした、シンプルな[Bladeテンプレート](/docs/{{version}}/blade)で構成しています。さらに、Breezeには[Livewire](https://livewire.laravel.com)、または[Inertia](https://inertiajs.com)に基づいたスカフォールドオプションがあり、InertiaベースのスカフォールドにはVueまたはReactを使用できます。

<img src="https://laravel.com/img/docs/breeze-register.png">

#### Laravel Bootcamp

Laravelが初めての方は、気軽に[Laravel Bootcamp](https://bootcamp.laravel.com)に飛び込んでみてください。Laravel Bootcampでは、Breezeを使用して最初のLaravelアプリケーションを構築する手順が説明されています。LaravelとBreezeのすべてを知るには最適な方法です。

<a name="laravel-breeze-installation"></a>
### インストール

First, you should [create a new Laravel application](/docs/{{version}}/installation). If you create your application using the [Laravel installer](/docs/{{version}}/installation#creating-a-laravel-project), you will be prompted to install Laravel Breeze during the installation process. Otherwise, you will need to follow the manual installation instructions below.

If you have already created a new Laravel application without a starter kit, you may manually install Laravel Breeze using Composer:

```shell
composer require laravel/breeze --dev
```

After Composer has installed the Laravel Breeze package, you should run the `breeze:install` Artisan command. This command publishes the authentication views, routes, controllers, and other resources to your application. Laravel Breeze publishes all of its code to your application so that you have full control and visibility over its features and implementation.

`breeze:install`コマンドを実行すると、希望するフロントエンドスタックとテストフレームワークの入力を求められます：

```shell
php artisan breeze:install

php artisan migrate
npm install
npm run dev
```

<a name="breeze-and-blade"></a>
### BreezeとBlade

Breezeのデフォルト「スタック」はBladeスタックで、シンプルな[Bladeテンプレート](/docs/{{version}}/blade)を使用してアプリケーションのフロントエンドをレンダします。Bladeスタックをインストールするには、`breeze:install`コマンドを引数なしで実行し、Bladeフロントエンド・スタックを選択します。Breezeのスカフォールドがインストールされたら、アプリケーションのフロントエンドアセッツもコンパイルする必要があります。

```shell
php artisan breeze:install

php artisan migrate
npm install
npm run dev
```

次に、ウェブブラウザでアプリケーションの`/login`か`/register` URLへアクセスしてください。Breezeのすべてのルートは、`routes/auth.php`ファイルに定義してあります。

> [!NOTE]
> アプリケーションのCSSとJavaScriptのコンパイルについて詳しく知りたい方は、Laravelの[Viteドキュメント](/docs/{{version}}/vite#running-vite)を参照してください。

<a name="breeze-and-livewire"></a>
### BreezeとLivewire

Laravel Breezeは、[Livewire](https://livewire.laravel.com)のスカフォールドも提供しています。Livewireは、PHPだけでダイナミックでリアクティブなフロントエンドUIを構築する強力な方法です。

Livewireは、主にBladeテンプレートを使用し、VueやReactのようなJavaScript駆動のSPAフレームワークのシンプルな代替を探しているチームに最適です。

Livewireスタックを使用するには、`breeze:install` Artisanコマンドを実行する際に、Livewireフロントエンドスタックを選択します。Breezeのスカフォールドをインストールし終えたら、データベースのマイグレーションを実行します：

```shell
php artisan breeze:install

php artisan migrate
```

<a name="breeze-and-inertia"></a>
### BreezeとReact／Vue

Laravel Breezeは、[Inertia](https://inertiajs.com)フロントエンド実装により、ReactとVueを使用するスカフォールドも提供しています。Inertiaは、従来のサーバサイドルーティングとコントローラを使用して、モダンなシングルページのReactとVueのアプリケーションを構築することができます。

Inertiaはあなたへ、ReactやVueのフロントエンドのパワーと、Laravelの驚異的なバックエンドの生産性の組み合わせ、そして超早い[Vite](https://vitejs.dev)コンパイルを楽しませてくれます。Inertiaスタックを使用するには、`breeze:install` Artisanコマンドの実行時に、VueまたはReactフロントエンドスタックを選択します。

VueまたはReactフロントエンド・スタックを選択したら、Breezeのインストーラは、[Inertia SSR](https://inertiajs.com/server-side-rendering)またはTypeScriptのサポートを希望するかどうかの確認も行います。Breezeのスカフォールドをインストールしたら、アプリケーションのフロントエンドアセットもコンパイルする必要があります：

```shell
php artisan breeze:install

php artisan migrate
npm install
npm run dev
```

次に、ウェブブラウザでアプリケーションの`/login`か`/register` URLへアクセスしてください。Breezeのすべてのルートは、`routes/auth.php`ファイルに定義してあります。

<a name="breeze-and-next"></a>
### BreezeとNext.js／API

Laravel Breezeは、[Next](https://nextjs.org)や[Nuxt](https://nuxt.com)などのモダンなJavaScriptアプリケーションで認証するAPIもスカフォールドできます。これを使い始めるには、`breeze:install` Artisanコマンドを実行する時に、希望スタックとしてAPIスタックを指定します。

```shell
php artisan breeze:install

php artisan migrate
```

インストール中、Breezeはアプリケーションの`.env`ファイルへ、`FRONTEND_URL`環境変数を追加します。このURLは、JavaScriptアプリケーションのURLである必要があります。ローカル開発では、これは通常`http://localhost:3000`になります。さらに、`APP_URL`を`http://localhost:8000`に設定する必要があります。これは、`serve` Artisanコマンドで使用されるデフォルトのURLです。

<a name="next-reference-implementation"></a>
#### Next.jsリファレンス実装

ついに、このバックエンドとお好みのフロントエンドを組み合わせる準備ができました。BreezeフロントエンドのNextリファレンス実装は[GitHubで公開](https://github.com/laravel/breeze-next)しています。このフロントエンドはLaravelがメンテナンスし、Breezeが提供する従来のBladeスタックやInertiaスタックと同じユーザーインターフェイスを備えています。

<a name="laravel-jetstream"></a>
## Laravel Jetstream

Laravel Breezeは、Laravelアプリケーションを構築するためのシンプルで最小限の開始点を提供しますが、Jetstreamはより堅牢な機能と、追加のフロントエンドテクノロジースタックで、その機能を強化します。**Laravelを初めて使用する場合は、Laravel Jetstreamへ進む前に、Laravel Breezeで勘所を掴むことをおすめします。**

Jetstreamは、Laravelのために美しくデザインされたアプリケーションのスカフォールドを提供し、ログイン、登録、電子メール検証、二要素認証、セッション管理、Laravel Sanctum経由のAPIサポート、およびオプションのチーム管理を備えています。Jetstreamは、[Tailwind CSS](https://tailwindcss.com)を使用して設計しており、[Livewire](https://livewire.laravel.com)または[Inertia](https://inertiajs.com)駆動のフロントエンドスカフォールドから選択可能です。

Laravel Jetstreamをインストールするための完全なドキュメントは、[公式Jetstreamドキュメント](https://jetstream.laravel.com)にあります。
