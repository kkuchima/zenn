---
title: "Google Cloud とオンプレミス環境間でマルチクラスタ サービスメッシュ (ハイブリッドメッシュ) を構成する"
emoji: "🕸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP, anthos, istio]
published: true
---

# はじめに
Anthos Service Mesh (以降 ASM) で異なる環境間 (ハイブリッド・マルチクラウド) でのマルチクラスタ サービスメッシュの構成がサポートされました。2022.05 現在 Preview 機能としてご利用できます。  
GKE (on Google Cloud) のマルチクラスタメッシュは比較的簡単に構成できるのですが、 ハイブリッド・マルチクラウド環境間でのマルチクラスタメッシュは少し工夫が必要だったので備忘も兼ねて記事にします。  

ちなみに ASM とは、簡単に言うと Google Cloud が提供するマネージド版 Istio で、Google Cloud 環境だけでなくオンプレミスや他社クラウド上の Kubernetes クラスタでも動作します。  
(本記事では、サービスメッシュとは？ ASM とは？という点にはあまり触れませんので、気になる方は以下のブログ記事もご覧ください)
https://medium.com/google-cloud-jp/implementing-anthos-service-mesh-3d8205bf48ed

# tl;dr
- 複数クラスタ間でサービスメッシュを構成することにより、可用性の向上やセキュアなクラスタ間通信等を実現することができるようになります
- マルチクラスタメッシュでは、サービス単位でのリージョンを跨いだ failover やクラスタ間でのトラフィック分散など、高度なトラフィックコントロールも可能です
- 異なる環境間でのマルチクラスタメッシュでは、ネットワーク周りなど考慮が必要な点がいくつか出てくるので少し注意が必要です

# マルチクラスタでメッシュを組むと何が嬉しいのか
そもそもマルチクラスタメッシュとは何かというと、複数クラスタ間でサービスメッシュを
複数クラスタ間でサービスメッシュを組む目的として一番多いと思うのが、可用性の向上です。例えば東京と大阪にそれぞれ GKE クラスタを構築しマルチクラスタ サービスメッシュを構成することで、東京クラスタ上のサービスが利用不可になった場合も大阪クラスタ上の正常なサービスに自動的に接続する、といったことが実現できるようになります。(他にもクラスタ間でのサービスディスカバリやセキュアな通信 (mTLS 等)、可観測性など色々な目的があると思います)  

![Multi-cluster Meshの例](/images/hybrid-mesh/multi-cluster-mesh.png)

