---
title: "オンプレから Cloud Run にプライベートな経路でアクセスする"
emoji: "🐶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [CloudRun, GCP]
published: true
---
オンプレミス環境等から Cloud Interconnect / Cloud VPN を経由して Cloud Run で公開しているサービスにアクセスしたい！外部からはアクセスさせたくない！という方向けの記事です。

# tl;dr
- デフォルトドメイン (*.run.app) で構成された Cloud Run サービスは Private Service Connect および Private Google Access for On-prem を使うことで Cloud Interconnect / Cloud VPN 経由でのアクセスが可能です
- 対象サービスをインターネットからアクセスさせたくない場合は Ingress Control を「内部 (Internal)」に設定することでアクセス経路を絞ることができます

# はじめに
デフォルト状態の Cloud Run サービスではインターネット向けにサービスが公開されており、オンプレミスや他社クラウド環境から閉域網だけを通って Cloud Run のサービスにアクセスさせるよう構成するには一工夫が必要です。
この記事では Cloud Run 上のサービスに Cloud Interconnect / Cloud VPN のような閉域網経由でアクセスできるよう構成する方法と、Cloud Run サービスへのアクセス経路を内部からのみに絞る方法についてご紹介します。

# Interconnect / Cloud VPN を経由して Cloud Run サービスにアクセスする方法
通常、何も追加設定をしていないデフォルト状態では、Cloud Run サービスへのアクセスはインターネットを通じて行われます。
これをプライベートな経路でアクセスさせるには、「オンプレミスホスト用の限定公開の Google アクセス (Private Google Access for On-prem) や 「Private Service Connect (PSC)」という機能を利用します。

Private Google Access for On-prem は、オンプレミス等の環境から Google Cloud API に対するアクセスをプライベートな経路を使って実現することができる機能です。サポート対象の Google Cloud API / ドメインとして `*.run.app` も含まれています（現状カスタムドメインは含まれていないためご注意ください）。
https://cloud.google.com/vpc/docs/private-google-access-hybrid

一方、Private Service Connect (PSC) は Private Google Access for On-prem 同様の機能を持っており、 Google Cloud API に対するアクセスをプライベートな経路を通すために使われます。その他、内部ロードバランサ経由で公開したサービスを他の利用者と直接的なネットワーク接続を構成することなくプライベートに利用できるようにする機能も持っています。詳細については以下ドキュメント / 解説ブログをご参照ください。
https://cloud.google.com/vpc/docs/configure-private-service-connect-apis
https://medium.com/google-cloud-jp/psc-01-ac1aba58b161

上記の通り `*.run.app` のプライベート接続については「Private Google Access for On-prem」 と 「Private Service Connect (PSC)」 どちらを使っても構成可能ですが、PSC ではエンドポイントで利用する IP アドレスを自分で選択でき柔軟な構成がとれる（PGA for On-prem だと 199.36.x.x/30 という決められたアドレスレンジを利用する必要あり）ことから、個人的には PSC の方が使いやすいのではないかと考えています。なので今回は PSC を使った構成を試してみたいと思います。

# アクセス経路を絞る方法
PSC 等により閉域網経由でアクセスできるよう構成したはいいものの、インターネットからも引き続きアクセス可能なのはよろしくないためこの穴を塞ぐ必要があります。
これは Cloud Run の「上り（内向き）の制限 (Ingress Control)」という機能で実現可能です。

https://cloud.google.com/run/docs/securing/ingress

Ingress Control は以下３種類のオプションがあります：
- すべて ... インターネットを含め全てのアクセスを許可
- 内部 ... 同じプロジェクト / VPC-SC 境界内のリソースからのアクセスのみ許可
- 内部と Cloud Load Balancing ... 上記「内部」＋ Cloud Load Balancing 経由のアクセスのみ許可

