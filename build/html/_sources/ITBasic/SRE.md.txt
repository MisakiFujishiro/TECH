# SRE
## Observability
Observabilityとは、外から見えるデータを利用してシステムの内部状態を理解できる能力。
以下のMetrics/Traces/Logs/(Profiling)の要素を組み合わせて、Observabilityを担保する。

|要素|役割|フェーズ|
|:----|:----|:----|
|Metrics|状態の定量把握|予防・健全性監視|
|Traces|リクエストの経路把握|影響範囲特定|
|Logs|事実・イベントの記録|根本原因分析|
|Profiling|内部実装のボトルネック分析|最適化|
