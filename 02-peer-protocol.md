# BOLT #2: Peer Protocol for Channel Management

The peer channel protocol has three phases: establishment, normal
operation, and closing.

ピアチャネルプロトコルには、
establishment、
normal operation、
およびclosingという3つのフェーズがある。

（XXX: 以下、updateのpendingという言葉に曖昧さがあると思う。例えば以下の複数の状態が考えられる。<br>
自身のcommit txについて（相手のものについても対称的に同じ）、<br>
・updateをpeerに送ったがまだ自身のcommit txに落ちていない状態<br>
・update（送受信双方）がpeerからcommitされていない状態<br>
・updateがpeerからcommitされたがまだ前のcommit txがrevokeされていない状態<br>
）

# Table of Contents

  * [Channel](#channel)
    * [Channel Establishment](#channel-establishment)
      * [The `open_channel` Message](#the-open_channel-message)
      * [The `accept_channel` Message](#the-accept_channel-message)
      * [The `funding_created` Message](#the-funding_created-message)
      * [The `funding_signed` Message](#the-funding_signed-message)
      * [The `funding_locked` Message](#the-funding_locked-message)
    * [Channel Close](#channel-close)
      * [Closing Initiation: `shutdown`](#closing-initiation-shutdown)
      * [Closing Negotiation: `closing_signed`](#closing-negotiation-closing_signed)
    * [Normal Operation](#normal-operation)
      * [Forwarding HTLCs](#forwarding-htlcs)
      * [`cltv_expiry_delta` Selection](#cltv_expiry_delta-selection)
      * [Adding an HTLC: `update_add_htlc`](#adding-an-htlc-update_add_htlc)
      * [Removing an HTLC: `update_fulfill_htlc`, `update_fail_htlc`, and `update_fail_malformed_htlc`](#removing-an-htlc-update_fulfill_htlc-update_fail_htlc-and-update_fail_malformed_htlc)
      * [Committing Updates So Far: `commitment_signed`](#committing-updates-so-far-commitment_signed)
      * [Completing the Transition to the Updated State: `revoke_and_ack`](#completing-the-transition-to-the-updated-state-revoke_and_ack)
      * [Updating Fees: `update_fee`](#updating-fees-update_fee)
    * [Message Retransmission: `channel_reestablish` message](#message-retransmission)
  * [Authors](#authors)

# Channel

## Channel Establishment

After authenticating and initializing a connection ([BOLT #8](08-transport.md)
and [BOLT #1](01-messaging.md#the-init-message), respectively), channel establishment may begin.
This consists of the funding node (funder) sending an `open_channel` message,
followed by the responding node (fundee) sending `accept_channel`. With the
channel parameters locked in, the funder is able to create the funding
transaction and both versions of the commitment transaction, as described in
[BOLT #3](03-transactions.md#bolt-3-bitcoin-transaction-and-script-formats).
The funder then sends the outpoint of the funding output with the `funding_created`
message, along with the signature for the fundee's version of the commitment
transaction. Once the fundee learns the funding outpoint, it's able to
generate the funder's commitment for the commitment transaction and send it
over using the `funding_signed` message.

認証および接続の初期化後（それぞれBOLT #8とBOLT #1）、channel establishmentが開始されるであろう。
これは、fundingノード（funder）がopen_channelメッセージを送信し、
応答ノード（fundee）がaccept_channelを送信することで構成される。
チャネルパラメータが固定されると、
funderは、BOLT＃3で説明される、
funding transactionと、commitment transactionの両方のバージョンを登録することができる。
その後、funderは、funding outputのoutpointをfunding_createdメッセージとともに、
「fundeeのバージョンのcommitment transaction」への（XXX: funderによる）署名を加えて送信する。
fundeeがfunding outputを知ると、commitment transactionのためのfunderのコミットメント（XXX: 署名）を生成し、
それをfunding_signedメッセージを使用して送信することができる。

Once the channel funder receives the `funding_signed` message, it
must broadcast the funding transaction to the Bitcoin network. After
the `funding_signed` message is sent/received, both sides should wait
for the funding transaction to enter the blockchain and reach the
specified depth (number of confirmations). After both sides have sent
the `funding_locked` message, the channel is established and can begin
normal operation. The `funding_locked` message includes information
that will be used to construct channel authentication proofs.

チャネルのfunderがfunding_signedメッセージを受信すると、
Bitcoinネットワークにfunding transactionをブロードキャストしなければならない。
funding_signedメッセージの送受信後、双方はfunding transactionがブロックチェーンに入り、
指定された深さ（確認数）に達するのを待つべきである。
両方の側がfunding_lockedメッセージを送信した後、チャネルが確立され、normal operationが開始される。
funding_lockedメッセージは、チャネル認証の証明（XXX: ？）を構築するために使用される情報を含む。


        +-------+                              +-------+
        |       |--(1)---  open_channel  ----->|       |
        |       |<-(2)--  accept_channel  -----|       |
        |       |                              |       |
        |   A   |--(3)--  funding_created  --->|   B   |
        |       |<-(4)--  funding_signed  -----|       |
        |       |                              |       |
        |       |--(5)--- funding_locked  ---->|       |
        |       |<-(6)--- funding_locked  -----|       |
        +-------+                              +-------+

        - where node A is 'funder' and node B is 'fundee'

If this fails at any stage, or if one node decides the channel terms
offered by the other node are not suitable, the channel establishment
fails.

いずれかの段階でこれが失敗した場合、
または一方のノードが他方のノードによって提供されるチャネルの条件が適切でないと判断した場合、
channel establishmentは失敗する。

Note that multiple channels can operate in parallel, as all channel
messages are identified by either a `temporary_channel_id` (before the
funding transaction is created) or a `channel_id` (derived from the
funding transaction).

すべてのチャネルメッセージは、temporary_channel_id（funding transactionが作成される前）
またはchannel_id（funding transactionから導出される）いずれかによって識別されるので、
複数のチャネルを並行して運用することができる。

### The `open_channel` Message

This message contains information about a node and indicates its
desire to set up a new channel. This is the first step toward creating
the funding transaction and both versions of the commitment transaction.

このメッセージにはノードに関する情報が含まれており、新しいチャネルを設定したいという欲求を示している。
これは、funding transactionとcommitment transactionの両方のバージョンを作成するための第一歩である。

1. type: 32 (`open_channel`)
2. data:
   * [`32`:`chain_hash`]
   * [`32`:`temporary_channel_id`]
   * [`8`:`funding_satoshis`]
   * [`8`:`push_msat`]
   * [`8`:`dust_limit_satoshis`]
   * [`8`:`max_htlc_value_in_flight_msat`]
   * [`8`:`channel_reserve_satoshis`]
   * [`8`:`htlc_minimum_msat`]
   * [`4`:`feerate_per_kw`]
   * [`2`:`to_self_delay`]
   * [`2`:`max_accepted_htlcs`]
   * [`33`:`funding_pubkey`]
   * [`33`:`revocation_basepoint`]
   * [`33`:`payment_basepoint`]
   * [`33`:`delayed_payment_basepoint`]
   * [`33`:`htlc_basepoint`]
   * [`33`:`first_per_commitment_point`]
   * [`1`:`channel_flags`]
   * [`2`:`shutdown_len`] (`option_upfront_shutdown_script`)
   * [`shutdown_len`:`shutdown_scriptpubkey`] (`option_upfront_shutdown_script`)

The `chain_hash` value denotes the exact blockchain that the opened channel will
reside within. This is usually the genesis hash of the respective blockchain.
The existence of the `chain_hash` allows nodes to open channels
across many distinct blockchains as well as have channels within multiple
blockchains opened to the same peer (if it supports the target chains).

chain_hashの値は、開いているチャネルが存在する正確なブロックチェーンを示す。
これは、通常、それぞれのブロックチェーンのgenesis hashである。
chain_hashの存在により、多くの異なるブロックチェーンにわたるチャネルを開くことができ、
複数のブロックチェーン内のチャネルを同じピアにオープンすることもできる
（ターゲットチェーンをサポートしている場合）。

The `temporary_channel_id` is used to identify this channel until the
funding transaction is established, at which point it is replaced
by the `channel_id`, which is derived from the funding transaction.

temporary_channel_idは、
funding transactionが確立されるまでこのチャネルを識別するために使用される、
その時点でfunding transactionから導出したchannel_idに置き換えられる。

`funding_satoshis` is the amount the sender is putting into the
channel. `push_msat` is an amount of initial funds that the sender is
unconditionally giving to the receiver. `dust_limit_satoshis` is the
threshold below which outputs should not be generated for this node's
commitment or HTLC transactions (i.e. HTLCs below this amount plus
HTLC transaction fees are not enforceable on-chain). This reflects the
reality that tiny outputs are not considered standard transactions and
will not propagate through the Bitcoin network. `channel_reserve_satoshis`
is the minimum amount that the other node is to keep as a direct
payment. `htlc_minimum_msat` indicates the smallest value HTLC this
node will accept.

funding_satoshisは、送信者がチャンネルに入れている金額である。
push_msatは、送信者が無条件に受信者に与えている初期資金の金額である。
dust_limit_satoshisは、
このノードのcommitmentまたはHTLC transactionsに対してoutputsが生成されるべきではない閾値である
（つまり、この額足すHTLC transaction feesを下回るHTLCsはオンチェーンで実行可能ではない）。
これは、小さなoutputsが標準的なトランザクションとはみなされず、
Bitcoinネットワークを介して伝播しないという現実を反映している。
channel_reserve_satoshisは、他のノードが直接支払いとして保持する最小量である。
（XXX: この分は資金に残さないといけない。資金があっても送信できない分）
htlc_minimum_msatは、このノードが受け入れる最小値のHTLCを示す。

`max_htlc_value_in_flight_msat` is a cap on total value of outstanding
HTLCs, which allows a node to limit its exposure to HTLCs; similarly,
`max_accepted_htlcs` limits the number of outstanding HTLCs the other
node can offer.

max_htlc_value_in_flight_msatは、処理中のHTLCの総価格に対する上限であり、
これはノードがHTLCへの公開（XXX: 資金の割り当て）を制限することを可能にする、
同様に、 max_accepted_htlcsは他のノードが提供できる処理中のHTLCの数を制限する。

`feerate_per_kw` indicates the initial fee rate in satoshi per 1000-weight
(i.e. 1/4 the more normally-used 'satoshi per 1000 vbytes') that this
side will pay for commitment and HTLC transactions, as described in
[BOLT #3](03-transactions.md#fee-calculation) (this can be adjusted
later with an `update_fee` message).

feerate_per_kwは、BOLT＃3に記載されているように、
こちら側がcommitmentおよびHTLC transactionsに対して支払う1000-weight
（すなわち、より一般的に使用される'satoshi per 1000 vbytes'の1/4）
毎の初期fee rateを示す（これは後でupdate_feeメッセージで調整可能である）。
（XXX: マイナーへのfee）
（XXX: vbyte = virtual byte, 1 vbyte = 4 weight）

`to_self_delay` is the number of blocks that the other node's to-self
outputs must be delayed, using `OP_CHECKSEQUENCEVERIFY` delays; this
is how long it will have to wait in case of breakdown before redeeming
its own funds.

to_self_delayは、OP_CHECKSEQUENCEVERIFY遅延を使用して、
他のノードのto-self outputsを遅延させる必要があるブロックの数である。
これは、故障した場合に自分の資金を償還する前にどれくらい待たなければならないかである。
（XXX: to_self_delayは相手へのoutputにかかる）

`funding_pubkey` is the public key in the 2-of-2 multisig script of
the funding transaction output.

funding_pubkeyは、
funding transaction outputの2-of-2 multisigスクリプトのpublic keyである。

The various `_basepoint` fields are used to derive unique
keys as described in [BOLT #3](03-transactions.md#key-derivation) for each commitment
transaction. Varying these keys ensures that the transaction ID of
each commitment transaction is unpredictable to an external observer,
even if one commitment transaction is seen; this property is very
useful for preserving privacy when outsourcing penalty transactions to
third parties.

さまざまな_basepointフィールドは、BOLT＃3で説明されているように、
固有のキーを派生させるために各commitment transactionで使用される。
これらのキーを変更することにより、commitment transactionが1つ見られても、
各commitment transactionのtransaction IDが外部の観察者に予測不可能になる。
このプロパティは、penalty transactionsを第三者に委託する際のプライバシー保護に非常に役立つ。
（XXX: commitment transactionのTXIDがランダマイズされて予想不可能になっているが、それは、
txinのsequenceによる。これはpayment_basepointで隠されたcommitment numberである）

`first_per_commitment_point` is the per-commitment point to be used
for the first commitment transaction,

first_per_commitment_pointは、最初のcommitment transactionに使用されるper-commitment pointである。
（XXX: 最後カンマ間違い）

Only the least-significant bit of `channel_flags` is currently
defined: `announce_channel`. This indicates whether the initiator of
the funding flow wishes to advertise this channel publicly to the
network, as detailed within [BOLT #7](07-routing-gossip.md#bolt-7-p2p-node-and-channel-discovery).

channel_flagsの最下位ビットのみが現在定義されている：announce_channel。
これは、資金フローの開始者が、BOLT＃7内に詳述されているように、
このチャネルをネットワークに公的にアドバタイズしたいかどうかを示す。

The `shutdown_scriptpubkey` allows the sending node to commit to where
funds will go on mutual close, which the remote node should enforce
even if a node is compromised later.

shutdown_scriptpubkeyにより、
送信ノードがファンドがmutal closeに進むべき場所を約束する、
ノードが後で妥協された場合でも（XXX: ？）、リモートノードが強制すべき。

[ FIXME: Describe dangerous feature bit for larger channel amounts. ]

[ FIXME：より大きなチャンネル量の危険な機能ビットを説明する。 ]

#### Requirements

The sending node:
  - MUST ensure the `chain_hash` value identifies the chain it wishes to open the channel within.
  - MUST ensure `temporary_channel_id` is unique from any other channel ID with the same peer.
  - MUST set `funding_satoshis` to less than 2^24 satoshi.
  - MUST set `push_msat` to equal or less than 1000 * `funding_satoshis`.
  - MUST set `funding_pubkey`, `revocation_basepoint`, `htlc_basepoint`, `payment_basepoint`, and `delayed_payment_basepoint` to valid DER-encoded, compressed, secp256k1 pubkeys.
  - MUST set `first_per_commitment_point` to the per-commitment point to be used for the initial commitment transaction, derived as specified in [BOLT #3](03-transactions.md#per-commitment-secret-requirements).
  - MUST set `channel_reserve_satoshis` greater than or equal to `dust_limit_satoshis`.
  - MUST set undefined bits in `channel_flags` to 0.
  - if both nodes advertised the `option_upfront_shutdown_script` feature:
    - MUST include either a valid `shutdown_scriptpubkey` as required by `shutdown` `scriptpubkey`, or a zero-length `shutdown_scriptpubkey`.
  - otherwise:
    - MAY include a`shutdown_scriptpubkey`.

送信ノードは：
  - chain_hash値が、チャネルを開くことを望むチェーンを識別していることを保証しなければならない。
  - temporary_channel_idが、同じピアとの他のどのチャネルIDからも一意であることを保証しなければならない。
  - funding_satoshisは、2 ^ 24未満のsatoshiに設定する必要がある。
  - push_msatは、1000 * funding_satoshis以下に設定する必要がある。（XXX: funding_satoshisよりも少なく）
  - funding_pubkey、revocation_basepoint、htlc_basepoint、payment_basepoint、およびdelayed_payment_basepointは、
  有効なDERでエンコードされ、圧縮されたsecp256k1 pubkeysに設定されなければならない。
  - first_per_commitment_pointは、BOLT＃3で指定されたとおりに導出された、
  初期commitment transactionに使用されるper-commitment pointに設定する必要がある。
  - channel_reserve_satoshisは、dust_limit_satoshis以上に設定しなければならない。
  - channel_flagsの定義されていないビットを、0に設定しなければならない。
  - 両方のノードがoption_upfront_shutdown_script機能をアドバタイズした場合：
    - shutdown scriptpubkeyによって必要とされる、有効なshutdown_scriptpubkey、
    または長さがゼロのshutdown_scriptpubkeyのいずれかを含む必要がある。
  - そうでなければ：
    - shutdown_scriptpubkeyを含めてもよい。（XXX: ？）

The sending node SHOULD:
  - set `to_self_delay` sufficient to ensure the sender can irreversibly spend a commitment transaction output, in case of misbehavior by the receiver.
  - set `feerate_per_kw` to at least the rate it estimates would cause the transaction to be immediately included in a block.
  - set `dust_limit_satoshis` to a sufficient value to allow commitment transactions to propagate through the Bitcoin network.
  - set `htlc_minimum_msat` to the minimum value HTLC it's willing to accept from this peer.

送信ノードはすべき：
  - to_self_delayは、受取人による不正行為の場合には、
  送付者がcommitment transaction outputを不可逆的に使うことができるのに十分になるように設定する。
  - feerate_per_kwは、少なくともトランザクションが直ちにブロックに含まれると推定されるレートを設定する。
  - dust_limit_satoshisは、commitment transactionsがBitcoinネットワークを介して伝播するのに十分な値に設定する。
  - htlc_minimum_msatは、このピアから受け入れたい最小値のHTLCに設定する。

The receiving node MUST:
  - ignore undefined bits in `channel_flags`.
  - if the connection has been re-established after receiving a previous
 `open_channel`, BUT before receiving a `funding_created` message:
    - accept a new `open_channel` message.
    - discard the previous `open_channel` message.

受信ノードはしなければならない：
  - channel_flagsの未定義ビットを無視する。
  - 前のopen_channelを受信したが、funding_createdメッセージを受信する前に接続が再確立された場合：
    - 新しいopen_channelメッセージを受け入れる。
    - 前のopen_channelメッセージを破棄する。

The receiving node MAY fail the channel if:
  - `announce_channel` is `false` (`0`), yet it wishes to publicly announce the channel.
  - `funding_satoshis` is too small.
  - it considers `htlc_minimum_msat` too large.
  - it considers `max_htlc_value_in_flight_msat` too small.
  - it considers `channel_reserve_satoshis` too large.
  - it considers `max_accepted_htlcs` too small.
  - it considers `dust_limit_satoshis` too small and plans to rely on the sending node publishing its commitment transaction in the event of a data loss (see [message-retransmission](02-peer-protocol.md#message-retransmission)).

次の場合、受信ノードはチャネルに失敗してよい：
  - それがチャンネルを公表することを希望するが、announce_channelがfalse（0）である。
  - funding_satoshが小さすぎる。
  - htlc_minimum_msatが、大きすぎると考えれられる。
  - max_htlc_value_in_flight_msatが、小さすぎると考えられる。
  - channel_reserve_satoshisが、大きすぎる考えらえる。
  - max_accepted_htlcsが、小さすぎると考えられる。
  - dust_limit_satoshisが、小さすぎると考えられ、
  データ損失の場合にcommitment transactionを公開している送信ノードに頼る予定である（message-retransmission参照）。
  （XXX: ？）

The receiving node MUST fail the channel if:
  - the `chain_hash` value is set to a hash of a chain that is unknown to the receiver.
  - `push_msat` is greater than `funding_satoshis` * 1000.
  - `to_self_delay` is unreasonably large.
  - `max_accepted_htlcs` is greater than 483.
  - it considers `feerate_per_kw` too small for timely processing or unreasonably large.
  - `funding_pubkey`, `revocation_basepoint`, `htlc_basepoint`, `payment_basepoint`, or `delayed_payment_basepoint`
are not valid DER-encoded compressed secp256k1 pubkeys.
  - `dust_limit_satoshis` is greater than `channel_reserve_satoshis`.
  - the funder's amount for the initial commitment transaction is not sufficient for full [fee payment](03-transactions.md#fee-payment).
  - both `to_local` and `to_remote` amounts for the initial commitment transaction are less than or equal to `channel_reserve_satoshis` (see [BOLT 3](03-transactions.md#commitment-transaction-outputs)).

次の場合、受信ノードはチャネルに失敗しなければならない：
  - セットされているchain_hashの値が受信者にとって未知のチェーンのハッシュである
  - push_msatがfunding_satosh * 1000 より大きい。
  - to_self_delayが不当に大きい。
  - max_accepted_htlcsが483より大きい。
  - feerate_per_kwがタイムリーに処理するには小さすぎるか（XXX: オンチェーンのfeeは変動するため）、
  不当に大きいと考えられる。
  - funding_pubkey、revocation_basepoint、htlc_basepoint、payment_basepoint、またはdelayed_payment_basepointが、
  有効なDERでエンコードされた、圧縮された、secp256k1のpubkeysではないと考えらえる。
  - dust_limit_satoshisがchannel_reserve_satoshisより大きい。
  - funderの最初のcommitment transactionのためのファンドの金額が、完全なfee paymentには不十分である。
  - to_localとto_remoteの両方の最初のcommitment transactionの金額が、
  channel_reserve_satoshis以下（BOLT 3参照）。
  （XXX: push_msatで資金が動くかもしれないが、それでも条件を満たすようにする）

The receiving node MUST NOT:
  - consider funds received, using `push_msat`, to be received until the funding transaction has reached sufficient depth.

受信ノードはしてはならない：
  - funding transactionが十分な深さに達する前に、push_msatを使用して受信した資金を受信したものとみなす。

#### Rationale

The requirement for `funding_satoshi` to be less than 2^24 satoshi is a temporary self-imposed limit while implementations are not yet considered stable.
It can be lifted at any point in time, or adjusted for other currencies, since it is solely enforced by the endpoints of a channel.
Specifically, [the routing gossip protocol](07-routing-gossip.md) does not discard channels that have a larger capacity.

実装がまだ安定しているとは考えられないうちに、2^24未満のsatoshiのfunding_satoshiの要件は一時的な自己制約である。
それは、チャネルのエンドポイントによってのみ実施されるため、任意の時点で持ち上げることも、他の通貨に合わせて調整することもできる。
特に、ルーティングゴシッププロトコルは、より大きな容量を有するチャネルを廃棄しない。

The *channel reserve* is specified by the peer's `channel_reserve_satoshis`: 1% of the channel total is suggested. Each side of a channel maintains this reserve so it always has something to lose if it were to try to broadcast an old, revoked commitment transaction. Initially, this reserve may not be met, as only one side has funds; but the protocol ensures that there is always progress toward meeting this reserve, and once met, it is maintained.

channel reserveは、ピアのchannel_reserve_satoshisで指定される：
チャンネル総数の1％が提案されている。
チャネルの各側はこの予約を保持しているので、古いrevoked commitment transactionをブロードキャストしようとすると、失うものが常にある。
（XXX: 自分の残高が空になったときに試しにrevoked commitment transactionを行おうとするインセンティブをなくす。幾分かの残高が残るようにして。）
当初、この準備金は満たされていないかもしれない。片側だけが資金を持っているので。
このプロトコルでは、この準備金を満たすことに向けて常に前進があり、一旦満たされればそれが維持されることが保証されている。
（XXX: fundeeにとっては資金ははじめchannel_reserve_satoshisを下回っている、ので送信できないが、
溜まってもchannel_reserve_satoshisを上まらないと送信できない）

The sender can unconditionally give initial funds to the receiver using a non-zero `push_msat`, but even in this case we ensure that the funder has sufficient remaining funds to pay fees and that one side has some amount it can spend (which also implies there is at least one non-dust output). Note that, like any other on-chain transaction, this payment is not certain until the funding transaction has been confirmed sufficiently (with a danger of double-spend until this occurs) and may require a separate method to prove payment via on-chain confirmation.

送付者はゼロでないpush_msatで、無条件で最初の資金を受領者に与えることができますが、
この場合でも我々は、funderにはfeesを支払うのに十分な残高があり、
一方の側にはいくらかの使用できる額があることを保証する
（これはまた少なくとも1つのnon-dust outputがあることを暗示する）。
（XXX: funder側じゃなくone sideと書いてあるのは、push_msatで資金が移動しているかもしれないから）
他のon-chain transactionと同様に、この支払い（XXX: push_msatによる）はfunding transactionが十分に確認されるまで確かでなく（これが生じるまで二重支出の危険性がある）、
on-chain confirmationによる支払いの証明のための別の方法が必要になるであろう。
（XXX: push_msatによる支払いの証明はBOLT #13では行えない）

The `feerate_per_kw` is generally only of concern to the sender (who pays the fees), but there is also the fee rate paid by HTLC transactions; thus, unreasonably large fee rates can also penalize the recipient.

一般的にfeerate_per_kwは送信者のみ（手数料を支払う）の関心であるが、
HTLC transactionsにより支払われる手数料率でもある。
従って、不当に大きな料金率も受取人にペナルティを科す可能性がある。

Separating the `htlc_basepoint` from the `payment_basepoint` improves security: a node needs the secret associated with the `htlc_basepoint` to produce HTLC signatures for the protocol, but the secret for the `payment_basepoint` can be in cold storage.

payment_basepointからhtlc_basepointを分離することは、セキュリティを向上させる：
ノードが、このプロトコルのHTLCシグネチャを生成するのに、
htlc_basepointに関連付けられたsecretを必要とするが、
payment_basepointのためのsecretはcold storageに置くことができる。
（XXX: payment_basepointはcommitment transactionでhtlc_basepointはHTLCに関わる）

The requirement that `channel_reserve_satoshis` is not considered dust
according to `dust_limit_satoshis` eliminates cases where all outputs
would be eliminated as dust.  The similar requirements in
`accept_channel` ensure that both sides' `channel_reserve_satoshis`
are above both `dust_limit_satoshis`.

channel_reserve_satoshisがdust_limit_satoshisに関連してdustとみなされない
（XXX: ようにする）という要件は、
、すべてのoutputsがdustとして排除されるケースを排除する。
accept_channelにおける同様の要件は、
両サイドのchannel_reserve_satoshisが、両方のdust_limit_satoshisを超えることを保証する。

Details for how to handle a channel failure can be found in [BOLT 5:Failing a Channel](05-onchain.md#failing-a-channel).

チャネル障害を処理する方法の詳細は「BOLT 5:Failing a Channel」にある。

#### Future

It would be easy to have a local feature bit which indicated that a
receiving node was prepared to fund a channel, which would reverse this
protocol.

受信ノードが、
このプロトコルを逆転させるチャネルに資金提供する用意ができていることを示すlocal feature bitを持つことは容易であろう。

### The `accept_channel` Message

This message contains information about a node and indicates its
acceptance of the new channel. This is the second step toward creating the
funding transaction and both versions of the commitment transaction.

このメッセージにはノードに関する情報が含まれており、新しいチャネルの受け入れを示してる。
これは、funding transactionと両方のバージョンのcommitment transactionを作成するための第２のステップである。

1. type: 33 (`accept_channel`)
2. data:
   * [`32`:`temporary_channel_id`]
   * [`8`:`dust_limit_satoshis`]
   * [`8`:`max_htlc_value_in_flight_msat`]
   * [`8`:`channel_reserve_satoshis`]
   * [`8`:`htlc_minimum_msat`]
   * [`4`:`minimum_depth`]
   * [`2`:`to_self_delay`]
   * [`2`:`max_accepted_htlcs`]
   * [`33`:`funding_pubkey`]
   * [`33`:`revocation_basepoint`]
   * [`33`:`payment_basepoint`]
   * [`33`:`delayed_payment_basepoint`]
   * [`33`:`htlc_basepoint`]
   * [`33`:`first_per_commitment_point`]
   * [`2`:`shutdown_len`] (`option_upfront_shutdown_script`)
   * [`shutdown_len`:`shutdown_scriptpubkey`] (`option_upfront_shutdown_script`)

#### Requirements

The `temporary_channel_id` MUST be the same as the `temporary_channel_id` in
the `open_channel` message.

temporary_channel_idは、open_channelメッセージのtemporary_channel_idと同じでなければならない。

The sender:
  - SHOULD set `minimum_depth` to a number of blocks it considers reasonable to
avoid double-spending of the funding transaction.
  - MUST set `channel_reserve_satoshis` greater than or equal to `dust_limit_satoshis` from the `open_channel` message.
  （XXX: こっちのchannel_reserve_satoshisにはなんの意味があるのか？）
  - MUST set `dust_limit_satoshis` less than or equal to `channel_reserve_satoshis` from the `open_channel` message.

送信者：
  - minimum_depthは、
  funding transactionの二重使用を避けるために合理的であると考えられるブロックの数に設定すべきである。
  - channel_reserve_satoshisは、
  open_channelメッセージのdust_limit_satoshis以上の値を設定しなければならない。
  - dust_limit_satoshisは、
  open_channelメッセージのchannel_reserve_satoshis以下の値を設定しなければならない。

The receiver:
  - if `minimum_depth` is unreasonably large:
    - MAY reject the channel.
  - if `channel_reserve_satoshis` is less than `dust_limit_satoshis` within the `open_channel` message:
	- MUST reject the channel.
  - if `channel_reserve_satoshis` from the `open_channel` message is less than `dust_limit_satoshis`:
	- MUST reject the channel.

Other fields have the same requirements as their counterparts in `open_channel`.

受信者：
  - minimum_depthが不当に大きい場合：
    - チャンネルを拒否してもよい。
  - channel_reserve_satoshisが、
  open_channelメッセージ内のdust_limit_satoshisより少ない場合：
    - チャネルを拒絶しなければならない。
  - open_channelメッセージのchannel_reserve_satoshisが、
  dust_limit_satoshisより少ない場合：
    - チャネルを拒絶しなければならない。

他のフィールドには、open_channelの対応するフィールドと同じ要件がある。

### The `funding_created` Message

This message describes the outpoint which the funder has created for
the initial commitment transactions. After receiving the peer's
signature, via `funding_signed`, it will broadcast the funding transaction.

このメッセージは、資金提供者が初期commitment transactionsのために作成したoutpointを示す。
funding_signedを通してピアの署名を受け取った後、それはfunding transactionをブロードキャストする。
（XXX: これはfunderが送る）

1. type: 34 (`funding_created`)
2. data:
    * [`32`:`temporary_channel_id`]
    * [`32`:`funding_txid`]
    * [`2`:`funding_output_index`]
    * [`64`:`signature`]

#### Requirements

The sender MUST set:
  - `temporary_channel_id` the same as the `temporary_channel_id` in the `open_channel` message.
  - `funding_txid` to the transaction ID of a non-malleable transaction,
    - and MUST NOT broadcast this transaction.
  - `funding_output_index` to the output number of that transaction that corresponds the funding transaction output, as defined in [BOLT #3](03-transactions.md#funding-transaction-output).
  - `signature` to the valid signature using its `funding_pubkey` for the initial commitment transaction, as defined in [BOLT #3](03-transactions.md#commitment-transaction).

送信者は以下を設定しなければならない：
  - temporary_channel_idは、
  open_channelメッセージのtemporary_channel_idと同じである。
  - funding_txidは、
  非展性のトランザクションのtransaction ID、
    - このtransactionをブロードキャストしてはならない。
  - funding_output_indexは、
  BOLT＃3で定義されているように、
  funding transaction outputに対応するそのtransactionのoutput number。
  - signatureは、BOLT＃3で定義されているように、それのfunding_pubkeyを使った、
  最初のcommit transactionのための有効な署名。

The sender:
  - when creating the funding transaction:
    - SHOULD use only BIP141 (Segregated Witness) inputs.

送信者：
  - funding transactionを作成するとき：
    - BIP141（Segregated Witness）inputsのみを使用すべきである。

The recipient:
  - if `signature` is incorrect:
    - MUST fail the channel.

受信者：
  - もしsignature正しくないならば：
    - チャネルに失敗しなければならない。

#### Rationale

The `funding_output_index` can only be 2 bytes, since that's how it's packed into the `channel_id` and used throughout the gossip protocol. The limit of 65535 outputs should not be overly burdensome.

funding_output_indexは、
それがchannel_idにパックされてgossip protocolを通して使われているので、2バイトのみである。
65535の出力の上限は過度に負担にならないはずである。

A transaction with all Segregated Witness inputs is not malleable, hence the funding transaction recommendation.

すべてのSegregated Witness inputsを伴うtransactionは展性がないので、
funding transactionの推奨である。

### The `funding_signed` Message

This message gives the funder the signature it needs for the first
commitment transaction, so it can broadcast the transaction knowing that funds
can be redeemed, if need be.

このメッセージは、最初のcommitment transactionに必要な署名をfunderに提供するので、
必要に応じて、ファンドが償還できることを知ってtransactionをブロードキャストできる。

This message introduces the `channel_id` to identify the channel. It's derived from the funding transaction by combining the `funding_txid` and the `funding_output_index`, using big-endian exclusive-OR (i.e. `funding_output_index` alters the last 2 bytes).

このメッセージは、チャネルを識別するchannel_idを導入する。
これは、funding_txidとfunding_output_indexを結合することによってfunding transactionから導出され、
ビッグエンディアンの排他的論理和を使用する（すなわちfunding_output_indexが最後の2バイトを変更する）。

1. type: 35 (`funding_signed`)
2. data:
    * [`32`:`channel_id`]
    * [`64`:`signature`]

#### Requirements

The sender MUST set:
  - `channel_id` by exclusive-OR of the `funding_txid` and the `funding_output_index` from the `funding_created` message.
  - `signature` to the valid signature, using its `funding_pubkey` for the initial commitment transaction, as defined in [BOLT #3](03-transactions.md#commitment-transaction).

送信者は以下を設定しなければならない：
  - channel_idは、
  funding_createdのメッセージからの、funding_txidおよびfunding_output_indexの排他的論理和。
  - signatureは、
  BOLT＃3で定義されているように、それのfunding_pubkeyを使った、最初のcommit transactionのための有効な署名。

The recipient:
  - if `signature` is incorrect:
    - MUST fail the channel.
  - MUST NOT broadcast the funding transaction before receipt of a valid `funding_signed`.
  - on receipt of a valid `funding_signed`:
    - SHOULD broadcast the funding transaction.

受信者：
  - もしsignatureが正しくないならば：
    - チャネルに失敗しなければならない。
  - 有効なfunding_signedを受け取る前にfunding transactionをブロードキャストしてはならない。
  - 有効なfunding_signedを受け取ったとき：
    - funding transactionをブロードキャストすべきである。

### The `funding_locked` Message

This message indicates that the funding transaction has reached the `minimum_depth` asked for in `accept_channel`. Once both nodes have sent this, the channel enters normal operating mode.

このメッセージは、funding transactionがaccept_channelで依頼されたminimum_depthに達したことを示す。
両方のノードがこれを送信すると、チャネルはnormal operatingモードになる。

1. type: 36 (`funding_locked`)
2. data:
    * [`32`:`channel_id`]
    * [`33`:`next_per_commitment_point`]

#### Requirements

The sender MUST:
  - wait until the funding transaction has reached
`minimum_depth` before sending this message.
  - set `next_per_commitment_point` to the
per-commitment point to be used for the following commitment
transaction, derived as specified in
[BOLT #3](03-transactions.md#per-commitment-secret-requirements).

送信者はしなければならない：
  - このメッセージを送信する前に、funding transactionがminimum_depthに達するまで待つ。
  - next_per_commitment_pointは、BOLT #3で指定されているように、
  次に続くcommitment transactionに使用されるper-commitmentに設定される。
  （XXX: これはsecretから計算されている）

A non-funding node (fundee):
  - SHOULD forget the channel if it does not see the
funding transaction after a reasonable timeout.

fundingしていないノード（fundee）：
  - 妥当なタイムアウト後にfunding transactionが見られない場合は、チャネルを忘れすべきである。

From the point of waiting for `funding_locked` onward, either node MAY
fail the channel if it does not receive a required response from the
other node after a reasonable timeout.

funding_lockedを待っている時点から、妥当なタイムアウト後に他方のノードから必要な応答を受信しない場合、
どちらのノードもチャネルを失敗してもよい。

#### Rationale

The non-funder can simply forget the channel ever existed, since no
funds are at risk. If the fundee were to remember the channel forever, this
would create a Denial of Service risk; therefore, forgetting it is recommended
(even if the promise of `push_msat` is significant).

非funderは、資金が危険にさらされていないので、これまで存在していたチャネルを単に忘れることができる。
fundeeがチャネルを永遠に記憶しておけば、Denial of Service（DoS）リスクが発生する。
従って、それを忘れることは推奨される（たとえ、push_msatの約束が重要であっても）。

#### Future

An SPV proof could be added and block hashes could be routed in separate
messages.

SPV proof（全ブロックのPoW検証に必要なデータ）を追加し、
ブロックハッシュを別のメッセージでルーティングすることができる。
（XXX: block headerや必要なmarkle tree。gossipで送ることが検討されている）

## Channel Close

Nodes can negotiate a mutual close of the connection, which unlike a
unilateral close, allows them to access their funds immediately and
can be negotiated with lower fees.

ノードは、unilateral closeとは異なり、即座に資金にアクセスすることができ、
より低い手数料で交渉することができる、mutual closeを交渉することができる。

Closing happens in two stages:
1. one side indicates it wants to clear the channel (and thus will accept no new HTLCs)
2. once all HTLCs are resolved, the final channel close negotiation begins.

クローズは2つの段階で行われる：
1. 片側がチャネルをクリアしたいことを示す（従って、新しいHTLCsを受け入れないことを示す）
2. すべてのHTLCsが解決されると、最終的なチャネルクローズネゴシエーションが開始される。

        +-------+                              +-------+
        |       |--(1)-----  shutdown  ------->|       |
        |       |<-(2)-----  shutdown  --------|       |
        |       |                              |       |
        |       | <complete all pending HTLCs> |       |
        |   A   |                 ...          |   B   |
        |       |                              |       |
        |       |--(3)-- closing_signed  F1--->|       |
        |       |<-(4)-- closing_signed  F2----|       |
        |       |              ...             |       |
        |       |--(?)-- closing_signed  Fn--->|       |
        |       |<-(?)-- closing_signed  Fn----|       |
        +-------+                              +-------+

### Closing Initiation: `shutdown`

Either node (or both) can send a `shutdown` message to initiate closing,
along with the `scriptpubkey` it wants to be paid to.

いずれかのノード（またはその両方）が、そこへ支払を希望するscriptpubkeyとともに、
終了を開始するshutdownメッセージを送信することができる。

1. type: 38 (`shutdown`)
2. data:
   * [`32`:`channel_id`]
   * [`2`:`len`]
   * [`len`:`scriptpubkey`]

#### Requirements

A sending node:
  - if it hasn't sent a `funding_created` (if it is a funder) or a `funding_signed` (if it is a fundee):
    - MUST NOT send a `shutdown`
  - MAY send a `shutdown` before a `funding_locked`, i.e. before the funding transaction has reached `minimum_depth`.
  - if there are updates pending on the receiving node's commitment transaction:
    - MUST NOT send a `shutdown`.
  - MUST NOT send an `update_add_htlc` after a `shutdown`.
  - if no HTLCs remain in either commitment transaction:
	- MUST NOT send any `update` message after a `shutdown`.
  - SHOULD fail to route any HTLC added after it has sent `shutdown`.
  - if it sent a non-zero-length `shutdown_scriptpubkey` in `open_channel` or `accept_channel`:
    - MUST send the same value in `scriptpubkey`.
  - MUST set `scriptpubkey` in one of the following forms:

    1. `OP_DUP` `OP_HASH160` `20` 20-bytes `OP_EQUALVERIFY` `OP_CHECKSIG`
   (pay to pubkey hash), OR
    2. `OP_HASH160` `20` 20-bytes `OP_EQUAL` (pay to script hash), OR
    3. `OP_0` `20` 20-bytes (version 0 pay to witness pubkey), OR
    4. `OP_0` `32` 32-bytes (version 0 pay to witness script hash)

送信ノード：
  - funding_created（funderの場合）またはfunding_signed（fundeeの場合）が送られていない場合：
    - shutdownを送信してはならない
  - funding_lockedを送る前にshutdownを送っても良い、
  すなわちfunding transactionがminimum_depthに達する前に。
  - 受信ノードのcommitment transactionで保留中の更新がある場合：
  （XXX: ここのpendingがどのような状態かはっきりしない）
    - shutdownを送信してはならない。
  - shutdown後にupdate_add_htlcを送ってはならない。
  - いずれのcommitment transactionにもHTLCが残っていない場合：
    - shutdownの後にupdateメッセージを送信してはならない。
  - shutdownが送信された後に、追加されたHTLCをルーティングすべきではない。
  （XXX: 失敗させるのか）
  - open_channelまたはaccept_channelで、
  それが0以外の長さのshutdown_scriptpubkeyを送信した場合、
    - scriptpubkeyで同じ値を送らなければならない。
  - scriptpubkeyは、次のいずれかの形式で設定しなければならない。
    1. `OP_DUP` `OP_HASH160` `20` 20バイト `OP_EQUALVERIFY` `OP_CHECKSIG` （P2PKH）、または
    2. `OP_HASH160` `20` 20バイト `OP_EQUAL`（P2SH）、または
    3. `OP_0` `20` 20バイト（バージョン0 P2WPKH）、または
    4. `OP_0` `32` 32バイト（バージョン0 P2WSH）

A receiving node:
  - if it hasn't received a `funding_signed` (if it is a funder) or a `funding_created` (if it is a fundee):
    - SHOULD fail the connection
  - if the `scriptpubkey` is not in one of the above forms:
    - SHOULD fail the connection.
  - if it hasn't sent a `funding_locked` yet:
    - MAY reply to a `shutdown` message with a `shutdown`
  - once there are no outstanding updates on the peer, UNLESS it has already sent a `shutdown`:
    - MUST reply to a `shutdown` message with a `shutdown`
  - if both nodes advertised the `option_upfront_shutdown_script` feature, and the receiving node received a non-zero-length `shutdown_scriptpubkey` in `open_channel` or `accept_channel`, and that `shutdown_scriptpubkey` is not equal to `scriptpubkey`:
    - MUST fail the connection.

受信ノード：
  - funding_signed（funderの場合）またはfunding_created（fundeeの場合）が受信されていない場合：
    - 接続に失敗するべきである
  - scriptpubkeyが上記のいずれかの形式でない場合：
    - 接続に失敗するべきである。
  - それがまだfunding_lockedを送信していない場合：
    - shutdownメッセージにshutdownで応答することができる
  - いったんピアに未処理の更新がない場合、すでにそれがshutdownを送信していない限り：
    - shutdownメッセージにshutdownで返信しなければならない
  - 両方のノードがoption_upfront_shutdown_script featureをアドバタイズし、
  受信ノードが、open_channelまたはaccept_channelの中の、ゼロでない長さのshutdown_scriptpubkeyを受信した場合に、
  shutdown_scriptpubkeyとscriptpubkeyが等しくない場合：
    - 接続に失敗するべきである。

#### Rationale

If channel state is always "clean" (no pending changes) when a
shutdown starts, the question of how to behave if it wasn't is avoided:
the sender always sends a `commitment_signed` first.

シャットダウンが開始されたときにチャネル状態が常に「クリーン」（保留中の変更なし）の場合、
そうでない場合の振る舞いの問題は避けられる：
送信者は常にcommitment_signedを最初に送信する。
（XXX: 双方ともshutdownを送るまでに気にするべきなのは、
相手のcommitment transactionの状態のみ。
自分のshutdownと相手のshutdownの間に、相手からのcommitment_signedが来ることもあるだろう。
相手からのrevoke_and_ackやcommitment_signedが送ってこなくて処理が進まない場合は、
unirateral closeするしかない？
local commitment transactionにこれ以上更新が必要なければ、
revoke_and_ackは別に来なくてもいいんじゃないか？）

As shutdown implies a desire to terminate, it implies that no new
HTLCs will be added or accepted.  Once any HTLCs are cleared, the peer
may immediately begin closing negotiation, so we ban further updates
to the commitment transaction (in particular, `update_fee` would be
possible otherwise).

シャットダウンは終了の希望を暗示しているため、それは新しいHTLCが追加または受け入れないことを暗示する。
HTLCが一度クリアされると、ピアはすぐに交渉を終了する可能性があるため、
commitment transactionへの更なる更新を禁止する
（特に、そうでなければ（XXX: HTLCが残っていれば）update_feeは可能であろう）。

The `scriptpubkey` forms include only standard forms accepted by the
Bitcoin network, which ensures the resulting transaction will
propagate to miners.

このscriptpubkeyフォームには、Bitcoinネットワークで受け入れられた標準形式のみが含まれているため、
結果的なtransactionが確実にマイナーに伝播する。

The `option_upfront_shutdown_script` feature means that the node
wanted to pre-commit to `shutdown_scriptpubkey` in case it was
compromised somehow.  This is a weak commitment (a malevolent
implementation tends to ignore specifications like this one!), but it
provides an incremental improvement in security by requiring the cooperation
of the receiving node to change the `scriptpubkey`.

option_upfront_shutdown_script機能は、
ノードが何らかの形で妥協されたケースshutdown_scriptpubkeyに事前コミットしたかったことを意味する。
（XXX: ？）
これは弱いコミットメントだが（悪意のある実装はこのような仕様を無視する傾向がある）が、
scriptpubkeyを変えることによって、受信ノードの協力を必要とすることにより、セキュリティの段階的な改善を提供する。

The `shutdown` response requirement implies that the node sends `commitment_signed` to commit any outstanding changes before replying; however, it could theoretically reconnect instead, which would simply erase all outstanding uncommitted changes.

shutdown応答要件は、
ノードが返信する前に、すべての未処理の変更をコミットするために、commitment_signedを送信することを暗示する：
ただし、それは理論的には代わりに再接続することができ、
これにより、コミットされていないすべての未解決の変更が単純に消去される。

### Closing Negotiation: `closing_signed`

Once shutdown is complete and the channel is empty of HTLCs, the final
current commitment transactions will have no HTLCs, and closing fee
negotiation begins.  The funder chooses a fee it thinks is fair, and
signs the close transaction with the `scriptpubkey` fields from the
`shutdown` messages (along with its chosen fee) and sends the signature;
the other node then replies similarly, using a fee it thinks is fair.  This
exchange continues until both agree on the same fee or when one side fails
the channel.

一旦shutdownが完了し、チャネルのHTLCsが空になると、
最終的な現在のcommitment transactionにはHTLCsがなくなり、
closing fee negotiationが開始される。
funderは、公正であると思うfeeを選択し、shutdownメッセージのscriptpubkeyフィールドと共に
close transaction（XXX: closing transactionの間違い？）に署名し
（選択したfeeとともに）署名を送信する。
もう片方のノードは同様に、公平だと思う料金を使って同様に返信する。
この交換は、両方が同じ料金で同意するまで、または一方の側がチャネルに失敗するときまで続く。

1. type: 39 (`closing_signed`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`fee_satoshis`]
   * [`64`:`signature`]

#### Requirements

The funding node:
  - after `shutdown` has been received, AND no HTLCs remain in either commitment transaction:
    - SHOULD send a `closing_signed` message.

fundingノード：
  - shutdownが受信された後、いずれのcommitment transactionにもHTLCsは残っていない：
    - closing_signedメッセージを送信すべきである。
    （XXX: shutdownがまだ送信されてなくてもいいのか？なんでこれはfunding node？）

The sending node:
  - MUST set `fee_satoshis` less than or equal to the
 base fee of the final commitment transaction, as calculated in [BOLT #3](03-transactions.md#fee-calculation).
  - SHOULD set the initial `fee_satoshis` according to its
 estimate of cost of inclusion in a block.
  - MUST set `signature` to the Bitcoin signature of the close
 transaction, as specified in [BOLT #3](03-transactions.md#closing-transaction).

送信ノード：
  - BOLT＃3で計算されるように、fee_satoshisを、
  最後のcommitment transactionのbase fee以下に設定しなければならない。
  - 最初のfee_satoshisは、ブロックへの追加コストの推定値に従って設定すべきである。
  - BOLT＃3で指定されるように、signatureは、closing transactionのBitcoinシグニチャに設定しなければならない。

The receiving node:
  - if the `signature` is not valid for either variant of close transaction
  specified in [BOLT #3](03-transactions.md#closing-transaction):
    - MUST fail the connection.
  - if `fee_satoshis` is equal to its previously sent `fee_satoshis`:
    - SHOULD sign and broadcast the final closing transaction.
    - MAY close the connection.
  - otherwise, if `fee_satoshis` is greater than
the base fee of the final commitment transaction as calculated in
[BOLT #3](03-transactions.md#fee-calculation):
    - MUST fail the connection.
  - if `fee_satoshis` is not strictly
between its last-sent `fee_satoshis` and its previously-received
`fee_satoshis`, UNLESS it has since reconnected:
    - SHOULD fail the connection.
  - if the receiver agrees with the fee:
    - SHOULD reply with a `closing_signed` with the same `fee_satoshis` value.
  - otherwise:
    - MUST propose a value "strictly between" the received `fee_satoshis`
  and its previously-sent `fee_satoshis`.

受信ノード：
  - signatureが、BOLT＃3で指定されるように、closing transactionのいずれかの変種として有効ではない場合：
  （XXX: 変種が考えられる。自分のoutputをdustにしたりして）
    - 接続に失敗しなければならない。
  - fee_satoshisが以前に送信されたfee_satoshisと同じ場合：
    - 最後のclosing transactionに署名し、ブロードキャストすべきである。
    - 接続を閉じることができる。
  - そうでなければ、
  BOLT＃3で計算されるように、fee_satoshisが、最後のcommitment transactionのbase feeよりも大きい場合 ：
    - 接続に失敗しなければならない。
  - もしfee_satoshisが厳密にその送られた最後のfee_satoshisと以前に受信したfee_satoshisの間ではなく、
  以前に再接続していない限り：
    - 接続に失敗するべきである。
    （XXX: channelを失敗するわけではない。やりなおし）
  - 受信者がfeeに同意する場合：
    - 同じfee_satoshisの値でclosing_signedを返すべきである。
  - そうでなければ：
    - 受信したfee_satoshisと以前に送信したfee_satoshisの「厳密に間の」値を提示しなければならない。

#### Rationale

The "strictly between" requirement ensures that forward
progress is made, even if only by a single satoshi at a time. To avoid
keeping state and to handle the corner case, where fees have shifted
between disconnection and reconnection, negotiation restarts on reconnection.

「厳密には間の」という要件は、一度に1つのsatoshiであっても、進歩が確実に行われるようにする。
状態の保持を避け、切断と再接続の間でfeesがシフトした（XXX: やり直しのため）コーナーケースを処理するために、
再接続時にネゴシエーションが再開される。（XXX: shutdownからやり直し？）

Note there is limited risk if the closing transaction is
delayed, but it will be broadcast very soon; so there is usually no
reason to pay a premium for rapid processing.

closing transactionが遅れてもリスクは限定的だが、すぐにブロードキャストされることに注意すること；
従って、通常迅速な処理のためにプレミアムを支払う理由はない。

## Normal Operation

Once both nodes have exchanged `funding_locked` (and optionally [`announcement_signatures`](07-routing-gossip.md#the-announcement_signatures-message)), the channel can be used to make payments via Hashed Time Locked Contracts.

両方のノードがfunding_lockedを交換されると（オプションとしてannouncement_signatures
（XXX: announcement_signaturesを送らなくてもNormal Operationとなる。
announcement_signaturesはfunding_locked以降に送らないといけないためこのような注釈をつけたのであろうが、
不要だと思う））、
Hashed Time Locked Contrancts（HTLCs）を介して支払いを行うためにチャネルを使用することができる。
（XXX: announcement_signaturesが必要な時はそれを交換しないとNormal Operationに移行できない？）

Changes are sent in batches: one or more `update_` messages are sent before a
`commitment_signed` message, as in the following diagram:

変更はバッチで送信される。
次の図のように、commitment_signedメッセージの前に1つ以上のupdate_メッセージが送信される。

        +-------+                            +-------+
        |       |--(1)---- add_htlc   ------>|       |
        |       |--(2)---- add_htlc   ------>|       |
        |       |<-(3)---- add_htlc   -------|       |
        |       |                            |       |
        |       |--(4)----   commit   ------>|       |
        |   A   |                            |   B   |
        |       |<-(5)--- revoke_and_ack-----|       |
        |       |<-(6)----   commit   -------|       |
        |       |                            |       |
        |       |--(7)--- revoke_and_ack---->|       |
        |       |--(8)----   commit   ------>|       |
        |       |                            |       |
        |       |<-(9)--- revoke_and_ack-----|       |
        +-------+                            +-------+

（区切り）

        +-------+                                   +-------+
        |       |--(1)-----  update_add_htlc  ----->|       |
        |       |--(2)-----  update_add_htlc  ----->|       |
        |       |<-(3)-----  update_add_htlc  ------|       |
        |       |                                   |       |
        |       |--(4)----- commitment_signed ----->|       |
        |   A   |                                   |   B   |
        |       |<-(5)------- revoke_and_ack  ------|       |
        |       |<-(6)----- commitment_signed ------|       |
        |       |                                   |       |
        |       |--(7)------- revoke_and_ack  ----->|       |
        +-------+                                   +-------+

（XXX: なんで図でメッセージ名を省略する。。。）

Counter-intuitively, these updates apply to the *other node's*
commitment transaction; the node only adds those updates to its own
commitment transaction when the remote node acknowledges it has
applied them via `revoke_and_ack`.

直感的に反するが、これらの更新は、他のノードのcommitment transactionに適用される。
ノードは、遠隔ノードがrevoke_and_ackを介してそれを適用したことを確認すると、
それらの更新をそれ自身のcommitment transactionに追加するだけである。
（XXX: commitment_signedで相手のcommitment txに適用されたupdateのみが、
revoke_and_ack受信で自身のcommitment txに追加されることに注意。
commitment_signed送信とrevoke_and_ack受信の間のupdate送信については、
その次のrevoke_and_ackで自身のcommitment txに追加される。
）

Thus each update traverses through the following states:

1. pending on the receiver
2. in the receiver's latest commitment transaction
3. ... and the receiver's previous commitment transaction has been revoked,
   and the HTLC is pending on the sender
4. ... and in the sender's latest commitment transaction
5. ... and the sender's previous commitment transaction has been revoked

従って、各updateは、以下の状態を横断する。
（XXX: 対象はupdate）

1. （XXX: updateは）受信者で保留中
2. （XXX: updateは）受信者の最新のcommitment transactionにある
3. ... 受信者の前回のcommitment transactionが取り消され、
HTLCが送信者に保留中。
（XXX: TODO: ここでHTLCだけが対象になるのはおかしい。updateであろう）
4. ... （XXX: updateは）送信者の最新のcommitment transaction
5. ... 送信者の前回のcommitment transactionが取り消された
（XXX: update送信側でrevoke_and_ack前とcommitment_signed前とで状態の区別が必要であろう）

As the two nodes' updates are independent, the two commitment
transactions may be out of sync indefinitely. This is not concerning:
what matters is whether both sides have irrevocably committed to a
particular HTLC or not (the final state, above).

2つのノードの更新が独立しているので、2つのcommitment transactionは無期限に同期していない可能性がある。
これは関係ない：
重要なのは、両当事者が特定のHTLCがirrevocably committedであることを約束したか否かである（上の最終状態）。
（XXX: revoke_and_ackまででirrevocably committed）
（XXX: TODO: ここもHTLCではなく対象はupdateではないのか？）

### Forwarding HTLCs

In general, a node offers HTLCs for two reasons: to initiate a payment of its own,
or to forward another node's payment. In the forwarding case, care must
be taken to ensure the *outgoing* HTLC cannot be redeemed unless the *incoming*
HTLC can be redeemed. The following requirements ensure this is always true.

一般に、ノードは、自身の支払いを開始するか、別のノードの支払いを転送するかの2つの理由でHTLCsを提供する。
フォワーディングの場合、受信HTLCが償還されない限り、送信HTLCが償還されないように注意する必要がある。
次の要件は、これが常に真であることを保証する。

The respective **addition/removal** of an HTLC is considered *irrevocably committed* when:

1. The commitment transaction **with/without** it is committed to by both nodes, and any
previous commitment transaction **without/with** it has been revoked, OR
2. The commitment transaction **with/without** it has been irreversibly committed to
the blockchain.

それぞれのHTLCの（追加/削除）は、次の場合にirrevocably committedとみなされる。

1. 両方のノードでコミットされている（XXX: 全ての）commitment transactionで、そのHTLCが（ある／ない）
任意の以前の無効化されている（XXX: 全ての）commitment transactionで、そのHTLCが（ない／ある）
または、
2. ブロックチェーンにirreversibly committedされているcommitment transactionで、そのHTLCが（ある／ない）

#### Requirements

A node:
  - until the incoming HTLC has been irrevocably committed:
    - MUST NOT offer an HTLC (`update_add_htlc`) in response to an incoming HTLC.
  - until the removal of the outgoing HTLC is irrevocably committed, OR until the outgoing on-chain HTLC output has been spent via the HTLC-timeout transaction (with sufficient depth):
    - MUST NOT fail an incoming HTLC (`update_fail_htlc`) for which it has committed
to an outgoing HTLC.
  - once its `cltv_expiry` has been reached, OR if `cltv_expiry` minus `current_height` is less than `cltv_expiry_delta` for the outgoing channel:
    - MUST fail an incoming HTLC (`update_fail_htlc`).
  - if an incoming HTLC's `cltv_expiry` is unreasonably far in the future:
    - SHOULD fail that incoming HTLC (`update_fail_htlc`).
  - upon receiving an `update_fulfill_htlc` for the outgoing HTLC, OR upon discovering the `payment_preimage` from an on-chain HTLC spend:
    - MUST fulfill an incoming HTLC for which it has committed to an outgoing HTLC.

ノード：
  - 受信HTLCがirrevocably committedされるまで：
    - 受信HTLCに応答してHTLC（update_add_htlc）を提供してはいけない。
  - 送信HTLCの削除がirrevocably committedになるまで、
  またはオンチェーンの送信HTLC出力がHTLC-timeout transactionによって費やされるまで（十分な深さで）：
    - 送信HTLCのためにコミットした受信HTLCを失敗（update_fail_htlc）してはならない。
  - 一旦cltv_expiryに達するか、（cltv_expiry - current_height）が送信チャンネルのcltv_expiry_delta未満の場合：
    - 受信HTLCに失敗（update_fail_htlc）しなければならない。
    （XXX: すでにタイムアウトしている）
  - 受信HTLCのcltv_expiryが将来に不当に遠い場合：
    - 受信HTLCに失敗（update_fail_htlc）すべきである。
  - 送信HTLCのupdate_fulfill_htlc受信時か、オンチェーンのHTLCの使用からpayment_preimageの発見時
    - 送信HTLCにコミットした受信HTLCをupdate_fulfill_htlcしなければならない。

#### Rationale

In general, one side of the exchange needs to be dealt with before the other.
Fulfilling an HTLC is different: knowledge of the preimage is, by definition,
irrevocable and the incoming HTLC should be fulfilled as soon as possible to
reduce latency.

一般的に、取引の一方は他方よりも先に処理される必要がある。
HTLCの実行（fulfilling）は異なる。
プリイメージの認知は、定義上、取り消し不能であり、
受信HTLCは、待ち時間を短縮するためにできるだけ早く実行（be fulfilled）されるべきである。

An HTLC with an unreasonably long expiry is a denial-of-service vector and
therefore is not allowed. Note that the exact value of "unreasonable" is currently unclear
and may depend on network topology.

不当に長い期限を有するHTLCは、DoSベクトルであるため許されない。
「不当に」の正確な値は現在不明であり、ネットワークトポロジに依存する可能性があることに注意。

### `cltv_expiry_delta` Selection

Once an HTLC has timed out, it can either be fulfilled or timed-out;
care must be taken around this transition, both for offered and received HTLCs.

一旦HTLCがタイムアウトすると、それは実行される（be fulfilled）かタイムアウトになるかのどちらかである。
offered HTLCとreceived HTLCの両方について、この遷移の前後で注意を払う必要がある。

Consider the following scenario, where A sends an HTLC to B, who
forwards to C, who delivers the goods as soon as the payment is
received.

1. C needs to be sure that the HTLC from B cannot time out, even if B becomes
   unresponsive; i.e. C can fulfill the incoming HTLC on-chain before B can
   time it out on-chain.

2. B needs to be sure that if C fulfills the HTLC from B, it can fulfill the
   incoming HTLC from A; i.e. B can get the preimage from C and fulfill the incoming
   HTLC on-chain before A can time it out on-chain.

次のシナリオを考えなさい。
ここで、AはBにHTLCを送信し、（Bは）Cに転送し、（Cは）支払いを受け取るとすぐに商品を配送する。

1. Cは、Bが応答しなくなっても、BからのHTLCがタイムアウトしてないことを確認する必要がある：
すなわち、Bがそれをオンチェーンでタイムアウトする前に、
Cは受信HTLCをオンチェーンで実行する（fulfill）ことができる。

2. Bは、CがBからのHTLCを実行する（fulfill）場合、
それがAからの受信HTLCを実行する（fulfill）ことができることを保証する必要がある：
すなわち、Bは、Cからpreimageを取得し、
Aがそれをオンチェーンでタイムアウトする前に、
受信HTLCをオンチェーン上で実行する（fulfill）ことができる。

The critical settings here are the `cltv_expiry_delta` in
[BOLT #7](07-routing-gossip.md#the-channel_update-message) and the
related `min_final_cltv_expiry` in [BOLT #11](11-payment-encoding.md#tagged-fields).
`cltv_expiry_delta` is the minimum difference in HTLC CLTV timeouts, in
the forwarding case (B). `min_final_cltv_expiry` is the minimum difference
between HTLC CLTV timeout and the current block height, for the
terminal case (C).

ここでの重要な設定は、BOLT＃7のcltv_expiry_deltaとBOLT＃11の関連のmin_final_cltv_expiryである。
cltv_expiry_deltaは、転送ケース（B）における、HTLC CLTVタイムアウトの最小の差である。
min_final_cltv_expiryは、端末ケース（C）における、HTLC CLTVタイムアウトと現在のブロック高の最小の差である。

Note that if this value is too low for a channel, the risk is only to
the node *accepting* the HTLC, not the node offering it. For this
reason, the `cltv_expiry_delta` for the *outgoing* channel is used as
the delta across a node.

この値がチャネルにとって低すぎる場合、リスクは、提供するノードではなく、HTLCを受け入れるノードのみに注意すること。
この理由のため、送信チャネルのためのcltv_expiry_deltaは、ノードを横切るデルタとして使用される。
（XXX: 受信と送信の間を考えなくてはならないのでノードで考えなければならない。受信側と送信側の差分として）

The worst-case number of blocks between outgoing and
incoming HTLC resolution can be derived, given a few assumptions:

* a worst-case reorganization depth `R` blocks
* a grace-period `G` blocks after HTLC timeout before giving up on
  an unresponsive peer and dropping to chain
* a number of blocks `S` between transaction broadcast and the
  transaction being included in a block

いくつか仮定すると、
送信と受信間のHTLCの回答である、
最悪の場合のブロック数を導き出すことができる。

* 最悪の場合の再編成（reorg）深度、Rブロック
* 猶予期間のGブロックは、HTLCタイムアウト後、応答しないピアをあきらめてチェーンに落ちる前（まで）
* Sブロックの数は、transactionのブロードキャストとトランザクションがブロックに含まれるまでの間

The worst case is for a forwarding node (B) that takes the longest
possible time to spot the outgoing HTLC fulfillment and also takes
the longest possible time to redeem it on-chain:

1. The B->C HTLC times out at block `N`, and B waits `G` blocks until
   it gives up waiting for C. B or C commits to the blockchain,
   and B spends HTLC, which takes `S` blocks to be included.
2. Bad case: C wins the race (just) and fulfills the HTLC, B only sees
   that transaction when it sees block `N+G+S+1`.
3. Worst case: There's reorganization `R` deep in which C wins and
   fulfills. B only sees transaction at `N+G+S+R`.
4. B now needs to fulfill the incoming A->B HTLC, but A is unresponsive: B waits `G` more
   blocks before giving up waiting for A. A or B commits to the blockchain.
5. Bad case: B sees A's commitment transaction in block `N+G+S+R+G+1` and has
   to spend the HTLC output, which takes `S` blocks to be mined.
6. Worst case: there's another reorganization `R` deep which A uses to
   spend the commitment transaction, so B sees A's commitment
   transaction in block `N+G+S+R+G+R` and has to spend the HTLC output, which
   takes `S` blocks to be mined.
7. B's HTLC spend needs to be at least `R` deep before it times out,
   otherwise another reorganization could allow A to timeout the
   transaction.

転送ノード（B）の最悪のケースは、
出力HTLCの実行（fulfillment）を見つけるために可能な限り長い時間を要し、
またオンチェーンで償還するために可能な限り長い時間を要する：

1. B->CのHTLCはブロックNでタイムアウトし、BはCを待つのをあきらめるまでGブロック待つ。
BまたはCはブロックチェーンにコミットし、
BはHTLCを使用するが、それは含まれるまでSブロック要する。
2. 悪い例：Cがレースに勝って
HTLCを実行（XXX: オンチェーンで）（fulfill）し、
BがブロックN+G+S+1を見るとき、そのtransactionを見るだけである。
（XXX: preimageを抽出する）
3. 最悪の場合：Rの深さのreorgがあり、そこでCが勝ち実行（fulfill）する。
BはN+G+S+Rのtransactionを見ているだけである。

4. Bは今、受信A->BのHTLCを実行する（fulfill）必要があるが、Aは応答しない：
BはAを待つのをやめるまで、さらにGブロックを待つ。
AまたはBがブロックチェーンにコミットする。
5. 悪いケース：BはブロックN+G+S+R+G+1にある
Aのcommitment transaction見、
マイニングされるためのSブロックを要する、HTLC出力を使わなければならない。
6. 最悪の場合：Aがcommitment transactionを使用するために使う、別のreorgの深さRがあるため、
BはブロックN+G+S+R+G+R (XXX: ？) でAのcommitment transactionを見、
マイニングされるためのSブロックを要する、HTLC出力を使わなければならない。
7. BのHTLCの使用は、タイムアウトする前に少なくともR深くする必要がある。
そうしないと、別のreorgがAのトランザクションをタイムアウトさせる可能性がある。

（XXX: TODO: わからん）

（XXX:<br>
XXX: TODO: B->Cでタイムアウトするケースをまず考えてみる。
この場合、A->Bについてはもう考えなくてよくなる<br>
N:<br>
XXX: B->C<br>
G:猶予<br>
S:commit txがブロックに含まれるまで<br>
R:reorg<br>
S:Bがタイムアウトで回収する（勝つ）<br>
R:reorg<br>
XXX: 最後のreorgは絶対勝たなくてはいけないので必要<br>
）

（XXX:<br>
XXX: TODO: こうではないの？<br>
N:<br>
XXX: B->C<br>
G:猶予<br>
S:commit txがブロックに含まれるまで<br>
R:reorg<br>
S:Cがpreimageで回収する（負けるがpreimageを得る）<br>
XXX: preimageが回収できればreorgはどうでもいい、ということか？<br>
XXX: A->B<br>
G:猶予<br>
S:commit txがブロックに含まれるまで<br>
R:reorg<br>
S:Bがpreimageで回収する（勝つ）<br>
R:reorg<br>
）

Thus, the worst case is `3R+2G+2S`, assuming `R` is at least 1. Note that the
chances of three reorganizations in which the other node wins all of them is
low for `R` of 2 or more. Since high fees are used (and HTLC spends can use
almost arbitrary fees), `S` should be small; although, given that block times are
irregular and empty blocks still occur, `S=2` should be considered a
minimum. Similarly, the grace period `G` can be low (1 or 2), as nodes are
required to timeout or fulfill as soon as possible; but if `G` is too low it increases the
risk of unnecessary channel closure due to networking delays.

従って、最悪の場合は3R+2G+2S、Rは少なくとも1であると仮定する。
（XXX: なぜ1？）
すべての3回のreorgの機会で、
他方のノードがそれらの全てで勝つのは、2以上のRでは低いことに留意すること。
高いfeesが使用されるので（かつHTLC使用はほとんど任意のfeesを使うことができる）、
Sは小さくすべきである；（XXX: feeを適切に支払ってブロックに入りやすくする）
とはいえ、ブロック時間が不規則で空きブロックがまだ発生している場合の、S=2は最小限とみなすべきである。
同様に、猶予期間Gは、ノードができるだけ早くタイムアウトまたは実行する（fulfill）必要があるため、
低く（1または2）することができる；
しかし、Gが低すぎる場合には、それはネットワーク遅延による不要なチャネル閉鎖のリスクを増大させる。

There are four values that need be derived:

1. the `cltv_expiry_delta` for channels, `3R+2G+2S`: if in doubt, a
   `cltv_expiry_delta` of 12 is reasonable (R=2, G=1, S=2).

2. the deadline for offered HTLCs: the deadline after which the channel has to be failed
   and timed out on-chain. This is `G` blocks after the HTLC's
   `cltv_expiry`: 1 block is reasonable.

3. the deadline for received HTLCs this node has fulfilled: the deadline after which
the channel has to be failed and the HTLC fulfilled on-chain before its
   `cltv_expiry`. See steps 4-7 above, which imply a deadline of `2R+G+S`
   blocks before `cltv_expiry`: 7 blocks is reasonable.

4. the minimum `cltv_expiry` accepted for terminal payments: the
   worst case for the terminal node C is `2R+G+S` blocks (as, again, steps
   1-3 above don't apply). The default in
   [BOLT #11](11-payment-encoding.md) is 9, which is slightly more
   conservative than the 7 that this calculation suggests.

導出する必要がある4つの値がある：

1. チャネルのためのcltv_expiry_delta、3R+2G+2S：
不確定な場合、12のcltv_expiry_deltaが妥当（R = 2、G = 1、S = 2）である。

2. offered HTLCs（XXX: B->C）のデッドライン：
チャネルが失敗し、オンチェーンでタイムアウトしなければならないデッドライン。
（XXX: レースに負けるかもしないが）
これはHTLCのcltv_expiryの後のGブロックである：
1ブロックが妥当である。

3. このノードが実行した（fulfill）received HTLC（XXX: A->B）のデッドライン：
チャネルが失敗し、そのcltv_expiryの前、オンチェーンで実行される（fulfilled）デッドライン 。
上記のステップ4-7を参照し、
これは、cltv_expiryの前の、2R+G+Sブロックの締め切りを暗示する：
7ブロックが妥当である。

4. 終端の支払いのために受け入れられる最小のcltv_expiry：
終端ノードCの最悪のケースは2R+G+Sブロックである（再度、上記のステップ1-3が当てはまらない）。
BOLT＃11のデフォルト値は9である。
これは、この計算で示唆されている7よりわずかに控えめである。

#### Requirements

An offering node:
  - MUST estimate a timeout deadline for each HTLC it offers.
  - MUST NOT offer an HTLC with a timeout deadline before its `cltv_expiry`.
  - if an HTLC which it offered is in either node's current
  commitment transaction, AND is past this timeout deadline:
    - MUST fail the channel.

提供（offering）ノード：
  - それが提供する各HTLCのタイムアウトの期限を推定しなければならない。
  - そのcltv_expiry以前のタイムアウトの期限のHTLCを提供してはならない。
  - 提供したHTLCが、現在のcommitment transactionにあり、かつタイムアウトの期限を過ぎている：
    - チャネルに失敗しなければならない。
    （XXX: これ！一度閉じないといけない）

A fulfilling node:
  - for each HTLC it is attempting to fulfill:
    - MUST estimate a fulfillment deadline.
  - MUST fail (and not forward) an HTLC whose fulfillment deadline is already past.
  - if an HTLC it has fulfilled is in either node's current commitment
  transaction, AND is past this fulfillment deadline:
    - MUST fail the connection.

実行する（fulfilling）ノード：
（XXX: TODO: 重要そうなところだがこれでいいのか？）
  - 実行しよう（fulfill）としているHTLCごとに：
    - 実行（fulfillment）の期限を推定しなければならない。
  - 実行（fulfillment）期限がすでに過ぎているHTLCを失敗しなければならない（かつ転送しない）。
  （XXX: outgoing HTLCをコミットすると、
  自力ではincoming HTLCに対してupdate_fail_htlcしてはいけない。
  outgoing HTLCをコミットする前の話であろう）
  - 実行した(has fulfilled)HTLCが、現在のcommitment transactionにあり、
  （XXX: ここのfulfillした状態というのがどのような状態かよくわからない。
  commitment transactionにあるということは、まだコミットされていない？）
  かつ実行（fulfillment）のタイムアウトのデッドラインを過ぎている：
    - 接続に失敗しなければならない。
    （XXX: TODO: incoming/outgoingどっち？
    どっちがタイムアウトしている？
    どちらかfulfillとタイムアウトのレース状態になっていないか？
    再接続ではなくチャンネル失敗しなくて良いのか？）

### Adding an HTLC: `update_add_htlc`

Either node can send `update_add_htlc` to offer an HTLC to the other,
which is redeemable in return for a payment preimage. Amounts are in
millisatoshi, though on-chain enforcement is only possible for whole
satoshi amounts greater than the dust limit (in commitment transactions these are rounded down as
specified in [BOLT #3](03-transactions.md)).

いずれのノードも、HTLCを他のノードに提供するためにupdate_add_htlcを送信することができ、
これはpayment preimageを見返りに償還できる。
金額はmillisatoshiであるが、オンチェーンでの実施は、dust limitよりも多い量のsatoshiの量に対してのみ可能である
（commitment transactionsでは、これらはBOLT ＃3で指定されているように切り捨てられる）。

The format of the `onion_routing_packet` portion, which indicates where the payment
is destined, is described in [BOLT #4](04-onion-routing.md).

支払いの宛先を示すonion_routing_packetの部分の形式は、BOLT＃4に記載されている。

1. type: 128 (`update_add_htlc`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`id`]
   * [`8`:`amount_msat`]
   * [`32`:`payment_hash`]
   * [`4`:`cltv_expiry`]
   * [`1366`:`onion_routing_packet`]

#### Requirements

A sending node:
  - MUST NOT offer `amount_msat` it cannot pay for in the
remote commitment transaction at the current `feerate_per_kw` (see "Updating
Fees") while maintaining its channel reserve.
  - MUST offer `amount_msat` greater than 0.
  - MUST NOT offer `amount_msat` below the receiving node's `htlc_minimum_msat`
  - MUST set `cltv_expiry` less than 500000000.
  - for channels with `chain_hash` identifying the Bitcoin blockchain:
    - MUST set the four most significant bytes of `amount_msat` to 0.
  - if result would be offering more than the remote's
  `max_accepted_htlcs` HTLCs, in the remote commitment transaction:
    - MUST NOT add an HTLC.
  - if the sum of total offered HTLCs would exceed the remote's
`max_htlc_value_in_flight_msat`:
    - MUST NOT add an HTLC.
  - for the first HTLC it offers:
    - MUST set `id` to 0.
  - MUST increase the value of `id` by 1 for each successive offer.

送信ノード：
  - そのchannel reserveを維持したまま（XXX: channel reserveを下回るような送金をしてはいけない）、
  現在のfeerate_per_kw（「Updating Fees」参照）でremote commitment transactionに、
  そのノードが支払うことはできないamount_msatを提供してはならない。
  （XXX: remote commitment transactionのことだけを考えれば良い）
  - 0より大きいamount_msatを、提供しなければならない。
  - 受信ノードのhtlc_minimum_msat未満のamount_msatを提供してはならない
  - cltv_expiryは、500000000未満に設定しなければならない。
  - chain_hashが、Bitcoinブロックチェーンであると同定されるチャネルの場合：
    - amount_msatの4つの最上位バイトを0に設定しなければならない。
  - 結果が、remote commitment transactionにおいて、
  リモートのmax_accepted_htlcs HTLCs以上を提供する場合：
    - HTLCを追加してはならない。
  - 提供されたHTLCの合計が、リモートのmax_htlc_value_in_flight_msatを超えた場合：
    - HTLCを追加してはならない。
  - そのノードが提供する、最初のHTLCのために：
    - idを0に設定しなければならない。
  - idを連続するオファー毎に1ずつ値を増やさなければならない。
  （XXX: 再接続した場合、キャンセルされたupdateに関して、
  idは元に戻すのか戻さないのか？気にしないのか？
  元に戻すと以下のようにidが重複する。
  元に戻さない場合はidが飛ぶ）

A receiving node:
  - receiving an `amount_msat` equal to 0, OR less than its own `htlc_minimum_msat`:
    - SHOULD fail the channel.
  - receiving an `amount_msat` that the sending node cannot afford at the current `feerate_per_kw` (while maintaining its channel reserve):
    - SHOULD fail the channel.
  - if a sending node adds more than its `max_accepted_htlcs` HTLCs to
    its local commitment transaction, OR adds more than its `max_htlc_value_in_flight_msat` worth of offered HTLCs to its local commitment transaction:
    - SHOULD fail the channel.
  - if sending node sets `cltv_expiry` to greater or equal to 500000000:
    - SHOULD fail the channel.
  - for channels with `chain_hash` identifying the Bitcoin blockchain, if the four most significant bytes of `amount_msat` are not 0:
    - MUST fail the channel.
  - MUST allow multiple HTLCs with the same `payment_hash`.
  - if the sender did not previously acknowledge the commitment of that HTLC:
    - MUST ignore a repeated `id` value after a reconnection.
  - if other `id` violations occur:
    - MAY fail the channel.

受信ノード：
  - amount_msatが0に等しいか、自分のhtlc_minimum_msat未満を受け取る：
    - チャネルに失敗すべきである。
  - 送信ノードが現在のfeerate_per_kwで余裕がないamount_msatを受け取る（そのchannel reserveを維持したまま）。
    - チャネルに失敗すべきである。
  - 送信ノードが、
  それのmax_accepted_htlcsより大きなHTLCsを、それのローカルのcommitment transactionに追加するか、
  それのmax_htlc_value_in_flight_msatより大きなoffered HTLCsを、それのローカルcommitment transactionに追加する。
    - チャネルに失敗すべきである。
  - 送信ノードがcltv_expiryを500000000以上に設定する場合：
    - チャネルに失敗すべきである。
  - chain_hashがBitcoinブロックチェーンだと同定されるチャネルにおいて、amount_msatの4つの最上位バイトが0でない場合：
    - チャネルに失敗しなければならない。
  - 同じpayment_hashの複数のHTLCsを許さなければならない。
  - 送信者が以前にそのHTLCのコミットメントを認めなかった場合：
  （XXX: 送信者のchannel_reestablishのnext_local_commitment_numberでコミットされているHTLCを知る）
    - 再接続後には、繰り返しのidの値を無視しなければならない。
  - 他のid違反が発生した場合：
    - チャンネルに失敗してよい。

The `onion_routing_packet` contains an obfuscated list of hops and instructions for each hop along the path.
It commits to the HTLC by setting the `payment_hash` as associated data, i.e. includes the `payment_hash` in the computation of HMACs.
This prevents replay attacks that would reuse a previous `onion_routing_packet` with a different `payment_hash`.

onion_routing_packetには、パスに沿った各hopのためのhopと命令の難読化されたリストが含まれている。
それは関連するデータとしてpayment_hashを設定することでそのHTLCにコミットする。
つまりHMACの計算にpayment_hashを含める。
これにより、以前のonion_routing_packetと別のpayment_hashを再利用するリプレイ攻撃が防止される。
（XXX: onion_routing_packetがpayment_hashでのみHTLCにコミットしている場合、
Base AMPにおいて、同一payment_hashが複数経路で使用されている場合、
いずれの経路にも属する中継ノードにおいてonion_routing_packetを別の経路のものに置き換えて
経路を変更することができる。中継ノードは自分に都合のよりルートに変更するであろう。）

#### Rationale

Invalid amounts are a clear protocol violation and indicate a breakdown.

無効な金額は明確なプロトコル違反であり、故障を示している。

If a node did not accept multiple HTLCs with the same payment hash, an
attacker could probe to see if a node had an existing HTLC. This
requirement, to deal with duplicates, leads to the use of a separate
identifier; its assumed a 64-bit counter never wraps.

ノードが同じpayment hashを持つ複数のHTLCを受け入れなかった場合、
攻撃者は、ノードに既存のHTLCがあるかどうかを調べることができる。
重複を処理するためのこの要件は、別個の識別子の使用につながる；
それは決してラップしない64ビットカウンタである。

（XXX: Base AMPのように
全ての同一のpayment_hashのHTLCがコミットされてから
fulfillされるのが保証されているのであれば問題ないが、
そうでなければ（payment_hashの二重使用）すでにpreimageを知っている中継ノードが、
資金を中継せずに上流のHTLCをfulfillできてしまう）

Retransmissions of unacknowledged updates are explicitly allowed for
reconnection purposes; allowing them at other times simplifies the
recipient code (though strict checking may help debugging).

未確認の更新の再送は、再接続の目的で明示的に許可される；
他の時間にそれらを許可すると、受信者のコードが簡素化される（厳密なチェックはデバッグに役立つが）。
（XXX: TODO: 他の時間にidが同じ場合は再送とみなすべき。
異なるHTLCが同じidで送られるのは許容できないだろう。これは明文化されるべき）

`max_accepted_htlcs` is limited to 483 to ensure that, even if both
sides send the maximum number of HTLCs, the `commitment_signed` message will
still be under the maximum message size. It also ensures that
a single penalty transaction can spend the entire commitment transaction,
as calculated in [BOLT #5](05-onchain.md#penalty-transaction-weight-calculation).

max_accepted_htlcsは、
両方の側が最大数のHTLCを送信しても、
commitment_signedメッセージが最大メッセージサイズ以下になるように、
483に制限されている。
また、BOLT＃5で計算されるように、
単一のpenalty transactionがcommitment transaction全体を費やすことができる。

`cltv_expiry` values equal to or greater than 500000000 would indicate a time in
seconds, and the protocol only supports an expiry in blocks.

500000000以上のcltv_expiryの値は秒単位の時間を示し、プロトコルはブロック単位の有効期限のみをサポートする。

`amount_msat` is deliberately limited for this version of the
specification; larger amounts are not necessary, nor wise, during the
bootstrap phase of the network.

amount_msatは、この仕様のこのバージョンでは意図的に制限されている；
ネットワークのブートストラップフェーズでは、より多くの量を必要とせず、賢明でもない。

### Removing an HTLC: `update_fulfill_htlc`, `update_fail_htlc`, and `update_fail_malformed_htlc`

For simplicity, a node can only remove HTLCs added by the other node.
There are four reasons for removing an HTLC: the payment preimage is supplied,
it has timed out, it has failed to route, or it is malformed.

単純化のために、ノードは、他のノードによって追加されたHTLCのみを削除することができる。
HTLCを削除するには4つの理由がある：
payment preimageが供給されているか、
タイムアウトしているか、
ルーティングに失敗しているか、
または不正な形式である。

To supply the preimage:

preimageを供給するには：

1. type: 130 (`update_fulfill_htlc`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`id`]
   * [`32`:`payment_preimage`]

For a timed out or route-failed HTLC:

タイムアウトまたは経路障害のあるHTLCの場合：

1. type: 131 (`update_fail_htlc`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`id`]
   * [`2`:`len`]
   * [`len`:`reason`]

The `reason` field is an opaque encrypted blob for the benefit of the
original HTLC initiator, as defined in [BOLT #4](04-onion-routing.md);
however, there's a special malformed failure variant for the case where
the peer couldn't parse it: in this case the current node instead takes action, encrypting
it into a `update_fail_htlc` for relaying.

reasonフィールドは、BOLT＃4で定義されている、オリジナルのHTLC開始者のためのあいまいな暗号化されたBLOBである。
しかし、ピアがそれ（XXX: onion_routing_packet？）を解析できなかった場合のための特別な不正な形式の失敗のための変形がある：
この場合、現在のノードは代わりにアクションを行い、それをupdate_fail_htlcの中へ中継用に暗号化する。
（XXX: あるノード（上記、ピア）が、
update_add_htlcのonion_routing_packetを解析できなかったら、
上流へupdate_fail_malformed_htlcを送り、
受け取ったピア（上記、現在のノード）がupdate_fail_htlcを上流に送るのであろう）

For an unparsable HTLC:

解析不能なHTLCの場合：

1. type: 135 (`update_fail_malformed_htlc`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`id`]
   * [`32`:`sha256_of_onion`]
   * [`2`:`failure_code`]

#### Requirements

A node:
  - SHOULD remove an HTLC as soon as it can.
  - SHOULD fail an HTLC which has timed out.
  - until the corresponding HTLC is irrevocably committed in both sides'
  commitment transactions:
    - MUST NOT send an `update_fulfill_htlc`, `update_fail_htlc`, or
`update_fail_malformed_htlc`.

ノード：
  - できるだけ早くHTLCを削除すべきである。
  - タイムアウトしたHTLCは失敗するべきである。
  （XXX: その前に下流のHTLCがタイムアウトした場合、
  unirateral closeしてHTLCの削除がirrevocably committedされないといけない）
  - 対応するHTLCが両側のcommitment transactionsにおいてirrevocably committedされるまで：
    - update_fulfill_htlc、update_fail_htlcまたはupdate_fail_malformed_htlcを送ってはならない。
    （XXX: 下流でirrevocably committedされる前に送られてくるのは不正であるが、
    もし受け取ったら、irrevocably committedされるまで待つ。
    場合によってはオンチェーンでirrevocably committedされる。
    update_fulfill_htlcの場合、そこまで待つインセンティブはないが、
    そのほうが単純であろう）
    （XXX: TODO: update_fail_htlc/update_fail_malformed_htlcについては、
    下流でHTLCの削除までirrevocably committedされる必要がある。
    これは明文化されるべきであろう）

A receiving node:
  - if the `id` does not correspond to an HTLC in its current commitment transaction:
    - MUST fail the channel.
  - if the `payment_preimage` value in `update_fulfill_htlc`
  doesn't SHA256 hash to the corresponding HTLC `payment_hash`:
    - MUST fail the channel.
  - if the `BADONION` bit in `failure_code` is not set for
  `update_fail_malformed_htlc`:
    - MUST fail the channel.
  - if the `sha256_of_onion` in `update_fail_malformed_htlc` doesn't match the
  onion it sent:
    - MAY retry or choose an alternate error response.
  - otherwise, a receiving node which has an outgoing HTLC canceled by `update_fail_malformed_htlc`:
    - MUST return an error in the `update_fail_htlc` sent to the link which
      originally sent the HTLC, using the `failure_code` given and setting the
      data to `sha256_of_onion`.

受信ノード：
  - idが、それの現在のcommitment transactionのHTLCに対応していない場合：
    - チャネルを失敗しなければならない。    
  - update_fulfill_htlcのpayment_preimageの値が、
  対応するHTLCのpayment_hashのSHA256ハッシュでない場合:
    - チャネルを失敗しなければならない。
  - failure_codeのBADONIONビットがupdate_fail_malformed_htlcのために設定されていない：
    - チャネルを失敗しなければならない。
  - update_fail_malformed_htlcのsha256_of_onionがそれを送っているonionと一致していない：
    - 再試行するか代替のエラー応答を選択しても良い。
  - そうでなければ、update_fail_malformed_htlcでキャンセルされた送信HTLCを有する受信ノード：
    - 与えられたfailure_codeを使用して、データにsha256_of_onionを設定して、（XXX: reason部分であろう）
    HTLCを最初に送信したリンクに送信するupdate_fail_htlcでエラーを返さなければならない。

#### Rationale

A node that doesn't time out HTLCs risks channel failure (see
[`cltv_expiry_delta` Selection](#cltv_expiry_delta-selection)).

HTLCをタイムアウトさせないノードは、チャネル障害を引き起こすリスクがある
（cltv_expiry_deltaセレクション参照）。

A node that sends `update_fulfill_htlc`, before the sender, is also
committed to the HTLC and risks losing funds.

update_fulfill_htlcを送信するノード、
送信者の前に、
も、HTLCにコミットし、資金を失うリスクがある。
（XXX: ？）

If the onion is malformed, the upstream node won't be able to extract
the shared key to generate a response — hence the special failure message, which
makes this node do it.

onionが不正な形式の場合、
上流ノード（XXX: update_fail_malformed_htlc）は共有鍵を抽出して応答を生成することができない。
従って、このノードはそれを行う特別なエラーメッセージである。
（XXX: reasonはピアが作らなくてはならない）

The node can check that the SHA256 that the upstream is complaining about
does match the onion it sent, which may allow it to detect random bit
errors. However, without re-checking the actual encrypted packet sent,
it won't know whether the error was its own or the remote's; so
such detection is left as an option.

ノードは、上流側が苦情を申し立てているSHA256が、送信したonionと一致していることを確認できる。
これにより、ランダムビットエラーを検出できる。
しかし、送信された実際の暗号化されたパケットを再確認することなく、
エラーがそれ自身のものかリモートのものかを知ることはできない；
従ってそのような検出はオプションとして残される。

### Committing Updates So Far: `commitment_signed`

When a node has changes for the remote commitment, it can apply them,
sign the resulting transaction (as defined in [BOLT #3](03-transactions.md)), and send a
`commitment_signed` message.

ノードにリモートのコミットメントの変更があった場合、それを適用し、
結果のtransactionに署名し（BOLT＃3で定義されているように）、
commitment_signedメッセージを送信できる。

1. type: 132 (`commitment_signed`)
2. data:
   * [`32`:`channel_id`]
   * [`64`:`signature`]
   * [`2`:`num_htlcs`]
   * [`num_htlcs*64`:`htlc_signature`]

（XXX: signatureはマルチシグの署名（signature_for_pubkey1またはsignature_for_pubkey2））
（XXX: htlc_signatureはremotehtlcsigであろう）

#### Requirements

A sending node:
  - MUST NOT send a `commitment_signed` message that does not include any
updates.
  - MAY send a `commitment_signed` message that only
alters the fee.
  - MAY send a `commitment_signed` message that doesn't
change the commitment transaction aside from the new revocation hash
(due to dust, identical HTLC replacement, or insignificant or multiple
fee changes).
  - MUST include one `htlc_signature` for every HTLC transaction corresponding
  to BIP69 lexicographic ordering of the commitment transaction.
  - if it has not recently received a message from the remote node:
      - SHOULD use `ping` and await the reply `pong` before sending `commitment_signed`.

送信ノード：
  - 更新を含まないcommitment_signedメッセージを送信してはならない。
  - 手数料を変更するだけのcommitment_signedメッセージを送信できる。
  - 新たなrevocation hash以外はcommitment transactionを変更しない、
  commitment_signedメッセージを送信できる。
  （以下による、dustで同一のHTLCの置き換え、またはわずかか複数回の料金変更）。
  （XXX: これはHTLCが追加されたがそれはdustで他のHTLCは変わらないが、
  dustになった分がto_remote or to_localからfeeに落ちたのでcommitment txの署名は変わる状態）
  （XXX: TODO: これはその上の行の手数料の変更のみに含まれないのか？）
  - commitment transactionのBIP69辞書順に対応する、
  全てのHTLC transactionのためのhtlc_signatureを1つを含めなければならない。
  （XXX: BIP69では出力をamountとscriptPubkeyでソートするが、BOLTでは同じ値になるHTLCができる。
  具体的にはamountとpayment_hashが同じときである。
  これらが同じの場合でもHTLC txがHTLC timeout txの場合、
  cltv_expiryが異なるためhtlc_signatureが異なり順番が一意でなくなる）
  - 最近リモートノードからメッセージを受信していない場合：
    - committed_signedを送る前に、pingを使用しpong応答を待つべきである。

A receiving node:
  - once all pending updates are applied:
    - if `signature` is not valid for its local commitment transaction:
      - MUST fail the channel.
    - if `num_htlcs` is not equal to the number of HTLC outputs in the local
    commitment transaction:
      - MUST fail the channel.
  - if any `htlc_signature` is not valid for the corresponding HTLC transaction:
    - MUST fail the channel.
  - MUST respond with a `revoke_and_ack` message.

受信ノード：
  - すべての保留中の更新が適用されると：
    - signatureが、そのlocal commitment transactionに対して有効ではない場合：
      - チャネルに失敗しなければならない。
    - num_htlcsが、local commitment transactionのHTLC出力の数と同じではない場合：
      - チャネルを失敗しなければならない。
  - あるhtlc_signatureが、対応するHTLC取引に対して有効ではない：
    - チャネルを失敗しなければならない。
  - revoke_and_ackメッセージで応答しなければならない。
  （XXX: これ！）

#### Rationale

There's little point offering spam updates: it implies a bug.

スパムのアップデートを提供するという点はほとんどない：それはバグを暗示する。

The `num_htlcs` field is redundant, but makes the packet length check fully self-contained.

num_htlcsフィールドは冗長だが、パケット長チェックは完全に自己完結している。

The recommendation to require recent messages recognizes the reality
that networks are unreliable: nodes might not realize their peers are
offline until after sending `commitment_signed`.  Once
`commitment_signed` is sent, the sender considers itself bound to
those HTLCs, and cannot fail the related incoming HTLCs until the
output HTLCs are fully resolved.

最近のメッセージを要求する推奨、ネットワークが信頼できないという現実を認識している：
ノードは、commitment_signedを送信するまで、ピアがオフラインであることを認識しない可能性がある。
commit_signedが送信されると、送信側はHTLCにバインドされているとみなし、
出力HTLCが完全に解決されるまで、関連する入力HTLCを失敗することはできない。

### Completing the Transition to the Updated State: `revoke_and_ack`

Once the recipient of `commitment_signed` checks the signature and knows
it has a valid new commitment transaction, it replies with the commitment
preimage for the previous commitment transaction in a `revoke_and_ack`
message.

一旦commitment_signedの受信者が、署名をチェックし、
それが有効な新しいcommitment transactionを有することを知ると、
それはrevoke_and_ackメッセージ内の、
以前のcommitment transactionのcommitment preimage（XXX: per_commitment_secretであろう）で応答する。

This message also implicitly serves as an acknowledgment of receipt
of the `commitment_signed`, so this is a logical time for the `commitment_signed` sender
to apply (to its own commitment) any pending updates it sent before
that `commitment_signed`.

このメッセージ（XXX: revoke_and_ack）は暗黙のうちにcommitment_signedの受信確認を提供するので、
従ってこれは、
commitment_signed送信者が
それ以前に送信した保留中の更新を（自らのコミットメントに）適用する論理時間（XXX: ？）である。

The description of key derivation is in [BOLT #3](03-transactions.md#key-derivation).

キー導出の説明は、BOLT＃3にある。

1. type: 133 (`revoke_and_ack`)
2. data:
   * [`32`:`channel_id`]
   * [`32`:`per_commitment_secret`]
   * [`33`:`next_per_commitment_point`]

#### Requirements

A sending node:
  - MUST set `per_commitment_secret` to the secret used to generate keys for
  the previous commitment transaction.
  - MUST set `next_per_commitment_point` to the values for its next commitment
  transaction.

送信ノード：
  - per_commitment_secretは、
  以前のcommitment transactionの鍵を生成するために使用されたsecretに設定する必要がある。
  - next_per_commitment_pointは、
  次のcommitment transactionの値に設定する必要がある。

A receiving node:
  - if `per_commitment_secret` does not generate the previous `per_commitment_point`:
    - MUST fail the channel.
  - if the `per_commitment_secret` was not generated by the protocol in [BOLT #3](03-transactions.md#per-commitment-secret-requirements):
    - MAY fail the channel.

受信ノード：
  - per_commitment_secretが、以前のper_commitment_pointを生成しない場合：
    - チャネルを失敗しなければならない。
  - per_commitment_secretが、BOLT #3のプロトコルによって生成されていない場合：
    - チャンネルを失敗して良い。

A node:
  - MUST NOT broadcast old (revoked) commitment transactions,
    - Note: doing so will allow the other node to seize all channel funds.
  - SHOULD NOT sign commitment transactions, unless it's about to broadcast
  them (due to a failed connection),
    - Note: this is to reduce the above risk.

ノード：
  - 古い（取り消された）commitment transactionsをブロードキャストしてはならない、
    - 注：そうすることで、他のノードがすべてのチャネル資金を獲得できるようになる。
  - commitment transactionsに署名すべきではない、
  それらをブロードキャストしようとする場合を除き（失敗した接続のために）。
    - 注：これは、上記のリスクを減らすことである。

### Updating Fees: `update_fee`

An `update_fee` message is sent by the node which is paying the
Bitcoin fee. Like any update, it's first committed to the receiver's
commitment transaction and then (once acknowledged) committed to the
sender's. Unlike an HTLC, `update_fee` is never closed but simply
replaced.

update_feeメッセージは、ビットコインのfeeを払っているノードによって送信される。
あらゆるアップデートと同様、それは最初に受信者のcommitment transactionにコミットしてから、
それから（一旦確認された）送信者のcommitment transactionにコミットする。
HTLCと異なり、update_feeは決して閉じられず、単に置き換えられる。

There is a possibility of a race, as the recipient can add new HTLCs
before it receives the `update_fee`. Under this circumstance, the sender may
not be able to afford the fee on its own commitment transaction, once the `update_fee`
is finally acknowledged by the recipient. In this case, the fee will be less
than the fee rate, as described in [BOLT #3](03-transactions.md#fee-payment).

受信者はupdate_feeを受信する前に新しいHTLCsを追加できるので、競合の可能性がある。
このような状況下では、一旦update_feeが最終的に受信者に承認されると、
送信者はそれ自身のcommitment transactionで料金を支払う余裕がない可能性がある。
このケースでは、feeはBOLT #3に記載されているfee rateよりも低くなる。
（XXX: それでもチャンネル失敗にならないように余裕を持って更新すべきであろう）

The exact calculation used for deriving the fee from the fee rate is
given in [BOLT #3](03-transactions.md#fee-calculation).

fee rateからfeeを導出するために使用される正確な計算は、BOLT＃3に示されている。

1. type: 134 (`update_fee`)
2. data:
   * [`32`:`channel_id`]
   * [`4`:`feerate_per_kw`]

#### Requirements

The node _responsible_ for paying the Bitcoin fee:
  - SHOULD send `update_fee` to ensure the current fee rate is sufficient (by a
      significant margin) for timely processing of the commitment transaction.

Bitcoin feeの支払いを担当するノード：
  - commitment transactionを適時に処理するために、
  現在のfee rateが十分で（意味のあるマージンで）あることを保証するようにupdate_feeを送信するべきである。

The node _not responsible_ for paying the Bitcoin fee:
  - MUST NOT send `update_fee`.

Bitcoin feeを支払う責任を負わないノード：
  - update_feeを送信してはならない。

A receiving node:
  - if the `update_fee` is too low for timely processing, OR is unreasonably large:
    - SHOULD fail the channel.
  - if the sender is not responsible for paying the Bitcoin fee:
    - MUST fail the channel.
  - if the sender cannot afford the new fee rate on the receiving node's
  current commitment transaction:
    - SHOULD fail the channel,
      - but MAY delay this check until the `update_fee` is committed.

受信ノード：
  - update_feeが適時に処理するには低すぎる場合、または不当に大きい場合：
    - チャネルを失敗するべきである。
  - 送信者がBitcoin feeを支払う責任がない場合：
    - チャネルを失敗しなければならない。    
  - 送信ノードが、受信ノードの現在のcommitment transactionのおける新しいfee rateで余裕がない場合：
    - チャネルに失敗するべきである、
      - update_feeがコミットされるまで、このチェックを延期できる。
      （XXX: それまで送信ノードは再度update_feeを行い調節する？）

#### Rationale

Bitcoin fees are required for unilateral closes to be effective —
particularly since there is no general method for the broadcasting node to use
child-pays-for-parent to increase its effective fee.

Bitcoin feesは、unilateral closesで特に有効である、ブロードキャストをするノードが、
その有効なfeeを増加させるためのchild-pays-for-parent（XXX: CPFP）を使用する一般的な方法がないため。

Given the variance in fees, and the fact that the transaction may be
spent in the future, it's a good idea for the fee payer to keep a good
margin (say 5x the expected fee requirement); but, due to differing methods of
fee estimation, an exact value is not specified.

feesの変動と、そのtransactionが将来に費やされるであろうことを考えれば、
fee支払者が良いマージンを保つことは良い考えである（例えば予想されるfee要求の5倍）。
が、feeの推定方法が異なるため、正確な値は指定されていない。

Since the fees are currently one-sided (the party which requested the
channel creation always pays the fees for the commitment transaction),
it's simplest to only allow it to set fee levels; however, as the same
fee rate applies to HTLC transactions, the receiving node must also
care about the reasonableness of the fee.

feesは現在一方的であるため（チャネル作成を依頼した集団は、commitment transactionのfeesを常に支払う）、
feeレベル（XXX: ？）を設定させるのが最も簡単である；
しかしながら、HTLC transactionsに同じfee rateが適用されるので、
受信ノードはfeeの妥当性にも気を配らなければならない。

## Message Retransmission

Because communication transports are unreliable, and may need to be
re-established from time to time, the design of the transport has been
explicitly separated from the protocol.

通信輸送は信頼性が低く、時折再確立する必要があるため、
トランスポートの設計はプロトコルとは明確に分離されている。

Nonetheless, it's assumed our transport is ordered and reliable.
Reconnection introduces doubt as to what has been received, so there are
explicit acknowledgments at that point.

それにもかかわらず、我々の輸送手段は整然として信頼できるものであると考えられている。
再接続は、すでに何が受信されているのか、疑問が生じさせる。
従って、その時点で明示的な確認がある。

This is fairly straightforward in the case of channel establishment
and close, where messages have an explicit order, but during normal
operation, acknowledgments of updates are delayed until the
`commitment_signed` / `revoke_and_ack` exchange; so it cannot be assumed
that the updates have been received. This also means that the receiving
node only needs to store updates upon receipt of `commitment_signed`.

これは、メッセージの明確な順序があるchannel establishmentとcloseの場合はかなり簡単であるが、
normal operationでは、更新の確認応答はcommitment_signed/revoke_and_ackの交換まで遅れる；
従って、更新が受信されたと見なすことはできない。
これはまた、受信ノードがcommitment_signed受信時に更新を格納するだけでよいことを意味する。

Note that messages described in [BOLT #7](07-routing-gossip.md) are
independent of particular channels; their transmission requirements
are covered there, and besides being transmitted after `init` (as all
messages are), they are independent of requirements here.

BOLT＃7に記述されているメッセージは、特定のチャネルに依存しないことに注意すること；
それらの輸送の要求はそこにカバーされており、
initの後に送信される（すべてのメッセージがそうである）以外は、
ここでの要件とは独立している。

1. type: 136 (`channel_reestablish`)
2. data:
   * [`32`:`channel_id`]
   * [`8`:`next_local_commitment_number`]
   * [`8`:`next_remote_revocation_number`]
   * [`32`:`your_last_per_commitment_secret`] (option_data_loss_protect)
   * [`33`:`my_current_per_commitment_point`] (option_data_loss_protect)

`next_local_commitment_number`: A commitment number is a 48-bit
incrementing counter for each commitment transaction; counters
are independent for each peer in the channel and start at 0.
They're only explicitly relayed to the other node in the case of
re-establishment, otherwise they are implicit.

next_local_commitment_number：
commitment numberは、各commitment transactionの48ビット増分カウンタである。
カウンタはチャネル内の各ピアに対して独立しており、0から開始する。
re-establishmentの場合には、他のノードに明示的に中継されるだけであり、
そうでない場合は暗黙的である。
（XXX: ？）

### Requirements

A funding node:
  - upon disconnection:
    - if it has broadcast the funding transaction:
      - MUST remember the channel for reconnection.
    - otherwise:
      - SHOULD NOT remember the channel for reconnection.

fundingノード：
  - 切断時：
    - funding transactionをブロードキャストしている場合：
      - 再接続のためにチャネルを覚えていなければならない。
    - そうでなければ：
      - 再接続のためのチャンネルを覚えるべきではない。

A non-funding node:
  - upon disconnection:
    - if it has sent the `funding_signed` message:
      - MUST remember the channel for reconnection.
    - otherwise:
      - SHOULD NOT remember the channel for reconnection.

non-fundingノード：
  - 切断時：
    - funding_signedメッセージを送信している場合：
      - 再接続のためにチャネルを覚えていなければならない。
    - そうでなければ：
      - 再接続のためのチャンネルを覚えるべきではない。

A node:
  - MUST handle continuation of a previous channel on a new encrypted transport.
  - upon disconnection:
    - MUST reverse any uncommitted updates sent by the other side (i.e. all
    messages beginning with `update_` for which no `commitment_signed` has
    been received).
      - Note: a node MAY have already used the `payment_preimage` value from
    the `update_fulfill_htlc`, so the effects of `update_fulfill_htlc` are not
    completely reversed.
  - upon reconnection:
    - if a channel is in an error state:
      - SHOULD retransmit the error packet and ignore any other packets for
      that channel.
    - otherwise:
      - MUST transmit `channel_reestablish` for each channel.
      - MUST wait to receive the other node's `channel_reestablish`
        message before sending any other messages for that channel.

ノード：
  - 新しい暗号化された輸送路で、前のチャネルを継続して処理しなければならない。
  - 切断時：
    - 他方から送信された、コミットされていない更新を戻さなくてはならない
    （すなわち、そのためにcommitment_signedをまだ受信していない、
    update_で始まるすべてのメッセージを戻さなければならいない）。
    （XXX: 逆に送信側はcommitment_signed送信以降に、
    自分のcommitment txの保留updateを元に戻してはいけない）
      - 注：
      ノードはすでにupdate_fulfill_htlcからのpayment_preimage値を使用していた可能性があるので、
      update_fulfill_htlcの効果は完全には戻らない。
      （XXX: 少なくともすでにpreimageが明らかになっているので、見なかったことにはならない）      
  - 再接続時：
    - チャネルがエラー状態にある場合：
      - エラーパケットを再送し、そのチャネルの他のパケットを無視すべきである。
    - そうでなければ：
      - channel_reestablishを、各チャネルに対して送信しなければならない。
      - そのチャネルの他のメッセージを送信する前に、
      他のノードのchannel_reestablishメッセージを受信するのを待たなければならない。

The sending node:
  - MUST set `next_local_commitment_number` to the commitment number of the
  next `commitment_signed` it expects to receive.
  - MUST set `next_remote_revocation_number` to the commitment number of the
  next `revoke_and_ack` message it expects to receive.
  - if it supports `option_data_loss_protect`:
    - if `next_remote_revocation_number` equals 0:
      - MUST set `your_last_per_commitment_secret` to all zeroes
    - otherwise:
      - MUST set `your_last_per_commitment_secret` to the last `per_commitment_secret`
    it received

送信ノード：
（XXX: 送信するどのフィールドもメッセージ受信に関わる。commitment_signedとrevoke_and_ackの受信）
  - next_local_commitment_numberは、
  受け取る予定の次のcommitment_signedのcommitment numberに設定しなければならない。
  - next_remote_revocation_numberは、
  受け取る予定の次のrevoke_and_ackメッセージのcommitment numberに設定しなければならない。
  - それがoption_data_loss_protectをサポートしている場合：
    - next_remote_revocation_numberが0の場合：
      - your_last_per_commitment_secretを、すべて0に設定しなければならない
    - そうでなければ：
      - your_last_per_commitment_secretを、
      受信した最後のper_commitment_secretに設定しなければならない

A node:
  - if `next_local_commitment_number` is 1 in both the `channel_reestablish` it
  sent and received:
    - MUST retransmit `funding_locked`.
  - otherwise:
    - MUST NOT retransmit `funding_locked`.
  - upon reconnection:
    - MUST ignore any redundant `funding_locked` it receives.
  - if `next_local_commitment_number` is equal to the commitment number of
  the last `commitment_signed` message the receiving node has sent:
    - MUST reuse the same commitment number for its next `commitment_signed`.
  - otherwise:
    - if `next_local_commitment_number` is not 1 greater than the
  commitment number of the last `commitment_signed` message the receiving
  node has sent:
      - SHOULD fail the channel.
  - if `next_remote_revocation_number` is equal to the commitment number of
  the last `revoke_and_ack` the receiving node sent, AND the receiving node
  hasn't already received a `closing_signed`:
    - MUST re-send the `revoke_and_ack`.
  - otherwise:
    - if `next_remote_revocation_number` is not equal to 1 greater than the
    commitment number of the last `revoke_and_ack` the receiving node has sent:
      - SHOULD fail the channel.
    - if it has not sent `revoke_and_ack`, AND `next_remote_revocation_number`
    is equal to 0:
      - SHOULD fail the channel.

ノード：
（XXX: 送信するどのフィールドもメッセージ受信に関わる。commitment_signedとrevoke_and_ackの受信）
  - 送受信された両方のchannel_reestablishのnext_local_commitment_numberが1の場合
    - funding_lockedを再送しなければならない。
  - そうでなければ：
    - funding_lockedを再送してはならない。
  - 再接続時：
    - 冗長なfunding_lockedの受信を無視しなければならない。
  - 受信したnext_local_commitment_numberが、
  受信ノードが送信した最後のcommitment_signedメッセージのcommitment numberに等しい場合：
  （XXX: 送信したcommitment_signedが届いていない可能性がある）
    - そのcommitment numberを次のcommitment_signedに再利用しなければならない。
    （XXX: reuseという表現はあぶない。
    後述されるように、相手が嘘をついていれば、相手はcommitment_signedを受け取っていて、
    コミットされたcommitment txを持っているかもしれない。
    それを前提に動くのは複雑になる。
    commitment_signedは届いていると仮定しチャネル失敗するのが安全？）
  - そうでなければ：
    - next_local_commitment_numberが、
    受信ノードが送信した最後のcommitment_signedメッセージのcommitment numberよりも1だけ大きくない場合：
    （XXX: 1だけ大きいのが普通）
      - チャネルを失敗すべきである。
  - 受信したnext_remote_revocation_numberが、
  受信ノードが送信した最後のrevoke_and_ackのcommitment numberと等しく、
  （XXX: 送信したrevoke_and_ackが届いていない可能性がある）
  受信ノードはまだclosing_signedを受信していない場合
  （XXX: closing_signedを受信しているということはrevoke_and_ackは届いていると判断される。後述。）：
    - revoke_and_ackを再送しなければならない。
  - そうでなければ：
    - next_remote_revocation_numberが、
    受信ノードが送信した最後のrevoke_and_ackのcommitment numberより1だけ大きくない場合：
    （XXX: 1だけ大きいのが普通）
      - チャネルを失敗すべきである。
    - revoke_and_ackを送信しておらず、next_remote_revocation_numberが0の場合：
    （XXX: TODO: revoke_and_ackをまだ送信したことがないケース。これがなぜだめなのか？）
      - チャネルを失敗すべきである。

 A receiving node:
  - if it supports `option_data_loss_protect`, AND the `option_data_loss_protect`
  fields are present:
    - if `next_remote_revocation_number` is greater than expected above, AND
    `your_last_per_commitment_secret` is correct for that
    `next_remote_revocation_number` minus 1:
      - MUST NOT broadcast its commitment transaction.
      - SHOULD fail the channel.
      - SHOULD store `my_current_per_commitment_point` to retrieve funds
        should the sending node broadcast its commitment transaction on-chain.
    - otherwise (`your_last_per_commitment_secret` or `my_current_per_commitment_point`
    do not match the expected values):
      - SHOULD fail the channel.

受信ノード：
  - option_data_loss_protectがサポートしており、
  option_data_loss_protectフィールドが存在する場合：
    - next_remote_revocation_numberが上記の期待値より大きく
    your_last_per_commitment_secret（XXX: 相手に持ってる最後のrevoke_and_ackのnumber）が、
    そのnext_remote_revocation_numberマイナス1に対して、正しい場合：（XXX: 自分の側でデータがロスしている）
      - それのcommitment transactionをブロードキャストしてはならない
      - チャネルに失敗すべきである。
      - 送信ノードがオンチェーンにcommitment transactionをブロードキャストする場合に資金を回収するために、
      （XXX: 相手の）my_current_per_commitment_pointを保存すべきである。
      （XXX: データロスしてるので相手に任せるしかない）
    - そうでなければ（XXX: TODO: ここはotherwiseじゃなくてelse ifだろう。どちらも期待値である場合の正常系）
    （your_last_per_commitment_secretまたはmy_current_per_commitment_pointが期待値と一致しない場合）：
      - チャネルを失敗すべきである。

A node:
  - MUST NOT assume that previously-transmitted messages were lost,
    - if it has sent a previous `commitment_signed` message:
      - MUST handle the case where the corresponding commitment transaction is
      broadcast at any time by the other side,
        - Note: this is particularly important if the node does not simply
        retransmit the exact `update_` messages as previously sent.
  - upon reconnection:
    - if it has sent a previous `shutdown`:
      - MUST retransmit `shutdown`.

ノード：
  - 以前に送信されたメッセージが失われたと仮定してはならない、
  （XXX: 相手が嘘をついているかもしない）
    - 前のcommitment_signedメッセージを送信した場合：
      - 対応するcommitment transactionがいつでも相手側によってブロードキャストされるケースを処理しなければならず、
      （XXX: 相手のchannel_reestablishからcommitment_signedが届いていないことを推測されたとしても、嘘をついてるかもしれない）
        - 注：これは特に重要であるが、
        もしノードがupdate_以前に送信された正確なメッセージを単に再送するだけでない場合。
        （XXX: 単に再送してcommitment txが上記と同じ状態になるのであれば問題ない）
  - 再接続時：
    - 以前にshutdownを送信した場合：
      - shutdownを再送しなければならない。
      （XXX: closingは全てやりなおし）

### Rationale

The requirements above ensure that the opening phase is nearly
atomic: if it doesn't complete, it starts again. The only exception
is if the `funding_signed` message is sent but not received. In
this case, the funder will forget the channel, and presumably open
a new one upon reconnection; meanwhile, the other node will eventually forget
the original channel, due to never receiving `funding_locked` or seeing
the funding transaction on-chain.

上の要件は、開始フェーズがほぼアトミックであることを保証する：
それが完了しなければ、再び開始する。
唯一の例外は、funding_signedメッセージが送信されたが受信されなかった場合である。
この場合、funderはチャンネルを忘れて、おそらく再接続時に新しいチャンネルを開く。
一方で、片方のノードは、
（XXX: funding_signedを受信していないため）funding_lockedを受け取ったり、
オンチェーンのfunding transactionを見ることがないため、
最終的に元のチャネルを忘れてしまう。（XXX: 忘れていい）

There's no acknowledgment for `error`, so if a reconnect occurs it's
polite to retransmit before disconnecting again; however, it's not a MUST,
because there are also occasions where a node can simply forget the
channel altogether.

errorのための何の確認もないので、再接続が発生した場合、再接続する前に再送信することは丁寧である；
しかし、それは必須ではない、なぜなら、ノードが単にチャンネルを完全に忘れることがあるからである。

`closing_signed` also has no acknowledgment so must be retransmitted
upon reconnection (though negotiation restarts on reconnection, so it needs
not be an exact retransmission).
The only acknowledgment for `shutdown` is `closing_signed`, so one or the other
needs to be retransmitted.

closing_signedにも確認がないので、再接続時に再送信する必要がある。
（ただし、再接続時にネゴシエーションが再開されるため、厳密な再送信は必要ない）。
shutdownのための唯一の確認はclosing_signedであり、どちらか一方が再送される必要がある。
（XXX: TODO: 「MUST retransmit shutdown.」なのでは？
おそらく記述が古いまま。
以下を記述すべきであろう。
１、shutdownを再送しChannel Closeをやり直す。
２、shutdown以降のclosing_signedを同じ値で再送する必要はない。
）

The handling of updates is similarly atomic: if the commit is not
acknowledged (or wasn't sent) the updates are re-sent. However, it's not
insisted they be identical: they could be in a different order,
involve different fees, or even be missing HTLCs which are now too old
to be added. Requiring they be identical would effectively mean a
write to disk by the sender upon each transmission, whereas the scheme
here encourages a single persistent write to disk for each
`commitment_signed` sent or received.

更新の取り扱いも同様にアトミックである：
コミットが確認されなかった場合（または送信されなかった場合）、更新は再送信される。
しかし、それらは同じであるとは主張されない：
彼らは異なった順序であるかもしれない、異なったfeeを含んでいるか、
今追加するには余りに古いHTLCsを除外していることさえあるかもしれない。
それらを同一にすることを必要とすることは、
各送信時に送信者がディスクに書き込むことが効果的であることを意味するが、
ここではcommitment_signedの送受信毎にディスクへの単一の永続的書き込みを促進する。

A re-transmittal of `revoke_and_ack` should never be asked for after a
`closing_signed` has been received, since that would imply a shutdown has been
completed — which can only occur after the `revoke_and_ack` has been received
by the remote node.

revoke_and_ackの再送は、closing_signedが受信された後に決して求められてはならない、
なぜならこれは、シャットダウンが完了したことを暗示するためである、
リモートノードによってrevoke_and_ackが受信された後にのみ発生できる。
（XXX: ここではrevoke_and_ackを受信できていないとclosing_signedできないような記述。
おそらくclosing_signedの前提として全てのupdateがirrevocably committedなのであろう）

Note that the `next_local_commitment_number` starts at 1, since
commitment number 0 is created during opening.
`next_remote_revocation_number` will be 0 until the
`commitment_signed` for commitment number 1 is received, at which
point the revocation for commitment number 0 is sent.

開始時にcommitment number 0が作成されるため、
next_local_commitment_numberは1から始まることに注意すること。
next_remote_revocation_numberは、
commitment number 1のcommitment_signed受信まで0になり、
そのときcommitment number 0の取り消しが送信される。

`funding_locked` is implicitly acknowledged by the start of normal
operation, which is known to have begun after a `commitment_signed` has been
received — hence, the test for a `next_local_commitment_number` greater
than 1.

funding_lockedは、normal operationの開始によって暗黙的に確認される、
それは、commitment_signedが受信された後に開始されたことが知られている。
従って、next_local_commitment_numberが1より大きい値であるテストである。
（XXX: そのテストが通れば、funding_lockedはもう不要）

A previous draft insisted that the funder "MUST remember ...if it has
broadcast the funding transaction, otherwise it MUST NOT": this was in
fact an impossible requirement. A node must either firstly commit to
disk and secondly broadcast the transaction or vice versa. The new
language reflects this reality: it's surely better to remember a
channel which hasn't been broadcast than to forget one which has!
Similarly, for the fundee's `funding_signed` message: it's better to
remember a channel that never opens (and times out) than to let the
funder open it while the fundee has forgotten it.

以前の草案では、funderは
「funding transactionをブロードキャストしていればそれを覚えていなければならず、そうでなければそうしてはならない」
と主張していた：
これは実際のところ不可能な要件だった；
ノードは最初にディスクにコミットして、次にtransactionをブロードキャストするか、
またはその逆でなければならない。
新しい言葉遣いではこの現実を反映している：
ブロードキャストされていないチャンネルを覚えておくことは、ブロードキャストしたものを忘れるよりも、確実により良い！
同様に、fundeeのfunding_signedメッセージについて：
決して開かない（そしてタイムアウトする）チャンネルを覚えていることは、
fundeeがそれを忘れてしまっていて、funderがそれを開くことを許すよりも良い。

`option_data_loss_protect` was added to allow a node, which has somehow fallen behind
(e.g. has been restored from old backup), to detect that it's fallen-behind. A fallen-behind
node must know it cannot broadcast its current commitment transaction — which would lead to
total loss of funds — as the remote node can prove it knows the
revocation preimage. The error returned by the fallen-behind node
(or simply the invalid numbers in the `channel_reestablish` it has
sent) should make the other node drop its current commitment
transaction to the chain. This will, at least, allow the fallen-behind node to recover
non-HTLC funds, if the `my_current_per_commitment_point`
is valid. However, this also means the fallen-behind node has revealed this
fact (though not provably: it could be lying), and the other node could use this to
broadcast a previous state.

option_data_loss_protectは、何らかの形で後退した（例えば、古いバックアップから復元された）ノードが、
後退していることを検出するために追加された。
後退したノードは、
それの現在のcommitment transactionをブロードキャストできないことを知る必要がある、
それは全てのfundsを失うことに導く、
リモートノードはrevocation preimageを知っていることを証明することができるが。
後退ノードによって返されたエラー（または単にそれが送信したchannel_reestablish送信の不正な（XXX: 後退した）数値）によって、
他のノードが現在のcommitment transactionをチェーンにドロップするようにする必要がある。
これは、少なくとも、my_current_per_commitment_pointが有効であれば、
後退ノードがnon-HTLC fundsを回収することを可能にする。（XXX: to_remote）
しかしながら、これはまた、後退ノードがこの事実を明らかにしたことを意味する
（しかしそれは確かではない： それは嘘つきかもしれない）
他のノードはこれを使用して（XXX: 自分に都合のよい）以前の状態をブロードキャストできる。
（XXX: これは賭けである）

# Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
