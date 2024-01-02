# Reveal.jsとは
Reveal.jsとは、ウェブベースのプレゼンテーションフレームワークであり、ウェブ上にスライドを構築することができる。


# Reveal.jsの初期設定
## ディレクトリの作成と移動
最初に、プレゼンテーションのプロジェクト用に新しいディレクトリを作成し、そのディレクトリに移動。

```bash
mkdir ~/github_pages_reveal/hello_reveal
```

## reveal.jsのクローン
GitHubからreveal.jsのリポジトリをクローン。
これにより、revealに必要なファイルとフォルダ構造がローカル環境にコピーされる。

```bash
git clone https://github.com/hakimel/reveal.js.git
```

## npmのインストール
reveal.jsはnpmを使用して依存関係を管理しているため、npmがシステムにインストールされていることを確認する。
npmはNode.jsのパッケージマネージャーで、Node.jsをインストールすることで利用できるようになるので、次のコマンドを使用してNode.js（およびnpm）をインストール。

```bash
brew install node
```

## 依存関係のインストール
reveal.jsの`npm i`コマンドでカレントディレクトリのpackage.jsonを利用して、依存関係をインストールする
```
cd reveal.js && npm i
```

## Markdownのための設定

## github pagesの設定
作成したhtmlをgithub pagesで公開することができる。
また、githubで公開することで、どの環境からも修正を加えることができる。

### リポジトリ設定
github pagesで公開を行う場合は、publicとして作成を行う。Readmeは作らなくて良い。

初期設定を行い、リモートとローカルにリポジトリを作成する。


### github pagesの設定
[sphinx](https://misakifujishiro.github.io/TECH/Sphinx/quickstart.html#github-pages)の設定と同様


## トラブルシューティング

以下のエラーが出た場合は
```
Fetching submodules
  /usr/bin/git submodule sync --recursive
  /usr/bin/git -c protocol.version=2 submodule update --init --force --depth=1 --recursive
  Error: fatal: No url found for submodule path 'reveal.js' in .gitmodules
  Error: The process '/usr/bin/git' failed with exit code 128
```

以下の内容を`.gitmodules`という名前にして、ルートディレクトリに保存してpushする
```
[submodule "reveal.js"]
    path = reveal.js
    url = https://github.com/hakimel/reveal.js.git
```






これが完了してから、サーバーを立ち上げると、サンプルスライドがlocalhost:8080で起動している
```
http://localhost:8080
```



# Reveal.jsの基本的な利用方法