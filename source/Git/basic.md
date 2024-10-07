# gitの基本知識
## コミット
gitにおけるコミットでは、実行のたびにファイルの内容をスナップショットとして全て記録する。
また、コミットでは以下のような情報が含まれている
- Commitリビジョン（SHA-１のハッシュ）
- Parents（1つ前のコミットのリビジョン）
- Author（コミットを作成した人）
- Committer（コミットを適用した人）
- Date（日付）
- コメント（コミット時のコメント）
- ファイルのスナップショット（ディレクトリツリー構造）

![](../img/Git/git_commit.png)


## ブランチ
ブランチとは、最新のコミットに対する「ラベル」付与である。  
ブランチを切るという操作は、現在の最新コミットに対してさらに別名を付与することを意味する。

![](../img/Git/git_branch_make.png)
[こわくない Git](https://kotas.hatenablog.jp/entry/2012/11/22/000046)

ここから、それぞれのブランチに対して新しいコミットとプッシュを行うと、ブランチごとにコミットを管理するため、枝分かれする。

![](../img/Git/git_branch_commit.png)
[こわくない Git](https://kotas.hatenablog.jp/entry/2012/11/22/000046)


## マージ
マージとは、異なるブランチに存在するコミットを別のブランチに統合することである。

![](../img/Git/git_merge.png)
[こわくない Git](https://kotas.hatenablog.jp/entry/2012/11/22/000046)

上記のmasterブランチに対して、Topicブランチをマージすると、Masterブランチの最新コミットに対してTopicの持っている3つのコミットを反映させる。

![](../img/Git/git_merge_2.png)
[こわくない Git](https://kotas.hatenablog.jp/entry/2012/11/22/000046)


結果として新たなコミットが作成されることになる。このコミットを*マージコミット*と呼ぶ。
![](../img/Git/git_merge_commit.png)
[こわくない Git](https://kotas.hatenablog.jp/entry/2012/11/22/000046)

### Fast-Forwardマージ
上記のマージをNon-Fast-Forwardマージと呼ぶ。
Fast-Forwardマージは基本的に非推奨で、Non-Fast-Forwardマージを利用する。

Fast-Forwardとは、ブランチが枝分かれしていない時にマージコミットを作成せずにブランチをマージする方法。

![](../img/Git/git_fast-forward.png)
[こわくない Git](https://kotas.hatenablog.jp/entry/2012/11/22/000046)

ブランチを作成してマージをした履歴が分かり辛くなるのでブランチ戦略や状況にもよるが基本的にはマージコミットが残るNon-Fast-Forwardのマージを実施する。

## Rebaseについて
Rebaseは異なるブランチの持つコミットを無理やり別ブランチの最新コミットから実施する作業。  
Rebaseをマージの代わりに異なるブランチの統合に利用するのは基本的に非推奨。マージを利用する。

![](../img/Git/git_rebase.png)
[こわくない Git](https://kotas.hatenablog.jp/entry/2012/11/22/000046)

結果として以下のような形になる。こちらに関してもFast-Forward同様にマージの履歴が残らないため、異なるブランチを統合したなどの履歴がわかり辛くなる。
![](../img/Git/git_rebase_2.png)
[こわくない Git](https://kotas.hatenablog.jp/entry/2012/11/22/000046)

## コンフリクトについて
masterからAブランチとBブランチを作成し、それぞれで改修を行うとする。

Aブランチをmasterにマージする際には、masterのブランチに対してブランチAのコミットを統合することになるので、スムーズにマージすることができる。

その後にBブランチをmasterマージする際には、masterには既にブランチAの内容が反映されている。元々のmasterから同じ箇所が修正されていると、ブランチAとブランチBどちらを採用すればいいかgitが理解することができないためコンフリクトが発生する。

![](../img/Git/git_conflict.png)
[サル先生のgit入門](https://backlog.com/ja/git-tutorial/)

## 参考
- [ATLASSIAN きっとできるGit](https://www.atlassian.com/ja/git)
- [サル先生のgit入門](https://backlog.com/ja/git-tutorial/)
- [こわくない Git](https://kotas.hatenablog.jp/entry/2012/11/22/000046)