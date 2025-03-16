# SES
コスト効率の高いメールサービス。メールの送信および受信機能が備わっている。
受信と送信に関して、オプション機能を有している。

## SESのメール送信
SESを利用したメール送信方法には大きく3つの種類が存在。
|インターフェース|概要|特徴|
|:----|:----|:----|
|SESコンソール|AWSマネジメントコンソールで直接メール送信をテストできる|GUIで簡単に操作可能。設定の確認やテスト送信に便利|
|SES API|アプリケーションから直接メールを送信するためのAPIを提供|IAMで送信権限を設定する必要がある|
|｜---SendEmail API|From、To、Subject、Body を指定してメールを送信|シンプルなテキストメールやHTMLメールの送信が可能|
|｜---SendRawEmail API|メッセージ全体をアプリケーションで生成し、カスタムヘッダーや添付ファイルを含めて送信|より高度なメール構成が可能|
|SMTP インターフェース|SESが準備しているSMTPエンドポイントが経由でメールを送信|認証情報（SMTPクレデンシャル）が必要|


### メール送信の制限
SESでは、送受信制限が設定されている。
これは、SES自体がスパム判定されてしまうと、配信先のメールプロバイダーとの信頼関係が崩れてしまうためである。
具体的には、24時間帯の送信可能メールや1秒あたりの送信可能メール数に制約が設けられている

### サンドボックス
AWSアカウントでSESを初めて利用する場合、不正利用を防ぐためSES機能はサンドボックス状態となっている。
サンドボックス状態ではメール送信の宛先に制約がある他、メールの送信のスロットリングに制約が設けられる。

サンドボックスで要件を満たすことができない場合は、AWSサポート経由で上限緩和申請を行いプロダクションへ変更する必要がある。

|制約項目|サンドボックス (初期状態)|上限緩和後 (プロダクション)|備考|
|:----|:----|:----|:----|
|送信対象|検証済みメールアドレス/ドメインのみ|制限なし (任意の宛先に送信可能)|初期状態ではテスト用途に限定される|
|24時間あたりの送信可能メール数 (Sending Quota)|200通|AWSサポート申請で引き上げ可能|引き上げ後の上限はリージョンや使用状況による|
|1秒あたりの送信可能メール数 (Sending Rate)|1通/秒|AWSサポート申請で引き上げ可能|高速な大量送信が必要な場合は申請が必要|
|バウンス・苦情率|厳しく制限|過剰なバウンス・苦情があると制限|SESの信頼性維持のため、適切なリスト管理が必要|
|承認プロセス|なし (デフォルト)|上限緩和申請が必要|AWS サポートに申請して解除|

### SESのモニタリング
SESではメールに対して以下のようなメトリクスを取得し監視することができる。

|メトリクス名|説明|活用用途|
|:----|:----|:----|
|Send (送信数)|SESが送信リクエストを受け付けたメールの数|送信ボリュームの監視|
|Reject (拒否数)|SESがポリシー違反やブラックリスト登録などの理由で送信を拒否した数|不正な送信リクエストの監視、送信ポリシーの調整|
|Bounce (バウンス数)|送信したメールが受信者のメールサーバーに拒否された数|配信リストのクリーンアップ、信頼性向上|
|Complaint (苦情数)|受信者が迷惑メール報告を行った数|メール品質の改善、リスト管理|
|Delivery (配信数)|正常に受信者に配信されたメールの数|送信成功率の分析|
|Open (開封数) (new)|受信者がメールを開封した回数|メールのエンゲージメント測定|
|Click (クリック数) (new)|メール内のリンクがクリックされた回数|コンテンツの有効性分析|
|Rendering Failure (レンダリング失敗数) (new)|テンプレートのエラーなどによりメールが正常に生成されなかった回数|メールテンプレートの検証、エラー防止|

### バウンス
送信した際に、エラーになってしまうアドレスについては、ハードバウンスとしてSurppression Listに登録される。



## SESのメール受信




