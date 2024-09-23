# S3のオブジェクト管理
S3におけるオブジェクト管理の基本的な考えはKVS(Key-Value Store)である。
例えば、以下のような階層構造があってもS3としてはKeyにフルパスが管理されているだけ。
```
(ルート)
 └ foo/
    └ bar.txt
```

|キー（フルパス名）|バリュー（ファイルの内容）|
|:----|:----|
|foo/bar.txt|bar|


## S3におけるフォルダの扱い
S3においてはフォルダという概念は正確にはない。
解説した通り、S3においては基本的に、/に特別な意味はなくKVS管理のKeyの一部に過ぎない。
GUIとして/ごとにフォルダを作成しているだけ

S3で以下の2つで実はオブジェクトの管理方法が異なる
1. 空フォルダを作成してからその中にファイル配置
2. コマンドでフォルダとファイルを一気に作成

### 空フォルダを作成してからその中にファイル配置
空フォルダを作成すると、ファイルサイズ0のオブジェクトが作成される
```
(ルート)
 └ foo/
```

|キー（フルパス名）|バリュー（ファイルの内容）|
|:----|:----|
|foo/|-|

ここにファイルを配置すると、一見一つのファイルしかないが、2つのオブジェクトとして管理される
```
(ルート)
 └ foo/
    └ bar.txt
```

|キー（フルパス名）|バリュー（ファイルの内容）|
|:----|:----|
|foo/|-|
|foo/bar.txt|bar|

### コマンドでフォルダとファイルを一気に作成
AWS CLIのコマンドを利用して"foo/bar.txt"を作成すると、フォルダ構成としては上記と同じになるが、オブジェクト管理としては一つしか管理されない

|キー（フルパス名）|バリュー（ファイルの内容）|
|:----|:----|
|foo/bar.txt|bar|

因みに、この状態から、bar.txtを削除すると、foo/のオブジェクトが管理される

|キー（フルパス名）|バリュー（ファイルの内容）|
|:----|:----|
|foo/|-|