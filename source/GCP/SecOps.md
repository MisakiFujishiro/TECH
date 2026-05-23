# SecOps
Google SecOpsとは、
セキュリティログ、脅威インテリジェンス、エンティティ情報、検出ルール、SOARを統合して、
脅威の検知・調査・対応を行うためのセキュリティ運用基盤である。

単なるログ検索基盤ではなく、
「大量のセキュリティデータを正規化し、コンテキストを付与し、リスクに基づいて優先順位付けする」
ことを目的としている。

Google SecOpsの中心思想は以下：
- ログをそのまま見るのではなく、UDMに正規化する
- イベント単体ではなく、Entityを中心に調査する
- IOCだけでなく、脅威インテリジェンスとの関係性を見る
- すべてのアラートを同じ重みで扱わず、risk scoreで優先順位を付ける
- 手作業ではなく、SOARで対応を自動化する

全体イメージ：

    各種ログ
      ↓
    パーサー
      ↓
    UDMに正規化
      ↓
    Detection / Threat Intelligence / UEBA
      ↓
    アラート
      ↓
    ケース調査
      ↓
    SOARで対応

## UDM

UDMとは、Unified Data Modelの略で、
Google SecOpsに取り込まれたログを共通形式で扱うためのデータモデルである。

製品ごとにログ形式は異なる。

例：
- Firewallの送信元IP
- Proxyのclient IP
- DNSのquery
- EDRのprocess情報

これらをそのまま扱うと、
製品ごとに検索方法が変わってしまう。

UDMでは、
異なるログを共通の意味に変換する。

例：
- principal.ip
- target.ip
- principal.user.userid
- target.hostname
- metadata.event_type
- network.dns.questions.name

### UDMの考え方

UDMは「ログ形式」ではなく、
「そのイベントが何を意味するか」
を表す。

| 概念 | 意味 |
|---|---|
| principal | 行為をした主体 |
| target | 行為の対象 |
| intermediary | 通信の中継点 |
| network | 通信情報 |
| process | 実行されたプロセス |
| file | ファイル情報 |
| metadata | イベント種別や製品情報 |

たとえば、ユーザーがサーバーへアクセスした場合：

- principal = アクセスしたユーザーや端末
- target = アクセスされたサーバー
- network = 通信情報
- metadata = イベント種別

UDMの重要ポイント：
- 製品依存のログを共通形式にする
- 複数ログを横断検索できる
- 検出ルールの土台になる
- Google SecOpsの中心概念である

## Entity

Entityとは、
セキュリティ調査における分析対象のことである。

例：
- ユーザー
- 端末
- IPアドレス
- ドメイン
- ファイルハッシュ
- サービスアカウント
- クラウドリソース

セキュリティ調査では、
単一ログだけでは全体像は分からない。

たとえば：

    ユーザー
      ↓
    端末へログイン
      ↓
    不審プロセス実行
      ↓
    外部IPへ通信
      ↓
    悪性ドメインへDNS問い合わせ

このように攻撃は複数イベントにまたがる。

そのためGoogle SecOpsでは、
イベント単体ではなくEntityを中心に調査する。

### Entity Graph

Entity Graphとは、
ユーザー、端末、IP、ドメイン、ハッシュなどの関係性を表す仕組みである。

例：

    user@example.com
      ↓ 使用
    host-001
      ↓ 実行
    powershell.exe
      ↓ 通信
    malicious.example.com

Entity Graphの目的：
- 関連イベントをつなげる
- 攻撃の流れを把握する
- 過去の振る舞いを見る
- リスク分析に利用する

## Threat Intelligence

Threat Intelligenceとは、
既知の攻撃者、マルウェア、悪性IP、悪性ドメイン、攻撃インフラなどに関する脅威情報である。

Google SecOpsでは、
Google Threat IntelligenceやApplied Threat Intelligenceを利用して、
ログ内の通信先やハッシュを既知脅威と照合できる。

### IOC

IOCとは、Indicator of Compromiseの略で、
侵害の痕跡を示す情報である。

例：
- 悪性IP
- 悪性ドメイン
- URL
- ファイルハッシュ
- メールアドレス

IOCは、
「この値が出てきたら怪しい」
という点の情報である。

