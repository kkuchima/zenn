---
title: "GKE の 2022 年アップデートを振り返る"
emoji: "🆕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP, GoogleCloud, GKE]
publication_name: "google_cloud_jp"
published: false
---
公開がかなり遅くなってしまいましたが、昨年 2022 年の GKE アップデートを振り返ってみたいと思います。
2022 年も GKE には多くのアップデートがありました。多すぎて全部のアップデートを書くと読むのも大変だと思うので、本記事では今年のアップデートの中でも個人的に注目度の高いものをピックアップして紹介していこうと思います。とはいえ結構あります。

* ノード
* ネットワーク
* ストレージ
* オブザーバビリティ

※本記事で紹介する各機能・サービスの [Product launch stages](https://cloud.google.com/products#product-launch-stages)(GA, Preview) は記事執筆時点のものです

# tl;dr
* 


# ノード

## GKE Autopilot コンピューティングクラス (Preview)
GKE Autopilot 1.24.1-gke.1400 以降で `Balanced` と　`Scale-Out` のコンピューティングクラスが利用可能になりました。
以下 3 種類のコンピューティングクラスからワークロードに適切なクラスを選択することにより、ワークロードが動く Node のマシンファミリーや CPU アーキテクチャを指定することができるようになりました。
![コンピューティングクラスの種類](/images/gke-updates-2022/compute-class.png)

選択するコンピューティングクラスにより、ベースとなるマシンファミリーが自動的に選択されます。ちなみに `Scale-Out` は 2022.12 現在、asia-northeast1 では利用不可です。
* `General-purpose` は E2 シリーズ (Default)
* `Balanced` は N2, N2D シリーズ
* `Scale-Out` は T2D(x86), T2A(Arm) シリーズ

使い方としてはデプロイする Pod の nodeSelector や nodeAffinity の条件として `cloud.google.com/compute-class` label を設定するだけです。
以下は `Scale-Out` コンピュートクラスの Node 上にワークロードをデプロイする例です：
```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hello-app
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: hello-app
      template:
        metadata:
          labels:
            app: hello-app
        spec:
          nodeSelector:
            cloud.google.com/compute-class: "Scale-Out"
          containers:
          - name: hello-app
            image: us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0
            resources:
              requests:
                cpu: "2000m"
                memory: "2Gi"
```
上記のマニフェストが適用されることにより、`Scale-Out` class の Node 自動的にプロビジョニングされ、その Node 上でワークロードが稼働するようになります。
arm64 など特定の CPU アーキテクチャを指定したい場合は、`kubernetes.io/arch:arm64` のように、条件を追加することで実現できます。
価格の観点ですが、コンピューティングクラス毎に Pod 単価が異なりますので利用にあたっては事前に[料金体系](https://cloud.google.com/kubernetes-engine/pricing)もご確認ください。


## GKE Autopilot GPU サポート (Preview)
GKE Autopilot 1.24.2-gke.1800 以降で GPU の利用がサポートされました。(後述するタイムシェアリング GPU とマルチインスタンス GPU は GKE Autopilot でサポートされていません)
これにより、機械学習やレンダリング等のワークロードでも GKE Autopilot を活用いただけるようになりました。利用者側での GPU ドライバのインストールも不要です。
GPU を使った Pod をデプロイしたい場合はコンピューティングクラス同様、nodeSelector の条件として `cloud.google.com/gke-accelerator` label を指定します。
2022.12 現在、指定可能な GPU のタイプは `nvidia-tesla-t4` もしくは `nvidia-tesla-a100` です。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-gpu-pod
spec:
  nodeSelector:
    cloud.google.com/gke-accelerator: nvidia-tesla-a100
  containers:
  - name: my-gpu-container
    image: nvidia/cuda:11.0.3-runtime-ubuntu20.04
    command: ["/bin/bash", "-c", "--"]
    args: ["while true; do sleep 600; done;"]
    resources:
      limits:
        nvidia.com/gpu: 4
```
GPU を要求する Pod がデプロイされると、GKE は割り当て可能な GPU ノードが存在しない場合、新しい GPU Node をプロビジョニングします。(その際、Node に NVIDIA のドライバが自動的にインストールされます)
料金は [GPU Pods 用の料金](https://cloud.google.com/kubernetes-engine/pricing)が課金されます。

## Compact placement Policy (GA)

## Spot VMs / Spot Pods (GA)
[Spot VM](https://cloud.google.com/kubernetes-engine/docs/concepts/spot-vms) という標準的な GCE インスタンスより安価な VM を GKE Standard の Node として利用可能になりました
これまではプリエンプティブル VM という最大 24 時間の稼働時間制限がある安価な Node を利用することができましたが、最近はこの後継として、稼働時間制限のない Spot VM という安価な Node が利用可能になりました。

Spot VM は GKE Autopilot では [Spot Pods](https://cloud.google.com/kubernetes-engine/docs/how-to/autopilot-spot-pods) という形で利用可能です。
Spot Pods では GKE が自動的に Spot VM のスケジュールや taints / toleration の付与をしてくれます。
Spot Pods を利用する際は、Compute Class と同様に nodeSelector や Node Affinity 設定で `cloud.google.com/gke-spot: "true"` の Node 上で Pod が稼働するよう指定するだけです。
Spot VMs / Pods はステートレスなバッチワークロードや、コスト削減を目的として開発環境等での利用に適しています。
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    metadata:
      labels:
        app: pi
    spec:
      nodeSelector:
        cloud.google.com/gke-spot: "true"
      terminationGracePeriodSeconds: 25
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

## CA location policy
Cluster Autscaler で [Location Policy](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-autoscaler#location_policy) という機能が利用可能になりました。とはいっても普段はあまり気にするような機能でもなく

GKE 1.24.1-gke.800 以降で、Cluster Autoscaler の分散ポリシーを変更することができます。
* **BALANCED**...
* **ANY**... 


## GKE Arm Nodes (Preview)
GKE version 1.24 以降で Arm ベースの Node ([Tau T2A](https://cloud.google.com/compute/docs/general-purpose-machines#t2a_machines)) がサポートされました。
GKE Autopilot では `Scale-Out` Compute Class を指定することで Arm ベースの Node が利用できます。
1 クラスタ内に x86 ベースの Node と Arm ベースの Node を共存させることも可能です。その際、ワークロードは NodeSelector や toleration を使ってデプロイ先の Node を選択するよう構成します。
ただし 2023.01 現時点では日本リージョン (asia-northeast1/2) での利用不可です。また以下の機能をサポートしていません。
* Container Threat Detection
* Confidential GKE Node
* ローカル SSD
* GPU
* Windows Node
* Google Managed Service for Prometheus
* Filestore CSI ドライバ
https://cloud.google.com/kubernetes-engine/docs/concepts/arm-on-gke

## Confidential GKE Nodes (GA) Standard
[Confidential GKE Nodes](https://cloud.google.com/kubernetes-engine/docs/how-to/confidential-gke-nodes) という [Confidential VM](https://cloud.google.com/compute/confidential-vm/docs/about-cvm) を GKE Node として利用できるオプションが GA になりました。本機能を有効化することにより、GKE Node 上で処理中のデータも暗号化することが可能となります。

Confidential VM Service を有効化することにより、GKE Node 上で処理中のデータを暗号化することが可能となる

実行中のコードの変更なく、メモリ内や中央処理装置（CPU）外のどの場所でも、データが暗号化された状態が保持されるよう構成することができる

2022.06 時点での主な制約：
Container-Optimized OS のみサポート
GPU 非サポート
PD ベースの Persistent Volume のみサポート (GKE 1.22 以降)


## GPU タイムシェア (GA)
ノード上の単一の物理 GPU を複数のコンテナで共有可能に
GPU をコンテナ間でシェアすることにより高いコスト効率を実現
GKE クラスタやノードプール単位で有効化・無効化可能

GPU タイムシェアが有効化されたノードのラベルを指定することで、ワークロードをデプロ
Autopilot では利用できない
マルチインスタンスとの違い




# ネットワーク

## Gateway API (シングルクラスタ:GA, マルチクラスタ:Preview)

## Gateway Cert manager

## Gateway で Envoy ベースの HTTP(S) LB をサポート (Preview)
個人的にはかなり嬉しいアップデートです。Envoy ベースの次世代 External HTTP(S) LB を Gateway API を使って利用することができるようになりました。
これまでは (同じく Envoyベースである) Internal HTTP(S) LB でしか使えなかったような、ヘッダベースのルーティングや重み付けルーティングなど高度なトラフィック制御

## Cloud DNS for GKE


# セキュリティ
## Security Posture Management

# ストレージ

## Backup for GKE

# アップグレード

## メンテナンスウィンドウ

## B/G upgrade

## Control Plane metrics / Logs

## 高スループット Log Agent

# まとめ




![Multi-cluster Gateway のキャパシティベースのルーティング](/images/gke-korekara-101/multi-cluster-capacitylb.png)

