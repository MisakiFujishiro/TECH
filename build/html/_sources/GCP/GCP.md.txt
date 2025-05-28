# GCPについて
GCPは、Googleが提供するクラウドコンピューティングプラットフォーム。
GCPを使うことで、Google社内で使われているものと同じテクノロジーやサービスを利用することができる。

GCPはAWS同様柔軟性や拡張性、高速なインフラをクラウドから利用できる特徴があるとともに、データ分析やAIという分野において強みがある。

## ドキュメント
- [Google Cloud ドキュメント](https://cloud.google.com/docs?hl=ja)
- [コンソールログイン](https://cloud.google.com/?hl=ja)
- [プロダクト別 参考セッション](https://www.gc-solution-design-pattern.jp/product-sessions)
- [GCPSketchnote](https://github.com/priyankavergadia/GCPSketchnote)

## インフラ
[公式ドキュメント](https://cloud.google.com/about/locations?hl=ja#regions)

GCPでは、ゾーン（Zone）がインフラストラクチャの最小単位で、1つのゾーンは通常1つのデータセンターまたはその一部を指す。
複数のゾーンが集まってリージョン（Region）を構成し、リージョンは同一地理的エリア内で低レイテンシ通信が可能。
複数のリージョンをまとめた概念をロケーション（Location）と呼ぶことがあるが、これはより論理的な分類（例：US、EU、ASIAなどのマルチリージョン）。

![](../img/GCP/GCP/GCP_location.png)
[Google Cloud Fundamentals: Core Infrastructure 日本語版](https://www.coursera.org/learn/gcp-fundamentals-jp/lecture/40yXj/google-cloud-netutowaku)


|階層|単位|説明|例|
|:----|:----|:----|:----|
|ロケーション|Location|複数リージョンを論理的にまとめた地理的分類|us, eu, asia（BigQuery などで使用）|
|リージョン|Region|複数のゾーンを持つエリア。地理的に近接したゾーンで構成される|asia-northeast1（東京）、us-central1|
|ゾーン|Zone|物理的なデータセンター。1リージョン内に複数あり、可用性分散に利用|asia-northeast1-a, us-central1-c|

GCPでは約40のリージョンと121のゾーンが提供されている。


## 基本
### 構成
GCPにおけるリソースの階層構造は以下の通りで、組織、フォルダ、プロジェクト、リソースで構成される。
組織、フォルダー、プロジェクトにポリシーを設定することで、下位改装に対してポリシーを継承することができる。
また、リソースは必ず、一つのPJに所属する。

![](../img/GCP/GCP/GCP_kaisou.png)
[まずは知っておくべき IAM の基礎と最新の便利機能](https://services.google.com/fh/files/events/0224-infra-onair-seesion-1.pdf)

|階層|要素|説明|主な用途・備考|
|:----|:----|:----|:----|
|🏢 1|組織（Organization）|GCP全体の最上位階層。ドメイン（例：example.com）に紐づく1つの組織。|大企業・法人向け。すべてのリソースの親。|
|📁 2|フォルダ（Folder）|プロジェクトをグルーピングする中間階層。部門単位、環境（dev/stg/prd）単位などで利用。|IAMやポリシーの継承に使える。多階層可。|
|📦 3|プロジェクト（Project）|実際のGCPサービスを利用するための基本単位。課金、IAM、API有効化などすべてここで行う。|GCPリソースはすべてプロジェクトに属する。|
|🔧 4|リソース（Resource）|VM、Cloud Storage、Pub/Subなどの個別サービス。実体となるコンピューティングやストレージ。|すべてのリソースは1つのプロジェクトに属する。|


プロジェクトを作成すると、以下のプロジェクトID・プロジェクト番号・プロジェクト名が決まる。
それぞれの役割は以下。

|項目|説明|変更可否|ユニーク性|誰が指定するか|用途・備考|
|:----|:----|:----|:----|:----|:----|
|プロジェクトID (projectId)|GCP上で一意の文字列ID。ユーザーが一度指定（自動候補あり）し、以後変更不可。API・CLIでよく使われる。|❌変更不可|✅一意|ユーザーが指定（自動候補あり）|多くのAPIでこのIDを使ってプロジェクトを指定|
|プロジェクト番号 (projectNumber)|GCPが自動的に発行する内部的に一意な数値ID。IAMや課金設定で利用されることがある。|❌変更不可|✅一意|GCPが自動で発行|通常は意識されないが、一部APIやログで使われる|
|プロジェクト名 (name)|表示用の名称。一意である必要はなく、いつでも変更可能。UIでの識別に使われる。|✅変更可|❌重複可|ユーザーが自由に設定|管理コンソールなどの画面に表示されるラベル的な存在|

### リソース・ユーザーの管理
リソースやユーザーの管理にはIAM（Identity and Access Management）を利用する。
IAMを利用することで、`誰が`、`どういう操作を`、`何に対して`、`どういう条件で`を制御することができる。

![](../img/GCP/GCP/GCP_IAM.png)
[まずは知っておくべき IAM の基礎と最新の便利機能](https://services.google.com/fh/files/events/0224-infra-onair-seesion-1.pdf)


