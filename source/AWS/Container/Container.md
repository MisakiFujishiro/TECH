# AWSにおけるコンテナサービス
AWSではコンテナサービスとして充実したサービスラインナップがある。

![](../../img/AWS/Container/container_services.png)
[Amazon ECS 入門【AWS Black Belt】](https://www.youtube.com/watch?v=stpz_SNiWmA&list=PLzWGOASvSx6FIwIC2X1nObr1KcMCBBlqY&index=4)

## オーケストレーションサービス
AWSではオーケストレーションサービスにはECSとEKSがあり、そのどちらを利用するかは案件特性とそのサービスの特徴を踏まえて判断する必要がある。

ECSの最大の特徴はそのシンプルさがメリットである。構築や導入についてもスピーディーに適用することができる。  
EKSの最大の特徴は、Kubernetesというオープンソースの特徴を活かして、周辺の行為範囲なコミュニティの再利用性。さらに、ACK(AWS Contreollers for Kubernetes)を利用するとK8sとAWSの連携もシームレスになる。


|項目|Amazon ECS|Amazon EKS|
|:----|:----|:----|
|管理方法|AWSによるマネージドサービス|AWSによるマネージドKubernetesサービス|
|簡便性|シンプルでAWSサービスと緊密に統合(ALB etc..)|Kubernetesの標準機能を活用|
|オーケストレーションエンジン|ECS独自のエンジン|Kubernetesエンジン|
|設定の容易さ|簡単にセットアップ可能|Kubernetesの知識が必要|
|マルチクラウド対応|制限あり|マルチクラウドに対応しやすい|
|カスタマイズ性|比較的少ない|高い|
|スケーラビリティ|自動スケーリングが容易|Kubernetesの自動スケーリングを利用可能|
|コスト|使用するリソースに応じた課金|Kubernetesクラスターの管理コストが発生|

