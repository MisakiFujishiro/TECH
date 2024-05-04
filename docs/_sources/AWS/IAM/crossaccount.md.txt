# クロスアカウント
別AWSアカウントのリソースの操作をすることをクロスアカウントと呼ぶ。
公式ドキュメントの[IAM でのクロスアカウントのリソースへのアクセス](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies-cross-account-resource-access.html)参考。

クロスアカウントには、以下の2つの方法がある
- リソースベースポリシーを利用して、別アカウントのIAMプリンシパルに権限を追加してあげる方法
- Roleを利用して、別アカウントのRoleが自アカウントのRoleにスイッチできるようにしてあげる方法

## 基本的な考え方
基本的にクロスアカウントでは、どちらの方法を利用するにしても、以下の2つを準備すればよい
- アカウントAのPrincipalにアカウントBのIAMプリンシパルの情報を記述
- アカウントAのActionに許可したいアクションのみを記述
- アカウントBのIAMプリンシパルの許可ポリシーにアカウントAのリソースに対する操作を記述



## リソースベースポリシーを利用したクロスアカウント
リソースベースポリシーを利用したクロスアカウントの設定イメージと設定方法について見ていく。

注意点としては自アカウント内で許可する際は、リソースベースポリシーか許可ポリシーどちらかに許可が記載されていれば十分だったが、クロスアカウントではリソースベースと許可ポリシー両方で記述が必要である。
また、認可に対してもリソースベースと許可ポリシーの共通部分だけが許可される動きとなる。


### 設定イメージ
クロスアカウントの基本的な考え方に沿って、以下を実施
- アカウントAのリソースベースポリシーのPrincipalにアカウントBのIAMプリンシパルを記述
- アカウントAのリソースベースポリシーのActionに許可したいアクションのみを記述
- アカウントBのIAMプリンシパルの許可ポリシーにアカウントAのリソースに対する操作権限を記述

![](../../img/AWS/iam/iam_resourcebase_cross_account.png)


### 設定方法
#### アカウントAのリソースベースポリシーのPrincipal/Action設定
以下のようにアカウントAでは、PrincipalとしてアカウントBを指定して、操作してよい対象と操作してよいAction両方を記述する
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {"AWS": "arn:aws:iam::AccountB:root"},
            "Action": "[許可したいアクション]",
            "Resource": "[許可したい対象リソース(リソース自身)]"
            }
        ]
}
```


#### アカウントBの許可ポリシーにAction設定
以下のようにアカウントBでは操作したいリソースとアクションを記述する。
```json
{
     "Version": "2012-10-17",
     "Statement": [
         {
             "Sid": "",
             "Effect": "Allow",
             "Action": "[実行したいアクション]",
             "Resource": [
                 "arn:aws:iam::AccountA:[操作したい対象リソース]"
             ]
         }
     ]
 }
```


## Roleを利用したクロスアカウント（スイッチロール）
スイッチロールを利用したクロスアカウントの設定イメージと設定方法について見ていく。

注意点はスイッチロールの方法を採用すると、別アカウントが元々利用していたRoleは捨てて、自アカウントのRoleに完全に切り替わっている点。


### 設定イメージ
クロスアカウントの基本的な考え方に沿って、以下を実施
- アカウントAのRoleの信頼ポリシーにアカウントBのIAMプリンシパルを記述
- アカウントAのRoleの許可ポリシーに許可したいアクションのみを記述
- アカウントBのIAMプリンシパルの許可ポリシーにアカウントAのRoleへのAssumeRoleを記述

![](../../img/AWS/iam/iam_switchrole_cross_account.png)




### 設定方法
#### アカウントAのRoleの信頼ポリシー設定
以下のようにアカウントAではスイッチロールを許可するために、信頼ポリシーにrole-YYYをprincipalに記述する。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::AccountB:role/role-YYY"
                ]
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```
#### アカウントAのRoleの許可ポリシー設定
以下のようにアカウントAではスイッチロール後に操作してよいリソースやアクションを記述する。
```json
{
     "Version": "2012-10-17",
     "Statement": [
         {
             "Sid": "",
             "Effect": "Allow",
             "Action": "[別アカウントに許可したいアクション]",
             "Resource": [
                 "[別アカウントに許可したいリソース]"
             ]
         }
     ]
 }
```

#### アカウントBのIAMプリンシパルの許可ポリシー設定
以下のようにアカウントBではスイッチロール先のRole-XXXに対してのassumeRoleの許可ポリシーを記述する。
```json
{
     "Version": "2012-10-17",
     "Statement": [
         {
             "Sid": "",
             "Effect": "Allow",
             "Action": "sts:AssumeRole",
             "Resource": [
                 "arn:aws:iam::AccountA:role/role-XXX"
             ]
         }
     ]
 }
```

## 混乱した代理問題
サードパーティに対して、自分のAWSのIAM Roleを伝えることで、サードパーティのAWSユーザーやRoleがスイッチロールをして、自分のAWSを利用することが可能になる。

自分のAWSに対してサードパーティのAWSがアクセスすることを考える。一般的なクロスアカウント設定を行うと以下の設定をする。
- 外部のAWSのIDを受領
- IAM Roleを作成し、許可ポリシーを付与
- 信頼ポリシーにPrincipalに外部のAWS IDを設定

ただしこれには、混乱した代理問題という問題が発生する。

このままだと、サードパーティを利用しようとする別の第三者が、自分のIAM RoleのARNを予測することで、自分のIAM Role権限を利用してしまうという問題。
これに対しては、サードパーティに対してIAM Roleを登録する際に、ランダム文字列などを発行して自分のAWSの信頼ポリシーの中のConditionにその文字列を追加することによって、対応する。

![](../../img/AWS/iam/iam_cossaccount_confuseddeputyproblem.png)
公式ドキュメント[混乱する代理問題](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/confused-deputy.html)