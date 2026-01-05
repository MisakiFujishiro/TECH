# Kubernetes
Kubernetesはコンテナのオーケストレーションサービスであり、コンテナのデプロイや自動修復、デプロイ管理、ロードバランシングなどの操作を支援する

## 主要なコンポーネント
ECSと比較しながら、	Kubernetesで扱われる用語を整理すると以下。

|Kubernetes|ECS|説明|
|:----|:----|:----|
|Pod|ECSのタスク|コンテナをまとめて管理する単位（1つ以上のコンテナを含む）。|
|Node|ECSのFargate/ECSインスタンス|Podが実行されるホスト（EC2またはFargate上で動作）。|
|Deployment|ECSのサービス|Podの管理をする仕組み。スケーリングやローリングアップデートを管理。|
|Service|ECSのロードバランサ|内部/外部向けにPodを公開し、負荷分散を提供。|
|Namespace|1つの Kubernetes クラスタの中を論理的に区切るための「仮想的な区画」|
|ConfigMap/Secret|ECSのタスク定義の環境変数管理|設定値や機密情報（DBパスワードなど）を管理。|

### Pod
複数コンテナが1つの Linux ネットワーク名前空間を共有している。  

そのため、PodでSQL Proxyを設定して、アプリ側でlocalhost:5432の通信をすると、Proxy側がlocalhost:5432としてlistenしており、結果としてSQLへの通信を代替してくれる。

### Node
ノードには2種類存在し、MasterNode（コントロールプレーン）とWorkerNode（データプレーン）がある。

### Deployment
Deploymentは、運用・スケール・自己修復の責務を担う

以下の事柄が責任範囲
- 何個のpodを動かすか
- podの自動修復
- ローリングアップデート

### Service
Serviceとは、Pod の集合に対して、安定した到達手段を提供する

Serviceは、ネットワーク・発見・安定アドレスの責務を担う
- pod群に対する固定IP/DNSの提供
- podの増減を隠蔽
- L4のロードバランシング

具体的には、Service が 代表となる IP アドレスや DNS 名を持ち、その宛先に届いた通信を 内部の Pod 群のいずれかへ振り分けることで、呼び出し元から Pod の存在や入れ替わりを意識させない仕組みを提供する。

### namespace
Namespace を使うと、同一クラスタ内で以下を分離できる。
- Pod
- Deployment
- Service
- ConfigMap / Secret
- RBAC（権限）
- ResourceQuota（使用量）

```
cluster
 ├─ namespace: team-a
 │   ├─ deployment: api
 │   └─ service: api
 ├─ namespace: team-b
 │   ├─ deployment: api
 │   └─ service: api
```

Namespace は 「置き場所を分ける」だけ であって：
- 誰が操作できるか
- どれだけリソースを使えるか

は 別途設定が必要。

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

#### 分離まとめ
K8sにおける分離の種別をまとめると以下

|レベル|手段|コスト|
|:----|:----|:----|
|論理分離|Namespace|低|
|権限制御|RBAC|低|
|資源制御|ResourceQuota|低|
|物理分離|複数クラスタ|高|



### Podの起動数の制御
|項目|略称|詳細|
|:----|:----|:----|
|Horizontal Pod Autoscaler |HPA|ワークロードの負荷に応じて自動的にレプリカ数をスケールアウト・スケールインするための仕組み|
|Vertical Pod Autodcaler|VPA|Podのリソースの割り当てを調整する仕組み|
|Pod Dsiruption Budget|PDB|ノードのメンテナンスなどアップグレードなどによる意図的なPodの終了（Disruption）が行われる際のPod数が許容範囲内に収まるようにするための仕組み。<br>maxUnavailable: 10にすると、90%は常に稼働などの設定ができる。|

## マニュフェスト(Manufest)
Kubernetesにおける主要なコンポーネントで解説したリソースたちはManufestと呼ばれるyamlファイルで管理する。

## kubectl
Kubernetesについて実際に操作する際にはkubectlコマンドを利用する。
### クラスターやノードへの操作
|コマンド|説明|
|:----|:----|
|kubectl cluster-info|クラスタの情報を表示|
|kubectl get nodes|クラスタ内のノード一覧を取得|
|kubectl describe node <NODE_NAME>|指定したノードの詳細情報を表示|
|kubectl top node|ノードのリソース使用状況を確認|

### Podsへの操作
|コマンド|説明|
|:----|:----|
|kubectl get pods|Pod の一覧を表示|
|kubectl get pods -o wide|詳細な情報付きで Pod の一覧を表示|
|kubectl describe pod <POD_NAME>|指定した Pod の詳細情報を表示|
|kubectl logs <POD_NAME>|指定した Pod のログを表示|
|kubectl logs -f <POD_NAME>|リアルタイムでログを監視|
|kubectl exec -it <POD_NAME> -- /bin/sh|Pod 内のコンテナに入る（shがない場合は/bin/bash）|
|kubectl delete pod <POD_NAME>|指定した Pod を削除|

### Deployへの操作
|コマンド|説明|
|:----|:----|
|kubectl get deployments|Deployment の一覧を取得|
|kubectl describe deployment <DEPLOYMENT_NAME>|Deployment の詳細情報を表示|
|kubectl rollout status deployment <DEPLOYMENT_NAME>|Deployment のデプロイ進行状況を確認|
|kubectl rollout history deployment <DEPLOYMENT_NAME>|過去のデプロイ履歴を確認|
|kubectl rollout undo deployment <DEPLOYMENT_NAME>|前のバージョンにロールバック|
|kubectl scale deployment <DEPLOYMENT_NAME> --replicas=5|Pod のレプリカ数を変更|
|kubectl delete deployment <DEPLOYMENT_NAME>|Deployment を削除|

### Serviceへの操作
|コマンド|説明|
|:----|:----|
|kubectl get services|Service の一覧を取得|
|kubectl describe service <SERVICE_NAME>|Service の詳細情報を表示|
|kubectl get service <SERVICE_NAME> -o yaml|Service のYAMLを取得|
|kubectl delete service <SERVICE_NAME>|Service を削除|

### ConfigMap & Secret の操作
|コマンド|説明|
|:----|:----|
|kubectl get configmaps|ConfigMap の一覧を取得|
|kubectl describe configmap <CONFIGMAP_NAME>|ConfigMap の詳細情報を表示|
|kubectl get secret|Secret の一覧を取得|
|kubectl describe secret <SECRET_NAME>|Secret の詳細情報を表示|
|kubectl delete configmap <CONFIGMAP_NAME>|ConfigMap を削除|
|kubectl delete secret <SECRET_NAME>|Secret を削除|

### リソースの作成
|コマンド|説明|
|:----|:----|
|kubectl apply -f <FILE>.yaml|YAMLファイルを適用（作成または更新）|
|kubectl create -f <FILE>.yaml|YAMLファイルからリソースを作成|
|kubectl delete -f <FILE>.yaml|YAMLファイルに記載されたリソースを削除|
|kubectl get all|クラスタ内のすべてのリソースを一覧表示|



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