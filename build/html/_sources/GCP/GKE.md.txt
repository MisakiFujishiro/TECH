# GKE
## Workload Identity
GKEのKSAでGoogleCloud IAM SAを利用する際の詳細はIAM側で解説。
GSAを利用しつつ、GSAを代替利用することができる。

## Managed Service for Prometheus
Prometheus 互換のメトリクスをフルマネージドで扱える Google Cloud のサービスであり、Prometheusのストレージ管理や高可用性、スケーリングを意識せずに PromQL を利用できる。

GKE 環境では、マニフェストにラベルやアノテーションを設定するだけで、Pod や Node のメトリクスを自動的に収集できる。

