# IAMの基本
AWS Identity and Access Management（IAM）はAWSに関する認証と認可を扱い、リソースへのアクセスを安全に管理するためのウェブサービスである。

## AWSのアカウント
AWSのアカウントには２種類存在する
- Root User アカウント     
    全ての権限を持つユーザーのアカウント：基本的に利用しない
- IAM User アカウント
    IAMで作成されるユーザーのアカウント：利用者に応じた権限を付与して利用する

## IAMのリソース
IAMのリソースには以下の4つがある。
- IAM Policy：認可に関する設定を行う
- IAM Role：許可されたPrincipalに引き受けられる
- IAM User：人に対して払い出される
- IAM Group：Userが所属できる。所属UserにはGroupに付与されたPolicyが引き継がれる。

基本的な考え方としては認可情報が定義されたPolicyをRoleやUser、Groupに対して付与する。
Roleは引き受けることができる相手をPrincipalとして定義しており、Userは認証がされた人間と紐づいている。



## AWSリソースへのアクセスイメージ
![](../../img/AWS/iam/iam_whole_access_image.png)

