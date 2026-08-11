# DB
Google Cloudで提供される代表的なデータサービスには、RDB系のCloud SQL / AlloyDB / Spannerと、NoSQL系のFirestore / Bigtableがある。

また、アプリケーションのトランザクションを処理するデータベースではないが、大規模な分析・レポーティング用途ではBigQueryも重要な選択肢になる。

簡単なまとめは以下の通り。

|サービス|データモデル|得意な用途|
|:----|:----|:----|
|Cloud SQL|リレーショナル SQL|一般的なOLTP、既存MySQL / PostgreSQL / SQL Serverの移行|
|AlloyDB|リレーショナル PostgreSQL互換|高性能なPostgreSQL系OLTP、HTAP|
|Spanner|分散リレーショナル SQL|大規模OLTP、グローバル分散、強整合性|
|Firestore|ドキュメント NoSQL|Web / モバイルアプリ、リアルタイム同期|
|Bigtable|ワイドカラム NoSQL|時系列、大量Key-Valueアクセス、低レイテンシ|
|BigQuery|分析データウェアハウス|大規模分析、レポート、集計|

データベースを選ぶときには、単に「SQLが使えるか」「NoSQLか」だけではなく、何をスケールさせたいのかを見るとわかりやすい。

|要件|代表的な候補|
|:----|:----|
|一般的なRDB|Cloud SQL|
|PostgreSQL互換の高性能RDB|AlloyDB|
|書き込みも含めて水平スケールしたい|Spanner|
|グローバルな強整合トランザクション|Spanner|
|モバイル / Webのリアルタイム同期|Firestore|
|大量の時系列 / Key-Valueアクセス|Bigtable|
|数百TB以上の分析・レポート|BigQuery|

特にSpannerとBigQueryはどちらも巨大なデータを扱えるが、役割が異なる。

Spannerは「巨大な業務データベース」であり、BigQueryは「巨大な分析基盤」と考えるとよい。

例えば数百TBのマーケティングデータを集計・分析してレポートを作ることが中心ならBigQueryが向いている。

一方、数百TBのデータに対してユーザーの注文や残高更新などのトランザクションを継続的に処理するのであればSpannerが候補になる。


## Cloud SQL

Cloud SQLはMySQL、PostgreSQL、SQL Serverを提供するフルマネージドのリレーショナルデータベースサービス。

従来のRDBに近い考え方で利用できるため、既存アプリケーションの移行先として使いやすい。

Cloud SQLではCPU・メモリなどのマシン構成を選択し、必要に応じてスケールアップする。

Cloud SQLの基本的な特徴は次のようになる。

- 既存RDBとの互換性を維持しやすい
- バックアップ、PITR、HAなどをマネージドサービスとして利用できる
- Read Replicaによって読み取り処理を分散できる
- 書き込みは基本的に1つのPrimaryに集約される
- Spannerのように書き込みを複数ノードへ水平分散するデータベースではない

Cloud SQLにはEnterpriseとEnterprise Plusがあり、可用性や計画メンテナンス時の停止時間、DR機能などに違いがある。

HA構成のSLAもエディションによって異なり、現在はEnterprise HAが99.95%、Enterprise Plus HAが99.99%となっている。

### HA構成

Cloud SQLのHA構成は、主にゾーン障害やPrimary Instance障害からデータベースを保護するための仕組み。

典型的には次の構成になる。

    Region A

    Zone A
    Primary

        |
        | synchronous / managed HA
        v

    Zone B
    Standby

Primaryが利用できなくなるとCloud SQLが障害を検知し、StandbyがPrimaryとして処理を引き継ぐ。

アプリケーション側では同じIPアドレスや接続情報を使用できるため、DR時のRead Replica昇格とは異なり、通常は接続先を書き換える必要がない。

HAは

「同一リージョン内のゾーン障害への対策」

と考えるとわかりやすい。


#### Standby Instance

HAを有効化したCloud SQLの待機系がStandby Instance。

つまり、

    HA構成
      =
    Primary + Standby

という関係になる。

StandbyはRead Replicaとは用途が異なる。

