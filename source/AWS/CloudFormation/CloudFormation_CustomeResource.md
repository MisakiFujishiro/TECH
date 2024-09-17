

https://qiita.com/willco21/items/534436ba89a3c6f217ac
https://medium.com/@angusyuen/cloudformation-custom-resources-be-cautious-of-physicalresourceid-b4a359657c12


https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/template-custom-resources.html

https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-resource-cloudformation-customresource.html

https://medium.com/@sch.bar/a-deep-dive-on-aws-cloudformation-custom-resources-8e04c6d155fd


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
   "StackId" : "arn:aws:cloudformation:us-west-2:123456789012:stack/mystack/5b918d10-cd98-11ea-90d5-0a9cd3354c10",
   "RequestId" : "unique id for this create request",
   "ResourceType" : "Custom::TestResource",
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


### Physical Resource IDに気をつける



#### cfn-responseモジュール
[cfn-response モジュールの公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/cfn-lambda-function-code-cfnresponsemodule.html)
で記載されているように、Lambda側のソースコードでphysicalResourceIdを指定しないと、CloudWatchのログストリームが指定される。
```
physicalResourceId
オプション。関数を呼び出したカスタムリソースの一意の識別子。モジュールのデフォルトでは、Lambda 関数に関連付けられている Amazon CloudWatch Logs ログストリームの名前が使用されます。

PhysicalResourceId に返された値は、カスタムリソース更新オペレーションを変更できます。返される値が同じであれば、通常の更新と見なされます。返された値が異なる場合には、CloudFormation は新しい方が更新用のものであると認識し、古いリソースに削除リクエストを送信します。詳細については、
```