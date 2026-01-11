# ServiceMesh
サービスメッシュとは、マイクロサービス間通信をアプリケーションの外側で一元的に制御するためのアーキテクチャである。
アプリケーションコードに通信制御を記載しないことで通信のルールを一元的に管理することができる。

ポイントとしては、East-Westトラフィックと呼ばれ、クラスタ内部・サービス内部の通信を制御する。

## 用語
### Istio(制御レイヤ)
サービスメッシュを実現する代表的なOSSであり、通信ルールやポリシーを定義する。  
コントロールプレーンであり、実際の通信は行わない。

役割としては大きく以下
- セキュリティ
    - mTLS
    - AuthorizationPolicy(L7)
    - NetworkPolicy(L4)
- トラフィック制御
    - VirtualService(L7)

#### mTLS
- Pod 間通信を暗号化・相互認証
- STRICT で全通信必須化

#### NetworkPolicy
- Pod 間通信の可否を制御
- URL 単位制御は不可

#### AuthorizationPolicy
- HTTP path / method / source 単位で認可
- DENY が最優先
- ALLOW に一致しなければ拒否

#### トラフィック制御
- VirtualService：L7 ルーティング
- DestinationRule：接続・subset 定義



### Envoy（実働レイヤ）
高機能のL7プロキシであり、各podにサイドカーとして配置される。  
データプレーンであり、実際の通信・ヘルスチェック・リトライを担当

アプリコンテナの横に配置される補助コンテナであり、アプリ通信を横取りしてEnvoyに流す