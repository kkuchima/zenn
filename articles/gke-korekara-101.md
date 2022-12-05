---
title: "これから始める GKE (基本編)"
emoji: "🦄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP, GoogleCloud, GKE]
publication_name: "google_cloud_jp"
published: false
---
この記事は [Google Cloud Japan Advent Calendar 2022 (今から始める Google Cloud)](https://zenn.dev/google_cloud_jp/articles/12bd83cd5b3370#%E4%BB%8A%E3%81%8B%E3%82%89%E5%A7%8B%E3%82%81%E3%82%8B-google-cloud) の 6 日目の記事です。
`今から始める Google Cloud` ということで、これから [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine) (以降 GKE) を使っていこうと考えられている方向けに GKE の基本的な特徴や他のアプリケーション プラットフォームとの違い・使い分けなどについて紹介しようと思います。
ちなみにこの「これから始める GKE」はシリーズ化しようと思っていますので、今後は設計に関する考慮ポイントなど細かめな話も書いていく予定です。

# tl;dr
* 

# GKE とは
GKE は Google Cloud が提供するフルマネージドな Kubernetes プラットフォームです。Kubernetes をより簡単かつ安全に使っていただくための機能を多く持っています。

## 特徴① 運用負荷を軽減するための各種自動化機能

## 特徴② 

## 特徴③ 

# GKE の種類
GKE には現在 GKE Standard と GKE Autopilot という 2 種類のモードが存在します。

GKE Standard と Autopilot の違いを説明する前にまず GKE の大まかなアーキテクチャについて説明します。 
GKE は Control Plane と Node というコンポーネントに分かれています。Control Plane は Kubernetes の api-server や 
Control Plane は Google が管理しており、利用者側で Control Plane の運用をしていただく必要はありません。
Node は Google 管理もしくは利用者側で管理するという2つのオプションを選ぶことができます。

GKE は現在 2 種類のモードが存在しており、Control Plane が Google 管理となり Node はユーザー管理となるクラスタが `GKE Standard` (従来の GKE)、一方 Control Plane に加えて Node も Google 管理となるクラスタが `GKE Autopilot` となります。

# 他のサービスとの違い、使い分け
Cloud Run , Anthos clusters

# Anthos と GKE の関係性

Anthos 自体は様々なサービスの総称・ブランドのようなもので、Anthos = ハイブリッド・マルチクラウドという訳ではないです。GKE で利用する分には

# まとめ