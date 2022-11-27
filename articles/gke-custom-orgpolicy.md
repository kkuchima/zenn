---
title: "カスタム組織ポリシーで GKE クラスタの安全な設定を強制する"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP, GoogleCloud, GKE]
publication_name: "google_cloud_jp"
published: false
---
この記事は Google Cloud Japan Advent Calendar 2022 の 2 日目の記事です。カスタム組織ポリシー (Preview) を使って、GKE クラスタに安全な設定を強制できるようにしてみます。

# tl;dr
- 

# 組織ポリシーとは
元々組織ポリシーは事前に定義されたポリシー (制約) の中から自組織に合うものを選択し適用するものでしたが、カスタム組織ポリシーの登場により利用者側で独自にポリシーを作成したり自由にカスタマイズすることができるようになりました。

ただ現在は [GKE と Dataproc しかサポートされていない](https://cloud.google.com/resource-manager/docs/organization-policy/custom-constraint-supported-services)のでご注意ください。

# GKE のカスタムポリシー
Google Kubernetes Engine API v1 の Cluster または NodePool リソースの任意フィールドで、CREATE メソッドまたは UPDATE メソッドのカスタム制約を作成でき

Terraform や Config Controller のレイヤーで制約をかけることもできますが、GUI で直接触られた場合など CI を通さない変更に対しても強いのが組織ポリシーの特徴だと思います。

# 実際に試してみる
以前ブログ記事にした[構成](https://medium.com/google-cloud-jp/gkesecurity-2022-1-ea4d55bcf4f7)をベースに

- Regional
- プライベートクラスタ
- Workload Identity
- Shielded GKE
- COS
- asia-northeast1
- RBAC for Google Groups
- 検証環境だけ Spot VM

```yaml
name: organizations/ORGANIZATION_ID/customConstraints/custom.CONSTRAINT_NAME
resourceTypes:
- container.googleapis.com/RESOURCE_NAME
methodTypes:
- METHOD1
- METHOD2
condition: resource.OBJECT_NAME.FIELD_NAME == VALUE
actionType: ACTION
displayName: DISPLAY_NAME
description: DESCRIPTION
```
* `condition` で


1. カスタム制約を作成

```yaml:enableGkeAutopilot.yaml
name: organizations/ORGANIZATION_ID/customConstraints/custom.enableGkeAutopilot
resourceTypes:
- container.googleapis.com/Cluster
methodTypes:
- CREATE
condition: resource.autopilot.enabled == false
actionType: DENY
displayName: Enable GKE Autopilot
description: All new clusters must be Autopilot clusters.
```

```bash
gcloud org-policies set-custom-constraint ~/constraint-enable-autopilot.yaml
gcloud org-policies list-custom-constraints --organization=ORGANIZATION_ID
```

2. カスタム制約の適用
```yaml:policy-enable-autopilot.yaml
name: projects/PROJECT_ID/policies/custom.enableGkeAutopilot
spec:
  rules:
  - enforce: true
```

```bash

```

```yaml:enableWorkloadIdentity.yaml
name: organizations/ORGANIZATION_ID/customConstraints/custom.enableWorkloadIdentity
resourceTypes:
- container.googleapis.com/Cluster
methodTypes:
- CREATE
condition: has(resource.workloadIdentityConfig.workloadPool) || resource.workloadIdentityConfig.workloadPoolSize() > 0
actionType: ALLOW
displayName: Enable Workload Identity on new clusters
description: All new clusters must use Workload Identity.
```


# まとめ
まだ Preview の機能なので、対応しているサービスが少なかったり、GUI での操作ができなかったりという点はありますが、今後成熟してくると便利に使えるようになりそうです。



![aaa](/images/gke-orgpolicy/overview.png)

実践的なフォルダ構成

Tag を使った