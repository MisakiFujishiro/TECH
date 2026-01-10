# Load Balancing
Google Cloud における Load Balancing（LB）は、  
単なるトラフィック分散装置ではなく、Google のグローバルネットワークへの入口として設計されている。

AWS の ALB / NLB と似た用語は存在するが、  
GCP の LB は 「グローバル制御プレーン + 分散データプレーン」 という思想を前提に理解する必要がある。

## GCP Load Balancing の本質
GCP の Load Balancing の本質は次の 3 点に集約される。
- Google のエッジ（PoP）で最初にトラフィックを受ける
- 制御はグローバル、転送は最寄りエッジ
- バックエンドの種類を問わず同一思想で扱える

つまり、
「どこから来た通信であっても、Google のネットワークに入った瞬間に最適化される」
という前提で設計されている。

## GCP Load Balancer の分類軸
GCP の LB は、以下の 4 つの軸で整理すると理解しやすい。

|分類軸|区分|概要|代表例・補足|
|:--|:--|:--|:--|
|スコープ|Global|世界中の Google エッジ（PoP）でトラフィックを受信|External HTTP(S) Load Balancer など|
| |Regional|特定リージョン内で完結|Internal Load Balancer など|
|公開範囲|External|インターネットから到達可能|外部公開 Web / API|
| |Internal|VPC 内限定（RFC1918）|マイクロサービス間通信|
|プロキシ方式|Proxy 型|LB が通信を終端し、再接続する方式（L7/L4）|HTTP(S), SSL Proxy, TCP Proxy|
| |Pass-through 型|IP を保持したまま転送|Internal TCP/UDP Load Balancer|
|プロトコルレイヤ|L7|アプリケーション層で制御|HTTP / HTTPS（URL, Host, Path ルーティング）|
| |L4|トランスポート層で制御|TCP / UDP（ポート単位）|

## Load Balancer を構成する要素
GCP の LB は、必ず次の構成要素に分解できる。

|構成要素|役割|主な内容|補足・ポイント|
|:--|:--|:--|:--|
|Frontend|クライアント接続点|IP アドレス / ポート / プロトコル|External LB では Anycast IP が割り当てられ、世界中のエッジで受信される|
|Routing / Proxy|トラフィック制御|Host / Path / URL ルーティング<br>TLS 終端（HTTPS）|Proxy 型 LB の中核。L7 制御や証明書管理を担当|
|Backend|転送先リソース|Compute Engine（MIG）<br>GKE（NEG）<br>Cloud Run<br>Cloud Functions|バックエンド種別を意識せず同一モデルで扱えるのが GCP の特徴|
|Health Check|生存判定|HTTP / TCP などによる定期チェック|LB の転送判断は ヘルスチェック結果が絶対。Unhealthy なバックエンドには送られない|



## 代表的な Load Balancer
### External HTTP(S) Load Balancer
- グローバル
- L7 Proxy 型
- Anycast IP
- Cloud Run / GKE / MIG に対応

用途
- インターネット公開 Web
- グローバルサービス


### Internal HTTP(S) Load Balancer
- リージョナル
- VPC 内専用
- L7 ルーティング可能

用途
- マイクロサービス間通信
- 社内向け API


### TCP / UDP Load Balancer

#### Proxy 型（TCP Proxy / SSL Proxy）
- L4 Proxy
- クライアント IP を抽象化
- TLS 終端可能（SSL Proxy）

#### Pass-through 型
- IP を保持したまま転送
- 低レイヤ制御向け
- 内部通信で多用
