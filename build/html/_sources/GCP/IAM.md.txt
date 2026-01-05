# IAM
リソースやユーザーの管理にはIAM（Identity and Access Management）を利用する。
IAMを利用することで、`誰が`、`どういう操作を`、`何に対して`、`どういう条件で`を制御することができる。
AWSと同じような用語が用いられるがそれぞれで役割が異なるため注意する。

![](../img/GCP/IAM/IAM.png) 
[まずは知っておくべき IAM の基礎と最新の便利機能](https://services.google.com/fh/files/events/0224-infra-onair-seesion-1.pdf)

GCPのIAMの本質は`リソースベースでのアクセス制御`である。
以下の流れで、リソース側で誰がどの操作をして良いかについて定義を行う。
1. IAM Roleにより権限を定義する
2. IAM PolicyによりIAM RoleとPrincipalを紐づける（バインディング）
3. IAM Policyをリソースに付与する


## プリンシパル
`誰が`については、principalとも呼ばれ、大きく以下の4つが対象となる。
![](../img/GCP/IAM/IAM_identity.png)
[まずは知っておくべき IAM の基礎と最新の便利機能](https://services.google.com/fh/files/events/0224-infra-onair-seesion-1.pdf)

プリンシパルにはそれぞれメールアドレス形式の識別子がを持つ。

|プリンシパルの種類|識別子形式|説明・特徴|
|:----|:----|:----|
|Google アカウント|user:alice@gmail.com|個人のGoogleアカウント（Gmail や Workspace）。人間のユーザー。|
|Cloud Identity ドメイン|domain:example.com|ドメイン全体にIAM権限を付与。Google Workspace や Cloud Identity ドメインが対象。|
|Google グループ|group:dev-team@example.com|複数ユーザーを一括管理するメールグループ。IAM権限をまとめて付与可。|
|サービス アカウント|serviceAccount:my-sa@project.iam.gserviceaccount.com|アプリケーションやGCPサービスが操作するためのアイデンティティ。|

## IAM Role
`どういう操作を`については、IAM Roleによって定義される。
GCPのIAM Roleは、どの操作（API 権限）を許可するかのみを定義する。

具体的には、以下のように`includedPermissions`で認可される操作が定義される。
```yaml
title: "My Custom Read-Only Storage Role"
description: "Allows listing and reading GCS objects"
stage: "GA"
includedPermissions:
  - storage.objects.get
  - storage.objects.list
```

例から分かる通り、AWS IAMポリシーとは異なり利用できるprincipalや、対象のResourceの指定は含まない。
後述するがGCPでは、リソースベースポリシーの許可ルールがされており、principalやresouceの紐付けはIAM Policyのバインディングを利用する。

Roleに関しては、おおきく3つが準備されており、事前定義Roleの利用を検討して、より細かい制御が必要な時にカスタムロールを利用する。

|ロール種別|名称（英語）|説明・特徴|カスタマイズ|使用推奨度|例|
|:----|:----|:----|:----|:----|:----|
|基本ロール|Primitive Roles|プロジェクト全体への大雑把な権限（旧式）|❌ できない|🔻 非推奨|roles/editor|
|事前定義ロール|Predefined Roles|Googleが用意した細かく設計されたロール|❌ できない（そのまま使う）|✅ 推奨|roles/compute.viewer|
|カスタムロール|Custom Roles|ユーザーが必要な権限だけを指定して作成|✅ 可能|⭕ 条件付きで推奨|roles/custom.storageReader|


## IAM Policy
IAM Policyは、`バインディング`、`監査構成`、`メタデータ`を定義している。

`バインディング`により、`誰が`が`どの操作を`するかの紐付けを定義し、リソースに対して付与する。
どの操作についてはRoleで定義を行なったことを踏まえると、IAM Policyにより`principal`と`IAM Role`を`紐づける`定義が行われる。

IAM Policyでは、bindingsとして、roleとprincipal(members)を紐づけている。
```yaml
bindings:
  - role: roles/storage.objectViewer
    members:
      - user:alice@example.com
  - role: roles/storage.objectAdmin
    members:
      - serviceAccount:app-sa@my-project.iam.gserviceaccount.com
etag: BwWWja0YfJA=
version: 1
```

IAM Policyはリソースに付与するため、以下のようなコマンドで付与を行う。

loud Storage バケットへの適用
```bash
gcloud storage buckets set-iam-policy gs://my-bucket policy.json
```

プロジェクトへの適用
```bash
gcloud projects set-iam-policy my-project-id policy.json
```

### 上位階層への付与
上記の例のように、特定のバケットにポリシーを付与することができるとともに、組織やフォルダ、プロジェクトも同様にリソースであるため、ポリシーを付与することができる。
付与された権限については、GCPのリソース階層（組織 > フォルダ > プロジェクト > リソース）に基づき、上位階層での設定が下位リソースにも継承される。そのため、プロジェクトに対してIAM Policyを付与して権限を与えると、そのプロジェクト配下のリソース全てに権限を付与していることになる点に注意。

![](../img/GCP/IAM/IAM_kaisou.png)
[Google Cloud Fundamentals: Core Infrastructure 日本語版](https://www.coursera.org/learn/gcp-fundamentals-jp/lecture/KUBlM/identity-and-access-management-iam)

上位階層をリソースとして、ポリシーを付与するとアイデンティティベースのように認可を与えることになる点に注意。
例えばPJに対してバインディングを行い、特定のSAをprincipalにした場合、その後はPolicyにRoleを付与していくことでSAは対象リソースへの権限を持つことができ、リソース側にバインディングする手間が省ける。

## SA(Service Account)
サービスアカウント（SA）は、ユーザーではなくGCP内のサービスが認証・認可を行うための機能。2つの役割を持っていることを意識すると理解しやすい
- `プリンシパル`・・・Policyでバインディングを指定する際にSAをprincipalにすることができる
- `リソース`・・・SA自体がリソースであるので誰がSAを引き受けられるかを定義する

例として、VM などが Cloud Storage や BigQuery といった他のリソースにアクセスすることを考える。
以下の2つのステップで認可を定義する。

- 「SAにどんな権限を与えるか？」を定義（アクセスしたいリソース側にバインディング）
  - 対象リソース: Cloud Storage、BigQuery、Pub/Subなど
  - Principal: サービスアカウント
  - 付与するロール: roles/storage.objectViewer など、操作対象に応じたロール
```
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="serviceAccount:my-sa@project.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```
- 「誰が SA を使ってよいか？」を定義（SA 自体にバインディング）
  - 対象リソース: サービスアカウントそのもの
  - Principal: VM やユーザー
  - 付与するロール: roles/iam.serviceAccountUser（または roles/iam.serviceAccountTokenCreator など）
```
gcloud iam service-accounts add-iam-policy-binding my-sa@project.iam.gserviceaccount.com \
  --member="user:bob@example.com" \
  --role="roles/iam.serviceAccountUser"
```

### 自動バインディングに関する補足
Cloud Run や GCE などのサービスに SA を指定してリソースを作成すると、そのリソースがSAを使うためのバインディング（上記の後半部分）はGCPが自動で設定してくれる。
したがって、通常はユーザーが明示的に設定する必要はない。

### SAのメリット
SAを利用することで以下のようなメリットを享受できる

|理由|解説|
|:----|:----|
|① アクセス権限の一元管理|VMインスタンスごとにアクセス権限をIAMで直に設定していくと管理が煩雑になる。<br>SAにアクセス権限を集中させることで、1箇所で制御可能になる。|
|② 責任の分離|	「誰がこの操作を行ったのか」をSA単位でトレースできる。<br>SAはCloud Audit Logsでの識別にも使われるため、ロールごとの責任区分が明確になる。|
|③ 再利用性・構成の再現性|	複数のVMやCloud Functionsに、同じSAを割り当てることで、共通の権限設定を再利用できる。<br> Infrastructure as Codeとの相性も良い。|
|④ 実行単位に応じた設計が可能|実行環境（VMやCloud Functionなど）に応じて別々のSAを割り当てれば、<br>最小権限の原則（Principle of Least Privilege）を実践できる。|

SAは、サービスアカウントキー（秘密鍵と公開鍵）を発行することができ、その情報を利用することでSAに許可されている権限を行使することができる。注意点として、サービスアカウントキーをgithubなどに後悔しないように注意する。

## クロスプロジェクト
GCP におけるプロジェクト間をまたいだリソース操作は、
「実行主体（principal）」と「リソース所有側（resource）」を分離して設計する。

1. 実行主体となるプロジェクト（principal 側）でサービスアカウント（SA）を作成する
2. 実行主体の SA を用いて、アプリケーション（Cloud Run / Functions など）を実行する
3. リソース所有側（resource 側）で、IAM ポリシーに以下を指定し、リソースにバインディングする
   - member: principal 側の SA
   - role: 必要最小限の Role

### 補足：API 有効化が必要なマネージドサービス
一部の GCP マネージドサービス（例：Cloud SQL）は、リソースへのアクセス時に内部的に管理 API を呼び出す。

そのため、クロスプロジェクト構成では IAM 設定に加えて、実行主体側・リソース側の両プロジェクトで該当 API（例：Cloud SQL Admin API）を有効化する必要がある。

これは IAM 権限とは別の「サービス利用可否」の設定であり、権限が正しく付与されていても API が無効な場合は接続に失敗する。


## Workload Identity
Workload Identityとは、外部のワークロード（k8s/VM/他クラウド）が、GCPのIAMSAとして振る舞うための仕組みである。
代表的な利用例は、K8Sである。

### K8SにおけるWorkload Identity
K8SにおいてWorkload Identityを利用すると、Kubernetes ServiceAccount（KSA）と GCP IAM ServiceAccount（GSA）という異なる認証ドメインに属する ID を安全に関連付けることができる。

- KSA（Kubernetes ServiceAccount）
  - Kubernetes クラスタ内の ID
  - Pod が Kubernetes API にアクセスするための身分証
- GSA（GCP IAM ServiceAccount）
  - Google Cloud IAM 上の ID
  - Cloud Storage や Pub/Sub など GCP API にアクセスするための身分証

これらを関連付けることで、Kubernetes 上のアプリケーションは KSA を使用したまま、サービスアカウントキーを用いることなく、GSA の権限で GCP リソースにアクセスできる。

具体的な手順は以下
- Kubernetes ServiceAccount（KSA）を作成する
- IAM ServiceAccount（GSA）を作成し、KSA を principal として roles/iam.workloadIdentityUser を付与する
- KSA にアノテーションを設定し、対応する GSA を指定する(両方向で指定が必要)
- Pod（または Deployment）で使用する ServiceAccount として KSA を指定する

この構成により、Pod は 短命な認証情報を用いてGSA の権限で GCP API を呼び出すことが可能となる。

## Identity-Aware Proxy (IAP)

IAP は Google Cloud が提供する認証・認可プロキシであり、
アプリケーションの前段に配置され、Google アカウントを用いたアクセス制御を提供する。
Identity-Aware Proxyにより、アプリケーションに直接認証ロジックを実装しなくても、Google アカウントを用いた認証およびアクセス制御を実現できる

IAP のアクセス制御は IAM に基づいて行われ、
IP やネットワークではなく「誰がアクセスしているか」を基準とする。