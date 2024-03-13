# Laravel Mix

- [イントロダクション](#introduction)

<a name="introduction"></a>
## イントロダクション

[Laravel Mix](https://github.com/laravel-mix/laravel-mix) は、[Laracasts](https://laracasts.com) を作った、Jeffrey Wayが開発したパッケージで、Laravelアプリケーションのため、一般的なCSSとJavaScriptプリプロセッサを使い、[webpack](https://webpack.js.org)ビルド手順を定義する流暢なAPIを提供するものです。

言い換えると、Mixを使用すると、アプリケーションのCSSファイルとJavaScriptファイルを簡単にコンパイルして圧縮できます。シンプルなメソッドチェーンにより、アセットパイプラインを流暢に定義できます。例をご覧ください。

```js
mix.js('resources/js/app.js', 'public/js')
    .postCss('resources/css/app.css', 'public/css');
```

Webpackとアセットのコンパイルを使い始めようとして混乱し圧倒されたことがある方は、Laravel　Mixを気に入ってくれるでしょう。ただし、アプリケーションの開発に必ず使用する必要はありません。お好きなアセットパイプラインツールを使用するも、まったく使用しないのも自由です。

> [!NOTE]  
> 新しいLaravelのインストールから、Laravel MixをViteへ置き換えました。Mixのドキュメントは、[公式Laravel Mix](https://laravel-mix.com/)のウェブサイトをご覧ください。Viteへ切り替える場合は、[Vite移行ガイド](https://github.com/laravel/vite-plugin/blob/main/UPGRADE.md#migrating-from-laravel-mix-to-vite)を参照してください。
