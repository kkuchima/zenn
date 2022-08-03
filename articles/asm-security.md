---
title: "Istio / ASM のセキュリティ観点での考慮ポイント"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Istio, Anthos, GCP, Gatekeeper, OPA]
published: false
---
自分の知識の整理のため、Istio / ASM (Anthos Service Mesh) におけるセキュリティの考慮ポイントをまとめてみました。
アプリケーション コンテナや Kubernetes クラスタ関連のセキュリティの話は割愛しています。
ちなみに ASM (Anthos Service Mesh) とは Google Cloud が提供するフルマネージド版 Istio です。今回の記事では ASM の中でも in-cluster Control Plane というモードを前提に話を進めていきます。また、記事内で紹介するサンプルのマニフェストは 1.14 をベースにしています。
もし変なことを書いてたら教えてください。

# tl;dr
- 
- 意図しない / 誤った構成が適用されるのを防ぐために OPA Gatekeeper 等を使って Istio 構成に対するガードレールを設定するのもおすすめ



# メッシュ外との通信


Authentication Policy
Authorized Policy
-> deny all , dry-run

## 外部からメッシュ内の通信
クラウドサービスを使っているのであれば、WAF や DDoS 対策はマネージドサービスを活用するのが良いと思う

## メッシュ内から外部への通信

# メッシュ内の通信
## mTLS を STRICT mode で設定する
Man-in-the-middle を防ぐ
Istio では ver 1.6 以降では自動的に mTLS を有効化にしますが、デフォルトでは `Permissive mode` という mTLS で暗号化された通信だけでなく平文も許可するモードが設定されています。これだと Istio の sidecar が injection されていないクライアントからの通信も許可してしまい、mTLS による通信元 / 通信先の相互認証や通信の暗号化の恩恵を受けることができない可能性があるため、なるべく mTLS を強制させたいです。
その場合は mTLS を `STRICT mode` で設定します。以下のように `PeerAuthentication` で mtls のモードを `STRICT` に設定するだけで OK です。
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
  namespace: "istio-system"
spec:
  mtls:
    mode: STRICT
```
特定のネームスペース / Pod のみ mTLS を強制させることもできますが、可能であれば `namespace: "istio-system"` (istio-system が root namespace の場合) と設定し、メッシュ全体で強制する方が安全だと思います。特定のネームスペースのみを対象にするのであれば `namespace` を対象のものに変更します。

また、mTLS で利用可能な TLS バージョンを制限することもできますので必要に応じてこの辺りを設定しても良さそうです。
```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    meshMTLS:
      minProtocolVersion: TLSV1_3
```
https://preliminary.istio.io/latest/docs/tasks/security/tls-configuration/workload-min-tls-version/


ちなみに mTLS を使わず平文で通信を行なっているワークロードは Grafana Dashboard や ASM dashboard (ASM を使っている場合) で確認することも可能です。
https://istio.io/latest/docs/tasks/observability/metrics/using-istio-dashboard/

## Authorization Policy で default-deny なポリシーを設定する
Authorization Policy を設定することで、
default-deny を設定しておくと、Autorization Policy の穴を突いてアクセスされることを防ぐことが可能になります。
```yaml

```

## サイドカーの Injection を強制する
Istio でせっかく諸々のセキュリティポリシーを設定したものの、そもそものサイドカープロキシを含まない形でアプリケーションがデプロイされてしまうと、各種ポリシーがバイパスされてしまいます。そこで、Gatekeeper 等を使ってそもそもの Injection を強制させるのもおすすめです。
通常は Istio でサイドカープロキシを injection する場合は、対象の namespace に `` 等のラベルを追加しアプリケーションをデプロイすることでコントロールをすることが多いと思いますが、これだと Pod が annotation を上書きしてしまうことにより injection されないケースがあります。もちろん意図したものであれば良いのですが、意図しない場合や悪意を持って当該設定をされる可能性もあります。


## 例外設定をコントロールする
サイドカープロキシは正しく injection されたものの、Istio の設定によっては特定ポートをプロキシにキャプチャさせないように設定することもできます。
ここも穴になってしまう可能性があるため、可能であればコントロールしたいです。



## Authentication Policy
JWT 認証を適用すると、有効な認証情報なしで、実際のエンドユーザーに代わりサービスへのアクセスを試みる攻撃を阻止します。

Authentication / Authorization policy


# Workload
Istio CNI
Distroless proxy
-> 
mTLS 
-> Peer Authentication Policy

# Control Plane
istio-system に対する network policy, RBAC

# 不適切な設定を防ぐ
Gatekeeper

Restrict VS and DR tp use local namespaces
Restrict in-use subeset modification

# 脆弱性対策
脆弱性情報を集める
自動的にアップグレードする

# 参考資料
Security Best Practices
https://istio.io/latest/docs/ops/best-practices/security/

Anthos Service Mesh セキュリティのベスト プラクティス
https://cloud.google.com/service-mesh/docs/security/anthos-service-mesh-security-best-practices
