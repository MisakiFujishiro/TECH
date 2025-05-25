# CloudRun
Cloud Runはフルマネージドの`コンテナ実行環境`でありながら、自動スケーリング・HTTPリクエスト処理を行なってくれる。
実行環境としては、LambdaとFargateのいいとこ取りをしつつ、実行方法としてはAPIGW+Lambdaのような動きをしてくれる。

![](../img/GCP/CloudRun/CloudRun_gaiyou.png)
[Google Cloud Fundamentals: Core Infrastructure 日本語版](https://www.coursera.org/learn/gcp-fundamentals-jp/lecture/40yXj/google-cloud-netutowaku)

実行のイベントトリガーにはHTTPリクエストだけでなく様々なものが準備されている。

|トリガー種別|サービス名|起動条件|ユースケース例|
|:----|:----|:----|:----|
|✅ リクエスト駆動|HTTP リクエスト|任意の HTTP クライアントからのアクセス|REST API、Webhook受信など|
|✅ メッセージ駆動|Pub/Sub|Pub/Sub トピックにメッセージが publish|非同期バッチ、イベント連携|
|✅ スケジュール駆動|Cloud Scheduler|cron形式で定期実行（例：毎日9時）|バッチ実行、定期レポート生成|
|✅ イベント駆動|Eventarc|GCS、Firebase、BigQuery などのイベント|ファイルアップロード検知、監査ログ処理など|
|✅ CI/CD駆動|Cloud Build|コード更新に伴うビルド後に自動デプロイ|継続的デプロイ構成（CD）|
