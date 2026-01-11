# API Gateway
GCPで提供されるGatewayの比較は以下

|観点|Cloud Endpoints|Cloud API Gateway|Apigee|
|:----|:----|:----|:----|
|位置づけ|軽量 API 管理（OSS寄り）|マネージド API Gateway|フル機能 API 管理プラットフォーム|
|主用途|既存バックエンドに API 管理を“後付け”|サーバーレス向けの標準 API Gateway|エンタープライズ向け API 管理|
|想定ユーザー|開発者 / 小〜中規模 API|クラウドネイティブ開発者|大規模組織 / B2B / 外部公開 API|
|認証方式|API Key / OAuth2 / JWT|API Key / OAuth2 / JWT|OAuth2 / mTLS / SAML / OIDC など|
|レート制限・クォータ|⚪︎（基本的）|⚪︎（基本的）|⭕⭕⭕（高度・柔軟）|
|API メトリクス|⭕（Cloud Monitoring）|⭕（Cloud Monitoring）|⭕⭕⭕（分析・可視化が非常に強力）|
|API 分析 / 可視化|❌（最小限）|❌（最小限）|⭕（トラフィック分析・利用者分析）|
|API ライフサイクル管理|❌|❌|⭕（設計〜廃止まで）|
|開発者ポータル|❌|❌|⭕（提供可能）|
|変換・ポリシー制御|❌|❌|⭕（リクエスト/レスポンス変換など）|
|対応バックエンド|GCE / GKE / Cloud Run / App Engine|Cloud Run / Cloud Functions|ほぼ何でも（GCP / オンプレ / 他クラウド）|
|構成の重さ|軽い|軽い（完全マネージド）|重い（その分できることが多い）|
|試験での立ち位置|「API 管理したい」= まずこれ|Cloud Endpoints の後継・簡易版|要件が重いときの最終兵器|



## Cloud Endpoint
Cloud Endpoints は、Google Cloud が提供するパブリック API 向けの軽量な API 管理サービスである。

API の実装とは独立して、API の入口（フロント）での制御を担うことを目的としている。  
特徴は、`技術的な API を安全に公開するためのサービス`

主な役割は、
- API 呼び出し元の 認証・認可（APIキー、JWT など）
- 過剰なリクエストの制御（クォータ・レート制限）
- API の メトリクス収集と可視化

Cloud Run、GKE、Compute Engine、App Engine など、さまざまなバックエンドと組み合わせて利用できる。
REST / gRPC の両方をサポートしており、API の仕様（OpenAPI / proto）を定義することで管理を有効化する。
アプリケーション側のコードを大きく変更せずに、安全で運用しやすいパブリックAPIを提供できるのが特徴。


## Apigee
Apigee は、Google Cloud が提供する API 管理（API Management）プラットフォーム。  
Apigee は API Gateway ではなく、`API Management Platform`である。

Apigee はアプリケーションのバックエンドの前段に配置される API のエントリポイントとして機能し、以下のような責務を担う。
- 認証・認可（OAuth 2.0、JWT、API キーなど）
- レート制限・スロットリング・クォータ管理
- API 利用者ごとのプラン管理（サブスクリプション管理）
- トラフィック制御・不正アクセス対策
- API 利用状況の分析・モニタリング

これにより、Apigee は単なる API の公開手段ではなく、API を「プロダクト」として安全かつスケーラブルに提供・運用するための機能群を提供する。

### Apigee hybrid
通常の無印Apigeeは、正式には、Apigee X。Apigeeにはhybridバージョンが存在し、「コントロールプレーンは Google、データプレーンは 自分の Kubernetes」というhybrid構成となる。
- Apigee X = 完全マネージド API Gateway
- Apigee hybrid = Control Plane SaaS + Self-managed Gatewayhybridは、

hybridでは、APIにおけるコントロールプレーンとデータプレーンの役割をhybrid構成にしている
- Data Plane = 本番の API 通信そのもの
- Control Plane = その通信の “ルール” を決める世界

