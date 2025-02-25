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

## 基本的なコマンド
### 汎用
#### printf(print formatted)
フォーマットを指定しながら、出力を成形することができる。

`%`を利用して、フォーマット指定子を明示する。例えば、`%.2f`は浮動小数2桁を意味する。
```
printf "%.2f %.2f\n" 3.14159 2.71828
```
とすると以下の結果が出力される(\nはコマンドライン上の見栄えのため)
```
3.14 2.72
```

#### xargs(extended arguments)
標準入力から受け取ったデータを引数として別のコマンドに渡すためのコマンド。

パイプ(|)で繋いで、コマンドの出力結果を次のコマンドへ連続して渡すことができるが、出力された結果は標準出力となるため、引数として渡されるわけではない。
例えば、以下のfindの出力結果は"txt"形式のファイルを一覧として出力する。一方でrmは引数としてファイル名を受け取る必要があるため、xargsがないと失敗する。
```
find . -name "*.txt" | xargs rm
```



引数がどう渡されるか分かりにくい場合は以下のように、-Iを利用すると引数の受け渡し場所がわかりやすい
```
find . -name "*.log" | xargs -I {} mv {} /backup/logs/
```


### プロセス系
#### lsof(list open files)
"list open files" の略で、システム上の開かれているファイルの情報を表示するためのコマンド。Unix系OSでのファイル使用状況の監視に役立つ。

```
lsof /var/log/xxx.log
```

#### ps(process status)
psコマンドは、システムで実行中のプロセスに関する情報を表示することが可能。
PIDを指定して、psコマンド実行すると、システム上で動作中のプロセスの情報を取得するために作られたコマンド

以下のコマンドでは、特定のプロセスに関する情報を取得する"-p"や前情報を出力する"-f"のオプションを付与している。
```
ps -fp PID
```

#### kill
killコマンドで、PIDを指定すると、強制的にプロセスを殺すことができる
```
kill PID
```

### ファイル操作系
#### grep(global regular expression print)
正規表現を用いたテキスト検索のために作られたコマンド。

"-i"を利用すると、大文字小文字を無視して検索してくれる
```
grep -i "error" server.log
```

"-r"を利用すると、対象のフォルダを再起的にサブディレクトリまで含めて検索してくれる。  "-n"を利用すると、ヒットした行番号まで表示してくれる。  
以下の例は、"error"という文字列を大文字小文字関係なくルートディレクトリは以下のファイルを再起的に検索して、ヒットした行数を表示して、エラーは/dev/nullに書き込む。
```
sudo grep -inr "error" / 2>/dev/null
```

#### sed(Stream Edito)
テキストストリームを編集するために作られたコマンド。
ファイルの置換やテキスト抽出などが可能


#### find
ファイルシステム内で特定の名前や条件に合致するファイルやディレクトリを探したい。

"-name"で検索したいファイル名を指定する。
```
sudo find / -name "knockd.conf" 2>/dev/null
```

#### awk(由来は人名)
テキストデータを処理・変換・抽出したいとき、特定のフィールドの値を抽出したり、条件に基づいて行をフィルタリングしたい場合に使用

以下の例では、file.csvに対して、-Fで指定した文字を区切り文字として扱い、中括弧の中に含まれるコマンドを実行する。つまり、","を区切り文字として、2列目の要素を出力する。
```
awk -F, '{print $2}' file.csv
```
また、条件指定をすることもできる
```
awk '/ERROR/' logfile.txt
```

### ネットワーク系
#### netstat(network statistics)
自分のネットワーク接続の状況、ポートの使用状況、ルーティングテーブルなどを確認したい場合に使用。
- "-n"でIPやポートを数値で表示
- "-t"でTCP通信
- "-l"でリッスンしているポートを表示することができる
- "-p"で関連するプロセスを表示
```
netstat -pntl
```
結果の例。TCP/UDP接続状況や接続可能なAddress（localのみか外部まで可能など）まで知ることができる。
```
Proto Recv-Q Send-Q Local Address      Foreign Address    State
tcp   0      0      0.0.0.0:22         0.0.0.0:*          LISTEN
tcp   0      0      127.0.0.1:3306     0.0.0.0:*          LISTEN
```


#### nmap(Network Mapper)
ネットワークの利用状況などを確認したい場合に使用。開いているポートなどを知ることができる。
出力結果が比較的見やすいので、localhostを対象にしても良いと思う。-pを指定すると指定したポートをスキャンしてくれるが-p-とすると全てのポートをスキャンしてくれる。
```
nmap -p- localhost
```
結果の例。基本的にはPORTとその状態、利用されるサービスなどが確認できる
```
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
443/tcp   open     https
```


### ストレージ系
#### df
"disk filesystem" の略で、ディスクファイルシステムに関する情報を表示するためのコマンド。
ストレージの容量不足などを確認できる。

"-h"は人間が読みやすい形で出力させる
```
df -h
```
出力例
```
Filesystem     Type     Size   Used  Avail Use% Mounted on
/dev/sda1      ext4      50G    25G    25G   50% /
tmpfs          tmpfs    2.0G   0      2.0G   0%  /dev/shm
/dev/sdb1      ext4     100G   75G    25G   75% /mnt/data
```

#### du
duは"disk usage"の略で、ディスク上のファイルやディレクトリが使用している容量を確認するために作られたコマンド

"-sh"は人間が読みやすい形でサマリ表示してくれる
```
sudo du -sh --max-depth=1 /opt/pgdata
```
出力例
```
4.0K    /opt/pgdata/backup
12G     /opt/pgdata/main
512M    /opt/pgdata/logs
```