|項目|HA Standby|Read Replica|
|:----|:----|:----|
|主目的|可用性|読み取り分散|
|通常の読み取り|不可|可能|
|通常の書き込み|不可|不可|
|障害時|自動フェイルオーバー|通常は自動フェイルオーバーしない|
|配置|Primaryと同一リージョン、別ゾーン|同一リージョンまたは別リージョン|
|レプリケーション|HA用|非同期レプリケーション|

「レプリカ」という大きな意味ではデータのコピーだが、Cloud SQLの設計上はHA StandbyとRead Replicaを別物として考えた方がよい。


### Read Replica

Read ReplicaはPrimaryのデータを複製した読み取り専用インスタンス。

    Primary
       |
       +---- Read Replica A
       |
       +---- Read Replica B
       |
       +---- Read Replica C

Primaryへの更新がReplicaへ非同期に伝搬する。

主な目的は読み取り処理のオフロード。

例えば、

- 月次レポート
- BI
- 分析処理
- 大量SELECT
- 読み取り中心のAPI

などをRead Replicaへ逃がすことでPrimaryの負荷を減らす。

重要なのは、

Read Replicaを作っただけではアプリケーションのクエリが自動分散されるわけではない

ということ。

アプリケーションや接続プール、ロードバランサなどで、

    WRITE -> Primary
    READ  -> Read Replica

というルーティングを設計する必要がある。


#### Read ReplicaにもHAを設定できる

Read Replica自体にもHAを有効化できる。

例えば、

    Primary HA

    Zone A             Zone B
    Primary  <------>  Standby


                async replication
                       |
                       v


    Read Replica HA

    Zone C             Zone D
    Replica   <------> Replica Standby

という構成も可能。

これは、

「分析処理はRead Replicaへ逃がしたいが、その分析基盤自体も停止させたくない」

という場合に利用できる。

またDR用のクロスリージョンRead ReplicaにHAを設定しておけば、DRでReplicaをPrimaryへ昇格した時点ですでにHA構成になっているというメリットもある。


#### Cross-region Read Replica

Read Replicaは別リージョンにも配置できる。

    Region A
    Primary HA
      |
      |
      | async replication
      |
      v
    Region B
    Read Replica

Cross-region Read Replicaには大きく2つの用途がある。

1. ユーザーに近いリージョンで読み取りを行う
2. リージョン障害に備えたDR

特にPDBEで重要なのは2つ目。


### Cloud SQLのHAとDR

HAとDRは似ているように見えるが、守っている障害範囲が違う。

|構成|守るもの|
|:----|:----|
|HA|インスタンス障害、ゾーン障害|
|Cross-region Replica|リージョン障害|

そのため、本番環境でゾーン障害とリージョン障害の両方を考えるなら、

    Region A

    Zone A
    Primary

    Zone B
    Standby


          async cross-region replication
                     |
                     v


    Region B

    Zone A
    Read Replica

という構成になる。

さらにDR先でも高可用性が必要なら、

    Region A

    Zone A              Zone B
    Primary  <--------> Standby


          async cross-region replication
                     |
                     v


    Region B

    Zone A              Zone B
    DR Replica <------> Standby

とする。

これがCloud SQLで覚えておきたい基本形。


### HAとDRの違い

ゾーン障害の場合。

    Region A

    Primary X
       |
       |
       v
    Standby -> Primary

Cloud SQLが自動フェイルオーバーする。

一方、リージョンそのものが利用できなくなった場合。

    Region A
    Primary X
    Standby X

          |
          | cross-region replication
          v

    Region B
    Read Replica
          |
          | promote
          v
    New Primary

通常のCross-region Read ReplicaではReplicaのPromoteが必要になる。

つまり、

HA
    自動フェイルオーバー

Cross-region Read Replicaによる通常DR
    Replicaを意図的にPromote

という違いがある。


### PromoteはHA Failoverとは意味が違う

HA Failoverでは、

    Primary
       |
       X
       |
    Standby
       |
       v
    New Primary

となり、Cloud SQLが一つのHAインスタンスとして管理する。

一方Read ReplicaのPromoteでは、

    Primary
       |
       v
    Read Replica

       Promote

    Primary
       
    New Primary

という形でレプリケーション関係そのものが変わる。

そのため通常のRead Replica Promotionでは、

「障害が直ったから旧Primaryを起動すれば元通り」

とはならない。


