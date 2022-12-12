---
title: "これから始める GKE (基本編)"
emoji: "🦄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP, GoogleCloud, GKE]
publication_name: "google_cloud_jp"
published: false
---
この記事は [Google Cloud Japan Advent Calendar 2022 (今から始める Google Cloud)](https://zenn.dev/google_cloud_jp/articles/12bd83cd5b3370#%E4%BB%8A%E3%81%8B%E3%82%89%E5%A7%8B%E3%82%81%E3%82%8B-google-cloud) の 6 日目(だったはず)の記事です。
`今から始める Google Cloud` ということで、これから [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine) (以降 GKE) を使っていこうと考えられている方向けに GKE の基本的な特徴や他の Google Cloud のアプリケーションプラットフォームとの違い・使い分けなどについて紹介しようと思います。

ちなみにこの「これから始める GKE」はシリーズ化しようと思っていますので、今後は設計に関する考慮ポイントなど細かめな話も書いていく予定です。

# tl;dr
* GKE は運用を楽にするための各種自動化機能を持っています。また、GKE には多くの周辺エコシステム・マネージドサービスが存在します。
* とにかく K8s を楽に運用したい方は GKE Autopilot がおすすめです。一方、細かなチューニングを必要とする場合やバースト性のあるワークロードを動かす場合など Autopilot が苦手な特性のワークロードを動かす予定がある場合は、GKE Standard がおすすめです。
* GKE を管理する専任チームがいなかったり Kubernetes の有識者があまりいないような環境であれば Cloud Run の利用もご検討ください。Cloud Run では要件的に合わない場合や Kuberentes の豊富なエコシステムを活用して柔軟なプラットフォームを構築・運用していきたい方には GKE が合うのではないかと思います。

# GKE とは
GKE は Google Cloud が提供するフルマネージドな Kubernetes プラットフォームです。Kubernetes をより簡単かつ安全に使っていただくための機能を多く持っています。
GKE は大きく Control Plane と Node というコンポーネントに分かれています。そのうち Control Plane は Google が管理しており、利用者側で Control Plane の運用 (アップグレードやセキュリティ対策、スケール等) をしていただく必要はありません。
一方 Node は 利用者側で管理するもしくは Google で管理するという以下 2 つのオプションから選ぶことができます。
* **GKE Standard** ... Control Plane は Google 管理、**Node はユーザー管理**
* **GKE Autopilot** ... Control Plane は Google 管理、**Node も Google 管理**   

※GKE Standard と Autopilot の違いについては後程説明します
ではここから、GKE の特徴について簡単にご紹介していきます。
## 特徴① 運用負荷を軽減するための各種自動化機能
まず一つ目の特徴として、GKE は Kuberentes クラスタをより簡単に運用するための各種自動化機能を持っています。

### 自動アップグレード
前述の通り、GKE の Control Plane は自動的にアップグレードされるので、そもそも利用者側でアップグレード作業をしていただく必要はありません。
一方 Node については手動でアップグレードすることもできますが、[リリースチャンネル](https://cloud.google.com/kubernetes-engine/docs/concepts/release-channels)という仕組みを使って自動的にアップグレードにすることもできます (Autopilot は自動アップグレードのみ)。

リリースチャンネルとは、バージョニングとアップグレードを行う際のベストプラクティスを提供する仕組みです。リリースチャンネルにクラスタを登録すると、 Google により Control Plane だけでなく Node のバージョンとアップグレードサイクルも自動的に管理されるようになります。
リリースチャンネルは、利用できる機能 (GKE バージョン) や安定性の異なる以下３種類のチャンネルを提供しています。
* **Rapid** ... 最新のバージョンが利用可能。検証目的での利用を推奨 (SLA 対象外)
* **Regular** ... 機能の可用性とリリースの安定性のバランスが取れている
* **Stable** ... 最も安定したバージョンを提供。新機能よりも安定性を優先するケースで採用

自動アップグレードは便利だけどクラスタが勝手にアップグレードされると困るという方もいらっしゃるかと思います。
そこで GKE では自動アップグレードのタイミングをコントロールする仕組みも提供しています。[メンテナンスの時間枠](https://cloud.google.com/kubernetes-engine/docs/concepts/maintenance-windows-and-exclusions#maintenance_windows)という設定をしていただくことで、アップグレードなど自動メンテナンスを**許可**する時間枠を設定することができます。また[メンテナンスの除外](https://cloud.google.com/kubernetes-engine/docs/concepts/maintenance-windows-and-exclusions#exclusions)により、自動メンテナンスを**禁止**する期間を設定することができます。
* **メンテナンスの時間枠** ... 自動メンテナンスを許可する時間枠を設定。営業時間・ピーク時間帯を避けてアップグレードしてほしいケースやオンコール対応のしやすさなどから日中の業務時間帯にアップグレードをしてほしいケース等で利用。
* **メンテナンスの除外** ... 自動メンテナンスを許可する時間枠を設定。大型イベント期間中や人手が足りない時期などアップグレードをしてほしくない期間を設定するケース等で利用。マイナーバージョンのアップグレード (例：1.22 -> 1.23) を最大 180 日間[^1]停止することも。

[^1]: リリースチャンネルに登録されているクラスタが対象。また、マイナーバージョンの EOL を超過することはできません。

その他、GKE には今後のマイナー バージョンでサポートされない Kubernetes API や構成がクラスタで利用されていることを自動的に検出し通知する機能も提供しています (Preview)。この機能を活用いただくことにより、例えば次のマイナーバージョンで削除予定の API が利用されているクラスタの自動アップグレードを自動的に一時停止し、アップグレードによる影響を抑えることができます。
本機能は 2022.12 現在、以下３種類の Deprication を検出することが可能です。

![]()

Node のアップグレードタイミングは自分でコントロールしたいという場合は手動アップグレードを選択いただく形になります。
手動アップグレードだからといって手間がかかるというものでもなく、Node Pool 単位で `gcloud ` 以下 ２ 種類のアップグレード方式を以下2つから選択することができます。要件に応じて選択することが可能です。(自動アップグレードの場合はサージアップグレードが利用されます)
* サージアップグレード... デフォルトのアップグレード戦略。ノードを順次新しいバージョンにローリングアップデートする。
* Blue / Green アップグレード... 新旧2つのバージョンを保持したままアップグレードを行う。サージアップグレードに比べてロールバックが高速だが、アップグレード中の必要リソースが多い。

![2つのアップグレード戦略のイメージ]()

ちなみに GKE に元から入っているようなシステム系コンポーネントは GKE の Control Plane / Node のアップグレードとあわせて自動的にアップグレードされるため手動対応は不要です。

### 自動スケール
![GKE がサポートする自動スケール機能](/images/gke-korekara-101/autoscaling.png)

### 自動修復

## 特徴② 幅広い周辺サービス・エコシステム
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


一方、Autopilot は、2022年に入ってからも多く機能のアップデートが入っており、例えば、以下のようにより多くのユースケースをサポートできるようになりました。
* ASM (Managed Control Plane) が GA に
* GPU が利用可能に
* その他利用可能なソフトウェアの制限が緩和


# Cloud Run との違い、使い分け
ステートフルなワークロード、Serverless VPC access connector 、サービスディスカバリー、Pull ベースの CDなど K8s エコシステム、Operator を使った運用自動化など

# Anthos と GKE の関係性
Anthos 自体は様々なサービスの総称・ブランドのようなもので、Anthos = ハイブリッド・マルチクラウドという訳ではないです。GKE で利用する分には

# まとめ