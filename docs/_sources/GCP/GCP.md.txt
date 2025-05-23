# GCPについて
GCPは、Googleが提供するクラウドコンピューティングプラットフォーム。
GCPを使うことで、Google社内で使われているものと同じテクノロジーやサービスを利用することができる。

GCPはAWS同様柔軟性や拡張性、高速なインフラをクラウドから利用できる特徴があるとともに、データ分析やAIという分野において強みがある。

## ドキュメント
- [Google Cloud ドキュメント](https://cloud.google.com/docs?hl=ja)
- [コンソールログイン](https://cloud.google.com/?hl=ja)
- [プロダクト別 参考セッション](https://www.gc-solution-design-pattern.jp/product-sessions)

### 勉強中ドキュメント
- [[実践編] Google Cloud をゼロから学ぶならこれ! おすすめの学習リソース](https://zenn.dev/google_cloud_jp/articles/15e9a147deabc9)

## インフラ
- [https://cloud.google.com/about/locations?hl=ja#regions](https://cloud.google.com/about/locations?hl=ja#regions)


## 他のクラウドとの比較
### リソース・ユーザーの管理
GPCユーザーやリソースの管理がAWSとは異なる。

AdminConsoleでユーザーを作成。CloudConsoleでプロジェクト内のリソースを操作する形。
AWSではアカウント単位でリソース境界があるが、GCPではアカウントに紐造りソースはなく、プロジェクト単位で分けられている。

![](../img/GCP/GCP/GCP_hikaku.png) 
[Google Cloud でクラウド二刀流エンジニアを目指すための「早わかり集中技術講座」](https://lp.cloudplatformonline.com/rs/808-GJW-314/images/GC-two-way-cloud_Session3.pdf)

![](../img/GCP/GCP/GCP_kousei.png)
[Google Cloud でクラウド二刀流エンジニアを目指すための「早わかり集中技術講座」](https://lp.cloudplatformonline.com/rs/808-GJW-314/images/GC-two-way-cloud_Session3.pdf)

独立したユーザーが、参画するPJのリソースを操作している。
![](../img/GCP/GCP/GPC_login.png)
[Google Cloud でクラウド二刀流エンジニアを目指すための「早わかり集中技術講座」](https://lp.cloudplatformonline.com/rs/808-GJW-314/images/GC-two-way-cloud_Session3.pdf)

GCPにおけるリソースの階層構造。
![](../img/GCP/GCP/GCP_kaisou.png)
[まずは知っておくべき IAM の基礎と最新の便利機能](https://services.google.com/fh/files/events/0224-infra-onair-seesion-1.pdf)