### DR後に元リージョンへ戻す

例えば元々、

    Region A
    DB-1 Primary

       |
       +--------> Region B DB-2 Replica
       |
       +--------> Region C DB-3 Replica

だったとする。

Region A障害によってDB-2をPromoteすると、

    Region B
    DB-2 Primary

になる。

通常のReplica Promotionでは、DB-1をオンラインへ戻すだけでDB-1が自動的にPrimaryへ戻るわけではない。

元のRegion Aへ戻したい場合は、

    DB-2 Primary
        |
        v
    Region A
    New Replica

という形で、新しいPrimaryから元リージョンへ再びレプリケーション関係を作る。

そのReplicaが追いついた後、必要であれば再度PromoteしてRegion AをPrimaryに戻す。

つまりDRでは、

FailoverだけでなくFailbackまで設計する

必要がある。


### Enterprise PlusのAdvanced DR

現在のCloud SQL Enterprise Plusには、通常のRead Replica Promotionとは別にAdvanced DRがある。

Cross-region Read ReplicaをDR Replicaとして指定しておくことで、Replica Failoverを実行できる。

通常のPromotionとの大きな違いは、旧Primaryを新PrimaryのReplicaとして扱えること。

概念的には、

    Before

    Region A
    Primary
       |
       v
    Region B
    DR Replica


    Replica Failover


    Region A
    Old Primary
       ^
       |
    Region B
    New Primary

となる。

Region Aが復旧した後はSwitchoverによって役割を戻すこともできる。

ここは古いCloud SQL問題と現在のCloud SQLで挙動が違って見える原因になりやすい。

試験問題で単にCross-region Read ReplicaのPromoteと書かれている場合は通常のPromotionとして考え、

Enterprise Plus
DR Replica
Replica Failover
Switchover

というキーワードが出た場合はAdvanced DRを意識する。


### DR構成の考え方

Cloud SQLのDRでは、まずRTOとRPOを考える。

RTO
    どれくらいの時間でサービスを復旧させる必要があるか

RPO
    どれくらいまでのデータ損失を許容できるか

バックアップから別リージョンへDBを復元することもできるが、巨大なデータベースほど復旧に時間がかかる。

数分レベルのRTOが必要なら、あらかじめCross-region Replicaを用意しておく方が現実的。

ただしRead Replicaは非同期レプリケーションなので、リージョン障害のタイミングによっては最新トランザクションがReplicaに到達していない可能性がある。

そのため、

    HA
    -> 可用性

    Backup / PITR
    -> データ復旧

    Cross-region Replica
    -> リージョンDR

と役割を分けて考える。


### HA / Read Replica / DRまとめ

|構成|主目的|配置|Replication|自動切替|通常読み取り|
|:----|:----|:----|:----|:----|:----|
|HA Standby|可用性|同一リージョン別ゾーン|HA用|○|×|
|Read Replica|読み取り分散|同一 / 別リージョン|非同期|×|○|
|HA Read Replica|読み取り基盤のHA|別ゾーン|非同期 + HA|Replica内部では○|○|
|Cross-region Replica|DR / 地理的Read|別リージョン|非同期|通常は×|○|
|Enterprise Plus DR Replica|Advanced DR|別リージョン|非同期|Replica Failoverを実行|○|

覚え方としては、

    Standby
    -> Primaryが死んだ時の代役

    Read Replica
    -> SELECTを肩代わり

    Cross-region Replica
    -> リージョンが死んだ時の避難先

とするとわかりやすい。


### Cascading Replica

Read Replicaの下にさらにReplicaを作る構成をCascading Replicaと呼ぶ。

通常。

    Primary
     | | |
     | | +---- Replica C
     | +------ Replica B
     +-------- Replica A

Cascading。

    Primary
       |
       v
    Replica A
       |
       v
    Replica B
       |
       v
    Replica C

すべてのReplicaがPrimaryから直接レプリケーションするとPrimary側のレプリケーション負荷やネットワーク負荷が増える。

Cascading Replicaを利用すると、中間Replicaからさらに下流へレプリケーションできる。

特に多数のReplicaや複数リージョンへ展開する場合に利用される。


### BackupとPITR

Cloud SQLには主に、

    Backup
    PITR

