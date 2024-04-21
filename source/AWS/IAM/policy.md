# Policyについて
詳細は[ユーザーガイド](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies.html)を参照

## Policyの構成要素
ポリシーは以下の構成要素で成り立っている
```
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "①",
            "Action": "②",
            "Resource": "③",
            "Principal": "④",
            "Condition⑤": {
                "{Operator⑥}": {
                    "Key⑦": "Value⑧"
                }
            }
        }
    ]
}
```

|No|項目|内容|
|:----|:----|:----|
|①|Effect|ポリシーの効果（許可または拒否）を定義する。</br>これにより、ポリシーが許可する行動か、それとも拒否する行動かが決まる。|
|②|Action|実行可能なアクションを定義する。</br>これはIAMポリシーで許可または拒否したいAWSサービスの具体的な操作を指す。|
|③|Resource|アクションが適用されるリソースを定義する。</br>これにより、ポリシーが影響を及ぼす特定のAWSリソース（例：特定のS3バケットやEC2インスタンス）が指定される。|
|④|Principal|ポリシーにより認可される対象を定義する。</br>どのAWSアカウント、ユーザー、ロール、またはサービスがポリシーによって許可されるかを指定される。</br>リソースベースのポリシーで使用されることが多い。Roleの場合は信頼ポリシーで対象を指定するので通常は必要ない。|
|⑤|Condition|特定の条件を満たす場合にのみ認可が有効になるように定義する。</br>条件はポリシーの適用をより詳細に制御するために使用されます。|
|⑥|Operator|比較演算子。</br>これは、条件が真かどうかを評価するために使用される演算子（例：StringEquals, NumericLessThan など）。|
|⑦|Key|リクエストのどの属性をチェックするかを定義する。</br>これは、条件の評価の基準となるリクエストの特定の部分（例：aws:sourceIp, aws:username など）を指す。|
|⑧|Value|比較の基準値を定義する。</br>Key に指定された属性がこの値とどのように比較されるかにより、条件の結果が決定される。|


## Policyの種類
認可の視点で考えるとアイデンティティベースとリソースベースの２種類がある。
- リソースベースポリシーは各AWSリソース側で指定するポリシー
- アイデンティティベースポリシーはIAM側で設定するポリシー

どちらかで許可のポリシーが記述されていれば、アクセスの許可がされる。

### リソースベースポリシー
対象のAWSリソースでアクセスの権限付与を行う。

対象のリソースに対して個別にルール設定を行うことができるためS3やAPIGWなどで利用することがおおい。
また、クロスアカウントを設定する際には、アイデンティティベースとリソースベース両方で許可をする必要があるためその際にも利用される。

リソースベースポリシーを設定する際には誰からのアクセスを許可するかを定義するため、principalの定義を行い、どのどのAWSアカウントやユーザーがそのポリシーに基づいてリソースにアクセスできるかを指定する。


### アイデンティティベースポリシー
IAMで指定するポリシーである。
アイデンティティベースポリシーでは以下の二つがあることを意識すると良い
- 許可ポリシー
- 信頼ポリシー


#### 信頼ポリシー
信頼ポリシーはRoleに付与され、対象のRoleを引き受けることができる対象を指定する。
これは、RoleがAWSリソースに紐づけられることを前提にしており、Roleを誰に付与することができるかの定義をする必要があるために利用される。




#### 許可ポリシー
許可ポリシーはポリシーを引き受けた対象がどのようなアクションを実行することができるかを指定する。
また、Conditionをりようすることで、許可される対象を絞ることができるので、信頼ポリシーで許可された対象からさらに絞り込みを行うこともできる。


## 許可ポリシーの読み解き方

### 読み解き方
セクションでみると、以下の順番で読み解く
1. Conditon:この条件を満たしている場合、
2. Resource:このリソースを対象として、
3. Action/Effect: このアクションを許可する

※Effectが"Deny"であったり、Actionが"NotAction"であったとしても、上記に当てはまるように考えると読み解きしやすくなる。

