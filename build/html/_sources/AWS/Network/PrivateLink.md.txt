# PrivateLink
別VPCに所属するアプリケーションに対して、AWS内のみを介したPrivateな通信でアクセスすることができる技術をPrivateLinkと呼ぶ。

![](../../img/AWS/Network/PrirvateLink.png)
[【初心者向け】VPCエンドポイントとAWS PrivateLinkの違いを実際に構築して理解してみた](https://dev.classmethod.jp/articles/aws-vpcendpoint-privatelink-beginner/)より引用

Private Linkではサービスの利用側とサービスの提供側それぞれで設定が必要。

## サービス提供側
VPC Endpointサービスの作成を行う。その際、ロードバランサーとしてNLBを選択する。


## サービス利用側
VPC Endpointの作成を行う。その際のAWSリソースの種類としては、その他のエンドポイントサービスを指定し、サービス提供側で作成した、エンドポイントサービスのサービス名を指定する。

その後、認証依頼をサービス提供側に送信し、認証してもらう。

