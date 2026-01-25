# CloudOps
CloudOpsとは、Google Cloudにおける運用系サービス群のブランド。
具体的には以下のサービスが含まれる。

|分類|実体のサービス名|
|:----|:----|
|ログ|Cloud Logging|
|メトリクス|Cloud Monitoring|
|トレース|Cloud Trace|
|プロファイル|Cloud Profiler|
|エラー|Error Reporting|



## Cloud Logging
GCPのログ収集・管理サービス。各種サービスから集まるログの保管・分析・モニタリングを一手に引き受ける。
[Cloud Logging](https://cloud.google.com/logging#documentation)

本記事では以下の流れで整理
- ログはどうやって集まるのか？ – 各サービスのログ収集の仕組みと、まずすべてのログが Cloud Logging に集約される流れ
- どのサービス由来かをどう見分けるのか？ – ログエントリに付与される resource.type と ラベルによる自動分類
- どうやってアラートにつなげるのか？ – Cloud Monitoring と連携したログベースのメトリクスとアラートポリシー設定

### ログの収集
Cloud Logging は GCP 上で発生したすべてのログを一箇所に集約する仕組み。Compute Engine（GCE）や GKE、Cloud Run、Cloud Functions といった主要サービスのログは、特別な設定をしなくても自動的に Cloud Logging に収集される。

例えば GKE では、各ノードにデプロイされたログ収集エージェント（Ops エージェント）がコンテナの標準出力・エラー出力を監視し、付加情報（メタデータ）をつけて Cloud Logging に送信する。
そのため、アプリケーションがコンテナ上で stdout や stderr に出力したログは自動的に Logging に蓄積される。Cloud Run や Cloud Functions も同様に、サービスの実行環境が標準出力などに書き出したログを自動で収集し Cloud Logging に送信する。

一方、オンプレミスやカスタムアプリケーションのログ、あるいはファイルに直接出力しているログなど自動収集の対象外となるログについても、Cloud Logging に送信する方法が用意されている。例えば GCEのVM 上で動くアプリケーションのログファイルは Ops エージェントを導入してログパイプラインを設定することで収集可能。
また、任意のプログラムから Cloud Logging API を使ってログを書き込むこともできる。
こうした方法で「それ以外のログもすべてまず Cloud Logging に集める」ことが可能となる。

### Cloud Logging のログエントリ構造（全体像）
Cloud Loggingで収集されたログで出力される項目は以下が主要なものになる。

|大分類（所属）|中分類|項目名|意味 / 役割|出力例|補足|
|:----|:----|:----|:----|:----|:----|
|LogEntry メタ情報|識別|insertId|ログ一意ID|1abcd1234|重複排除用|
| |時刻|timestamp|ログ発生時刻|2025-01-01T12:00:00Z|遅延送信でも保持|
| |重要度|severity|ログレベル|INFO / ERROR|Monitoringと直結|
| |分類|logName|ログの論理名|projects/xxx/logs/stdout|stdout / stderr / custom|
|発生元リソース|種別|resource.type|どのサービス由来か|k8s_container|最重要フィールド|
| |識別|resource.labels|リソース固有情報|cluster / pod / container|自動付与|
|補助メタ情報|ラベル|labels|追加のキー情報|env=prod|検索・整理用|
|ログ本文（Payload）|非構造|textPayload|プレーンテキスト|order failed|grep的検索|
| |構造化|jsonPayload|JSONログ本文|{ "order_id": 123 }|分析・集計向き|
|分散トレース|Trace|trace|Trace ID|projects/.../traces/...|Cloud Trace連携|
| |Span|spanId|スパンID|a1b2c3|マイクロサービス|
|HTTP 情報|L7|httpRequest|HTTP詳細|status / latency|LB / Run / Ingress|

jso形式の例では以下となる
```
{
    "insertId": "1abcd1234",
    "timestamp": "2025-01-01T12:00:00.123456Z",
    "severity": "ERROR",
    "logName": "projects/my-project/logs/stdout",

    "resource": {
        "type": "k8s_container",
        "labels": {
            "project_id": "my-project",
            "location": "asia-northeast1",
            "cluster_name": "prod-cluster",
            "namespace_name": "default",
            "pod_name": "orders-api-7c9f9d6d5b-abcde",
            "container_name": "orders-api"
        }
    },

    "labels": {
        "compute.googleapis.com/resource_name": "gke-prod-cluster-default-pool-12345"
    },

    "textPayload": "order failed: database timeout",
    "jsonPayload": {
    "order_id": 123,
    "user_id": "u-999",
    "error": "timeout"
    }

    "trace": "projects/my-project/traces/105445aa7843bc8bf206b120001000",
    "spanId": "a1b2c3d4e5f6",

    "httpRequest": {
        "requestMethod": "POST",
        "requestUrl": "/orders",
        "status": 500,
        "latency": "1.234s",
        "userAgent": "curl/8.0"
    }
}
```


### resource.typeとlabel
Cloud Logging に全ログが集まった後、重要になるのが各ログエントリに付加される **リソースタイプ (resource.type)** と **ラベル (labels)** 。
これらは「そのログがどのサービスのどのリソースから来たか」を示す情報で、GCP が自動で付与するメタデータであり、アプリケーション開発者や運用者が個別に設定する必要はなく、GCP側でログごとに適切なリソースタイプとラベルが割り当てらる。
[監視対象のリソースとサービス](https://cloud.google.com/logging/docs/api/v2/resource-list)

代表的なresource.typeとlabelを整理すると以下。

|サービス|resource.type|階層|対応する resource.labels|
|:----|:----|:----|:----|
|GKE|k8s_container|Cluster|cluster_name|
| | |Namespace|namespace_name|
| | |Pod|pod_name|
| | |Container|container_name|
| | |Location|location|
|Cloud Run（Service）|cloud_run_revision|Service|service_name|
| | |Revision|revision_name|
| | |Location|location|
|Cloud Run（Job）|cloud_run_job|Job|job_name|
| | |Execution|execution_name|
| | |Task|task_index|
| | |Location|location|
|Cloud Functions|cloud_function|Function|function_name|
| | |Location|location|


これらの情報のおかげで、「このログはどのプロジェクトのどのサービス・どのリソースから出力されたものか」を機械的に判別で可能。
クラスタ名やサービス名、関数名などが自動でログに紐づくため、後述するフィルタで「どのクラスタのログだけを見る」「どのCloud Runサービスのエラーのみ抽出する」といった絞り込みが簡単に行える。


### フィルタリング
Cloud Logging では、ログの検索やログシンク（BigQuery / Pub/Sub などへのエクスポート）を行う際に
ログフィルタ（クエリ） を記述して対象ログを指定する。

Cloud Logging のクエリでは、改行は論理積（AND）として扱われる。
```
resource.type="k8s_container"
resource.labels.namespace_name="backend"
severity>=ERROR
```
これは次の意味を持つ：
- GKE コンテナログのうち、
- Namespace が backend で、
- かつ ERROR 以上のログだけを表示する

#### フィルタの記述例
Cloud Run Job のログから特定エラーIDを含むものを抽出
```
resource.type="cloud_run_job"
textPayload:"e.abc."
```
- Cloud Run Job の実行ログのみ
- メッセージ本文に e.abc. を含むログを検索

特定のエラーIDだけを除外する
```
resource.type="cloud_run_job"
textPayload:"e.abc."
-textPayload:"e.abc.001"
```
- e.abc. を含むログは対象
- ただし e.abc.001 を含むものは除外


#### 構造化ログ（jsonPayload）の検索
アプリケーションが JSON 形式でログを出力している場合、フィールド単位での検索が可能になる。
```
jsonPayload.order_id=123
jsonPayload.error_code="TIMEOUT"
```
- フィールド検索
- インデックスが効く
- 集計・メトリクス化に最適

#### フィルタ演算子の整理
Cloud Logging のフィルタでは、
「どのフィールドに対して検索しているか」 によって使える演算子と意味が変わる。

特に混乱しやすいのが、
- = と =~
- :（payload検索）
の使い分け。

結論としては以下を意識する。
- 構造化されたフィールド（resource / labels / jsonPayloadのフィールド）
    - → = / >= / =~ を使う
- ログ本文（文字列）
    - → : を使う
- 除外したい条件
    - → 先頭に - を付ける

|用途|対象フィールド|書き方|意味|補足|
|:----|:----|:----|:----|:----|
|完全一致|resource / labels / jsonPayload.<field>|=|値が完全一致|インデックスが効く|
|範囲比較|severity / timestamp|>=|大小比較|ERROR以上など|
|正規表現|resource / labels / jsonPayload.<field>|=~|正規表現一致|prefix検索で多用|
|文字列包含|textPayload / jsonPayload.<field>|:|部分一致（含む）|grep的検索|
|除外条件|すべて|-条件|NOT 条件|組み合わせ可|

### アラート
Cloud Logging 単体では、ログを閲覧・検索・フィルタリングすることができるが、ログ発の通知（アラート） 機能は持っていない。
そこで活用するのが Cloud Monitoring との連携。
Logging と Monitoring は統合されたオブザーバビリティ基盤の一部であり、Logging のデータをもとに Monitoring 側でアラートを発報できる。

ログによるアラートを実現する一般的な方法は「ログベースの指標（ログベースメトリクス）」を作り、それに対してアラートポリシーを設定するという手順。
[ログベースのアラート ポリシーを構成する](https://cloud.google.com/logging/docs/alerting/log-based-alerts?hl=ja)


具体的には次の3ステップによる設定を行う。
- ログフィルタを作成  
    - まずアラートのトリガーとしたいログを抽出するフィルタ条件を決める
        - 先の例でいえば、Cloud Run Job のログから特定のエラーID (e.abc.) を含み、かつ無視してよいID (e.abc.001) は除外するといったフィルタを作る
        - 例: resource.type="cloud_run_job" textPayload:"e.abc." -textPayload:"e.abc.001"
        - このフィルタは後続のメトリクスとアラートで共通して利用
- ログベースメトリクスを作成  
    - 作成したフィルタを元にカウンタ型のメトリクスを定義
        - メトリクスの種類は「Counter（カウンタ）」を選択し、フィルタにマッチするログエントリを1件検出するごとにカウントが1増えるように設定
        - こうして Logging 内でログに基づくメトリクスが生成されると、Cloud Monitoring 側からそのメトリクスの時系列データを参照できるようになる
        - 例えば上記フィルタにマッチするログが1件発生すればメトリクスが「1」を記録し、発生しない間は「0」のまま推移するようなイメージ
- アラートポリシーを設定  
    - Cloud Monitoring のアラートポリシーで、上述のログベースメトリクスに対ししきい値条件を設定
        - 例えば「直近5分間で該当メトリクスの値が1以上になったら（= 該当ログが1件でも発生したら）アラート発報」という条件を作成
        - しきい値を 0→1 に超えたタイミングで即アラート発生、のように設定すれば、対象ログが出力された事実をリアルタイムに捉えて通知可能
        - 通知先には Slack や Email、PagerDuty など様々なチャネルを指定可能


### ログシンク：エクスポート
Cloud Logging に蓄積されたログは、必要に応じて他のストレージやシステムにエクスポート（転送）することも可能。
これを実現するのが ログシンク (Log Sink) 機能。
Cloud Logging に蓄積されたログは、Logs Router によって評価され、ユーザーが定義したログシンク（Log Sink）の条件に一致したログを他のストレージやシステムへエクスポートできる。

ログシンクは、プロジェクト・フォルダ・組織のいずれのレベルでも作成可能であり、ログの保存先を追加・分岐させたい場合に利用される。

主なログシンクの出力先

|出力先|主な用途|利用シーン例|
|:----|:----|:----|
|BigQuery|分析・検索・長期保持|1年以上分のアプリケーションログを蓄積し、SQLでエラー傾向や利用状況を分析する|
|Cloud Storage|アーカイブ・監査|規制対応のためログを削除せず長期保存する。圧縮やライフサイクル管理でコスト最適化|
|Pub/Sub|リアルタイム連携|ログをストリーミングで外部SIEMや自社解析基盤（Datadog / Splunk 等）へ送信|
|Logging バケット|論理分離|フォルダ単位・用途別にログバケットを分け、閲覧権限や保持期間を分離|

※Logging バケットはCloud Logging内の内蔵のストレージでバケットとあるがGCSとは無関係。後述の監査ログでも利用される。

### 監査ログ
Google Cloud では、操作やアクセスの履歴を 監査ログ（Audit Logs） として記録する。

監査ログの主な種類

|種類|内容|デフォルト|
|:----|:----|:----|
|Admin Activity|リソースの作成・削除・設定変更などの管理操作|常に有効|
|System Event|Google 管理下での内部操作|常に有効|
|Policy Denied|IAM による拒否イベント|常に有効|
|Data Access|データの読み書き（例：Cloud Storage オブジェクト参照）|原則無効|

上記の情報はLogging バケットに保管され、以下の2つに保管される。

|ログバケット|役割|
|:----|:----|
|_Required|重要な監査ログを必ず保存するためのバケット|
|_Default|_Required に該当しないログの保存先|

#### _Required バケット
以下の監査ログは、Cloud Logging に組み込まれた内部ルールにより 必ず _Required に保存される
- Admin Activity
- System Event
- Policy Denied

Data Access 監査ログは、明示的に有効化された場合のみ生成され、生成された場合は _Required に保存される

_Required バケット内のログは
- 除外不可
- 無効化不可

とされており、セキュリティ・コンプライアンス上の基盤となる

#### _Default バケット
_Defaultバケットには、_Required に該当しない すべてのログ が保存される。
アプリケーションログ、インフラログ、デバッグログなどが主な対象

#### 監査ログを閲覧するのに必要な権限
監査ログを閲覧するためには、`roles/logging.viwer`か`roles/privateLogViewe`のどちらかの権限が必要。Data Accessを閲覧する場合にはprivateLogViewerの権限が必要。

|ログの種類|位置づけ|必要なロール|
|:----|:----|:----|
|Admin Activity|監査ログ（非 private）|roles/logging.viewer|
|System Event|監査ログ（非 private）|roles/logging.viewer|
|Policy Denied|監査ログ（非 private）|roles/logging.viewer|
|Data Access|private logs|roles/logging.privateLogViewer|

Data Accessがどうしてプライベートの権限かというと、Data Access監査ログは「管理操作」ではなく、「実データへのアクセス」を記録しており、Cloud Storageの特定のオブジェクト名など、データの存在や構造を推測できるようなノグが出力されている。

### ログビュー
Cloud Loggingを誰に見せるかの制御は`ログビュー`で管理される。
ログビューは、フィルタリングおよびIMAの付与が可能で制御に向いている。
```
[ Logging Project ]
     └─ Log Bucket: _Default
        └─ Log View: _Default
        └─ Log View: _AllLogs
        └─ Log View: <custom-view>
```

ビューの例は以下

|ビュー|意味|
|:----|:----|
|_Default|そのログバケットに「格納されているログ」だけを表示するビュー|
|_AllLogs|プロジェクト内で「どのバケットにあっても」すべてのログを横断表示するビュー|
|カスタムビュー|フィルタ条件に合致したログのみ|

### 集約ログシンク
集約ログシンクは、Cloud Logging において
組織またはフォルダ配下の複数プロジェクトのログを一括でエクスポートするための仕組みである。

シンクの作成場所を、以下のレベルで設定
- 組織
- フォルダ

配下に存在するすべての既存プロジェクト将来追加されるプロジェクトのログが自動的に対象になる。

### Opsエージェント
Compute Engine「自体」の一部のログは自動で送られるが、VM 上で動く OS・アプリのログや詳細メトリクスは送られない。
ログやメトリクスを収集し、Cloud LoggingやCloud Monitoringに送信するためのエージェントとしてOps Agentを利用する。

OpsAgentはVMの中で、ログを集め、メトリクスを集め、Cloud LoggingやCloud Monitoringに送信する。

ロギングについては、ロギングレシーバーと呼ばれる設定を行なって、「どのログをどの形式で」読み取るか定義して送信する。
```
アプリログ
  ↓
Ops Agent（ロギングレシーバー）
  ↓
Cloud Logging
```

メトリクスについては、メトリクスレシーバーと呼ばれる設定を行なって、「どのメトリクスをどの間隔で」取得するかを定義して送信する。
```
VM 内のリソース状況
  ↓
Ops Agent（メトリクスレシーバー）
  ↓
Cloud Monitoring
```

|観点|ロギング|メトリクス|
|:----|:----|:----|
|扱うもの|テキスト|数値|
|例|IPアドレス、URL|CPU%、メモリMB|
|保存先|Cloud Logging|Cloud Monitoring|
|検索|強い|弱い|
|グラフ|ログベース指標経由|直接可能|

### エージェントポリシー
Ops Agentについて、どのような状態であるべきかについて宣言的に定義する仕組み。
既存VMおよび将来新規作成されるすべてのVMを対象に設定可能。