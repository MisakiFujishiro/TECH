# Kubernetes
Kubernetesはコンテナのオーケストレーションサービスであり、コンテナのデプロイや自動修復、デプロイ管理、ロードバランシングなどの操作を支援する

## 主要なコンポーネント
ECSと比較しながら、	Kubernetesで扱われる用語を整理すると以下。

|Kubernetes|ECS|説明|
|:-|:-|:-|
|Node|ECSのFargate/ECSインスタンス|Podが実行されるホスト（EC2またはFargate上で動作）。|
|Namespace|1つの Kubernetes クラスタの中を論理的に区切るための「仮想的な区画」|
|Service|ECSのロードバランサ|内部/外部向けにPodを公開し、負荷分散を提供。|
|Deployment|ECSのサービス|Podの管理をする仕組み。スケーリングやローリングアップデートを管理。|
|Pod|ECSのタスク|コンテナをまとめて管理する単位（1つ以上のコンテナを含む）。|
|StatefulSet|-|Pod に個体性（ID/ディスク）を持たせて管理|
|ConfigMap/Secret|ECSのタスク定義の環境変数管理|設定値や機密情報（DBパスワードなど）を管理。|

### Node
Podが実行される実態。
ノードには2種類存在し、MasterNode（コントロールプレーン）とWorkerNode（データプレーン）がある。

### namespace
Namespaceは、クラスタの中を論理的に区切るための「仮想的な区画」であり、同一クラスタ内でリソースを分離できる。

```
cluster
 ├─ namespace: team-a
 │   ├─ deployment: api
 │   └─ service: api
 ├─ namespace: team-b
 │   ├─ deployment: api
 │   └─ service: api
```

### Service
Serviceとは、Pod の集合に対して、安定した到達手段を提供する

Serviceは、ネットワーク・発見・安定アドレスの責務を担う
- pod群に対する固定IP/DNSの提供
- podの増減を隠蔽
- L4のロードバランシング

具体的には、Service が 代表となる IP アドレスや DNS 名を持ち、その宛先に届いた通信を 内部の Pod 群のいずれかへ振り分けることで、呼び出し元から Pod の存在や入れ替わりを意識させない仕組みを提供する。

### Deployment
Deploymentは、運用・スケール・自己修復の責務を担う

以下の事柄が責任範囲
- 何個のpodを動かすか
- podの自動修復
- ローリングアップデート

### Pod
複数コンテナが1つの Linux ネットワーク名前空間を共有している。  

そのため、PodでSQL Proxyを設定して、アプリ側でlocalhost:5432の通信をすると、Proxy側がlocalhost:5432としてlistenしており、結果としてSQLへの通信を代替してくれる。



### 権限・リソースの制御
#### 権限制御：RBAC（Role-Based Access Control）
「誰が」「どの名前空間で」「何をしていいか」を決める仕組み

- チーム A：
    - team-a namespace で
    - Pod / Deployment を操作できる
- チーム B：
    - team-b namespace だけ操作可能
- 他の namespace にはアクセス不可


#### リソース制御：リソースクォータ（ResourceQuota）
「この名前空間は、ここまでしか使ってはいけない」という上限
- 1チームの暴走でクラスタ全体が遅くなるのを防ぐ
- 性能（パフォーマンス）を安定させる

```
limits:
  cpu: "8"
  memory: "32Gi"
pods: "50"
```

#### 制御まとめ
K8sにおける分離の種別をまとめると以下

|レベル|手段|コスト|
|:-|:-|:-|
|論理分離|Namespace|低|
|権限制御|RBAC|低|
|資源制御|ResourceQuota|低|
|物理分離|複数クラスタ|高|



### 起動制御
#### Nodeの起動数制御
Cluster AutoscalerによりNodeの起動数を自動調整する。
Pod の リソース要求（requests） に基づいて、
- Pod が載りきらない場合 → ノードを追加
- ノードが余っている場合 → ノードを削除

これにより、Pod の需要に応じてノード数を自動調整できる。

もう少し具体的に、スケールインについて、整理するとスケールアウトしたNodeB側に載っているPodを安全にNodeAに移すことができると判断した時点で、PodをNodeAに移動させる。

