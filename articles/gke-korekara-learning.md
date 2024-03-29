---
title: "GKE をこれから学びたい・詳しくなりたい人向け！おすすめ学習リソース集"
emoji: "🕵️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP, GoogleCloud, GKE]
publication_name: "google_cloud_jp"
published: true
---
この記事は [Google Cloud Japan Advent Calendar 2023 (入門編)](https://zenn.dev/google_cloud_jp/articles/65eb509ce7dc91) の 3 日目の記事です。  

今回は [Google Cloud をゼロから学ぶならこれ! おすすめの学習リソース](https://zenn.dev/google_cloud_jp/articles/6a8a0e571841ef)という素敵記事に倣って [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/docs/concepts/kubernetes-engine-overview?hl=ja) をこれから学びたい・詳しくなりたい人向けにおすすめの学習リソース (無償のもの) を紹介します。  
初学者向けのリソースだけでなく、実際のクラスタ設計にも役立つリソースを紹介します！

# プロダクトの概要を知る
そもそも GKE って何？という方はプロダクトの概要を知ることから始めるのが良いかもしれません。  
概要を知るのに良いコンテンツをいくつか紹介します。  

## ブログ記事で学習する
世の中には GKE の概要を紹介したブログ記事が多く存在します。  
（初っ端から宣伝ですみませんが）一例として、私が以前書いた記事では、GKE のアーキテクチャや特徴のご紹介などをしています。もしご興味ありましたら読んでみてください。  
https://gihyo.jp/article/2023/09/modern-app-development-on-google-cloud-02

## YouTube 動画で学習する
YouTube にも GKE の解説動画が公開されているので、基本的な概要を知るのに役立ちます。  
例えば、英語では以下のように `GKE Essentials` というシリーズが公開されています。  
https://www.youtube.com/playlist?list=PLIivdWyY5sqLQ3m7WJDfBdMMqO12Q0vqg

また、日本語では `Google Cloud スタートアップ向けテクニカル ガイド` というシリーズの動画が短時間で情報がまとまっていて理解しやすいのではないかと思います。  
https://youtu.be/rEMDQf16Jdc?si=zI5FUSVtPGpii-vX

## App Modernization OnAir
（最近はあまり頻繁に更新されていませんが）`App Modernization OnAir` というウェビナープログラムもプロダクトカットで有用な情報がまとまっていて学習リソースとしておすすめです。  
https://cloudonair.withgoogle.com/events/solution-app-modernization

# 事例から学ぶ
どういう企業がどういう背景や構成で GKE を使っているかを知りたい場合、事例から学ぶのも良い方法だと思います。  

## Google Cloud の事例ページや公式ブログから事例を探す
GKE を実際に利用されているお客さんの事例を知りたい場合、まず確認すると良いのが Google Cloud の事例ページや公式ブログです。  

### 事例ページ
[Google Cloud をゼロから学ぶならこれ! おすすめの学習リソース](https://zenn.dev/google_cloud_jp/articles/6a8a0e571841ef)でも紹介されていましたが、この事例ページのプロダクトカテゴリを「コンテナ」で絞っていただくと、GKE や Cloud Run 等を活用された事例を確認することができます。  
https://cloud.google.com/customers?hl=ja#/products=%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A

### Google Cloud の公式ブログ
また、Google Cloud の公式ブログでも事例や最新情報が閲覧できます。  
とはいえブログ記事の数も多いので、特定製品 (例: GKE) に絞って記事をみたい場合は `https://cloud.google.com/blog/ja/products/gke` とアクセスしていただくのが良いと思います。  
https://cloud.google.com/blog/ja/products/gke?hl=ja

## イベントのアーカイブ動画から事例を探す
Google Cloud の公式イベントでは多くの事例が発表されています。多くのイベントでアーカイブが閲覧可能となっているため、興味のある事例などがあればぜひ確認してみてください。  
ここでは一例として、直近開催されたイベントをいくつかピックアップしてご紹介します。  

### Google Cloud Day ’23 Tour
2023 年 5 ~ 6月に開催された `Google Cloud Day ’23 Tour` では 4 都市でイベントが開催され、GKE 関連の事例も多く発表されました。  
https://cloudonair.withgoogle.com/events/google-cloud-day-23

### Modern App Summit
2023 年 7 月に開催された `Modern App Summit` でも多くの事例が発表されました。  
テクニカルな話に留まらず、組織体制も含めた興味深い事例もありました。  
https://cloudonair.withgoogle.com/events/modern-app-summit-23q3

### Google Cloud Next Tokyo ’23
2023 年 11 月に開催された `Google Cloud Next Tokyo ’23` でも多くの事例が発表されました。  
GKE の最新アップデートや事例紹介、高度な活用方法に関するセッションも公開されています。  
https://cloudonair.withgoogle.com/events/next-tokyo

### Innovators Live Japan 
`Innovators Live Japan` はライブ配信型のウェビナープログラムで、その中でも「サーバレス・コンテナ」というカテゴリで GKE や Cloud Run に関連した情報発信を行っています。  
お客さんの事例の話を中心に紹介しているので、興味があればこちらもご確認ください！アーカイブ閲覧可能です。  
https://cloudonair.withgoogle.com/events/innovators-live-jp?tab=serverless_containers&expand=module:serverless_containers_top

# サンプルコード・マニフェストを試す
実際の環境であれこれサンプルを動かして試したいという場合は、Google Cloud が公開しているサンプルコード、マニフェストを使ってみるのも良いかと思います。  

## 公式サンプルコード集
GKE の公式ドキュメント内で紹介されているようなサンプルコードが公開されています。  
https://cloud.google.com/kubernetes-engine/docs/samples?hl=ja

## ジャンプスタート ソリューション
[ジャンプスタート ソリューション](https://cloud.google.com/solutions?hl=ja#section-3) には各プロダクトを使ったリファレンス アーキテクチャが載っており、Google Cloud コンソール上でもワンクリックで環境デプロイできるようになっています。  
GKE を使ったソリューションもあるので、まずはここから試してみるのも良いかと思います。
![](/images/gke-korekara-learning/jumpstart-sol.png)  
https://cloud.google.com/solutions/ecommerce-web-app?hl=ja

## GitHub 上のサンプルマニフェスト集
GitHub 上にもサンプルコードやマニフェストが公開されているリポジトリがあります。  
その中でも私が個人的によく活用しているものをいくつか紹介します。  

### GKE Networking Recipes
GKE のネットワーク関連 (Gateway / Ingress / Service) のサンプルマニフェストが公開されています。  
気になる構成をすぐにお試しできて便利です。  
https://github.com/GoogleCloudPlatform/gke-networking-recipes

### AI/ML on GKE
AI/ML ワークロードを GKE で動かす場合のサンプル集です。AI/ML ワークロードのスタートアップ遅延を小さくするためのプラクティスなども公開されています。  
https://github.com/GoogleCloudPlatform/ai-on-gke

# 公式のベストプラクティス ドキュメントを読み込む
公式ドキュメントでは Google Cloud としておすすめしている GKE の設定をさまざまな観点から紹介しています。  
クラスタ設計をする場合など、まずはここに書いている内容をベースに検討を始めていくのが良いのではないでしょうか。  
ちなみに（わざわざ書くまでもないことですが）ここに書いている内容が唯一解ではないので、要件的に自社に合わない場合はドキュメント内容は参考程度にしつつ他のオプションも検討していきましょう。  

## 大規模な GKE クラスタを計画する
大きい規模の GKE クラスタを設計する前に考慮すべき点がまとまっているドキュメントです。  
https://cloud.google.com/kubernetes-engine/docs/concepts/planning-scalability?hl=ja
https://cloud.google.com/kubernetes-engine/docs/concepts/planning-large-clusters?hl=ja
https://cloud.google.com/kubernetes-engine/docs/concepts/planning-large-workloads?hl=ja

また、大規模クラスタ設計の場合は GKE のリソース上限も予め確認しておきましょう。  
https://cloud.google.com/kubernetes-engine/quotas?hl=ja

## マルチテナント構成
1つの GKE クラスタ上で複数のチーム・サービスが乗ってくるようなケースで役にたつ情報がまとまっています。  
https://cloud.google.com/kubernetes-engine/docs/best-practices/enterprise-multitenancy?hl=ja

## GKE ネットワーク構成
GKE の VPC 設計、IP アドレス設計の参考に役立ちます。  
https://cloud.google.com/kubernetes-engine/docs/best-practices/networking?hl=ja

## GKE セキュリティ
セキュリティ観点での GKE の設定について情報がまとまっています。  
https://cloud.google.com/kubernetes-engine/docs/how-to/hardening-your-cluster?hl=ja

また、被る部分は多いのですが、私が昔書いた記事でもセキュリティ観点で各オプション・機能を紹介しています。よろしければ読んでみてください。  
https://medium.com/google-cloud-jp/gkesecurity-2022-1-ea4d55bcf4f7

## コスト最適化
GKE クラスタのコスト最適化を検討されている場合、以下の記事が参考になると思います。  
https://cloud.google.com/architecture/best-practices-for-running-cost-effective-kubernetes-applications-on-gke?hl=ja

## バッチワークロード
トレーニングジョブやバッチ処理など、バッチワークロードを GKE 上で実行する際のプラクティスが紹介されています。  
https://cloud.google.com/kubernetes-engine/docs/best-practices/batch-platform-on-gke?hl=ja

# ハンズオン イベントで学ぶ
実は GKE 道場というハンズオンイベントを定期的に開催しています。コンテナや Kubernetes の基本的な話から、サービスメッシュやマルチクラスタなど高度な構成までハンズオンで学ぶことができるイベントです。  
定期的に開催のお知らせを発信していますので、もしご興味あればご参加ください！  
https://cloudonair.withgoogle.com/events/gkehandson-q4

# まとめ
GKE をこれから学びたい・詳しくなりたい方向けに、おすすめの学習リソースをご紹介しました。  
本記事で紹介したもの以外にも、[リリースノート](https://cloud.google.com/kubernetes-engine/docs/release-notes)や X (旧 Twitter) による情報収集もおすすめです。 X では [#gcpja](https://twitter.com/hashtag/gcpja) というハッシュタグで日本語でのアップデート情報を流していたりするので興味があれば定期的に覗いてみてください。  
ではみなさま素敵な GKE ライフを〜  

明日は [Issei](https://zenn.dev/hoisjp) さんによる Cloud Workstations の記事です！お楽しみに！