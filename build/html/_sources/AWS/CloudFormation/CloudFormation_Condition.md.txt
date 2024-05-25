# CloudFormationの条件関数
## 条件関数詳細
条件関数を使用すると、特定の条件に基づいてリソースの作成を制御することができます。
条件は主に「条件設定」と「評価設定」の二つのセクションで定義されます。
### 条件設定
条件設定では、!Equals、!And、!Or、!Notなどの論理演算子を使用して、条件の真偽を定義できます。
これにより、複雑な条件ロジックを構築することが可能です。

以下の例では、ConditionセクションでEqualsを利用して条件設定を行い、Environmentがproductionの場合のみTrueを返します。
```yaml
Conditions:
  IsProduction:
    Fn::Equals:
      - !Ref Environment
      - production
```

以下の例では、Conditionセクションで設定した設定値をさらにAndで組み合わせることで、Productionかつ設定されたインスタンスがLargeである場合にのみTrueを返す
```yaml
  ProductionOrLargeInstance:
    Fn::And:
      - Condition: IsProduction
      - Condition: IsLargeInstance
```


### 評価設定
評価設定では、!IfやConditionを使用して、条件に基づいてリソースの作成を評価します。
これにより、設定した条件応じたリソースの制御をすることができます。

Conditionはリソース自体の作成制御をするのに適しており、Ifはリソース内のプロパティについて条件に応じた設定変更に適している。

#### Condition
Conditionを利用することで、関連づけたリソースの作成有無を設定できます。

以下の例では、ConditionセクションでEqualsを利用して条件設定を行い、Environmentがproductionの場合のみ、バケットを作成する条件が定義されています。
```yaml
Resources:
  MyBucket:
    Type: 'AWS::S3::Bucket'
    Condition: IsProduction
```

#### If
!Ifを利用することで、関連づけたリソースについて、条件に基づいたリソース内のプロパティ設定の設定有無を制御できます。
以下の例では、Ifを利用して、Bucketの名前を変更している。
```yaml
Resources:
  MyBucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: !If
        - IsProduction
        - MyProductionBucket
        - MyDevelopmentBucket
```

以下の例では、疑似パラメータの"AWS::NoValue"と組み合わせることで、条件に合わない場合は対象のプロパティを設定しない。
```yaml
Resources:
  MyBucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: !If
        - IsProduction
        - ProductionBucket
        - !Ref AWS::NoValue
```