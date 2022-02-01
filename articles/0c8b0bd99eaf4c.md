---
title: "ASM / Istio でクラスタ間トラフィックを制御する"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [gcp,istio]
published: false
---
ASM (Istio) でマルチクラスタメッシュを構成した際のトラフィック制御方法についての情報をまとめてみました。
# tl;dr
- 
- 

# なぜマルチクラスタでメッシュを組むのか？
以下ドキュメントに記載されているように、マルチクラスタを構成するユースケースは色々と考えられますが、
今回のようにクラスタ間でサービスメッシュを構成するような場合は、可用性の向上というのが目的になっているケースが多いかと思います。
例えば東京と大阪にそれぞれクラスタを作成してリージョンレベルでの冗長構成をとったり、オンプレミス-クラウド間や複数クラウド間で冗長構成をとったりなどなど
https://cloud.google.com/anthos/multicluster-management/use-cases

マルチクラスタの間でサービスメッシュを構成するメリットとしては、

# マルチクラスタ構成例
Istio はデプロイメントモデルは
https://istio.io/latest/docs/ops/deployment/deployment-models/

# トラフィックの制御方法


# 実際に試してみる

## GKE クラスタ作成
### Tokyo クラスタ作成
```bash
export GCP_EMAIL_ADDRESS=$(gcloud auth list --filter=status:ACTIVE \
  --format="value(account)")
export PROJECT_ID=$(gcloud config list --format   "value(core.project)")
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID}   --format="value(projectNumber)")
export CLUSTER_NAME=gke-tokyo
export CLUSTER_LOCATION=asia-northeast1-a

gcloud container clusters create ${CLUSTER_NAME} \
--zone ${CLUSTER_LOCATION} \
--num-nodes=5 \
--release-channel=regular
```

### Osaka クラスタ作成
```bash
export GCP_EMAIL_ADDRESS=$(gcloud auth list --filter=status:ACTIVE \
  --format="value(account)")
export PROJECT_ID=$(gcloud config list --format   "value(core.project)")
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID}   --format="value(projectNumber)")
export CLUSTER_NAME=gke-osaka
export CLUSTER_LOCATION=asia-northeast2-a

gcloud container clusters create ${CLUSTER_NAME} \
--zone ${CLUSTER_LOCATION} \
--num-nodes=5 \
--release-channel=regular
```

## ASM インストール
### asmcli ダウンロード
```bash
mkdir asm-multicluster && cd $_

curl https://storage.googleapis.com/csm-artifacts/asm/asmcli_1.12 > asmcli
chmod +x asmcli
```

### Tokyo クラスタに ASM をインストール
```bash
export CLUSTER_NAME=gke-tokyo
export CLUSTER_LOCATION=asia-northeast1-a

./asmcli install \
  --project_id ${PROJECT_ID} \
  --cluster_name ${CLUSTER_NAME} \
  --cluster_location ${CLUSTER_LOCATION} \
  --fleet_id ${PROJECT_ID} \
  --output_dir ${CLUSTER_NAME} \
  --enable_all \
  --ca mesh_ca
```

### Osaka クラスタに ASM をインストール
```bash
export CLUSTER_NAME=gke-osaka
export CLUSTER_LOCATION=asia-northeast2-a

./asmcli install \
  --project_id ${PROJECT_ID} \
  --cluster_name ${CLUSTER_NAME} \
  --cluster_location ${CLUSTER_LOCATION} \
  --fleet_id ${PROJECT_ID} \
  --output_dir ${CLUSTER_NAME} \
  --enable_all \
  --ca mesh_ca
```

### インストールの確認
```bash
${CLUSTER_NAME}/istioctl version

# client version: 1.12.0-asm.4
# control plane version: 1.12.0-asm.4
# data plane version: none
```

### マルチクラスタインストール
https://cloud.google.com/service-mesh/docs/unified-install/gke-install-multi-cluster
```bash
kubectx gke-tokyo=gke_kuchima002_asia-northeast1-a_gke-tokyo
kubectx gke-osaka=gke_kuchima002_asia-northeast2-a_gke-osaka
```

```bash
export PROJECT_ID=$(gcloud config list --format   "value(core.project)")
export LOCATION_1=asia-northeast1-a
export CLUSTER_1=gke-tokyo
export LOCATION_2=asia-northeast2-a
export CLUSTER_2=gke-osaka

./asmcli create-mesh \
    ${PROJECT_ID} \
    ${PROJECT_ID}/${LOCATION_1}/${CLUSTER_1} \
    ${PROJECT_ID}/${LOCATION_2}/${CLUSTER_2}

```

