# Sphinxとは
python製のドキュメント生成ツールでreSTやmarkdownで書かれたファイルをHTMLなどに変換する。

github Pagesと連携することで、github上でドキュメントを公開できる。



# Sphinxの初期設定
## Sphinxのインストール
python環境がインストールされていることが前提となる。
以下のコマンドでインストールされているか確認して、されていなければインストールする。
```sh
$ python -V
```

pipでsphinxをinstall
```sh
$ pip install sphinx
```


## Sphinxのクイックスタート
クイックスタートコマンドを実行し、SphinxのProjectを作成する。対話形式での設定例は以下。
```sh
$ sphinx-quickstart

> ソースディレクトリとビルドディレクトリを分ける（y / n） [n]: y
> プロジェクト名: project
> 著者名（複数可）: author
> プロジェクトのリリース []: 
> プロジェクトの言語 [en]: ja
```

以下の動作確認コマンド実行する。
```sh 
$ make html
```

`build/html/index.html`のindex.htmlを開いてサンプルページが正しく作成されているか確認をする



## ローカルホストでのホスティング
毎回、make htmlを実行するのは手間なので、autobuildをさせる。
autobuildを設定することで、localホストでホスティングされ、ファイルをsaveするたびに自動更新される。

ライブラリのインストール
```sh
$ pip install sphinx-autobuild
```

autobuildの実行は以下コマンド。PORT_NUMを指定しない場合は8080で起動する。
```sh
$ sphinx-autobuild -b html source build/html --port [PORT_NUM]
```


## Markdownのための設定
Sphinxをドキュメント化するにあたって、Markdownを利用するためには以下の設定が必要。

myst-parserをインストール
```sh
$ pip install --upgrade myst-parser
```

インストール後に、`source/conf.py`について、以下のように修正を追加する

変更前
```sh
# -- General configuration ---------------------------------------------------
# https://www.sphinx-doc.org/en/master/usage/configuration.html#general-configuration

extensions = []

templates_path = ['_templates']
```

変更後
```sh
# -- General configuration ---------------------------------------------------
# https://www.sphinx-doc.org/en/master/usage/configuration.html#general-configuration

extensions = [
    'myst_parser'
]

source_suffix = {
    '.rst': 'restructuredtext',
    '.md': 'markdown',
}

templates_path = ['_templates']
```

## テーマの変更
Sphinxで作成するドキュメントにはいくつかのテーマが準備されている。
conf.pyを修正することによって、簡単にテーマを変更することが可能。

`readthedocs`というテーマが見やすくておすすめ。

sphinx_rtd_themeのインストール
```sh
$ pip install sphinx_rtd_theme
```

インストール後に、`source/conf.py`について、以下のように修正を追加する


修正前
```sh
# -- Options for HTML output -------------------------------------------------
# https://www.sphinx-doc.org/en/master/usage/configuration.html#options-for-html-output

html_theme = 'alabaster'
html_static_path = ['_static']
```

修正後
```sh
# -- Options for HTML output -------------------------------------------------
# https://www.sphinx-doc.org/en/master/usage/configuration.html#options-for-html-output

import sphinx_rtd_theme
html_theme = 'sphinx_rtd_theme'
html_theme_path = [sphinx_rtd_theme.get_html_theme_path()]
```




## github pagesの設定
作成したhtmlをgithub pagesで公開することができる。
また、githubで公開することで、どの環境からも修正を加えることができる。

### リポジトリ設定
github pagesで公開を行う場合は、publicとして作成を行う。

初期設定を行い、リモートとローカルにリポジトリを作成する。

### jekelly対応
GitHub PagesではJekyllという静的サイトジェネレーターを使って、MarkdownやHTMLファイルを受け取り、整ったウェブサイトの形に変換する。

しかし、SphinxではJekyllによって画面が正しく表記されなくなってしまう。
そこで、Jekyllを無効化させるための`.nojekyll`というからのファイルを配置する必要がある。このファイルがあると、GitHub PagesはJekyllによる処理をスキップし、ファイルをそのままの形でホスティングします。つまり、Jekyllの変換機能を使わずに純粋な静的ホスティングを利用できるようになります。

`docs/.nojekyll`という配置になるようにdocs/配下に移動して、以下を実行する。
```sh
touch .nojekyll
```


### 出力先の変更
github pagesでは、`docs/index.html`というパスを固定で参照しにいくので、docsというフォルダにも出力する必要がある。

`Makefile`を修正して、docsにも出力する設定を行う。

修正前
```sh
# Catch-all target: route all unknown targets to Sphinx using the new
# "make mode" option.  $(O) is meant as a shortcut for $(SPHINXOPTS).
%: Makefile
	@$(SPHINXBUILD) -M $@ "$(SOURCEDIR)" "$(BUILDDIR)" $(SPHINXOPTS) $(O)
```

修正後
```sh
# Catch-all target: route all unknown targets to Sphinx using the new
# "make mode" option.  $(O) is meant as a shortcut for $(SPHINXOPTS).
%: Makefile
	@$(SPHINXBUILD) -M $@ "$(SOURCEDIR)" "$(BUILDDIR)" $(SPHINXOPTS) $(O)

html: Makefile
	@$(SPHINXBUILD) -b html "$(SOURCEDIR)" "docs" $(SPHINXOPTS) $(O)
```


### github pagesの設定
リポジトリの`/Setting/Pages`から、以下のように設定。

![](../img/Sphinx/github_pages_setting.png)

設定後、SphinxをPushするとGitHub Pageが作成され、URLからアクセス可能になる。

![](../img/Sphinx/github_pages.png)



# Sphinxの基本的な利用方法
## index.rst
作成するファイルの階層構造を`source/index.rst`で記述しておく。

- 【カテゴリー名】: 紫色枠部分のカテゴリー
- 【maxdepth】: 青色枠部分の各ファイルのセクションをどこまで表示するかの設定
- 【caption】：黄色枠部分のセクションを区切る部分

```
【カテゴリー名】
====================================

.. toctree::
   :maxdepth: 1
   :caption: Reveal:

   Reveal/reveal.md
```

![](../img/Sphinx/github_pages_sample.png)

## mdファイル
index.rstで定義したものと同じファイル構成でmdファイルを作成する。

