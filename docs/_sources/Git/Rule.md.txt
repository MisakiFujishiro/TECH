# git運営ルール
## 戦略
gitlab-strategyをベースとしたブランチ戦略を採用する。

|ブランチ名|特徴|
|:----|:----|
|master=PRD|本番環境にデプロイされるコードを保持するブランチ。STGでのリハーサルおよびST試験後のものがマージされる|
|staging=STG|STG環境にデプロイされるコードを保持するブランチ。リハーサル時にマージされ、STG環境でST試験が行われる|
|release branch|リリース前にfeatureブランチを集約するブランチ。集約後にSTGにマージされる。STGへマージする度にCICD Pipelineが実行されてしまうため、release branchを利用する。<br>feature同士でコンフリクトがある際は、releaseブランチ上で解消する。|
|testing=TST|TST環境が存在する場合は、TSTにデプロイされるコードを保持するブランチ。性能試験に必要なブランチが集約される。<br>TST環境が存在しない場合は、N月リリースの状態を保持するブランチ。|
|development=DEV|DEV環境にデプロイされるコードを保持するブランチ。featureを最初にマージし、DEV環境で開発と改善を行う|
|feature branches|特定の機能やバグ修正のために作成される短期的なブランチ。|
|devmerge branches|featureブランチをdevにマージする際にコンフリクトが発生した際に解消するためのブランチ|
|tstmerge branches|featureブランチをtstにマージする際にコンフリクトが発生した際に解消するためのブランチ|

以下の条件では、コンフリクトが発生するので、masterはリリース毎にdevelopmentとtestingにマージする
- developmentやtestingにマージする際に、devmergeやtstmergeでコンフリクトを解消
- developmentやtestingを経由せずに、releaseブランチを利用してコンフリクトを解消

上記は同じコンフリクトを解消しているが、異なる場所でコンフリクトを解消している。
これらを蓄積させないためにN月リリースのタイミングで、masterをdevelopmentとtestingにマージして、developmentやtesting上でコンフリクトを解消する。（改修自体は同じなので変更はないはず）

また、リリースされなかったfeatureに対してもmasterをマージしてN月リリースから切り出されたのと同じ状態にする。

## ルール
リリース後にはTagを打つ。