という復旧方法がある。


#### Backup

ある時点で保存したバックアップからDBを復元する。

    Backup at 03:00
        |
        v
    Restore

というイメージ。


#### PITR

Point-in-Time Recovery。

トランザクションログを利用して特定時刻へDBを戻す。

例えば、

    10:00 正常
    10:10 正常
    10:15 DELETE FROM users
    10:20 障害発覚

なら、

    PITR -> 10:14

のように事故直前へ復元できる。


### CloneとPITR

Cloud SQLのCloneは、既存Instanceから別Instanceを作成する機能。

開発環境や検証環境を作るときに利用できる。

またPITRでは、過去時点のデータを新しいInstanceへ復元するという使い方もできる。

つまり、

Clone
    現在または指定時点を別Instanceとして複製

PITR
    特定時刻へ復旧

という関係。

「開発者に本番DBのコピーを渡したい」のか、

「誤操作前の状態へ戻したい」のか

で使い分ける。


### Cloud SQL Auth Proxy

Cloud SQL Auth Proxyを一言で言うと、

「IAMを利用してCloud SQLへの安全な接続経路を作るCloud SQL Connector」

となる。

概念的には、

    Application
        |
        v
    Cloud SQL Auth Proxy
        |
        | IAM authorization
        | TLS
        v
    Cloud SQL

という構造。

ProxyはCloud SQL Admin APIを利用して接続情報や短時間有効な証明書を取得し、Cloud SQLとの暗号化された接続を確立する。

アプリケーションから見ると、

    localhost
      |
      v
    Proxy
      |
      v
    Cloud SQL

のように通常のDB接続として扱える。


#### Cloud SQL Auth Proxyがしてくれること

主に次の2つ。

接続認可
    IAMによって、そのPrincipalがCloud SQL Instanceへ接続してよいか確認する

暗号化
    Cloud SQLとの通信をTLSで暗号化する

一方でCloud SQL Auth Proxyは、

- DBユーザーの権限管理
- SQLのSELECT / UPDATE権限管理
- Connection Pooling

を行うものではない。


### Cloud SQL Client Role

ProxyやCloud SQL Connector経由でCloud SQLへ接続するPrincipalには基本的に、

    roles/cloudsql.client

を付与する。

このロールにはCloud SQL Instanceへの接続に必要な、

    cloudsql.instances.connect

などの権限が含まれる。


### Cloud SQL IAMとDB権限は別

Cloud SQLで特に混乱しやすいポイント。

例えばサービスアカウントに、

    roles/cloudsql.client

を付与したとしても、

    SELECT users
    UPDATE orders

などのSQL権限が自動的に与えられるわけではない。

概念的には、

    Google Cloud IAM
        |
        | Cloud SQLへ接続してよいか
        v
    Database Authentication
        |
        | DBの誰としてログインするか
        v
    Database Privileges
        |
        | SELECT / UPDATE可能か
        v
    Table

という層になっている。


### Cloud SQL Admin API

Cloud SQL Admin APIはCloud SQL Instanceを管理するAPI。

Cloud SQL Auth ProxyやLanguage Connectorも、安全な接続を確立するためにこのAPIを利用する。

そのためCloud SQL Connector系を利用するときは、

    Cloud SQL Admin API
    sqladmin.googleapis.com

を有効にする必要がある。

Cloud RunからCloud SQLへ接続するときも、このAPIを利用する。


### Cloud SQL Query Insights

Query InsightsはCloud SQLのクエリパフォーマンスを分析する機能。

例えば、

- DB Loadが高いクエリ
- 遅いクエリ
- 頻繁に実行されているクエリ
- 特定アプリケーションから発行されているクエリ

などを調査できる。


#### Query Tags

複数アプリケーションが1つのCloud SQLを利用している場合、

「重いSQLはわかったが、どのアプリケーションが発行したのかわからない」

という問題が起きる。

そこでQuery Tagを付与する。

例えば、

    application=payment

    application=inventory

    application=shipping

のような情報をクエリに関連付ける。

Query InsightsではTag単位のDatabase Loadも確認できるため、

    Query
        |
        v
    Application

を追跡できる。