#### Podの起動数の制御
|項目|略称|詳細|
|:-|:-|:-|
|Horizontal Pod Autoscaler |HPA|ワークロードの負荷（CPU / メモリ / カスタム指標など）に基づいて Pod のレプリカ数を自動でスケールアウト・スケールインする仕組み|
|Vertical Pod Autodcaler|VPA|実際の使用状況をもとに Pod の CPU / メモリの requests / limits を自動調整する仕組み（Pod 数は増減しない）|
|Pod Dsiruption Budget|PDB|ノードメンテナンスやアップグレードなど、意図的な Pod の中断（Disruption）時に許容される最小稼働数を定義する仕組み。<br>maxUnavailable: 10% とすると、常に 90% 以上の Pod が稼働することを保証|


### ヘルスチェック
k8s におけるヘルスチェックでは以下が Pod に対して行われる。
readinessProbe はトラフィック制御、livenessProbe は再起動制御を担う。
startupProbe は起動が遅いアプリで、liveness の誤検知を防ぐために必要な場合のみ追加する。

|Probe|読み|目的|失敗したら|主な用途|
|:-|:-|:-|:-|:-|
|livenessProbe|ライブネス|生きてるか|コンテナ再起動|デッドロック検知|
|readinessProbe|レディネス|受信できるか|トラフィック停止|LB ルーティング制御|
|startupProbe|スタートアップ|起動完了したか|起動できないコンテナとして再起動される|起動が遅いアプリ|



## Kubernetes における通信と Load Balancing
Kubernetes では、Pod は動的に生成・削除され IP も変化するため、Pod は直接外部から通信を受けることを想定していない。
そのため、外部・内部からのアクセスは、Service / Ingress(L7) / Gateway API(L7) などの
抽象リソースを通じて公開される。

### Service
Service は、Pod を論理的に束ねて安定したアクセス手段（仮想IPとDNS名）を提供する仕組みである。
- Pod の IP 変動を吸収する
- L4（TCP/UDP）レベルでロードバランシングを行う
- 実際の転送処理は各 Node 上の kube-proxy が実装する

#### Service の主な種類
Service は L4 通信の公開方法を定義する。
- ClusterIP：クラスタ内部専用の仮想IPを提供
- NodePort：各 Node のポートを開放し外部から到達可能にする
- `LoadBalancer：NodePort の前段にクラウドの L4 Load Balancer を自動作成する`

それぞれの特徴を表に整理すると以下

|項目|ClusterIP|NodePort|LoadBalancer|
|:----|:----|:----|:----|
|外部Client接続|×|○（NodeIP経由）|○（LB経由）|
|内部Client接続|○|○|○|
|Cloud LB|×|×|○|
|到達経路|Pod→Service→Pod|Client→Node→Pod|Client→LB→Node→Pod|
|主用途|内部通信|簡易公開|本番外部公開|

#### Serviceにおける通信イメージ
Client IP
```
[Cluster内のClient Pod]
  ↓
ClusterIP（仮想IP）
  ↓
kube-proxy(iptables/IPVS)
  ↓
Pod
```

NodePort
```
[外部Client]
  ↓
NodeIP:NodePort
  ↓
kube-proxy(iptables/IPVS)
  ↓
Pod
```

LoadBalancer（デフォルト：NodePort）
```
[外部Client]
  ↓
Cloud L4 LB（パススルー） ← 自動作成
  ↓
Backend Service
  ↓
Instance Group（Node）
  ↓
NodePort                ← Service作成時に自動割当自動でNordPortが作成されてLBのバックエンドに設定
  ↓
kube-proxy(iptables/IPVS)
  ↓
Pod
```

LoadBalancer（バックエンドをNEGで指定する場合）
```
Client
  ↓
Cloud L4 Network LB
  ↓
Backend Service
  ↓
NEG（Pod IP）
  ↓
Pod
```


#### LoadBalancerのマニフェスト
k8sのマニフェストで、`type: LoadBalancer`を指定すると自動でL4 LBが作成される。
L4のパススルーのため、LBで通信を受けると、自動的にこのServiceで定義されているPodまで通信が届く。
```
apiVersion: v1
kind: Service
metadata:
  name: app-l4-external
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - name: tcp
    port: 80
    targetPort: 8080
```

内部のLBを作成したい場合は以下のように`balancer-type: "Internal"`を設定する。
```
apiVersion: v1
kind: Service
metadata:
  name: app-l4-internal
  annotations:
    networking.gke.io/load-balancer-type: "Internal"  
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - name: tcp
    port: 80
    targetPort: 8080
```

