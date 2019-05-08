# BOLT #5: Recommendations for On-chain Transaction Handling

## Abstract

Lightning allows for two parties (a local node and a remote node) to conduct transactions
off-chain by giving each of the parties a *cross-signed commitment transaction*,
which describes the current state of the channel (basically, the current balance).
This *commitment transaction* is updated every time a new payment is made and
is spendable at all times.

Lightningでは、2つのパーティ（local nodeとremote node）が、
channelの現在の状態（基本的には現在の残高）を記述するcross-signed commitment transactionを各パーティに提供することにより、
transactionsをオフチェーンで実行することができる。
このcommitment transactionは、新しい支払いが行われるたびに更新され、常に消費可能である。

There are three ways a channel can end:

1. The good way (*mutual close*): at some point the local and remote nodes agree
to close the channel. They generate a *closing transaction* (which is similar to a
commitment transaction, but without any pending payments) and publish it on the
blockchain (see [BOLT #2: Channel Close](02-peer-protocol.md#channel-close)).
2. The bad way (*unilateral close*): something goes wrong, possibly without evil
intent on either side. Perhaps one party crashed, for instance. One side
publishes its *latest commitment transaction*.
3. The ugly way (*revoked transaction close*): one of the parties deliberately
tries to cheat, by publishing an *outdated commitment transaction* (presumably,
a prior version, which is more in its favor).

channelを終了できる方法は3つある：

1. 良い方法（mutual close）：
ある時点で、local nodeとremote nodeはchannelを閉じることに同意する。
彼らは、closing transactionを生成し（commitment transactionに似ているが、保留中の支払いなし）、
blockchainに公開する（BOLT #2: Channel Close参照）。
2. 悪い方法（unilateral close）：
何かが良くない、おそらくどちらにも悪意がない。
多分おそらく当事者の一方がクラッシュした。
一方はlatest commitment transactionを公表する。
3. 危険な方法（revoked transaction close）：
当事者の一方が、outdated commitment transactionを公表することによって意図的に不正行為を試みている
（おそらく、以前のバージョンの方が有利である）。

Because Lightning is designed to be trustless, there is no risk of loss of funds
in any of these three cases; provided that the situation is properly handled.
The goal of this document is to explain exactly how a node should react when it
encounters any of the above situations, on-chain.

Lightningはtrustlessに設計されているため、これらの3つのケースでは資金の損失リスクはない。
状況が適切に処理されることを条件とするが。
このドキュメントの目的は、オンチェーンで上記のいずれかの状況に直面したときに、
nodeがどのように反応すべきか正確に説明することである。

# Table of Contents
  * [General Nomenclature](#general-nomenclature)
  * [Commitment Transaction](#commitment-transaction)
  * [Failing a Channel](#failing-a-channel)
  * [Mutual Close Handling](#mutual-close-handling)
  * [Unilateral Close Handling: Local Commitment Transaction](#unilateral-close-handling-local-commitment-transaction)
      * [HTLC Output Handling: Local Commitment, Local Offers](#htlc-output-handling-local-commitment-local-offers)
      * [HTLC Output Handling: Local Commitment, Remote Offers](#htlc-output-handling-local-commitment-remote-offers)
  * [Unilateral Close Handling: Remote Commitment Transaction](#unilateral-close-handling-remote-commitment-transaction)
      * [HTLC Output Handling: Remote Commitment, Local Offers](#htlc-output-handling-remote-commitment-local-offers)
      * [HTLC Output Handling: Remote Commitment, Remote Offers](#htlc-output-handling-remote-commitment-remote-offers)
  * [Revoked Transaction Close Handling](#revoked-transaction-close-handling)
	  * [Penalty Transactions Weight Calculation](#penalty-transactions-weight-calculation)
  * [General Requirements](#general-requirements)
  * [Appendix A: Expected Weights](#appendix-a-expected-weights)
	* [Expected Weight of the `to_local` Penalty Transaction Witness](#expected-weight-of-the-to-local-penalty-transaction-witness)
	* [Expected Weight of the `offered_htlc` Penalty Transaction Witness](#expected-weight-of-the-offered-htlc-penalty-transaction-witness)
	* [Expected Weight of the `accepted_htlc` Penalty Transaction Witness](#expected-weight-of-the-accepted-htlc-penalty-transaction-witness)
  * [Authors](#authors)

# General Nomenclature

Any unspent output is considered to be *unresolved* and can be *resolved*
as detailed in this document. Usually this is accomplished by spending it with
another *resolving* transaction. Although, sometimes simply noting the output
for later wallet spending is sufficient, in which case the transaction containing
the output is considered to be its own *resolving* transaction.

未使用の出力はunresolvedとみなされ、本書で説明するようにresolvedにできる。
（XXX: たぶんこの表現（Any）は誤解を生む。
unspentをspentしてもまたそれをspentしないとresolvedとみなせないならいつまでもresolvedにならない。
そうではなくあるunspent outputが競合状態、つまりまだ相手のものになる可能性があったり、
それをspentするためにnodeがスクリプトを覚えておかなければならない状態を解消したいのでは。
自分のウォレットに落ちるとこまで確定すればresolvedぐらいであろう。
たぶん最後が推奨として語られる）
通常、これは別のresolving transactionでそれを費やすことによって達成される。
後のウォレットの支出のためのoutputに注目すれば十分である場合もあるが、
その場合、outputを含むtransactionはそれ自身がresolving transactionと見なされる。

Outputs that are *resolved* are considered *irrevocably resolved*
once the remote's *resolving* transaction is included in a block at least 100
deep, on the most-work blockchain. 100 blocks is far greater than the
longest known Bitcoin fork and is the same wait time used for
confirmations of miners' rewards (see [Reference Implementation](https://github.com/bitcoin/bitcoin/blob/4db82b7aab4ad64717f742a7318e3dc6811b41be/src/consensus/tx_verify.cpp#L223)).

resolvedなoutputsは 、remoteのresolving transactionが、
最も長いblockchain上の少なくとも100の深さのブロック（XXX: 17時間弱）に含まれると、
irrevocably resolvedとみなされる。
100ブロックは、最も長い既知のBitcoin forkよりもはるかに大きく、
minersの報酬のconfirmationsに使用されるのと同じ待ち時間である
（Reference Implementation参照）。

## Requirements

A node:
  - once it has broadcast a funding transaction OR sent a commitment signature
  for a commitment transaction that contains an HTLC output:
    - until all outputs are *irrevocably resolved*:
      - MUST monitor the blockchain for transactions that spend any output that
      is NOT *irrevocably resolved*.
  - MUST *resolve* all outputs, as specified below.
  - MUST be prepared to resolve outputs multiple times, in case of blockchain
  reorganizations.
  - upon the funding transaction being spent, if the channel is NOT already
  closed:
    - SHOULD fail the channel.
    - MAY send a descriptive error packet.
  - SHOULD ignore invalid transactions.

A node:
  - 一旦それがfunding transactionをブロードキャストするか、
  またはHTLC outputを含むcommitment transactionのcommitment signatureを送った：
    - すべてのoutputがirrevocably resolvedになるまで：
      - irrevocably resolvedでないoutputsを費やすtransactionsのblockchainを監視しなければならない。
  - 以下に指定されるように、すべてのoutputsがresolveでないといけない。
  - blockchain reorganizationsの場合、outputsを何度も解決する準備をしなければならない。
  - funding transactionが費やされたときに、channelがまだクローズされていない場合：
    - channelに失敗すべきである。
    - 記述的なエラーパケットを送っても良い。
  - 無効なtransactionsを無視すべきである。

## Rationale

Once a local node has some funds at stake, monitoring the blockchain is required
to ensure the remote node does not close unilaterally.

local nodeの資金が賭けられると、remote nodeが一方的に閉じないようにblockchainを監視する必要がある。

Invalid transactions (e.g. bad signatures) can be generated by anyone,
(and will be ignored by the blockchain anyway), so they should not
trigger any action.

無効なtransactions（不正な署名など）は誰でも生成する可能性があるので
（とにかくblockchainによって無視されるため）、アクションを引き起こすべきではない。

# Commitment Transaction

The local and remote nodes each hold a *commitment transaction*. Each of these
commitment transactions has four types of outputs:

1. _local node's main output_: Zero or one output, to pay to the *local node's*
commitment pubkey.
2. _remote node's main output_: Zero or one output, to pay to the *remote node's*
commitment pubkey.
3. _local node's offered HTLCs_: Zero or more pending payments (*HTLCs*), to pay
the *remote node* in return for a payment preimage.
4. _remote node's offered HTLCs_: Zero or more pending payments (*HTLCs*), to
pay the *local node* in return for a payment preimage.

localとremote nodesはそれぞれにcommitment transactionが保持される。
これらのcommitment transactionsには、次の4種類のoutputsがある：

1. local nodeのmain output：（XXX: to_local）
local nodeのcommitment pubkeyに支払うためのゼロまたは1つの出力。
2. remote nodeのmain output：（XXX: to_remote）
remote nodeのcommitment pubkeyに支払うためのゼロまたは1つの出力。
3. local nodeのoffered HTLCs：（XXX: offered HTLCs）
payment preimageの引き換えとしてremote nodeに支払う、ゼロ以上の保留中の支払い（HTLCs）。
4. remote nodeのoffered HTLCs：（XXX: received HTLCs）
payment preimageの引き換えとしてlocal nodeに支払う、ゼロ以上の保留中の支払い（HTLCs）。

To incentivize the local and remote nodes to cooperate, an `OP_CHECKSEQUENCEVERIFY`
relative timeout encumbers the *local node's outputs* (in the *local node's
commitment transaction*) and the *remote node's outputs* (in the *remote node's
commitment transaction*). So for example, if the local node publishes its
commitment transaction, it will have to wait to claim its own funds,
whereas the remote node will have immediate access to its own funds. As a
consequence, the two commitment transactions are not identical, but they are
(usually) symmetrical.

localとremoteのnodesが連携するようにインセンティブを与えるために、
OP_CHECKSEQUENCEVERIFY相対タイムアウトは、
local nodeのoutputs（local nodeのcommitment transactionにおける）と
remote nodeのoutputs（remote nodeのcommitment transactionにおける）を妨げる。
（XXX: いずれもto_local）
たとえば、local nodeがcommitment transactionを公開している場合、
remote nodeは自分の資金にすぐにアクセスできるのに対し、
local nodeは自身の資金を請求するのを待たなければならない。
結果として、2つのcommitment transactionsは同一ではなく、（通常）対称的である。

See [BOLT #3: Commitment Transaction](03-transactions.md#commitment-transaction)
for more details.

詳細は、BOLT #3: Commitment Transaction参照。

# Failing a Channel

Although closing a channel can be accomplished in several ways, the most
efficient is preferred.

channelを閉じることはいくつかの方法で達成できるが、最も効率的である。

Various error cases involve closing a channel. The requirements for sending
error messages to peers are specified in
[BOLT #1: The `error` Message](01-messaging.md#the-error-message).

さまざまなエラーのケースが、channelを閉じることに伴う。
peersにエラーメッセージを送信するための要件は、BOLT #1: The error Messageで指定されている。

## Requirements

A node:
  - if a *local commitment transaction* has NOT ever contained a `to_local`
  or HTLC output:
    - MAY simply forget the channel.
  - otherwise:
    - if the *current commitment transaction* does NOT contain `to_local` or
    other HTLC outputs:
      - MAY simply wait for the remote node to close the channel.
      - until the remote node closes:
        - MUST NOT forget the channel.
    - otherwise:
      - if it has received a valid `closing_signed` message that includes a
      sufficient fee:
        - SHOULD use this fee to perform a *mutual close*.
      - otherwise:
        - MUST use the *last commitment transaction*, for which it has a
        signature, to perform a *unilateral close*.

A node:
  - local commitment transactionにまだto_localやHTLC outputが含まれたことがない場合：
    - 単にchannelを忘れて良い。
  - そうでなければ：
    - current commitment transactionにto_localとHTLC outputsが含まれていない場合：
      - 単にremote nodeがchannelを閉じるのを待っても良い。
      - remote nodeが閉じるまで：
        - channelを忘れてはいけない。
    - そうでなければ：
      - 十分なfeeを含む有効なclosing_signedメッセージを受信した場合：
        - このfeeを使用してmutual closeすべきである。
      - そうでなければ：
        - unilateral closeを実行するために、
        それが署名を有するlast commitment transactionを使用しなければならない。

## Rationale

Since `dust_limit_satoshis` is supposed to prevent creation of uneconomic
outputs (which would otherwise remain forever, unspent on the blockchain), all
commitment transaction outputs MUST be spent.

dust_limit_satoshisが非経済的なoutputsの生成を防ぐことが想定されているので
（そうでなければ永遠に残り、blockchainでは消費されない）、
すべてのcommitment transaction outputsを費やさなければならない。

In the early stages of a channel, it's common for one side to have
little or no funds in the channel; in this case, having nothing at stake, a node
need not consume resources monitoring the channel state.

channelの初期段階では、一方がchannelにほとんどかまったく資金を持たないことが一般的である。
この場合、何の問題もなく、nodeはchannel状態を監視するリソースを消費する必要はない。

There exists a bias towards preferring mutual closes over unilateral closes,
because outputs of the former are unencumbered by a delay and are directly
spendable by wallets. In addition, mutual close fees tend to be less exaggerated
than those of commitment transactions. So, the only reason not to use the
signature from `closing_signed` would be if the fee offered was too small for
it to be processed.

unilateral closesよりもmutual closesを好むバイアスがある。
これは、前者のoutputsが遅延に妨げられず、ウォレットによって直接消費できるためである。
さらに、mutual closeのfeesはcommitment transactionsよりも誇張されにくい傾向がある。
したがって、closing_signedからの署名を使用しない唯一の理由は、
提案されたfeeが処理するには小さすぎる場合である。

# Mutual Close Handling

A closing transaction *resolves* the funding transaction output.

closing transactionは、funding transaction outputをresolvesする。

In the case of a mutual close, a node need not do anything else, as it has
already agreed to the output, which is sent to its specified `scriptpubkey` (see
[BOLT #2: Closing initiation: `shutdown`](02-peer-protocol.md#closing-initiation-shutdown)).

それはすでにその指定されたscriptpubkeyに送られるoutputに合意したとして、
mutual closeの場合には、nodeは他に何もする必要がない
（BOLT #2: Closing initiation: `shutdown`参照）。

# Unilateral Close Handling: Local Commitment Transaction

This is the first of two cases involving unilateral closes. In this case, a
node discovers its *local commitment transaction*, which *resolves* the funding
transaction output.

これはunilateral closesを含む2つのケースのうちの最初のものである。
この場合、nodeはそのlocal commitment transactionを検出する、
これはfunding transaction outputをresolvesする。

However, a node cannot claim funds from the outputs of a unilateral close that
it initiated, until the `OP_CHECKSEQUENCEVERIFY` delay has passed (as specified
by the remote node's `to_self_delay` field). Where relevant, this situation is
noted below.

しかしながら、ノードは、OP_CHECKSEQUENCEVERIFYの遅延が経過するまで
（remote nodeのto_self_delayフィールドによって指定されるように）、
開始したunilateral closeのoutputsから資金を請求することができない。
関連するものとして、この状況を以下に記載する。

## Requirements

A node:
  - upon discovering its *local commitment transaction*:
    - SHOULD spend the `to_local` output to a convenient address.
    - MUST wait until the `OP_CHECKSEQUENCEVERIFY` delay has passed (as
    specified by the remote node's `to_self_delay` field) before spending the
    output.
      - Note: if the output is spent (as recommended), the output is *resolved*
      by the spending transaction, otherwise it is considered *resolved* by the
      commitment transaction itself.
    - MAY ignore the `to_remote` output.
      - Note: No action is required by the local node, as `to_remote` is
      considered *resolved* by the commitment transaction itself.
    - MUST handle HTLCs offered by itself as specified in
    [HTLC Output Handling: Local Commitment, Local Offers](#htlc-output-handling-local-commitment-local-offers).
    - MUST handle HTLCs offered by the remote node as
    specified in [HTLC Output Handling: Local Commitment, Remote Offers](#htlc-output-handling-local-commitment-remote-offers).

A node:
  - それのlocal commitment transaction発見時：
    - to_local outputを都合の良いアドレスに使うべきである。
    - outputを使う前にOP_CHECKSEQUENCEVERIFYの遅延が過ぎるまで待たなくてならない
    （remote nodeのto_self_delayフィールドで指定された通りに）。
      - 注：outputが費やされた場合（推奨通り）、outputはspending transactionによってresolvedになるか、
      そうでなければ、commitment transaction自体によってresolvedとみなされる。
      （XXX: でもこれだとredeem scriptを覚えておかないといけない。Rationale参照）
    - to_remote outputを無視してもよい。
      - 注：to_remoteは、commitment transaction自体によってresolvedと見なされるため、
      local nodeによるアクションは必要ない。
      （XXX: resolvedであるか否かに関わらずピアへの出力は関係ない。
      またto_remoteはper_commitment_pointに依存するため、
      HDウォレットにそのまま落ちないのでcommitment tx自体で消費されると考えていいのか？）
  - 「HTLC Output Handling: Local Commitment, Local Offers」で指定されているように、
  それ自体が提供するHTLCsを処理しなければならない。
  - 「HTLC Output Handling: Local Commitment, Remote Offers」で指定されているように、
  remote nodeによって提供されるHTLCsを処理しなければならない。

## Rationale

Spending the `to_local` output avoids having to remember the complicated
witness script, associated with that particular channel, for later
spending.

to_local outputを消費することは、後の支出のために、その特定のchannelに関連付けられている
複雑なwitness scriptを覚えておく必要がなくなる。

The `to_remote` output is entirely the business of the remote node, and
can be ignored.

to_remote outputは完全にremote nodeの仕事なので、無視することができる。

## HTLC Output Handling: Local Commitment, Local Offers

Each HTLC output can only be spent by either a local offerer, by using the HTLC-timeout
transaction after it's timed out, or a remote recipient, if it has the payment
preimage.

それぞれのHTLC outputは、
localの提供者がタイムアウト後にHTLC-timeout transactionを使用するか、
またはpayment preimageを持っていればremoteの受信者によって、
それらの場合によってのみ消費される。

There can be HTLCs which are not represented by an output: either
because they were trimmed as dust, or because the transaction has only been
partially committed.

outputによって表されていないHTLCsがある可能性がある：
それらがdustとして切り取られたか、
またはtransactionが部分的にコミットされただけであるためである。

The HTLC has *timed out* once the depth of the latest block is equal to
or greater than the HTLC `cltv_expiry`.

HTLCは、最新のブロックの深さがHTLCのcltv_expiry以上になるとtimed outになる。

### Requirements

A node:
  - if the commitment transaction HTLC output is spent using the payment
  preimage, the output is considered *irrevocably resolved*:
    - MUST extract the payment preimage from the transaction input witness.
  - if the commitment transaction HTLC output has *timed out* and hasn't been
  *resolved*:
    - MUST *resolve* the output by spending it using the HTLC-timeout
    transaction.
    - once the resolving transaction has reached reasonable depth:
      - MUST fail the corresponding incoming HTLC (if any).
      - MUST resolve the output of that HTLC-timeout transaction.
      - SHOULD resolve the HTLC-timeout transaction by spending it to a
      convenient address.
        - Note: if the output is spent (as recommended), the output is
        *resolved* by the spending transaction, otherwise it is considered
        *resolved* by the commitment transaction itself.
      - MUST wait until the `OP_CHECKSEQUENCEVERIFY` delay has passed (as
      specified by the remote node's `open_channel` `to_self_delay` field)
      before spending that HTLC-timeout output.
  - for any committed HTLC that does NOT have an output in this commitment
  transaction:
    - once the commitment transaction has reached reasonable depth:
      - MUST fail the corresponding incoming HTLC (if any).
    - if no *valid* commitment transaction contains an output corresponding to
    the HTLC.
      - MAY fail the corresponding incoming HTLC sooner.

A node:
  - commitment transaction HTLC outputがpayment preimageを使用して費やされた場合、
  その出力はirrevocably resolvedとみなされる。
    - transaction input witnessからpayment preimageを抽出しなければならない。
  - commitment transaction HTLC outputがtimed outしていて、まだresolvedになっていない場合：
    - HTLC-timeout transactionを使用してoutputをresolveしないといけない。
    - resolving transactionが合理的な深さに達すると：
      - 対応する入力HTLC（存在する場合）に失敗しなければならない。
      - そのHTLC-timeout transactionのoutputを解決しなければならない。
      - それを都合の良いアドレスに費やしてHTLC-timeout transactionを解決すべきである。
      　- 注：outputが費やされた場合（推奨通り）、outputはspending transactionによってresolvedになるか、
      　そうでなければ、commitment transaction（XXX: TODO: HTLC-timeout transactionじゃないの？）
      自体によってresolvedとみなされる。
      - HTLC-timeout outputを消費する前に、OP_CHECKSEQUENCEVERIFYの遅延が過ぎるまで、
      （remote nodeのopen_channelのto_self_delayフィールドで指定された通りに）待たなければならない。
  - このcommitment transactionでoutputを持たないコミット済みHTLCの場合：
    - commitment transactionが合理的な深さに達すると、
      - 対応する入力HTLC（存在する場合）に失敗しなければならない。
    - 有効なcommitment transactionがHTLCに対応する出力を含まない場合。
    （XXX: 下流の有効なlocal/remote commitment txがいずれも当該HTLC outputを含まない）
      - 対応する入力HTLCをすぐに失敗させてよい。

### Rationale

The payment preimage either serves to prove payment (when the offering node
originated the payment) or to redeem the corresponding incoming HTLC from
another peer (when the offering node is forwarding the payment). Once a node has
extracted the payment, it no longer cares about the fate of the HTLC-spending
transaction itself.

payment preimageは、（提供するnodeが支払いの発生元のとき）支払いを証明するか、
（提供するノードが支払いを転送しているときに）他のpeerから対応する入力HTLCを償還する。
nodeが支払いを（XXX: preimageを）抽出すると、
HTLC-spending transaction自体の成り行きに関心はなくなる。

In cases where both resolutions are possible (e.g. when a node receives payment
success after timeout), either interpretation is acceptable; it is the
responsibility of the recipient to spend it before this occurs.

両方の解決が可能である場合（例えば、nodeがタイムアウト後に支払い成功を受け取る場合）、
いずれの解釈も許容される：
これが起こる前にそれを費やすのは受取人の責任である。
（XXX: 下流がタイムアウトしたらオンチェーンでタイムアウトさせようとする。
ピアのupdate_fulfill_htlcはもう受け付けないほうがいい
（上流があきらかに余裕がある場合には受け付けてもいい）。
勝ってタイムアウトできたら上流へupdate_fail_htlcを送る。
負けてfulfillされた場合、上流に余裕があればupdate_fulfill_htlcを行い、
余裕がなければオンチェーンでfulfillしなければならない
（このときupdate_fulfill_htlcしてたらタイムアウトに負ける可能性がある）。
channelの生成消滅コストを考えると若干条件が変わるかもしれない。）

The local HTLC-timeout transaction needs to be used to time out the HTLC (to
prevent the remote node fulfilling it and claiming the funds) before the
local node can back-fail any corresponding incoming HTLC, using
`update_fail_htlc` (presumably with reason `permanent_channel_failure`), as
detailed in
[BOLT #2](02-peer-protocol.md#forwarding-htlcs).
If the incoming HTLC is also on-chain, a node must simply wait for it to
timeout: there is no way to signal early failure.

BOLT＃2に詳述されるように、
local nodeが、対応する入力HTLCをupdate_fail_htlcを使用して
（おそらくpermanent_channel_failureという理由で）元に戻す前に
local HTLC-timeout transactionを使用して、HTLCをタイムアウトさせて
（remote nodeがfulfillingして資金を請求するのを防ぐ）必要がある。
入力HTLCもオンチェーンの場合、nodeは単にタイムアウトするのを待たなければならない。
早期障害を知らせる手段はない。
（XXX: オンチェーンの場合、もうupdate_fail_htlcはできない）

If an HTLC is too small to appear in *any commitment transaction*, it can be
safely failed immediately. Otherwise, if an HTLC isn't in the *local commitment
transaction*, a node needs to make sure that a blockchain reorganization, or
race, does not switch to a commitment transaction that does contain the HTLC
before the node fails it (hence the wait). The requirement that the incoming
HTLC be failed before its own timeout still applies as an upper bound.

HTLCが小さすぎてany commitment transactionに現れない場合は、すぐに安全に失敗することができる。
そうでなければ、HTLCがlocal commitment transactionに含まれていない場合、
nodeは、blockchainのreorganizationまたは競争が、
HTLCを含むcommitment transactionに切り替えないことを確認する必要がある。
nodeがそれに失敗する前にそれをやる（したがって待機する）。
（XXX: これは支払い上流の話）
入力HTLCがそれ自身のタイムアウトの前に失敗する要件は依然として上限として適用される。
（XXX: ？）

## HTLC Output Handling: Local Commitment, Remote Offers

Each HTLC output can only be spent by the recipient, using the HTLC-success
transaction, which it can only populate if it has the payment
preimage. If it doesn't have the preimage (and doesn't discover it), it's
the offerer's responsibility to spend the HTLC output once it's timed out.

各々のHTLC outputは、HTLC-success transactionを使用して、受取人が費やすことができる。
これは、payment preimageがある場合にのみ投入できる。
もしそれがpreimageを持っていない（そしてそれを発見していない）ならば、
タイムアウトした後、HTLC outputを使うのは、提供者（XXX: remote node）の責任である。

There are several possible cases for an offered HTLC:

1. The offerer is NOT irrevocably committed to it. The recipient will usually
   not know the preimage, since it will not forward HTLCs until they're fully
   committed. So using the preimage would reveal that this recipient is the
   final hop; thus, in this case, it's best to allow the HTLC to time out.
2. The offerer is irrevocably committed to the offered HTLC, but the recipient
   has not yet committed to an outgoing HTLC. In this case, the recipient can
   either forward or timeout the offered HTLC.
3. The recipient has committed to an outgoing HTLC, in exchange for the offered
   HTLC. In this case, the recipient must use the preimage, once it receives it
   from the outgoing HTLC; otherwise, it will lose funds by sending an outgoing
   payment without redeeming the incoming payment.

offered HTLCには、いくつかのケースが考えられる：

1. 提供者は、それ（XXX: HTLC）をirrevocably committedにしていない。
受信者は通常、完全にコミットするまでHTLCsを転送しないため、preimageを知らない。
従って、（XXX: 中継者だったら知らないはずの）preimageを使用すると、
この受信者が最終的なhop（XXX: final node）であることが明らかになってしまう。
従って、この場合、HTLCのタイムアウトを許可するのが最善である。
2. 提供者は、offered HTLCにirrevocably committedしているが、受信者はまだ出力HTLCにコミットしていない。
この場合、受信者はoffered HTLCを転送またはタイムアウトすることができる。
（XXX: 受信者の自由）
3. 受信者は、offered HTLCと引き換えに、出力HTLCをコミットしている。
この場合、受信側は、出力HTLCから受信したpreimageを使用する必要がある。
そうでなければ、それは入金を償還せずに出金を送ることによって資金を失うだろう。

### Requirements

A local node:
  - if it receives (or already possesses) a payment preimage for an unresolved
  HTLC output that it has been offered AND for which it has committed to an
  outgoing HTLC:
    - MUST *resolve* the output by spending it, using the HTLC-success
    transaction.
    - MUST resolve the output of that HTLC-success transaction.
  - otherwise:
    - if the *remote node* is NOT irrevocably committed to the HTLC:
      - MUST NOT *resolve* the output by spending it.
  - SHOULD resolve that HTLC-success transaction output by spending it to a
  convenient address.
  - MUST wait until the `OP_CHECKSEQUENCEVERIFY` delay has passed (as specified
    by the *remote node's* `open_channel`'s `to_self_delay` field), before
    spending that HTLC-success transaction output.

A local node:
  - 提供されたunresolvedのHTLC outputのpayment preimageを受信し（または既に所有している）、
  それに対応する出力HTLCにコミットしている場合：
  （XXX: TODO: 最後の条件は間違っている？当該入力HTLCがirrevocably committedであるべき）
    - HTLC-success transactionを使用してoutputをresolveしなければならない。
    - そのHTLC-success transactionのoutputを解決しなければならない。
  - そうでなければ：
    - remote nodeがHTLCをirrevocably committedしていない場合：
      - それ（XXX: payment preimage）を使ってoutputをresolveにしてはいけない。
      （XXX: final nodeであることが露呈する。）
  - 都合の良いアドレスを使ってHTLC-success transaction outputを解決すべきである。
  - HTLC-success transaction outputを使う前に、
  OP_CHECKSEQUENCEVERIFYの遅延が経過するまで待たなければならない。
  （remote nodeのopen_channelのto_self_delayで指定されるように）。

If the output is spent (as is recommended), the output is *resolved* by
the spending transaction, otherwise it's considered *resolved* by the commitment
transaction itself.

outputが使用された場合（推奨されるように）、outputはspending transactionによってresolvedになる。
そうでなければ、commitment transaction自体によってresolvedと見なされる。
（XXX: TODO: ここはcommitment transactionじゃなくてHTLC-success transaction？）

If it's NOT otherwise resolved, once the HTLC output has expired, it is
considered *irrevocably resolved*.

それ以外の方法で解決されない場合は、HTLC outputが期限切れになれば、irrevocably resolvedとみなされる。
（XXX: 「それ以外の方法で解決されない場合」って？）

# Unilateral Close Handling: Remote Commitment Transaction

The *remote node's* commitment transaction *resolves* the funding
transaction output.

remote nodeのcommitment transactionがfunding transactionのoutputをresolvesにする。

There are no delays constraining node behavior in this case, so it's simpler for
a node to handle than the case in which it discovers its local commitment
transaction (see [Unilateral Close Handling: Local Commitment Transaction](#unilateral-close-handling-local-commitment-transaction)).

この場合、nodeの動作を制約する遅延はないため、
nodeがlocal commitment transactionを検出した場合よりも簡単に処理できる
（Unilateral Close Handling: Local Commitment Transaction参照）。

## Requirements

A local node:
  - upon discovering a *valid* commitment transaction broadcast by a
  *remote node*:
    - if possible:
      - MUST handle each output as specified below.
      - MAY take no action in regard to the associated `to_remote`, which is
      simply a P2WPKH output to the *local node*.
        - Note: `to_remote` is considered *resolved* by the commitment transaction
        itself.
      - MAY take no action in regard to the associated `to_local`, which is a
      payment output to the *remote node*.
        - Note: `to_local` is considered *resolved* by the commitment transaction
        itself.
      - MUST handle HTLCs offered by itself as specified in
      [HTLC Output Handling: Remote Commitment, Local Offers](#htlc-output-handling-remote-commitment-local-offers)
      - MUST handle HTLCs offered by the remote node as specified in
      [HTLC Output Handling: Remote Commitment, Remote Offers](#htlc-output-handling-remote-commitment-remote-offers)
    - otherwise (it is NOT able to handle the broadcast for some reason):
      - MUST send a warning regarding lost funds.


A local node:
  - remote nodeによる有効なcommitment transactionのブロードキャストを発見：
    - 可能なら：
      - 次のように各outputを処理しなければならない。
      - local nodeへの単純なP2WPKH outputである、関連するto_remoteについては何もしなくてよい。
        - 注：to_remoteはcommitment transaction自身によってresolvedと見なされる。
        （XXX: TODO: to_remoteはremotepubkeyへのP2WPKH outputだが
        per_commitment_secretに依存するので、HDウォレットに落とすまでやったほうがいい）
      - remote nodeへの支払いである、関連するto_localについては何もしなくてよい。
      　- 注：to_localはcommitment transaction自身によってresolvedと見なされる。
      - HTLC Output Handling: Remote Commitment, Local Offersで指定されるように、
      自身が提供するHTLCsを処理しなければならない
      - HTLC Output Handling: Remote Commitment, Remote Offersで指定されるように、
      remote nodeによって提供されるHTLCsを処理しなければならない
    - そうでなければ（何らかの理由でブロードキャストを処理できない）：
      - 失われた資金に関する警告を送信しなければならない。
      （XXX: TODO: プロトコル外だがこういったことを多分やらないといけない）

## Rationale

There may be more than one valid, *unrevoked* commitment transaction after a
signature has been received via `commitment_signed` and before the corresponding
`revoke_and_ack`. As such, either commitment may serve as the *remote node's*
commitment transaction; hence, the local node is required to handle both.

commitment_signedで署名が受け取られた後、対応するrevoke_and_ackの前に、
1つ以上の有効なunrevokedなcommitment transactionが存在する可能性がある。
このように、どちらのcommitmentもremote nodeのcommitment transactionとして機能することができる。
したがって、local nodeは両方を処理する必要がある。
（XXX: TODO: ３つ以上もありうるのであればeitherやbothという表現は不適切であろう）

In the case of data loss, a local node may reach a state where it doesn't
recognize all of the *remote node's* commitment transaction HTLC outputs. It can
detect the data loss state, because it has signed the transaction, and the
commitment number is greater than expected. If both nodes support
`option_data_loss_protect`, the local node will possess the remote's
`per_commitment_point`, and thus can derive its own `remotepubkey` for the
transaction, in order to salvage its own funds. Note: in this scenario, the node
will be unable to salvage the HTLCs.

データが損失している場合、
local nodeは、remote nodeのcommitment transaction HTLC outputsを認識しない状態にあることがある。
transactionに署名しており、commitment numberが予想よりも大きいため、データ損失状態を検出できる。
両方のnodeがoption_data_loss_protectをサポートしている場合、
local nodeはremote nodeのper_commitment_pointを所有するため
従って、自己の資金をサルベージするために、そのtransactionのための、自身のremotepubkeyを導出できる。
（XXX: to_remoteはサルベージできる）
注：このシナリオでは、nodeはHTLCsをサルベージすることができない。
（XXX: なぜ？payment_hashとcltv_expiryが残っていないだろうから？）

## HTLC Output Handling: Remote Commitment, Local Offers

Each HTLC output can only be spent by the *offerer*, after it's timed out, or by
the *recipient*, if it has the payment preimage.

各HTLC outputは、期限切れになった後にoffererによって消費されるか、
またはpayment preimageがある場合はrecipientによって消費される。

The HTLC output has *timed out* once the depth of the latest block is equal to
or greater than the HTLC `cltv_expiry`.

最新のブロックの深度がHTLC cltv_expiry以上になると、HTLC outputがtimed outする。。

There can be HTLCs which are not represented by any outputs: either
because the outputs were trimmed as dust or because the remote node has two
*valid* commitment transactions with differing HTLCs.

outputsがdustとしてトリミングされたか、
またはremote nodeにHTLCsが異なる2つの有効なcommitment transactionsがあるため、
outputsによってHTLCsが現れない可能性がある。

### Requirements

A local node:
  - if the commitment transaction HTLC output is spent using the payment
  preimage:
    - MUST extract the payment preimage from the HTLC-success transaction input
    witness.
      - Note: the output is considered *irrevocably resolved*.
  - if the commitment transaction HTLC output has *timed out* AND NOT been
  *resolved*:
    - MUST *resolve* the output, by spending it to a convenient address.
  - for any committed HTLC that does NOT have an output in this commitment
  transaction:
    - once the commitment transaction has reached reasonable depth:
      - MUST fail the corresponding incoming HTLC (if any).
    - otherwise:
      - if no *valid* commitment transaction contains an output corresponding to
      the HTLC:
        - MAY fail it sooner.

A local node:
  - commitment transaction HTLC outputがpayment preimageを使用して費やされた場合：
    - HTLC-success transaction input witnessからpayment preimageを抽出しなければならない。
      - 注：outputはirrevocably resolvedとみなされる。      
  - commitment transaction HTLC outputがtimed outとなりresolvedになっていない場合：
    - それを手頃なアドレスに費やしてoutputをresolveしなければならない。
  - このcommitment transactionではoutputを持たないコミット済みのHTLCのためには
  （XXX: トリムされたのであろう）：  
    - commitment transactionが合理的な深さに達すると：
    （XXX: それまでは別のversionのcommitment transactionに置き換わる可能性がある）  
      - 対応する入力HTLC（存在する場合）を失敗しなければならない。
    - そうでなければ：  
      - 有効なcommitment transactionがHTLCに対応するoutputを含まない場合
      （XXX: 全ての有効なcommitment transactionでトリムされているのであろう）：
        - すぐに失敗しても良い。

### Rationale

If the commitment transaction belongs to the *remote* node, the only way for it
to spend the HTLC output (using a payment preimage) is for it to use the
HTLC-success transaction.

remote nodeに属するcommitment transactionの場合、
HTLC outputを（payment preimageを使用して）費やす唯一の方法は、
HTLC-success transactionを使用することである。

The payment preimage either serves to prove payment (when the offering node is
the originator of the payment) or to redeem the corresponding incoming HTLC from
another peer (when the offering node is forwarding the payment). After a node has
extracted the payment, it no longer need be concerned with the fate of the
HTLC-spending transaction itself.

payment preimageは、（提供ノードが支払いの発信者であるとき）支払いを証明するため、
または（提供ノードが支払いを転送しているときに）他のピアからの対応する入力HTLCを引き換えるために役立つ。
nodeが支払いを抽出した後、HTLC-spending transactionそのもの成り行きに関心を持つ必要はなくなる。

In cases where both resolutions are possible (e.g. when a node receives payment
success after timeout), either interpretation is acceptable: it's the
responsibility of the recipient to spend it before this occurs.

両方の解決が可能な場合（nodeがタイムアウト後に支払い成功を受け取るなど）には、
どちらかの解釈が許容される：これが発生する前に受信者がそれを費やすのは担当者の責任である。

Once it has timed out, the local node needs to spend the HTLC output (to prevent
the remote node from using the HTLC-success transaction) before it can
back-fail any corresponding incoming HTLC, using `update_fail_htlc`
(presumably with reason `permanent_channel_failure`), as detailed in
[BOLT #2](02-peer-protocol.md#forwarding-htlcs).
If the incoming HTLC is also on-chain, a node simply waits for it to
timeout, as there's no way to signal early failure.

タイムアウトすると、
update_fail_htlcを使って（おそらくpermanent_channel_failureという理由で）対応する入力HTLCを元に戻す前に、
local nodeはHTLC outputを費やす必要がある
（remote nodeがHTLC-success transactionを使用しないように（XXX: タイムアウト後に受信したpreimageで））。
BOLT #2に詳述されるように。
入力HTLCもオンチェーンである場合、早期障害を通知する手段がないため、nodeは単にタイムアウトするのを待つだけである。

If an HTLC is too small to appear in *any commitment transaction*, it
can be safely failed immediately. Otherwise,
if an HTLC isn't in the *local commitment transaction* a node needs to make sure
that a blockchain reorganization or race does not switch to a
commitment transaction that does contain it before the node fails it: hence
the wait. The requirement that the incoming HTLC be failed before its
own timeout still applies as an upper bound.

HTLCが小さすぎてcommitment transactionに現れない場合は、すぐに安全に失敗できる。
そうでなければ、HTLCがlocal commitment transactionに含まれていない場合、
nodeは、blockchainのreorganizationまたはレースによって、
nodeが失敗する前に（XXX: 入力HTLCを失敗させる前に）、
HTLCを含むcommitment transactionに切り替わらないか確認する必要がある。
入力HTLCがそれ自身のタイムアウトの前に失敗する要件は依然として上限として適用される。
（XXX: ？）

## HTLC Output Handling: Remote Commitment, Remote Offers

Each HTLC output can only be spent by the recipient if it uses the payment
preimage. If a node does not possess the preimage (and doesn't discover
it), it's the offerer's responsibility to spend the HTLC output once it's timed
out.

各HTLC outputは、payment preimageを使用する場合にのみ受取人が費やすことができる。
nodeがpreimageを所有していない（そしてそれを発見していない）場合は、
期限切れになるとHTLC outputを費やすことは提供者の責任である。

The remote HTLC outputs can only be spent by the local node if it has the
payment preimage. If the local node does not have the preimage (and doesn't
discover it), it's the remote node's responsibility to spend the HTLC output
once it's timed out.

remote HTLC outputsは、payment preimageを有する場合にのみ、local nodeによって費やされ得る。
local nodeにpreimageがない（そしてそれを発見していない）場合は、
期限切れになるとHTLC outputを費やすことはremote nodeの責任である。

There are actually several possible cases for an offered HTLC:

1. The offerer is not irrevocably committed to it. In this case, the recipient
   usually won't know the preimage, since it won't forward HTLCs until
   they're fully committed. As using the preimage would reveal that
   this recipient is the final hop, it's best to allow the HTLC to time out.
2. The offerer is irrevocably committed to the offered HTLC, but the recipient
   hasn't yet committed to an outgoing HTLC. In this case, the recipient can
   either forward it or wait for it to timeout.
3. The recipient has committed to an outgoing HTLC in exchange for an offered
   HTLC. In this case, the recipient must use the preimage, if it receives it
   from the outgoing HTLC; otherwise, it will lose funds by sending an outgoing
   payment without redeeming the incoming one.

offered HTLCには、実際にはいくつかのケースが考えられる：

1. 提供者はそれをirrevocably committedにしていない。
この場合、受信者は通常、完全にコミットされるまでHTLCsを転送しないため、preimageを知らない。
preimageを使用すると、この受信者が最終的なhopであることが明らかになるので、
HTLCのタイムアウトを許可するのが最善である。
2. 提供者は、offered HTLCをirrevocably committedにしているが、
受取人はまだ出力HTLCにコミットしていない。
この場合、受信者はそれを転送またはタイムアウトするまで待つことができる。
（XXX: 受信者の自由）
3. 受信者は、offered HTLCと引き換えに、出力HTLCをコミットしている。
この場合、受信側は、出力HTLCから受信したpreimageを使用する必要がある。
そうでなければ、それは入金を償還せずに出金を送ることによって資金を失うだろう。

### Requirements

A local node:
  - if it receives (or already possesses) a payment preimage for an unresolved
  HTLC output that it was offered AND for which it has committed to an
outgoing HTLC:
    - MUST *resolve* the output by spending it to a convenient address.
  - otherwise:
    - if the remote node is NOT irrevocably committed to the HTLC:
      - MUST NOT *resolve* the output by spending it.

A local node:
  - 提供されたunresolvedのHTLC outputのpayment preimageを受け取っている（または既に所有している）おり、
  それのための出力HTLCにコミットしている場合：
  （XXX: TODO: 最後の条件は間違っている？当該入力HTLCがirrevocably committedであるべき）
    - それを手頃なアドレスに費やしてoutputをresolveにしなければならない。
  - そうでなければ：
    - remote nodeがそのHTLCをirrevocably committedにしていない場合：
      - それを使ってoutputをresolveにしてはならない。（XXX: タイムアウトさせる）

If not otherwise resolved, once the HTLC output has expired, it is considered
*irrevocably resolved*.

他の方法で解決されず、HTLC outputが期限切れになったら、irrevocably resolvedとみなされる。

# Revoked Transaction Close Handling

If any node tries to cheat by broadcasting an outdated commitment transaction
(any previous commitment transaction besides the most current one), the other
node in the channel can use its revocation private key to claim all the funds from the
channel's original funding transaction.

いずれかのノードが、古いcommitment transaction（最新のcommitment transaction以外の以前のもの）を
ブロードキャストして不正行為しようとする場合、
（XXX: TODO: 有効なcommitment txは複数ありうるのでこの文言は不正確。）
channelの他のnodeはrevocation private keyを使用してchannelの元のfunding transactionから、
すべての資金を請求することができる。

## Requirements

Once a node discovers a commitment transaction for which *it* has a
revocation private key, the funding transaction output is *resolved*.

nodeが、
そのためのrevocation private keyを持っている、
commitment transactionを発見すると、
funding transaction outputはresolvedである。
(XXX: なぜ唐突にresolvedなのか？)

A local node:
  - MUST NOT broadcast a commitment transaction for which *it* has exposed the
  `per_commitment_secret`.
  - MAY take no action regarding the _local node's main output_, as this is a
  simple P2WPKH output to itself.
    - Note: this output is considered *resolved* by the commitment transaction
      itself.
  - MUST *resolve* the _remote node's main output_ by spending it using the
  revocation private key.
  - MUST *resolve* the _local node's offered HTLCs_ in one of three ways:
    * spend the *commitment tx* using the payment revocation private key.
    * spend the *commitment tx* using the payment preimage (if known).
    * spend the *HTLC-timeout tx*, if the remote node has published it.
  - MUST *resolve* the _remote node's offered HTLCs_ in one of two ways:
    * spend the *commitment tx* using the payment revocation key.
    * spend the *commitment tx* once the HTLC timeout has passed.
  - MUST *resolve* the _remote node's HTLC-timeout transaction_ by spending it
  using the revocation private key.
  - MUST *resolve* the _remote node's HTLC-success transaction_ by spending it
  using the revocation private key.
  - SHOULD extract the payment preimage from the transaction input witness, if
  it's not already known.
  - MAY use a single transaction to *resolve* all the outputs.
  - MUST handle its transactions being invalidated by HTLC transactions.

A local node:
  - それがper_commitment_secretを公開したら、commitment transactionをブロードキャストしてはならない。
  - local nodeのmain outputに関しては何もしなくて良い。これはそれ自身の単純なP2WPKH outputである。
    - 注：このoutputはcommitment transaction自身によってresolvedであるとみなされる。
  - revocation private keyを使用してremote nodeのmain outputをresolveしなければならない。
  - local（XXX: remoteでは？） nodeの、
  offered HTLCsを次の3つの方法のいずれかでresolveしなければならない。
    * payment revocation private keyを使用してcommitment txを費やす。
    * （既知であれば）payment preimageを使用してcommitment txを費やす。
    * remote nodeが公開している場合は、HTLC-timeout txを費やす。
  - remote（XXX: localでは？） nodeの、
  offered HTLCsを次の2つの方法のいずれかでresolveしなければならない。
    * payment revocation keyを使用してcommitment txを費やす。
    （XXX: TODO: なぜこちらにはprivateが付かない？）
    * HTLC timeoutが過ぎていれば、commitment txを費やす。
    （XXX: TODO: HTLC-success txを費やすケースも入れるべき）
  - revocation private keyを使用して、remote nodeのHTLC-timeout transactionをresolveする。
  - revocation private keyを使用して、remote nodeのHTLC-success transactionをresolveする。
  - もしそれがまだ既知ではない場合、（XXX: remoteのHTLC-success transactionの）
  transaction input witnessからpayment preimageを抽出すべきである。
  - 単一のtransactionを使用してすべてのoutputをresolveすることができる。
  - HTLC transactionsによって無効にされるそのtransactionsを処理しなければならない。
  （XXX: outputsを消費しようとしたらremoteのHTLC transactionsに負けてしまったケース）

## Rationale

A single transaction that resolves all the outputs will be under the
standard size limit because of the 483 HTLC-per-party limit (see
[BOLT #2](02-peer-protocol.md#the-open_channel-message)).

すべてのoutputsを解決する単一のtransactionは、
483 HTLC-per-party limit（XXX: max_accepted_htlcs）（BOLT＃2参照）のため、
標準のサイズ制限を下回る。

Note: if a single transaction is used, it may be invalidated if the remote node
refuses to broadcast the HTLC-timeout and HTLC-success transactions in a timely
manner. Although, the requirement of persistence until all outputs are
irrevocably resolved, should still protect against this happening. [ FIXME: May have to divide and conquer here, since the remote node may be able to delay the local node long enough to avoid a successful penalty spend? ]

注：単一のtransactionが使用される場合、
remote nodeがタイムリーにHTLC-timeoutおよびHTLC-success transactionsをブロードキャストすることにより拒絶された場合、
それは無効になる可能性がある。
従って、すべてのoutputsがirrevocably resolvedになるまでの持続性の要件は、この事態を未然に防ぐべきである。
[FIXME：remote nodeがペナルティーの消費の成功を避けるために十分長い間、
local nodeを遅らせることができるので、ここで分裂して克服しなければならないかもしれない]
（XXX: おそらく単一のtransactionで消費すべきではない。メーリングリストのtrustless watchtower参照）

## Penalty Transactions Weight Calculation

There are three different scripts for penalty transactions, with the following
witness weights (details of weight computation are in
[Appendix A](#appendix-a-expected-weights)):

penalty transactionsのための3つの異なるスクリプトがあり、以下のwitness weightsがある
（weight計算の詳細はAppendix Aにある）。（XXX: witness部分はweights換算でも4倍にしない。
このto_localはremoteへの出力）

    to_local_penalty_witness: 160 bytes
    offered_htlc_penalty_witness: 243 bytes
    accepted_htlc_penalty_witness: 249 bytes

The penalty *txinput* itself takes up 41 bytes and has a weight of 164 bytes,
which results in the following weights for each input:

ペナルティtxinput（XXX: 各々の入力）自体は41バイトを占め、
164バイトのweightを持ち（XXX: 41 * 4 = 164）、
各inputに対して以下のweightsを与える。（XXX: 前述のものに164足した）
（XXX: https://github.com/bitcoin/bips/blob/master/bip-0144.mediawiki のtxins）


    to_local_penalty_input_weight: 324 bytes
    offered_htlc_penalty_input_weight: 407 bytes
    accepted_htlc_penalty_input_weight: 413 bytes

The rest of the penalty transaction takes up 4+1+1+8+1+34+4=53 bytes of
non-witness data: assuming it has a pay-to-witness-script-hash (the largest
standard output script), in addition to a 2-byte witness header.

残りのpenalty transactionは、
pay-to-witness-script-hash（最大の標準出力スクリプト）とみなすと、
4 + 1 + 1 + 8 + 1 + 34 + 4 = 53バイトのnon-witnessデータを占める。
あと2-byte witness headerに加える。（XXX: おそらくmaker、flagで2バイト）
（XXX:<br>
4: version<br>
1: vin count<br>
1: vout count<br>
8: amount<br>
1: scriptpubkeyの長さ<br>
34: txouts（P2WSHとみなす）<br>
4: lock_time<br>
）

In addition to spending these outputs, a penalty transaction may optionally
spend the commitment transaction's `to_remote` output (e.g. to reduce the total
amount paid in fees). Doing so requires the inclusion of a P2WPKH witness and an
additional *txinput*, resulting in an additional 108 + 164 = 272 bytes.

これらのoutputsを費やすことに加えて、
penalty transactionは、オプションとして、
commitment transactionのto_remote output（XXX: これは自分に対する）を費やすことがある。
（例えば、feesの支払い総額を減らすために）
これを行うには、P2WPKH witnessと追加のtxinputを含める必要があり、
その結果、108（XXX: witness） + 164（XXX: txinput） = 272バイトが追加される。

In the worst case scenario, the node holds only incoming HTLCs, and the
HTLC-timeout transactions are not published, which forces the node to spend from
the commitment transaction.

最悪の場合のシナリオ（XXX: 単一のトランザクションの最大サイズ）では、
ノードは入力HTLC（XXX: サイズが大きい）のみを保持し、
HTLC-timeout transactionsは公開されず、
nodeはcommitment transactionから消費することを強いられる。

With a maximum standard weight of 400000 bytes, the maximum number of HTLCs that
can be swept in a single transaction is as follows:

最大標準重量が400000バイトの場合、単一のtransactionで掃き集めることができるHTLCsの最大数は次のとおりである。

    max_num_htlcs = (400000 - 324 - 272 - (4 * 53) - 2) / 413 = 966

（XXX:<br>
400000: bitcoinで最大txが100000bytes、400000weights<br>
324: to_local_penalty_input_weight<br>
272: to_remote<br>
4 * 53: non-witness部分（XXX: weightなので4倍）<br>
2: witness header（XXX: これもwitness同様4倍しない）<br>
413: accepted_htlc_penalty_input_weight<br>
）

Thus, 483 bidirectional HTLCs (containing both `to_local` and
`to_remote` outputs) can be resolved in a single penalty transaction.
Note: even if the `to_remote` output is not swept, the resulting
`max_num_htlcs` is 967; which yields the same unidirectional limit of 483 HTLCs.

したがって、483個の双方向HTLCs（to_localとto_remote outputsの両方を含む）は、
単一のpenalty transactionで解決することができる。
注：to_remote outputが掃き集めない場合でも、結果のmax_num_htlcsは967である。
これは同じ単方向の限界の483 HTLCsをもたらす。

# General Requirements

A node:
  - upon discovering a transaction that spends a funding transaction output
  which does not fall into one of the above categories (mutual close, unilateral
  close, or revoked transaction close):
    - MUST send a warning regarding lost funds.
      - Note: the existence of such a rogue transaction implies that its private
      key has leaked and that its funds may be lost as a result.
  - MAY simply monitor the contents of the most-work chain for transactions.
    - Note: on-chain HTLCs should be sufficiently rare that speed need not be
    considered critical.
  - MAY monitor (valid) broadcast transactions (a.k.a the mempool).
    - Note: watching for mempool transactions should result in lower latency
    HTLC redemptions.

A node:
  - 上記のカテゴリのいずれかに該当しない、funding transaction outputを消費するtransaction
    （mutual close、unilateral close、またはrevoked transaction close）を発見すると、
    - 失われた資金に関する警告を送信しなければならない。（XXX: どうやって？）
      - 注：このような不正なtransactionの存在は、private keyが漏洩し、
      その結果として資金が失われる可能性があることを意味する。
  - transactionsのために最も長いchainの内容を監視してもよい。
    - 注：オンチェーンHTLCsは、速度が重要と見なされる必要がないほど、稀にすべきである。（XXX: ？）
  - （有効な）broadcast transactions（別名mempool）を監視してもよい。
    - 注：mempool transactionsを監視することで、待ち時間の少ないHTLCの償還が行われるはずである。

# Appendix A: Expected Weights

## Expected Weight of the `to_local` Penalty Transaction Witness

As described in [BOLT #3](03-transactions.md), the witness for this transaction
is:

    <sig> 1 { OP_IF <revocationpubkey> OP_ELSE to_self_delay OP_CSV OP_DROP <local_delayedpubkey> OP_ENDIF OP_CHECKSIG }

（XXX: これが下記のto_local_penalty_witnessで、\<sig\> 1 \<to_local_script\>）

The *expected weight* of the `to_local` penalty transaction witness is
calculated as follows:

    to_local_script: 83 bytes
        - OP_IF: 1 byte
            - OP_DATA: 1 byte (revocationpubkey length)
            - revocationpubkey: 33 bytes
        - OP_ELSE: 1 byte
            - OP_DATA: 1 byte (delay length)
            - delay: 8 bytes
            - OP_CHECKSEQUENCEVERIFY: 1 byte
            - OP_DROP: 1 byte
            - OP_DATA: 1 byte (local_delayedpubkey length)
            - local_delayedpubkey: 33 bytes
        - OP_ENDIF: 1 byte
        - OP_CHECKSIG: 1 byte

    to_local_penalty_witness: 160 bytes
        - number_of_witness_elements: 1 byte
        - revocation_sig_length: 1 byte
        - revocation_sig: 73 bytes
        - one_length: 1 byte
        - witness_script_length: 1 byte
        - witness_script (to_local_script)

## Expected Weight of the `offered_htlc` Penalty Transaction Witness

The *expected weight* of the `offered_htlc` penalty transaction witness is
calculated as follows (some calculations have already been made in
[BOLT #3](03-transactions.md)):

    offered_htlc_script: 133 bytes

    offered_htlc_penalty_witness: 243 bytes
        - number_of_witness_elements: 1 byte
        - revocation_sig_length: 1 byte
        - revocation_sig: 73 bytes
        - revocation_key_length: 1 byte
        - revocation_key: 33 bytes
        - witness_script_length: 1 byte
        - witness_script (offered_htlc_script)

## Expected Weight of the `accepted_htlc` Penalty Transaction Witness

（XXX: accepted -> receivedだろう）

The *expected weight*  of the `accepted_htlc` penalty transaction witness is
calculated as follows (some calculations have already been made in
[BOLT #3](03-transactions.md)):

    accepted_htlc_script: 139 bytes

    accepted_htlc_penalty_witness: 249 bytes
        - number_of_witness_elements: 1 byte
        - revocation_sig_length: 1 byte
        - revocation_sig: 73 bytes
        - revocationpubkey_length: 1 byte
        - revocationpubkey: 33 bytes
        - witness_script_length: 1 byte
        - witness_script (accepted_htlc_script)

# Authors

[FIXME:]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
