# Roleについて
## ポリシーの種類
Roleでは、主に以下2つのポリシーを定義する
- 信頼ポリシー:Roleを引き受けることができるプリンシパルの定義(認証)
- 許可ポリシー:どのサービスにアクセスできるかの定義（認可）

### 信頼ポリシー
信頼ポリシーはRoleに付与され、対象のRoleを引き受けることができる対象を指定する。
RoleがAWSリソースに紐づけられることを前提にしており、Roleを誰に付与することができるかの定義をする必要があるために利用される。

Role引き受けるアクションは"AssumeRole"であるため、信頼ポリシーのActionは常に"AssumeRole"となる。


```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": [ "123456789012" ] // ← AWSアカウントを信頼
        "Service": [ "ec2.amazonaws.com" ] // ← EC2サービスを信頼
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### 許可ポリシー
 信頼ポリシーで設定されているプリンシパルがAssumeRoleAPIを実行することで一時的に認証情報が作成され、アクセスポリシーに定義されたサービスを利用できるようになる。
```json
{
  "Version": "2008-10-17",
  "Id": "__default_policy_ID",
  "Statement": [
    {
      "Sid": "",
      "Effect":"Allow",// ← 許可か拒否か
      "Action":"SQS:*",// ← アクションの内容
      "Resource": "arn:aws:sqs:ap-northeast-1:123456789012:MyQueue"// ← アクションを実行できるリソース
    }
  ]
}
```

## STSとAssumeRole
IAM Roleは「API権限を委譲する」機能である。
信頼ポリシーに記載された、principalに対して、許可ポリシーに記載した権限を一時的に委譲するという一連の動作を担うためである。

上記を実現する機能としてstsを利用したAssumeRole がある。 
AssumeRoleとは、stsに信頼できるリクエスト元を定義しておいて、信頼できることを確認できれば、一時的にRoleを与える機能である。
