# Shell


## 基本的な文法
### 変数
変数の呼び出しは"$"でできる
```sh
# 変数の定義
name="John Doe"
# 変数の使用
echo "Hello, $name"
```

### 配列
配列の定義はすみカッコではなく通常の括弧で行う。
```sh
# 配列の定義
files=("file1.txt" "file2.txt" "file3.txt")
```


### 条件分岐
if・elif・elseとthenでブロックを作って条件に一致した処理を流すことができる。
```sh
if [ "$name" = "John Doe" ]; then
  echo "Name is John Doe"
elif ["$name" = "Bob" ]; then
    echo "Name is Bob"
else
  echo "Name is not John Doe"
fi
```

#### 比較演算子
数字や文字、ファイルを扱う際の演算子について特徴的なので注意。

数値比較演算子

|演算子|説明|例|
|:----|:----|:----|
|-eq|等しい（equal）|[ "$a" -eq "$b" ] は $a が $b と等しい場合に真|
|-ne|等しくない（not equal）|[ "$a" -ne "$b" ] は $a が $b と等しくない場合に真|
|-lt|より小さい（less than）|[ "$a" -lt "$b" ] は $a が $b より小さい場合に真|
|-le|以下（less than or equal to）|[ "$a" -le "$b" ] は $a が $b 以下の場合に真|
|-gt|より大きい（greater than）|[ "$a" -gt "$b" ] は $a が $b より大きい場合に真|
|-ge|以上（greater than or equal to）|[ "$a" -ge "$b" ] は $a が $b 以上の場合に真|


文字列比較演算子

|演算子|説明|例|
|:----|:----|:----|
|=|等しい（equal）|[ "$str1" = "$str2" ] は $str1 が $str2 と等しい場合に真|
|!=|等しくない（not equal）|[ "$str1" != "$str2" ] は $str1 が $str2 と等しくない場合に真|
|-z|文字列が空である（zero length）|[ -z "$str" ] は $str が空文字列の場合に真|
|-n|文字列が空でない（not empty）|[ -n "$str" ] は $str が空でない場合に真|


ファイル比較演算子

|演算子|説明|例|
|:----|:----|:----|
|-e|ファイルが存在する（exists）|[ -e "$file" ] は $file が存在する場合に真|
|-f|ファイルが存在し通常のファイルである（file）|[ -f "$file" ] は $file が通常のファイルの場合に真|
|-d|ディレクトリが存在する（directory）|[ -d "$directory" ] は $directory がディレクトリの場合に真|
|-r|読み取り可能である（readable）|[ -r "$file" ] は $file が読み取り可能な場合に真|
|-w|書き込み可能である（writable）|[ -w "$file" ] は $file が書き込み可能な場合に真|
|-x|実行可能である（executable）|[ -x "$file" ] は $file が実行可能な場合に真|
|-s|ファイルのサイズがゼロより大きい（size）|[ -s "$file" ] は $file のサイズがゼロより大きい場合に真|



複合条件

|演算子|説明|例|
|:----|:----|:----|
|&&|両方の条件が真である（AND）|[ "$a" -eq 1 ] && [ "$b" -eq 2 ]|
|｜｜|いずれかの条件が真である（OR）|[ "$a" -eq 1 ] ｜｜ [ "$b" -eq 2 ]|
|!|条件が偽である（NOT）|[ ! -f "$file" ] は $file が存在しない場合に真|


### ループ
for文やwhile文が利用できる
```sh
for i in 1 2 3 4 5; do
  echo "Number: $i"
done

# while ループ
count=1
while [ $count -le 5 ]; do
  echo "Count: $count"
  count=$((count + 1))
done
```

ディレクトリの中身をループさせるときは、ディレクトリを直接指定して要素取り出しが可能
```sh
# for loop
for file in /path/to/directory/*; do
  echo "Processing $file"
done
```

配列をループさせるときは、[@]を追記する必要がある
```sh
# 配列の定義
files=("file1.txt" "file2.txt" "file3.txt")

# 配列の使用
for file in "${files[@]}"; do
  echo "Processing $file"
done
```


### 関数
```sh
# 関数の定義
greet() {
  echo "Hello, $1"
}

# 関数の呼び出し
greet "Alice"
```


### エラーハンドリング
shellにおいては、エラーが発生しても次のコマンドを実行しようとする。
そのため、エラーが発生した際にその時点でコマンド実行を停止させるためには、以下のコマンドをセットしておく
```sh
# Exit immediately if a command exits with a non-zero status.
set -e
# Treat unset variables as an error when substituting.
set -u
```

"set -e"と設定しておくことで、コマンド処理が失敗した際にスクリプト全体を終了させる。
```sh
#!/bin/bash
set -e

# このコマンドがエラーを返した場合、スクリプトはここで終了します
cp /path/to/nonexistent/file /path/to/destination

# ここには到達しません
echo "This will not be printed if the previous command fails"
```

"set -u"と設定しておくことで、未設定の変数を使用した際にエラーを発生させることができる。
```sh
#!/bin/bash
set -u

# この行はエラーを引き起こします（未定義の変数を参照）
echo $UNDEFINED_VARIABLE

# ここには到達しません
echo "This will not be printed if the previous command fails"
```

ifを利用したtry-catchも可能である。以下の例ではcpが失敗すると、thenの処理に流れて "exit 1"でスクリプトが終了する。
```sh
#!/bin/bash
set -e
set -u

SOURCE_FILE="/path/to/source"
DEST_FILE="/path/to/destination"

if ! cp "$SOURCE_FILE" "$DEST_FILE"; then
  echo "Failed to copy $SOURCE_FILE to $DEST_FILE" >&2
  exit 1
fi

echo "File copied successfully"
```