バックエンドをNEGに指定したい場合は、`cloud.google.com/neg`アノテーションを付与する
```
apiVersion: v1
kind: Service
metadata:
  name: app-l4
  annotations:
    cloud.google.com/neg: '{"exposed_ports": {"80":{}}}'
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```


### Ingress
Ingress は、Kubernetes における 外部からクラスタ内への HTTP(S) トラフィックのルーティングルールを定義する L7 リソースである。
- Ingress 自体はトラフィックを処理しない
- 「どの Host / Path を、どの Service に転送するか」を宣言する
- 実際の通信処理は Ingress Controller が担う
- `GKE では GCE Ingress Controller が GCP の L7 Load Balancer を構成する`


#### Ingressにおける通信イメージ
IngressにおいてはServiceとは異なり、NodePort や kube-proxy は経由しない。
NEGを経由したContainer-native LB通信を行う。
```
[外部Client]
  ↓
DNS(example.com)
  ↓
GCP HTTPS Load Balancer  ← GCPリソース（自動作成）
  ↓
URL Map
  ↓
Backend Service
  ↓
NEG（istio-ingressgateway Pod）
  ↓
Istio IngressGateway（Envoy）
  ↓
VirtualService
  ↓
app-svc（ClusterIP）
  ↓
Pod
```

#### Ingressのマニフェスト
Ingress は Service をバックエンドとして参照するため、以下のようなマニフェストが必要

下記の補足
- Cloud HTTPS LB（Ingress）は 入口としてIGWに集約（原則 / を全部）
- パス/ホストでの詳細ルーティングは VirtualService 側でやる前提


1. Ingress（Host/Path → Service）
`kind: Ingress`を作成することで、自動的に、L7のHTTPS LBが作成される。
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ing
  namespace: istio-system
spec:
  ingressClassName: gce
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: istio-ingressgateway
            port:
              number: 80
```

2. istioのService
cloud.google.com/negのアノテーションを追加することで、GKEはそのServiceに対応するPod用のNegを作成する
```
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-system
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: ClusterIP
  selector:
    istio: ingressgateway
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

3. アプリ Deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 8080
```

4. アプリ Service
※ この Service は Ingress のバックエンドではなく、VirtualService から参照される
```
apiVersion: v1
kind: Service
metadata:
  name: app-svc
  namespace: app
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```


### Gateway API
Gateway API は、Ingress の後継として設計された Kubernetes API であり、
「入口（インフラ責務）」と「ルーティング（アプリ責務）」を明確に分離する」ことを目的とする。

主なリソース：
- GatewayClass：どの種類の Load Balancer を使うか（実装の選択）
- Gateway：トラフィックの入口（ポート・公開範囲）
- HTTPRoute：L7 ルーティングルール（Host / Path / Header → Service）

Ingress が「1リソースに責務が集中」していたのに対し、
Gateway API は、GatewayとHTTPRouteを分けることで役割分離とマルチテナント性を重視した設計になっている。

#### Gatewayにおける通信イメージ
- `Gateway を作成すると GKE Controller が LB を構築`
- HTTPRoute を追加すると URL Map / Backend 設定が更新される
- NodePort や kube-proxy は経由しない（Container-native）

```
[外部Client]
  ↓
Global HTTP(S) Load Balancer   ← GCPリソース（自動作成）
  ↓
URL Map
  ↓
Backend Service
  ↓
NEG（Pod IP）
  ↓
Pod
```

### GatewayClass による LB 種別の指定
GKE では gatewayClassName によりどの種類の GCP Load Balancer を作るか が決まる。

- 軸① 公開範囲
  - External：インターネット向け（外部IP）
  - Internal：VPC 内向け

- 軸② スコープ
  - Global：Google エッジ（Anycast IP）
  - Regional：特定リージョン
  - Cross-Regional：内部向け複数リージョン

上記を踏まえると、設定できるGateway Classは以下の通り
- ※1. mcはマルチクラスター
- ※2. gxlbは次世代

|GatewayClass|External|Internal|Global|Regional|Cross-Regional|Multi-Cluster|
|:----|:----|:----|:----|:----|:----|:----|
|gke-l7-global-external-managed|○|−|○|−|−|−|
|gke-l7-regional-external-managed|○|−|−|○|−|−|
|gke-l7-rilb|−|○|−|○|−|−|
|gke-l7-cross-regional-internal-managed-mc|−|○|−|−|○|○|
|gke-l7-global-external-managed-mc ※1|○|−|○|−|−|○|
|gke-l7-regional-external-managed-mc ※1|○|−|−|○|−|○|
|gke-l7-rilb-mc ※1|−|○|−|○|−|○|
|gke-l7-gxlb ※2|○|−|○|−|−|−|
|gke-l7-gxlb-mc ※2|○|−|○|−|−|○|

#### GatewayClassのマニフェスト
① Gateway（入口定義）
- gatewayClassName で LB種別を選択
- listeners で公開ポートとホスト制限を定義
- 作成時に GCP 側に LB が構築される（GKE環境）
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: app-gw
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: example.com
```

