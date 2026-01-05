
# Google Cloud APIs
Google Cloud APIsは、GCPが提供するAPI群が定義されている。
- [Google Cloud APIS](https://cloud.google.com/apis/docs/overview?hl=ja)

Google Cloud APIsは、gRPCをベースにして設計されている。  
`gRPC`は、googleが開発した`Protocol Buffers`をベースとした、軽量・高速な`RPC(Remote Procedure Call)`フレームワークであり、通信の効率性と型安全性を重視した設計となっている。

## Protobuf(Protocol Buffers)
ProtobufはGoogleが開発したデータ定義のフレームワークであり、`「APIで利用するデータ構造」と「そのインターフェース（RPC）」を宣言的に定義するフォーマット`。

このフォーマットの最大の特徴は、単なるデータ形式（例：JSONやYAML）の代替ではなく、`定義されたスキーマ（.protoファイル）がなければ開発を始められないという点`。

つまり、スキーマが仕様書であり、かつコードの出発点でもあるため、ドキュメント管理と開発実装が一体化されている。

まず、APIのインターフェースを宣言的に定義しているという特徴は、具体的に.protoファイルの例をみると理解しやすい。
```
syntax = "proto3";

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
}

message GetUserRequest {
  string user_id = 1;
}

message User {
  string name = 1;
  int32 age = 2;
}
```
上記の.protoにより、関数名・引数・戻り値などのAPI定義、データ構造について明確かつ機械可読に定義されている。
- どんなAPIがあるか → GetUserというRPCがある
- どんなリクエストを送るか → user_idを送る
- どんなレスポンスが返るか → nameとageが入ったUser構造が返る

Protobufで定義されたスキーマがなければ開発を始められないという特徴は、gRPCがそのスキーマからStubコードを自動生成し、それを基にAPI実装やクライアント開発を行う設計になっていることに起因する。
この仕組みの詳細は、後述のgRPCセクションで解説する。

最後にRESTと比較すると、一層Protobufの特徴である、APIを定義したドキュメントをベースに開発が始まる点が理解しやすい。

|項目|REST（+Swagger/OpenAPI）|gRPC（+Protobuf）|
|:----|:----|:----|
|APIの定義|任意（後からでもOK）|必須（ないと通信できない）|
|定義の役割|人が読むためのドキュメント|コード生成・通信の契約|
|実装との関係|実装とズレがち|実装も定義から生成される|




## RPC
RPC(Remote Procedure Call)とは、離れた場所（リモート）にある関数を、ローカルの関数のように呼び出す仕組み。
通信処理の実装を抽象化し、開発者はシンプルな関数呼び出しの形でAPIを利用できる。


## gRPC
Googleが開発したRPC通信フレームワークであり、Protobufで定義されたAPI仕様に基づいて、Stubコードを自動生成して双方向通信を行う。

開発者は .proto ファイルでサービスとメッセージの定義を行い、それをコンパイルして得られるStubコードを使ってAPIを呼び出す。
.protoファイルを protoc コマンドでコンパイルすると、次のようなStubファイルが生成される。使用言語（Python、Go、Javaなど）に応じたコードを自動生成できる。）

|ファイル名|内容|
|:----|:----|
|user_pb2.py|メッセージ構造（User, GetUserRequest）などを定義|
|user_pb2_grpc.py|gRPCのクライアント/サーバーStubコードを定義|


gRPCのAPIの開発においては、この生成されたファイルをimportして、開発を行う。すなわち、.protoファイルが前提となって開発が始まる。
以下は、生成されたStubファイルをPythonでimportして使うことで UserService に対して GetUser を呼び出す例：

```python
import grpc
from user_pb2 import GetUserRequest
from user_pb2_grpc import UserServiceStub

channel = grpc.insecure_channel('localhost:50051')
stub = UserServiceStub(channel)

response = stub.GetUser(GetUserRequest(user_id="123"))
print(response.name)
```

このコードでは、stub.GetUser(...) が関数呼び出しのように見えるが、実際には gRPC を通じてネットワーク越しのリモートAPIを呼び出している。


## Google Cloud SDKにおける gRPC と REST の切り替え
Google Cloud SDK のクライアントライブラリには、主に2種類のAPI層が存在しており、開発者は transport オプションを指定することで、gRPC通信かREST通信かを選択することができる。

|レイヤー|概要|例|
|:----|:----|:----|
|1. High-level Client (手軽API)|REST/gRPC両対応。独自の使いやすいラッパー|pubsub_v1.PublisherClient|
|2. Low-level gRPC Client|Protobuf/gRPCベースの生のStub|publisher_pb2_grpc.PublisherServiceStub|

.proto ファイルに以下のような記述を追加することで、gRPCのメソッドをRESTエンドポイントとしても利用可能になる。この仕組みは gRPC Transcoding と呼ばれる。

```
rpc GetUser(GetUserRequest) returns (User) {
  option (google.api.http) = {
    get: "/v1/users/{user_id}"
  };
}
```

このように、同じAPIメソッドに対して gRPC でも REST でもアクセスできるようになっているため、Google Cloudのクライアントライブラリでは、通信方式（transport）を明示的に選択できる

|通信方法|呼び出し例|
|:----|:----|
|gRPC|stub.GetUser(request)|
|REST|GET /v1/users/123|


Google Cloudのクライアントライブラリでは、transportの方式を明示的に選択することができる：
```python
from google.cloud import pubsub_v1

# gRPCベース
client = pubsub_v1.PublisherClient(transport="grpc")

# RESTベース（内部的にはHTTP/1.1 + JSON）
client_rest = pubsub_v1.PublisherClient(transport="rest")
```

デフォルトはサービスによって異なるが、性能重視ならgRPC、環境制約やデバッグ性重視ならRESTを選択することが多い。

## フィーチャーフラグ
gRPC では、次のような変更が特にリスクになりやすい。
- レスポンス構造の変更
- 処理ロジックの差し替え
- 新旧ロジックの共存期間が必要な変更

REST のように URL を増やすのではなく、同じメソッドの中で振る舞いを変えたいケースが多くなる。

ここで登場するのが フィーチャーフラグ。
**フィーチャーフラグ（Feature Flag）**とは、gRPC サービスの実装内部で、処理の振る舞いを条件付きで切り替える仕組み である。
gRPC におけるフィーチャーフラグは、gRPC そのものの機能ではなくアプリケーション設計・運用のための仕組みとして使われる。
```
gRPCリクエスト
  ↓
ユーザー / テナント / 環境などの属性を取得
  ↓
feature_flag を評価
  ↓
新旧どちらのロジックを実行するか決定
```
