# CloudFormationのパラメータ参照
## テンプレート内のパラメータの参照
RefやGetAttを利用して、参照する場合、CloudFormationは自動で依存関係を解決してくれる。

!Subを利用して、ARNを組み立てると依存関係を解決してくれない点に注意。
どうしても！Subを利用する場合は、DependsOnリソースを利用して、作成順序を担保する
```yaml
Resources:
  MyRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: role-for-my-bucket
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: s3.amazonaws.com
            Action: sts:AssumeRole

  MyBucket:
    Type: AWS::S3::Bucket
    DependsOn: MyRole
    Properties:
      BucketName: !Sub 'my-bucket-${AWS::Region}-${AWS::AccountId}'
      LifecycleConfiguration:
      BucketPolicy:
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:role/role-for-my-bucket'
              Action: 's3:*'
              Resource: !Sub 'arn:aws:s3:::my-bucket-${AWS::Region}-${AWS::AccountId}/*'
```


## テンプレート間のパラメータ参照
### クロススタックリファレンス
ExportとImportを利用することで、同一アカウントのリージョン内でパラメータを参照することが可能。
依存関係となるので、Import側を削除してからでないと、Export側は削除できなくなる。

具体例としては、出力側でエクスポート（Output）の定義を行う。`MyVPCId`という値で、VPCのIDがエクスポートされている。注意点として、エクスポートする際のエクスポート名はアカウント内のリージョンでユニークになる必要がある。
```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

Outputs:
  VPCId:
    Description: "VPC ID"
    Value: !Ref MyVPC
    Export:
      Name: MyVPCId
```

Import側では先ほどのImportValueの構文を利用する。
```yaml
Resources:
  MySubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !ImportValue MyVPCId
      CidrBlock: 10.0.1.0/24
```



### ネステッドスタックでの参照
ネステッドスタックにおいては、クロススタックリファレンスのようにImportValueを利用せず、NestedStack-AでExportした値をroot-stackで受け取って、GetAttとして受け取ることができる。これをパラメータとして別のNestedStack-Bに渡すことができる。[re:Postの記事参照](https://repost.aws/ja/knowledge-center/cloudformation-nested-stacks-values)

この方法を利用することで、NestedStack内での依存関係をCloudFormationが自動で解決してくれるとともに、人間の目線でもRootStackで把握しやすくなるというメリットがある。一方でImportValueを利用した場合、AWS側は自動では依存関係を解決してくれない点に注意。


具体的例を挙げると、Nested-AではOutputsセクションで値をエクスポート
```yaml
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

Outputs:
  VPCId:
    Description: "VPC ID"
    Value: !Ref MyVPC
```

RootStackではNested-AがOutputした値をGetAttで取得し、Nested-Bに対してパラメータとして渡す。
```yaml
Resources:
  NestedStackA:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: "https://s3.amazonaws.com/mybucket/NestedStackA.template"
  
  NestedStackB:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: "https://s3.amazonaws.com/mybucket/NestedStackB.template"
      Parameters:
        VPCId: !GetAtt NestedStackA.Outputs.VPCId
```

最終的にNested-Bではパラメータセクションで受け取るだけで良い
```yaml
Parameters:
  VPCId:
    Type: String

Resources:
  MySubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPCId
      CidrBlock: 10.0.1.0/24
```

### ダイナミックリファレンス
パラメータストアやシークレットマネージャーに格納されている値を直接参照することができる。

CloudFormationのリソースセクションで記述すると、CloudFormationのリソースセクションやMappingセクションに機密情報を残すことなく参照を解決することができるという強みがある。

#### パラメータストアの参照
パラメータストアの2つのパターンに対応している
- ssm: Systems Manager パラメータストアのプレーンテキストパラメータ。
- ssm-secure: Systems Manager パラメータストアの安全な文字列パラメータ。

記述としては以下で構成される
```
'{{resolve:ssm:parameter-name:version}}'
'{{resolve:ssm-secure:parameter-name:version}}'
```

以下のように記述することで、参照が可能であり、!Subや!Refは不要
```yaml
 MyS3Bucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      AccessControl: '{{resolve:ssm:S3AccessControl:2}}' 
```

安全な文字鉄を解決するときは以下のように記述
```yaml
  MyIAMUser:
    Type: AWS::IAM::User
    Properties:
      UserName: 'MyUserName'
      LoginProfile:
        Password: '{{resolve:ssm-secure:IAMUserPassword:10}}'
```

#### シークレットマネージャの参照
記述方法はパラメータストアと同様
```yaml
`{{resolve:secretsmanager:secret-id:secret-string:json-key:version-stage:version-id}}`
```

具体例は以下
```yaml
MyRDSInstance:
    Type: 'AWS::RDS::DBInstance'
    Properties:
      DBName: MyRDSInstance
      AllocatedStorage: '20'
      DBInstanceClass: db.t2.micro
      Engine: mysql
      MasterUsername: '{{resolve:secretsmanager:MyRDSSecret:SecretString:username}}'
      MasterUserPassword: '{{resolve:secretsmanager:MyRDSSecret:SecretString:password}}'
```