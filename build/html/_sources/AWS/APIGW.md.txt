# API Gateway
Web APIの作成や公開、管理を行うためのフルマネージドサービス。
サーバーレスのため、APIの実行基盤のサーバー構築や維持管理はAWSが責任を負い、開発者はAPIの開発に集中することができる。

API GWでは、APIとして`REST API`と`WebSocket`を提供している。

## note
[Blackbelt](https://pages.awscloud.com/rs/112-TZM-766/images/20190514_AWS-Blackbelt_APIGateway_rev.pdf)


## REST API
### 全体像
API GWでは、`リソースとメソッド`を作成し、それぞれのメソッドに対して`メソッド設定`を行うことでAPIを開発する。

![](../img/AWS/apige_restapi_setting.png)
[Blackbelt](https://pages.awscloud.com/rs/112-TZM-766/images/20190514_AWS-Blackbelt_APIGateway_rev.pdf)から引用


### リソース
`リソース`とは、APIで利用される`パス`である。  
`/`を最上位として、階層構造でリソースを定義する。  
上記の例では「pets」や「ptes/{petId}」がリソースとして定義されている。

最終的にAPI GWをデプロイすることで発行されるAPIのURLは以下のようになり、URLの末尾にリソースを付与して、各リソースに問い合わせをしていく。
```url
https://{api-id}.execute-api.{region}.amazonaws.com/{stageName}/pets
```
- {api-id}: AWSがAPIに割り当てる一意の識別子
- {stageName}: ステージ名(後述)


### メソッド
`メソッド`ではリソースで受け付けるHTTPのメソッドを事前に定義する。
受け付けることができるのは以下の7メソッド
- GET
- POST
- PUT
- HEAD
- DELETE
- OPTIONS
- PATCH
- ANY


### メソッド設定
リソースとメソッドの組み合わせごとに`メソッド設定`と呼ばれる処理フローを定義する。  
定義は4つのセクションに分かれている
- メソッドリクエスト
- 統合リクエスト
- 統合レスポンス
- メソッドレスポンス


#### メソッドリクエスト
***受け付けたリクエストに対するチェック処理***
以下のような処理が可能
- バリデーション：クエリ文字列やヘッダーなどの必須設定
- IAM Roleによる認証と認可


#### 統合リクエスト
***受け付けたリクエストをバックエンドに渡すための処理***

以下のような処理が可能
- バックエンドタイプの定義：受付可能なバックエンドは５つ
    - Lambda: 作成したLambdaとの統合
    - HTTP: 外部APIなどとの統合(HTTPプロトコルを指し、HTTP/HTTPS両方対応)
    - Mock: プロトタイピング用
    - AWS サービス: Lambda以外のAWSサービス(DynamoやSQS etc..)
    - VPCリンク: VPCリンクが作成されているサービス
- リクエストをカスタマイズして、バックエンド向けに変換

#### 統合レスポンス
***バックエンドから帰ってきたレスポンスを整形する処理***

以下のような処理が可能
- レスポンスをカスタマイズしてクライアント向けに変換
- エラーが発生していた場合のエラーハンドリング定義

#### メソッドレスポンス
***クライアントに渡すための最終調整処理***

以下のような処理が可能
- クライアントに返すためのステータスコードの定義
- レスポンスの形式やHTTPヘッダーの追加

#### プロキシ統合
プロキシ統合（Lambdaプロキシ統合またはHTTPプロキシ統合）は、API Gatewayとバックエンドサービス間のリクエストとレスポンスのマッピングを簡素化するための機能。
- クライアントからのリクエストはほとんどそのままの形でバックエンドサービスに渡される
- バックエンドからのレスポンスもまた、プロキシ統合を通じて直接クライアントに返される

利用時には、リクエストとレスポンスが固定のフォーマットに従っている前提の機能となる。

### エンドポイントタイプ
クライアント側から見た、アクセス先エンドポイントのタイプを3つから選択することができる
- エッジ最適化
- リージョン
- プライベート

![](../img/AWS/apigw_endpoint_type.png)
[Blackbelt](https://pages.awscloud.com/rs/112-TZM-766/images/20190514_AWS-Blackbelt_APIGateway_rev.pdf)から引用







## デプロイ
作成したAPIをデプロイし公開するためには、ステージとデプロイメントという概念を理解する必要がある。

### ステージ
ステージとは、API GWのバージョンのことであり同じAPIに対して異なるバージョンを設定することができる機能。
利用方法としては以下が挙げられる
- バージョン管理：リリースごとにステージを分ける
- 環境分離：dev/stg/prdのように、リリースの段階に応じてステージを分ける
- 設定カスタマイズ：独自の設定ごとにステージを分ける

発行されるURLは、ステージ名を含むこととなる
```shell
https://{api-id}.execute-api.{region}.amazonaws.com/{stageName}/...
```

### デプロイメント
実際に、ステージに対してURLを発行するにはデプロイメントが必要。

## キャッシュ
パフォーマンスと API 実行速度を向上させるため、オプションとしてお客様の API の各ステージに専用のキャッシュをプロビジョニングすることができる。キャッシュには費用がかかるが、その分APIGW側のリクエスト数やデータ転送量を削減できるため、トレードオフとなる。## 監視モニタリング


## 料金
詳細は、[公式ドキュメント](https://aws.amazon.com/jp/api-gateway/pricing/)参照。

受信した API コール数と、転送データ量に対してのみ料金が発生する従量課金制。

かかる費用の全体像としては以下
```
実行回数＋データ転送料金＋キャッシュメモリ量
```

### 実行回数
REST APIの目安としては、100万リクエスト当たり4USD


## バージョニング
★★★★★Coming Soon★★★★★
## APIキーと使用量プラン
★★★★★Coming Soon★★★★★
### エラーパターン
★★★★★Coming Soon★★★★★




## WebSocket
★★★★★Coming Soon★★★★★



## Cloudformationのテンプレートファイル
### RestAPI
[参考サイト](https://blog.usize-tech.com/integration-apigateway-and-lambda/)
```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: CloudFormation template that creates API Gateways and relevant IAM roles.

# ------------------------------------------------------------#
# Input Parameters
# ------------------------------------------------------------#
Parameters:
  EnvID:
    Type: String


Resources:

  # ------------------------------------------------------------#
  # IAM Role
  # ------------------------------------------------------------#
  # API GWの実行Role
  RoleApiGwExecRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "role-${EnvID}-ApiGwExecRole"
      Description: API Gateways Exec Role.
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - 
            Effect: Allow
            Principal:
              Service:
                - apigateway.amazonaws.com
            Action:
                - sts:AssumeRole
      MaxSessionDuration: 3600
      Path: /
      ManagedPolicyArns:
          - arn:aws:iam::aws:policy/service-role/AWSLambdaRole
          - arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
          - arn:aws:iam::aws:policy/AmazonKinesisFirehoseFullAccess
      Tags:
        - 
          Key: CreatedBy
          Value: !Ref "AWS::StackName"


  # ------------------------------------------------------------#
  # API Gateway
  # ------------------------------------------------------------#

  # RestApiのベース
  ApiGwRestBase:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: !Sub ApiGw-${EnvID}
      Description: REST API Gateway
      EndpointConfiguration: #エンドポイント設定
        Types:
          - REGIONAL
      Tags:
        - 
          Key: CreatedBy
          Value: !Ref "AWS::StackName"



  # リソース(パス設定)
  RestApiResource:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref ApiGwRestBase                   #リソースが所属するRest
      ParentId: !GetAtt ApiGwRestBase.RootResourceId  #リソースが所属するRestのID
      PathPart: GetExampledata                        #リソースのパス



  # Postメソッド
  RestApiMethodPost:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref ApiGwRestBase                   #メソッドが所属するRest API
      ResourceId: !Ref RestApiResource                #メソッドが所属するリソース
      HttpMethod: POST                                #メソッドのタイプ
      AuthorizationType: NONE                         #認証を利用する場合のタイプ
      Integration: # バックエンドの指定
        Type: AWS_PROXY                               #統合タイプ
        IntegrationHttpMethod: POST 
        Credentials: !GetAtt RoleApiGwExecRole.Arn    #APIGWに付与するRole
        #バックエンドの参照
        Uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:ここにLambda関数名を記載/invocations
        PassthroughBehavior: WHEN_NO_MATCH
      MethodResponses: #メソッドレスポンス
        - StatusCode: 200
          ResponseModels:
            application/json: Empty
          ResponseParameters:
            method.response.header.Access-Control-Allow-Origin: true



  # デプロイメント
  RestApiDeployment:
    Type: AWS::ApiGateway::Deployment
    Properties:
      RestApiId: !Ref ApiGwRestBase
    DependsOn:
      - RestApiMethodPost


  # ステージ
  RestApiStage:
    Type: AWS::ApiGateway::Stage
    Properties:
      StageName: prod #stage名
      Description: production stage
      RestApiId: !Ref ApiGwRestBase           #対象のRest
      DeploymentId: !Ref RestApiDeployment    #対象のデプロイメント
      TracingEnabled: true
      Tags:
        - 
          Key: CreatedBy
          Value: !Ref "AWS::StackName"

```