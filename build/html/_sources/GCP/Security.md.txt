# セキュリティ
## フェデレーション
GCPにおけるフェデレーションとは、
Googleアカウントを発行せずに、外部のIDプロバイダー（IdP）を信頼して認証・認可を行う仕組みである。

従来のようにGCP側でIDを管理するのではなく、
外部IdPに認証を委任し、GCPはトークンを検証してアクセスを許可する。



| 種類                            | 認証主体     | 認証対象                   | 主な用途           | 特徴                 |
| ----------------------------- | -------- | ---------------------- | -------------- | ------------------ |
| Workforce Identity Federation | 人（ユーザー）  | GCP（コンソール / CLI / API） | 社外ユーザー・パートナー   | Googleアカウント不要でログイン |
| Workload Identity Federation  | マシン（アプリ） | GCP API                | マルチクラウド・CI/CD  | 鍵レス認証（SAキー不要）      |
| Workload Identity（GKE）        | Pod      | GCP API                | Kubernetes内の認証 | KSA→GSAのマッピング      |

フェデレーションのコア概念は以下：
- 認証は外部IdPが実施
- GCPはトークンを信頼して検証
- IAMで最終的な認可を実施
```
外部IdP
   ↓（認証）
トークン発行（OIDC / SAML）
   ↓
GCP（STS / IAM）
   ↓（認可）
リソースアクセス
```

### Workforce Identity Federation
外部のID（Azure AD, Okta, SAML IdPなど）を使用して、
人間ユーザーがGoogle Cloudにログインするための仕組み

ユースケース
- パートナー企業
- 委託業者
- 外部監査人

構成
```
Workforce Identity Pool
 ├── Provider (Azure AD)
 ├── Provider (Okta)
 └── Provider (SAML IdP)
```
アクセスフロー
```
ユーザー（ブラウザ）
   ↓
外部IdPで認証
   ↓
GCPがトークンを受け入れ
   ↓
IAMロールに基づきアクセス許可
```

### Workload Identity Federation
外部環境（オンプレ / AWS / Azureなど）で実行されるアプリケーションが、
サービスアカウントキーを使わずにGCPにアクセスする仕組み

```
Workload Identity Pool
 └── Provider（OIDC / AWS / Azure ADなど）
        ↓
   サービスアカウントへのなりすまし（Impersonation）
```
アクセスフロー
```
アプリ（オンプレ / 他クラウド）
   ↓
外部IdPで認証
   ↓
Security Token Service（STS）でトークン交換
   ↓
サービスアカウントとして実行
   ↓
GCP APIアクセス
```


## Cloud Identity SSO
Cloud Identity SSOとは、自社ユーザーがGoogleにログインする方法を、外部IdPに委ねる仕組み
```
社員（自社ユーザー）
   ↓
自社IdP（Okta / AD）
   ↓
Google Workspace / GCP
```
ポイント
- 対象は 自社ユーザー
- Googleアカウントは存在する（ここ重要）
- 認証だけを外部IdPに任せる

## Identity Platform
Identity Platformは、Google Cloud の公式CIAM（Customer Identity and Access Management）サービス

Identity Platform は以下をサポートしており、
- SAML 2.0
- OpenID Connect (OIDC)

以下のように、アプリに対するログインでフェデレーション（自社ドメインでログイン）が可能
```
企業ユーザー
  ↓
自社IdP（Okta / Azure AD など）
  ↓（フェデレーション）
Identity Platform
  ↓
アプリ
```


「顧客のログイン基盤（アプリ用）」

対象：顧客（不特定多数）
- 一般ユーザー（B2C）
- 企業ユーザー（B2B）
- アクセス先：あなたのアプリ
Googleアカウント：不要（あってもOK）


## Identity-Aware Proxy (IAP)
IAP は Google Cloud が提供する認証・認可プロキシである。
アプリケーション自身にログイン処理や認証ロジックを実装しなくても、Google アカウント（Google Workspace 含む）を使った安全なアクセス制御を実現できます。
Identity-Aware Proxyにより、アプリケーションに直接認証ロジックを実装しなくても、Google アカウントを用いた認証およびアクセス制御を実現できます。



### IAPの役割
IAP はアプリケーションの 前段（HTTP(S) ロードバランサ等）に配置され、次を担当します。
- ユーザー認証
- Google アカウントでログイン
- アクセス認可
- IAM / Google グループでアクセス可否を判定
- 認証済みリクエストのみをバックエンドへ転送

### IAP と Load Balancer の関係
IAP は 単体で通信を受けるサービスではない。
必ず Cloud Load Balancing（HTTP(S) Load Balancer）と組み合わせて利用される。

立ち位置の整理
- IAP は HTTP(S) Load Balancer の機能の一部として動作
- IAP は Backend Service に対して有効化される
- クライアントは 必ず Load Balancer 経由で IAP を通過する


### IAPの処理詳細
AP による認証が成功すると、IAP が署名した JWT がバックエンドに渡されます。
- 処理イメージ
  - ユーザーがアプリにアクセス
  - IAP が Google アカウントで認証
  - IAP がアクセス許可を判定
- 許可された場合のみ
  - JWT を HTTP ヘッダに付与
  - バックエンド（Compute Engine / GKE / Cloud Run 等）へ転送
- アプリ側の役割
  - JWT の 署名検証
  - ユーザー情報（email / ID）を参照
  - 「認証済みユーザー」として処理する

結果として、アプリは「JWT を検証するだけ」ログイン画面・OAuth 実装は不要。

IAP が付与する JWT には、例えば次の情報が含まれる。
- ユーザーの Google アカウント（email）
- IAP が発行者であること（issuer）
- 対象アプリ（audience）
- 有効期限

これにより、アプリは「誰が」「どのアプリに」「正しく認証されて来たか」を安全に判定できる。