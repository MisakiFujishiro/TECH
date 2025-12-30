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