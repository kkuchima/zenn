---
title: "PodSecurityPolicy (PSP) からの移行"
emoji: "🦖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP, GoogleCloud, GKE]
publication_name: "google_cloud_jp"
published: false
---
本記事では GKE において PSP からの移行先としてどのようなオプションがあるか、また PSP からその後継である PSA への移行ステップや移行に役立つツールのご紹介など、公式ドキュメントの補足となる情報をお伝えします。
# tl;dr
* GKE 1.25 以降で PodSecurityPolicy (PSP) が削除されるので、現在 PSP を利用されている方は移行が必要になります
* PSP の後継として PSA がありますが、PSP を完全にカバーしているわけではないので移行に工夫が必要です

# なぜ PodSecurityPolicy の移行が必要なのか
[PodSecurityPolicy (以降 PSP)](https://kubernetes.io/docs/concepts/security/pod-security-policy/) は、Kubernetes クラスタに対するセキュリティポリシーを設定する機能です。Kubernetes 組み込みの Admission Controller として提供されています。PSP を活用することで、例えば特権コンテナや hostpath の利用など、Node 側に影響が出る可能性のある設定のコンテナのデプロイを防止できるようになります。

ではなぜ PSP から移行しないといけないかというと、PSP は Kubernetes 1.21 で deprecated ステータスとなっており、1.25 で Remove されてしまうからです。
これは Google Kubernetes Engine (GKE) を使っていても同様で、[GKE 1.25 以降では PSP が利用できなくなってしまう](https://cloud.google.com/kubernetes-engine/docs/deprecations/podsecuritypolicy)ため、1.24 以前の段階で PSP から他の機能へ移行する必要があります。

PSP が削除される背景について、既に多くの記事で述べられているため本記事では深くは触れませんが、ポイントとしては以下になります：
* 意図せず広範囲の権限を付与してしまいやすい
* どのポリシーが適用されているか分かりにくく混乱やエラーを引き起こしやすい
PSP はユーザーアカウントやサービスアカウントに対して利用可能なポリシーを設定しますが、設定が直感的ではなくポリシーの適用範囲がわかりにくいとよく言われてきました。
また、PSP には Dry-run mode も無いため、ポリシー適用の影響を事前に確認し辛いというのもネックです。

気になる方は以下の公式ブログや KEP も読んでみてください。
https://kubernetes.io/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/#why-is-podsecuritypolicy-going-away
https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/2579-psp-replacement

# PodSecurity Admission
## PodSecurity Admission の概要
PSP が無くなってしまうことはわかったのですが、では 1.25 以降では PSP 同様の要件に対してどう対処したら良いのかという問題がでてきます。そこで [PodSecurity Admission](https://cloud.google.com/kubernetes-engine/docs/how-to/podsecurityadmission) の登場です。
PodSecurity Admission (以降 PSA) は PSP の後継となる Kubernetes 組み込みの Admission Controller です。PSP よりも、ポリシーをシンプルに設定できるのが特徴になっています。
PSA では [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) という Kubernetes プロジェクトで整備している、セキュリティ標準に沿った構成をクラスタ内ワークロードに強制することができます。

まず、Pod Security Standards では以下 3 種類のセキュリティレベルの異なるプロファイルが定義されています。
|プロファイル|概要|
|----|----|
|Privileged|制限の無いポリシー。プロファイルの指定が無い場合のデフォルト値。|
|Baseline|最低限の制限が設定されたポリシー。|
|Restricted|ベストプラクティスに沿った、最も厳しいポリシー|

PSA では上記 Pod Security Standards で定義された 3 種類のプロファイルを Namespace 単位で設定することができます。例えば特権コンテナが稼働するシステム系の Namespace は `Privileged` を設定し、アプリケーション Pod が稼働する Namespace では `Restricted` を設定するなどの使い方が可能です。 (現状、Pod 単位など Namespace より細かい単位でプロファイルを設定することはできません)

また、PSA では PSP とは異なり各ポリシーが意図した挙動となっているかログ等から確認ができる (実際に Pod 作成等は拒否されない) モードが提供されています。
これにより、実際のポリシー適用前に現在稼働している環境への影響を確認することができます。ステージング環境等で `Audit` モードで運用し、本番環境に適用する前にポリシーの影響を確認するという使い方もできます。
|モード|概要|
|----|----|
|Enforce|ポリシー違反により Pod の作成が拒否される。監査ログにイベントが追加される。|
|Audit|ポリシー違反により Pod の作成が**拒否されない**。監査ログにイベントが追加される。|
|Warn|ポリシー違反により Pod の作成が**拒否されない**。warning が表示されるが、監査ログにイベントが追加されない。|

実際に PSA を設定する場合ですが、対象の Namespace に設定対象となるモードとプロファイルを `pod-security.kubernetes.io/<MODE>=<PROFILE>` の形式で label として追記します。Label は各モードごとに複数設定することもできます。
```bash
# Restricted 基準で Warning を出力する設定
$ kubectl label --overwrite ns default pod-security.kubernetes.io/warn=restricted

$ kubectl describe ns default
Name:         default
Labels:       kubernetes.io/metadata.name=default
              pod-security.kubernetes.io/warn=restricted
Annotations:  <none>
Status:       Active

No resource quota.

# Restricted に違反する Pod をデプロイする（デプロイ自体は成功する）
$ kubectl create deployment nginx --image=nginx
Warning: would violate PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/nginx created
```

## PSP との違い
前述の通り、PSA では事前定義されたプロファイルを各 Namespace に設定するという方式をとっていて、 PSP に無かったシンプルさや Dry-run による影響範囲のわかりやすさを持っていますが、あらかじめ定義された 3 種類のポリシーしか使えず、PSP のようなカスタマイズ性が失われています。
また、他にも PSP との大きな違いとして、PSP ではサポートしていた Mutation を PSA ではサポートしていないというのもあります。
PSP と Pod Security Standards の細かな違いについては以下のドキュメントもご参照ください：
https://kubernetes.io/docs/reference/access-authn-authz/psp-to-pod-security-standards/

また、PSA では Namespace の Label に基づいてポリシーが設定されるため、アプリケーション開発者の方などが Namespace の変更権限を持っている場合は意図せず(or 意図的に) ポリシーが変更される可能性がある点に注意が必要です。そのような場合は PSA 設定前に Namespace に対する権限を制限する

# PSP からの移行

0. Decide whether Pod Security Admission is the right fit for your use case.
1. Review namespace permissions
2. Simplify & standardize PodSecurityPolicies
3. Update namespaces
3-1. Identify an appropriate Pod Security level
3-2. Verify the Pod Security level
3-3. Enforce the Pod Security level
3-4. Bypass PodSecurityPolicy
4. Review namespace creation processes
5. Disable PodSecurityPolicy

# PSA 以外の移行候補
OPA Gatekeeper (Policy Controller)
サポート付き、有償版の Policy Controller がある
Rego によりポリシーを定義

PSA はKubernetes に組み込まれているため、基本的にメンテナンス不要
基となる Pod Security Standard


Kyverno
yaml でポリシーを定義。validation/mutation をサポート

CEL Admission
独自実装
(結論・提案) PSA + α が良いのでは？

Istio (istio-proxy, init) との組み合わせ
Warn と Dry run の違い


# PSP から PSA への移行

PSA には Dry-run 機能があり、既存の Pod に対するポリシーの影響を確認することができます。これにより、PSA に移行した際の影響範囲を確認することができます。
warn に設定していたとしても既存 Pod のポリシー違反は検知されないため、実際にPSA を適用する前に Dry-run mode を試していただくのをお勧めします。★Todo: 要確認


差分の確認
Mutating しているか
Pod security standard に沿っているかどうかは GKE Security posture management で確認可能
warn / audit mode から始める
pspmigrator
PSP の無効化

PSP 1.24(Beta) と 1.25 (Stable) での違い
https://kubernetes.io/blog/2022/08/25/pod-security-admission-stable/
- Improved violation messages
- Improved namespace warnings
- Changes to the Pod Security Standards
Seccomp - The seccompProfile.type field for Pod and container security contexts
Privilege escalation - The allowPrivilegeEscalation field on container security contexts
Capabilities - The requirement to drop ALL capabilities in the capabilities field on containers


## pspmigrator

```bash
go install github.com/kubernetes-sigs/pspmigrator/cmd/pspmigrator
```

```bash
$ pspmigrator migrate
Checking if any pods are being mutated by a PSP object
The table below shows the pods that were mutated by a PSP object
+------------------------------------+-----------+------------+
|              POD NAME              | NAMESPACE |    PSP     |
+------------------------------------+-----------+------------+
| nginx-8f458dc5b-g86sk              | default   | psp-sample |
| nginx-unprivileged-bf9c4b944-552pq | default   | psp-sample |
+------------------------------------+-----------+------------+
There were 2 pods mutated. Please modify the PodSpec such that PSP no longer needs to mutate your pod.
You can run `pspmigrator mutating pod nginx-8f458dc5b-g86sk -n default` to learn more why and how your pod is being mutated. Please re-run the tool again after you've modified your PodSpecs.
```

```bash
$ pspmigrator mutating pod nginx-8f458dc5b-g86sk -n default
Pod nginx-8f458dc5b-g86sk is mutated by PSP psp-sample: true, diff: [slice[0]: <nil pointer> != v1.SecurityContext]
PSP profile psp-sample has the following mutating fields: [RunAsUser] and annotations: []
```

```bash
$ pspmigrator mutating pods
There are 5 pods in the cluster
+--------------------------------------+--------------+---------+----------------+
|                 NAME                 |  NAMESPACE   | MUTATED |      PSP       |
+--------------------------------------+--------------+---------+----------------+
| nginx-8f458dc5b-g86sk                | default      | true    | psp-sample     |
| nginx-unprivileged-bf9c4b944-552pq   | default      | true    | psp-sample     |
| istio-egressgateway-688d4797cd-zx5mz | istio-system | false   | gce.privileged |
| istio-ingressgateway-6bd9cfd8-mzml5  | istio-system | false   | gce.privileged |
| istiod-68fdb87f7-bdzzh               | istio-system | false   | gce.privileged |
+--------------------------------------+--------------+---------+----------------+
```


# Todo: 検証内容
・事前に違反 Pod をデプロイしておく、enforce: baseline, audit/warn: restricted
 -> Dry run mode を実行する
 -> 
・1つのラベルのとき
・1.24 で PSP -> PSA 移行
・1.24 でPSP有効化して、1.25 にそのままアップグレードする
・Istio 環境(not CNI)、Istio なしの環境
-> CNI 使いましょう？

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

複数の PodSecurityPolicy が使用可能な場合、アドミッション コントローラは、検証に成功した最初のポリシーを使用します。ポリシーはアルファベット順に並べられ、コントローラは、変更ポリシーよりも非変更ポリシー（ポッドを変更しないポリシー）を優先します。