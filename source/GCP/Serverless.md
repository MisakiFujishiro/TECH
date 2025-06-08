# サーバーレスアーキテクチャ
サーバーレスアーキテクチャを利用することで、インフラの管理を最小限に抑え、開発者がAPP開発に集中することができる。
GCPでは、`Cloud Functions`・`Cloud Run`・`Cloud Run Functions`といったサーバーレスサービスが提供されている。

## Cloud Functions
`CloudFunctions`は、イベント駆動型の`関数実行環境`である。  
開発者は、インフラ管理は一切不要であり、コードをデプロイするだけで、処理を実行することができ、AWSのLambdaのようなサービスである。

![](../img/GCP/Serverless/CloudFunction_illustration.png)   
[Learn Cloud Functions in a snap!](https://cloud.google.com/blog/topics/developers-practitioners/learn-cloud-functions-snap?utm_source=ext&utm_medium=partner&utm_campaign=CDR_pve_gcp_gcpsketchnote_&utm_content=-&hl=en)



## Cloud Run
`Cloud Run`とは、フルマネージドなコンテナ実行環境である。
任意の言語やライブラリを含むアプリケーションを、コンテナ化してデプロイすると、自動で実行環境が構築される。`Cloud Run`はAWS Lambdaの気軽さとFargateのカスタマイズ性を併せ持ちつイメージ。また特徴は以下のようなものがある。

- オートスケーリング＆ゼロスケール  
リクエストの有無に応じてインスタンス数が自動調整され、使わないときにはスケールダウンして課金が停止 。

- Dockerイメージ管理：Artifact Registry連携  
リポジトリにDockerイメージを格納し、Cloud Runで実行。CLIやソースデプロイも可能。


CloudRunには、`Cloud Run Services`と`Cloud Run Jobs`の2つの機能が存在。
![](../img/GCP/Serverless/CloudRun_services_jobs.png)
[Cloud Run ジョブ ことはじめ](https://zenn.dev/google_cloud_jp/articles/cloudrun-jobs-basic)


特徴をまとめると下表になるが詳細は、次節で解説。
|種類|特徴|主な利用ケース|
|:----|:----|:----|
|Cloud Run Services|HTTP/Gatewayを通じたリクエストに応答する<br>常駐型のコンテナ<br>HTTPS・HTTP/2・gRPC・WebSocket 対応<br>自動TLS、トラフィック分割可|Web API、マイクロサービス、イベント駆動（Pub/Sub/Eventarc）|
|Cloud Run Jobs|一時的タスクを実行し終えると停止する<br>バッチ処理型<br>最大10,000タスク並列可<br>環境変数でタスク制御<br>最大7日間実行可能|データ処理バッチ、画像リサイズ、DBマイグレーション、定期ジョブ|


### Cloud Run Services
`Cloud Run Services`は、HTTPエンドポイントに対するリクエストなど、外部からのリクエストをトリガーとして実行される特徴がある。


### Cloud Run Jobs
`Cloud Run Jobs`は、HTTPサーバーを立てず、一度実行して終了するバッチ的な処理に設計されたリソース。手動やスケジュールワークフローの一部など、任意のタイミングで実行される特徴がある。

Cloud Run Jobsは下図のように`Job`、`Execution`、`Task`から構成される。
![](../img/GCP/Serverless/CloudRun_jobs.png)
[Cloud Run jobsを徹底解説！](https://blog.g-gen.co.jp/entry/cloud-run-jobs-explained)

これら3つの特徴をまとめると以下。

|要素|説明|主な構成・制御対象|成功・失敗の定義|
|:----|:----|:----|:----|
|Job|コンテナイメージやTask数、並列度、CPU/メモリ、環境変数などを定義・保持するルートリソース|• 使用するコンテナイメージ<br>• --tasks, --parallelism<br>• CPU/メモリ制限<br>• 環境変数・秘密情報|—|
|Execution|Job の実行単位。1回の gcloud run jobs execute や Workflows等の起動により生成される|• 登録された Job 設定に基づいて Task を生成・実行<br>• リトライポリシーやタイムアウトの制御|全 Task が成功 →成功<br>どれか Task がリトライ上限超 →失敗|
|Task|Execution によって起動された個別のコンテナインスタンス（1 Task = 1 コンテナ）|• 環境変数 CLOUD_RUN_TASK_INDEX, CLOUD_RUN_TASK_COUNT による役割決定<br>• 一定時間内（最大168時間）で処理|コンテナが正常に終了、もしくはタイムアウトや再試行制限到達でタスク完了。未終了であれば失敗とみなされる|



## Cloud Run Functions
2024年8月に、CloudRunとCloud Functionsが統合され、Cloud Run Functionsのサービスが開始された。  
[Cloud Functions is now Cloud Run functions — event-driven programming in one unified serverless platform](https://cloud.google.com/blog/products/serverless/google-cloud-functions-is-now-cloud-run-functions?hl=en)

`Cloud Run Functions`は、Cloud Functionsの開発の簡易性とCloud Runの実行環境の柔軟性を兼ね備えたサービス。
ユーザーはソースコードをデプロイするだけで、Google Cloudが自動的にコンテナイメージをビルドし、Cloud Run上で実行されます。

サービスの比較表は以下

|特徴|Cloud Functions（第1世代）|Cloud Run Functions|Cloud Run|
|:----|:----|:----|:----|
|デプロイ方法|ソースコード|ソースコード|コンテナイメージ|
|対応言語|特定の言語に限定|複数の言語に対応|任意の言語に対応|
|実行時間制限|最大9分|最大60分|最大60分|
|スケーリング|自動スケーリング|自動スケーリング|自動スケーリング|
|イベントトリガー|HTTP、Pub/Sub、Cloud Storage|HTTP、Pub/Sub、Eventarcなど|HTTP、Pub/Subなど|
|VPC接続|制限あり|可能|可能|
|トラフィック分割|不可|可能|可能|
|最小インスタンス設定|不可|可能|可能|