アプリを順番に停止して原因を探したり、ログを独自スクリプトで解析するより、Query InsightsとQuery Tagを利用する方がGoogle Cloudのマネージド機能を活用した設計になる。


### Cloud SQLのスケールアップ

Cloud SQLはマシン構成を変更してCPUやメモリを増やせる。

例えばCPU利用率が常時90%から100%なら、CPU不足が疑われる。

Cloud SQLでは既存Instanceの設定を変更できるため、

    Backup
      |
      v
    New Instance
      |
      v
    Restore

と作り直す必要はない。

gcloudではCloud SQL Enterpriseの構成によって、

    gcloud sql instances patch

にCPU / Memoryなどを指定して変更できる。

Cloud SQL Enterprise Plusでは事前定義Machine Typeを選択する。

Compute Engineの、

    gcloud compute instances update

ではない。

Cloud SQLはCompute Engine VMを内部的に利用していても、ユーザーがそのVMを直接管理するサービスではないため。

## Database Migration Service

Database Migration Service、DMSは既存データベースをGoogle CloudのマネージドDBへ移行するサービス。

オンラインMigrationでは概念的に、

    Source DB
       |
       | Initial Load
       v
    Destination DB

       +
       
    Change Data Capture
       |
       v
    Continuous Replication

という流れになる。

Initial Load中にもSourceは更新されるため、その差分をCDCで追従してCutover時の停止時間を減らす。


### Initial LoadとSource負荷

Migrationを速くするためInitial LoadのParallelismを上げると、Source DBへのCPU・IO負荷も増える。

そのため重要な本番DBでは、

    Production Primary
          |
          v
    Source Read Replica
          |
          v
    DMS

のように、可能な構成であればMigration処理をReplicaへ逃がしてProductionへの影響を抑えるという考え方がある。

ただしDMSがSource Replicaをサポートする条件はDB EngineやReplica状態によって異なるため、ここは製品別の制約を確認する必要がある。



## AlloyDB

AlloyDB for PostgreSQLはGoogle Cloudが提供するフルマネージドのPostgreSQL互換データベース。

標準的なPostgreSQLとの高い互換性を維持しながら、Google独自のストレージ・コンピュートArchitectureやColumnar Engineなどによって高いOLTP / Analytical Query性能を提供する。

Cloud SQL for PostgreSQLよりも、要求の厳しいエンタープライズワークロードを意識したサービスと考えるとわかりやすい。


### AlloyDBの構成

AlloyDBには、

    Cluster
    Instance
    Node

という概念がある。

|構成要素|役割|
|:----|:----|
|Cluster|Database、Log、MetadataなどをまとめるRegion単位の論理コンテナ|
|Primary Instance|Read / Writeを提供するInstance|
|Read Pool Instance|読み取り専用のInstance|
|Node|実際にDatabase Engineを実行するVM|

概念的には、

    AlloyDB Cluster

       Primary Instance
       +----------------+
       | Active Node    |
       | Standby Node   |
       +----------------+

       Read Pool Instance
       +----------------+
       | Node           |
       | Node           |
       | Node           |
       +----------------+

となる。


### Primary Instance

Clusterには1つのPrimary Instanceが存在し、Read / Writeを担当する。

HA Primary Instanceでは、

    Zone A
    Active Node

    Zone B
    Standby Node

という構成になる。

Active Nodeが利用できなくなればStandby Nodeへ自動Failoverする。


### Read Pool Instance

Read Poolは読み取り処理をPrimaryからオフロードするためのInstance。

    Application READ
         |
         v
    Read Pool Instance
       /     |     \
    Node   Node   Node

Read Pool内部ではAlloyDBがNode間へクエリをLoad Balanceする。

Cloud SQL Read Replicaではアプリケーション側でReplicaを選択する設計が必要になることが多いが、AlloyDB Read PoolではInstance Endpointの背後に複数Nodeを置ける点が特徴。


### Storage / Compute分離

AlloyDBではDatabase Engineを動かすCompute NodeとStorage Layerが分離されている。

従来型DBでは、

    DB Server
      |
    Local / Attached Storage

という結び付きが強い。

AlloyDBでは、

    Compute Node
         |
         v
    Distributed Storage

というArchitectureになっている。

これによってCompute側のFailoverやRead Poolの追加などを行いやすくしている。


