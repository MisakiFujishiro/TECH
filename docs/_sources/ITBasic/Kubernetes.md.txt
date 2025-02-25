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