## 【実装】SESの実装イメージ
SESを利用してメールの受信をしてS3に配置。メールの送信を行う場合の実装イメージをまとめる。

### サンドボックスの解消
AWSアカウントで、初めてSESを利用する場合はサンドボックス状態となっている。
サンドボックスでは、認証したメールアドレスにしか送信をすることができないため、一般的なメールサーバーとしての使用ができない。
そのため、最初に上限緩和申請をしてプロダクションにする必要がある。

### 制約条件の確認
SESの送受信では、メールの容量に制約があるため、要件に一致しているかを確認する。

|項目|デフォルト制限|申請後の最大制限|
|:----|:----|:----|
|送信メールの最大サイズ|10MB|40MB|
|受信メールの最大サイズ|30MB|40MB|

また、ファイルを添付する場合SESとして添付を拒否している拡張子がある。
[Amazon SES unsupported attachment types](https://docs.aws.amazon.com/ses/latest/dg/mime-types.html#:~:text=You%20can%20send%20messages%20with,extensions%20in%20the%20following%20list)を確認して、条件を満たしているかを確認する。

### 受信実装
#### 利用ドメインの検証（TXTレコード）
独自のドメインを経由してメールの受信を行う場合、利用するドメインの所有者であることをSESに認識させる必要がある。
SESのコンソール上で利用するドメインを登録すると、登録するべきランダムな値が提示されるので、ドメイン側のTXTレコードとして、その値を設定する。
```
ホスト名（Name）: _amazonses.example.com
種類（Type）: TXT
値（Value）: SESが指定したランダムな文字列（例: `abcdef1234567890xyz=`）
TTL: 3600（またはデフォルト）
```

上記を設定して数時間でSESのコンソール側でドメインが認証されるので、認証後はTXTレコードを削除して良い。

#### 受信メール設定（MXレコード）
ドメイン宛のメールをSESに転送するため、MXレコードをDNS側に設定する。
値については、リージョンごとに固定となっている。この設定から分かる通り、ドメインレベルでSESへの転送を行う。
仕組みとしては、`@example.com`宛のメールについてはSESへの転送を行い、検証を行なっているSESへの配信を行う。

```
ホスト名（Name）: example.com
種類（Type）: MX
値（Value）: 10 inbound-smtp.us-east-1.amazonaws.com
TTL: 3600
```

#### 受信ルール(Receipt Rule)の設定
受信したメールに対してのアクションは`受信ルール(Receipt Rule)`で定義したものに従う。

受信ルールにより、受信したアドレスごとの制御`aaa@example.com`と`bbb@example.com`やS3に配置することやLambdaを呼び出して処理すると言ったアクションを定義することができる。

以下のCFnの例では`aaa@example.com`に対しての受信メールはS3バケットの`aaa_emails`に配信する設定をしている。
```yaml
EmailReceivingRuleSet:
    Type: AWS::SES::ReceiptRuleSet
    Properties:
        RuleSetName: "MyReceiptRuleSet"

EmailReceivingRule:
    Type: AWS::SES::ReceiptRule
    Properties:
        RuleSetName: !Ref EmailReceivingRuleSet
        Rule:
            Name: "EmailToS3"
            Enabled: true
            ScanEnabled: true
            TlsPolicy: Optional
            Recipients:
                - "aaa@example.com"
            Actions:
                - S3Action:
                    BucketName: !Ref EmailS3Bucket
                    ObjectKeyPrefix: "aaa_emails/"
```


#### 受信元の制限
SESでは特定のIPだけに受信元を制限するフィルター機能が存在する。
送信元のメールアドレスによる制限機能は存在しないため、実装したい場合はLambda側で設定を行う必要がある。

特定のIPだけに受信元を制限する場合は、受信ルールの設定を追加で行う。
```yaml
EmailReceivingRule:
    Type: AWS::SES::ReceiptRule
    Properties:
        ~~~~~
        Actions:
          - S3Action:
                BucketName: !Ref DefaultS3Bucket
                ObjectKeyPrefix: "aaa-emails/"
            - BounceAction:
                Message: "Unauthorized sender IP"
                Sender: "ses@example.com"
                SmtpReplyCode: "550"
        Condition:
            IpAddressFilter:
                Policy: Allow
                IpList:
                    - "192.0.2.1/32"
```
#### 受信に関する権限設定
SESについては、権限付与をするようなRole設定を行うことはできない。
そのため、SESの機能としてS3に配置を行う場合は、S3側のバケットポリシー側にSESからの操作の認可を与える必要がある。

さらに、SESからのアクセスを許可してしまうと、世界中のAWSからのアクセスが可能となってしまうので、制限を入れることを忘れない。

以下のバケットポリシー例では、principalとして`ses`を設定しているが、Conditon句でアカウントIDや受信ルールからのアクセスに限定している。SESの受信ルールがS3に操作をしようとしている関係になっている点に注意。
```yaml
    EmailS3BucketPolicy:
        Type: AWS::S3::BucketPolicy
        Properties:
            Bucket: !Ref EmailS3Bucket
            PolicyDocument:
                Version: "2012-10-17"
                Statement:
                    - Effect: Allow
                    Principal:
                        Service: ses.amazonaws.com
                    Action: "s3:PutObject"
                    Resource: !Sub "arn:aws:s3:::${EmailS3Bucket}/*"
                    Condition:
                        StringEquals:
                            aws:SourceAccount: "123456789012"
                            aws:SourceArn: !Sub "arn:aws:ses:us-east-1:123456789012:receipt-rule/MyReceiptRule"
```


### 送信実装
#### 送信APIの利用
送信については、いくつかの選択肢があるが、シンプルな構成でない限りは`SendRawEmail API`を利用する。
アプリケーション側でMIME規約に従ってメールを作成することで添付ファイルなども付与してメール送信が可能である。

#### 送信に関する権限設定
SESを経由してメールを送信する場合、アプリケーション側にSESに対する権限を付与する必要がある。
SendRawEmailの権限を付与すれば良い
```yaml
        Policies:
            - PolicyName: "SendEmailPolicy"
            PolicyDocument:
                Version: "2012-10-17"
                Statement:
                - Effect: Allow
                    Action:
                        - "ses:SendEmail"
                        - "ses:SendRawEmail"
                    Resource: "*"
```
なお、リソースの部分には、SESで検証したドメインのうち利用できるものを指定する。
`arn:aws:ses:us-east-1:123456789012:identity/example.com`

上記の設定をしておくと、メールのFromとして`@example.com`を利用することができる。

#### バウンス設定(SNS配信)
SESがメールを送信した際に、メールアドレスの不存在などでメールが送れないメールをバウンスメールと呼ぶ。

SESのNortificaion設定でバウンスメールが発生した場合のアクションを指定することができる。
そのため、バウンスメール設定でSNSのメールアドレスを指定しておくことで、メールの配信に失敗するとSNSに配信することができる。


#### 送信の信頼性を高めるSPF（TXTレコード）
#### 送信の信頼性を高めるDKIM（CNAMEレコード*3)

### DNS設定のまとめ
|設定目的|レコード種別|Name（ホスト名）|Value（設定値）|必須か？|
|:----|:----|:----|:----|:----|
|ドメイン所有権の証明|TXT (_amazonses)|_amazonses.example.com|SESが提供する検証コード|✅ 検証後削除OK|
|送信元認証（SPF）|TXT|example.com|v=spf1 include:amazonses.com -all|✅ 必須|
|送信メールの改ざん防止（DKIM）|CNAME ×3|randomstring1._domainkey.example.com</br>randomstring2._domainkey.example.com</br>randomstring3._domainkey.example.com|SESが提供するDKIMキー|✅ 必須|
|送信ポリシー（DMARC）|TXT (_dmarc)|_dmarc.example.com|v=DMARC1; p=none; rua=mailto:reports@example.com|✅ 推奨|
|受信メールの設定|MX|example.com|10 inbound-smtp.us-east-1.amazonaws.com|✅ 必須|