### Cloud SQLとの違い

大まかには、

Cloud SQL
    既存PostgreSQLをマネージドで使いたい

AlloyDB
    PostgreSQL互換を保ちながら、より高いPerformance / Availabilityを求める

という関係。

どちらもPrimaryへの書き込みが中心になるため、

「世界中の複数Regionから大量Writeを水平分散する」

という要件ではSpannerの方が適している。


## Spanner

SpannerはGoogle Cloudの分散リレーショナルデータベース。

リレーショナルデータベースの、

- Schema
- SQL
- Transaction
- ACID
- Strong Consistency

を維持しながら、分散Databaseとして水平Scaleできることが最大の特徴。

従来型RDBでは、

    Bigger Server
        |
        v
    Scale Up

が中心。

Spannerでは、

    Node
    Node
    Node
    Node

のようにCompute Capacityを追加してScale Outできる。

ReadだけでなくWriteも分散できる。


### Spannerを使う理由

代表的な要件。

- 数百TBからPB規模のOperational Database
- 非常に高いRead / Write Throughput
- Global Application
- Strong Consistency
- ACID Transaction
- Region障害への高い耐性
- 水平Scale

そのため、

    Global Payment
    Inventory
    Financial Transaction
    Large-scale Game Backend

などに向いている。


### Regional Spanner

Regional構成ではデータを1Region内の複数ZoneにReplica配置する。

    Region A

    Zone A Replica
    Zone B Replica
    Zone C Replica

Region内なのでReplica間のNetwork Latencyが比較的小さく、Write Latencyを低くしやすい。


### Multi-region Spanner

Multi-region構成では複数RegionにReplicaを配置する。

    Region A
    Replica

         |
         |
         v

    Region B
    Replica

         |
         |
         v

    Region C
    Replica

Region障害にも耐えられる一方、WriteではReplica間の合意が必要になるためRegional構成よりLatencyが増える。


### LeaderとWrite Latency

SpannerのWriteで重要なのがLeader。

Read-write Transactionの調整はLeader Regionが重要な役割を持つ。

概念的には、

    Application
        |
        v
    Leader Region
        |
        +---- Replica
        |
        +---- Replica

Write時にはReplica間のConsensusが必要になる。

そのためApplicationがLeaderから遠いと、

    Application
       |
       | long network RTT
       v
    Leader

というLatencyが追加される。


### Multi-regionでWriteが遅い場合

Regional Spannerでは速いのにMulti-regionでWriteが極端に遅くなった場合、まずApplicationとLeader Regionの距離を確認する。

基本的な対策は、

    Write Workload
       |
       v
    Leader Regionの近くへ配置

すること。

つまり、

「Writeを行うComputeをLeader Regionに近付ける」

のが重要。


### Stale ReadではWriteは速くならない

SpannerではStalenessを指定したReadができる。

例えば、

    15 seconds stale

のReadなら、最新状態でなくてもよいので近くのReplicaから効率よくReadできる場合がある。

これはRead Optimization。

    Stale Read
    -> READのLatency改善

であって、

    WRITEのLatency改善

ではない。

Multi-region Writeが遅い問題にStale Readを設定しても根本解決にはならない。


### Read-only Replicaを増やしてもWrite Quorumは速くならない

Read-only ReplicaはRead Scaleや地理的Read Latency改善には利用できる。

しかしRead-only ReplicaはWrite Quorumへ参加しない。

したがって、

    Add Read-only Replica
        |
        v
    Write Faster

とはならない。


### Regional / Multi-region

|構成|特徴|
|:----|:----|
|Regional|低いWrite Latency、Region内HA|
|Dual-region|2Regionへ分散|
|Multi-region|Global Availability、Region障害耐性|

現在のSpannerではEditionとInstance Configurationによって利用できる構成やSLAが異なるため、

「Spanner Multi-regionは必ず99.999%」

という単純な暗記より、

「Enterprise Plusなど対応構成では99.999%クラスのAvailabilityを提供する」

と理解した方が現在のサービス体系に近い。


### Spannerと小規模スタート

サービス開始時は1か国の小規模Pilotでも、

    将来Global
    数百万人
    24 / 365
    Maintenance Downtimeを最小化