イメージ：

    ログ内のIP
      ↓
    Threat Intelligenceと照合
      ↓
    既知C2サーバーと一致
      ↓
    アラート

ただしIOCには限界がある。

攻撃者はIPやドメインを変更できるため、
IOC一致だけでは不十分な場合がある。

### Fusion Feed

Fusion Feedとは、
Google Applied Threat Intelligenceが提供する脅威関連データである。

単一IOCだけではなく、
APTグループや攻撃インフラ同士の関係性を保持している。

例：

    APT41
      ↓
    関連ドメイン
      ↓
    関連IP
      ↓
    関連マルウェア
      ↓
    関連インフラ

重要なのは、
「単一の悪性IP」
ではなく、
「攻撃グループ全体」
を見ようとしている点である。

### Curated Detection

Curated Detectionとは、
Googleが管理する事前定義済み検出ルールである。

Google側が、
脅威インテリジェンスや攻撃手法をもとに検出ロジックを管理する。

| 種類 | 意味 |
|---|---|
| IOC | 脅威を示す値そのもの |
| Curated Detection | Google管理の検出ルール |
| Custom Rule | ユーザー作成の検出ルール |

イメージ：

- IOC = 材料
- Curated Detection = Googleが作った検出レシピ
- Custom Rule = 自分で作る検出レシピ

## YARA-L

YARA-Lとは、
Google SecOpsで検出ルールや検索を記述するためのルール言語である。

UDMに正規化されたイベントを対象に、
条件を指定して脅威を検出する。

YARA-Lでできること：
- 単一イベント検出
- 複数イベント相関
- 一定時間内集計
- Entity利用
- Risk Score付与
- Threat Intelligence利用

### outcomes

outcomesとは、
YARA-Lルールで検出結果に付与する値を定義する部分である。

たとえば：
- risk_score
- severity
- metadata
- Entity情報

を保存できる。

流れ：

    検出条件一致
      ↓
    outcomesでrisk_score設定
      ↓
    アラートへ付与
      ↓
    後続分析で利用

重要なのは、
Google SecOpsでは
「検出したか」
だけではなく、
「どれくらい危険か」
を扱う点である。

## Risk Score

Risk Scoreとは、
Entityや検出結果に対して危険度を数値化する考え方である。

SOCでは大量のアラートが発生するため、
すべてを同じ優先度で見ることはできない。

そのため、
以下のような情報をもとに優先順位を付ける。

- Threat Intelligence一致
- 重要アセットか
- 過去にも不審活動があるか
- 通常と異なる行動か
- 複数検出が集中しているか

イメージ：

    単発の低リスクイベント
      ↓
    低優先度

    重要サーバー
      ＋
    悪性IOC
      ＋
    異常行動
      ↓
    高優先度

Risk Scoreは、
Alert Fatigueを減らすための考え方でもある。

## UEBA

UEBAとは、
User and Entity Behavior Analyticsの略で、
ユーザーやEntityの通常行動をもとに異常を検出する仕組みである。

従来の検出は、
「既知の悪いもの」
を見る。

UEBAは、
「普段と違うか」
を見る。

例：
- 普段より大量DL
- 普段使わない国からログイン
- 普段使わない端末
- 深夜大量API実行
- 初めて見るプロセス実行

流れ：

    通常行動
      ↓
    ベースライン化
      ↓
    現在行動と比較
      ↓
    異常なら検出

UEBAが重要な理由：
- 攻撃者が正規アカウントを使う
- ログイン成功だけでは不正か分からない
- IOCに一致しない攻撃を検出したい

## SOAR

SOARとは、
Security Orchestration, Automation and Responseの略で、
セキュリティ対応を自動化する仕組みである。

Google SecOps SOARでは、
アラートやケースに対してPlaybookを実行できる。

### Playbook

Playbookとは、
セキュリティ対応手順を自動化したワークフローである。

例：

    アラート発生
      ↓
    Threat Intelligence照合
      ↓
    重大度判定
      ↓
    チケット作成
      ↓
    Slack通知
      ↓
    必要ならIPブロック

Playbookの目的：
- 手作業削減
- 対応品質標準化
- 初動高速化
- SOC負荷削減

