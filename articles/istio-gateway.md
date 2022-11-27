---
title: "Istio+Gateway API を試してみる"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
# はじめに
https://istio.io/latest/blog/2022/gateway-api-beta/

Istio 1.14 Kubernetes 1.2
より抽象化が

# tl;dr

# Gateway API とは

# Istio リソースと Gateway API リソースのマッピング
違いが知りたい、今対応できているリソースとできていないもの


どこまで抽象化してくれるのか？
→自動デプロイ対象リソースを確認する

# 実際にためしてみる
External Internal 
L7LB で expose してみる

```bash
$ kubectl get crd gateways.gateway.networking.k8s.io
Error from server (NotFound): customresourcedefinitions.apiextensions.k8s.io "gateways.gateway.networking.k8s.io" not found

$ kubectl apply -k "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v0.5.0"
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created

$ kubectl get crd gateways.gateway.networking.k8s.io
NAME                                 CREATED AT
gateways.gateway.networking.k8s.io   2022-08-21T11:15:42Z
```

```bash
$ curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.14.3 TARGET_ARCH=x86_64 sh -
$ cd istio-1.14.3
$ export PATH=$PWD/bin:$PATH
$ istioctl install --set profile=minimal -y
✔ Istio core installed                
✔ Istiod installed                                                           
✔ Installation complete                                                         Making this installation the defaultfor injection and validation.

Thank you for installing Istio 1.14.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/yEtCbt45FZ3VoDT5A

$ kubectl get gatewayclass
NAME          CONTROLLER                    ACCEPTED   AGE
gke-l7-gxlb   networking.gke.io/gateway     True       41m
gke-l7-rilb   networking.gke.io/gateway     True       41m
istio         istio.io/gateway-controller   True       92s
```

```bash
kubectl apply -f samples/httpbin/httpbin.yaml
kubectl create namespace istio-ingress

```

```yaml:gateway-istio.yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: Gateway
metadata:
  name: gateway
  namespace: istio-ingress
spec:
  gatewayClassName: istio
  listeners:
  - name: default
    hostname: "*.example.com"
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
```

```bash
$ kubectl apply -f gateway-istio.yaml
gateway.gateway.networking.k8s.io/gateway created

$ kubectl get gateway -n istio-ingress
NAME      CLASS   ADDRESS                                      READY   AGE
gateway   istio   gateway.istio-ingress.svc.cluster.local:80   True    48s

# Ingress Gateway が自動的にデプロイされている
$ kubectl get deploy -n istio-ingress
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
gateway   1/1     1            1           4m10s

# Istio の Gateway リソースはデプロイされていない
$ kubectl get gw -n istio-ingress
No resources found in istio-ingress namespace.
```

```yaml
apiVersion: v1
items:
- apiVersion: apps/v1
  kind: Deployment
  metadata:
    annotations:
      deployment.kubernetes.io/revision: "1"
    creationTimestamp: "2022-08-21T12:08:19Z"
    generation: 1
    labels:
      gateway.istio.io/managed: istio.io-gateway-controller
    name: gateway
    namespace: istio-ingress
    ownerReferences:
    - apiVersion: gateway.networking.k8s.io/v1alpha2
      kind: Gateway
      name: gateway
      uid: 2a047720-90d3-4a43-b577-4046278b5a12
    resourceVersion: "34906"
    uid: de09cf51-1e8a-4459-a568-ebc82de4ec72
  spec:
    progressDeadlineSeconds: 600
    replicas: 1
    revisionHistoryLimit: 10
    selector:
      matchLabels:
        istio.io/gateway-name: gateway
    strategy:
      rollingUpdate:
        maxSurge: 25%
        maxUnavailable: 25%
      type: RollingUpdate
    template:
      metadata:
        annotations:
          inject.istio.io/templates: gateway
        creationTimestamp: null
        labels:
          istio.io/gateway-name: gateway
          sidecar.istio.io/inject: "true"
      spec:
        containers:
        - image: auto
          imagePullPolicy: Always
          name: istio-proxy
          ports:
          - containerPort: 15021
            name: status-port
            protocol: TCP
          readinessProbe:
            failureThreshold: 10
            httpGet:
              path: /healthz/ready
              port: 15021
              scheme: HTTP
            periodSeconds: 2
            successThreshold: 1
            timeoutSeconds: 2
          resources: {}
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
              - ALL
            privileged: false
            readOnlyRootFilesystem: true
            runAsGroup: 1337
            runAsNonRoot: true
            runAsUser: 1337
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext:
          sysctls:
          - name: net.ipv4.ip_unprivileged_port_start
            value: "0"
        terminationGracePeriodSeconds: 30
```

```
$ kubectl get pods -n istio-ingress
NAME                       READY   STATUS    RESTARTS   AGE
gateway-57d696448d-8n5vv   1/1     Running   0          44m

$ kubectl get svc -n istio-ingress
NAME      TYPE           CLUSTER-IP   EXTERNAL-IP    PORT(S)                        AGE
gateway   LoadBalancer   10.8.1.200   34.84.65.158   15021:32553/TCP,80:31436/TCP   77m

$ istioctl pc listener gateway-57d696448d-8n5vv.istio-ingress
ADDRESS PORT  MATCH DESTINATION
0.0.0.0 80    ALL   Route: http.80
0.0.0.0 15021 ALL   Inline Route: /healthz/ready*
0.0.0.0 15090 ALL   Inline Route: /stats/prometheus*

$ istioctl pc route gateway-57d696448d-8n5vv.istio-ingress                 
NAME        DOMAINS     MATCH                  VIRTUAL SERVICE
http.80     *           /*                     404
            *           /stats/prometheus*     
            *           /healthz/ready* 
```

```yaml:httproute-istio.yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: HTTPRoute
metadata:
  name: http
  namespace: default
spec:
  parentRefs:
  - name: gateway
    namespace: istio-ingress
  hostnames: ["httpbin.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /get
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: my-added-header
          value: added-value
    backendRefs:
    - name: httpbin
      port: 8000
```

```bash
$ kubectl apply -f httproute-istio.yaml
$ export INGRESS_HOST=$(kubectl get gateways.gateway.networking.k8s.io gateway -n istio-ingress -ojsonpath='{.status.addresses[*].value}')

# アクセスしてみる (成功)
$ curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST/get"
HTTP/1.1 200 OK
server: istio-envoy
date: Sun, 21 Aug 2022 13:17:31 GMT
content-type: application/json
content-length: 2279
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 33

# 許可されていないパスへのアクセスは失敗する
$ curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST/headers"
HTTP/1.1 404 Not Found
date: Sun, 21 Aug 2022 13:24:20 GMT
server: istio-envoy
transfer-encoding: chunked
```

```bash
# Virtual Service が作られている
$ istioctl pc route gateway-57d696448d-8n5vv.istio-ingress
NAME        DOMAINS                 MATCH                  VIRTUAL SERVICE
http.80     httpbin.example.com                            http-0-istio-autogenerated-k8s-gateway.default
            *                       /stats/prometheus*     
            *                       /healthz/ready*

$ kubectl get virtualservice
No resources found in default namespace.

$ kubectl get virtualservice -A
No resources found
```


```
cloud.google.com/load-balancer-type: "Internal"

```

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: traffic-split
spec:
  rules:
  - forwardTo:
    - serviceName: store-v1
      port: 8080
      weight: 90
    - serviceName: store-v2
      port: 8080
      weight: 10
```

# 参考情報

https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/
https://gateway-api.sigs.k8s.io/