ざっくりとした使い分けとしては、インターネット上に素の Cloud Run サービスを公開する場合は「すべて」を選択し、Serverless NEG を使って Cloud Load Balancing 経由でサービスを公開したい（かつ GCLB を経由しないアクセスは許可しない）場合は「内部と Cloud Load Balancing」、完全に内部リソースからのアクセスしか許可しない場合は「内部」を選択する形になるかと思います。今回は、Cloud Interconnect / Cloud VPN 経由のみからアクセスを受け付けたいので「内部」を設定します。

# 試してみる
以下のような環境で実際に試してみました。
クライアントはオンプレミスにあり、オンプレミスと Google Cloud 環境は Cloud VPN で接続します。
![](/images/run-psc/overview.png)

## Cloud Run サービスのデプロイ
ではまず Cloud Console 上から hello サービスをデプロイしていきます。
![](/images/run-psc/run001.png)

Ingress Control は「すべて」を選択し、インターネットからのアクセスも許可します。
また、「未認証の呼び出しを許可」を設定し誰でもアクセスが可能にしておきます。
![](/images/run-psc/run002.png)

「作成」ボタンを押すと hello アプリケーションがデプロイできました。
![](/images/run-psc/run003.png)

## PSC の設定
続いてオンプレミスからアクセスできるように、PSC のエンドポイントを作成します。
PSC の対象として「すべての Google API」を選択し、Cloud VPN が接続されている VPC 上にエンドポイントを作成します。
![](/images/run-psc/psc001.png)

また、VPC エンドポイントの IP アドレスは `10.10.10.10` というプライベートレンジの IP アドレスを指定しています。
![](/images/run-psc/psc002.png)

## Cloud VPN の設定
オンプレミスと Google Cloud 環境を Cloud VPN で接続します。
その際、先ほど作成した PSC エンドポイントの IP アドレス (`10.10.10.10`) をオンプレミス側に広告するよう構成します（今回は検証ということで Classic VPN で Static なルート設定を行って設定しています）。
![](/images/run-psc/vpn001.png)

オンプレ VPN 機器も同様に設定を行い、トンネルが UP になれば OK です。

※以下記事でご紹介したものとほぼ同じ VPN 設定を使っています。
https://medium.com/google-cloud-jp/hybrid-load-balancing-27e77a4ec62

## オンプレミス側で名前解決の設定を行う
最後にオンプレミス側で、発行された Cloud Run サービス FQDN が PSC エンドポイントである `10.10.10.10` に解決されるように設定します。
本来は DNS を使うことが望ましいのですが、今回は検証ということで以下の内容を hosts に直接追記してしまいます。
```
10.10.10.10 hello-xxxx-an.a.run.app
```

## オンプレミスからのアクセスを試す
これで準備は完了したので、オンプレミスにあるクライアントから curl で Cloud Run サービスにアクセスしてみます。
```bash
curl https://hello-xxxx-an.a.run.app
<!doctype html>
<html lang=en>
<head>
<meta charset=utf-8>
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Congratulations | Cloud Run</title>
<link href="https://fonts.googleapis.com/css?family=Roboto" rel="preload" as="font">
...略...
```

ちゃんとアクセスできることが確認できました。

## アクセス経路を絞る
ただ今のままだと不要な経路（インターネットからのアクセス）が残ってしまっているため、この穴を塞ぎます。現在のサービスを編集し、Ingress Control に「内部」を設定します。
![](/images/run-psc/run004.png)

設定更新後、インターネット経由でのアクセスが拒否されました。Ingress Control の設定が正しく反映されていることが分かります。
![](/images/run-psc/run005.png)

一方、オンプレミスからのアクセスは変わらずできています。
```bash
curl https://hello-xxxx-an.a.run.app
<!doctype html>
<html lang=en>
<head>
<meta charset=utf-8>
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Congratulations | Cloud Run</title>
<link href="https://fonts.googleapis.com/css?family=Roboto" rel="preload" as="font">
...略...
```

これで設定完了です。

# 最後に
そんなに難しい設定をせずにオンプレミスからのプライベートな経路でアクセスできたり、アクセス経路を内部のみに絞ることができたかと思います。
同じような悩みを持っている方がいらっしゃいましたらぜひお試しください。