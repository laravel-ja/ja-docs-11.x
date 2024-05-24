# Laravel Pint

- [イントロダクション](#introduction)
- [インストール](#installation)
- [Pintの実行](#running-pint)
- [Pintの設定](#configuring-pint)
    - [プリセット](#presets)
    - [ルール](#rules)
    - [ファイル／フォルダの除外](#excluding-files-or-folders)
- [継続的インテグレーション](#continuous-integration)
    - [GitHub Actions](#running-tests-on-github-actions)

<a name="introduction"></a>
## イントロダクション

[Laravel Pint](https://github.com/laravel/pint)（ピント：PHP+lint）は、ミニマリストのための主張を持ったPHPコードスタイルフィクサです。PintはPHP-CS-Fixer上に構築されており、あなたのコードスタイルがクリーンで一貫したものになるよう、シンプルにします。

Pintは、すべての新しいLaravelアプリケーションに自動的にインストールされますので、すぐに使い始めることができます。デフォルトで、Pintは設定を必要とせず、Laravelの主張を取り入れたコーディングスタイルに従い、コードスタイルの問題を修正します。

<a name="installation"></a>
## インストール

PintはLaravelフレームワークの最近のリリースに含まれているため、通常インストールは不要です。しかし、古いアプリケーションでは、Composer経由でLaravel Pintをインストールできます。

```shell
composer require laravel/pint --dev
```

<a name="running-pint"></a>
## Pintの実行

プロジェクトの`vendor/bin`ディレクトリにある、`pint`バイナリを起動し、Pintへコードスタイルの問題を修正するように指示できます。

```shell
./vendor/bin/pint
```

また、特定のファイルやディレクトリに対してPintを実行することもできます。

```shell
./vendor/bin/pint app/Models

./vendor/bin/pint app/Models/User.php
```

Pintは更新した全ファイルの完全なリストを表示します。Pintを起動する際に、`-v`オプションを指定すれば、Pintが行う変更についてさらに詳しく確認できます。

```shell
./vendor/bin/pint -v
```

もし、実際にファイルを変更せず、Pintにコードのスタイルエラーを検査させたい場合は、`--test`オプションを使用します。Pintはコードスタイルのエラーを見つけた場合、0以外の終了コードを返します。

```shell
./vendor/bin/pint --test
```

もし、Gitへコミットされていない変更のあるファイルだけをPintに修正させたい場合は、`--dirty` オプションを使用します。

```shell
./vendor/bin/pint --dirty
```

Pintにコードスタイルのエラーがあるファイルを修正させたいが、エラーを修正した場合にコードを0以外で終了させたい場合は、`--repair`オプションを使用します。

```shell
./vendor/bin/pint --repair
```

<a name="configuring-pint"></a>
## Pintの設定

前述したように、Pintは設定を一切必要としません。しかし、プリセットやルール、インスペクトフォルダをカスタマイズしたい場合は、プロジェクトのルートディレクトリに、`pint.json`ファイルを作成してください。

```json
{
    "preset": "laravel"
}
```

また、特定のディレクトリにある`pint.json`を利用したい場合は、Pintを起動する際に`--config`オプションを指定してください。

```shell
pint --config vendor/my-company/coding-style/pint.json
```

<a name="presets"></a>
### プリセット

プリセットは、コード内のスタイルの問題を修正するために使用するルールセットを定義しています。デフォルトでPintは、`laravel`プリセットを使用します。これは、Laravelの意見に基づいたコーディングスタイルに従って問題を修正するものです。しかし、Pintに`--preset`オプションを指定することで、別のプリセットも指定できます。

```shell
pint --preset psr12
```

お望みならば、プロジェクトの`pint.json`ファイルにプリセットを設定できます。

```json
{
    "preset": "psr12"
}
```

Pintが現在サポートしているプリセットは、`laravel`、`per`、`psr12`、`symfony` です。

<a name="rules"></a>
### ルール

ルールは、コードのスタイルに関する問題を修正するためにPintが使用するスタイルのガイドラインです。前述したように、プリセットはあらかじめ定義されたルールのグループであり、ほとんどのPHPプロジェクトに最適であるため、通常、含まれる個々のルールについて心配する必要はありません。

しかし、必要に応じて、`pint.json`ファイルで特定のルールの有効／無効を指定できます。

```json
{
    "preset": "laravel",
    "rules": {
        "simplified_null_return": true,
        "braces": false,
        "new_with_braces": {
            "anonymous_class": false,
            "named_class": false
        }
    }
}
```

Pintは、[PHP-CS-Fixer](https://github.com/FriendsOfPHP/PHP-CS-Fixer)上に構築しています。したがって、プロジェクトのコードスタイルの問題を修正するため、いずれかのルールを使用できます。[PHP-CS-Fixerの設定](https://mlocati.github.io/php-cs-fixer-configurator)を参照してください。

<a name="excluding-files-or-folders"></a>
### ファイル／フォルダの除外

デフォルトでPintは、プロジェクト内の`vendor`ディレクトリにあるものを除く、すべての`.php`ファイルを検査します。もし、より多くのフォルダを除外したい場合は、`exclude`設定オプションを使用して除外可能です。

```json
{
    "exclude": [
        "my-specific/folder"
    ]
}
```

もし、指定した名前のパターンに一致するファイルをすべて除外したい場合は、`notName`設定オプションを使用します。

```json
{
    "notName": [
        "*-my-file.php"
    ]
}
```

もし、ファイルの正確なパスを指定して除外したい場合は、`notPath`設定オプションを使用して除外できます。

```json
{
    "notPath": [
        "path/to/excluded-file.php"
    ]
}
```

<a name="continuous-integration"></a>
## 継続的インテグレーション

<a name="running-tests-on-github-actions"></a>
### GitHub Actions

Laravel Pintでプロジェクトのリントを自動化するには、[GitHub Actions](https://github.com/features/actions)を設定し、新しいコードをGitHubにプッシュするたびにPintを実行します。まず、**Settings > Actions > General > Workflow permissions**で、GitHub内のワークフローへ、"Read and write permissions"を付与してください。次に、`.github/workflows/lint.yml`ファイルを以下の内容で作成します。

```yaml
name: Fix Code Style

on: [push]

jobs:
  lint:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: true
      matrix:
        php: [8.3]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
          extensions: json, dom, curl, libxml, mbstring
          coverage: none

      - name: Install Pint
        run: composer global require laravel/pint

      - name: Run Pint
        run: pint

      - name: Commit linted files
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "Fixes coding style"
```