② HTTPRoute（ルーティング定義）
- parentRefs で Gateway に紐付け
- Host / Path → Service を定義
- URL Map / Backend Service が更新される

```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
spec:
  parentRefs:
  - name: app-gw
  hostnames:
  - example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: app-svc
      port: 80
```



### NEG（Network Endpoint Group）について
NEG（Network Endpoint Group）とは、
Load Balancer がトラフィックを転送する“実際のエンドポイント集合”を表す GCP のバックエンドリソース

- Kubernetes のリソースではない
- GCP 側の Load Balancer 構成要素
- Backend Service に紐付けられる

NEGで設定できるものは、
- VM インスタンス
- 特定のIP:Port
- GKE の場合：Pod IP（Container-native Load Balancing）

GKEでは、主に`PodのIPが直接NEGとして設定される`


#### NEGを利用した通信のイメージ
NEGを使わない従来構成（Instance Group経由）
- LBはNodeをバックエンドに持つ
- NodePort経由
- kube-proxyでPodへ転送
- ヘルスチェックはNode単位

```
Client
  ↓
Load Balancer
  ↓
Backend Service
  ↓
Instance Group（Node）
  ↓
NodePort
  ↓
kube-proxy
  ↓
Pod
```


NEG構成（Container-native）
- Podが直接バックエンドになる
- NodePortをスキップ
- kube-proxyを経由しない
- ヘルスチェックはPod単位
- スケール追従が高速
```
Client
  ↓
Load Balancer
  ↓
Backend Service
  ↓
NEG（Pod IP）
  ↓
Pod
```

