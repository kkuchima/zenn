---
title: "Istio Certified Associate (ICA) を取得した"
emoji: "🚢"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [Istio]
published: false
---
Istio Certified Associate (ICA) という資格を取得したので、備忘録的に情報を残しておきます。2024 年 01 月時点での情報となるので、念の為最新情報は公式サイトで確認をお願いします。  
https://training.linuxfoundation.org/certification/istio-certified-associate-ica/

# 試験の概要
サービスメッシュ プロダクトである [Istio](https://istio.io/) の入門資格のような立ち位置です。  
[CKA (Certified Kubernetes Administrator)](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/) や [CKAD (Certified Kubernetes Application Developer)](https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/) 同様 Lunux Foundation から出ています。  

* 試験実施方法：オンライン
* 試験時間：2 時間
* 出題形式：実技試験 + 4択問題
* 合格スコア：75 点以上
* 受験料：$250
* 再受験ポリシー：１回まで再受験可能
* 試験の言語：英語
* 受験に関する事前要件：特になし

出題範囲 (2024 年 01 月時点) は以下のようになっており、Istio の基本的な機能の使い方やカスタマイズ、トラブルシューティング方法についての知識が求められます。
![](/images/ica-totta/agenda.png)

試験中は Istio の公式ドキュメント (https://istio.io/) を閲覧することが可能です。  
試験対策の学習を通して Istio の利用方法について網羅的に学ぶことができるため、個人的な感想としては実務で **Istio をこれから使う方**や現在利用しているけど**知識に偏りがある方**などにおすすめできる試験なのではないかと思います。  

# 試験準備
## 事前知識・バックグラウンドの紹介
正直、必要となる学習リソースや学習時間はスタートラインによって違ってくると思うので、参考までに私のプロフィールを記載しておきます。  
普段はパブリッククラウドベンダーで Kubernetes やサービスメッシュ(含む Istio)領域の技術支援をしています。なので基本的な Istio 機能は普段から触っていて、また趣味で細かな検証や Advanced な構成も試していたりします。  
ちなみに CKA と CKAD も取得済み（期限切れましたが）です。  

## 受験するまでに行なったこと
私の場合、元々ある程度知識があったため試験準備期間は半日くらいでした（あと、リテイクできるので、とりあえず受けてみて感触を確かめようと思った） 。
あまり準備期間は確保できませんでしたが、以下のように試験の準備を進めました。  
1. 受験申し込み
2. 試験の Important Instructions や FAQ を読む
https://docs.linuxfoundation.org/tc-docs/certification/important-instructions-ica
https://docs.linuxfoundation.org/tc-docs/certification/frequently-asked-questions-ica
3. 試験対策の実施

## おすすめ学習リソース
個人的なおすすめ学習リソースを紹介します。  
ちなみに学習を進めるにあたって、念の為[試験環境](https://docs.linuxfoundation.org/tc-docs/certification/important-instructions-ica#ica-exam-environment)に記載されている Istio バージョンを利用するのが良いのではないかと思います。  

### 1. Tetrate Academy の Istio Fundamentals
（試験対策とか関係なく）おすすめの教材です。Istio の基本的な仕組みや機能について学ぶことができます。  
また、ICA は [Tetrate の Istio 認定資格](https://academy.tetrate.io/courses/certified-istio-administrator)がもとになっているように見受けられるので、試験対策としてもぴったりだと思います。  
私は試験前日から着手したので時間の都合上全てのセクションは通せませんでした(出題範囲と照らし合わせて出題されそうな部分にフォーカスしました)が、余裕がある方は一通り通していただくのが良いと思います。  
https://academy.tetrate.io/courses/istio-fundamentals

### 2. Istio 公式ドキュメント
試験中に唯一閲覧可能なドキュメントです。試験中にサイト内の検索窓が利用できなくて焦ったので、事前にドキュメント内のどのセクションにどんな内容が書いてるかを（ざっくりでも良いので）把握するのが良いと思います。  
https://istio.io/

ただ、改めて見てみると[公式のページ](https://docs.linuxfoundation.org/tc-docs/certification/certification-resources-allowed#istio-certified-associate-ica)には以下のように記載があるので、私の検索の仕方が悪かった可能性が高いです...
`use the search function provided on https://istio.io/latest/docs/ however, they may only open search results that have a domain matching the sites listed above`

### 3. 模擬試験
[Mesh Week](https://www.youtube.com/playlist?list=PLy0Gle4XyvbG0rQBZ3dUCvxtekoJ7XjCr) のメンバーが作ったと思われる模擬試験です。どんな感じの試験が出題されるかを知るために最初に試してみても良いかと思います。  
https://docs.google.com/forms/d/e/1FAIpQLSfD4BLLQfdUwnIyiTBSGC_OzmSbiyrIlNp5Am61fTOhRbfiLw/viewform

### 4. Istio in Action
2-3 年前くらいにこの書籍を一通り読む＆手を動かして Istio の基本的な挙動を学びました。試験対策としては必須では無いと思うので、時間に余裕がある人はこの辺りも読んでも良いかもしれません。  
https://www.manning.com/books/istio-in-action

# 試験当日~結果通知まで
## PSI Secure Browser との格闘
数年ぶりに Linux Foundation の試験を受けたので当日は色々と大変でした。特に PSI Secure Browser なるものがすんなり試験を開始してくれずに苦労しました。  
私の場合、試験に関係のないアプリケーション（Mac の Sidecar Relay) が動いているということで試験を開始できず、当該アプリケーションの正しい落とし方がよく分からなかったので Bluetooth を切ってプロセスを直接落としてなんとか始めることができました。  

## 試験環境
試験は CKA / CKAD と同様（なんですかね？勘で書きました）の仮装デスクトップ環境 (GNOME) 上で受けました。試験環境のマシンでは VS Code も使えたので私は VS Code を使って試験を進めました。試験環境の詳細については事前に以下公式ページもご確認ください。  
https://docs.linuxfoundation.org/tc-docs/certification/important-instructions-ica#ica-exam-environment

## 試験の感想
難易度自体はそこまで高くなく分からなくて解けないというレベルの設問はありませんでしたが、１問だけ時間が足りなくて完了できませんでした（というか正直疲れて気力がなくなりました..）。  
CKA / CKAD 同様あまり時間に余裕のない試験なので、時間がかかりそうな設問は後回しにする等の対策をとるのが良いと思います。ただ、設問に見直し用のフラグを立てることができなかった（私がやり方分からなかっただけかもしれませんが）ので、試験環境で使えるメモ帳に後回しにする設問番号を記載しておくとかが良いかもしれません。  
（内容は面白かったけどしんどかったので、再受験は絶対したくない...と思いながら問題を解いてました。）  

## 試験結果通知
試験完了から 24 時間以内にメールで通知がくると書いていたので待っていると、24時間ギリギリくらいに通知がきました。  



ということで ICA の受験体験記は以上です。これから受験される方の参考になれば幸いです。