クラスタレベルでの障害に関しては、[Multi-cluster Ingress](https://cloud.google.com/kubernetes-engine/docs/concepts/multi-cluster-ingress) や [Multi-cluster Gateway](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-multi-cluster-gateways) 等の機能や DNS を用いて外部クライアントから来るトラフィックをコントロールすることでクラスタ単位で failover することはできますが、サービス単位というさらに細かい粒度で failover をしたい場合などにマルチクラスタメッシュの機能が役に立ちます。

ちなみに Multi-cluster Ingress / Gateway とマルチクラスタメッシュ機能の併用も可能です。Multi-cluster Ingress / Gateway でクラスタ外 (外部クライアント等) からのトラフィック (North-South Traffic) をコントロールし、マルチクラスタメッシュでサービス間通信 (East-West Traffic) をコントロールする形になります。

![MCI + Multi-cluster Meshの例](/images/hybrid-mesh/multi-cluster-mesh-mci.png)

今回はいわゆるハイブリッドクラウド環境間でのマルチクラスタメッシュを試してみます。  
イメージとしては 2020 年の KubeCon で Walmart 社が発表していた事例のように、ハイブリッドクラウド環境間でサービス単位での Failover ができるようにマルチクラスタメッシュでサービス間通信をコントロールすることを目指します。  

![Walmart Hybrid Mesh](/images/hybrid-mesh/walmart-hybrid-mesh.png)

https://static.sched.com/hosted_files/kccncna19/20/3%20Manesh%20and%20Siram%20New%20V2.pptx.pdf


# ASM でサポートしているマルチクラスタメッシュ構成
ASM では 2022.05 現在、以下 3 種類のマルチクラスタメッシュ構成をサポートしています：
1. 複数の GKE クラスタ間でのマルチクラスタメッシュ (Single Cloud)
2. GKE クラスタと Anthos on-prem クラスタ (on Bare Metal / on VMware) 間でのマルチクラスタメッシュ (Hybrid Cloud)
3. GKE クラスタと EKS クラスタ間でのマルチクラスタメッシュ (Multi Cloud)

今回はこの中の `2. GKE クラスタと Anthos on-prem クラスタ (on Bare Metal / on VMware) 間でのマルチクラスタメッシュ (Hybrid Cloud)` を試してみたいと思います。ハイブリッドメッシュは異なるネットワーク間でサービスメッシュを組むことになるので、 Istio のデプロイメントパターンとしては `Multi-primary / Multi-network` の構成となります。(現状、ASM では各クラスタで istiod を持つ Multi-primary のみサポートしています)
![Multi-primary / Multi-network](https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/arch.svg)

ちなみに `1. 複数の GKE クラスタ間でのマルチクラスタメッシュ (Single Cloud)` の場合は `Multi-primary / Single-network` の構成をサポートしています。クラスタが同一ネットワーク上にあり、ネットワーク的にサービス間で相互に通信可能であるため `Multi-network` で利用されているような East-West Traffic 用 Ingress Gateway は不要になります。
![Multi-primary / Single-network](https://istio.io/latest/docs/setup/install/multicluster/multi-primary/arch.svg)

ASM でサポートしているマルチクラスタ構成については、本記事の最後に参考資料として掲載している公式ドキュメントをご参照ください。

# マルチクラスタ間でのトラフィックコントロール
実際に試す前に、マルチクラスタメッシュでトラフィックをコントロールする方法の例をご紹介します。もっと細かく分けることもできると思いますが、今回は大きく以下 3 種類のユースケースに分けてみます。
1. ローカルのサービスが利用できない場合に他クラスタ上のサービスに Failover する
2. トラフィックをクラスタ間で分散させる
3. トラフィックをローカルのクラスタ内に閉じる (他クラスタ上のサービスにアクセスさせない)

## 1. ローカルのサービスが利用できない場合に他クラスタ上のサービスに Failover する
マルチクラスタメッシュを構成するとデフォルトでは複数クラスタ間でトラフィックを分散させる挙動となります。  
この設定で問題ない場合はそのままでも良いのですが、例えばレイテンシや通信コスト最適化の観点から、なるべく地理的に近い (= 同一クラスタ / リージョン) エンドポイントに優先して接続させたいという要望もあるかと思います。そのような場合は Locality Failover を有効化することで実現可能です。  
エンドポイントの Locality は K8s クラスタやノードの地理的な場所を示し、`topology.kubernetes.io/region` や `topology.kubernetes.io/zone` といった Label をベースに定義されています。  

Loacality Failover を有効化するためには DestinationRule で Outlier Detection を設定し、接続先エンドポイントの異常を検知するため (どういう状況になったらエンドポイントが異常であるとマークするか) の基準を定義する必要があります。また、`localityLbSetting` の `enabled` を `true` にします。  
これにより通常時は同一 Zone / Region 上のエンドポイントに優先的にアクセスし、それら優先エンドポイントが異常と判断されると別ロケーションのエンドポイントにアクセスする動きとなります。  
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld.sample.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
      localityLbSetting:
        enabled: true
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 1m
```

また `failover` のフィールドを設定することにより、failover 先を明示的に定義することもできます。以下の例だと、us-east 内のエンドポイントが異常 (unhealthy) と判断されると eu-west 内のエンドポイントに failover します。同様に us-west のエンドポイントが異常になると us-east に failover します。  

```yaml
    loadBalancer:
      simple: ROUND_ROBIN
      localityLbSetting:
        enabled: true
        failover:
          - from: us-east
            to: eu-west
          - from: us-west
            to: us-east
```

さらに `failoverPriority` というフィールドで failover する際に評価するラベルの優先順位を定義することもできます。 以下の例だと、Network / Region / Zone が同一のエンドポイントを最優先とし、次に Network / Region が同じエンドポイント、その次が Network が同じエンドポイント、一番優先度が低いのが全ての Label が異なるエンドポイント、となります。  

```yaml
failoverPriority:
- "topology.istio.io/network"
- "topology.kubernetes.io/region"
- "topology.kubernetes.io/zone"
```
ちなみにデフォルトでは以下の優先順位となっているようです ([Doc](https://istio.io/v1.8/docs/ops/configuration/traffic-management/locality-load-balancing/))
```
The hierarchy of prioritization matches in the following order:

1. Region
2. Zone
3. Sub-zone
```

## 2. トラフィックをマルチクラスタ間で分散させる
先述した通り、マルチクラスタメッシュを構成するとデフォルトでは複数クラスタ間でトラフィックを分散させる挙動となりますが、例えばこのリージョンのクラスタにこれくらいの割合でトラフィックを分散させる等高度なトラフィック制御をすることもできます。(Locality weighted distribution)  
Locality weighted distribution を有効化するためには、Locality failover と同様に DestinationRule で Outlier Detection を設定し、`localityLbSetting` の `enabled` を `true` にします。そして、`localityLbSetting.distribute` に重み付けの設定を入れていきます。  
  
以下の例だと、us-west/zone1 からのトラフィックは us-west/zone1 上のエンドポイントに 80%、us-west/zone2 に残り 20% を流します。同様に、us-west/zone2 からのトラフィックは us-west/zone2 上のエンドポイントに 80%、us-west/zone1 に残り 20% を流すように構成しています。

```yaml
  distribute:
    - from: us-west/zone1/*
      to:
        "us-west/zone1/*": 80
        "us-west/zone2/*": 20
    - from: us-west/zone2/*
      to:
        "us-west/zone1/*": 20
        "us-west/zone2/*": 80
```


## 3. トラフィックをローカルのクラスタ内に閉じる (他クラスタ上のサービスにアクセスさせない)
マルチクラスタ間でトラフィックを分散させたりするのは便利な一方、トラフィックを別クラスタに流したくないという要件もあるかと思います。その場合は、`MeshConfig.ServiceSettings.Settings` で `clusterLocal` を `true` にすることで、トラフィックがクラスタ内に閉じるように構成することができます。  
ローカルに閉じたい対象のサービスをワイルドカードを使って指定することにより、サービス・Namespace ・全てのトラフィックという単位で制御することができます。  
以下は Namespace 単位で設定する例です (ワイルドカードを使わず特定サービスを指定することもできますし、`*` とだけ指定し全てのサービスを対象にすることもできます)。  

```yaml
serviceSettings:
- settings:
    clusterLocal: true
  hosts:
  - "*.myns.svc.cluster.local"
```

他にも、クラスタを特定できるように Label を付与しておきクラスタ単位で Subset を作成し、Virtual Service 側でクラスタ内にトラフィックが閉じるように設定する方法等もあります。

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: mysvc-per-cluster-dr
spec:
  host: mysvc.myns.svc.cluster.local
  subsets:
  - name: cluster-1
    labels:
      topology.istio.io/cluster: cluster-1
  - name: cluster-2
    labels:
      topology.istio.io/cluster: cluster-2
```

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: mysvc-cluster-local-vs
spec:
  hosts:
  - mysvc.myns.svc.cluster.local
  http:
  - name: "cluster-1-local"
    match:
    - sourceLabels:
        topology.istio.io/cluster: "cluster-1"
    route:
    - destination:
        host: mysvc.myns.svc.cluster.local
        subset: cluster-1
  - name: "cluster-2-local"
    match:
    - sourceLabels:
        topology.istio.io/cluster: "cluster-2"
    route:
    - destination:
        host: mysvc.myns.svc.cluster.local
        subset: cluster-2
```

# ハイブリッドメッシュを実際に試してみる
前置きが長くなってしまったのですが、ここから実際にハイブリッドメッシュを試してみます。今回は以下のような環境を作っていきます。  

![Hybrid Mesh 構成例](/images/hybrid-mesh/hybrid-mesh.png)

環境構築の前提としては以下の通りです。基本的にデフォルト構成をそのまま使っています。
- GKE / Anthos on Bare Metal クラスタは構築済み
    - GKE ver: 1.21.10-gke.2000 , Anthos クラスタ ver: v1.21.5-gke.1300
    - GKE は Public Cluster (現状 Private Cluster はハイブリッドメッシュをサポートしていないため)
    - 同一 Project 上にクラスタを構築
- GKE / Anthos クラスタに ASM 導入済み 
    - ASM ver: 1.13.2-asm.2 
    - ASM は In-cluster Control Plane (現状 Managed Control Plane はハイブリッドメッシュをサポートしていないため)
    - CA として MeshCA を採用
- オンプレミスと Google Cloud 環境を Cloud VPN で接続
    - VPN トンネル構築時に GKE の Node / Pod IP レンジをオンプレミス側に広告するよう構成 (IP マスカレード等の設定入れていない場合、GKE Pod IP レンジの広告ができていないと Endpoint discovery 時の通信が上手くいかないはず)
    - オンプレ→Google Cloud の方向は Anthos ノードの IP レンジを広告 (Anthos はノード IP で SNAT するので)

その他要件などについては以下ドキュメントもご参照ください。  
https://cloud.google.com/service-mesh/docs/unified-install/multi-cloud-hybrid-mesh

## サンプルアプリケーションのデプロイ
まずはサンプルアプリケーションを GKE クラスタの default Namaespace 上にデプロイします。
```bash
$ kubectx gke
$ kubectl label namespace default istio.io/rev=asm-1132-2 --overwrite
$ kubectl apply -f echo-gke.yaml
```

マニフェストの中身はこんな感じです。`"GKE!!!"` と返すだけのアプリケーションを `echo-svc` というサービスとしてデプロイしています。
```yaml:echo-gke.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echoserver
  template:
    metadata:
      labels:
        app: echoserver
    spec:
      containers:
      - name: echoserver
        image: hashicorp/http-echo
        args:
          - -listen=:8080
          - -text="GKE!!!"
---
apiVersion: v1
kind: Service
metadata:
  name: echo-svc
spec:
  type: ClusterIP
  selector:
    app: echoserver
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

次に同様のサンプルアプリケーションを Anthos on Bare Metal クラスタの default Namaespace 上にデプロイします。
```bash
$ kubectx abm
$ kubectl label namespace default istio.io/rev=asm-1132-2 --overwrite
$ kubectl apply -f echo-anthos.yaml
```

マニフェストの中身は先ほど GKE クラスタにデプロイしたものとほぼ同じで、返す文字列のみ `"Anthos!!!"` に変更しています。
```yaml:echo-anthos.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echoserver
  template:
    metadata:
      labels:
        app: echoserver
    spec:
      containers:
      - name: echoserver
        image: hashicorp/http-echo
        args:
          - -listen=:8080
          - -text="Anthos!!!"
---
apiVersion: v1
kind: Service
metadata:
  name: echo-svc
spec:
  type: ClusterIP
  selector:
    app: echoserver
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

以下のように Proxy が injection された Pod が各クラスタにデプロイされていれば OK です。
```bash
$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
echoserver-865484cd65-mc8f4   2/2     Running   0          3m25s
echoserver-865484cd65-sxds2   2/2     Running   0          3m24s
```

## ハイブリッドメッシュの構成
ここからハイブリッドメッシュの構成をします。まずは諸々必要な情報を環境変数として設定します。
`NETWORK_1` には GKE クラスタのネットワークラベルの値を入力します。GKE の場合は `{PROJECT ID}-{VPC名}` というフォーマットになっているはずです。  
`NETWORK_2` には Anthos クラスタのネットワークラベルの値を入力します。Anthos の場合はデフォルトだと `default` と設定されていると思います。  
```bash
export PROJECT_NUMBER=$(gcloud projects describe kuchima-demo --format="value(projectNumber)")
export MESH_ID="proj-${PROJECT_NUMBER}"
export NETWORK_1="kuchima-demo-vpc01"
export NETWORK_2="default"
```

実際に設定されている値を見たい場合は、istio-system namespace や先ほどデプロイしたサンプルアプリの Pod に付与されている `topology.istio.io/network` ラベルの値を確認いただくのが良いと思います。
```bash
$ kubectl describe pods echoserver-64f897b98c-c7gjg
Name:         echoserver-64f897b98c-c7gjg
Namespace:    default
Priority:     0
Node:         gke-gke-cluster-default-pool-655cd558-tfn3/10.100.10.2
Start Time:   Thu, 05 May 2022 10:39:58 +0000
Labels:       app=echoserver
              pod-template-hash=64f897b98c
              security.istio.io/tlsMode=istio
              service.istio.io/canonical-name=echoserver
              service.istio.io/canonical-revision=latest
              topology.istio.io/network=kuchima-demo-vpc01
```

まずは GKE クラスタ用の East-West Gateway (異なるネットワーク間を通信させるための Gateway) マニフェストを用意します。ASM インストール時に `--output_dir` で指定したディレクトリ配下にある `asm/istio/expansion/gen-eastwest-gateway.sh` を使って雛形のマニフェストを作成します。ちなみに先述した通り、複数 GKE クラスタ (Single Cloud のケース) の場合はこの East-West Gateway のデプロイはしなくても大丈夫です。ハイブリッド環境など Multi-Network なデプロイメントモデルの場合に必要になります。
```bash
$ asm/istio/expansion/gen-eastwest-gateway.sh --mesh ${MESH_ID} --network ${NETWORK_1} --revision asm-1132-2 > eastwest-gateway-gke.yaml
```

生成したマニフェストに少し手を加えていきます。今回の構成ではオンプレと Google Cloud 間を VPN で接続していることもあり、East-West Gateway はパブリックなサービスとして公開したくないため、Internal LB で公開するように `k8s.erviceAnnotations.cloud.google.com/load-balancer-type` に `"internal"` と設定します。  
また、Ingress Gateway は istio-system とは別の Namespace にデプロイするのが最近の[ベストプラクティス](https://cloud.google.com/service-mesh/docs/gateways#best_practices_for_deploying_gateways)なので、`gateway` という namespace 上にデプロイするようにします。(なんなら istio opeator も使わず k8s manifest でデプロイしてしまっても良いとは思いますが、今回はそのまま istio operator を利用します)
```yaml:eastwest-gateway-gke.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: eastwest
spec:
  revision: "asm-1132-2"
  profile: empty
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        namespace: gateway
        label:
          istio: eastwestgateway
          app: istio-eastwestgateway
          topology.istio.io/network: kuchima-demo-vpc01
        enabled: true
        k8s:
          serviceAnnotations:
            cloud.google.com/load-balancer-type: "internal"
          env:
            # traffic through this gateway should be routed inside the network
            - name: ISTIO_META_REQUESTED_NETWORK_VIEW
              value: kuchima-demo-vpc01
          service:
            ports:
              - name: status-port
                port: 15021
                targetPort: 15021
              - name: tls
                port: 15443
                targetPort: 15443
              - name: tls-istiod
                port: 15012
                targetPort: 15012
              - name: tls-webhook
                port: 15017
                targetPort: 15017
  values:
    gateways:
      istio-ingressgateway:
        injectionTemplate: gateway
    global:
      network: kuchima-demo-vpc01
```

マニフェストが用意できたら GKE クラスタ上に East-West Gateway をデプロイします。  
※istioctl のバージョンが古いとマニフェスト apply 時にエラーが出力される可能性ありますのでご注意ください。
```bash
$ kubectx gke
$ kubectl create namespace gateway
$ kubectl label namespace gateway istio.io/rev=asm-1132-2 --overwrite
$ istioctl install -f eastwest-gateway-gke.yaml
```

同様に Anthos クラスタにも East-West Gateway をデプロイします。まずは先ほどと同様にマニフェストを生成します。
```bash
$ asm/istio/expansion/gen-eastwest-gateway.sh --mesh ${MESH_ID} --network ${NETWORK_2} --revision asm-1132-2 > eastwest-gateway-anthos.yaml
```

`gateway` という namespace 上に East-West Gateway をデプロイするようにマニフェストを修正します。Anthos クラスタの場合は元々プライベートな IP アドレスレンジを使ってサービス公開される設定にしているため、Service 周りは特に変更しません。
```yaml:eastwest-gateway-anthos.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: eastwest
spec:
  revision: "asm-1132-2"
  profile: empty
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        namespace: gateway
        label:
          istio: eastwestgateway
          app: istio-eastwestgateway
          topology.istio.io/network: default
        enabled: true
        k8s:
          env:
            # traffic through this gateway should be routed inside the network
            - name: ISTIO_META_REQUESTED_NETWORK_VIEW
              value: default
          service:
            ports:
              - name: status-port
                port: 15021
                targetPort: 15021
              - name: tls
                port: 15443
                targetPort: 15443
              - name: tls-istiod
                port: 15012
                targetPort: 15012
              - name: tls-webhook
                port: 15017
                targetPort: 15017
  values:
    gateways:
      istio-ingressgateway:
        injectionTemplate: gateway
    global:
      network: default
```

修正したマニフェストを使って Anthos クラスタ上に East-West Gateway をデプロイします。  
```bash
$ kubectx abm
$ kubectl create namespace gateway
$ kubectl label namespace gateway istio.io/rev=asm-1132-2 --overwrite
$ istioctl install -f eastwest-gateway-anthos.yaml
```

以下のように各クラスタに East-West Gateway がデプロイされていれば OK です。
```bash
$ kubectl get pods -n gateway
NAME                                    READY   STATUS    RESTARTS   AGE
istio-eastwestgateway-cc9ff9b65-sfcv9   1/1     Running   0          25s

$ kubectl get svc -n gateway
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.96.7.102   172.16.10.152   15021:32410/TCP,15443:32526/TCP,15012:31330/TCP,15017:31722/TCP   31s
```

デプロイできたら East-West Gateway 用の Gateway リソース (ややこしいですね) を作成し、全サービス (`*.local`) を他クラスタに公開できるようにします。  
`asm/istio/expansion/expose-services.yaml` というマニフェストの namespace のみ `gateway` に変えておきます。
```yaml:asm/istio/expansion/expose-services.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: cross-network-gateway
  namespace: gateway
spec:
  selector:
    istio: eastwestgateway
  servers:
    - port:
        number: 15443
        name: tls
        protocol: TLS
      tls:
        mode: AUTO_PASSTHROUGH
      hosts:
        - "*.local"
```

修正したマニフェストを各クラスタに apply します。
```bash
$ kubectx gke
$ kubectl apply -f asm/istio/expansion/expose-services.yaml
gateway.networking.istio.io/cross-network-gateway created

$ kubectx abm
$ kubectl apply -f asm/istio/expansion/expose-services.yaml
gateway.networking.istio.io/cross-network-gateway created
```

## クラスタ間のサービスディスカバリ設定
最後に各クラスタ間でサービスディスカバリができるように `istioctl x create-remote-secret` を実行します。これでマルチクラスタメッシュの構成自体は完了です。
```bash
$ export CTX_CLUSTER1=gke
$ export CTX_CLUSTER2=abm

$ istioctl x create-remote-secret \
  --context="${CTX_CLUSTER1}" \
  --name=gke | \
  kubectl apply -f - --context="${CTX_CLUSTER2}"

$ istioctl x create-remote-secret \
  --context="${CTX_CLUSTER2}" \
  --name=anthos | \
  kubectl apply -f - --context="${CTX_CLUSTER1}"
```

## 動作確認
疎通確認用の sleep pod を各クラスタにデプロイします。
```bash
# GKE
$ kubectx gke
$ kubectl create ns sleep
$ kubectl label namespace sleep \
  istio.io/rev=asm-1132-2 --overwrite
$ kubectl apply -f istio-1.13.2-asm.2/samples/sleep/sleep.yaml -n sleep

# Anthos
$ kubectx abm
$ kubectl create ns sleep
$ kubectl label namespace sleep \
  istio.io/rev=asm-1132-2 --overwrite
$ kubectl apply -f istio-1.13.2-asm.2/samples/sleep/sleep.yaml -n sleep
```

sleep pod からは echo-svc サービスのエンドポイントとして自クラスタのエンドポイント 2 つと、対向クラスタの East-West Gateway が登録されていることが分かります。(15443 で待ち受けているのが対向クラスタの East-West Gateway です)
```bash
# GKE
$ kubectx gke
$ export SLEEP_POD=$(kubectl get pod -n sleep -l app=sleep -o jsonpath='{.items[0].metadata.name}')

$ istioctl pc ep $SLEEP_POD.sleep | grep echo-svc
10.24.1.9:8080                   HEALTHY     OK                outbound|80||echo-svc.default.svc.cluster.local
10.24.2.10:8080                  HEALTHY     OK                outbound|80||echo-svc.default.svc.cluster.local
172.16.10.152:15443              HEALTHY     OK                outbound|80||echo-svc.default.svc.cluster.local

# GKE の East-West Gateway の IP アドレス確認
$ kubectl get svc -n gateway
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.28.3.185   10.100.10.6   15021:31339/TCP,15443:30870/TCP,15012:30502/TCP,15017:32405/TCP   18h

# Anthos
$ kubectx abm
$ export SLEEP_POD=$(kubectl get pod -n sleep -l app=sleep -o jsonpath='{.items[0].metadata.name}')

$ istioctl pc ep $SLEEP_POD.sleep | grep echo-svc
10.10.2.59:8080                  HEALTHY     OK                outbound|80||echo-svc.default.svc.cluster.local
10.10.3.80:8080                  HEALTHY     OK                outbound|80||echo-svc.default.svc.cluster.local
10.100.10.6:15443                HEALTHY     OK                outbound|80||echo-svc.default.svc.cluster.local

# Anthos の East-West Gateway の IP アドレス確認
$ kubectl get svc -n gateway
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   10.96.7.102   172.16.10.152   15021:32410/TCP,15443:32526/TCP,15012:31330/TCP,15017:31722/TCP   18h
```

現在の状態で実際にアクセスを試してみます。デフォルト設定なので各クラスタ上のエンドポイントに分散される動きになるはずです。  
Sleep pod から echo-svc 宛てに curl を何回か打ってみます。
```bash
# GKE
$ kubectx gke
$ export SLEEP_POD=$(kubectl get pod -n sleep -l app=sleep -o jsonpath='{.items[0].metadata.name}')
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"GKE!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"GKE!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"GKE!!!"

# Anthos
$ kubectx abm
$ export SLEEP_POD=$(kubectl get pod -n sleep -l app=sleep -o jsonpath='{.items[0].metadata.name}')
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"GKE!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"GKE!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"GKE!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
```
GKE とオンプレミス両方から実行した場合も各クラスタに分散しアクセスしていることが分かります。

## Locality Load Balancing を試す
個人的には、ハイブリッドメッシュな環境ではクラスタ間でトラフィックを常に分散させるよりも、ローカルサービスが利用不可になった場合に Failover をするような構成の方が需要があるのではないかと思うので (もちろんケースバイケースだと思いますが)、Locality Failover を試してみます。  
まずは以下の Destination Rule マニフェストを用意します。`localityLbSetting` を有効化し、`outlierDetection` も設定されています(パラメーターは適当です)。また、オンプレミス `tokyo-onprem` の failover 先として `asia-northeast1` リージョンを指定しています。
```yaml:echo-svc-dr.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: echo-svc
spec:
  host: echo-svc.default.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
      localityLbSetting:
        enabled: true
        failover:
          - from: tokyo-onprem
            to: asia-northeast1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 1m
```

ではこの Destination Rule を適用し動作を確認してみます。実は Anthos クラスタのノードには `topology.kubernetes.io/region` や `topology.kubernetes.io/zone` ラベルがデフォルトで付与されていないため、先ほど Destination Rule 内で指定した Region label もこのタイミングで追加しておきます。
```bash
$ kubectx abm

# Region label の追加
$ kubectl label nodes node01 topology.kubernetes.io/region=tokyo-onprem
$ kubectl label nodes node02 topology.kubernetes.io/region=tokyo-onprem
$ kubectl label nodes node03 topology.kubernetes.io/region=tokyo-onprem

# Destination Rule の作成
$ kubectl apply -f echo-svc-dr.yaml
destinationrule.networking.istio.io/echo-svc created

# 動作確認
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
```
オンプレミス環境上の sleep pod からは、オンプレミスの echo-svc に優先的に接続していることが分かります。  
ではオンプレミス側の echo pod の Proxy を Drain し、疑似的にオンプレミス側サービスで異常が発生した際の挙動もみてみます。

```bash
# Proxy の Drain
$ kubectl  exec \
  "$(kubectl get pod -l app=echoserver \
  -o jsonpath='{.items[0].metadata.name}')" \
  -c istio-proxy -- curl -sSL -X POST 127.0.0.1:15000/drain_listeners
OK

# Drain したエンドポイントが削除されている
$ istioctl pc ep $SLEEP_POD.sleep | grep echo-svc
10.10.2.111:8080                 HEALTHY     OK                outbound|80||echo-svc.default.svc.cluster.local
10.100.10.6:15443                HEALTHY     OK                outbound|80||echo-svc.default.svc.cluster.local

# まだオンプレミス側にアクセスしている
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"

# もう 1 つの Pod の Proxy も Drain する
$ kubectl  exec \
  "$(kubectl get pod -l app=echoserver \
  -o jsonpath='{.items[1].metadata.name}')" \
  -c istio-proxy -- curl -sSL -X POST 127.0.0.1:15000/drain_listeners

# GKE 側 East-West Gateway のエンドポイントのみ残っている
$ istioctl pc ep $SLEEP_POD.sleep | grep echo-svc
10.100.10.6:15443                HEALTHY     OK                outbound|80||echo-svc.default.svc.cluster.local

# オンプレミス側が利用不可になっても GKE 側にアクセスを逃すことができている
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"GKE!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"GKE!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"GKE!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"GKE!!!"
```
以上より、Locality Failover が構成されていることを確認できました。同様の設定を GKE 側でも行うことでクラウド側でも locality を意識したルーティングが設定されます。  

## (おまけ) clusterLocal 設定
おまけですが、Destination Rule を消し、下記のように default namespace 配下のサービスを対象に `clusterLocal` を有効化してみたところ、これまでは見えていた他クラスタの echo-svc エンドポイントが載らなくなっていました。(default 以外の namespace のエンドポイントはちゃんと拾っている)
```yaml:clusterlocal.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    serviceSettings:
    - settings:
        clusterLocal: true
      hosts:
      - "*.default.svc.cluster.local"
```

```bash
# 自クラスタのエンドポイントしか見えていない
$ istioctl pc ep $SLEEP_POD.sleep | grep echo-svc
10.10.2.240:8080                 HEALTHY     OK                outbound|80||echo-svc.default.svc.cluster.local
10.10.3.73:8080                  HEALTHY     OK                outbound|80||echo-svc.default.svc.cluster.local

# default namespace 以外のサービスは他クラスタのエンドポイント情報を拾ってきている
$ istioctl pc ep $SLEEP_POD.sleep | grep httpbin-svc
10.10.3.248:80                   HEALTHY     OK                outbound|80||httpbin-svc.sleep.svc.cluster.local
10.100.10.6:15443                HEALTHY     OK                outbound|80||httpbin-svc.sleep.svc.cluster.local

# 自クラスタ上のエンドポイントにのみアクセスしている
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
$ kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS echo-svc.default
"Anthos!!!"
```

# 最後に
異なる環境間でサービスメッシュを組む場合には色々と検討ポイントがあり、少しとっつきにくく感じた方もいらっしゃったかもしれませんが、使いこなせれば強い武器にもなる機能ではないかと思います。ちなみにマルチクラウドの環境ではまだ試せていませんが概ね同様の手順で構成できるかと思います。  
この記事を読んでいただいた方で興味を持ってくれた方、要件的にマルチクラスタメッシュの検討が必要そうな方がいらっしゃいましたらぜひ試してみてください。

# 参考資料
## ASM のマルチクラスタメッシュ関連ドキュメント 
https://cloud.google.com/service-mesh/docs/unified-install/multi-cloud-hybrid-mesh
https://cloud.google.com/service-mesh/docs/supported-features#multi-cluster_support
https://cloud.google.com/service-mesh/docs/managed/supported-features-mcp#platform_environment

## Istio のマルチクラスタメッシュ関連ドキュメント
https://istio.io/latest/docs/tasks/traffic-management/locality-load-balancing/
https://istio.io/latest/docs/ops/configuration/traffic-management/multicluster/

## Envoy ドキュメント
https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/locality_weight
https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/priority