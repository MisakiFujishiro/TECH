# IAMの基本
AWS Identity and Access Management（IAM）はAWSに関する認証と認可を扱い、リソースへのアクセスを安全に管理するためのウェブサービスである。

## IAMのリソース
IAMのリソースには以下の4つがある。
- IAM Policy：認可に関する設定を行う
- IAM Role：許可されたPrincipalに引き受けられる
- IAM User：人に対して払い出される
- IAM Group：Userが所属できる。所属UserにはGroupに付与されたPolicyが引き継がれる。


認可情報はPolicyによって定義され、特定のRole、User、またはGroupに付与される。
それによってどのような操作が許可されるかが決定される。

認証は主にUserまたはRoleによって担われる。
Userは具体的な個人に関連付けられ、認証プロセス（例えば、パスワードによるログイン）を通じてシステムへのアクセスが許可される。

RoleはAWSリソースなどの特定の対象（Principal）に付与することができる。
Roleを使用する際は、Principalの定義に基づいて認証が行われる。

Groupは、複数のUserに対して一括でPolicyを適用するために使用される。
これにより、Policyの管理が簡素化され、効率的に行えるようになる。




## AWSリソースへのアクセスイメージ


![](../../img/AWS/iam/iam_whole_access_image.png)

