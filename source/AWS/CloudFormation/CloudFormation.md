# CloudFormationの基本
テンプレートファイルと呼ばれるAWSリソースの定義を記述したyamlやjson形式ファイルを実行することで記述されたAWSリソースを自動で管理することができるIaCの技術。

## CloudFormationの価値
インフラについてコードとして記述することで以下のメリットが得られる
- 再現性の担保による実行の品質を担保できる
- 実行と削除の作業効率向上
- コードとして管理することによるバージョン管理

AWSのWell-Architectedの柱である「運用上の優秀性」のポイントとして「運用をコードとして実行する」が定義されている。


## 公式ドキュメント
 CloudFormationの基本的な挙動については以下を確認すると良い。
- [Userガイド](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)

実際にCloudFormationを記述するときには各リソースの定義方法を以下で確認すると良い。
- [リソースリファレンス](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)



## テンプレートファイルとスタックとリソース
テンプレートファイルを実行することで作成されるリソースのセットをスタックと呼ぶ。
スタックによって作成されたAWSのサービスひとつひとつはリソースと呼ばれる。

テンプレートファイルを実行する際に、リソース間の依存関係などは解決される。
例えば、リソースAの値をリソースBで参照するような記述をしておけば、自動でCloudFormationはリソースAを作成してからリソースBを作成してくれる。



## ネステッドスタック


## CloudFormationの実行と更新
### 実行
### チェンジセット
### ドリフト検出


## 組み込み関数と疑似パラメータ
### 組み込み関数
AWSが準備している関数であり、実行されるまでわからない値などを代入する際に利用する。
よく利用するものを以下に整理するが、一覧は[公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html)参照

|組み込み関数名|説明|利用例|
|:----|:----|:----|
|Ref|指定したパラメータやリソースの物理名を参照することができる。リソースの物理名は名前だったり、ARNだったりするが、|!Ref logicalName|
|組み込み関数名|説明|利用例|
|:----|:----|:----|
|Ref|指定したパラメータやリソースの物理名を参照することができる。リソースの物理名は名前だったり、ARNだったりするが、|!Ref logicalName|
|Fn::GetAtt|論理名で指定したAWSリソースの属性を参照することができる。どのような属性が定義されているかは参照先のAWSのリソースに依存|!GetAtt logicalNameOfResource.attributeName|
|Fn::Sub|入力文字列の変数を、指定した値に置き換える|!Sub 'arn:aws:ec2:${AWS::Region}:${AWS::AccountId}:vpc/${vpc}'|
|Fn::FindInMap|Mappingセクションと合わせて利用して、Mappingセクションで指定された値を参照することができる。|!FindInMap [ MapName, TopLevelKey, SecondLevelKey ]|
|Fn::ImportValue|クロススタック参照を利用して、別スタックの出力を参照する場合に利用する|!ImportValue sharedValueToImport|
|条件関数|Fn::If、Fn::Equals、Fn::Not などの組み込み関数を使用して、条件付きでスタックリソースを作成|ConditionとFn::If、Fn::Equals、Fn::Notの組み合わせ|

### Ref詳細
### GetAtt詳細
### Sub詳細
### FindInMap詳細
### ImportValut詳細







### 疑似パラメータ
AWSが事前に定義したパラメータであり、アカウントIDやリージョンが準備されており、実行環境によって参照してくれる。
組み込み関数の"Sub"や"Ref"で利用することができる。

|疑似パラメータ|返却値|
|:----|:----|
|AWS::AccountId|スタックが作成されているアカウントの AWS アカウント ID
|AWS::Partition|リソースがあるパーティション。標準の AWS リージョンの場合、パーティションは aws
|AWS::Region|リソースが作成されているリージョンを表す文字列(ap-northeast-1)|
|AWS::StackId|スタックの ID|
|AWS::StackName|スタックの名前|
|AWS::URLSuffix|ドメインのサフィックスを返します。サフィックスは、通常 "amazonaws.com"|
|AWS::NoValue|Fn::If 組み込み関数の戻り値として指定すると、対応するリソースプロパティは作成されない|
|AWS::NotificationARNs|CloudFormation スタックが通知を送信する SNS トピックの一覧|

例えば、作成されるリソースのタグにStackNameを付与するとリソースからStackを参照することができる
```yaml
Tags:
    - 
        Key: CreatedBy
        Value: !Ref "AWS::StackName"
```