### Connector

Connectorとは、
外部システムからアラートやデータを取り込むための仕組みである。

例：
- EDR
- SIEM
- メールセキュリティ
- チケットシステム
- クラウドサービス

Connectorは、
外部データをGoogle SecOps SOARで扱える形式へ変換する。

## SCC

Security Command Centerとは、
Google Cloud環境のセキュリティ状態を管理するサービスである。

Google SecOpsが
運用・検知・調査を担うのに対し、
SCCはクラウド環境のセキュリティポスチャ管理を担う。

### Posture

Postureとは、
セキュリティ上の「姿勢」や「状態」を意味する。

例：
- 公開されてはいけないリソースが公開されていないか
- IAM権限が過剰ではないか
- 脆弱な設定がないか
- 監査ログが有効か
- 暗号化機能が有効か

### Security Health Analytics

Security Health Analyticsとは、
Google Cloudリソースの設定不備や脆弱構成を検出するSCC機能である。

例：
- 公開Cloud Storage
- 過剰IAM権限
- 暗号化設定不備
- Confidential Computing無効
- Firewall設定不備

重要なのは、
Findingを「ミュートする」のではなく、
根本原因を修復することが本来の目的である点である。

## Detection と Prevention

セキュリティ対策には大きく2種類ある。

| 種類 | 意味 | 例 |
|---|---|---|
| Detection | 起きたことを検出 | アラート、UEBA、ログ分析 |
| Prevention | 起きる前に防ぐ | IAM、Org Policy、VPC SC |

Google Cloudセキュリティでは、
この2つを組み合わせることが重要である。

流れ：

    Prevention
      ↓
    危険操作を防ぐ

    Detection
      ↓
    異常を見つける

    Response
      ↓
    SOARで対応

## Google SecOps 全体像

    ログソース
      ├ Firewall
      ├ Proxy
      ├ DNS
      ├ EDR
      ├ Cloud Audit Logs
      └ IAM Logs

              ↓

    Google SecOps
      ├ UDM
      ├ Entity Graph
      ├ Threat Intelligence
      ├ YARA-L
      ├ UEBA
      ├ Risk Score
      └ SOAR

              ↓

    SOC運用
      ├ 検知
      ├ 調査
      ├ 優先順位付け
      ├ 対応
      └ 再発防止

## Google SecOpsで覚えるべき思想

### 1. ログではなくUDMで見る

製品ログではなく、
共通意味モデルで見る。

### 2. イベントではなくEntityで見る

単発イベントではなく、
関係性を見る。

### 3. IOCだけでなくContextで見る

悪性IP一致だけではなく、
状況を見る。

### 4. 検出だけでなくRiskで見る

アラート有無ではなく、
危険度を見る。

### 5. 手作業ではなくSOARで対応する

通知、封じ込め、調査を自動化する。

### 6. カスタム実装よりManaged機能を優先する

Curated Detection
UEBA
SCC
Security Health Analytics
など、
Google提供機能を優先利用する。

## 用語関係まとめ

| 用語 | 役割 |
|---|---|
| UDM | ログ共通化 |
| Entity | 調査対象 |
| Entity Graph | Entity関係分析 |
| IOC | 既知侵害指標 |
| Threat Intelligence | 脅威情報 |
| Curated Detection | Google管理検出 |
| YARA-L | 検出ルール言語 |
| outcomes | 検出結果付与情報 |
| risk_score | 危険度 |
| UEBA | 行動異常分析 |
| SOAR | 対応自動化 |
| Playbook | 自動対応フロー |
| SCC | クラウドセキュリティ管理 |
| SHA | 設定不備検出 |

## まとめ

Google SecOpsは、
単なるログ検索サービスではない。

大量ログをUDMで正規化し、
Entity Graphで関係性を把握し、
Threat IntelligenceやUEBAで文脈を付与し、
Risk Scoreで優先順位付けし、
SOARで対応を自動化するための基盤である。

中心となる流れ：

    UDM
      ↓
    Entity
      ↓
    Context
      ↓
    Detection
      ↓
    Risk
      ↓
    Response
      ↓
    Automation

この流れを理解すると、
各機能が「なぜ存在するのか」を思い出しやすくなる。