Apigee hybrid では、API トラフィックを処理する Data Plane を自社 Kubernetes 環境に配置することで、API の実通信や証明書・秘密情報の実体を自社ネットワーク内に留めたまま、
API 管理の Control Plane のみを Google に委ねることができる。
上記を踏まえて、Apigee Xと Apigee hybridを比較したものが以下

|観点|Apigee X|Apigee hybrid|
|:----|:----|:----|
|API Proxy の実行場所|Google管理|自分のKubernetes|
|コントロール|Google|Google|
|データプレーン|Google|自社環境|
|ネットワーク閉域|△|◎|
|運用負荷|最小|K8s分は必要|

### Apigeeによるホスティング
```
[Host / URL]
   └ Environment Group
        └ Environment
             └ API Proxy
                  └ TargetServer
                       └ Backend API
```
Apigee では、Cloud Run や GKE、Compute Engine などで動作している 既存のバックエンド API の前段に配置され、
外部からのリクエストを受け付け、制御したうえでバックエンドへ中継する役割を担う。

Apigee では、API を安全に外部公開するために、
- API を実行する場所(Environment)
- API を公開する URL(Host名)

を明確に分離して管理する設計が採られている。

#### Eivironment
まず、API は `Environment` と呼ばれる論理的な環境にデプロイされる。
Environment は、テスト用・本番用といった APIの実行環境を表す単位であり、同じ API であっても test 用と prod 用で別々に管理される。

一方で、外部から API にアクセスする際に使用されるホスト名（ドメイン名） は、Environment そのものには直接割り当てられない。

#### Environment Group
Apigee では、ホスト名は Environment Group という別の概念に紐づけて管理される。
Environment Group は、「この URL で公開してよい API の環境はどれか」を定義するための単位である。

Environment Group に特定の Environment を関連付けることで、その Environment にデプロイされた API だけが、対応するホスト名を使って外部公開される。

この仕組みにより、
- テスト環境の API が、本番用の URL で公開される
- 本番環境の API が、テスト用の URL で呼び出される

といった状態は、設定ミスや運用ルールに関係なく構造上発生しない。

つまり Apigee では、どの API が、どの環境で、どの URL を使って公開されるかという関係そのものを、API の契約としてプラットフォームが保証する。

#### TargetServer（バックエンド定義）
Apigee では、バックエンド API の接続先を直接 API Proxy に書くのではなく、
`TargetServer` というリソースとして定義して管理する。

TargetServer は以下の情報を持つ。
- バックエンドのホスト名
- ポート
- プロトコル（HTTP / HTTPS）
- mTLS 設定などの接続情報

API Proxy は、TargetServer を参照することでバックエンドへリクエストを中継する。

この設計により、
- バックエンドの切り替え（リージョン別 / DR / フェイルオーバー）
- ネットワーク構成の変更
- セキュアな証明書管理

を、API Proxy のロジックを変更せずに実現できる。

##### TargetServer と Environment の関係
TargetServer は Environment に紐づいて管理される。

そのため、
- prod Environment では prod 用バックエンド
- test Environment では test 用バックエンド

といった分離が構造的に保証される。

##### フローバリアブルによる動的切り替え
Apigee では、API プロキシの処理中に利用できる フローバリアブル（Flow Variables） が多数用意されている。
フローバリアブルは、リクエスト・レスポンス・実行環境・セキュリティ状態などを表す読み取り専用／書き込み可能な変数であり、API プロキシの挙動を動的に制御するための中核的な仕組みである。

例えば、`system.region.name` は、現在 API プロキシが実行されている Apigee ランタイムのリージョン名を表すフローバリアブルである。
これにより、マルチリージョン構成において、各リージョンの Apigee ランタイムが、最も近い（局所的な）バックエンドにルーティングする設計が可能となる。



## Cloud API Gateway
Comming soon