#### Kubernetes側でのNEG指定方法
serviceとして`cloud.google.com/neg`を指定
```
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-system
  annotations:
    cloud.google.com/neg: '{"ingress": true}'   # (入口がL7の場合)
spec:
  type: ClusterIP
  selector:
    istio: ingressgateway
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

## 永続ストレージと容量管理
Kubernetes では、コンテナのライフサイクルとは独立してデータを保持するために、永続ストレージ（Persistent Storage） の仕組みが用意されている。

この仕組みの特徴は、クラウドプロバイダが提供するディスクを直接扱わず、Kubernetes が抽象化したリソースを介して利用するという点にある。

### ストレージを構成するレイヤ
Kubernetes における永続ストレージは、以下のレイヤで構成される。
- Persistent Disk（PD）
    - クラウドプロバイダ（GCP など）が提供する実体のディスク
- PersistentVolume（PV）
    - Kubernetes に登録されたストレージリソース
- PersistentVolumeClaim（PVC）
    - Pod が要求するストレージの宣言（サイズやクラス）

Pod は PV や PD を直接参照せず、PVC を通じてのみストレージを利用する。
永続ストレージの容量を拡張する場合は、PVC の要求サイズを更新する ことで Kubernetes に変更を伝える必要がある。

## k8sのデバッグ
kubectlのデバッグコマンドには3つのアプローチ方法がある
- Ephemeral Container: Podの中にEphemeral Containerを追加してデバグ対象のコンテナのツール不足を補う。基本的には参照権限のみがデフォルト。
- Copied Pod: デバック用のPodをコピーを作成して、デバッグを行う。Ephemeral Containerのようにサイドカーなどを作成してあげて、デバッグを支援する。
- Node Debug: ノード上で動作する特権Podを作成し、Podの中からホストのファイルシステム画角人cBeanstalk系、ノードのログやネットワーク状態の調査が可能。

Debug Profileとは、デバッグように作成されるコンテナに対して、どのような権限やリソースアクセスを許可するかを定義するプリセット。k8sでは、デフォルトでコンテナの権限が厳しく制限されているため、調査するために適切なプロフィールを選択するためのコマンド。

## Prometheus
Prometheusとは、システムやアプリケーションの状態を数値（メトリクス）として収集・保存・検索・可視化するオープンソースの監視ツール。

Prometheusは「Kubernetes上のどの メトリクスエンドポイント を、どの条件で収集するか」を定義・制御できる仕組みである。
Cloudのモニタリング機能単体でも標準メトリクスは取得できるが、アプリやKubernetes内部の詳細なメトリクスは Prometheus 形式を使わないと扱えないものが多い。

Prometheus はオープンソースの監視ツールであり、Kubernetes 専用ではないが、Kubernetes と非常に相性が良いため、K8s 環境で利用されることが多い。

Kubernetes との相性の良さの本質は、Prometheus が Pull 型の監視方式を採用している点にある。

Kubernetes 環境では、Pod やコンテナが頻繁に起動・停止（短命）を繰り返す。
このような環境で Push 型の監視方式を採用すると、コンテナ側でメトリクス送信先の設定や登録処理が必要となり、登録後すぐにコンテナが停止した場合、登録解除や後処理を行えない という問題が発生する。

一方、Pull 型である Prometheus では、Prometheus 自身が Kubernetes API を参照し、生存している Pod / コンテナの一覧を自動的に取得したうえで、メトリクスを収集する。

そのため、
- コンテナ側は送信先や登録処理を意識する必要がない
- コンテナが停止すれば、自動的に収集対象から外れる
- 登録・解除といったライフサイクル管理が不要

という特性を持ち、短命で動的な Kubernetes 環境に自然に適合する監視方式となっている。

# Istio
Istioとはk8s上で動くサービスメッシュの仕組み。Istioを利用することによってk8s上の通信をより高度に制御することが可能になる。

Istioは、Envoyプロキシをサイドカーとして各Podに配置し、iptablesによってpodからの通信をEnvoyにリダイレクトする。
EnvoyはIstioで定義されたVirtualServiceやDestinationRuleに基づき、トラフィック制御を行う。

後述するが、`VirtualService が「行き先の名前」を決め、DestinationRule が「その名前の実体（Pod 群）」を定義する`

## VirtualService
VSは`namespaceに登録しておく送信先の看板`のイメージ。

Envoyは、istiodから配布された設定の中から、通信先のhostに一致するVirtualServiceを参照する。
一致するVSがない場合は通常通りの通信としてSVCに対して通信が行われるが、
該当するVSがある場合は、VirtualServiceおよび関連するDestinationRuleの定義に従って、通信が制御される。

### VSの種類
VSは、IGW(IngressGateway)に関連する場合と、Pod間通信に関連する場合の２つに大別することができる。

#### VS for IGW
IGWに関連する場合は、マニフェストでもspec.gatewaysを指定する。
IGWに関連する場合、EnvoyはアプリケーションのPodのサイドカーとして通信を奪うという動きというよりも、送信を行う主体として動作する。

#### VS for Pod
Pod間の通信に関連する場合は、上記のgatewaysの指定は不要。
pod内のアプリケーションが、DNSの名前解決を行い、ClusterIPで通信を開始したタイミングでiptablesによりEnvoyに通信が流れる。
Envoyは、通信の宛先情報（Hostヘッダやoriginal destination）をもとに、VirtualServiceのhostsと照合する。
VirtualServiceおよびDestinationRuleの定義をもとに、Envoyが保持するService/Endpoint情報から実際の通信先Pod（IP）を選択する。

|観点|Ingress Gateway 関連|Pod間通信（mesh）|
|:----|:----|:----|
|主な用途|外部 → クラスタ内通信|クラスタ内サービス間通信|
|spec.gateways|明示的に指定（例: istio-system/my-gateway）|指定しない（mesh が暗黙）|
|通信を処理するEnvoy|Ingress Gateway Pod の Envoy|各Podのサイドカー Envoy|
|通信の流れ|Gatewayが受信し送信主体として動作|iptables によりサイドカーへリダイレクト|
|host の意味|外部向けドメイン名|Service の FQDN|
|主なユースケース|外部公開、API Gateway|カナリア、A/B、blue/green|


### VSの具体例
以下に VirtualService の具体例を示す。
今回の例は、blue/greenデプロイに利用される例であり、後続のDRの説明時でも同様の前提で具体例を記載する。

|大分類|小分類|フィールド|例|役割・意味|
|:----|:----|:----|:----|:----|
|メタ情報|おまじない|apiVersion|networking.istio.io/v1beta1|VirtualService の API バージョン|
|メタ情報|おまじない|kind|VirtualService|リソース種別|
|メタ情報|メタ定義|metadata.name|myapp|VirtualService 名|
|メタ情報|メタ定義|metadata.namespace|prod|適用される Namespace|
|ルーティング定義|host設定|spec.hosts|myapp.example.com|この VS が制御する宛先ホスト|
|ルーティング定義|gateway設定|spec.gateways|istio-system/myapp-gateway|この VS を適用する入口（Ingress Gateway）|
|ルーティング定義|HTTP制御|spec.http|―|HTTP リクエスト単位の制御定義（上から順に評価）|
|ルーティング定義|HTTP制御 / マッチ|match.uri.prefix|/green|/green で始まるリクエストに一致|
|ルーティング定義|HTTP制御 / ルーティング時制御|rewrite.uri|/|バックエンドへ送る際にパスを書き換え|
|ルーティング定義|HTTP制御 / 宛先host|route.destination.host|myapp.prod.svc.cluster.local|転送先 Kubernetes Service|
|ルーティング定義|HTTP制御 / 宛先サブセット（DR）|route.destination.subset|green|green バージョンの Pod 群を指す論理名|
|ルーティング定義|HTTP制御 / マッチ|match.uri.prefix|/blue|/blue で始まるリクエストに一致|
|ルーティング定義|HTTP制御 / ルーティング時制御|rewrite.uri|/|パスを書き換え|
|ルーティング定義|HTTP制御 / 宛先host|route.destination.host|myapp.prod.svc.cluster.local|転送先 Kubernetes Service|
|ルーティング定義|HTTP制御 / 宛先サブセット（DR）|route.destination.subset|blue|blue バージョンの Pod 群を指す論理名|
|ルーティング定義|HTTP制御 / デフォルト|route.destination.host|myapp.prod.svc.cluster.local|上記に一致しない場合の転送先|
|ルーティング定義|HTTP制御 / デフォルト|route.destination.subset|blue|デフォルトは blue|

マニュフェストの例
```
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
  namespace: prod

