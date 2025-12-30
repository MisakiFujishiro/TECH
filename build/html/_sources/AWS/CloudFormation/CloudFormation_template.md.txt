# CloudFormationのテンプレートファイルの書き方

## テンプレートの書き方
### おまじない
テンプレートファイルの最初に記載する部分で、AWS CloudFormationがテンプレートを正しく認識するために必要な情報を記述。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  This is a sample CloudFormation template.
  It creates a simple S3 bucket.
```


### Parameter
スタック作成時にユーザーから入力を受け取るためのパラメータを定義

```yaml
# --------------------
# Parameters
# --------------------
Parameters:
  EnvironmentType:
    Description: "Specify the environment type (e.g., dev, prod)."
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - prod
    ConstraintDescription: "The environment type must be either 'dev' or 'prod'."
```

### Condition
リソースの作成を条件付きで行うための条件を定義
```yaml
# --------------------
# Conditions
# --------------------
Conditions:
  IsProdEnvironment: !Equals [ !Ref EnvironmentType, prod ]
```


### Mapping
Mapping定義で利用するMapの設定を定義
```yaml
# --------------------
# Mappings
# --------------------
Mappings:
  RegionMap:
    us-west-1:
      S3Endpoint: "s3-us-west-1.amazonaws.com"
    ap-northeast-1:
      S3Endpoint: "s3-ap-northeast-1.amazonaws.com"
```


### Resources
Resourcesセクションは必須項目であり、作成するリソースの定義を行う
```yaml
# --------------------
# Resources
# --------------------
Resources:
  MyS3Bucket:
    Type: "AWS::S3::Bucket"
    Properties:
      BucketName: !If 
        - IsProdEnvironment
        - !Sub "my-s3-bucket-${EnvironmentType}"
        - !Ref AWS::NoValue
```

### Output
スタックの出力値を定義します。これにより、他のスタックや外部のシステムがこれらの値を参照できるようになります。
```yaml
# --------------------
# Outputs
# --------------------
Outputs:
  BucketName:
    Description: "The name of the S3 bucket"
    Value: !Ref MyS3Bucket
  BucketEndpoint:
    Description: "The endpoint of the S3 bucket"
    Value: !FindInMap
      - RegionMap
      - !Ref "AWS::Region"
      - S3Endpoint
```

## 具体例
上記をまとめると以下のようなテンプレートとなる。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  This is a sample CloudFormation template.
  It creates a simple S3 bucket.
# --------------------
# Parameters
# --------------------
Parameters:
  EnvironmentType:
    Description: "Specify the environment type (e.g., dev, prod)."
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - prod
    ConstraintDescription: "The environment type must be either 'dev' or 'prod'."
# --------------------
# Conditions
# --------------------
Conditions:
  IsProdEnvironment: !Equals [ !Ref EnvironmentType, prod ]
# --------------------
# Mappings
# --------------------
Mappings:
  RegionMap:
    us-west-1:
      S3Endpoint: "s3-us-west-1.amazonaws.com"
    ap-northeast-1:
      S3Endpoint: "s3-ap-northeast-1.amazonaws.com"
# --------------------
# Resources
# --------------------
Resources:
  MyS3Bucket:
    Type: "AWS::S3::Bucket"
    Properties:
      BucketName: !If 
        - IsProdEnvironment
        - !Sub "my-s3-bucket-${EnvironmentType}"
        - !Ref AWS::NoValue
# --------------------
# Outputs
# --------------------
Outputs:
  BucketName:
    Description: "The name of the S3 bucket"
    Value: !Ref MyS3Bucket
  BucketEndpoint:
    Description: "The endpoint of the S3 bucket"
    Value: !FindInMap
      - RegionMap
      - !Ref "AWS::Region"
      - S3Endpoint
```