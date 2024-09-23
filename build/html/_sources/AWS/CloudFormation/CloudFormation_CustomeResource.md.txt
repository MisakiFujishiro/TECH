# カスタムリソース
カスタムリソースとは、CloudFormationからLambdaを呼び出し、デフォルトのCloudFormationではできないような処理を行うための機能である。
基本的な流れとしては、以下になる
1. Custome Resourceとして呼び出すLambdaを開発
2. CloudFormationのCustomeResourceとして対象のLambdaを指定
3. CloudFormationのCustomeResourceのパラメータを設定し、LambdaへParameterを渡す

## Lambdaの呼び出し
### CloudFomation側の指定
CloudFormationでは、TypeとしてCustomeを指定した上で、ServiceTokeとして、lambdaのARNを指定する。
また、同じテンプレート内で作成したリソースの値をパラメータとして渡すことができ、作成したリソースに対する操作をテンプレート内で実施することができる。
```yaml
SampleCustomResource: 
  Type: 'Custom::CustomResourceName'
  Properties:
    ServiceToken: !Sub 'arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:lambda-template'
    ParamX: !Ref XXXXX
    ParamY: !GetAtt YYYY.Id
```


### CloudFormationの呼び出しEvent
CloudFormationからCustomeResourceとしてLambdaを呼び出す際には、以下のEventがLambdaへ渡される。
CustomeResourceの作成・削除・更新のステータスを表すRequestTypeや、Parameterとして指定した値を格納しているResourcePropertiesなどが重要。
```json
{
   "RequestType" : "Create",
   "ResponseURL" : "http://pre-signed-S3-url-for-response",
   "StackId" : "arn:aws:cloudformation:ap-northeast-1:123456789012:stack/mystack/{xxx-yyy-zzz}",
   "RequestId" : "unique id for this create request",
   "ResourceType" : "Custom::Resource",
   "LogicalResourceId" : "MyTestResource",
   "ResourceProperties" : {
      "ParamX" : "ValueX",
      "ParamY" : "ValueY",
   }
}
```

設定されるKey-Valueの詳細は
[公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/crpg-ref-requests.html)
参照


### Lambda側の設定
CustomeResourceが実行されるタイミングはCloudFormationが実行されるタイミングであり、そのリクエストのタイプには以下の3つがある
- 作成時（Create）
- 更新時（Update）
- 削除時（Delete）

CustomeResourceどのリクエストタイプで実行されているかは、CloudFormationからLambdaを呼び出す時イベント内に含まているため、
Lambda側でもそれぞれのEventタイプに応じた場合分けを行う。

```py
from cfnResponse import send

def main(event, context):
    try:
        XXXX = event['ResourceProperties']['ParamX']
        YYYY = event['ResourceProperties']['ParamY']
        if event['RequestType'] == 'Create':
            # 作成イベントに対応する処理
        send(event, context, cfnresponce.SUCCESS, {'Response': 'Success'})
        if event['RequestType'] == 'Delete':
            # 削除イベントに対応する処理
        send(event, context, cfnresponce.SUCCESS, {'Response': 'Success'})
        if event['RequestType'] == 'Update':
            # 更新イベントに対応する処理
        send(event, context, cfnresponce.SUCCESS, {'Response': 'Success'}, event.get('PhysicalResourceId')
    except Exception as e:
        send(event, context, cfnresponce.FAILED, {})
```


### Physical Resource ID
CustomeReosurceでUpdate処理を行なった際、CloudFormationのCleanUpのタイミングでDeleteの実行がされてしまう問題が発生した。これは、PhysicalResourceIDの扱いが正しく行われていないために発生していた。

CloudFormationからカスタムリソースに対して、Update実行する際にPhysicalResourceIDをやりとりしている。
Lambda側から返すPhysicalResourceIDとCloudFormation側で管理しているPhysicalResourceIDが異なると、CloudFormatinでは、新しいカスタムリソースに更新されていると認識し、古いカスタムリソースを削除する動作をする。そのため、Deleteが流れてしまう。