spec:
  # このVirtualServiceが適用されるホスト
  hosts:
    - myapp.example.com

  # Ingress Gateway 経由の通信に適用
  gateways:
    - istio-system/myapp-gateway

  http:
    # /green へのリクエストは green にルーティング
    - match:
        - uri:
            prefix: /green
      rewrite:
        uri: /
      route:
        - destination:
            host: myapp.prod.svc.cluster.local
            subset: green

    # /blue へのリクエストは blue にルーティング
    - match:
        - uri:
            prefix: /blue
      rewrite:
        uri: /
      route:
        - destination:
            host: myapp.prod.svc.cluster.local
            subset: blue

    # それ以外のパスはデフォルトで blue
    - route:
        - destination:
            host: myapp.prod.svc.cluster.local
            subset: blue

```


サービスの例
```
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: prod
spec:
  selector:
    app: myapp
```

Service の name / namespace により
myapp.prod.svc.cluster.local という Service FQDN が決まり、
selector に一致する Pod 群が実際の通信先となる。

## DestionationRule
DR（DestinationRule）は、
`Deploymentで定義されたPodの分類をVirtualServiceが利用できる形に整理するための橋渡し`というイメージ。

DR（DestinationRule）は、k8s内の通信に対して、どの Pod 群（subset）をどのように扱うかを定義するリソースである。
Deployment などで付与された Pod のラベルをもとに subsets を定義し、VirtualService から指定された subset に対応する Pod 群を Envoy が選択できるようにする。

VirtualService は「どの subset に送るか」を決定し、
DestinationRule は「その subset がどの Pod で構成されるか」を定義する。
この 2 つを組み合わせることで、特定バージョンの Pod へのルーティング制御が可能になる。

### DRの例
Blue GreenデプロイのDRの例
```
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
  namespace: prod

