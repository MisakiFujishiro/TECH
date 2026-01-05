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