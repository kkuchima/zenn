---
title: "カスタム組織ポリシーで GKE クラスタの安全な設定を強制する"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP, GoogleCloud, GKE]
publication_name: "google_cloud_jp"
published: false
---
この記事は [Google Cloud Japan Advent Calendar 2022](https://zenn.dev/google_cloud_jp/articles/12bd83cd5b3370) の 2 日目の記事です。Preview で利用可能になった組織ポリシーの[カスタム制約](https://cloud.google.com/resource-manager/docs/organization-policy/creating-managing-custom-constraints)を使って、GKE クラスタに安全な設定を強制するよう構成してみます。

# tl;dr
* 組織ポリシーのカスタム制約機能により、利用者による独自の制約を設定することができるようになりました (2022.12 現在 Preview です)
* カスタム制約を活用し、GKE のクラスタ構成に関する柔軟かつ強固な制約を組織全体や特定フォルダ配下のプロジェクトに適用することができます

# 組織ポリシーとは
[組織ポリシー](https://cloud.google.com/resource-manager/docs/organization-policy/overview)は Google Cloud の組織やフォルダ配下、特定プロジェクトに対して制約を設定することができる機能です。組織全体や特定フォルダに対してポリシーを設定すると、その配下のプロジェクトに自動的にポリシーが継承されるため、ガバナンスを効かせることができます。

元々組織ポリシーは事前に定義された制約の中から自組織に合うものを選択し適用するものでしたが、本記事で紹介するカスタム制約の登場により**利用者側でも独自にポリシーを作成し柔軟な制約を設定**できるようになりました。  

Google Cloud リソースに対する柔軟な制約は Terraform ([Conftest](https://github.com/open-policy-agent/conftest) 等の利用) や [Config Controller](https://cloud.google.com/anthos-config-management/docs/concepts/config-controller-overview) のレイヤーでかけることもできますが、組織ポリシーを使うことで **gcloud や Cloud Console で直接触られた場合など CI を通さない変更に対しても強制力を効かせる**ことができます。

ただし 2022.12 現在、カスタム制約は Preview ステータスであり、また [GKE と Dataproc しかサポートしていない](https://cloud.google.com/resource-manager/docs/organization-policy/custom-constraint-supported-services)のでご注意ください。

# カスタム制約
カスタム制約のフォーマットは以下のようになっています。
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
* `name` でカスタム制約の名前を設定 (接頭辞を抜いて最大 100 文字) 
* `resourceTypes` で制約をかける対象のリソースを指定 (`Cluster` や `NodePool`) 
* `methodTypes` で制約をかける対象のメソッドを指定 (`CREATE` または `UPDATE`) 
* `condition` で制約の条件を [CEL (Common Expression Language)](https://cloud.google.com/resource-manager/docs/organization-policy/creating-managing-custom-constraints#common_expression_language) で設定
* `actionType` で制約が条件に合致した際のアクションを定義 (`ALLOW` または `DENY`) 

GKE クラスタに対するカスタム制約は Google Kubernetes Engine API v1 の `Cluster` または `NodePool` リソースの任意フィールドに対する `CREATE` or `UPDATE` メソッドに対して設定することができます。各種 API の仕様については以下ドキュメントをご参照ください。
https://cloud.google.com/kubernetes-engine/docs/reference/rest/v1beta1/projects.locations.clusters
https://cloud.google.com/kubernetes-engine/docs/reference/rest/v1/projects.locations.clusters.nodePools

# 実際に試してみる
では以下 ３ パターンを例に実際に試してみようと思います。
![試す構成](/images/gke-orgpolicy/overview.png)

**① 本番環境ではアルファクラスタのデプロイを禁止する**
[アルファクラスタ](https://cloud.google.com/kubernetes-engine/docs/concepts/alpha-clusters) は Kubernetes のアルファ版機能をお試しいただけるクラスタです。アルファクラスタは検証用途のクラスタとなっており SLA 無しかつアップグレード不可、またクラスタ自体の有効期限も 30 日となっていますので本番環境での利用は非推奨となっています。  
開発環境でアルファクラスタを使ってもらう分にはいいけど、本番環境に間違ってデプロイしてほしくないというケースもあるかと思いますのでカスタム制約を作って防いでみます。

**② 本番環境ではセキュアな構成の GKE クラスタを強制する**
本番環境ではなるべくセキュアな構成のクラスタしか動かしたくないということもあると思います。今回は 1 例として以下のような構成のクラスタのみデプロイさせるような制約を設定してみます。
* [プライベートクラスタ](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters) ([Private Endpoint](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters#private_cp))
* [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)
* [Shielded GKE Node](https://cloud.google.com/kubernetes-engine/docs/how-to/shielded-gke-nodes)
* [COS Node Pool](https://cloud.google.com/container-optimized-os/docs/concepts/security)

本記事では上記の各オプション・機能の説明は割愛しますが、気になる方は以下のブログ記事も読んでみてください。
https://medium.com/google-cloud-jp/gkesecurity-2022-1-ea4d55bcf4f7

**③ 開発環境では Spot VM のみ許可する**
(安全な構成という趣旨から逸れますが）コスト削減のために開発環境は [Spot VM](https://cloud.google.com/kubernetes-engine/docs/concepts/spot-vms) のみ利用を許可したいというケースもあるかもしれないので、そういうパターンも試してみます。

これからご紹介する制約はあくまでデモ用に用意したものなので、もし試す場合は検証用の組織など本番ワークロードに影響がでない環境でお試しください。(あとツッコミどころなどあれば教えてください)

## ① 本番環境ではアルファクラスタのデプロイを禁止する
まずは本番環境ではアルファクラスタが作成できないようにカスタム制約を設定していきます。

### 1. カスタム制約の作成
はじめに、設定したい内容の制約を書いていきます。Kubernetes Engine API の仕様を確認したところ今回防ぎたいアルファクラスタの設定は [Cluster](https://cloud.google.com/kubernetes-engine/docs/reference/rest/v1/projects.locations.clusters) リソース内で設定するようなので、`resourceTypes` は `container.googleapis.com/Cluster` を指定します。  
アルファクラスタの新規作成を防ぎたいため、`methodTypes` には `CREATE` を指定しています。  
制約の条件を設定する `condition` には `resource.enableKubernetesAlpha == true` を設定し、条件に合致すると拒否するように `actionType` を `DENY` に設定します。
```yaml:constraint-disableAlphaClusters.yaml
name: organizations/ORGANIZATION_ID/customConstraints/custom.disableAlphaClusters
resourceTypes:
- container.googleapis.com/Cluster
methodTypes:
- CREATE
condition: resource.enableKubernetesAlpha == true
actionType: DENY
displayName: Disable Alpha Clusters
description: All new GKE clusters must have Alpha Clusters disabled
```

続いて、カスタム制約を作成をします。 `gcloud org-policies set-custom-constraint` コマンドにより、先ほど用意した yaml ファイルを使って組織内にカスタム制約を作成します。作成したカスタム制約は `gcloud org-policies list-custom-constraints` や Cloud Console 上から確認することができます。
```bash
$ export ORGANIZATION_ID=<Organization ID>
$ gcloud org-policies set-custom-constraint constraint-disableAlphaClusters.yaml
Created custom constraint [organizations/ORGANIZATION_ID/customConstraints/custom.disableAlphaClusters].
$ gcloud org-policies list-custom-constraints --organization=$ORGANIZATION_ID
CUSTOM_CONSTRAINT            ACTION_TYPE  METHOD_TYPES  RESOURCE_TYPES                    DISPLAY_NAME
custom.disableAlphaClusters  DENY         CREATE        container.googleapis.com/Cluster  Disable Alpha Clusters
```
![Cloud Console 上からの確認](/images/gke-orgpolicy/disableAlphaClusters.png)

### 2. カスタム制約の適用
この制約は本番環境だけに適用したいので、作成したカスタム制約 `disableAlphaClusters` を `prod` フォルダに適用します。これにより、 `prod` フォルダ配下に作成されるプロジェクト全てに対して `disableAlphaClusters` が適用されるようになります。  
せっかくなので Cloud Console 上から適用してみます。
![Cloud Console 上からの適用](/images/gke-orgpolicy/apply-disableAlphaClusters.png)

### 3. アルファクラスタを作成してみる
では挙動を確認していきます。アルファクラスタでは自動アップグレードができないため Release Channel からは外し、また自動アップグレード・自動修復を無効にした上で `--enable-kubernetes-alpha` を付けてクラスタを作成してみます。

```bash
$ export PROJECT_ID=<PROD PROJECT ID>
$ gcloud config set project ${PROJECT_ID}
$ export CLUSTER_LOCATION=asia-northeast1-b
$ export CLUSTER_NAME=alpha-prod
$ gcloud container clusters create ${CLUSTER_NAME} \
    --zone ${CLUSTER_LOCATION} \
    --num-nodes=2 \
    --cluster-version "1.23.12-gke.100" \
    --release-channel "None" \
    --no-enable-autoupgrade \
    --no-enable-autorepair \
    --enable-kubernetes-alpha
ERROR: (gcloud.container.clusters.create) ResponseError: code=400, message=Operation denied by custom org policy: ["customConstraints/custom.disableAlphaClusters": "All new GKE clusters must have Alpha Clusters disabled"].
```
想定通りエラーが出力しクラスタ作成に失敗しました。これで本番環境ではカスタム制約によりアルファクラスタが有効になっているクラスタは作れないようになりました。ただ、`dev` フォルダ配下にある開発環境ではアルファクラスタの作成はできるはずです。念の為そちらも試してみます。

```bash
$ export PROJECT_ID=<DEV PROJECT ID>
$ gcloud config set project ${PROJECT_ID}
$ export CLUSTER_NAME=alpha-dev
$ gcloud container clusters create ${CLUSTER_NAME} \
    --zone ${CLUSTER_LOCATION} \
    --num-nodes=2 \
    --cluster-version "1.23.12-gke.100" \
    --release-channel "None" \
    --no-enable-autoupgrade \
    --no-enable-autorepair \
    --enable-kubernetes-alpha
Creating cluster alpha-dev in asia-northeast1-b... Cluster is being health-checked (master is healthy)...done.
Created [https://container.googleapis.com/v1/projects/kuchima-adventcal2022-dev/zones/asia-northeast1-b/clusters/alpha-dev].
```
無事クラスタが作成できました。これでフォルダを使った組織ポリシー適用の挙動が確認できました。アルファクラスタを無効にしたクラスタが `prod` 配下で作成できるかは、次のパターン`② 本番環境ではセキュアな構成の GKE クラスタを強制する`であわせて確認していきます。

## ② 本番環境ではセキュアな構成の GKE クラスタを強制する
本番環境ではなるべくセキュアな構成のクラスタしか動かしたくないということで、1 例として以下の構成のクラスタのみデプロイさせるような制約を設定してみます。
* プライベートクラスタ (Private Endpoint)
* Workload Identity
* Shielded GKE Node
* COS Node Pool

上記は割とよくある設定なので、制約を書き始める前にまず既存の組織ポリシーでカバーができないか考えてみることにします。既存のポリシーは[公式ドキュメント](https://cloud.google.com/resource-manager/docs/organization-policy/org-policy-constraints)や Cloud Console 上から確認ができるので見てみましょう。  

確認したところ、プライベートクラスタに関しては `constraints/compute.vmExternalIpAccess` という既存のポリシーで Node への外部 IP アドレスの付与を防いでくれそうなので活用できそうです。ただし Privtate Endpoint を強制してくれるわけでは無いのでそこはカスタムで作成が必要そうです。  
また、Shielded GKE Node については `constraints/compute.requireShieldedVm` という既存のポリシーで事足りそうです。
その他の Workload Identity や COS Node の利用を強制する既存ポリシーはなさそうなので、以下３種類のカスタム制約を作成してみます。
* Privtate Endpoint を強制する制約
* Workload Identity を強制する制約
* COS Node の利用を強制する制約

### 1. カスタム制約の作成
まずは Private Endpoint の有効化を強制する制約を作成します。Private Endpoint は `Cluster` リソースで設定しています。今回は `resource.privateClusterConfig.enablePrivateEndpoint` が `true` になっているかをチェックするような制約を書きます。
```yaml:constraint-enablePrivateEndpoint.yaml
name: organizations/ORGANIZATION_ID/customConstraints/custom.enablePrivateEndpoint
resource_types: container.googleapis.com/Cluster
method_types:
 - CREATE
 - UPDATE
condition: resource.privateClusterConfig.enablePrivateEndpoint == true
action_type: ALLOW
display_name: Enable GKE Private Endpoint
description: Only allow GKE Cluster resource create or update if Private Endpoint is enabled
```

続いて Workload Identity の有効化を強制する制約です。Workload Identity の有効・無効を判断するために `resource.workloadIdentityConfig.workloadPool` の有無をチェックしています。
```yaml:constraint-enableWorkloadIdentity.yaml
name: organizations/ORGANIZATION_ID/customConstraints/custom.enableWorkloadIdentity
resourceTypes:
- container.googleapis.com/Cluster
methodTypes:
- CREATE
condition: has(resource.workloadIdentityConfig.workloadPool) || resource.workloadIdentityConfig.workloadPool.size() > 0
actionType: ALLOW
displayName: Enable Workload Identity on new clusters
description: All new clusters must use Workload Identity.
```

最後に COS Node のチェックです。Node の設定になるため、`NodePool` リソースの制約を設定します。具体的には `resource.config.imageType` が COS (containerd) かどうかを確認しています。
```yaml:constraint-requireCOSNode.yaml
name: organizations/ORGANIZATION_ID/customConstraints/custom.requireCOSNode
resourceTypes:
- container.googleapis.com/NodePool
methodTypes:
- CREATE
condition: resource.config.imageType == "COS_CONTAINERD"
actionType: ALLOW
displayName: Require COS Nodes
description: All cluster nodes must be COS.
```

マニフェストが作成できたら `gcloud org-policies set-custom-constraint` でカスタム制約を組織内に作成します。
```bash
$ gcloud org-policies set-custom-constraint constraint-enablePrivateEndpoint.yaml
Created custom constraint [organizations/ORGANIZATION_ID/customConstraints/custom.enablePrivateEndpoint].

$ gcloud org-policies set-custom-constraint constraint-enableWorkloadIdentity.yaml
Created custom constraint [organizations/ORGANIZATION_ID/customConstraints/custom.enableWorkloadIdentity].

$ gcloud org-policies set-custom-constraint constraint-requireCOSNode.yaml
Created custom constraint [organizations/ORGANIZATION_ID/customConstraints/custom.useCOSNode].

$ gcloud org-policies list-custom-constraints --organization=$ORGANIZATION_ID
CUSTOM_CONSTRAINT              ACTION_TYPE  METHOD_TYPES   RESOURCE_TYPES                     DISPLAY_NAME
custom.disableAlphaClusters    DENY         CREATE         container.googleapis.com/Cluster   Disable Alpha Clusters
custom.enablePrivateEndpoint   ALLOW        CREATE,UPDATE  container.googleapis.com/Cluster   Enable GKE Private Endpoint
custom.enableWorkloadIdentity  ALLOW        CREATE         container.googleapis.com/Cluster   Enable Workload Identity on new clusters
custom.requireCOSNode          ALLOW        CREATE         container.googleapis.com/NodePool  Require COS Nodes
```

### 2. カスタム制約の適用
作成したカスタム制約を Prod フォルダに適用します。先ほどのように Console からポチポチ変更しても良いのですが、せっかくなので gcloud で適用します。  
以下のようにポリシー用のファイルを用意し、`name` に適用対象のフォルダ ID と制約を指定し、`enforce` を `true` に設定します。
```yaml:policy-enablePrivateEndpoint.yaml
name: folders/PROD_FOLDER_ID/policies/custom.enablePrivateEndpoint
spec:
  rules:
  - enforce: true
```

`gcloud org-policies set-policy` コマンドで先ほどのポリシーファイルを指定し、ポリシーを適用します。ちゃんとポリシーが設定されたかどうかは `gcloud org-policies list` で確認可能です。
```bash
$ export FOLDER_ID=<PROD_FOLDER_ID>
$ gcloud org-policies set-policy policy-enablePrivateEndpoint.yaml
Created policy [folders/PROD_FOLDER_ID/policies/custom.enablePrivateEndpoint].

$ gcloud org-policies list --folder=$FOLDER_ID
CONSTRAINT                    LIST_POLICY  BOOLEAN_POLICY  ETAG
custom.disableAlphaClusters   -            SET             CI/yn5wGEKiulp8C
custom.enablePrivateEndpoint  -            SET             COf0oZwGEMiyte4B
```

残りの制約も同様に適用します。既存のポリシー (`compute.requireShieldedVm`, `compute.vmExternalIpAccess`) と今回新規に作成した制約の適用が完了すると以下のような状態になっているかと思います。
```bash
$ gcloud org-policies list --folder=$FOLDER_ID
CONSTRAINT                     LIST_POLICY  BOOLEAN_POLICY  ETAG
compute.requireShieldedVm      -            SET             CKT7oZwGELicwPIB
compute.vmExternalIpAccess     SET          -               CIb7oZwGEMiQstQC
custom.disableAlphaClusters    -            SET             CI/yn5wGEKiulp8C
custom.enablePrivateEndpoint   -            SET             COf0oZwGEMiyte4B
custom.enableWorkloadIdentity  -            SET             CMn7oZwGEID0pO8B
custom.requireCOSNode          -            SET             COT7oZwGEIidtMsC
```

### 3. 制約の挙動を確認する
それでは実際に設定した制約が想定通りに動いているか確認してみます。とはいえ今回は時間(文字数)の都合上、網羅的な確認ではなくピンポイントで挙動を確認していきます。
まずは失敗するであろうパターンとして `プライベートクラスタだけどパブリックエンドポイント` かつ `Workload Identity が設定されていない`クラスタをデプロイしてみます。
```bash
$ export PROJECT_ID=<PROD PROJECT ID>
$ gcloud config set project ${PROJECT_ID}
$ export CLUSTER_NAME=dame-cluster
$ gcloud container clusters create ${CLUSTER_NAME} \
    --zone ${CLUSTER_LOCATION} \
    --num-nodes=2 \
    --enable-shielded-nodes \
    --shielded-secure-boot \
    --shielded-integrity-monitoring \
    --enable-ip-alias \
    --enable-private-nodes \
    --master-ipv4-cidr "172.16.0.0/28" \
    --release-channel=regular
ERROR: (gcloud.container.clusters.create) ResponseError: code=400, message=Operation denied by custom org policy: ["customConstraints/custom.enablePrivateEndpoint": "Only allow GKE Cluster resource create or update if Private Endpoint is enabled" "customConstraints/custom.enableWorkloadIdentity": "All new clusters must use Workload Identity."].
```
想定通り `enablePrivateEndpoint` と `enableWorkloadIdentity` に引っかかってエラーが出力されました。

続いて成功パターンで試してみます。先ほど怒られた内容を踏まえて `--workload-pool` と `--enable-private-endpoint` をちゃんと設定して試してみます。
```bash
$ export WORKLOAD_POOL=${PROJECT_ID}.svc.id.goog
$ export CLUSTER_NAME=ok-cluster
$ gcloud container clusters create ${CLUSTER_NAME} \
    --zone ${CLUSTER_LOCATION} \
    --num-nodes=2 \
    --workload-pool=${WORKLOAD_POOL} \
    --enable-shielded-nodes \
    --shielded-secure-boot \
    --shielded-integrity-monitoring \
    --enable-ip-alias \
    --enable-private-nodes \
    --master-ipv4-cidr "172.16.0.0/28" \
    --enable-private-endpoint \
    --release-channel=regular
Creating cluster ok-cluster in asia-northeast1-b... Cluster is being health-checked (master is healthy)...done.                                            
Created [https://container.googleapis.com/v1/projects/kuchima-adventcal2022-prod/zones/asia-northeast1-b/clusters/ok-cluster].
```
無事作成できました。続いて COS 以外のノード (Ubuntu) を追加したときの挙動も確認してみます。
```bash
$ gcloud container node-pools create ubuntu-pool \
    --cluster ${CLUSTER_NAME} \
    --zone ${CLUSTER_LOCATION} \
    --num-nodes=2 \
    --shielded-secure-boot \
    --shielded-integrity-monitoring \
    --image-type "UBUNTU_CONTAINERD"
ERROR: (gcloud.container.node-pools.create) ResponseError: code=400, message=Operation denied by custom org policy: ["customConstraints/custom.requireCOSNode": "All cluster nodes must be COS."].
```
こちらも想定通り `requireCOSNode` によりノードの作成がブロックされました。これで本番環境ではある程度安全な GKE クラスタの作成が強制されるようになりました。

## ③ 開発環境では Spot VM のみ許可する
最後にコスト節約のために開発環境では Spot VM のみ許可するような制約を作ります。
### 1. カスタム制約の作成
まずマニフェストを作成します。Spot VM は `NodePool` の `resource.config.spot` で設定するので、これが `true` になっているかを確認します。
```yaml:constraint-enableSpotVM.yaml
name: organizations/ORGANIZATION_ID/customConstraints/custom.enableSpotVM
resourceTypes:
- container.googleapis.com/NodePool
methodTypes:
- CREATE
condition: resource.config.spot == true
actionType: ALLOW
displayName: Enable Spot VMs
description: All cluster nodes must be Spot VMs.
```

組織内にカスタム制約を作成します。
```bash
$ gcloud org-policies set-custom-constraint constraint-enableSpotVM.yaml
Created custom constraint [organizations/ORGANIZATION_ID/customConstraints/custom.enableSpotVM].
```

### 2. カスタム制約の適用
ポリシーのマニフェストを作成し、カスタム制約を `dev` フォルダに適用します。
```yaml:policy-enableSpotVM.yaml
name: folders/DEV_FOLDER_ID/policies/custom.enableSpotVM
spec:
  rules:
  - enforce: true
```

```bash
$ gcloud org-policies set-policy policy-enableSpotVM.yaml
Created policy [folders/DEV_FOLDER_ID/policies/custom.enableSpotVM].
```
### 3. 制約の挙動を確認する
では実際に試してみます。`② 本番環境ではセキュアな構成の GKE クラスタを強制する`でデプロイに成功したセキュアな構成のクラスタをそのまま開発環境にもデプロイしてみます。
```bash
$ export PROJECT_ID=<DEV PROJECT ID>
$ gcloud config set project ${PROJECT_ID}
$ export WORKLOAD_POOL=${PROJECT_ID}.svc.id.goog
$ export CLUSTER_NAME=secured-cluster
$ gcloud container clusters create ${CLUSTER_NAME} \
    --zone ${CLUSTER_LOCATION} \
    --num-nodes=2 \
    --workload-pool=${WORKLOAD_POOL} \
    --enable-shielded-nodes \
    --shielded-secure-boot \
    --shielded-integrity-monitoring \
    --enable-ip-alias \
    --enable-private-nodes \
    --master-ipv4-cidr "172.16.0.0/28" \
    --enable-private-endpoint \
    --release-channel=regular
ERROR: (gcloud.container.clusters.create) ResponseError: code=400, message=Operation denied by custom org policy: ["customConstraints/custom.enableSpotVM": "All cluster nodes must be Spot VMs."].
```
そうすると想定通り `enableSpotVM` によってクラスタの作成が阻止されました。
続いて `--spot` を追加して、Spot VM としてクラスタを作成してみます。

```bash
$ export CLUSTER_NAME=spot-cluster
$ gcloud container clusters create ${CLUSTER_NAME} \
    --zone ${CLUSTER_LOCATION} \
    --num-nodes=2 \
    --workload-pool=${WORKLOAD_POOL} \
    --enable-shielded-nodes \
    --shielded-secure-boot \
    --shielded-integrity-monitoring \
    --enable-ip-alias \
    --enable-private-nodes \
    --master-ipv4-cidr "172.16.0.0/28" \
    --enable-private-endpoint \
    --release-channel=regular \
    --spot
Creating cluster spot-cluster in asia-northeast1-b... Cluster is being health-checked (master is healthy)...done. 
Created [https://container.googleapis.com/v1/projects/kuchima-adventcal2022-dev/zones/asia-northeast1-b/clusters/spot-cluster].
```
クラスタの作成に成功しました。これで動作確認は終了です。

## お片付け
このまま制約を残しておくと他の作業に影響が出る可能性もあるので、`gcloud org-policies delete-custom-constraint` で作成した制約を削除しましょう。
```bash
$ gcloud org-policies delete-custom-constraint custom.CONSTRAINT_NAME --organization=ORGANIZATION_ID
```

# まとめ
組織ポリシーのカスタム制約により、柔軟かつ強固な制約を組織全体や特定フォルダ配下のプロジェクトに適用しガバナンスを効かせることができるようになりました。
まだ Preview の機能なので対応しているサービスが少なかったり色々と足りていない部分はありますが、今後成熟してくると便利に使えそうで個人的にはかなり期待しています。気になった方はぜひ試してみてください。
明日 3 日目は Google Workspace に関する記事です。お楽しみに〜

# 参考資料
https://cloud.google.com/kubernetes-engine/docs/how-to/custom-org-policies
https://cloud.google.com/resource-manager/docs/organization-policy/creating-managing-custom-constraints#common_expression_language
https://github.com/kkuchima/custom-orgpolicy-sample