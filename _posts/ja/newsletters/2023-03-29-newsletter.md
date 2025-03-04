---
title: 'Bitcoin Optech Newsletter #244'
permalink: /ja/newsletters/2023/03/29/
name: 2023-03-29-newsletter-ja
slug: 2023-03-29-newsletter-ja
type: newsletter
layout: newsletter
lang: ja
---
今週のニュースレターでは、Tunable Penaltyを使用してLNの資金効率を向上させる提案を掲載しています。
また、Bitcoin Stack Exchangeによく寄せられる質問と回答の要約や、
新しいリリースおよびリリース候補の発表、人気のあるBitcoinインフラストラクチャソフトウェアの注目すべき変更など、
恒例のセクションも含まれています。

{% assign S0 = "_S<sub>0</sub>_" %}
{% assign S1 = "_S<sub>1</sub>_" %}

## ニュース

- **<!--preventing-stranded-capital-with-multiparty-channels-and-channel-factories-->マルチパーティチャネルやチャネルファクトリーでの資金滞留を防止する:**
  John Lawは、彼の書いた[論文][law stranded paper]の概要をLightning-Devメーリングリストに[投稿しました][law stranded post]。
  彼は、現在利用できないノード（モバイルLNウォレットのユーザーなど）とチャネルを共有している場合でも、
  常時利用可能なノードが自身の資金を使用して支払いを転送し続けられるようにする方法について説明しています。
  これには、マルチパーティチャネルを使用する必要があり、彼が以前説明したチャネルファクトリーの設計とうまく構成できます。
  彼はまた、チャネルをオフチェーンでリバランスできるという[チャネルファクトリー][topic channel factories]の
  既知の利点を再度述べています。これにより資金の有効活用が可能になる可能性があります。
  この2つの利点を、彼が以前考案したLNの _Tunable Penalty_ レイヤーの文脈で利用する方法を説明しています。
  ここでは、Tunable Penaltyを要約し、それがどのようにマルチパーティチャネルや、
  チャネルファクトリーで使用できるかを説明し、その後Lawの新しい結果を文脈に沿って説明します。

    アリスとボブは、それぞれ50M sat（合計100M sat）を、使用時に両者の協力が必要な
    _ファンディング・アウトプット_ に支払うトランザクションを作成します（ただしすぐには署名しません）。
    以下の図では、承認されたトランザクションを網掛けしています。

    {:.center}
    ![アリスとボブがファンディング・トランザクションを作成](/img/posts/2023-03-tunable-funding.dot.png)

    また、それぞれ個別に制御する異なるアウトプットを使用して、両者に１つずつ、2つの _ステート・トランザクション_
    を作成します（ただし、ブロードキャストはしません）。各ステート・トランザクションの最初のアウトプットは、
    タイムロックされたオフチェーンの _コミットメント・トランザクション_ へのインプットとして小額（たとえば1,000 sat）を支払います。
    相対的なタイムロックにより、各コミットメント・トランザクションは、
    その親のステート・トランザクションがオンチェーンで承認された後、一定時間経過するまで、
    オンチェーンでの承認の対象になりません。
    また、それぞれの2つのコミットメント・トランザクションは、
    競合するファンディング・アウトプットから資金提供されています（つまり、
    どちらかのコミットメント・トランザクションのみが最終的に承認されます）。
    すべての子トランザクションが作成されたら、ファンディング・アウトプットを作成するトランザクションに
    署名してブロードキャストできます。

    {:.center}
    ![アリスとボブがそれぞれコミットメント・トランザクションを作成](/img/posts/2023-03-tunable-commitment.dot.png)

    各コミットメント・トランザクションは、チャネルの現在の状態に対して支払いをします。
    初期状態({{S0}})では、50M satがアリスとボブにそれぞれ払い戻されます（単純にするため、トランザクション手数料は無視します）。
    アリスとボブは、自身のバージョンのステート・トランザクションを公開することで、
    チャネルを一方的に閉じるプロセスを開始できます。強制的な遅延の後、
    続いて対応するコミットメント・トランザクションを公開できるようになります。
    たとえば、アリスが自分のステート・トランザクションと
    （自分とボブに支払いをする）コミットメント・トランザクションを公開します。
    その時点で、ボブは自分のステート・トランザクションを使用することはできず、
    代わりに、ステート・トランザクションを作成するために使用した資金を後で好きに使用できます。

    {:.center}
    ![アリスは正直にチャネルから支払う](/img/posts/2023-03-tunable-honest-spend.dot.png)

    初期状態で一方的にチャネルを閉じる以外に、2つの選択肢があります。
    1つは、アリスとボブがファンディング・トランザクションのアウトプットを使用することで、
    いつでも協力してチャネルを閉じることができます（現在のLNプロトコルで行われているのと同じです）。
    2つめに、両者は状態を更新することができます。たとえば、アリスの残高を10M sat増やし、
    ボブの残高を同じ額減らすことができます。状態 {{S1}} は、初期状態( {{S0}} )と似てますが、
    前の状態( {{S0}} )の各ステート・トランザクションの最初のアウトプットを使用するために必要な
    witness[^keychain]を相手に与えることで、前の状態を取り消すことができます。
    {{S0}} ステート・トランザクション自体にはまだwitnessが含まれておらず、ブロードキャストできないため、
    どちらの当事者も相手のwitnessを使用することはできません。

    複数の状態が利用できるようになると、誤って、あるいは意図的に、古い状態でチャネルを閉じることが可能です。
    たとえば、ボブは10M satoshiを余分に持っている状態 {{S0}} でチャネルを閉じようとするかもしれません。
    そうする場合、ボブは彼の {{S0}} のステート・トランザクションに署名し、ブロードキャストします。
    コミットメント・トランザクションにはタイムロックがあるため、ボブはそれ以上のアクションをすぐにとることはできません。
    その待ち時間の間に、アリスは古い状態がブロードキャストされたことを検出し、
    ボブが以前与えたwitnessを使用して、ボブのステート・トランザクションの最初のアウトプットを使用し、
    ペナルティ額の一部または全額をトランザクション手数料として使用します。
    このアウトプットは、ボブに余分に10M sat支払うコミットメント・トランザクションを
    ブロードキャストするために必要なアウトプットと同じなので、アリスが作成したトランザクションが承認されると、
    ボブはその資金の請求をブロックされることになります。
    ボブがブロックされているため、アリスだけが一方的に最新の状態をオンチェーンで公開することができます。
    代わりに、アリスとボブはまだいつでも協力してチャネルを閉じることができます。

    {:.center}
    ![ボブがチャネルから不正に支払おうとしてアリスによりブロックされる](/img/posts/2023-03-tunable-dishonest-spend.dot.png)

    もしボブがアリスが彼の古いステート・トランザクションを使おうとしているのに気づいたら、
    アリスとRBF（Replace By Fee）による入札競争を試みることが可能ですが、
    この場合、ペナルティ額が _調整可能_ であることが特に強力になります。
    ペナルティ額は、少額（たとえば、この例のように1K satの場合）か、
    ステーク額（10M sat）に近いか、チャネル全体の金額よりも大きい場合があります。
    この決定は、チャネルを更新する際に、アリスとボブが互いに交渉することに完全に委ねられています。

    TPP（Tunable-Penalty Protocol）の他の利点の１つは、
    古い状態のトランザクションをオンチェーンに投入したユーザーがペナルティ額を全額支払うということです。
    共有したファンディング・トランザクションのビットコインは一切使用しません。
    これにより、2人以上のユーザーがTPPチャネルを安全に共有できます。
    たとえば、アリスとボブ、キャロル、ダンがチャネルを共有しているとしましょう。
    各自、自身のステート・トランザクションから資金を得る自身のコミットメントトランザクションを持っています:

    {:.center}
    ![アリスとボブとキャロル、ダンの間のチャネル](/img/posts/2023-03-tunable-multiparty.dot.png)

    彼らはこれをマルチパーティチャネルとして運用し、各状態が各参加者によって取り消される必要があります。
    あるいは、共同のファンディング・トランザクションをチャネルファクトリーとして利用し、
    ペアまたは複数のユーザー間で複数のチャネルを作ることも可能です。
    昨年、LawがTPPのこの意味について説明する（[ニュースレター #230][news230 tp]参照）前は、
    Bitcoinでチャネルファクトリーを実用化するためには[SIGHASH_ANYPREVOUT][topic sighash_anyprevout]のような
    コンセンサスの変更を必要とする[eltoo][topic eltoo]のような仕組みが必要と考えられていました。
    TPPはコンセンサスの変更を必要としません。下の図を簡単にするため、
    参加者4人のファクトリーで作成された1つのチャネルだけを図示しています。
    チャネル参加者が管理する必要のある状態の数は、ファクトリーの参加者の数と等しくなります。
    Lawは単一の状態を使用する構成を[以前説明していました][law factories]が、
    一方的にチャネルを閉じる際のコストが高くなります。

    {:.center}
    ![アリスとボブとキャロル、ダンのファクトリーで作られたアリスとボブのチャネル](/img/posts/2023-03-tunable-factory.dot.png)

    [元の論文][channel factories paper]に記載されているチャネルファクトリーの利点は、
    ファクトリー内の参加者が、オンチェーントランザクションを作成することなくチャネルを協力的にリバランスできることです。
    たとえば、ファクトリーがアリス、ボブ、キャロル、ダンで構成されている場合、
    オフチェーンでファクトリーの状態を更新することで、アリスとボブのチャネルの総額を減らし、
    キャロルとダンのチャネルの総額を同額分増やすことができます。
    LawのTPPベースのファクトリーも同じ利点を提供します。

    今週、Lawはマルチパーティチャネルを提供する機能を持つファクトリー（TPPにより可能になる）には、
    1人のチャネル参加者がオフラインの場合でも、資金を使用できるという追加の利点があると指摘しました。
    たとえば、アリスとボブは専用のLNノードを持っていて、ほぼ常に支払いを転送することができるものの、
    キャロルとダンはカジュアルなユーザーで、そのノードはよく利用できなくなる場合を考えてみましょう。
    オリジナルのチャネルファクトリーでは、アリスはキャロルとのチャネル（{A,C}）と
    ダンとのチャネル（{A,D}）を持っています。キャロルとダンが利用できない場合、
    アリスはこれらのチャネルの資金を使用することはできません。
    ボブも、（{B,C}と{B,D}のチャネルについて）同じ問題を抱えています。

    TPPベースのファクトリーでは、アリス、ボブ、キャロルでマルチパーティチャネルを開くことができ、
    状態の更新にその3人の協力が必要になります。そのチャネルにおいて、
    コミットメント・トランザクションのアウトプットの1つはキャロルに支払われますが、
    他のアウトプットはアリスとボブが協力した場合にのみ使用することができます。
    キャロルが利用できない場合、アリスとボブは協力して共同アウトプットの残高の配分をオフチェーンで変更でき、
    他のLNチャネルを持っている場合はLN支払いの実行や転送が可能になります。
    アリスが長い間利用できない状態が続くと、どちらかが一方的にチャネルをオンチェーンにすることができます。
    アリスとボブがダンとチャネルを共有している場合も、同じ利点が適用されます。

    これにより、キャロルやダンが利用できない時でも、アリスとボブは転送手数料を稼ぎ続けることができ、
    それらのチャネルが非生産的であると思われるのを防ぐことができます。
    また、オフチェーン（オンチェーン手数料なし）でチャネルのリバランスができるため、
    アリスとボブがチャネルファクトリーに資金を長期間置いておくことのデメリットも減らせるかもしれません。
    これらの利点を合わせると、オンチェーントランザクションの数を減らし、
    Bitcoinネットワークの総支払い能力を高め、LN上で支払いを転送するコストを下げることができます。

    この記事を書いている時点では、Tunable Penaltyとそれを利用するためのLawのさまざまな提案は、
    あまり公に議論されていません。

## Bitcoin Stack Exchangeから選ばれたQ&A

*[Bitcoin Stack Exchange][bitcoin.se]はOptech Contributor達が疑問に対して答えを探しに（もしくは他のユーザーの質問に答える時間がある場合に）アクセスする、
数少ない情報ソースです。この月刊セクションでは、前回アップデート以降にされた、最も票を集めた質問・回答を紹介しています。*

{% comment %}<!-- https://bitcoin.stackexchange.com/search?tab=votes&q=created%3a1m..%20is%3aanswer -->{% endcomment %}
{% assign bse = "https://bitcoin.stackexchange.com/a/" %}

- [TaprootのデプロイはなぜBitcoin Coreに埋め込まれてないのでしょうか？]({{bse}}117569)
  Andrew Chowは、Taprootのソフトフォークの[デプロイ][topic soft fork activation]が、
  [他のデプロイ][bitcoin buried deployments]のように[埋め込まれていない][BIP90]根拠を説明しています。

- [ブロックヘッダーのversionフィールドにはどんな制約がありますか？]({{bse}}117530)
  Murchは、[overtタイプのASICBoost][topic ASICBoost]を使用してマイニングされた[ブロック][explorer block 779960]の増加を指摘し、
  versionフィールドの制約をリストアップし、[ブロックヘッダーのversionフィールド][FCAT block header blog]の例を説明しています。

- [トランザクションデータとIDの関係は？]({{bse}}117453)
  Pieter Wuilleは、`txid`識別子がカバーする従来のトランザクションシリアライゼーションフォーマットと、
  `hash`と`wtxid`識別子がカバーするwitnessの拡張シリアライゼーションフォーマットについて説明し、
  [別の回答][se117577]で仮想的な追加トランザクションデータが`hash`識別子でカバーされると指摘しています。

- [他のピアにtxメッセージを要求することは可能ですか？]({{bse}}117546)
  ユーザーRedGrittyBrickは、Bitcoin CoreのP2Pレイヤーではピアからの任意のトランザクションの要求がサポートされていない
  [パフォーマンス][wiki getdata]と[プライバシー上][Bitcoin Core #18861]の理由を説明するリソースを指摘しています。

- [Eltoo: 最初のUTXOの相対的ロックタイムがチャネルのライフタイムを決めるのですか？]({{bse}}117468)
  Murchは、質問の例の[eltoo][topic eltoo]構成のLNチャネルのライフタイムが限定されていることを確認しますが、
  タイムアウトを失効させないための[eltooの論文][eltoo whitepaper]の緩和策を指摘しました。

## リリースとリリース候補

*人気のBitcoinインフラストラクチャプロジェクトの新しいリリースとリリース候補。
新しいリリースにアップグレードしたり、リリース候補のテストを支援することを検討してください。*

- [Rust Bitcoin 0.30.0][]は、Bitcoinに関連するデータ構造を使用するためのこのライブラリの最新リリースです。
  [リリースノート][rb rn]では、[新しいウェブサイト][rust-bitcoin.org]と大量のAPIの変更について言及しています。

- [LND v0.16.0-beta.rc5][]は、この人気のLN実装の新しいメジャーバージョンのリリース候補です。

- [BDK 1.0.0-alpha.0][]は、[先週のニュースレター][news243 bdk]で紹介したBDKの主な変更のテストリリースです。
  下流プロジェクトの開発者は、統合テストを開始することをお勧めします。

## 注目すべきコードとドキュメントの変更

*今週の[Bitcoin Core][bitcoin core repo]、[Core
Lightning][core lightning repo]、[Eclair][eclair repo]、[LDK][ldk repo]、
[LND][lnd repo]、[libsecp256k1][libsecp256k1 repo]、[Hardware Wallet
Interface (HWI)][hwi repo]、[Rust Bitcoin][rust bitcoin repo]、[BTCPay
Server][btcpay server repo]、[BDK][bdk repo]、[Bitcoin Improvement
Proposals（BIP）][bips repo]、および[Lightning BOLTs][bolts repo]の注目すべき変更点。*

- [Bitcoin Core #27278][]は、ノードがIBD（Initial Block Download）中でない限り、
  新しいブロックのヘッダーを受信すると、デフォルトでログに記録するようになりました。
  これは、複数のノードオペレーターが、3つのブロックがとても近いタイミングで到着し、
  後の2つがベストブロックチェーンでの最初のブロックの再編成によるものであることに気づいた事が[発端です][obeirne selfish]。
  分かりやすくするために、最初のブロックを _A_、それに置き換わるブロックを _A'_ 、
  最後のブロックを _B_ とします。

  これはブロックA'とBが同じマイナーによって作られたことを示している可能性があり、
  他のマイナーのブロックが古くなるまで意図的にブロードキャストを遅らせ、
  そのマイナーが通常Aから受け取るはずだったブロック報酬を拒否させる
  _セルフィッシュマイニング_ として知られる攻撃である可能性があります。
  偶然の一致か、セルフィッシュマイニングの可能性もあります。
  しかし、調査中に開発者によって[提起された][sanders requests]１つの可能性は、
  タイミングが実際には見かけほど近くはなかった可能性があるというものでした。
  A'だけでは再編成を引き起こすには十分ではなかったため、
  Bitcoin CoreはBを受信するまでA'を要求しなかった可能性があります。

  ヘッダーを受信した時間をログに記録するのは、将来このような状況が繰り返された場合に、
  ノードオペレーターが、自分のノードがすぐにダウンロードを選択しなかったとしても、
  いつ最初にA'の存在を知ったかを判断できるようにすることを意味します。
  このログの記録により、1ブロックあたり最大2行の新しい行が追加される可能性がありますが（ただし、
  将来のPRでは1行に減らされる可能性があります）、これはセルフィッシュマイニング攻撃や、
  重要なブロックリレーに関するその他の問題の検出に役立つ、十分に小さなオーバーヘッドであると考えられています。

- [Bitcoin Core #26531][]では、以前のPR（[ニュースレター #133][news133 usdt]参照）で実装された
  eBPF（Extended Berkeley Packet Filter）を使用してmempoolに影響するイベントを監視するためのトレースポイントを追加しました。
  また、トレースポイントを使用してmempoolの統計とアクティビティをリアルタイムで監視するためのスクリプトも追加されました。

- [Core Lightning #5898][]は、[libwally][]の依存関係をより新しいバージョン（
  [ニュースレター #238][news238 libwally]参照）に更新しました。これにより、
  [Taproot][topic taproot]および[PSBT][topic psbt]バージョン2（[ニュースレター #128][news128 psbt2]参照）のサポートが追加され、
  ElementsスタイルのサイドチェーンのLNサポートに影響します。

- [Core Lightning #5986][]は、msatsで値を返すRPCを更新し、
  結果の一部として「msat」という文字列を含めないようにしました。
  代わりに、返される値はすべて整数になります。これにより、
  いくか前のリリースから始まった非推奨化が完了します（[ニュースレター #206][news206 msat]参照）。

- [Eclair #2616][]は、日和見的な[ゼロ承認チャネル][topic zero-conf channels]のサポートを追加しました。
  リモートピアが期待される承認数より前に`channel_ready`メッセージを送信した場合、
  Eclairはファンディングトランザクションがローカルノードによって完全に作成されたことを確認した上で
  （つまりリモートピアは競合するトランザクションを作れません）、チャネルの使用を許可します。

- [LDK #2024][]は、[ゼロ承認チャネル][topic zero-conf channels]など、
  開設されたものの承認数が足りずまだ公表されていないチャネルのルートヒントを含めるようになりました。

- [Rust Bitcoin #1737][]は、プロジェクトの[セキュリティ報告ポリシー][rb sec]を追加しました。

- [BTCPay Server #4608][]は、プラグインがその機能を
  BTCPayのユーザーインターフェースのアプリとして公開できるようにしました。

- [BIPs #1425][]は、[ニュースレター #239][news239 codex32]に掲載したように、
  [BIP32][]のリカバリーシードをシャミアの秘密分散法（SSSS）アルゴリズムや、
  チェックサム、32文字のアルファベットを用いてエンコードするCodex32方式に[BIP93][]を割り当てました。

- [Bitcoin Inquisition #22][]は、0から126バイトのデータをTaprootインプットのannexフィールドにプッシュできる
  `-annexcarrier`実行オプションを追加しました。このPRの著者は、Core Lightningのフォークを使用して、
  signetで[eltoo][topic eltoo]の実験を開始できるようにすることを計画しています。

## 脚注

[^keychain]:
    このハイレベルの概要では、witnessが何であるか説明することは重要ではありませんが、
    提案された利点のいくつかはその詳細に依存します。元の[Tunable Penalty][Tunable Penalties]プロトコルの説明では、
    コミットメント・トランザクションを使用するための署名を生成するために秘密鍵をリリースすることを提案しています。
    秘密鍵を順に生成することは可能で、ある秘密鍵を知っている人はその後の鍵も導出することができます
    （ただし、前の鍵は導出できません）。
    これは、アリスが後の状態を破棄するたびに、アリスはボブに以前の鍵を渡すことができ、
    ボブはそれを使って（以前の状態に対して）後の鍵を導出することができることを意味します。たとえば、

      | チャネルの状態 | 鍵の状態 |
      | 0     | MAX |
      | 1     | MAX - 1 |
      | 2     | MAX - 2 |
      | x     | MAX - x |
      | MAX   | 0 |

    これによりボブは、古い状態のトランザクションを使用するために必要な情報を、
    非常に小さな一定量のスペース（計算では100バイト未満）で保存できます。
    また、その情報は[Watchtower][topic watchtowers]と簡単に共有できます。
    （トラストする必要なく、古いステート・トランザクションの使用が成功すると、
    古いコミットメントトランザクションがオンチェーンで公開されるのを防ぐことができます。
    古いステート・トランザクションに含まれる資金は、プロトコルに違反している参加者に完全に帰属するため、
    そこからの支払いに関する情報を外部委託してもセキュリティ上のリスクはありません。）

{% include references.md %}
{% include linkers/issues.md v=2 issues="27278,26531,5898,5986,2616,2024,1737,4608,1425,22,18861" %}
[lnd v0.16.0-beta.rc5]: https://github.com/lightningnetwork/lnd/releases/tag/v0.16.0-beta.rc5
[bdk 1.0.0-alpha.0]: https://github.com/bitcoindevkit/bdk/releases/tag/v1.0.0-alpha.0
[rust bitcoin 0.30.0]: https://github.com/rust-bitcoin/rust-bitcoin/releases/tag/bitcoin-0.30.0
[news230 tp]: /ja/newsletters/2022/12/14/#ln
[channel factories paper]: https://tik-old.ee.ethz.ch/file//a20a865ce40d40c8f942cf206a7cba96/Scalable_Funding_Of_Blockchain_Micropayment_Networks%20(1).pdf
[law factories]: https://raw.githubusercontent.com/JohnLaw2/ln-efficient-factories/main/efficientfactories10.pdf
[news206 msat]: /ja/newsletters/2022/06/29/#core-lightning-5306
[rb sec]: https://github.com/rust-bitcoin/rust-bitcoin/blob/master/SECURITY.md
[news239 codex32]: /ja/newsletters/2023/02/22/#codex32-bip
[law stranded post]: https://lists.linuxfoundation.org/pipermail/lightning-dev/2023-March/003886.html
[law stranded paper]: https://github.com/JohnLaw2/ln-hierarchical-channels
[obeirne selfish]: https://twitter.com/jamesob/status/1637198454899220485
[sanders requests]: https://twitter.com/theinstagibbs/status/1637235436849442817
[news133 usdt]: /ja/newsletters/2021/01/27/#bitcoin-core-19866
[libwally]: https://github.com/ElementsProject/libwally-core
[news128 psbt2]: /en/newsletters/2020/12/16/#new-psbt-version-proposed
[tunable penalties]: https://github.com/JohnLaw2/ln-tunable-penalties
[bitcoin buried deployments]: https://github.com/bitcoin/bitcoin/blob/master/src/consensus/params.h#L19
[explorer block 779960]: https://blockstream.info/block/00000000000000000003a337a676b0101f3f7ef7dcbc01debb69f85c6da04dcf?expand
[FCAT block header blog]: https://medium.com/fcats-blockchain-incubator/understanding-the-bitcoin-blockchain-header-a2b0db06b515#b9ba
[se117577]: https://bitcoin.stackexchange.com/a/117577/87121
[wiki getdata]: https://en.bitcoin.it/wiki/Protocol_documentation#getdata
[eltoo whitepaper]: https://blockstream.com/eltoo.pdf#page=15
[news238 libwally]: /ja/newsletters/2023/02/15/#libwally-0-8-8
[rust-bitcoin.org]: https://rust-bitcoin.org/
[rb rn]: https://github.com/harding/rust-bitcoin/blob/bbda9599fa32936f31472620d014893fda17d8c3/bitcoin/CHANGELOG.md#030---2023-03-21-the-first-crate-smashing-release
[news243 bdk]: /ja/newsletters/2023/03/22/#bdk-793