という明確な要件があるなら、最終Architectureを見越してSpannerを選択する場合がある。

ただし最初からMulti-regionにするとCostやWrite Latencyは増えるため、

    Current Requirement
    Future Requirement
    Migration Cost

のバランスで判断する。


## Firestore

FirestoreはフルマネージドのNoSQL Document Database。

データをCollectionとDocumentとして管理する。

概念的には、

    users
      |
      +---- user001
      |       name
      |       email
      |
      +---- user002
              name
              email

という構造。

モバイル / Web Applicationとの統合が強く、

- Realtime Update
- Offline Cache
- Automatic Scaling
- Client SDK

などが特徴。


### Realtime Update

ClientがDocumentをListenしておくと、Firestoreのデータ変更をRealtimeに受け取ることができる。

    Client A
       |
       v
    Firestore
       |
       v
    Client B

チャットや共同編集などに向いている。


### Offline Support

Firestore Client SDKは利用中のデータをLocal Cacheに保存できる。

Networkが切断されてもCacheへRead / Writeし、再接続後にFirestoreと同期できる。

Mobile Applicationと相性がよい理由の一つ。


### Serverless

FirestoreではCloud SQLのように、

    4 vCPU
    16 GB RAM

などのDatabase InstanceをユーザーがProvisioningする必要がない。

Workloadに応じてService側がScaleする。

ただし、

「Serverlessだから無料」

という意味ではない。

Document Read / Write / Delete、Storage、Networkなどに応じて課金される。


### Firestore Native Mode / Datastore Mode

FirestoreにはNative ModeとDatastore Modeが存在する。

現在Googleは新規ApplicationにはNative Modeを推奨している。

Datastore Modeは主に既存Cloud Datastore / Datastore API依存Application向け。



#### Native Mode

新規Web / Mobile / Server Application向け。

特徴。

- Collection / Document Model
- Realtime Update
- Offline Persistence
- Firestore Client SDK
- Firebaseとの統合

現在はServer ApplicationについてもNative Modeが推奨されているため、

「Native = Mobileだけ」

と覚えない方がよい。


#### Datastore Mode

既存のDatastore APIを利用するApplication向け。

特徴。

- Entity / Kind Model
- Datastore API互換
- Datastore Client Library
- Realtime Listenerなし
- Firestore Native Client APIは利用しない

つまり、

Native Mode
    新しいFirestore Application

Datastore Mode
    Datastore Compatibility

と考えるとわかりやすい。


## Bigtable

Cloud Bigtableは大規模な低Latency処理に特化したNoSQL Database。

Wide-column Modelを採用し、非常に大量のRowを扱える。

代表的な用途。

- Time Series
- IoT
- Monitoring Data
- Financial Market Data
- Recommendation Feature
- User Activity
- Marketing Data
- Large-scale Key-Value Access

Spannerのような一般的なRelational Databaseではなく、

    Row Key
       |
       v
    Column Family
       |
       v
    Columns / Versions

という構造を中心に設計する。


### Time Seriesとの相性

BigtableではCellにTimestampを持ち、複数Versionの値を保持できる。

そのため、

    Device
       |
       +---- 10:00 value=10
       +---- 10:01 value=12
       +---- 10:02 value=15

のようなTime Series Dataと相性がよい。


### BigtableのInstance / Cluster / Node

Bigtableでは、

    Instance
       |
       +---- Cluster A
       |       |
       |       +---- Node
       |       +---- Node
       |
       +---- Cluster B
               |
               +---- Node
               +---- Node

という構造になる。

Instance
    Bigtableの論理的なコンテナ

Cluster
    ClientがアクセスするCompute / Storage Replica

Node
    Data処理を担当するCompute Unit


### Bigtable Schema Design

BigtableではSchema、特にRow Key DesignがPerformanceに大きく影響する。

例えば時系列データで、

    202608101200
    202608101201
    202608101202

のように単調増加するTimestampをRow Keyの先頭にすると、Writeが特定Key Rangeへ集中する可能性がある。

これをHotspotと呼ぶ。

BigtableではDataをKey Range単位で分散するため、

    device001#timestamp
    device002#timestamp
    device003#timestamp

など、Writeが適切に分散するRow Key Designが重要。


### Bigtable Multi Cluster

