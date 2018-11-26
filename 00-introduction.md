# BOLT #0: Introduction and Index

Welcome, friend! These Basis of Lightning Technology (BOLT) documents
describe a layer-2 protocol for off-chain bitcoin transfer by mutual
cooperation, relying on on-chain transactions for enforcement if
necessary.

ようこそ、みんな！
これらのLightning Technology（BOLT）基本文書は、
必要に応じてon-chain transactionsを執行する、
相互連携によるoff-chain bitcoin転送のためのlayer-2 protocolを記述している。

Some requirements are subtle; we have tried to highlight motivations
and reasoning behind the results you see here. I'm sure we've fallen
short; if you find any part confusing or wrong, please contact us and
help us improve.

いくつかの要件は理解しにくい。
我々はあなたがここにある結果の背後にある動機と論拠を強調しようとした。
我々は不足があると確信している。
混乱したり間違った部分が見つかった場合は、我々に連絡し助力を頂きたい。

This is version 0.

これはバージョン0である。

1. [BOLT #1](01-messaging.md): Base Protocol
2. [BOLT #2](02-peer-protocol.md): Peer Protocol for Channel Management
3. [BOLT #3](03-transactions.md): Bitcoin Transaction and Script Formats
4. [BOLT #4](04-onion-routing.md): Onion Routing Protocol
5. [BOLT #5](05-onchain.md): Recommendations for On-chain Transaction Handling
7. [BOLT #7](07-routing-gossip.md): P2P Node and Channel Discovery
8. [BOLT #8](08-transport.md): Encrypted and Authenticated Transport
9. [BOLT #9](09-features.md): Assigned Feature Flags
10. [BOLT #10](10-dns-bootstrap.md): DNS Bootstrap and Assisted Node Location
11. [BOLT #11](11-payment-encoding.md): Invoice Protocol for Lightning Payments

## The Spark: A Short Introduction to Lightning

Lightning is a protocol for making fast payments with Bitcoin using a
network of channels.

Lightningは、チャネルのネットワークを使用してBitcoinで迅速に支払いを行うためのプロトコルである。

### Channels

Lightning works by establishing *channels*: two participants create a
Lightning payment channel that contains some amount of bitcoin (e.g.,
0.1 bitcoin) that they've locked up on the Bitcoin network. It is
spendable only with both their signatures.

Lightningはチャネルを確立して動作する：
2人の参加者が、
Bitcoinネットワークにロックアップしているある量のbitcoin（例：0.1 bitcoin）を含む
Lightning payment channelを作成する。
それは両方の署名のみで使用することができる。

Initially they each hold a bitcoin transaction that sends all the
bitcoin (e.g. 0.1 bitcoin) back to one party.  They can later sign a new bitcoin
transaction that splits these funds differently, e.g. 0.09 bitcoin to one
party, 0.01 bitcoin to the other, and invalidate the previous bitcoin
transaction so it won't be spent.

最初に彼らは、すべてのbitcoin（例：0.1 bitcoin）を1つのパーティに返すbitcoin transactionを保持する。
彼らは、その後これらの資金を異なるように分割する新しいbitcoin transactionに署名することができる。
たとえば、0.09 bitcoinを一方のパーティに、0.01ビットコインを他方に、
そして以前のbitcoin transactionは無効にされ使用されなくなる。

See [BOLT #2: Channel Establishment](02-peer-protocol.md#channel-establishment) for more on
channel establishment and [BOLT #3: Funding Transaction Output](03-transactions.md#funding-transaction-output) for the format of the bitcoin transaction that creates the channel.  See [BOLT #5: Recommendations for On-chain Transaction Handling](05-onchain.md) for the requirements when participants disagree or fail, and the cross-signed bitcoin transaction must be spent.

channel establishmentの詳細については「BOLT #2: Channel Establishment」を、
チャネルを作成するbitcoin transactionのフォーマットについては「BOLT #3: Funding Transaction Output」を参照。
参加者が不一致または失敗した場合、およびクロスサインされたbitcoin transactionが使用されるべき要件については、
「BOLT #5: Recommendations for On-chain Transaction Handling」を参照。

### Conditional Payments

A Lightning channel only allows payment between two participants, but channels can be connected together to form a network that allows payments between all members of the network. This requires the technology of a conditional payment, which can be added to a channel,
e.g. "you get 0.01 bitcoin if you reveal the secret within 6 hours".
Once the recipient presents the secret, that bitcoin transaction is
replaced with one lacking the conditional payment and adding the funds
to that recipient's output.

Lightning channelは2人の参加者間の支払いのみを可能にするが、
チャネルを一緒に接続してネットワークのすべてのメンバー間の支払いを可能にするネットワークを形成することができる。
これには条件付き支払いの技術が必要であり、これはチャネルに追加することができる。
たとえば、「secretを6時間以内に明らかにすれば0.01ビットコインを得る」となる。
受取人がsecretを提示すると、そのbitcoin transactionは条件付き支払いがないものと置き換えられ、受取人の出力に資金が追加される。

See [BOLT #2: Adding an HTLC](02-peer-protocol.md#adding-an-htlc-update_add_htlc) for the commands a participant uses to add a conditional payment, and [BOLT #3: Commitment Transaction](03-transactions.md#commitment-transaction) for the
complete format of the bitcoin transaction.

参加者が条件付き支払いを追加するために使用するコマンドについては「BOLT #2: Adding an HTLC」を、
bitcoin transactionの完全なフォーマットについては「BOLT #3: Commitment Transaction」を参照。

### Forwarding

Such a conditional payment can be safely forwarded to another
participant with a lower time limit, e.g. "you get 0.01 bitcoin if you reveal the secret
within 5 hours".  This allows channels to be chained into a network
without trusting the intermediaries.

そのような条件付き支払いは、例えば「あなたが5時間以内にsecretを明らかにすれば0.01ビットコインを得る」など、
時間制限の低い別の参加者に安全に転送することができます。
これにより、仲介者を信頼することなく、チャネルをネットワークに連鎖させることができる。

See [BOLT #2: Forwarding HTLCs](02-peer-protocol.md#forwarding-htlcs) for details on forwarding payments, [BOLT #4: Packet Structure](04-onion-routing.md#packet-structure) for how payment instructions are transported.

中継支払いの詳細については「BOLT #2: Forwarding HTLCs」を、
支払い命令が輸送される方法については「BOLT #4: Packet Structure」を参照。

### Network Topology

To make a payment, a participant needs to know what channels it can
send through.  Participants tell each other about channel and node
creation, and updates.

支払いを行うには、参加者はどのチャネルを通ることができるかを知る必要がある。
参加者は、チャネルとノードの作成と更新についてお互いに話す。

See [BOLT #7: P2P Node and Channel Discovery](07-routing-gossip.md)
for details on the communication protocol, and [BOLT #10: DNS
Bootstrap and Assisted Node Location](10-dns-bootstrap.md) for initial
network bootstrap.

通信プロトコルの詳細については「BOLT #7: P2P Node and Channel Discovery」を、
最初のネットワークブートストラップは「BOLT #10: DNS Bootstrap and Assisted Node Location」を参照。

### Payment Invoicing

A participant receives invoices that tell her what payments to make.

参加者は、何をなすかを彼女に伝える請求書を受け取る。

See [BOLT #11: Invoice Protocol for Lightning Payments](11-payment-encoding.md) for the protocol describing the destination and purpose of a payment such that the payer can later prove successful payment.

支払人が後で支払いをうまく証明できるように、支払いの宛先と目的を説明するプロトコルについては、
「BOLT #11: Invoice Protocol for Lightning Payments」を参照。

## Glossary and Terminology Guide

* *Announcement*:
   * A gossip message sent between *peers* intended to aid the discovery of a *channel* or a *node*.


* Announcement：
   * channelまたはnodeの発見を支援するためのpeers間で送信されるgossip message。

* `chain_hash`:
   * The uniquely identifying hash of the target blockchain (usually the genesis hash).
     This allows *nodes* to create and reference *channels* on
     several blockchains. Nodes are to ignore any messages that reference a
     `chain_hash` that are unknown to them. Unlike `bitcoin-cli`, the hash is
     not reversed but is used directly.

     For the main chain Bitcoin blockchain, the `chain_hash` value MUST be
     (encoded in hex):
     `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000`.

* chain_hash：
   * target blockchainを一意に識別するhash（通常はgenesis hash）。
     これにより、nodesは複数のblockchains上にchannelsを作成し、参照することができる。
     nodesは、それらにとって未知のchain_hashを参照するmessagesを無視する。
     bitcoin-cliとは異なり、hashは逆順にされずにそのまま使用される。

     main chain Bitcoin blockchainの場合、chain_hashの値は（16進数で）以下のように符号化されなければならない： 6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000。

     (XXX: BTCとBCHのgenesis hashは同じだろうから、BCHはLNで使えないのだろうか？)

* *Channel*:
   * A fast, off-chain method of mutual exchange between two *peers*.
   To transact funds, peers exchange signatures to create an updated *commitment transaction*.
   * _See closure methods: mutual close, revoked transaction close, unilateral close_
   * _See related: route_

* Channel：
   * 2つのpeers間の相互交換の、高速でoff-chainの手法。
   資金を取引するために、peersは署名を交換して最新のcommitment transactionを作成する。
   * closure methodsを参照：mutual close、revoked transaction close、unilateral close
   * _See related: route_


* *Closing transaction*:
   * A transaction generated as part of a _mutual close_. A closing transaction is similar to a _commitment transaction_, but with no pending payments.
   * _See related: commitment transaction, funding transaction, penalty transaction_

* Closing transaction：
   * mutual closeの一部として生成されたtransaction。
   closing transactionは、commitment transactionに似ているが、未払いの支払いはない。
   * _See related: commitment transaction, funding transaction, penalty transaction_

* *Commitment number*:
   * A 48-bit incrementing counter for each *commitment transaction*; counters
    are independent for each *peer* in the *channel* and start at 0.
   * _See container: commitment transaction_
   * _See related: closing transaction, funding transaction, penalty transaction_

* Commitment number：
   * それぞれのcommitment transactionのための、48ビットの増分カウンタ。
   カウンタはchannel内の各peerに対して独立しており、0から開始する。
   * _See container: commitment transaction_
   * _See related: closing transaction, funding transaction, penalty transaction_

* *Commitment revocation private key*:
   * Every *commitment transaction* has a unique commitment revocation private-key
    value that allows the other *peer* to spend all outputs
    immediately: revealing this key is how old commitment
    transactions are revoked. To support revocation, each output of the
    commitment transaction refers to the commitment revocation public key.
   * _See container: commitment transaction_
   * _See originator: per-commitment secret_

* Commitment revocation private key：
   * すべてのcommitment transactionには、
   他のpeerがすべてのアウトプットを即座に費やすことを可能にするユニークなcommitment revocation private-keyの値がある。
   （XXX: 当方じゃなく「他の」方であるということが大事）
   このキーを明らかにすることは、古いcommitment transactionsを取り消す方法である。
   revocationをサポートするために、commitment transactionの各出力は、commitment revocation public keyを参照する。
   * _See container: commitment transaction_
   * _See originator: per-commitment secret_

* *Commitment transaction*:
   * A transaction that spends the *funding transaction*.
   Each *peer* holds the other peer's signature for this transaction, so that each
   always has a commitment transaction that it can spend. After a new
   commitment transaction is negotiated, the old one is *revoked*.
   * _See parts: commitment number, commitment revocation private key, HTLC, per-commitment secret_
   * _See related: closing transaction, funding transaction, penalty transaction_
   * _See types: revoked commitment transaction_

* Commitment transaction：
   * funding transactionを費やすtransaction。
   各peerは、このtransactionのために他のpeerの署名を保持するので、それぞれは常に、それが費やすことができるcommitment transactionがある。
   新しいcommitment transactionがネゴシエートされると、古いものはrevokedになる。
   * _See parts: commitment number, commitment revocation private key, HTLC, per-commitment secret_
   * _See related: closing transaction, funding transaction, penalty transaction_
   * _See types: revoked commitment transaction_

* *Final node*:
   * The final recipient of a packet that is routing a payment from an _origin node_ through some number of _hops_. It is also the final *receiving peer* in a chain.
   * _See category: node_
   * _See related: origin node, processing node_

* Final node：
   * original nodeからいくつかのhopsを経た支払いをルーティングしているパケットの最終的な受信者。
   それはチェーン内の最後のreceiving peerでもあります。
   * _See category: node_
   * _See related: origin node, processing node_

* *Funding transaction*:
   * An irreversible on-chain transaction that pays to both *peers* on a *channel*.
   It can only be spent by mutual consent.
   * _See related: closing transaction, commitment transaction, penalty transaction_

* Funding transaction：
   * channel上の両方のpeersに支払いのある不可逆的なオンチェーンのtransaction。
   それは相互の同意によってのみ費やすことができる。
   * _See related: closing transaction, commitment transaction, penalty transaction_

* *Hop*:
   * A *node*. Generally, an intermediate node lying between an *origin node* and a *final node*.
   * _See category: node_

* Hop：
   * node。一般に、origin nodeとfinal nodeとの間に位置するintermediate node。
   * _See category: node_

* *HTLC*: Hashed Time Locked Contract.
   * A conditional payment between two *peers*: the recipient can spend
    the payment by presenting its signature and a *payment preimage*,
    otherwise the payer can cancel the contract by spending it after
    a given time. These are implemented as outputs from the
    *commitment transaction*.
   * _See container: commitment transaction_
   * _See parts: Payment hash, Payment preimage_

* HTLC： Hashed Time Locked Contract.
   * 2つのpeers間の条件付き支払い：
   受取人は、その署名とpayment preimageを提示することによって支払いを費やすことができ、
   そうでなければ、支払人は所定時間後にcontractを取り消すことができる。
   これらはcommitment transactionのアウトプットとして実施される。
   * _See container: commitment transaction_
   * _See parts: Payment hash, Payment preimage_

* *Invoice*: A request for funds on the Lightning Network, possibly
    including payment type, payment amount, expiry, and other
    information. This is how payments are made on the Lightning
    Network, rather than using Bitcoin-style addresses.

* Invoice：
   * Lightning Network上の資金の要求。
   支払いの種類、支払い金額、有効期限、その他の情報が含まれる可能性がある。
   Bitcoin形式のアドレスを使用するのではなく、Lightning Networkで支払いを行う方法である。

* *It's ok to be odd*:
   * A rule applied to some numeric fields that indicates either optional or
     compulsory support for features. Even numbers indicate that both endpoints
     MUST support the feature in question, while odd numbers indicate
     that the feature MAY be disregarded by the other endpoint.

* It's ok to be odd：
   * 機能がオプションであるか強制サポートであるかを示す数値フィールドに適用されるルール。
   偶数は、問題の機能を両方のエンドポイントがサポートしなければならないことを示すが、
   奇数は機能が他のエンドポイントによって無視しても良いことを示す。

* *MSAT*:
   * A millisatoshi, often used as a field name.

* MSAT：
   * millisatoshi、しばしばフィールド名として使われる。

* *Mutual close*:
   * A cooperative close of a *channel*, accomplished by broadcasting an unconditional
    spend of the *funding transaction* with an output to each *peer*
    (unless one output is too small, and thus is not included).
   * _See related: revoked transaction close, unilateral close_

* Mutual close：
   * （1つの出力が小さすぎて、故に含まれない場合を除いて、）
   各peersへの出力を伴う、funding transactionの条件のない消費を、ブロードキャストすることによって達成される、
   channelの協調的なclose。
   * _See related: revoked transaction close, unilateral close_

* *Node*:
   * A computer or other device that is part of the Lightning network.
   * _See related: peers_
   * _See types: final node, hop, origin node, processing node, receiving node, sending node_

* Node：
   * Lightning networkの一部であるコンピュータまたはその他のデバイス。
   * _See related: peers_
   * _See types: final node, hop, origin node, processing node, receiving node, sending node_

* *Origin node*:
   * The _node_ that originates a packet that will route a payment through some number of _hops_ to a _final node_. It is also the first _sending peer_ in a chain.
   * _See category: node_
   * _See related: final node, processing node_

* Origin node：
   * いくつかhopsを経由してfinal nodeまで支払いを送付するパケットを発信するnode。
   それはチェーン内の最初のsending peerでもある。
   * _See category: node_
   * _See related: final node, processing node_

* *Payment hash*:
   * The *HTLC* contains the payment hash, which is the hash of the
    *payment preimage*.
   * _See container: HTLC_
   * _See originator: payment preimage_

* Payment hash：
   * HTLCはpayment preimageのhashであるpayment hashを含む。
   * _See container: HTLC_
   * _See originator: payment preimage_

* *Payment preimage*:
   * Proof that payment has been received, held by
    the final recipient, who is the only person who knows this
    secret. The final recipient releases the preimage in order to
    release funds. The payment preimage is hashed as the *payment hash*
    in the *HTLC*.
   * _See container: HTLC_
   * _See derivation: payment hash_

* Payment preimage：
   * このsecretを知っている唯一の人である最終受取人が支払いを受けたことの証明。
   最終的な受信者は、資金を解放するためにpreimageを解放する。
   payment preimageは 、HTLCのpayment hashとしてハッシュされる。
   * _See container: HTLC_
   * _See derivation: payment hash_

* *Peers*:
   * Two *nodes* that are in communication with each other.
      * Two peers may gossip with each other prior to setting up a channel.
      * Two peers may establish a *channel* through which they transact.
   * _See related: node_

* Peers：
   * 互いに通信する2つのnodes。
      * channelを設定する前に、2人のpeersがお互いにgossipするであろう。
      * 2つのpeersは、それらが取引するchannelを確立するであろう。
      (XXX: gossipはchannel確立と独立して行って良い)
   * _See related: node_

* *Penalty transaction*:
   * A transaction that spends all outputs of a *revoked commitment
    transaction*, using the *commitment revocation private key*. A *peer* uses this
    if the other peer tries to "cheat" by broadcasting a *revoked
    commitment transaction*.
   * _See related: closing transaction, commitment transaction, funding transaction_

* Penalty transaction：
   * commitment revocation private keyを使用して、revoked commitment transactionのすべての出力を費やすtransaction。
   peerは、他のpeerがcommitment transactionをブロードキャストしてずる「cheat」しようとすると、これを使用する。
   * _See related: closing transaction, commitment transaction, funding transaction_

* *Per-commitment secret*:
   * Every *commitment transaction* derives its keys from a per-commitment secret,
     which is generated such that the series of per-commitment secrets
     for all previous commitments can be stored compactly.
   * _See container: commitment transaction_
   * _See derivation: commitment revocation private key_

* Per-commitment secret：
   * すべてのcommitment transactionはper-commitment secretからそのキーを導出する。
   per-commitment secretは、コンパクトに格納できるように、
   すべての以前のcommitmentsのための一連のper-commitment secretから生成される。
   （XXX: shachain）
   * _See container: commitment transaction_
   * _See derivation: commitment revocation private key_

* *Processing node*:
   * A *node* that is processing a packet that originated with an *origin node* and that is being sent toward a *final node* in order to route a payment. It acts as a _receiving peer_ to receive the message, then a _sending peer_ to send on the packet.
   * _See category: node_
   * _See related: final node, origin node_

* Processing node：
   origin nodeで起こり、final nodeへの支払いを送付するために送信されるパケットを処理しているnode。
   これは、メッセージを受信するreceiving peerとして機能し、それからパケットを送信するsending peerとして機能する。
   * _See category: node_
   * _See related: final node, origin node_

* *Receiving node*:
   * A *node* that is receiving a message.
   * _See category: node_
   * _See related: sending node_

* Receiving node：
   * メッセージを受信しているnode。
   * _See category: node_
   * _See related: sending node_

* *Receiving peer*:
   * A *node* that is receiving a message from a directly connected *peer*.
   * _See category: peer_
   * _See related: sending peer_

* Receiving peer：
   * 直接接続しているpeerからメッセージを受信しているnode。
   * _See category: peer_
   * _See related: sending peer_

* *Revoked commitment transaction*:
   * An old *commitment transaction* that has been revoked because a new commitment transaction has been negotiated.
   * _See category: commitment transaction_

* Revoked commitment transaction：
   * 新しいcommitment transactionがネゴシエートされたために取り消された古いcommitment transaction。
   * _See related: mutual close, unilateral close_

* *Revoked transaction close*:
   * An invalid close of a *channel*, accomplished by broadcasting a *revoked
    commitment transaction*. Since the other *peer* knows the
    *commitment revocation secret key*, it can create a *penalty transaction*.
   * _See related: mutual close, unilateral close_

* Revoked transaction close：
   * revoked commitment transactionをブロードキャストすることによって達成される、
   不正なchannelのclose。
   他方のpeerはcommitment revocation secret keyを知っているので、
   penalty transactionを作成することができる。
   * _See related: mutual close, unilateral close_

* *Route*: A path across the Lightning Network that enables a payment
    from an *origin node* to a *final node* across one or more
    *hops*.
  * _See related: channel_

* Route：
  * 1つ以上のhopsを横断し、origin nodeからfinal nodeまでの支払いを可能にするLightning Networkのパス。
  * _See related: channel_

* *Sending node*:
   * A *node* that is sending a message.
   * _See category: node_
   * _See related: receiving node_

* Sending node：
   * メッセージを送信しているnode。
   * _See category: node_
   * _See related: receiving node_

* *Sending peer*:
   * A *node* that is sending a message to a directly connected *peer*.
   * _See category: peer_
   * _See related: receiving peer_.

* Sending peer：
   * 直接接続しているpeerへにメッセージを送信しているnode。
   * _See category: peer_
   * _See related: receiving peer_.

* *Unilateral close*:
   * An uncooperative close of a *channel*, accomplished by broadcasting a
    *commitment transaction*. This transaction is larger (i.e. less
    efficient) than a *mutual close transaction*, and the *peer* whose
    commitment is broadcast cannot access its own outputs for some
    previously-negotiated duration.
   * _See related: mutual close, revoked transaction close_

* Unilateral close：
   * commitment transactionをブロードキャストすることによって達成される、非協力的なchannelのclose。
   このtransactionは、mutual close transactionよりも大きく（すなわち、非効率）、
   commitmentがブロードキャストされているpeerは、
   以前にネゴシエートされた期間は、自身のoutputsにアクセスできない。
   * _See related: mutual close, revoked transaction close_

## Theme Song

      Why this network could be democratic...
      Numismatic...
      Cryptographic!
      Why it could be released Lightning!
      (Release Lightning!)


      We'll have some timelocked contracts with hashed pubkeys, oh yeah.
      (Keep talking, whoa keep talkin')
      We'll segregate the witness for trustless starts, oh yeah.
      (I'll get the money, I've got to get the money)
      With dynamic onion routes, they'll be shakin' in their boots;
      You know that's just the truth, we'll be scaling through the roof.
      Release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      [Chorus:]
      Oh released Lightning, it's better than a debit card..
      (Release Lightning, go release Lightning!)
      With released Lightning, micropayments just ain't hard...
      (Release Lightning, go release Lightning!)
      Then kaboom: we'll hit the moon -- release Lightning!
      (Go, go, go, go; go, go, go, go, go, go)


      We'll have QR codes, and smartphone apps, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      P2P messaging, and passive incomes, oh yeah.
      (Ooo ooo ooo ooo ooo ooo ooo)
      Outsourced closure watch, gives me feelings in my crotch.
      You'll know it's not a brag when the repo gets a tag:
      Released Lightning.


      [Chorus]
      [Instrumental, ~1m10s]
      [Chorus]
      (Lightning! Lightning! Lightning! Lightning!
       Lightning! Lightning! Lightning! Lightning!)


      C'mon guys, let's get to work!


   -- Anthony Towns <aj@erisian.com.au>

## Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
