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

### 参考サイト
- 同じRoomを完了させている人：[TryHackMeでハッキングを勉強してみた話](https://zenn.dev/ci/articles/aa15d88b06bbed#%E5%8F%82%E8%80%83%E3%81%AB%E3%81%95%E3%81%9B%E3%81%A6%E3%81%84%E3%81%9F%E3%81%A0%E3%81%84%E3%81%9F%E6%83%85%E5%A0%B1%E7%BE%A4)

## Day3
