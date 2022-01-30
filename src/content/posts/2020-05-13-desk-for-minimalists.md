---
template: blog-post
title: Openseaのハッキングの原因について色々と考察してみる
slug: /minimalists-desk
date: 2020-05-13 12:46
description: sdasd
featuredImage: /assets/bench-accounting-nvzvopqw0gc-unsplash.jpg
---
<p>OpenSeaでNFTが不正に取引されたようです。(1)  <a href="https://gigazine.net/news/20220125-opensea-user-steal-nfts/" title="https://gigazine.net/news/20220125-opensea-user-steal-nfts/">Gigazineの記事</a><br>NFT MarketPlaceを運営する身としてはとても気になる話題です。ざっくりと調査したのでまとめました。</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":1} -->
<h1 id="opensea-market-contractの仕様">OpenSea Market Contractの仕様</h1>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>まず、OpenSea Marketでの出品と購入のプロセスを再掲します。（<a href="https://blog.suishow.net/2021/08/25/opensea-market-contract%e8%a7%a3%e8%aa%ac/">こちらの記事</a>に詳しくあります）</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>OpenSea Market Contractのコードは<a href="https://rinkeby.etherscan.io/address/0xf57b2c51ded3a29e6891aba85459d600256cf317#code">こちら</a>にあります。</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>ExchangeCore ContractというContractで出品や購入、出品キャンセルなど基本的な機能を提供します。理解する上で欠かせないのが構造体Orderです。Orderは出品やオファーの情報を保持します</p>

```solidity
   struct Order {
          address exchange; //このExchangeCore Contractのaddress
          address maker; // Orderを作成したEOA
          address taker; // Orderを受け入れたEOA
          uint makerRelayerFee; // feeRecipientに払う手数料 (アフィリエイト等に使われる?)
          uint takerRelayerFee; // feeRecipientに払う手数料 (アフィリエイト等に使われる?)
          uint makerProtocolFee; // OpenSeaに払う手数料?
          uint takerProtocolFee; // OpenSeaに払う手数料?
          address feeRecipient;
          FeeMethod feeMethod; // ProtocolFee or SplitFee
          SaleKindInterface.Side side; // Buy or Sell
          SaleKindInterface.SaleKind saleKind; // FixedPrice or DutchAuction
          address target; // 対象となるNFT Contract Address
          AuthenticatedProxy.HowToCall howToCall; //call or delegateCall
          bytes calldata; // ex) abi.encodeWithSignature("safeTransferFrom(1, 0x...)")
          bytes replacementPattern;
          address staticTarget;
          bytes staticExtradata;
          address paymentToken; // 支払いに使われるERC20
          uint basePrice;
          uint extra;
          uint listingTime;
          uint expirationTime;
          uint salt; // hash値が被るのを防ぐ
      }

```
      
   
      


<p></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>このOrderに操作を施すことで、出品やオファー、出品キャンセル等を実現します。具体的な流れを見ていきましょう。</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2 id="出品">出品</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>ExchangeCore ContractのapproveOrder_を実行します。引数の情報から新たなOrderを作成し、approveOrder内でOrderのhash値を求めます。approvedOrders[hash] = trueによりOrderの登録が完了です。approvedOrders[hash] のbool値のみで出品状態を表すというスマートな設計になっています。Order自体をstorageに保存するのではなく、そのhash値をうまく活用することで使用領域が減り、ガス代を節約できます。ところが、その一見スマートな設計が今回は仇となってしまいました。</p>
<!-- /wp:paragraph -->

<!-- wp:syntaxhighlighter/code -->

```solidity
bytes32 hash = hashToSign(order);
 
/* Assert order has not already been approved. */
require(!approvedOrders[hash]);
 
/* EFFECTS */
     
/* Mark order as approved. */
approvedOrders[hash] = true
emit OrderApprovedPartOne(...)
emit OrderApprovedPartTwo(...)
```

<!-- /wp:syntaxhighlighter/code -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2 id="購入">購入</h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>atomicMatch_(…)を実行します。購入の条件を満たしていれば、ETHのtransferが行われ、続いてsafeTransferFromによりNFTの所有が移ります。購入の条件を満たしているかどうかの確認では、atomicMatchの引数に渡されている出品情報から該当するOrder及びそのhashを求めて、approvedOrders[hash] = true であるかどうかを確かめています。</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":1} -->
<h1 id="ハッキングされた経緯">ハッキングされた経緯</h1>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>今回、不正な取引が行われた直接の原因は、NFTの所有者が移ってもapprovedOrders[hash] = trueのままであったことです。<br>具体的に見ていきましょう。</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>AliceがAというNFTを持っていたとします。Aliceは1ETHでAを出品しました。ところが、AというNFTが今後値上がりしそうだと考えたAliceは一度Aの出品をキャンセルしようと思いました。より高い価格で売るためです。</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>この時、素直に出品をキャンセルするとどうしてもガス代が高くつきます。そこでAliceは妙案を思いつきます。自分の管理している他のアカウントAlice'にAを一時的に移すことにしました。</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>そしてAlice'からAliceへとAを送り直します。一度所有者が変わると、OpenSeaのサイト上では出品がキャンセルされたことになるので、正規の方法より安く出品をキャンセルすることができます。(2)</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p><br>こうしてOpenSeaのサイト上では出品がキャンセルされたように見えます。ところが、Market Contract上ではapprovedOrders[hash] = true となったままです。つまり、出品はキャンセルされていません。よってatomicMatch_(…)を実行すれば、以前の出品価格でAを購入することができます。これが今回の問題の本質です。</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":1} -->
<h1 id="問題の深刻さ">問題の深刻さ</h1>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>今回の事件?ではこのようにして、以前出品されていた安い価格で人気のNFTが買われてしまいました。<br>このバグの深刻なところは影響範囲が広いことと、特権などを必要とせず誰でも再現できてしまうところです。<br>OpenSeaは現段階(日本時間の2022/1/25/15:00)では特に声明をを出していないようですが、近いうちに対策がとられると思います。というか早急に対策がなされなければいけません。</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":1} -->
<h1 id="終わりに">終わりに</h1>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>端的にまとめると今回の問題の原因は、オフチェーンで管理されているデータとオンチェーンで管理されているデータの相違です。一度デプロイしてしまったContractを修正するのは簡単ではありませんし、対応にも時間がかかります。また、OpenSeaのような大規模なサービスの問題が及ぼす影響は凄まじく、ユーザーが被る大きな不利益、ブロックチェーン技術自体への不信感の増長など目も当てられないほどです。</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>NFTやWeb3というのはまだまだ未熟です。確かにOpenSeaにはセキュアな設計を徹底する責任はありますが、それと共にユーザーもある程度のリテラシーが求められるのだと再認識しました。</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>(1) … コントラクト上のpublicで誰でも実行可能な関数が実行されただけなので、不正というべきかはわかりません。<br>(2) … OpenSea Market ContractのcancelOrder関数を実行するより、NFTのtransferを二回実行する方が安くすむというのが本当かどうかはまだ確証を得ていません。が、cancelOrder関数内ではhashの計算など重い処理をするのでガス代が高くつくと思われます。</p>
<!-- /wp:paragraph -->
