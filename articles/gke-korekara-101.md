---
title: "これから始める GKE (基本編)"
emoji: "🦄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP, GoogleCloud, GKE]
publication_name: "google_cloud_jp"
published: false
---
この記事は [Google Cloud Japan Advent Calendar 2022 (今から始める Google Cloud)](https://zenn.dev/google_cloud_jp/articles/12bd83cd5b3370#%E4%BB%8A%E3%81%8B%E3%82%89%E5%A7%8B%E3%82%81%E3%82%8B-google-cloud) の 6 日目(だったはず)の記事です。
`今から始める Google Cloud` ということで、これから [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine) (以降 GKE) を使っていこうと考えられている方向けに GKE の基本的な特徴をご紹介しようと思います。本記事では Kubernetes 自体のベーシックな話は割愛し、既に Kubernetes の基本的な内容を理解されている方向けに GKE の良さをお伝えできればと思っています。
ちなみにこの「これから始める GKE」はシリーズ化しようと思っています。今後は設計に関する考慮ポイントなど細かめな話も書いていく予定です。

# tl;dr
* GKE は運用を楽にするための各種自動化機能を持っています。また、GKE には多くの周辺エコシステム・マネージドサービスが存在します。
* とにかく K8s を楽に運用したい方は GKE Autopilot がおすすめです。一方、細かなチューニングを必要とする場合やバースト性のあるワークロードを動かす場合など Autopilot が苦手な特性のワークロードを動かす予定がある場合は、GKE Standard がおすすめです。

# GKE とは　備忘：NodePoolの話を追加
GKE は Google Cloud が提供するフルマネージドな Kubernetes プラットフォームです。Kubernetes をより簡単かつ安全に使っていただくための機能を多く持っています。
GKE は大きく Control Plane と Node というコンポーネントに分かれています。そのうち Control Plane は Google が管理しており、利用者側で Control Plane の運用 (アップグレードやセキュリティ対策、スケール等) をしていただく必要はありません。
一方 Node は 利用者側で管理するもしくは Google で管理するという以下 2 つのモードから選ぶことができます。
* **GKE Standard** ... Control Plane は Google 管理、**Node はユーザー管理**
* **GKE Autopilot** ... Control Plane は Google 管理、**Node も Google 管理**   

GKE Standard と Autopilot の違いについては後程説明します。一先ずざっくりと、通常 (従来) の GKE クラスタが GKE Standard で、より簡単に運用できるようマネージド度合いが上がったものが GKE Autopilot と考えていただければ大丈夫です。
ではここから、GKE の特徴について簡単にご紹介していきます。

## 特徴① 運用負荷を軽減するための各種自動化機能
まず一つ目の特徴として、GKE は以下のような Kuberentes クラスタをより簡単に運用するための各種自動化機能を持っているというのが挙げられます。
* 自動アップグレード
* 自動スケール
* 自動修復

### 自動アップグレード

#### リリースチャンネル
前述の通り、GKE の Control Plane は自動的にアップグレードされるので、そもそも利用者側でアップグレード作業をしていただく必要はありません。
一方 Node については手動でアップグレードすることもできますが、[リリースチャンネル](https://cloud.google.com/kubernetes-engine/docs/concepts/release-channels)という仕組みを使って自動的にアップグレードにすることもできます (Autopilot は自動アップグレードのみ利用可能です)。

リリースチャンネルとは、バージョニングとアップグレードを行う際のベストプラクティスを提供する仕組みです。リリースチャンネルにクラスタを登録すると、 Google により Control Plane だけでなく Node のバージョンとアップグレードサイクルも自動的に管理されるようになります。
リリースチャンネルは、利用できる機能 (GKE バージョン) や安定性の異なる以下３種類のチャンネルを提供しています。利用環境や求められる安定性等に応じて使い分けることが可能です。
* **Rapid** ... 最新のバージョンが利用可能。検証目的での利用を推奨 (SLA 対象外)
* **Regular** ... 機能の可用性とリリースの安定性のバランスが取れている
* **Stable** ... 最も安定したバージョンを提供。新機能よりも安定性を優先するケースで採用

#### 自動アップグレードタイミングの制御
自動アップグレード自体は便利だけどクラスタが勝手にアップグレードされると困る、という方もいらっしゃるかと思います。
そこで GKE では自動アップグレードのタイミングをコントロールする仕組みも提供しています。[メンテナンスの時間枠](https://cloud.google.com/kubernetes-engine/docs/concepts/maintenance-windows-and-exclusions#maintenance_windows)という設定をしていただくことで、アップグレードなど自動メンテナンスを**許可**する時間枠を設定することができます。また[メンテナンスの除外](https://cloud.google.com/kubernetes-engine/docs/concepts/maintenance-windows-and-exclusions#exclusions)により、自動メンテナンスを**禁止**する期間を設定することができます。
* **メンテナンスの時間枠** ... 自動メンテナンスを許可する時間枠を設定。営業時間・ピーク時間帯を避けてアップグレードしてほしいケースやオンコール対応のしやすさなどから日中の業務時間帯にアップグレードをしてほしいケース等で利用。
* **メンテナンスの除外** ... 自動メンテナンスを許可する時間枠を設定。大型イベント期間中や人手が足りない時期などアップグレードをしてほしくない期間を設定するケース等で利用。マイナーバージョンのアップグレード (例：1.22 -> 1.23) を最大 180 日間[^1]停止することも。

[^1]: リリースチャンネルに登録されているクラスタが対象。また、マイナーバージョンの EOL を超過することはできません。

#### アップグレードに影響のある構成 / API 利用の検知
その他、GKE には今後のマイナー バージョンでサポートされない構成や Kubernetes API がクラスタで利用されていることを自動的に検出し通知する [Deprecation Insights](https://cloud.google.com/kubernetes-engine/docs/deprecations/viewing-deprecation-insights-and-recommendations) という機能も提供しています (Preview)。
2022.12 現在 Deprecation Insights では以下３種類の Deprication を検出することが可能です。ちなみにこれら Deprecated な構成や API の利用が検知されるとマイナーバージョンの自動アップグレードが一時停止されるので、知らないうちにクラスタがアップグレードされて手元のマニフェストと互換性がなくなっていた。。的なトラブルの発生を防ぐことが期待できます。(但し各[マイナーバージョンの EOL](https://cloud.google.com/kubernetes-engine/versioning#lifecycle) を過ぎた場合は強制的にアップグレードされてしまうので注意)
* 1.22 で削除予定の Kubernetes API の利用
* 1.23 でサポートされなくなる SAN の無い X.509 証明書の利用
* 1.24 でサポートされなくなる Docker ベースのノードイメージの利用

#### Node 手動アップグレードのオプション
ここまでは自動アップグレードの話を中心にしてきましたが、前述の通り、Node のアップグレードタイミングは自分でコントロールしたいという場合は手動アップグレードを選択することも可能です。
手動アップグレードといっても一連の Node アップグレード作業は自動的に実施されます。アップグレードのトリガーを利用者側でコントロールできるというのがポイントになります。
Node の手動アップグレードは Node Pool 単位で以下 2 種類のアップグレード方式から選択可能です。(Release Channel など自動アップグレード利用の場合はサージアップグレードが選択されます)
* サージアップグレード... デフォルトのアップグレード戦略。ノードを順次新しいバージョンにローリングアップデートする。
* Blue / Green アップグレード... 新旧2つのバージョンを保持したままアップグレードを行う。サージアップグレードに比べてロールバックが高速だが、アップグレード中に必要となるリソースが多い。

![2つのアップグレード戦略のイメージ]()
Blue / Green アップグレードは何か問題が発生したときのロールバックが迅速に行えるという利点がありますが、一時的に Node Pool 内のリソースが 2 倍になります。この辺りに懸念がある場合はサージアップグレードを選択いただくのが良いのではないかと思います。

ここまででざっくりと GKE の自動アップグレードにおける特徴を説明しました。Kubernetes クラスタの運用のなかでも最も大変なのではないかと思うアップグレード作業の負荷を軽減してくれる機能が豊富なのが GKE の大きな特徴の 1 つなのではないかと思います。
ちなみに GKE に元から入っているようなシステム系コンポーネントは GKE の Control Plane / Node のアップグレードとあわせて自動的にアップグレードされるので、コンポーネントごとに個別でアップグレードしていただく必要はありません。

### 自動スケール
GKE では以下5種類の Pod / Node のスケール機能を提供しています。Kuberentes として元々備わっているスケーリング機能もありますが、GKE が独自に拡張したような機能も存在します。本記事では特に特徴的な、Multidimensional Pod Autoscaler (MPA) と Node Auto-Provisioning (NAP) を紹介します。
![GKE がサポートする自動スケール機能](/images/gke-korekara-101/autoscaling.png)

#### Multidimensional Pod Autoscaler (MPA) 
MPA は [Horizontal Pod Autoscaler (HPA)](https://cloud.google.com/kubernetes-engine/docs/concepts/horizontalpodautoscaler) というワークロードの CPU やメモリの消費量等に応じて自動的に Pod 数を水平方向にスケールさせる機能と [Vertical Pod Autoscaler (VPA)](https://cloud.google.com/kubernetes-engine/docs/concepts/verticalpodautoscaler) という実行中のワークロードを分析し、CPU やメモリの requests / limits 値の推奨値算出やリソース値を自動更新し垂直スケールさせる機能を**併用**したオートスケール機能です (Preview)。本来、CPU やメモリの値をターゲットにした HPA と VPA の併用は非推奨となっていますが、MPA を利用することでこの2つを安全に併用することができます。
MPA 自体は CPU ベースの HPA とメモリベースの VPA を組み合わせたオートスケールを 1 つのオブジェクト (MultidimPodAutoscaler) で制御しています。MPA を活用することで **CPU の利用率ベースで Pod 数を調整しつつ OOM が発生しないよう VPA によるメモリの垂直スケールも併せて行う**ことが可能になっています。

#### Node Auto-Provisioning (NAP)
NAP は [Cluster Autoscaler (CA)](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-autoscaler) の仕組みの 1 つで、ワークロードにフィットする Node Pool を自動的に作成・削除する機能です。CA は既存の Node Pool に対して設定し、ワークロードの需要に基づいて特定 Node Pool 内の (特定のマシンサイズの) Node 数を増減させてくれますが、NAP を使うとスケジューリングされた Pod のサイズに応じて適切なマシンサイズの Node Pool をプロビジョニングしてくれるようになります。
これにより、ノードのサイジングを含めたリソース管理を GKE に任せることができ、またワークロードに合ったサイズのマシンがプロビジョニングされるためインフラリソースを効率的に活用することが可能になります。ちなみに GKE Autopilot では NAP を利用し Pod に必要な Node を自動的にプロビジョニングしています。

### 自動修復
GKE では Node が Unhealthy な状態になっていることを検知し、自動的に修復プロセスを開始する [Node Auto Repair](https://cloud.google.com/kubernetes-engine/docs/how-to/node-auto-repair) という機能を提供しています。
例えば以下のような状態の場合に Node が Unhealthy と判断され、対象 Node が Drain & 再作成されます。
* 約 10 分間、Node が NotReady ステータスを報告
* 約 10 分間、Node がステータスを何も報告しない
* 長期間（約 30 分間）、DiskPressure ステータスを報告

この機能はデフォルト有効です(Autopilot やリリースチャンネルに登録されているクラスタでは無効化できません)。
利用者側で Node 自体の健全性の確認やその復旧作業を実施する負荷が軽減できるため、特に理由がなければそのまま有効にしておくべき機能だと思います。

## 特徴② 高いスケーラビリティ
続いて、GKE の特徴として高いスケーラビリティを持っているというのが挙げられます。
例えば GKE Standard のクラスタあたりの最大ノード数は [15,000](https://cloud.google.com/kubernetes-engine/quotas#limits_per_cluster) です。(とはいえここまで大きなクラスタを実際に作るのはあまりお勧めしないので、数百数千 Node 以上の規模になる場合は一度 Google Cloud のカスタマーエンジニアにアーキテクチャやキャパシティのご相談いただくのが良いかと思います)

### Pod / Service IP レンジ
GKE では[Pod IP アドレス範囲を後から追加](https://cloud.google.com/kubernetes-engine/docs/how-to/multi-pod-cidr)することができます。
これにより、サービスの成長に伴い当初の想定より多くの Pod をデプロイする必要がある場合など、追加の Pod CIDR を設定し IP アドレスレンジの枯渇を防ぐことが可能です。また、クラスタ構築時や初期の設計段階で余分な IP アドレスを割り振る必要がなくなります。

![Multi-Pod CIDR](/images/gke-korekara-101/multi-pod-cidr.png)

また、[Service リソース用のセカンダリ IP アドレスレンジをクラスタ間で共有](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips#share_secondary_services)可能です。Service の IP アドレスレンジをクラスタ間で共有することにより、複数クラスタを利用されている環境などで VPC 内の IP アドレスの節約に貢献できます。

### クラスタ内 DNS
クラスタ内 DNS のスケーラビリティという観点では、[Cloud DNS for GKE](https://cloud.google.com/kubernetes-engine/docs/how-to/cloud-dns) という構成オプションを利用することにより、従来の `kube-dns` の代わりに [Cloud DNS](https://cloud.google.com/dns) を使った名前解決が可能になります。クラスタ内のコンポーネントではなくマネージドサービスを利用することにより、より高いスケーラビリティが期待できます。
![Cloud DNS for GKE](/images/gke-korekara-101/clouddns-gke.png)

また、[NodeLocal DNSCache](https://cloud.google.com/kubernetes-engine/docs/how-to/nodelocal-dns-cache) というアドオンを有効化することで各 Node 上で DNS キャッシュを提供することができます。ちなみに NodeLocal DNSCache は先ほど紹介した Cloud DNS for GKE と併用することも可能です。これにより Pod から送られる名前解決のリクエストを、まずその Pod が動いている Node 上のキャッシュで解決しようとするため、クラスタ内の kube-dns や Cloud DNS に対するクエリ発行を抑えることにより DNS 側(特に kube-dns)の負荷の低減やレイテンシーを小さくすることが期待できます。

### マルチクラスタ間での負荷分散
1 クラスタあたりのリソース上限の観点やリージョン毎にクラスタを構築する場合など、結果としてマルチクラスタ構成となるケースもあるかと思います。(スケーラビリティの話とはちょっと違う観点も含まれてますが...)
GKE では、複数クラスタ間での North-South トラフィック (クライアント等からのトラフィック) 分散を実現する方法として、以下 2 種類の機能を提供しています。
* [Multi-cluster Ingress (MCI)](https://cloud.google.com/kubernetes-engine/docs/concepts/multi-cluster-ingress) (GA)
* [Multi-cluster Gateway (MCG)](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-multi-cluster-gateways) (Preview)

MCI と MCG の大きな違いとしてはベースとなっているリソースが Ingress か Gateway かという部分がまずあります。Ingress と Gateway の細かな違いには今回触れませんので、気になる方は[公式ドキュメント](https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api#ingress_and_gateway)や同僚の [Kazuu さん解説記事](https://medium.com/google-cloud-jp/gke-gateway-4150649d8c37)も参照してみてください。
その他、MCI と MCG の違いとしては以下があります。
* 2022.12 現在、MCI は GA ステータスだが、MCG はまだ Preview 
* MCG では Internal LB を使ったマルチクラスタ負荷分散をサポートしている
* MCG では[複数クラスタ間での重み付けルーティングやヘッダベースのルーティング等高度なトラフィックコントロール](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-multi-cluster-gateways#blue-green)をサポート
* MCG では[キャパシティベースのルーティング](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-multi-cluster-gateways#capacity-load-balancing)をサポート

大きな違いとしては、MCI が GA で MCG はまだ Preview であるという点です。本番環境などであれば MCI を利用するのが安全なのではないかと思います。一方、上記の通り MCG でしか実現できない機能も多くありますので、MGC が GA になると本番環境での採用事例も増えてくるのではないかと思います。
ちなみに MCG のキャパシティベースのルーティングを使うと RPS ベースの負荷分散が実現でき、例えばとあるクラスタのキャパシティ(許容可能な RPS)を超えたリクエストがきた場合にもう片方のクラスタに自動的にオフロードするという構成をとることもできます。
![Multi-cluster Gateway のキャパシティベースのルーティング](/images/gke-korekara-101/multi-cluster-capacitylb.png)

## 特徴③ 幅広い周辺サービス・エコシステム
GKE には多くの周辺サービス、エコシステムが存在します。
例えば、Kubernetes エコシステムのマネージドサービスとして下記を提供しています。
* Prometheus のマネージドサービスである [Google Cloud Managed Service for Prometheus (GMP)](https://cloud.google.com/managed-prometheus)
* Istio のマネージドサービスである [Anthos Service Mesh (ASM)](https://cloud.google.com/anthos/service-mesh)
* OPA Gatekeeper のマネージドサービスである [Policy Controller](https://cloud.google.com/anthos-config-management/docs/concepts/policy-controller)
* Knative のマネージドサービスである [Cloud Run for Anthos](https://cloud.google.com/anthos/run/docs/deploy-application)
Kubernetes 運用の大変なところの 1 つとして、K8s エコシステムの運用があると思うのですが、上記のようなマネージドサービスを活用することによって、アップグレードやセキュリティ対応、可用性の担保、リソース管理などいわゆる Day2 Operation の負荷を軽減することができます。

その他セキュリティの観点でも [Binary Authorization](https://cloud.google.com/binary-authorization/docs/overview) というコンテナイメージの脆弱性をスキャンしてくれるサービスや Security Command Center の [Container Threat Detection](https://cloud.google.com/security-command-center/docs/concepts-container-threat-detection-overview) という実行中のコンテナに対する振る舞い検知機能を提供するサービスなど、多くのサービスが存在します。以下、サービスの例です。

* Container Analysis... コンテナイメージの脆弱性をスキャンしてくれるサービス。Artifact Registry や Container Registry といった Google Cloud マネージドのコンテナレジストリ上に Push されたイメージを自動的にスキャンしたり、オンデマンドでコンテナレジストリやローカルにあるイメージをスキャンすることができる。
* Binary Authorization... 信頼できるコンテナイメージのみが GKE クラスタにデプロイされることを保証してくれるサービス。「信頼できないリポジトリが提供しているベースイメージやコンテナイメージにマルウェアや脆弱性が含まれている」リスクや「CI / CD をバイパスした有害なコンテナイメージがデプロイされる」リスクを低減することができる。
* Container Threat Detection... 実行中のコンテナに対する振る舞い検知機能を提供するサービス。元のコンテナ イメージに存在しなかったバイナリ (マルウェアやクリプトマイナー等) の実行やリバースシェル実行 (ボットネット) のような異常な挙動・攻撃を検出することができる。
* GKE Security Posture... GKE 上で動くワークロードの K8s 構成や脆弱性を継続的にチェックしてくれるサービスです。例えばワークロードの構成スキャンを有効化すると、

GKE のセキュリティ機能やセキュリティ関連サービスについては以下の記事でまとめていますので、ご興味あれば見てみてください。
https://medium.com/google-cloud-jp/gkesecurity-2022-1-ea4d55bcf4f7

その他、[Backup for GKE](https://cloud.google.com/kubernetes-engine/docs/add-on/backup-for-gke/concepts/backup-for-gke) という GKE 上のワークロードのバックアップ・リストアを行ってくれるマネージドサービスもあります。クラスタ全体やステートフルなワークロードの保護、またクラスタを移行する際などに役立つサービスです。

# GKE Standard と Autopilot の違い
GKE には現在 GKE Standard と GKE Autopilot という 2 種類のモードが存在します。
GKE Standard との違いなどの詳細については同僚の [Kazuu-san の記事](https://medium.com/google-cloud-jp/gke-autopilot-87f8458ccf74)もあわせて読んでみてください。ここではもう少しアバウトな違いに言及していきます。

GKE Standard は従来の GKE クラスタであり、Control Plane は Google 管理ですが Node はユーザー管理となっています。一方 GKE Autopilot は Control Plane だけでなく Node も Google 管理になっています。`Google 管理`が具体的に何を指しているかというと
* Node のサイズや台数調整などリソースの管理が不要です (CA および NAP による Node の自動スケール)
* Node のアップグレード作業が不要です (Release Channel による自動的なバージョン管理)
* Unhealthy な Node を自動的に修復します (Node Auto Repair による自動修復)
* Node は利用者から見えません (`kubectl get nodes` 等で確認することはできる)
* Node リソースに対する課金はされず、Pod 単位の課金となります (システム系 Pod のリソースや Node 上の余剰リソースに対して課金が発生しません)

また GKE Autopilot は各種セキュリティのベストプラクティスが組み込みで実装されており、Production Ready なクラスタをすぐに利用することができます。
* Workload Identity 有効化 (無効化不可)
* Shieleded GKE Node / セキュアブート有効化 (無効化不可)
* Container-Optimized OS (COS) containerd イメージのみ利用可能
* Node への SSH 不可
* 特権コンテナのデプロイ不可
これらはセキュリティ面での利点でもありつつ、制約とも言える内容になっているかと思います。ワークロードがこれら制約を許容できるか、というのも Autopilot を選ぶ上で重要なポイントです。
その他セキュリティ面での特徴については[公式ドキュメント](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-security)もご参照ください。

また、Autopilot は Node リソースを Cluster Autoscaler / Node Auto Provisioning により自動的に管理しているため、Node 上に余剰リソースがあまりなく、新しい Pod をデプロイすると Node のプロビジョニングが走ることが多いため、突発的にスケールするようなワークロードにはあまり向いていません。そのようなワークロードの場合は GKE Standard を利用し、あらかじめスケールに耐えうる分の Node をプロビジョニングしておくなどの方法を取るのも良いではないかと思います。(もしくは GKE Autopilot でも[予め Priority の低い Pod をデプロイしておき余剰リソースを確保する](https://wdenniss.com/gke-autopilot-spare-capacity)という方法を取ることはできます)

その他、Autopilot は Standard に比べてデプロイ可能なワークロードに制限があったりするのですが、今年も多くの機能アップデートが入っており、例えば、以下のようにより多くのユースケースをサポートできるようになりました。
* Autopilot 上での ASM (Managed Control Plane) サポートが GA に
* GPU が利用可能に
* その他利用可能なソフトウェアの制限が緩和

# まとめ