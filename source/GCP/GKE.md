# GKE
## Network上の制約
GKE（VPCネイティブクラスタ）では、ネットワーク設計が重要。
特に重要なのは「Pod」「Service」「Node」のIPがそれぞれ別のレンジから払い出されるという点。

VPCネイティブクラスタでは、1つのサブネットに対して以下のIPが必要になります。

| 用途 | どこから払い出される？ | 何に使われる？ |
|------|------------------------|----------------|
| Node IP | サブネットのプライマリレンジ | ノード（VM）のIP |
| Pod IP | サブネットのセカンダリレンジ① | Podの実IP |
| Service IP | サブネットのセカンダリレンジ② | ClusterIP（仮想IP） |

重要なのは、**Pod用とService用は別のセカンダリレンジが必要**。

VPCネイティブクラスタでは、1つのサブネットに対して以下のIPが必要になる。

| 種類 | 払い出し元 | 何のIPか | 何に依存する？ | 増え方 |
|------|------------|----------|----------------|--------|
| Node IP | サブネットのプライマリレンジ | ノードVMのIP | ノード数 | ノード追加で1つずつ増える |
| Pod IP | サブネットのセカンダリレンジ（Pod用） | Podの実IP | ノード数 × ノードあたりCIDR | ノード単位でブロック増加 |
| Service IP | サブネットのセカンダリレンジ（Service用） | ClusterIP（仮想IP） | Service数 | Serviceごとに1つ |

重要なのは、Pod用とService用は別のセカンダリレンジが必要な点。

### Node IP
Node IPはシンプルにノード数=必要IP数

| 項目 | 内容 |
|------|------|
| 割り当て単位 | ノード1台につき1IP |
| 例 | 3ノード → 3IP |
| 注意点 | 通常は余裕を持ったサブネットを作る |

### Pod IP
Podはデフォルトでノードあたりに最大110IPを確保する必要がある。
110以上のPodを起動させる場合は、ノードを増やすがmaxPodsを変更して1ノードあたりのPodを110以上にする必要がある。

| 項目 | 内容 |
|------|------|
| 払い出し単位 | ノード単位でCIDRブロック |
| デフォルト | 1ノードあたり /24（256IP） |
| maxPods変更時 | 必要数に応じて /23 などに拡大 |
| 計算基準 | ノード数 × ノードあたりCIDR |

### Service IP
k8sで作成するサービスの数に合わせて提供するIPを決める

| 項目 | 内容 |
|------|------|
| 割り当て単位 | Serviceごとに1IP |
| Pod数との関係 | 無関係 |
| 例 | Web + API → Service 2個 → IP 2個 |



## Workload Identity
GKEのKSAでGoogleCloud IAM SAを利用する際の詳細はIAM側で解説。
GSAを利用しつつ、GSAを代替利用することができる。

## Config Connector
Config Connectorとは、Kubernetes のアドオンであり、Kubernetes API を通じて Google Cloud リソースを宣言的に管理する機能。

GCPのリソースについて宣言的な構成管理を実現し、リソースの現在の状態を監視し、宣言された構成との差異を検出すると自動的に調整を行う。これにより、構成ドリフトを防止し、インフラストラクチャが期待される状態に保たれる。

## Anthos Config Management
Google Kubernetes Engine（GKE）では、単にアプリケーションを動かすだけでなく、
- 構成の一貫性
- ポリシーによるガードレール
- Git を正とした運用（GitOps）

が求められる。

これらを支える中核が`Anthos Config Management`を中心とした一連のコンポーネント群である。

### Anthos Config Managementとは
Anthos Config Management は、Kubernetes クラスターの構成とポリシーを Git リポジトリで一元管理し、自動同期する Google Cloud のマネージドサービスである。

Git を唯一の正（Source of Truth）として、クラスターが定期的に Git を監視し、差分を自動適用手動で kubectl apply を実行する必要がない

管理対象
- Namespace
- RBAC
- NetworkPolicy
- Policy Controller（制約テンプレート・制約）
- その他 Kubernetes マニフェスト

### Config Sync
Config Sync は、Anthos Config Management の中核となる同期エンジンである。
- Git リポジトリを定期的にポーリング
- YAML マニフェストを Kubernetes API に適用
- 差分検出と再同期（drift correction）を実施

### Policy Controller
Policy Controller は、Kubernetes リソース作成・更新時にポリシーを強制するための仕組みである。
- Kubernetes Admission Controller として動作
- ルール違反のリソースを作成時に拒否
- 既存リソースの監査（audit）も可能

### OPA Gatekeeper
OPA Gatekeeper は、Kubernetes 向けのポリシーエンジン（OSS）である。
- OPA（Open Policy Agent）を Kubernetes に統合
- ポリシーをコード（Rego）として記述
- Admission Controller として動作

### 制約テンプレート（ConstraintTemplate）
制約テンプレートは、「どのようなポリシーを作れるか」を定義する設計図（型）である。
- Rego による検証ロジックを含む
- 再利用前提で組織共通化される
- 単体では制限を発動しない

### ポリシーパラメータ（Constraint）
ポリシーパラメータは、制約テンプレートに具体的な値を与えてポリシーを有効化するものである。
- 環境ごとに値を変えられる
- Git 管理される変更対象
- 実際に制限を発動する主体

### それぞれの関係
k8sにおいて、作成されるルールを統制するための仕組みの関係は以下

git~ConstraintTemplate/Constraint
```
Git Repository（正）
        ↓
Config Sync
        ↓
Kubernetes クラスター内に
ConstraintTemplate / Constraint を常に存在させる
```

ConstraintTemplate/Constraint~k8s cluster
```
ConstraintTemplate（ルールの定義）
        ↓
Constraint（ルールの具体化）
        ↓
Policy Controller（検査・判断・拒否を行う実行エンジン）
        ↓
Kubernetes API Server
        ↓
リソースを許可 or 拒否
```

## Managed Service for Prometheus
Prometheus 互換のメトリクスをフルマネージドで扱える Google Cloud のサービスであり、Prometheusのストレージ管理や高可用性、スケーリングを意識せずに PromQL を利用できる。

GKE 環境では、マニフェストにラベルやアノテーションを設定するだけで、Pod や Node のメトリクスを自動的に収集できる。