### サンプルアプリケーションのデプロイ
```bash
# ラベルの確認
kubectl -n istio-system get pods -l app=istiod --show-labels
# istiod-asm-1120-4-7986fbdf6b-cvl5p   1/1     Running   0          97m   app=istiod,istio.io/rev=asm-1120-4

kubectx gke-tokyo
kubectl create namespace sample
kubectl label namespace sample istio-injection- istio.io/rev=asm-1120-4 --overwrite
kubectl apply -f ${CLUSTER_NAME}/istio-1.12.0-asm.4/samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
kubectl apply -f ${CLUSTER_NAME}/istio-1.12.0-asm.4/samples/helloworld/helloworld.yaml \
  -l version=v1 -n sample
kubectl get pods -n sample
# NAME                             READY   STATUS    RESTARTS   AGE
# helloworld-v1-776f57d5f6-m9mm9   2/2     Running   0          2m7s

kubectx gke-osaka
kubectl create namespace sample
kubectl label namespace sample istio-injection- istio.io/rev=asm-1120-4 --overwrite
kubectl apply -f ${CLUSTER_NAME}/istio-1.12.0-asm.4/samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
kubectl apply -f ${CLUSTER_NAME}/istio-1.12.0-asm.4/samples/helloworld/helloworld.yaml \
  -l version=v2 -n sample
kubectl get pods -n sample
# NAME                            READY   STATUS    RESTARTS   AGE
# helloworld-v2-54df5f84b-vccq5   2/2     Running   0          47s

```

### sleep アプリケーションのデプロイ
```bash
kubectx gke-tokyo
kubectl apply -f ${CLUSTER_NAME}/istio-1.12.0-asm.4/samples/sleep/sleep.yaml -n sample

kubectx gke-osaka
kubectl apply -f ${CLUSTER_NAME}/istio-1.12.0-asm.4/samples/sleep/sleep.yaml -n sample
```

### Helloworld サービスの呼び出し
```bash
kubectl exec -n sample -c sleep \
    "$(kubectl get pod -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.sample:5000/hello

# Hello version: v1, instance: helloworld-v1-776f57d5f6-m9mm9
# Hello version: v2, instance: helloworld-v2-54df5f84b-vccq5
# Hello version: v1, instance: helloworld-v1-776f57d5f6-m9mm9
# Hello version: v2, instance: helloworld-v2-54df5f84b-vccq5
```

kubectl exec -n sample -c sleep \
    "$(kubectl get pod -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.sample.svc.cluster.local:5000/hello
→ 交互に各クラスタにアクセスしてる



```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld.sample.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        maxRequestsPerConnection: 1
    loadBalancer:
      simple: ROUND_ROBIN
      localityLbSetting:
        enabled: true
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 1m
```

```bash
cat << EOF > dr-failover.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld.sample.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        maxRequestsPerConnection: 1
    loadBalancer:
      simple: ROUND_ROBIN
      localityLbSetting:
        enabled: true
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 1m
EOF


kubectl apply -f dr-failover.yaml -n sample
```

$ kubectl exec -n sample -c sleep \
>     "$(kubectl get pod -n sample -l \
>     app=sleep -o jsonpath='{.items[0].metadata.name}')" \
>     -- curl -sS helloworld.sample:5000/hello
Hello version: v2, instance: helloworld-v2-54df5f84b-vccq5
kuchima@cloudshell:~/asm-multicluster (kuchima002)$ kubectl exec -n sample -c sleep     "$(kubectl get pod -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')"     -- curl -sS helloworld.sample:5000/hello
Hello version: v2, instance: helloworld-v2-54df5f84b-vccq5
kuchima@cloudshell:~/asm-multicluster (kuchima002)$ kubectl exec -n sample -c sleep     "$(kubectl get pod -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')"     -- curl -sS helloworld.sample:5000/hello
Hello version: v2, instance: helloworld-v2-54df5f84b-vccq5


### ローカルアプリケーションの Proxy を Drain してみる
```bash
kubectl  exec \
  "$(kubectl get pod -n sample -l app=helloworld \
  -o jsonpath='{.items[0].metadata.name}')" \
  -n sample -c istio-proxy -- curl -sSL -X POST 127.0.0.1:15000/drain_listeners
```
$ kubectl exec -n sample -c sleep     "$(kubectl get pod -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')"     -- curl -sS helloworld.sample:5000/hello
Hello version: v1, instance: helloworld-v1-776f57d5f6-m9mm9
kuchima@cloudshell:~/asm-multicluster (kuchima002)$ kubectl exec -n sample -c sleep     "$(kubectl get pod -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')"     -- curl -sS helloworld.sample:5000/hello
Hello version: v1, instance: helloworld-v1-776f57d5f6-m9mm9
kuchima@cloudshell:~/asm-multicluster (kuchima002)$ kubectl exec -n sample -c sleep     "$(kubectl get pod -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')"     -- curl -sS helloworld.sample:5000/hello
Hello version: v1, instance: helloworld-v1-776f57d5f6-m9mm9

