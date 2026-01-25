# CICD
GCP には、AWS の CodePipeline のような「これがパイプラインです」と明示された単一サービスは存在しない。

その代わりに `GCP では、複数のサービスが役割分担し、イベントと API 呼び出しによって“流れ”が生まれる`
という思想で CI/CD が構成される。
つまり「pipeline という箱は無い。pipeline は振る舞いとして現れる」というのが GCP CI/CD の前提である。

GCPにおける代表的なCI/CD構成
```
Cloud Build
    └─ ビルド・テスト・署名
        ↓
Artifact Registry
    └─ 成果物（コンテナ）を保管
        ↓
Cloud Deploy
    └─ 環境（EX：Dev / Test / Prod） への配布と昇格管理
        ↓
GKE
    └─ 実行
        ↑
Binary Authorization
    └─ 実行してよいかの最終チェック
```

AWSのCodePipelineのように、Pipelineサービスがあるわけではなく、複数のサービスを“役割分担”で組み合わせて結果としてパイプラインになる。  
それぞれのサービスの役割は以下の通りで、各サービスの詳細を説明した後に具体的なpipelineの流れのイメージを説明する。

|レイヤ|役割|
|:----|:----|
|実行|Cloud Build|
|状態管理|Cloud Deploy|
|成果物保管|Artifact Registry|
|自動化の接着剤|Pub/Sub + Trigger|
|最終ガード|Binary Authorization|



## Cloud Build
Cloud Buildの役割は、「CI を担う実行基盤」
- ソースコードからコンテナをビルド
- テスト・スキャン
- Binary Authorization 用の attestation を生成可能
- Cloud Deploy を操作するコマンドを実行することはあるが、昇格判断は行わない

### CloudStorageを利用したビルドキャッシュ
Cloud Build では、Cloud Storage をビルドキャッシュとして利用できる。
これは、ビルドステップで使用するディレクトリを GCS に保存・復元する仕組みである。
Docker ビルドに限定されるレイヤキャッシュとは異なり、ビルドツール非依存で利用できる。
Node.js、Java、Go など、各言語の依存関係キャッシュをそのまま再利用可能である。
既存のビルド手順を大きく変更せずに導入できる点が特徴である。

## ArtifactRegistry
ArtifactRegistryの役割は、「保管場所」
- コンテナイメージを保存
- Cloud Build が push
- Cloud Deploy / GKE が pull


## Cloud Deploy
Cloud Deployの役割は、「CDを担う：状態管理・昇格ルール」
- Target（例: Dev / Test / Prod）を順序付きで定義する
- 各リリースがどの環境まで到達したかを管理する
- promote（昇格）という明示的な操作で次の環境へ進める
- CI（ビルド・テスト）とは分離され、CD に専念する


## Binary Authorization
Binary Authorizationの役割は、「強制セキュリティゲート」
- GKE が Pod を起動する直前にチェック
- 「このイメージは信頼できるか？」
- 署名（attestation）がないと 実行不可

## GCPにおけるCICDの流れ
GCPにおける上記のサービスを組み合わせた際のCICDの流れのイメージ。
```
Developer
   ↓
Cloud Build
   ↓
Artifact Registry
   ↓
Cloud Deploy（Dev）
   ↓
GKE ← Binary Authorization
   ↓
Cloud Deploy（状態）
   ↓
Pub/Sub（clouddeploy-operations）
   ↓
Cloud Build Trigger
   ↓
Cloud Deploy（promote）

※ clouddeploy-operations は Cloud Deploy の状態変化を通知するための
   Pub/Sub トピックであり、promote を実行するわけではない。
   promote の実行には「実行主体」が必要であり、
   CI/CD の文脈では Cloud Build がその役割を担うことが多い。
```

それぞれの役割のイメージ

|Step|フェーズ|主体|何が起きるか|関連サービス|
|:----|:----|:----|:----|:----|
|①|開発|開発者|ソースコードを Git リポジトリに push|Git|
|②|CI|Cloud Build|コンテナをビルドし、テスト・スキャンを実行|Cloud Build|
| | | |成果物を Artifact Registry に push|Artifact Registry|
| | | |必要に応じて attestation（署名）を生成|Binary Authorization|
|③|CD開始|Cloud Deploy|リリースを作成し、進行状況を管理|Cloud Deploy|
| | | |Dev / Test / Prod の順序を定義| |
| | | |まず Dev 環境へロールアウト| |
|④|実行|Google Kubernetes Engine|Pod を起動|GKE|
| | | |起動直前に署名を検証|Binary Authorization|
| | | |正しければ実行、なければ拒否| |
|⑤|自動化|Cloud Deploy|成功・失敗をイベントとして通知|Pub/Sub|
| | | |clouddeploy-operations にイベント発行|clouddeploy-operations|
| | |Cloud Build|イベントをトリガーに次の処理を実行|Cloud Build Trigger|
| | | |promote などの追加操作|Cloud Deploy API|

### clouddeploy-operationsトピック
clouddeploy-operations は、  
Cloud Deploy が発生させる「状態変化イベント」を通知するための Pub/Sub トピック  
である。

Cloud Deploy は以下のような事実を内部で把握している。
- Dev 環境へのロールアウトが成功した
- ロールアウトが失敗した
- promote が完了した

しかし Cloud Deploy 自身は、
- 「成功したら次を自動で実行する」
- 「条件分岐して外部処理を起動する」

といった 実行責務を持たない。

その代わりに、「何が起きたか」だけをイベントとして外部に通知するための出口として
clouddeploy-operations という Pub/Sub トピックが用意されている。

### cloud-buildsトピック
cloud-buildsという、Cloud Build 自身の実行状態を通知するための Pub/Sub トピックも存在する。

Cloud Build が
- ビルド開始
- 成功
- 失敗

といった 自分自身のイベントについてはこちらのPub/Subトピックを参照する。