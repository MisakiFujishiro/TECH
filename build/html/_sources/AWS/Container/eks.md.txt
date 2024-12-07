# EKS
EKSは、AWSが提供するKubernetesに基づいたコンテナのオーケストレーションサービス。
オーケストレーションサービスを利用することで簡単に、以下のオーケストレーション操作を行うことができる
- コンテナのデプロイ
- ポート管理
- スケーリング
- ヘルスチェックや自動復旧

![](../../img/AWS/Container/container_services.png)
[Amazon Elastic Kubernetes Service(Amazon EKS) 入門【AWS Black Belt】](https://www.youtube.com/watch?v=E5TEqZhS0D0)

EKSとECSの最も大きな違いはKubernetesを実行するためのマネージドサービスである点

## Kubernetesについて
Kubernetesとは、コンテナを複数のホストにまたがって管理するオープンソースのコンテナオーケストレーションサービス。
ECS同様にオーケストレーションの機能を有しているが、オープンソースであるという特徴からさまざまなエコシステムとの連携を行うことができる。

### Kubernetesの課題
Kubernetesは、分散システムであり、コントロールプレーン・データプレーンでさまざまなコンポーネントだ動作しており、運用管理が煩雑になりがちである。

## EKSの価値
Kubernetesの課題である、運用の煩雑性について抑制しながらにKubernetesを利用することができるのがEKSのメリットである。

EKSを利用することでKubernetesのコントロールプレーンやデータプレーンの管理を容易にすることができる。
- コントロールプレーンをマネージドサービスとしてAWSが管理・提供
- データプレーンの提供（EC2/Fargate）
- アドオンによりエコシステムの連携
- AWSサービスとの連携（ネットワーク・セキュリティ・ストレージ

## EKSの仕組み
### EKSのアーキテクチャ
コントロールプレーンは、AWSが管理するアカウント上に存在する。データプレーンはユーザーのAWSアカウント上に存在する。

![](../../img/AWS/Container/eck_architecture.png)
[Amazon Elastic Kubernetes Service(Amazon EKS) 入門【AWS Black Belt】](https://www.youtube.com/watch?v=E5TEqZhS0D0)

