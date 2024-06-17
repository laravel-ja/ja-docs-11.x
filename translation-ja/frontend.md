# フロントエンド

- [イントロダクション](#introduction)
- [PHPの使用](#using-php)
    - [PHPとBlade](#php-and-blade)
    - [Livewire](#livewire)
    - [スターターキット](#php-starter-kits)
- [Vue／Reactの使用](#using-vue-react)
    - [Inertia](#inertia)
    - [スターターキット](#inertia-starter-kits)
- [アセットの結合](#bundling-assets)

<a name="introduction"></a>
## イントロダクション

Laravelは、[ルーティング](/docs/{{version}}/routing)、[バリデーション](/docs/{{version}}/validation)、[キャッシュ](/docs/{{version}}/cache), [キュー](/docs/{{version}}/queues), [ファイルストレージ](/docs/{{version}}/filesystem)など、最新のウェブアプリケーション構築に必要となる全ての機能が提供されているバックエンド・フレームワークです。しかし、私たちはアプリケーションのフロントエンドを構築するための強力なアプローチを含む、美しいフルスタック体験を開発者に提供することも重要であると考えています。

Laravelでアプリケーションを構築する場合、フロントエンドの開発には主に２つの方法があります。どちらの方法を選択するかは、PHPを活用してフロントエンドを構築するか、VueやReactなどのJavaScriptフレームワークを使用するかにより決まります。以下では、こうした選択肢について説明し、あなたのアプリケーションに最適なフロントエンド開発のアプローチの情報を十分に得た上で、決定してもらえるようにします。

<a name="using-php"></a>
## PHPの使用

<a name="php-and-blade"></a>
### PHPとBlade

以前、ほとんどのPHPアプリケーションでは、リクエスト時にデータベースから取得したデータを表示するため、PHPの`echo`文を散りばめた単純なHTMLテンプレートを使用し、ブラウザでHTMLをレンダしていました。

```blade
<div>
    <?php foreach ($users as $user): ?>
        Hello, <?php echo $user->name; ?> <br />
    <?php endforeach; ?>
</div>
```

このHTML表示の手法を使う場合、Laravelでは[ビュー](/docs/{{version}}/views)と[Blade](/docs/{{version}}/blade)を使用して実現できます。Bladeは非常に軽量なテンプレート言語で、データの表示や反復処理などに便利な、短い構文を提供しています。

```blade
<div>
    @foreach ($users as $user)
        Hello, {{ $user->name }} <br />
    @endforeach
</div>
```

この方法でアプリケーションを構築する場合、フォーム送信や他のページへの操作は、通常サーバから全く新しいHTMLドキュメントを受け取り、ページ全体をブラウザで再レンダします。現在でも多くのアプリケーションは、シンプルなBladeテンプレートを使い、この方法でフロントエンドを構築するのが、最も適していると思われます。

<a name="growing-expectations"></a>
#### 高まる期待

しかし、Webアプリケーションに対するユーザーの期待値が高まるにつれ、多くの開発者がよりダイナミックなフロントエンドを構築し、洗練した操作性を感じてもらう必要性を感じてきています。そのため、VueやReactといったJavaScriptフレームワークを用いた、アプリケーションのフロントエンド構築を選択する開発者もいます。

一方で、自分が使い慣れたバックエンド言語にこだわる人たちは、そのバックエンド言語を主に利用しながら、最新のWebアプリケーションUIの構築を可能にするソリューションを開発しました。たとえば、[Rails](https://rubyonrails.org/) のエコシステムでは、[Turbo](https://turbo.hotwired.dev/) や[Hotwire]、[Stimulus](https://stimulus.hotwired.dev/) などのライブラリ作成が勢いづいています。

Laravelのエコシステムでは、主にPHPを使い、モダンでダイナミックなフロントエンドを作りたいというニーズから、[Laravel Livewire](https://livewire.laravel.com)と[Alpine.js](https://alpinejs.dev/)が生まれました。

<a name="livewire"></a>
### Livewire

[Laravel Livewire](https://livewire.laravel.com)は、VueやReactといったモダンなJavaScriptフレームワークで作られたフロントエンドのように、ダイナミックでモダン、そして生き生きとしたLaravelで動作するフロントエンドを構築するためのフレームワークです。

Livewireを使用する場合、レンダし、アプリケーションのフロントエンドから呼び出したり操作したりできるメソッドやデータを公開するUI部分をLivewire「コンポーネント」として作成します。例えば、シンプルな"Counter"コンポーネントは、以下のようなものです。

```php
<?php

namespace App\Http\Livewire;

use Livewire\Component;

class Counter extends Component
{
    public $count = 0;

    public function increment()
    {
        $this->count++;
    }

    public function render()
    {
        return view('livewire.counter');
    }
}
```

そして、このCounterに対応するテンプレートは、次のようになります。

```blade
<div>
    <button wire:click="increment">+</button>
    <h1>{{ $count }}</h1>
</div>
```

ご覧の通り、Livewireでは、Laravelアプリケーションのフロントエンドとバックエンドをつなぐ、`wire:click`のような新しいHTML属性を書けます。さらに、シンプルなBlade式を使って、コンポーネントの現在の状態をレンダできます。

多くの人にとって、LivewireはLaravelでのフロントエンド開発に革命を起こし、Laravelの快適さを保ちながら、モダンでダイナミックなWebアプリケーションを構築することを可能にしました。通常、Livewireを使用している開発者は、[Alpine.js](https://alpinejs.dev/)も利用して、ダイアログウィンドウのレンダなど、必要な場合にのみフロントエンドにJavaScriptを「トッピング」します。

Laravelに慣れていない方は、[ビュー](/docs/{{version}}/views)と[Blade](/docs/{{version}}/blade)の基本的な使い方に、まず慣れることをお勧めします。その後、公式の[Laravel Livewireドキュメント](https://livewire.laravel.com/docs)を参照し、インタラクティブなLivewireコンポーネントでアプリケーションを次のレベルに引き上げる方法を学んでください。

<a name="php-starter-kits"></a>
### スターターキット

PHPとLivewireを使ってフロントエンドを構築したい場合、BreezeまたはJetstreamの[スターターキット](/docs/{{version}}/starter-kits)を活用して、アプリケーション開発を迅速に開始できます。これらのスターターキットは、[Blade](/docs/{{version}}/blade)と[Tailwind](https://tailwindcss.com)を使って、アプリケーションのバックエンドおよびフロントエンドの認証フローに対するスカフォールドを生成するため、次回に開発する皆さんの大きなアイデアの構築を簡単に開始できます。

<a name="using-vue-react"></a>
## Vue／Reactの使用

LaravelやLivewireを使用してモダンなフロントエンドを構築することは可能ですが、多くの開発者はVueやReactなどのJavaScriptフレームワークのパワーを活用することをまだ好んでいます。このため、開発者はNPMを使い、利用可能なJavaScriptパッケージやツールの豊富なエコシステムを活用できます。

しかし、LaravelとVueやReactを組み合わせるには、クライアントサイドのルーティング、データハイドレーション、認証など、様々な複雑な問題を解決する必要があり、追加のツールがなければLaravelを使うことはできません。クライアントサイドのルーティングは、[Nuxt](https://nuxt.com/)や[Next](https://nextjs.org/)など主義的なVue／Reactフレームワークを取り入れて簡素化されていることが多いですが、Laravelなどのバックエンドフレームワークとこれらのフレームワークを組み合わせる場合、データのハイドレートと認証で、複雑で面倒な問題が残ります。

さらに、開発者は２つの別々のコードリポジトリを管理することになり、しばしばメンテナンス、リリース、デプロイメントを両方のリポジトリにまたがって調整する必要が起きます。こうした問題は克服できないものではありませんが、アプリケーションを開発する上で、生産的で楽しい方法とは思えません。

<a name="inertia"></a>
### Inertia

幸運にも、Laravelは両方の世界の最良を提供しています。[Inertia](https://inertiajs.com)は、LaravelアプリケーションとモダンなVueまたはReactフロントエンドの間のギャップを埋めるもので、VueやReactを使って本格的でモダンなフロントエンドを構築しながら、ルーティング、データハイドレート、認証にLaravelルートとコントローラを活用できます。すべて単一のコードリポジトリ内で行えます。このアプローチではどちらのツールの能力も損なうことなく、LaravelとVue／Reactの両能力を享受することができます。

LaravelアプリケーションにInertiaをインストールしたあとで、通常通りにルートとコントローラを記述します。しかし、コントローラからBladeテンプレートを返すのではなく、Inertiaページを返すようにします。

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\User;
use Inertia\Inertia;
use Inertia\Response;

class UserController extends Controller
{
    /**
     * 指定ユーザーのプロフィールページを表示
     */
    public function show(string $id): Response
    {
        return Inertia::render('Users/Profile', [
            'user' => User::findOrFail($id)
        ]);
    }
}
```

Inertiaページは、VueまたはReactコンポーネントに対応し、通常アプリケーションの`resources/js/Pages`ディレクトリに格納します。`Inertia::render`メソッドにより、ページに与えたデータは、ページコンポーネントの"props"をハイドレートするために使用されます。

```vue
<script setup>
import Layout from '@/Layouts/Authenticated.vue';
import { Head } from '@inertiajs/vue3';

const props = defineProps(['user']);
</script>

<template>
    <Head title="User Profile" />

    <Layout>
        <template #header>
            <h2 class="font-semibold text-xl text-gray-800 leading-tight">
                Profile
            </h2>
        </template>

        <div class="py-12">
            Hello, {{ user.name }}
        </div>
    </Layout>
</template>
```

ご覧の通り、Inertiaを使うことで、Laravelを使ったバックエンドと、JavaScriptを使ったフロントエンドの間の軽量なブリッジが提供され、フロントエンドを構築する際にVueやReactのパワーをフル活用することができます。

#### サーバサイドレンダ

アプリケーションでサーバサイドレンダが必要なため、Inertiaに飛び込むことを躊躇している方も、安心してください。Inertiaは[サーバサイドレンダリングサポート](https://inertiajs.com/server-side-rendering)を提供しています。また、[Laravel Forge](https://forge.laravel.com)を介してアプリケーションをデプロイする場合、Inertiaのサーバサイドレンダリングプロセスが常に実行されていることを簡単に確認できます。

<a name="inertia-starter-kits"></a>
### スターターキット

InertiaとVue／Reactを使用してフロントエンドを構築したい場合は、BreezeまたはJetstreamの[スターターキット](/docs/{{version}}/starter-kits#breeze-and-inertia)を活用して、アプリケーション開発を迅速に開始できます。これらのスターターキットは、Inertia、Vue／React、[Tailwind](https://tailwindcss.com)、[Vite](https://vitejs.dev)を使用してアプリケーションのバックエンドとフロントエンドでの認証フローに必要なスカフォールディングを生成するため、次に開発する皆さんの大きなアイデアをすぐに構築開始できます。

<a name="bundling-assets"></a>
## アセットの結合

BladeとLivewire、Vue／ReactとInertiaのどちらを使用してフロントエンドを開発するにしても、アプリケーションのCSSをプロダクション用アセットへバンドルする必要があるでしょう。もちろん、VueやReactでアプリケーションのフロントエンドを構築することを選択した場合は、コンポーネントをブラウザ用JavaScriptアセットへバンドルする必要があります。

Laravelは、デフォルトで[Vite](https://vitejs.dev)を利用してアセットをバンドルします。Viteは、ローカル開発において、ビルドが非常に速く、ほぼ瞬時のホットモジュール交換（HMR）を提供しています。[スターターキット](/docs/{{version}}/starter-kits)を含むすべての新しいLaravelアプリケーションでは、`vite.config.js`ファイルがあり、軽量なLaravel Viteプラグインがロードされ、LaravelアプリケーションでViteを楽しく使用できるようにしています。

LaravelとViteを使い始める最も早い方法は、[Laravel Breeze](/docs/{{version}}/starter-kits#laravel-breeze)を使ってアプリケーションの開発を始めることです。これは、フロントエンドとバックエンドで認証に必要なスカフォールドを生成してくれます。アプリケーション開発をロケットスタートする、最もシンプルなスターターキットです。

> [!NOTE]
> LaravelでViteを活用するための詳細なドキュメントは、[アセットバンドルとコンパイルに関する専用のドキュメント](/docs/{{version}}/vite)を参照してください。
