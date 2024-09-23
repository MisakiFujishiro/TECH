# S3に対する操作
## List
lsコマンドによりS3に対する一覧表示が可能だが、オプションによりその対象が大きく異なる点に注意

何も指定しないとS3バケット全てを一覧表示する
```
aws s3 ls
```

バケットを指定すると、バケット内のオブジェクトを一覧表示する
```
aws s3 ls {BucketName}
```

## upload
S3へのアップロードは以下のコマンドで実行できる

uploadする時はアップロード元とアップロード先を指定するようにする。
```sh
aws s3 cp [FROM] [TO] 
```

### マルチパートアップロード
S3へのファイルアップロードはその容量によってアップロード方法が異なる。目安は5G。
ファイルをアップロードする際には5G以上の場合はマルチパートアップロードが必要になる。

ただし、aws s3コマンドは内部でファイルサイズに応じてマルチパートアップロードを実施してくれるため、意識せずに処理を行うことができる。
[公式ドキュメント](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-services-s3-commands.html)の記載は以下
```
aws s3 コマンドを使用して大きなオブジェクトを Amazon S3 バケットにアップロードする場合、 AWS CLI は自動的にマルチパートアップロードを実行します。
```