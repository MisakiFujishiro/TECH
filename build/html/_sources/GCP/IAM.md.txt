# IAM
GCPにおけるIAMとは、主に認可を司るサービスである。
![](../img/GCP/IAM/IAM.png) 
[まずは知っておくべき IAM の基礎と最新の便利機能](https://services.google.com/fh/files/events/0224-infra-onair-seesion-1.pdf)

## IAM Policy
どのロールを、どのメンバーに割り当てるかという割り当ての制御を行う。

## IAM Role
IAM Roleは、複数の権限の集合体であり、AWSのRoleとは役割が違う点に注意。
具体的な認可のアクションはIAM Roleにて記述される。

![](../img/GCP/IAM/IAM_Role.png) 
[まずは知っておくべき IAM の基礎と最新の便利機能](https://services.google.com/fh/files/events/0224-infra-onair-seesion-1.pdf)


![](../img/GCP/IAM/IAM_identity.png)
[まずは知っておくべき IAM の基礎と最新の便利機能](https://services.google.com/fh/files/events/0224-infra-onair-seesion-1.pdf)


## SA(Service Account)