→ちゃんと Fail Over した

```bash
cat << EOF > dr-distribution.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld.sample.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      localityLbSetting:
        enabled: true
        distribute:
        - from: asia-northeast1/*
          to:
            "asia-northeast1/*": 50
            "asia-northeast2/*": 50
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 1m
EOF


kubectl apply -f dr-distribution.yaml -n sample
```

### Node のリージョン/ゾーンを確認
```bash
$ kubectl describe nodes gke-gke-osaka-default-pool-a2626c2f-05m4
Name:               gke-gke-osaka-default-pool-a2626c2f-05m4
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=e2-medium
                    beta.kubernetes.io/os=linux
                    cloud.google.com/gke-boot-disk=pd-standard
                    cloud.google.com/gke-container-runtime=containerd
                    cloud.google.com/gke-nodepool=default-pool
                    cloud.google.com/gke-os-distribution=cos
                    cloud.google.com/machine-family=e2
                    failure-domain.beta.kubernetes.io/region=asia-northeast2
                    failure-domain.beta.kubernetes.io/zone=asia-northeast2-a
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=gke-gke-osaka-default-pool-a2626c2f-05m4
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=e2-medium
                    topology.gke.io/zone=asia-northeast2-a
                    topology.kubernetes.io/region=asia-northeast2
                    topology.kubernetes.io/zone=asia-northeast2-a
```

Distribute の割合に応じて振り分け。0:100% で振り分けたときに v1 側を落とすと、v2 側に Failover する動きになる
kuchima@cloudshell:~/asm-multicluster$ kubectl get pods -n sample
NAME                             READY   STATUS    RESTARTS   AGE
helloworld-v1-776f57d5f6-m9mm9   1/2     Running   0          15h
sleep-557747455f-qct2x           2/2     Running   0          15h
kuchima@cloudshell:~/asm-multicluster$ kubectl exec -n sample -c sleep     "$(kubectl get pod -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')"     -- curl -sS helloworld.sample:5000/hello
Hello version: v2, instance: helloworld-v2-54df5f84b-bpnb8
kuchima@cloudshell:~/asm-multicluster$ kubectl exec -n sample -c sleep     "$(kubectl get pod -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')"     -- curl -sS helloworld.sample:5000/hello
Hello version: v2, instance: helloworld-v2-54df5f84b-bpnb8
kuchima@cloudshell:~/asm-multicluster$ kubectl exec -n sample -c sleep     "$(kubectl get pod -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')"     -- curl -sS helloworld.sample:5000/hello
Hello version: v2, instance: helloworld-v2-54df5f84b-bpnb8

他クラスタ Pod への Egress を Network Policy で封じてみる
```bash
cat << EOF > disallow-gkeosaka.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: 
    matchLabels:
      app: sleep
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.52.0.0/14 # gke-tokyo の Pod IP range
EOF

kubectl apply -f disallow-gkeosaka.yaml -n sample
```
→ Network Policy は既存コネクションを止められない
→ VPC FW でノードプール単位で止めるのはOK


kubectl exec -n sample -c sleep     "$(kubectl get pod -n sample -l app=sleep -o jsonpath='{.items[0].metadata.name}')"     -- curl -sS helloworld.sample:5000/hello




```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld-subsets
spec:
  host: helloworld.sample.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
  - helloworld.sample.svc.cluster.local
  http:
  - route:
    - destination:
        host: helloworld.sample.svc.cluster.local
        subset: v1
      weight: 100
    - destination:
        host: helloworld.sample.svc.cluster.local
        subset: v2
      weight: 0
```

```bash
$ kubectl exec -n sample -c sleep     "$(kubectl get pod -n sample -l app=sleep -o jsonpath='{.items[0].metadata.name}')"     -- curl -sS helloworld.sample:5000/hello
no healthy upstream
```
→ Locality Balancing とは違って、負荷分散先に weight:0 のサービスを設定しない動き
### （参考）MCSを有効にした場合のローカルクラスタアクセス
1.12 からは MCS 有効化時において cluster.local にアクセスすると、ローカルクラスタのみにアクセスするようになるらしい