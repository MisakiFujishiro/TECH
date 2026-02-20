# その他
## Service Directory
```
Service Directory は「サービス名 → 実体（URL / IP）を引くための台帳」
```

Service Directoryは、GCP / マルチクラウド / オンプレミスなど 実行環境を問わず、
サービス情報（エンドポイント・メタデータ）を一元管理する サービスレジストリ である。

VM や Cloud Run などが Service Directory に登録されることで、
別のサービスは Service Directory にクエリするだけで 対象サービスの参照先情報を動的に取得できる。

例えば、

- 登場人物
    - 呼び出し元：frontend（Cloud Run）
    - 呼び出し先：inventory（Cloud Run / GCE / 将来増える可能性あり）
    - 管理台帳：Service Directory
- 名前空間の作成
```
namespace: ecommerce-prod
```
- サービス登録
    - 新しい Cloud Run サービスや VM をデプロイしたタイミングで、Service Directory に自分自身を登録する。
    - URL や IP が変わっても 登録情報を更新するだけ呼び出し元は影響を受けない
```
service: inventory
endpoint:
    address: https://inventory-xxxxx-uc.a.run.app
    metadata:
        env: prod
        region: asia-northeast1
        version: v1
```
- 実行時にクエリして通信
    - 呼び出し元（frontend）は、`「ecommerce-prod の inventory サービスを教えて」`と問い合わせ

## Cloud Trace
Cloud Trace は Google Cloud が提供する分散トレースのバックエンドであり、
OpenTelemetry で生成されたトレースデータを受信・保存・可視化できる。
OpenTelemetry は計装の標準であり、Cloud Trace はその実行基盤である。

## workforce identity federation
workforce identity federation は、Google Cloud Console に人間のユーザー（従業員）が外部 IdP のアカウントでシングルサインオンできるようにする機能です。
これを用いると、ユーザーがコンソールにログインした際に、外部 IdP 側のユーザー情報（名前や写真など）を Google Cloud に渡すことができ、属性マッピングによりアカウントを紐づけて表示情報をパーソナライズできます。

## Cloud Profiler
Cloud Profiler は、Google Cloud が提供するプロファイリングツールで、Go を含む複数の言語をサポートします。
CPU時間やメモリ使用量などを継続的にサンプリングし、フレームグラフ（Flame Graph）を提供してどの関数がどれだけリソースを使っているか可視化します。

## Cloud Asset Inventory
Cloud Asset Inventoryは、Google Cloudのリソースをまたがって、検索フィルタリストが可能。
- searchAllResources（または gcloud asset search-all-resources）で取得する
- query expression（検索クエリ）を渡せるので、ユーザーのフィルタ式で絞り込み可能


## App Engine
App Engine は Google Cloud が提供するフルマネージドなサーバーレス PaaS であり、
インフラ運用を極力意識せずにアプリケーションの開発・デプロイ・スケーリングを行える。

### AppEngineの特徴
App Engine は、Google Cloud が提供するフルマネージドな PaaS であり、
アプリケーションを「どのインフラで動かすか」をほとんど意識せずに
運用できることを目的としたサービスである。

App Engine の設計思想は一貫しており、
開発者や運用者が考える対象は「VM・コンテナ・ロードバランサー」ではなく、
あくまで「アプリケーションそのもの」である。
スケーリング、インスタンス管理、障害時の再起動といった要素は、
プラットフォーム側に完全に委譲される。

Cloud Run も同様にフルマネージドではあるが、
Cloud Run が「コンテナをどう実行するか」を中心に据えているのに対し、
App Engine は「アプリケーションをどう運用するか」を中心に設計されている。
そのため App Engine では、アプリケーションは
サービス・バージョンといった論理的な単位で管理される。

### App Engineのタイプ
- Standard 環境は、Cloud Runに近いが、より制限が強いイメージ
- Flexible 環境は、ほぼGCEだけど、App Engineっぽく扱えるイメージ

|項目|Standard|Flexible|
|:----|:----|:----|
|実行基盤|Google独自のサンドボックス|Compute Engine VM|
|コンテナ|直接は見えない|Dockerコンテナ|
|スケーリング|リクエスト単位|VM単位|
|起動速度|非常に速い|比較的遅い|
|SSH|不可|可能|
|カスタマイズ|制限あり|ほぼ自由|
|運用負荷|極小|ややある|


### App Engine のリリースモデル
App Engine の最大の特徴の一つは、環境ではなく「バージョン」を軸にリリースを管理する点にある。

一般的な構成は以下のようになる。
```
Application  
└─ Service  
　　├─ Version v1（現在の本番）  
　　└─ Version v2（新バージョン）  
```

以下の設計思想が、App Engine 特有のリリース体験を生んでいる。
- 同一サービス内に 複数のバージョンを同時に存在させられる
- どのバージョンに どれだけトラフィックを流すか を後から制御できる

