# カーネル

## カーネルパニック
カーネルパニックとは、オペレーティングシステムのカーネル（核となる部分）が重大なエラーを検出し、システムの通常の運用を続けられない場合に発生するシステムエラーのことです。カーネルパニックが発生すると、システムは安全な状態に戻すために再起動が必要になります。これは、データの破損や損失を防ぐための措置です。

## カーネルクラッシュ
カーネルのクラッシュとは、オペレーティングシステムのカーネルが実行中に重大なエラーに直面し、その結果システム全体が正常に動作しなくなる状態を指します。これはカーネルパニックと似ていますが、特にクラッシュという用語は、カーネルが予期せぬ状況に陥り、適切に処理できずに停止することを強調しています。

## kdump
kdumpは、Linuxカーネルで利用されるクラッシュダンプメカニズムの一つです。kdumpを利用することで、カーネルパニックやカーネルクラッシュが発生した際に、システムのメモリ内容をダンプファイルとして保存することができます。これにより、クラッシュの原因を後から調査することが可能になります。

### kexecを利用した再起動：  
kexecは、カーネルが再起動することなく新しいカーネルをロードして実行することを可能にする技術です。
kdumpでは、この機能を利用してクラッシュ時に再起動することなくクラッシュダンプ用の新しいカーネル（クラッシュカーネル）を起動します。
クラッシュカーネルは、現行のカーネルの状態を保存するために、システム全体が一時停止しする。この時点でプロセスやエージェントは停止される。

### クラッシュカーネルの設定：  
kdumpを有効にするためには、システムブート時に特別なクラッシュカーネル（crash kernel）を予約する必要があります。
このクラッシュカーネルは、メインカーネルがクラッシュした際に実行される軽量なカーネルです。
### クラッシュダンプの生成：
メインカーネルがクラッシュすると、kexecを利用してクラッシュカーネルが起動されます。
このクラッシュカーネルは、クラッシュしたメインカーネルのメモリ内容を取得し、指定されたストレージにダンプファイルとして保存します。


## dump
ダンプファイルは、システムのクラッシュ時にその時点でのメモリ内容、レジスタの状態、プロセス情報などを含むファイルです。これらのファイルは、システム管理者や開発者がクラッシュの原因を調査し、修正するために使用されます。




## 解析手順
### 前提
以下の手順でカーネルのクラッシュを引き起こすため、重要なデータがない環境で実施する

### 手順
kdumpのインストール
```
sudo yum install -y kexec-tools
```

kdumの有効化
```
sudo systemctl enable kdump
sudo systemctl start kdump
```

kdumpが動作したときに出力されるファイルの配置場所の設定
```
sudo cat /etc/kdump.conf | grep -v '^#'
```

クラッシュダンプ用のメモリを確保
```
sudo vi /etc/default/grub
```

ファイルの中身を以下のように編集
```
GRUB_CMDLINE_LINUX="... crashkernel=auto"
```

変更を反映させるために、GRUB設定ファイルを再生成
```
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

再起動させる
```
sudo reboot
```

復旧後に人為的にカーネルクラッシュを引き起こす
```
echo 1 | sudo tee /proc/sys/kernel/sysrq
echo c | sudo tee /proc/sysrq-trigger
```

kdumpファイルは以下のフォルダに作成される
```
sudo ls /var/crash/
```