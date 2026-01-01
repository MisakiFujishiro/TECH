# API Gateway
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

### Apigeeによるホスティング
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


### CloudEndpointとApigeeの違い
- Cloud Endpoints
    - シンプルなパブリックAPIの入口管理
    - 開発目線（技術的なAPI）
- Apigee
    - 本格的な API プラットフォーム / API ビジネス
    - ビジネス目線（APIの商用利用）
