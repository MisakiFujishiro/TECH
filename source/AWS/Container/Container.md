# AWSにおけるコンテナサービス
AWSではオーケストレーションサービスにはECSとEKSがあり、そのどちらを利用するかは案件特性とそのサービスの特徴を踏まえて判断する必要がある。

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