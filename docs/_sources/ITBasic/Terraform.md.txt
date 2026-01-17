# Terraform
## Terraformとは？
Terraform は Infrastructure as Code（IaC）ツール の一つで、  
クラウド上のインフラ構成を「宣言的」にコードで管理するための仕組みである。

Terraform の最大の特徴は次の点にある。
- GCP / AWS / Azure の API の違いを考えないで記述する   
    →API 呼び出しは Terraform Provider が吸収する
- 「どう変更するか」ではなく「どうあるべきか」を記述する
- 現在の状態と理想の状態との差分から、必要な操作を自動で決定する

Terraform に書くのは ゴールの姿 であり、作成順序・削除順序・更新手順は Terraform が判断する。
つまり Terraform は、
- 「今のインフラ」  
- 「こうなっていてほしいインフラ」  

この 差分を埋めるオーケストレーター である。


## 状態ファイル（terraform.tfstate）
`状態ファイル（terraform.tfstate）とは、Terraform が信じている「現実のスナップショット」`である。

Terraform は次の前提で動く。
- 状態ファイルに書かれている内容は正しい
- その状態とコードとの差分を解消する

状態ファイルには、次のような情報が含まれる。
- Terraform の resource と、実クラウド上のリソースの対応関係
- 実リソースの ID（self_link など）
- Terraform が管理している属性の現在値
- 依存関係の解決結果

重要なのは、状態ファイルがないと、Terraform は「どれがどのリソースか」分からなくなる。
例えば
- 状態ファイルと実際のインフラがズレる
- 状態ファイルを手で編集する
- 手動でクラウド側を変更する

これらを行うと、Terraform は 存在しないものを消そうとしたり、すでにあるものを作ろうとしたりする。

## Terraformのライフサイクル

Terraform では、リソースの変更は次のどちらかになる。
- in-place update（同じリソースを更新）
- replace（削除して作り直し）

この判定は Terraform provider（クラウド仕様） によって決まる。

GCPの例で言うと、instance Templateなどはreplaceのリソースであり、   
`変更(replace) = 新規作成 + 切り替え + 旧リソース削除`  
上記のようなことが発生する。

### create_before_destroy
replaceされるimmutableなリソースが他のリソースから参照されている場合、
Terraformで更新をかけると、依存関係により参照先のリソースを削除するタイミングでエラーが発生してしまう。

Terraformにはlifecycleメタ引数があり、create_before_destoryをTrueとすることで、リソースの削除と作成を制御できる。

この設定をすることで、
- 先に新しいリソースを作成
- 参照を切り替え
- 最後に古いリソースを削除

という順序になる。