spec:
  # VS の destination.host と一致させる
  host: myapp.prod.svc.cluster.local

  # subset 名（blue/green）を Pod のラベル条件に紐付ける
  subsets:
    - name: blue
      labels:
        version: blue

    - name: green
      labels:
        version: green
```

デプロイメントで以下のように、labelsとしてversion: blueを定義しているからこそ、以下のDeploymentをターゲットとして通信することができる。
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
  namespace: prod
spec:
  template:
    metadata:
      labels:
        app: myapp
        version: blue
```


## PeerAuthentication
PeerAuthentication は、Istio における mTLS（相互 TLS）の受信側ポリシーを定義するリソースである。
namespace 全体、または特定の workload（Pod）単位で、通信を mTLS 必須にするかどうかを制御する。

- VirtualService / DestinationRule が「どこへ・どう流すか」を制御するのに対し、
- PeerAuthentication は「その通信を暗号化・認証必須にするか」を制御する。

PeerAuthentication は 受信側（Server side）の Envoy に適用されるポリシーである。
ただし、受信側で mTLS を必須（STRICT）に設定した場合、送信側の Envoy もその要件を満たすために mTLS で通信を行う。

PAには以下の設定modeがある

|mode|意味|
|:----|:----|
|STRICT|mTLS のみ許可。平文通信は拒否|
|PERMISSIVE|mTLS / 平文どちらも許可（移行期間向け）|
|DISABLE|mTLS を無効化|

### PAの具体例
```
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: prod
spec:
  mtls:
    mode: STRICT
```

## Istioの通信まとめ

Ingress Gateway 経由

|カテゴリ|観点|主体|参照リソース|内容|
|:----|:----|:----|:----|:----|
|通信開始|通信元|外部クライアント|―|ブラウザや外部システムからリクエスト|
|入口|受信ポイント|Ingress Gateway（Envoy）|Gateway|外部通信の入口となる Envoy|
|入口|適用VS判定|Ingress Gateway（Envoy）|VirtualService|host / gateway が一致する VS を選択|
|ルーティング|条件判定|Ingress Gateway（Envoy）|VirtualService|Path / Header などでルール評価|
|ルーティング|宛先決定|Ingress Gateway（Envoy）|VirtualService|destination.host / subset を決定|
|Pod選択|subset解決|Ingress Gateway（Envoy）|DestinationRule|subset 名を Pod ラベル条件に変換|
|Pod選択|候補抽出|Ingress Gateway（Envoy）|Service|selector に一致する Pod を取得|
|Pod選択|最終選択|Ingress Gateway（Envoy）|Endpoint|対象 Pod IP をロードバランシング|
|セキュリティ|mTLS要否|受信側 Pod|PeerAuthentication|mTLS 必須かを判定|
|セキュリティ|通信方式|Ingress Gateway（Envoy）|PeerAuthentication|受信側要件に従い mTLS で送信|
|実通信|転送|Ingress Gateway → Pod|―|選択された Pod へ通信|

Pod 間通信

|カテゴリ|観点|主体|参照リソース|内容|
|:----|:----|:----|:----|:----|
|通信開始|通信元|Pod 内アプリ|―|アプリケーションが通信を開始|
|通信開始|名前解決|Pod|DNS / Service|Service 名 → ClusterIP|
|入口|通信奪取|iptables|―|通信をサイドカー Envoy にリダイレクト|
|入口|送信主体|サイドカー Envoy|―|実際の送信を Envoy が担当|
|ルーティング|適用VS判定|サイドカー Envoy|VirtualService|host が一致する VS（mesh）を選択|
|ルーティング|条件判定|サイドカー Envoy|VirtualService|Path / Header などでルール評価|
|ルーティング|宛先決定|サイドカー Envoy|VirtualService|destination.host / subset を決定|
|Pod選択|subset解決|サイドカー Envoy|DestinationRule|subset 名を Pod ラベル条件に変換|
|Pod選択|候補抽出|サイドカー Envoy|Service|selector に一致する Pod を取得|
|Pod選択|最終選択|サイドカー Envoy|Endpoint|対象 Pod IP をロードバランシング|
|セキュリティ|mTLS要否|受信側 Pod|PeerAuthentication|mTLS 必須かを判定|
|セキュリティ|通信方式|サイドカー Envoy|PeerAuthentication|受信側要件に従い mTLS で送信|
|実通信|転送|サイドカー → Pod|―|選択された Pod へ通信|
