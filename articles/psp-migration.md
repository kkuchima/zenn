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

# なぜ PSP の移行が必要なのか
PSP とは

PSP は Kubernetes 1.25 で Remove される


# PodSecurity Admission (PSA)

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