最終的に文面に書き起こす時は以下の穴を埋める形で考えると良い
```
[Condition]を満たしている場合、[Resource]を対象とした[Action]を[Allow]している。
※[Condition]を満たしていない場合何も許可していない
※[Resource]の対象外のリソースには何も許可されていない
※[Action]に指定されていないアクションは許可されていない
```



### Effectについて
許可をする"Allow"もしくは"Deny"を指定する。  
"何をAllow"しているかを意識するとpolicyを読み解きやすい。


### Principalについて
対象のポリシーが適用される主体を指定する。  
"誰にAllow"しているかを意識するとpolicyを読み解きやすい。

principalを利用するユースケースは以下の2つがほとんどであり、許可ポリシーの中では指定されない
- クロスアカウントのための信頼ポリシーでの指定
- リソースベースポリシーでの指定

#### NotPrincipal
principalで指定された対象以外を対象とする。  
かなり広範囲を対象とするため、利用には慎重になる必要がある。



### Actionについて
アクションセクションでは、リソースに対するアクションを指定することで、対象のリソースに対する認可の指定ができる。  
"何のアクション"が対象かを意識するとpolicyを読み解きやすい。

#### NotAction
NotActionで指定した場合、指定されたアクション以外を対象とする。
以下の例では、S3のGetObjectとListBucket以外を許可している。
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "NotAction": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": "*"
        }
    ]
}
```

#### NotActionとDeny
NotActionとDenyを組み合わせることで、指定されたアクション以外を拒否する制御になるため、結果的に指定されたアクションだけを許可することになる。
以下の例では、S3のGetObjectとListBucket以外を拒否することで結果的に、2つのアクションだけが許可されている。

Allowと同じような制御となっているが、NotActionとDenyの組み合わせは「対象のアクションのみを例外的に許可している」というニュアンスになる。
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "NotAction": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": "*"
        }
    ]
}
```

### Resourceについて
アクションを実行する対象のAWSリソースを指し示す。  
以下の例では、特定のS3だけが対象となっている
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "NotAction": [
                "s3:DeleteBucket",
                "s3:PutBucketPolicy"
            ],
            "Resource": "arn:aws:s3:::examplebucket/*"
        }
    ]
}
```
#### NotResouce
指定されたリソース以外のリソースを対象とする。


### Conditionについて
Policyで指定された許可を付与する対象を制限を行うことができる。  
例えば以下の例では、s3のアクション全てを実行できるが、特定のIP経由でないと実行できない
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::examplebucket/*",
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": "192.168.100.0/24"
                }
            }
        }
    ]
}
```

Conditionのセクションは以下で構成される
- 比較演算子：EX)IpAddress
- キー：EX)aws:SourceIp
- 値：EX)192.168.100.0/24



#### 比較演算子
比較演算子の一覧は[ユーザーガイド](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html)を参照されたい。

いくつか代表的なものについて解説する。
##### StringEquals
対象の条件の文字列と条件が完全に一致するか
##### StringLike
対象の条件の文字列と条件が一致するか。ワイルドカードなどを利用することができる
##### IfExists
他の比較演算子と組み合わせることができ、"StringLikeIfExists"のように利用できる。
対象のキーが存在した場合に適用されるが、もし対象のキーが存在しない場合は、対象のCondition自体が無視される点に注意。
##### IpAddress
IPアドレスと一致するか。CIDR形式に対応することができる点がStringLikeとの違い。


#### キー
ポリシーのキーで指定できるコンテキストキーは、グローバル条件コンテキストキーまたはサービス固有のコンテキストキーとなる。
- グローバル条件コンテキストキーは"aws:"というプレフィックスがついている  
条件キーの詳細は[AWS グローバル条件コンテキストキー](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/reference_policies_condition-keys.html)を参照されたい。

- サービス固有の条件では例えば"iam:"や"ec2"というプレフィックスがついている  
条件キーの詳細は[サービス認証リファレンス](https://docs.aws.amazon.com/ja_jp/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html)を参照されたい。