Single Cluster。

    Instance
       |
       v
    Cluster A

Cluster障害時にApplicationを継続できない可能性がある。

Multi Cluster。

    Instance
       |
       +---- Cluster A
       |
       +---- Cluster B

各ClusterにTable DataがReplicationされる。

Global 24 / 365 ApplicationやMission Critical WorkloadではMulti Cluster構成を検討する。


### Bigtable ReplicationとConsistency

Bigtableで複数Clusterを使用すると、DataはCluster間でReplicationされる。

    Cluster A
       |
       | replication
       v
    Cluster B

このReplicationは基本的にEventually Consistent。

例えばCluster AへWriteした直後、

    Cluster A
    value = NEW

    Cluster B
    value = OLD

という短い時間が存在する可能性がある。

Replication完了後、

    Cluster A
    value = NEW

    Cluster B
    value = NEW

となる。

ただし同じClusterにRead / WriteをRoutingすればRead-your-writes Consistencyを維持できる。


### Bigtable App Profile

Bigtableでは1つのInstanceを複数Applicationから利用できる。

例えば、

    API Server
    Batch
    Analyst
    Admin Tool

が同じBigtableへ接続する場合。

それぞれ、

    Latency
    Availability
    Consistency
    Priority

などの要件が異なる。

そこでApplication単位の接続PolicyをApp Profileとして定義する。


#### App Profileとは

App Profileは、

「ApplicationのRequestをどのClusterへRoutingするか」

を定義する設定。


#### Single-Cluster Routing

    Application
        |
        v
    Cluster A

特定Clusterへ固定Routingする。

メリット。

- Read-your-writes Consistencyを保ちやすい
- Latency特性を予測しやすい
- Clusterを明示的に選択できる

デメリット。

- Cluster Aが障害になると自動的にCluster Bへ切り替わらない

Consistencyを重視するWorkloadやBatchを特定Clusterに隔離したい場合に利用する。


#### Multi-Cluster Routing

    Application
        |
        +---- Cluster A
        |
        +---- Cluster B

利用可能なClusterへBigtableがRoutingする。

Applicationから近いClusterへRequestを送ることでLatencyを下げることができ、Cluster障害時には他のClusterへRoutingできる。

そのため高可用性を重視するApplicationで利用される。


#### Routing比較

|Routing|Availability|Consistency|用途|
|:----|:----|:----|:----|
|Single Cluster|Cluster障害の影響を受ける|同一ClusterでRead-your-writes|Consistency重視|
|Multi Cluster|高い|Cluster間はEventually Consistent|HA重視|


## Oracle on Google Cloud

Oracle DatabaseをGoogle Cloudへ移行するとき、Oracle Database自体をそのまま利用し続ける必要がある場合にはBare Metal Solutionが選択肢になる。

Bare Metal SolutionはGoogle Cloud Regionの近くにある専用Bare Metal環境をGoogle Cloudと接続して利用するサービス。

    Google Cloud VPC
          |
          | Interconnect
          v
    Bare Metal Solution
          |
          v
    Oracle Database

という構成になる。

Oracle RACやData Guardなど既存Oracle Architectureを維持する必要があるEnterprise Workloadで利用される。


### Oracle Backup

OracleではRMAN、

    Recovery Manager

が標準的なBackup / Recovery Tool。

RMANでは、

- Backup
- Restore
- Point-in-Time Recovery
- Database Clone

などを行える。

### Oracle Backup Architecture

Oracleの巨大Databaseでは、

    Fast Restore
    Long Term Retention

を同じStorageだけで満たそうとするとCostが高くなりやすい。

そのため、

    Oracle DB
       |
       v
    Local / Staging Backup
       |
       | old backups
       v
    Cloud Storage / Backup Vault

という階層化を行う。


### Short-term Backup

最近のBackupは高速にRestoreできるStorageへ保持する。

目的。

    Low RTO

障害時に数十TBのDatabaseをArchive Storageから全部戻すと、RTOを満たせない可能性がある。

そのため直近Backupを高速に利用できる場所へ保持する。


### Long-term Backup

古いBackupは、

    Cloud Storage
    Backup Vault
    OnVault

などへ移して長期保存する。

目的。

    Low Cost
    Long-term Retention

