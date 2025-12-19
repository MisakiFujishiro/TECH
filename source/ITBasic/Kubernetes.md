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
|ConfigMap/Secret|ECSのタスク定義の環境変数管理|設定値や機密情報（DBパスワードなど）を管理。|

### Pod
複数コンテナが1つの Linux ネットワーク名前空間を共有している。  

そのため、PodでSQL Proxyを設定して、アプリ側でlocalhost:5432の通信をすると、Proxy側がlocalhost:5432としてlistenしており、結果としてSQLへの通信を代替してくれる。

### Node
ノードには2種類存在し、MasterNode（コントロールプレーン）とWorkerNode（データプレーン）がある。



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