具体的には、カスタムリソースのUpdateでは以下のEventがLambdaに送信されている。
このPhysicalResourceIdについて、異なる値が返却されると上記の削除が実行されてしまう。

PhysicalResourceIdについては、自セクションで説明するが、cfn-responseモジュールのデフォルトで実行時のログストリームの値が利用されてしまうので、意識的に設定をしないと削除に流れやすい点に注意。
```json
{
  "RequestType": "Update",
  "ServiceToken": "arn:aws:lambda:us-west-2:*******:function:set_identity_notifications",
  "ResponseURL": "https://cloudformation-custom-resource-response-uswest2.s3-us-west-2.amazonaws.com/arn%3Aaws%3Acloudformation%3Aus-west-2%3A*******%3Astack/ses-suppression-register/6ba8e740-4989-11ea-9aac-06be751c1bd6%7CProvideSetId******",
  "StackId": "arn:aws:cloudformation:us-west-2:*******:stack/ses-suppression-register/6ba8e740-4989-11ea-9aac-06be751c1bd6",
  "RequestId": "c3a874a5-a181-4a5c-88fc-d2b2e1504aa0",
  "LogicalResourceId": "ProvideSetIdentityNotifications",
  "PhysicalResourceId": "2020/02/07/[$LATEST]0b367e84e59e46798738c00bece897be",
  "ResourceType": "AWS::CloudFormation::CustomResource",
  "ResourceProperties": {
    "ServiceToken": "arn:aws:lambda:us-west-2:*******:function:set_identity_notifications",
    "HeadersInBounceNotificationsEnabled": "False",
    "ComplaintTopic": "",
    "HeadersInComplaintNotificationsEnabled": "False",
    "BounceTopic": "arn:aws:sns:us-west-2:*******:ses-suppression-list-topic",
    "Identities": ["willco21.test.com", "example.co.jp"]
  },
  "OldResourceProperties": {
    "ServiceToken": "arn:aws:lambda:us-west-2:*******:function:set_identity_notifications",
    "HeadersInBounceNotificationsEnabled": "False",
    "ComplaintTopic": "",
    "HeadersInComplaintNotificationsEnabled": "False",
    "BounceTopic": "arn:aws:sns:us-west-2:*******:ses-suppression-list-topic",
    "Identities": ["willco21.test.com"]
  }
}
```
[【AWS】【Cloudformation】CustomResource作ったら、Stack更新時にDeleteアクション実行してしまう件](https://qiita.com/willco21/items/534436ba89a3c6f217ac)


#### cfn-responseモジュール
[cfn-response モジュールの公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/cfn-lambda-function-code-cfnresponsemodule.html)
で記載されているように、Lambda側のソースコードでphysicalResourceIdを指定しないと、CloudWatchのログストリームが指定される。
```
physicalResourceId
オプション。関数を呼び出したカスタムリソースの一意の識別子。モジュールのデフォルトでは、Lambda 関数に関連付けられている Amazon CloudWatch Logs ログストリームの名前が使用されます。

PhysicalResourceId に返された値は、カスタムリソース更新オペレーションを変更できます。返される値が同じであれば、通常の更新と見なされます。返された値が異なる場合には、CloudFormation は新しい方が更新用のものであると認識し、古いリソースに削除リクエストを送信します。
```

#### 対策
CustomeResourceのLambdaの設計として、Updateのタイミングでも同じPhysicalResoruceIDを返すような仕組み作りが必要。以下のようにリクエストEventに含まれているPhysicalResoruceIDをそのままレスポンスとして返す。

```py
        if event['RequestType'] == 'Update':
            # 更新イベントに対応する処理
        send(event, context, cfnresponce.SUCCESS, {'Response': 'Success'}, event.get
```