# アセットの構築（Vite）

- [イントロダクション](#introduction)
- [インストールと準備](#installation)
  - [Nodeのインストール](#installing-node)
  - [ViteとLaravelプラグインのインストール](#installing-vite-and-laravel-plugin)
  - [Viteの設定](#configuring-vite)
  - [スクリプトとスタイルの読み込み](#loading-your-scripts-and-styles)
- [Viteの実行](#running-vite)
- [JavaScriptの操作](#working-with-scripts)
  - [エイリアス](#aliases)
  - [Vue](#vue)
  - [React](#react)
  - [Inertia](#inertia)
  - [URL処理](#url-processing)
- [スタイルシートの操作](#working-with-stylesheets)
- [Bladeとルートの操作](#working-with-blade-and-routes)
  - [Viteによる静的アセットの処理](#blade-processing-static-assets)
  - [保存時の再描写](#blade-refreshing-on-save)
  - [エイリアス](#blade-aliases)
- [アセットの事前フェッチ](#asset-prefetching)
- [ベースURLのカスタマイズ](#custom-base-urls)
- [環境変数](#environment-variables)
- [テスト時のVite無効](#disabling-vite-in-tests)
- [サーバサイドレンダリング(SSR)](#ssr)
- [Scriptとstyleタグ属性](#script-and-style-attributes)
  - [コンテンツセキュリティポリシー（CSP)ノンス](#content-security-policy-csp-nonce)
  - [サブリソース完全性(SRI)](#subresource-integrity-sri)
  - [任意の属性](#arbitrary-attributes)
- [高度なカスタマイズ](#advanced-customization)
  - [開発サーバURLの修正](#correcting-dev-server-urls)

<a name="introduction"></a>
## イントロダクション

[Vite](https://vitejs.dev)は、非常に高速な開発環境を提供してくれる、コードを本番用に構築する最新のフロントエンド・ビルド・ツールです。Laravelでアプリケーションを構築する場合、通常、Viteを使用してアプリケーションのCSSとJavaScriptファイルを本番環境用のアセットへ構築することになります。

Laravelは、開発および実働用アセットをロードするため、公式プラグインとBladeディレクティブを提供し、Viteをシームレスに統合しています。

> [!NOTE]
> Laravel Mixを実行していますか？新しいLaravelのインストールでは、Laravel MixをViteへ置き換えました。Mixのドキュメントは、[Laravel Mix](https://laravel-mix.com/)のウェブサイトをご覧ください。Viteに切り替えたい場合は、[移行ガイド](https://github.com/laravel/vite-plugin/blob/main/UPGRADE.md#migrating-from-laravel-mix-to-vite)を参照してください。

<a name="vite-or-mix"></a>
#### ViteとLaravel Mixの選択

Viteへ移行する前、新しいLaravelアプリケーションは、アセットをバンドルする際に[webpack](https://webpack.js.org/)で動作する[Mix](https://laravel-mix.com/)を使用していました。Viteは、リッチなJavaScriptアプリケーションを構築する際に、より速く、より生産的な体験を提供することに重点を置いています。[Inertia](https://inertiajs.com) のようなツールで開発したものを含め、シングルページアプリケーション（SPA）を開発している場合、Viteは完璧にフィットするでしょう。

Viteは、[Livewire](https://livewire.laravel.com)を使用したものを含む、JavaScriptを「ふりかけ」程度に使った従来のサーバサイドレンダリングアプリケーションでもうまく機能します。しかし、Laravel Mixがサポートしている、JavaScriptアプリケーションで直接参照されていない任意のアセットをビルドにコピーする機能など、いくつかの機能が欠落しています。

<a name="migrating-back-to-mix"></a>
#### Mixへ戻す

Vite scaffoldingを使用して新しいLaravelアプリケーションを開始したが、Laravel Mixとwebpackへ戻る必要があるのですか？大丈夫です。[ViteからMixへの移行に関する公式ガイド](https://github.com/laravel/vite-plugin/blob/main/UPGRADE.md#migrating-from-vite-to-laravel-mix)を参照してください。

<a name="installation"></a>
## インストールと準備

> [!NOTE]
> 以下のドキュメントでは、Laravel Viteプラグインを手作業でインストールし、設定する方法について説明しています。しかし、Laravelの[スターターキット](/docs/{{version}}/starter-kits)には、すでにこのスカフォールドがすべて含まれており、LaravelとViteを始める最速の方法を用意しています。

<a name="installing-node"></a>
### Nodeのインストール

You must ensure that Node.js (16+) and NPM are installed before running Vite and the Laravel plugin:

```sh
node -v
npm -v
```

NodeとNPMの最新版は、[Node公式サイト](https://nodejs.org/en/download/)からグラフィカルインストーラを使って簡単にインストールできます。また、[Laravel Sail](/docs/{{version}}/sail)を使用している場合は、SailからNodeとNPMを呼び出せます。

```sh
./vendor/bin/sail node -v
./vendor/bin/sail npm -v
```

<a name="installing-vite-and-laravel-plugin"></a>
### ViteとLaravelプラグインのインストール

Laravelを新規にインストールすると、アプリケーションのディレクトリ構造のルートに`package.json`ファイルができます。デフォルトの`package.json`ファイルは、ViteとLaravelプラグインを使い始めるために必要なものを既に含んでいます。アプリケーションのフロントエンドの依存関係は、NPM経由でインストールできます。

```sh
npm install
```

<a name="configuring-vite"></a>
### Viteの設定

Viteは、プロジェクトルートの`vite.config.js`ファイルで設定します。また、アプリケーションが必要とする他のプラグイン、例えば、`@vitejs/plugin-vue`や`@vitejs/plugin-react`をインストールすることもできます。

Laravel Viteプラグインでは、アプリケーションのエントリーポイントを指定する必要があります。これらは、JavaScriptまたはCSSファイルであり、TypeScript、JSX、TSX、Sassなどのプリプロセス言語が含まれます。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel([
            'resources/css/app.css',
            'resources/js/app.js',
        ]),
    ],
});
```

Inertiaを使用したアプリケーションを含むSPAを構築する場合、ViteはCSSエントリポイントなしで最適に動作します。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel([
            'resources/css/app.css', // [tl! remove]
            'resources/js/app.js',
        ]),
    ],
});
```

代わりに、JavaScriptでCSSをインポートする必要があります。通常、これはアプリケーションの `resources/js/app.js` ファイルで行います。

```js
import './bootstrap';
import '../css/app.css'; // [tl! add]
```

また、Laravelプラグインは複数のエントリーポイントに対応し、[SSRエントリーポイント](#ssr)などの高度な設定オプションにも対応しています。

<a name="working-with-a-secure-development-server"></a>
#### セキュアな開発サーバの取り扱い

ローカル開発用Webサーバが、HTTPSでアプリケーションを提供している場合、Vite開発用サーバの接続に問題が発生することがあります。

[Laravel Herd](https://herd.laravel.com)を使用していて、サイトをセキュアにしている場合、または[Laravel Valet](/docs/{{version}}/valet)を使用していて、アプリケーションに対して[secureコマンド](/docs/{{version}}/valet#securing-sites)を実行している場合、Laravel Viteプラグインが自動的に検出し、生成したTLS証明書を使用します。

アプリケーションのディレクトリ名と一致しないホストを使用してサイトをセキュアにしている場合、アプリケーションの`vite.config.js`ファイルにより、ホストを手作業で指定してください。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            detectTls: 'my-app.test', // [tl! add]
        }),
    ],
});
```

他のWebサーバを使用する場合は、信頼できる証明書を生成し、その生成した証明書を使用するようにViteを手作業で設定する必要があります。

```js
// ...
import fs from 'fs'; // [tl! add]

const host = 'my-app.test'; // [tl! add]

export default defineConfig({
    // ...
    server: { // [tl! add]
        host, // [tl! add]
        hmr: { host }, // [tl! add]
        https: { // [tl! add]
            key: fs.readFileSync(`/path/to/${host}.key`), // [tl! add]
            cert: fs.readFileSync(`/path/to/${host}.crt`), // [tl! add]
        }, // [tl! add]
    }, // [tl! add]
});
```

もし、あなたのシステムで信頼できる証明書を生成できない場合は、[`@vitejs/plugin-basic-ssl`プラグイン](https://github.com/vitejs/vite-plugin-basic-ssl)をインストールし、設定してください。信頼できない証明書を使用する場合は、`npm run dev`コマンドを実行する際にコンソールの"Local"リンクをたどり、ブラウザからVite開発サーバが出す証明書の警告を受け入れる必要があります。

<a name="configuring-hmr-in-sail-on-wsl2"></a>
#### WSL2上のSailの中で開発サーバを実行する

Windows Subsystem for Linux 2(WSL2)上の[Laravel Sail](/docs/{{version}}/sail)内で、Vite 開発サーバを実行する場合は、ブラウザが開発サーバと通信できるように`vite.config.js`ファイルへ以下の設定を追加する必要があります。

```js
// ...

export default defineConfig({
    // ...
    server: { // [tl! add:start]
        hmr: {
            host: 'localhost',
        },
    }, // [tl! add:end]
});
```

開発サーバ実行中に、ファイルの変更がブラウザへ反映されない場合は、Viteの[`server.watch.usePolling`オプション](https://vitejs.dev/config/server-options.html#server-watch)の設定も必要です。

<a name="loading-your-scripts-and-styles"></a>
### スクリプトとスタイルの読み込み

Viteのエントリーポイントを設定したら、アプリケーションのルートテンプレートの`<head>`へ`@vite()` Bladeディレクティブを追加し、参照できるようになります。

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

JavaScriptでCSSをインポートする場合は、JavaScriptのエントリーポイントのみを記載するだけです。

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite('resources/js/app.js')
</head>
```

`@vite` ディレクティブは、Vite開発サーバを自動的に検出し、Viteクライアントを注入してホットモジュール置換を有効にします。ビルドモードでは、このディレクティブはインポートしたCSSを含む、コンパイル済みのバージョン管理しているアセットを読み込みます。

必要であれば、`@vite`ディレクティブを呼び出す際に、コンパイル済みアセットのビルドパスを指定することもできます。

```blade
<!doctype html>
<head>
    {{-- 指定するビルドパスは、publicパスからの相対パス --}}

    @vite('resources/js/app.js', 'vendor/courier/build')
</head>
```

<a name="inline-assets"></a>
#### インラインのアセット

時には、アセットでバージョン管理したURLへリンクするのではなく、アセットの素のコンテンツを含める必要があるかもしれません。例えば、HTMLコンテンツをPDFジェネレータに渡すときに、アセットコンテンツを直接ページに含める必要があるかもしれません。`Vite`ファサードが提供する`content`メソッドを使用して、Viteアセットのコンテンツを出力できます。

```blade
@use('Illuminate\Support\Facades\Vite')

<!doctype html>
<head>
    {{-- ... --}}

    <style>
        {!! Vite::content('resources/css/app.css') !!}
    </style>
    <script>
        {!! Vite::content('resources/js/app.js') !!}
    </script>
</head>
```

<a name="running-vite"></a>
## Viteの実行

Viteを起動する方法は2つあります。`dev`コマンドで開発サーバを起動するのは、ローカルで開発する際に便利です。開発サーバは、ファイルの変更を自動的に検出し、開いているブラウザウィンドウへ即座に反映させます。

もしくは、`build`コマンドを実行すると、アプリケーションのアセットをバージョン付けして構築し、本番環境にデプロイできる状態にします。

```shell
# Viteを開発サーバで実行する
npm run dev

# 実働用にアセットをバンドルし、バージョン付けする
npm run build
```

WSL2上の[Sail](/docs/{{version}}/sail)で開発サーバを動作させている場合、いくつかの[追加設定](#configuring-hmr-in-sail-on-wsl2)オプションが必要です。

<a name="working-with-scripts"></a>
## JavaScriptの操作

<a name="aliases"></a>
### エイリアス

デフォルトでLaravelプラグインは、アプリケーションのアセットを便利にインポートできるように、共用エイリアスを提供します。

```js
{
    '@' => '/resources/js'
}
```

設定ファイル`vite.config.js`に独自のエイリアスを追加すれば、`'@'`エイリアスを上書きできます。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel(['resources/ts/app.tsx']),
    ],
    resolve: {
        alias: {
            '@': '/resources/ts',
        },
    },
});
```

<a name="vue"></a>
### Vue

[Vue](https://vuejs.org/)フレームワークを使用してフロントエンドを構築したい場合は、`@vitejs/plugin-vue`プラグインもインストールする必要があります。

```sh
npm install --save-dev @vitejs/plugin-vue
```

続いて、`vite.config.js`設定ファイルの中で、プラグインをインクルードしてください。LaravelでVueプラグインを使用する場合、いくつかの追加オプションが必要です。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
    plugins: [
        laravel(['resources/js/app.js']),
        vue({
            template: {
                transformAssetUrls: {
                    // Vueプラグインは、Single File Componentsで
                    // 参照する場合、アセットのURLをLaravelのWebサーバを
                    // 指すように書き換えます。
                    // これを`null`に設定すると、Laravelプラグインは
                    // アセットURLをViteサーバを指すように書き換えます。
                    base: null,

                    // Vueプラグインは、絶対URLを解析し、ディスク上のファイルへの
                    // 絶対パスとして扱います。
                    // これを`false`に設定すると、絶対URLはそのままになり、
                    // 期待通りにpublicディレクトリの中で、アセットへ参照できます。
                    includeAbsolute: false,
                },
            },
        }),
    ],
});
```

> [!NOTE]
> Laravelの[スターターキット](/docs/{{version}}/starter-kits)には、すでに適切なLaravel、Vue、Viteの構成が含まれています。Laravel、Vue、Viteを最速で使い始めるには、[Laravel Breeze](/docs/{{version}}/starter-kits#breeze-and-inertia)をチェックしてください。

<a name="react"></a>
### React

[React](https://reactjs.org/)フレームワークを使用してフロントエンドを構築したい場合、`@vitejs/plugin-react`プラグインもインストールする必要があります。

```sh
npm install --save-dev @vitejs/plugin-react
```

その後、`vite.config.js`設定ファイル中で、プラグインをインクルードしてください。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import react from '@vitejs/plugin-react';

export default defineConfig({
    plugins: [
        laravel(['resources/js/app.jsx']),
        react(),
    ],
});
```

JSXを含むすべてのファイルの拡張子を確実に、`.jsx`または`.tsx`にし、必要であれば[上記のように](#configuring-vite)、エントリポイントの更新を忘れないでください。

更に、既存の`@vite`ディレクティブと一緒に、追加で`@viteReactRefresh` Bladeディレクティブをインクルードする必要があります。

```blade
@viteReactRefresh
@vite('resources/js/app.jsx')
```

`@viteReactRefresh`ディレクティブは、`@vite`ディレクティブの前に呼び出す必要があります。

> [!NOTE]
> Laravelの[スターターキット](/docs/{{version}}/starter-kits)には、すでに適切なLaravel、React、Viteの設定が含まれています。Laravel、React、Viteを最速で始めるには、[Laravel Breeze](/docs/{{version}}/starter-kits#breeze-and-inertia) をチェックしてください。

<a name="inertia"></a>
### Inertia

Laravel Viteプラグインは、Inertiaページコンポーネントを解決するのに便利な `resolvePageComponent` 関数を提供しています。以下はVue 3で使用するヘルパの例ですが、Reactなど他のフレームワークでもこの関数を利用することができます。

```js
import { createApp, h } from 'vue';
import { createInertiaApp } from '@inertiajs/vue3';
import { resolvePageComponent } from 'laravel-vite-plugin/inertia-helpers';

createInertiaApp({
  resolve: (name) => resolvePageComponent(`./Pages/${name}.vue`, import.meta.glob('./Pages/**/*.vue')),
  setup({ el, App, props, plugin }) {
    return createApp({ render: () => h(App, props) })
      .use(plugin)
      .mount(el)
  },
});
```

Viteのコード分割機能をInertiaで使用する場合は、[アセットの事前フェッチ](#asset-prefetching)を設定することをお勧めします。

> [!NOTE]
> Laravelの[スターターキット](/docs/{{version}}/starter-kits)には、すでに適切なLaravel、Inertia、Viteの構成が含まれています。Laravel、Inertia、Viteを最速で始めるには、[Laravel Breeze](/docs/{{version}}/starter-kits#breeze-and-inertia) をチェックしてください。

<a name="url-processing"></a>
### URL処理

Viteを使用して、アプリケーションのHTML、CSS、JSアセットを参照する場合、いくつか考慮すべき注意点があります。まず、絶対パスでアセットを参照した場合、Viteはそのアセットをビルドに含めません。したがって、そのアセットがパブリックディレクトリで利用可能であることを確認する必要があります。[専用CSSエントリポイント](#configuring-vite)を使用する場合は、絶対パスの使用は避けるべきです。その理由は開発中、ブラウザはこうしたパスをあなたの公開ディレクトリからではなく、CSSをホストしているVite開発サーバから読み込もうとするからです。

相対パスで参照する場合、参照するファイルからの相対パスであることを覚えておく必要があります。相対パスで参照されたアセットは、Viteによって書き直され、バージョン付けされ、バンドルされます。

以下のようなプロジェクト構成を考えてみましょう。

```nothing
public/
  taylor.png
resources/
  js/
    Pages/
      Welcome.vue
  images/
    abigail.png
```

以下の例は、Viteが相対URLと絶対URLをどのように扱うかを示しています。

```html
<!-- このアセットをViteは扱わず、バンドルへ含まれない -->
<img src="/taylor.png">

<!-- このアセットは書き換えられ、バージョン付けし、バンドルへ含まれる -->
<img src="../../images/abigail.png">
```

<a name="working-with-stylesheets"></a>
## スタイルシートの操作

ViteのCSSサポートは、[Viteドキュメント](https://vitejs.dev/guide/features.html#css) で詳しく説明されています。[Tailwind](https://tailwindcss.com)のようなPostCSSプラグインを使用している場合、プロジェクトのルートに`postcss.config.js`ファイルを作成すると、Vite が自動的にそれを適用してくれます。

```js
export default {
    plugins: {
        tailwindcss: {},
        autoprefixer: {},
    },
};
```

> [!NOTE]
> Laravelの[スターターキット](/docs/{{version}}/starter-kits)には、最初から適切なTailwind、PostCSS、Viteの構成を用意しています。また、スターターキットを使わずにTailwindとLaravelを使いたい場合は、[LaravelのためのTailwindインストールガイド](https://tailwindcss.com/docs/guides/laravel)をチェックしてください。

<a name="working-with-blade-and-routes"></a>
## Bladeとルートの操作

<a name="blade-processing-static-assets"></a>
### Viteによる静的アセットの処理

JavaScriptやCSSのアセットを参照する場合、Viteは自動的に処理し、バージョン付けを行います。また、Bladeベースのアプリケーションを構築する場合、Bladeのテンプレート内だけで参照する静的なアセットもViteで処理し、バージョン付け可能です。

これを実現するには、アプリケーションのエントリポイントで静的アセットをインポートすることにより、Viteにあなたの資産を認識させる必要があります。例えば、`resources/images`に格納しているすべての画像と、`resources/fonts`に保存しているすべてのフォントを処理してバージョン付けする場合、アプリケーションの`resources/js/app.js`エントリーポイントへ以下を追加してください。

```js
import.meta.glob([
  '../images/**',
  '../fonts/**',
]);
```

これで`npm run build`の実行時、Viteはこれらのアセットを処理するようになります。そして、Bladeテンプレートでは`Vite::asset`メソッドを使用してこれらのアセットを参照でき、指定したアセットのバージョン付けを含むURLを返します。

```blade
<img src="{{ Vite::asset('resources/images/logo.png') }}">
```

<a name="blade-refreshing-on-save"></a>
### 保存時の再描写

Bladeを用いる従来のサーバサイドレンダリングによりアプリケーションを構築している場合、アプリケーション内のビューファイルを変更したときに、自動でブラウザを再ロードすることにより、Viteは開発ワークフローを改善します。これを使用するには、`refresh`オプションを`true`へ指定するだけです。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            refresh: true,
        }),
    ],
});
```

`refresh`オプションが`true`の場合、`npm run dev`実行時に以下のディレクトリへファイルを保存すると、ブラウザでページがフルリフレッシュされます。

- `app/Livewire/**`
- `app/View/Components/**`
- `lang/**`
- `resources/lang/**`
- `resources/views/**`
- `routes/**`

[Ziggy](https://github.com/tighten/ziggy)を利用して、アプリケーションのフロントエンドでルートリンクを生成する場合、`routes/**`ディレクトリを監視すると便利です。

これらのデフォルトパスがニーズに合わない場合、監視するパスリストを独自に指定できます。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            refresh: ['resources/views/**'],
        }),
    ],
});
```

Laravel Viteプラグインの内部では、[`vite-plugin-full-reload`](https://github.com/ElMassimo/vite-plugin-full-reload)パッケージを使用しており、この機能の動作を微調整するために高度な設定オプションを用意しています。このレベルのカスタマイズが必要な場合は、`config`定義を指定してください。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            refresh: [{
                paths: ['path/to/watch/**'],
                config: { delay: 300 }
            }],
        }),
    ],
});
```

<a name="blade-aliases"></a>
### エイリアス

JavaScriptアプリケーションでは、定期的に参照するディレクトリに[エイリアス](#aliases)を作成することが一般的です。しかし、`Illuminate\Support\Facades\Vite`クラスの`macro`メソッドを使用して、Bladeで使用するエイリアスを作成することもできます。通常、「マクロ」は、[サービスプロバイダ](/docs/{{version}}/providers)の`boot`メソッド内で定義する必要があります。

    /**
     * 全アプリケーションサービスの初期起動処理
     */
    public function boot(): void
    {
        Vite::macro('image', fn (string $asset) => $this->asset("resources/images/{$asset}"));
    }

一度マクロを定義したら、テンプレート内で呼び出せます。例えば、上で定義した`image`マクロを使用して、`resources/images/logo.png`にあるアセットを参照してみましょう。

```blade
<img src="{{ Vite::image('logo.png') }}" alt="Laravel Logo">
```

<a name="asset-prefetching"></a>
## アセットの事前フェッチ

Viteのコード分割機能を使用してSPAを構築する場合、必要なアセットは各ページナビゲーションで取得されます。この動作は、UIレンダリングの遅延につながる可能性があります。フロントエンドフレームワークの選択でこれが問題となる場合、Laravelでは、最初のページロード時にアプリケーションのJavaScriptとCSSアセットを前もってプリフェッチする機能があります。

[サービスプロバイダ](/docs/{{version}}/providers)の`boot`メソッド内で`Vite::prefetch`メソッドを呼び出すことにより、Laravelへアセットを事前にプリフェッチするように指示できます。

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Vite;
use Illuminate\Support\ServiceProvider;

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
        Vite::prefetch(concurrency: 3);
    }
}
```

上記の例では、アセットがプリフェッチされ、各ページロード時に最大で`3`つの同時ダウンロードが行われます。アプリケーションのニーズに合わせて同時実行数を変更したり、アプリケーションがすべてのアセットを一度にダウンロードする場合は同時実行数の制限を設けないようにしたりすることもできます。

```php
/**
 * 全アプリケーションサービスの初期起動処理
 */
public function boot(): void
{
    Vite::prefetch();
}
```

プリフェッチはデフォルトで、[ページ*load*イベント](https://developer.mozilla.org/ja/docs/Web/API/Window/load_event)が発生したときに開始します。プリフェッチを開始するタイミングをカスタマイズしたい場合は、Viteがリッスンするイベントを指定できます。

```php
/**
 * 全アプリケーションサービスの初期起動処理
 */
public function boot(): void
{
    Vite::prefetch(event: 'vite:prefetch');
}
```

上記に提示したコードでは、`window`オブジェクト上の`vite:prefetch`イベントを手作業でディスパッチしたときにプリフェッチを開始します。例えば、ページがロードされてから3秒後にプリフェッチを開始させることができます：

```html
<script>
    addEventListener('load', () => setTimeout(() => {
        dispatchEvent(new Event('vite:prefetch'))
    }, 3000))
</script>
```

<a name="custom-base-urls"></a>
## ベースURLのカスタマイズ

ViteでコンパイルしたアセットをCDN経由など、アプリケーションとは別のドメインにデプロイする場合は、アプリケーションの`.env`ファイル内に`ASSET_URL`環境変数を指定する必要があります。

```env
ASSET_URL=https://cdn.example.com
```

アセットURLを設定すると、アセットを指す全ての書き換えられるURLの先頭に、設定した値が付きます。

```nothing
https://cdn.example.com/build/assets/app.9dce8d17.js
```

[絶対URLはViteで書き直されない](#url-processing)ため、プリフィックスが付かないことを覚えておいてください。

<a name="environment-variables"></a>
## 環境変数

アプリケーションの`.env`ファイルに、`VITE_` というプレフィックスを付ける環境変数を設置することで、それらの環境変数をJavaScriptへ注入できます。

```env
VITE_SENTRY_DSN_PUBLIC=http://example.com
```

注入された環境変数は、`import.meta.env`オブジェクトを介してアクセスできます。

```js
import.meta.env.VITE_SENTRY_DSN_PUBLIC
```

<a name="disabling-vite-in-tests"></a>
## テスト時のVite無効

LaravelのVite統合は、テストの実行中にアセットを解決しようとするので、Vite開発サーバを実行するか、アセットをビルドする必要があります。

テスト中にViteをモックしたい場合は、Laravelの`TestCase`クラスを拡張するすべてのテストで利用できる、`withoutVite`メソッドを呼び出してください。

```php tab=Pest
test('without vite example', function () {
    $this->withoutVite();

    // ...
});
```

```php tab=PHPUnit
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_without_vite_example(): void
    {
        $this->withoutVite();

        // ...
    }
}
```

すべてのテストで Vite を無効にしたい場合は、ベースとなる`TestCase`クラスの`setUp`メソッドから、`withoutVite`メソッドを呼び出すとよいでしょう。

```php
<?php

namespace Tests;

use Illuminate\Foundation\Testing\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    protected function setUp(): void// [tl! add:start]
    {
        parent::setUp();

        $this->withoutVite();
    }// [tl! add:end]
}
```

<a name="ssr"></a>
## サーバサイドレンダリング(SSR)

Laravel Viteプラグインを使用すると、Viteでサーバサイドレンダリングを簡単に設定できます。まず、SSRエントリーポイントを`resources/js/ssr.js`に作成し、Laravelプラグインに設定オプションを渡して、エントリーポイントを指定します。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: 'resources/js/app.js',
            ssr: 'resources/js/ssr.js',
        }),
    ],
});
```

SSRエントリポイントの再構築を忘れないようにするために、アプリケーションの`package.json`にある"build"スクリプトを拡張して、SSRビルドを作成することをお勧めします。

```json
"scripts": {
     "dev": "vite",
     "build": "vite build" // [tl! remove]
     "build": "vite build && vite build --ssr" // [tl! add]
}
```

最後に、SSRサーバの構築と起動のため、以下のコマンドを実行してください。

```sh
npm run build
node bootstrap/ssr/ssr.js
```

[SSRをInertia](https://inertiajs.com/server-side-rendering)で使用している場合、代わりに`inertia:start-ssr` Artisanコマンドを使用してSSRサーバを起動してください。

```sh
php artisan inertia:start-ssr
```

> [!NOTE]
> Laravelの[スターターキット](/docs/{{version}}/starter-kits)には、すでに適切なLaravel、Inertia SSR、Viteの構成が含まれています。Laravel、Inertia SSR、Viteを最速で使い始めるため、[Laravel Breeze](/docs/{{version}}/starter-kits#breeze-and-inertia) をチェックしてください。

<a name="script-and-style-attributes"></a>
## Scriptとstyleタグ属性

<a name="content-security-policy-csp-nonce"></a>
### コンテンツセキュリティポリシー（CSP）ノンス

[コンテンツセキュリティポリシー](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)の一環として、スクリプトやスタイルタグに、[`nonce`属性](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/nonce)を含めたい場合は、カスタム[ミドルウェア](/docs/{{version}}/middleware)の`useCspNonce`メソッドを使用して、ノンスを生成・指定できます。

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Vite;
use Symfony\Component\HttpFoundation\Response;

class AddContentSecurityPolicyHeaders
{
    /**
     * 受診リクエストの処理
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        Vite::useCspNonce();

        return $next($request)->withHeaders([
            'Content-Security-Policy' => "script-src 'nonce-".Vite::cspNonce()."'",
        ]);
    }
}
```

`useCspNonce`メソッドが呼び出だされると、Laravelは生成するすべてのscriptタグとstyleタグへ、`nonce`属性を自動的に含めます。

Laravelの[スターターキット](/docs/{{version}}/starter-kits)に含まれる[Ziggy `@route` directive](https://github.com/tighten/ziggy#using-routes-with-a-content-security-policy)など、別の場所でノンスを指定する必要がある場合、`cspNonce`メソッドを使用して取得できます。

```blade
@routes(nonce: Vite::cspNonce())
```

Laravelに使わせたいノンスが既にある場合は、`useCspNonce`メソッドへ、そのノンスを渡してください。

```php
Vite::useCspNonce($nonce);
```

<a name="subresource-integrity-sri"></a>
### サブリソース完全性（SRI）

Viteマニフェストにアセット用の`integrity`ハッシュが含まれている場合、Laravelは自動的に`integrity`属性を生成するスクリプトとスタイルタグに追加し、[サブリソース完全性](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)を強要します。Viteはデフォルトでは、`integrity`ハッシュをマニフェストに含みませんが、 [`vite-plugin-manifest-sri`](https://www.npmjs.com/package/vite-plugin-manifest-sri) NPMプラグインをインストールすれば、これを有効にできます。

```shell
npm install --save-dev vite-plugin-manifest-sri
```

このプラグインは、`vite.config.js`ファイルで有効にします。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import manifestSRI from 'vite-plugin-manifest-sri';// [tl! add]

export default defineConfig({
    plugins: [
        laravel({
            // ...
        }),
        manifestSRI(),// [tl! add]
    ],
});
```

必要であれば、完全性ハッシュを見つけることができるマニフェスト・キーもカスタマイズできます。

```php
use Illuminate\Support\Facades\Vite;

Vite::useIntegrityKey('custom-integrity-key');
```

この自動検出を完全に無効にする場合は、`useIntegrityKey`メソッドへ`false`を渡します。

```php
Vite::useIntegrityKey(false);
```

<a name="arbitrary-attributes"></a>
### 任意の属性

もし、スクリプトタグやスタイルタグに追加の属性、例えば [`data-turbo-track`](https://turbo.hotwired.dev/handbook/drive#reloading-when-assets-change)属性を含める必要がある場合は、`useScriptTagAttributes`と`useStyleTagAttributes`メソッドで指定できます。通常、このメソッドは [サービスプロバイダ](/docs/{{version}}/providers)から呼び出す必要があります。

```php
use Illuminate\Support\Facades\Vite;

Vite::useScriptTagAttributes([
    'data-turbo-track' => 'reload', // 属性の値を指定
    'async' => true, // 値を指定しない属性
    'integrity' => false, // 本来含まれる属性を除外
]);

Vite::useStyleTagAttributes([
    'data-turbo-track' => 'reload',
]);
```

条件付きで属性を追加する必要がある場合、アセットのソースパスとそのURL、マニフェストのチャンク、およびマニフェスト全体を受け取るコールバックを渡してください。

```php
use Illuminate\Support\Facades\Vite;

Vite::useScriptTagAttributes(fn (string $src, string $url, array|null $chunk, array|null $manifest) => [
    'data-turbo-track' => $src === 'resources/js/app.js' ? 'reload' : false,
]);

Vite::useStyleTagAttributes(fn (string $src, string $url, array|null $chunk, array|null $manifest) => [
    'data-turbo-track' => $chunk && $chunk['isEntry'] ? 'reload' : false,
]);
```

> [!WARNING]
> Vite開発サーバが起動している間は、`$chunk`と`$manifest`引数は、`null`になります。

<a name="advanced-customization"></a>
## 高度なカスタマイズ

LaravelのViteプラグインは、ほとんどのアプリケーションで動作するように、合理的な規約をはじめから使用しています。しかし、時にはViteの動作をカスタマイズする必要が起きるかもしれません。追加のカスタマイズオプションを有効にするための、`@vite` Bladeディレクティブの代わりに使用可能な、以下のメソッドとオプションを提供しています。

```blade
<!doctype html>
<head>
    {{-- ... --}}

    {{
        Vite::useHotFile(storage_path('vite.hot')) // 「ホット」なファイルのカスタマイズ
            ->useBuildDirectory('bundle') // ビルドディレクトリのカスタマイズ
            ->useManifestFilename('assets.json') // マニフェストファイル名のカスタマイズ
            ->withEntryPoints(['resources/js/app.js']) // エントリポイントの指定
            ->createAssetPathsUsing(function (string $path, ?bool $secure) { // Customize the backend path generation for built assets...
                return "https://cdn.example.com/{$path}";
            })
    }}
</head>
```

そして、`vite.config.js`ファイル内でも、同じ設定を指定する必要があります。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            hotFile: 'storage/vite.hot', // 「ホット」なファイルのカスタマイズ
            buildDirectory: 'bundle', // ビルドディレクトリのカスタマイズ
            input: ['resources/js/app.js'], // エントリポイントの指定
        }),
    ],
    build: {
      manifest: 'assets.json', // マニフェストファイル名のカスタマイズ
    },
});
```

<a name="correcting-dev-server-urls"></a>
### 開発サーバURLの修正

Viteエコシステム内のプラグインのいくつかは、フォワードスラッシュで始まるURLを常にVite開発サーバを指すと仮定しています。しかし、Laravelとの統合の関係上、これは好ましくありません。

例えば、`vite-imagetools`プラグインは、Viteがあなたのリソースを提供しているとき、以下のようなURLを出力します。

```html
<img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520">
```

`vite-imagetools`プラグインは、出力するURLがViteによりインターセプトされ、そのプラグインが`/@imagetools` から始まるすべてのURLを処理することを期待しています。このような挙動を期待するプラグインを使用している場合、手作業でURLを修正する必要があります。これは、`vite.config.js`ファイルの`transformOnServe`オプションを使用して実現できます。この例は、生成されたコード内における`/@imagetools`の全出現箇所で、開発サーバのURLを前へ追加します。

この例では、生成したコード内で、`/@imagetools`のすべての出現の前に開発サーバのURLを追加しています。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import { imagetools } from 'vite-imagetools';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            transformOnServe: (code, devServerUrl) => code.replaceAll('/@imagetools', devServerUrl+'/@imagetools'),
        }),
        imagetools(),
    ],
});
```

これで、Viteがアセットを配信する間、Viteの開発サーバを指すURLが出力されます。

```html
- <img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520"><!-- [tl! remove] -->
+ <img src="http://[::1]:5173/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520"><!-- [tl! add] -->
```
