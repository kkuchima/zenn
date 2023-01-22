---
title: "PodSecurityPolicy (PSP) からの移行"
emoji: "🦖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP, GoogleCloud, GKE]
publication_name: "google_cloud_jp"
published: false
---

# tl;dr
* GKE 1.25 以降で PodSecurityPolicy (PSP) が削除されるので、現在 PSP を利用されている方は移行が必要になります
* PSP の後継として PSA がありますが、PSP を完全にカバーしているわけではないので移行に工夫が必要です
* 

# なぜ PodSecurityPolicy の移行が必要なのか
[PodSecurityPolicy (以降 PSP)](https://kubernetes.io/docs/concepts/security/pod-security-policy/) は、Kubernetes クラスタに対するセキュリティポリシーを設定する機能です。Kubernetes 組み込みの Admission Controller として提供されています。PSP を活用することで、例えば特権コンテナや hostpath の利用など、Node 側に影響が出る可能性のある設定のコンテナのデプロイを防止できるようになります。

ではなぜ PSP から移行しないといけないかというと、PSP は Kubernetes 1.21 で deprecated ステータスとなっており、1.25 で Remove されてしまうからです。
これは Google Kubernetes Engine (GKE) を使っていても同様で、[GKE 1.25 以降では PSP が利用できなくなってしまう](https://cloud.google.com/kubernetes-engine/docs/deprecations/podsecuritypolicy)ため、1.24 以前の段階で PSP から他の機能へ移行する必要があります。

PSP が削除される背景について、既に多くの記事で述べられているため本記事では深くは触れませんが、ポイントとしては以下になります：
* 意図せず広範囲の権限を付与してしまいやすい
→ PSP はユーザーアカウント or サービスアカウントに対してポリシーを設定するが、
* どのポリシーが適用されているか分かりにくく混乱やエラーを引き起こしやすい
気になる方は以下の公式ブログや KEP も読んでみてください。
https://kubernetes.io/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/#why-is-podsecuritypolicy-going-away
https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/2579-psp-replacement

# PodSecurity Admission
[PodSecurity Admission](https://cloud.google.com/kubernetes-engine/docs/how-to/podsecurityadmission) (以降 PSA) は PSP の後継となる Kubernetes 組み込みの Admission Controller です。PSP よりも、よりシンプルに設定できるのが特徴になっています。
PSA では [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) という Kubernetes 側で整備している、セキュリティ標準に沿った構成を強制することができます。

Pod Security Standards では以下３種類のプロファイルが定義されています。

|プロファイル|概要|
|----|----|
|Privileged|制限の無いポリシー。プロファイルの指定が無い場合のデフォルト値。|
|Baseline|最低限の制限が設定されたポリシー。|
|Restricted|ベストプラクティスに沿った、最も厳しいポリシー|

また、PSP とは異なり各ポリシーが意図した挙動となっているかログ等から確認ができる(実際に Pod 作成等は拒否されない) モードが提供されています。
これにより、現在稼働している環境への影響を確認した上でポリシーの適用を検討することができます。
|モード|概要|
|----|----|
|Enforce | ポリシー違反により Pod の作成が拒否される。監査ログにイベントが追加される。|
|Audit| ポリシー違反により Pod の作成が**拒否されない**。監査ログにイベントが追加される。|
|Warn| ポリシー違反により Pod の作成が**拒否されない**。warning が表示される。|

## PSA の特徴
Kubernetes に組み込まれているため、基本的にメンテナンスは不要

Namespace の更新権限を持たせている場合は注意
Mutating の機能を持たない
細かなカスタマイズができない


## PSP と比べた PSA の制約

# PSA 以外の移行候補
OPA Gatekeeper (Policy Controller)
サポート付き、有償版の Policy Controller がある


Kyverno
CEL Admission
独自実装
(結論・提案) PSA + α が良いのでは？



# PSP から PSA への移行

差分の確認
Mutating しているか
Pod security standard に沿っているかどうかは GKE Security posture management で確認可能
warn / audit mode から始める
pspmigrator
PSP の無効化


# まとめ

# 参考資料
https://kubernetes.io/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/#why-is-podsecuritypolicy-going-away
https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/2579-psp-replacement
https://kubernetes.io/docs/tasks/configure-pod-container/migrate-from-psp/

https://github.com/open-policy-agent/gatekeeper-library/tree/master/library/pod-security-policy
https://static.sched.com/hosted_files/kccncna2022/ab/%5Bexport%5D%20Migrating%20from%20PodSecurityPolicy.pdf
https://cloud.google.com/kubernetes-engine/docs/how-to/migrate-podsecuritypolicy?hl=ja
https://kubernetes.io/docs/reference/access-authn-authz/psp-to-pod-security-standards/
https://github.com/kubernetes-sigs/pspmigrator


GKE deprecation insight
GKE security posture
