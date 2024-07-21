# サーバー


## OS
代表的なOSの系統は「Unix」「Linux」「Windows」3つ
### Unix
1960年に開発された古いOSで、後発に影響を与えたOSで、macOSはUnixの系譜。
### Linux
Unixに影響を受けたオープンソースのOS。様々なディストリビューションが存在。
具体的には、Ubuntu・CentOS・RedHatEnterpriseLinux(RHEL)などがある。
### WindowsOS
マイクロソフトが開発したOSでWindowsServerやWindows10などで利用されている・

## Linuxのディストリビューション
Linuxには様々なディストリビューションが存在する。
### Ubuntu
ユーザーフレンドリーで利用しやすいのが特徴。
Windowsのような使いやすいインターフェースがある。
### RedHatEnterpriseLinux
商用サポートが保証されているエンタープライズ版のLinuxディストリビューション。
### CentOS
RHELのクローンで、商用サポートなしの無料版。その代わりコミュニティーサポートが充実している。
### Amazon Linux
AWSのEC2で利用することができるAmazonLinuxは正式にはどのLinuxディストリビューションをベースにしているか発表されていないが、RHEL、CentOSをベースにしているらしい。


## Linux系のファイル
サーバーに配置される代表的なディレクトリとその役割を以下にまとめる。
'yum install'でソフトウェアパッケージのインストールをすると、以下のディレクトリに必要なファイルが配置される。

| ディレクトリ | 由来 | 代表的な内容 |
|--------------|------|-------------|
| /            | ファイルシステムの最上位ディレクトリ | 全てのファイルとディレクトリのルート |
| /bin         | "Binaries"の略 | システムの基本的なコマンド（例: `ls`, `cp`, `mv`, `rm`, `cat`） |
| /boot        | ブートローダとカーネルのイメージ | ブートに必要なファイル（例: `vmlinuz`, `initrd.img`, `grub`） |
| /dev         | "Devices"の略 | デバイスファイル（例: `sda`, `tty`, `null`） |
| /etc         | "Editable Text Configuration"の略 | システム設定ファイル（例: `passwd`, `fstab`, `ssh/sshd_config`） |
| /home        | ユーザーのホームディレクトリ | 各ユーザーの個人データ（例: `/home/user1`, `/home/user2`） |
| /lib         | "Libraries"の略 | 基本的な共有ライブラリ（例: `libc.so.6`, `ld-linux.so.2`） |
| /media       | リムーバブルメディアのマウントポイント | リムーバブルメディア（例: `/media/cdrom`, `/media/usb`） |
| /mnt         | "Mount"の略 | 一時的なマウントポイント（例: `/mnt/mydisk`） |
| /opt         | "Optional"の略 | サードパーティのソフトウェア（例: `/opt/lampp`） |
| /proc        | "Processes"の略 | カーネルとプロセスの情報（例: `/proc/cpuinfo`, `/proc/meminfo`） |
| /root        | rootユーザーのホームディレクトリ | システム管理者専用ディレクトリ（例: `/root`） |
| /run         | システムの一時ファイル | 一時ファイル（例: `/run/lock`, `/run/shm`） |
| /sbin        | "System Binaries"の略 | システム管理コマンド（例: `ifconfig`, `shutdown`, `reboot`） |
| /srv         | "Service"の略 | サーバー関連のデータ（例: `/srv/ftp`, `/srv/www`） |
| /sys         | システムデバイスの情報 | カーネルとデバイスの情報（例: `/sys/class/net`） |
| /tmp         | "Temporary"の略 | 一時ファイル（例: アプリケーションの一時ファイル） |
| /usr         | "Unix System Resources"の略 | ユーザープログラムとデータ（例: `/usr/bin`, `/usr/lib`, `/usr/share`） |
| /var         | "Variable"の略 | 変動するデータ（例: `/var/log`, `/var/spool`） |



## サービスとデーモン
サーバー内で動作する特定の機能を提供するプログラムやプロセスをサービスと呼ぶ。
例えばWebサーバー（Apache/Nginx）やDBサーバー（MySQL/Postgres）、メールサーバー。
これらのサービスはシステムの中でユーザーや他のサービスに対して特定の機能を提供している。

ユーザーが操作しないバックグラウンドでこれらのサービスが動作しているときにデーモンと呼ぶ。
ギリシャ神話のデーモンが、目に見えないところで様々な影響を与えているところから名前がつけられている。
デーモンは、システムの起動時に自動で開始され、システム停止時には自動で終了する。

### コマンド
システム起動時に自動的に開始するように有効化されているサービス（デーモン）を確認する
```
sudo systemctl list-unit-files --type=service --state=enabled
```

現在実行中のサービスを確認する
```
sudo systemctl list-units --type=service --state=running
```

特定のサービスの状態を確認する
```
sudo systemctl status <サービス名>
```

### デーモンの代表例
有効化されている代表的なデーモンの例は以下

| デーモン名                          | 役割                                                   |
|-------------------------------------|--------------------------------------------------------|
| auditd.service                      | Linuxの監査デーモン。システムのセキュリティ監査を行う。 |
| chronyd.service                     | 時刻同期デーモン。NTP(Network Time Protocol)を使用してシステム時間を同期。 |
| crond.service                       | cronデーモン。スケジュールされたタスクを実行する。      |
| dbus.service                        | D-Busメッセージバスシステム。プロセス間通信を提供。      |
| firewalld.service                   | 動的なファイアウォールデーモン。ネットワーク接続の制御。 |
| irqbalance.service                  | IRQバランスデーモン。システムの割り込み負荷を分散。    |
| iscsid.service                      | iSCSIデーモン。iSCSIターゲットに接続する。              |
| kdump.service                       | kdumpカーネルクラッシュダンプメカニズムのデーモン。    |
| network.service                     | レガシーなネットワークスクリプトのデーモン。          |
| NetworkManager.service              | ネットワーク管理デーモン。ネットワークの設定と管理。  |
| postfix.service                     | Postfixメール転送エージェントのデーモン。メールの送受信。 |
| rhel-dmesg.service                  | カーネルメッセージをログに記録するデーモン。            |
| rhsmcertd.service                   | Red Hat Subscription Manager証明書デーモン。サブスクリプション証明書の管理。 |
| rsyslog.service                     | ロギングデーモン。システムログを収集し、ログファイルに書き込む。 |
| sshd.service                        | OpenSSHデーモン。SSHによるリモートログインを提供。      |
| systemd-journald.service            | systemdのログデーモン。ログメッセージを収集。           |
| systemd-logind.service              | ユーザーログイン管理デーモン。ユーザーセッションを管理。|
| systemd-udevd.service               | デバイスイベントデーモン。デバイスのイベントを処理。   |
| tuned.service                       | パフォーマンス調整デーモン。システムパフォーマンスを最適化。|