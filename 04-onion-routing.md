# BOLT #4: Onion Routing Protocol

## Overview

This document describes the construction of an onion routed packet that is
used to route a payment from an _origin node_ to a _final node_. The packet
is routed through a number of intermediate nodes, called _hops_.

この文書では、origin nodeからfinal nodeに支払いをルーティングするために使用されるonion routed packetの構成について説明する。
パケットは、hopsと呼ばれるいくつかのintermediate nodesを経由してルーティングされる。
（XXX: これが受信update_add_htlcに入っている）

The routing schema is based on the [Sphinx][sphinx] construction and is
extended with a per-hop payload.

ルーティングスキーマはSphinxの構文に基づいており、per-hopペイロードで拡張されている。

Intermediate nodes forwarding the message can verify the integrity of
the packet and can learn which node they should forward the
packet to. They cannot learn which other nodes, besides their
predecessor or successor, are part of the packet's route; nor can they learn
the length of the route or their position within it. The packet is
obfuscated at each hop, to ensure that a network-level attacker cannot
associate packets belonging to the same route (i.e. packets belonging
to the same route do not share any correlating information). Notice that this
does not preclude the possibility of packet association by an attacker
via traffic analysis.

メッセージを転送するintermediate nodesは、パケットの完全性を検証し、どのノードにパケットを転送すべきかを知ることができる。
前任者または後継者のほかに、パケットのルートの一部である他のノードを知ることはできない。
ルートの長さやその中の位置を知ることもできない。
パケットは、各hopで難読化され、ネットワークレベルの攻撃者が同じルートに属するパケットを関連付けることはできない
（つまり、同じルートに属するパケットは相関情報を共有しない）。
これが、トラフィック分析による攻撃者によるパケット関連付けの可能性を排除するものではないことに注意すること。

The route is constructed by the origin node, which knows the public
keys of each intermediate node and of the final node. Knowing each node's public key
allows the origin node to create a shared secret (using ECDH) for each
intermediate node and for the final node. The shared secret is then
used to generate a _pseudo-random stream_ of bytes (which is used to obfuscate
the packet) and a number of _keys_ (which are used to encrypt the payload and
compute the HMACs). The HMACs are then in turn used to ensure the integrity of
the packet at each hop.

ルートは、各intermediate nodeおよびfinal nodeの公開鍵を知っているorigin nodeによって構築される。
各ノードの公開鍵を知ることにより、origin nodeは、各intermediate nodeおよびfinal nodeに対してshared secret（ECDHを使用して）を作成することができる。
次に、shared secretを使用して、（パケットを難読化するために使用される）pseudo-random streamのバイトと、
多数のkeysを生成する（ペイロードを暗号化し、HMACを計算するために使用される）。
次に、HMACsは、各hopでパケットの完全性を保証するために使用される。

Each hop along the route only sees an ephemeral key for the origin node, in
order to hide the sender's identity. The ephemeral key is blinded by each
intermediate hop before forwarding to the next, making the onions unlinkable
along the route.

ルートに沿った各hopは、送信者のアイデンティティを隠すために、origin nodeのためのephemeral keyしか見ることができない。
ephemeral keyは中間の各hopによってブラインドされてから、次のノードに転送され、ルートに沿ってリンクできないonionsを作る。
（XXX: 各hopがブラインドする？）

This specification describes _version 0_ of the packet format and routing
mechanism.

この仕様はversion 0のパケットフォーマットとルーティングメカニズムについて説明する。

A node:
  - upon receiving a higher version packet than it implements:
    - MUST report a route failure to the origin node.
    - MUST discard the packet.

ノード：
  - それよりも高いバージョンのパケットを受信すると：
    - origin nodeに経路障害を報告しなければならない。
    - パケットを破棄しなければならない。

# Table of Contents

  * [Conventions](#conventions)
  * [Key Generation](#key-generation)
  * [Pseudo Random Byte Stream](#pseudo-random-byte-stream)
  * [Packet Structure](#packet-structure)
    * [Payload for the Last Node](#payload-for-the-last-node)
  * [Shared Secret](#shared-secret)
  * [Blinding Ephemeral Keys](#blinding-ephemeral-keys)
  * [Packet Construction](#packet-construction)
  * [Packet Forwarding](#packet-forwarding)
  * [Filler Generation](#filler-generation)
  * [Returning Errors](#returning-errors)
    * [Failure Messages](#failure-messages)
    * [Receiving Failure Codes](#receiving-failure-codes)
  * [Test Vector](#test-vector)
    * [Returning Errors](#returning-errors)
  * [References](#references)
  * [Authors](#authors)

# Conventions

There are a number of conventions adhered to throughout this document:

 - HMAC: the integrity verification of the packet is based on Keyed-Hash
   Message Authentication Code, as defined by the [FIPS 198
   Standard][fips198]/[RFC 2104][RFC2104], and using a `SHA256` hashing
   algorithm.
 - Elliptic curve: for all computations involving elliptic curves, the Bitcoin
   curve is used, as specified in [`secp256k1`][sec2]
 - Pseudo-random stream: [`ChaCha20`][rfc7539] is used to generate a
   pseudo-random byte stream. For its generation, a fixed null-nonce
   (`0x0000000000000000`) is used, along with a key derived from a shared
   secret and with a `0x00`-byte stream of the desired output size as the
   message.
 - The terms _origin node_ and _final node_ refer to the initial packet sender
   and the final packet recipient, respectively.
 - The terms _hop_ and _node_ are sometimes used interchangeably, but a _hop_
   usually refers to an intermediate node in the route rather than an end node.
        _origin node_ --> _hop_ --> ... --> _hop_ --> _final node_
 - The term _processing node_ refers to the specific node along the route that is
   currently processing the forwarded packet.
 - The term _peers_ refers only to hops that are direct neighbors (in the
   overlay network): more specifically, _sending peers_ forward packets
   to _receiving peers_.
 - Each hop in the route has a variable length `hop_payload`, or a fixed-size
   legacy `hop_data` payload.
    - The legacy `hop_data` is identified by a single `0x00`-byte prefix
	- The variable length `hop_payload` is prefixed with a `varint` encoding
      the length in bytes, excluding the prefix and the trailing HMAC.

このドキュメントでは、いくつかの規則がある：

 - HMAC：パケットの完全性検証は、FIPS 198 Standard / RFC 2104で定義されているように、
 Keyed-Hash Message Authentication Codeに基づいている、SHA256ハッシュアルゴリズムを使用する。
 - Elliptic curve：楕円曲線を含むすべての計算では、secp256k1で指定されているように、Bitcoin curveが使用されている。
 - Pseudo-random stream：ChaCha20がpseudo-random byte streamを生成するために使用される。
 その生成のために、固定ヌルナンス（0x0000000000000000）が、
 shared secretから派生したキーと、
 メッセージとして期待される出力サイズの0x00-byte streamとともに使用される。（XXX: 0x00-byte streamって？）
 - origin nodeおよびfinal nodeという用語は、それぞれ、初期パケット送信者および最終パケット受信者を指す。
 - hopとnodeという用語は同じ意味で使用されることもあるが、hopは通常、end nodeではなくルート内のintermediate nodeを指す。
        _origin node_ --> _hop_ --> ... --> _hop_ --> _final node_
 - processing nodeという用語は、現在転送されているパケットを処理しているルートに沿った特定のnodeを指す。
 - peersという用語は、直接の隣人たち（オーバレイネットワーク内の）であるホップのみを指す。
 つまり、sending peersからreceiving peersにパケットを転送する。
 - ルート内の各ホップには、可変長のhop_payload、または固定長の従来のhop_dataペイロードがある。
    - レガシーのhop_dataは、単一の0x00バイト接頭辞によって識別される。
    - 可変長のhop_payloadの先頭には、接頭辞と末尾のHMACを除いたバイト単位の長さをエンコードするvarintが接頭辞になる。

# Key Generation

A number of encryption and verification keys are derived from the shared secret:

 - _rho_: used as key when generating the pseudo-random byte stream that is used
   to obfuscate the per-hop information
 - _mu_: used during the HMAC generation
 - _um_: used during error reporting

shared secretからいくつかのencryption keysとverification keysが導出される：

 - rho：hopごとの情報を難読化するために使用されるpseudo-random byte streamを生成するときにキーとして使用される
 - mu：HMAC生成中に使用される
 - um：エラー報告時に使用される

The key generation function takes a key-type (_rho_=`0x72686F`, _mu_=`0x6d75`,
or _um_=`0x756d`) and a 32-byte secret as inputs and returns a 32-byte key.

キー生成関数は、
キータイプ（rho = 0x72686F、mu = 0x6d75、またはum = 0x756d）と
32バイトのシークレットを入力として受け取り、
32バイトのキーを返す。
（XXX: TODO: ammagの説明が必要。場合によっては使っていないだろうけどその逆のgammaも）

Keys are generated by computing an HMAC (with `SHA256` as hashing algorithm)
using the appropriate key-type (i.e. _rho_, _mu_, or _um_) as HMAC-key and the
32-byte shared secret as the message. The resulting HMAC is then returned as the
key.

鍵は、適切な鍵タイプ（すなわち、rho、mu、またはum）をHMAC-keyとして使用し、
32バイトのshared secretをメッセージとして使用して、
（ハッシングアルゴリズムをSHA256として）HMACを計算することによって生成される。
結果のHMACがキーとして返される。

Notice that the key-type does not include a C-style `0x00`-termination-byte,
e.g. the length of the _rho_ key-type is 3 bytes, not 4.

key-typeにはC言語スタイルの0x00-termination-byteは含まれていないことに注意すること。
例えば、rhoのkey-typeの長さは4ではなく3バイトである。

# Pseudo Random Byte Stream

The pseudo-random byte stream is used to obfuscate the packet at each hop of the
path, so that each hop may only recover the address and HMAC of the next hop.
The pseudo-random byte stream is generated by encrypting (using `ChaCha20`) a
`0x00`-byte stream, of the required length, which is initialized with a key
derived from the shared secret and a zero-nonce (`0x00000000000000`).

pseudo-random byte streamは、パスの各hopでパケットを難読化するために使用されるため、
各hopは次のhopのアドレスとHMACのみを回復することができる。
pseudo-random byte streamは、必要な長さの0x00-byte streamを暗号化（ChaCha20を使用）することによって生成される。
これは、shared secretから得られたキーとゼロナンス（0x00000000000000）で初期化される。

The use of a fixed nonce is safe, since the keys are never reused.

キーが再利用されることはないため、固定ナンスの使用は安全である。

# Packet Structure

The packet consists of four sections:

 - a `version` byte
 - a 33-byte compressed `secp256k1` `public_key`, used during the shared secret
   generation
 - a 1300-byte `hop_payloads` consisting of multiple, variable length,
   `hop_payload` payloads or up to 20 fixed sized legacy `hop_data` payloads.
 - a 32-byte `HMAC`, used to verify the packet's integrity

パケットは4つのセクションで構成される：

 - versionバイト
 - 33バイトの圧縮されたsecp256k1 public_key、shared secretの生成の間に使用される
 - 複数の可変長のhop_payloadペイロード、または最大20個の固定サイズのレガシーhop_dataペイロードで構成される1300バイトのhop_payload。
 （XXX: TODO: これは以下でrouting informationと呼ばれている？）
 - 32バイトのHMAC、パケットの完全性を検証するために使用される

The network format of the packet consists of the individual sections
serialized into one contiguous byte-stream and then transferred to the packet
recipient. Due to the fixed size of the packet, it need not be prefixed by its
length when transferred over a connection.

パケットのネットワーク形式は、個々のセクションが1つの連続したバイトストリームにシリアライズされ、パケット受信者に転送される。
パケットが固定サイズのために、接続上に転送されたときにその長さの接頭語を付ける必要はない。

The overall structure of the packet is as follows:

パケットの全体的な構造は次のとおり：

1. type: `onion_packet`
2. data:
   * [`byte`:`version`]
   * [`point`:`public_key`]
   * [`1300*byte`:`hop_payloads`]
   * [`32*byte`:`hmac`]

For this specification (_version 0_), `version` has a constant value of `0x00`.

この仕様（version 0）ではversionは定数値0x00を持つ。

The `hop_payloads` field is a structure that holds obfuscated routing information, and associated HMAC.
It is 1300 bytes long and has the following structure:

hop_payloadsフィールドは、難読化されたルーティング情報および関連するHMACを保持する構造である。
このフィールドの長さは1300バイトで、次の構造を持つ。

1. type: `hop_payloads`
2. data:
   * [`varint`:`length`]
   * [`hop_payload_length`:`hop_payload`]
   * [`32*byte`:`HMAC`]
   * ...
   * `filler`

Where, the `length`, `hop_payload` (with contents dependent on `length`), and `HMAC` are repeated for each hop;
and where, `filler` consists of obfuscated, deterministically-generated padding, as detailed in [Filler Generation](#filler-generation).
Additionally, `hop_payloads` is incrementally obfuscated at each hop.

ここで、length、hop_payload（内容はlengthに依存する）、およびHMACは、各ホップについて繰り返される;
fillerは、Filler Generationで説明されているように、難読化された決定論的に生成されたパディングで構成されている。
また、hop_payloadsは各ホップで段階的に難読化される。

Using the `hop_payload` field, the origin node is able to specify the path and structure of the HTLCs forwarded at each hop.
As the `hop_payload` is protected under the packet-wide HMAC, the information it contains is fully authenticated with each pair-wise relationship between the HTLC sender (origin node) and each hop in the path.

origin nodeは、hop_payloadフィールドを使用して、各ホップで転送されるHTLCのパスと構造を指定できる。
hop_payloadはパケット全体のHMACの下で保護されるので、
そこに含まれる情報は、HTLC送信側（origin node）とパス内の各ホップとの間の各ペアごとの関係によって完全に認証されます。

Using this end-to-end authentication, each hop is able to cross-check the HTLC
parameters with the `hop_payload`'s specified values and to ensure that the
sending peer hasn't forwarded an ill-crafted HTLC.

このエンドツーエンド認証を使用すると、各hopはhop_payloadで指定された値でHTLCパラメータ
（XXX: 入力update_add_htlcの他のフィールド）をクロスチェックし、
送信ピアが不正なHTLCを転送していないことを確認できる。

The `length` field determines both the length and the format of the `hop_payload` field; the following formats are defined:

lengthフィールドは、hop_payloadフィールドの長さと形式の両方を決定する;
次の形式が定義されている。

 - Legacy `hop_data` format, identified by a single `0x00` byte for length. In this case the `hop_payload_length` is defined to be 32 bytes.
 - `tlv_payload` format, identified by any length over `1`. In this case the `hop_payload_length` is equal to the numeric value of `length`.
 - A single `0x01` byte for length is reserved for future use to signal a different payload format. This is safe since no TLV value can ever be shorter than 2 bytes. In this case the `hop_payload_length` MUST be defined in the future specification making use of this `length`.

 - 従来のhop_dataフォーマット。長さは1つの0x00バイトで識別される。この場合、hop_payload_lengthは32バイトとして定義される。
 - tlv_payloadフォーマット。1を超える任意の長さとして識別される。 この場合、hop_payload_lengthはlengthの数値と等しくなる。
 - lengthのための単一の0x01バイトは、異なるペイロード・フォーマットを通知するために将来使用するために予約されている。
 TLVの値が2バイトより短くなることはないため、これは安全である。この場合、hop_payload_lengthは、このlengthを利用して将来の仕様で定義されなければならない。

## Legacy `hop_data` payload format

The `hop_data` format is identified by a single `0x00`-byte length, for backward compatibility.
Its payload is defined as:

hop_dataフォーマットは、下位互換性のために、単一の0x00バイト長で識別される。
そのペイロードは次のように定義される。

1. type: `hop_data` (for `realm` 0)
2. data:
   * [`short_channel_id`:`short_channel_id`]
   * [`u64`:`amt_to_forward`]
   * [`u32`:`outgoing_cltv_value`]
   * [`12*byte`:`padding`]

Field descriptions:

   * `short_channel_id`: The ID of the outgoing channel used to route the
      message; the receiving peer should operate the other end of this channel.

   * `amt_to_forward`: The amount, in millisatoshis, to forward to the next
     receiving peer specified within the routing information.

     This value amount MUST include the origin node's computed _fee_ for the
     receiving peer. When processing an incoming Sphinx packet and the HTLC
     message that it is encapsulated within, if the following inequality doesn't hold,
     then the HTLC should be rejected as it would indicate that a prior hop has
     deviated from the specified parameters:

          incoming_htlc_amt - fee >= amt_to_forward

     Where `fee` is either calculated according to the receiving peer's advertised fee
     schema (as described in [BOLT #7](07-routing-gossip.md#htlc-fees))
     or is 0, if the processing node is the final node.

   * `outgoing_cltv_value`: The CLTV value that the _outgoing_ HTLC carrying
     the packet should have.

          cltv_expiry - cltv_expiry_delta >= outgoing_cltv_value

     Inclusion of this field allows a hop to both authenticate the information
     specified by the origin node, and the parameters of the HTLC forwarded,
	 and ensure the origin node is using the current `cltv_expiry_delta` value.
     If there is no next hop, `cltv_expiry_delta` is 0.
     If the values don't correspond, then the HTLC should be failed and rejected, as
     this indicates that either a forwarding node has tampered with the intended HTLC
     values or that the origin node has an obsolete `cltv_expiry_delta` value.
     The hop MUST be consistent in responding to an unexpected
     `outgoing_cltv_value`, whether it is the final node or not, to avoid
     leaking its position in the route.

   * `padding`: This field is for future use and also for ensuring that future non-0-`realm`
     `hop_data`s won't change the overall `hop_payloads` size.

フィールドの説明：

   * short_channel_id：メッセージをルーティングするために使用される出力チャネルのID。
     受信ピアはこのチャネルのもう一方の端を操作する必要がある。

   * amt_to_forward：ルーティング情報内で指定された次の受信ピアに転送するmillisatoshis単位の量。

     この値の量は、受信ピアのための、origin nodeが計算したfeeを含まなければならない。
     入ってくるSphinxパケットと、カプセル化されているHTLCメッセージを処理するとき、
     次の不等式が成り立たない場合、
     前のhopが指定されたパラメータから逸脱していることを示すので、HTLCを拒否する必要がある。
     （XXX: incoming_htlc_amtはupdate_add_htlcのamount_msat）

        incoming_htlc_amt - fee >= amt_to_forward

     feeは受信ピアのアドバタイズされたfee schema（BOLT＃7で説明）に従って計算されるか、
     またはprocessing nodeがfinal nodeである場合は0である。

   * outgoing_cltv_value：パケットを運ぶ出力HTLCが持たなければならないCLTV値。

        cltv_expiry - cltv_expiry_delta >= outgoing_cltv_value

     このフィールドを含めることによりhopは、
     origin nodeによって指定された情報と転送されるHTLCのパラメータの両方を認証することができ、
     origin nodeが現在のcltv_expiry_delta値を使用していることを保証することができる。
     次のhopがない場合、cltv_expiry_deltaは0である。
     （XXX: final nodeではoutgoing_cltv_valueがinvoiceからの値。後述）

     値が一致しない場合、
     これは、転送ノードのいずれかが意図されたHTLC値を改ざんしたこと、
     又はorigin nodeが廃止されたcltv_expiry_delta値を持つことを示し、
     HTLCは失敗し拒絶されるべきである。
     hopは、final nodeであろうとなかろうと、予期せぬoutgoing_cltv_valueに応答して、
     経路上の位置が漏れないように一貫していなければならない。
     （XXX: ？）

   * padding：このフィールドは将来の使用のためであり、
   将来のnon-0-realm hop_dataが全体のhop_payloadsサイズを変更しないことを保証するためのフィールドである。

When forwarding HTLCs, nodes MUST construct the outgoing HTLC as specified
within `hop_data` above; otherwise, deviation from the specified HTLC
parameters may lead to extraneous routing failure.

HTLCsを転送するとき、nodesは上記のようにhop_data内で指定されたように、出力HTLCを構築しなければならない。
そうしないと、指定されたHTLCパラメータからの逸脱が無関係なルーティング障害につながる可能性がある。

### `tlv_payload` payload format

This is a more flexible format, which avoids the redundant `short_channel_id` field for the final node.

これはより柔軟な形式で、最終ノードの冗長なshort_channel_idフィールドを回避する。

1. tlvs: `tlv_payload`
2. types:
    1. type: 2 (`amt_to_forward`)
    2. data:
        * [`tu64`:`amt_to_forward`]
    1. type: 4 (`outgoing_cltv_value`)
    2. data:
        * [`tu32`:`outgoing_cltv_value`]
    1. type: 6 (`short_channel_id`)
    2. data:
        * [`short_channel_id`:`short_channel_id`]

### Requirements

The writer:
  - MUST include `amt_to_forward` and `outgoing_cltv_value` for every node.
  - MUST include `short_channel_id` for every non-final node.
  - MUST NOT include `short_channel_id` for the final node.

The writer:
  - すべてのノードにamt_to_forwardとoutgoing_cltv_valueを含めなければならない。
  - 全ての非最終ノードに対してshort_channel_idを含まなければならない。
  - 最後のノードにshort_channel_idを含めてはならない。

The reader:
  - MUST return an error if `amt_to_forward` or `outgoing_cltv_value` are not present.

The reader:
  - amt_to_forwardまたはoutgoing_cltv_valueが存在しない場合は、エラーを返さなければならない。
（XXX: 削除されたshort_channel_idに関する要件を追加しないといけない）

The requirements for the contents of these fields are specified [above](#legacy-hop_data-payload-format).

これらのフィールドの内容の要件は、上記で指定される。

# Accepting and Forwarding a Payment

Once a node has decoded the payload it either accepts the payment locally, or forwards it to the peer indicated as the next hop in the payload.

ノードがペイロードをデコードすると、ローカルで支払いを受け入れるか、ペイロードのネクストホップとして示されているピアにペイロードを転送する。

## Non-strict Forwarding

A node MAY forward an HTLC along an outgoing channel other than the one
specified by `short_channel_id`, so long as the receiver has the same node
public key intended by `short_channel_id`. Thus, if `short_channel_id` connects
nodes A and B, the HTLC can forwarded across any channel connecting A and B.
Failure to adhere will result in the receiver being unable to decrypt the next
hop in the onion packet.

ノードは、short_channel_idで指定されたノード公開鍵と同じノードを持つ限り、
short_channel_idで指定されている以外の出力チャネルに沿ってHTLCを転送してもよい。
したがって、short_channel_idがノードAおよびBを接続する場合、
HTLCはAおよびBを接続する任意のチャネルを介して転送することができる。
遵守しなければ、受信者はonion packetの次のhopを復号できなくなる。

### Rationale

In the event that two peers have multiple channels, the downstream node will be
able to decrypt the next hop payload regardless of which channel the packet is
sent across.

2つのピアが複数のチャネルを有する場合、
下流のノードは、パケットがどのチャネルを介して送信されるかにかかわらず、
次のhopのペイロードを復号することができる。

Nodes implementing non-strict forwarding are able to make real-time assessments
of channel bandwidths with a particular peer, and use the channel that is
locally-optimal.

非厳密な転送を実装するノードは、
特定のピアとのチャネル帯域幅のリアルタイム評価を行い、
局所的に最適なチャネルを使用することができる。

For example, if the channel specified by `short_channel_id` connecting A and B
does not have enough bandwidth at forwarding time, then A is able use a
different channel that does. This can reduce payment latency by preventing the
HTLC from failing due to bandwidth constraints across `short_channel_id`, only
to have the sender attempt the same route differing only in the channel between
A and B.

たとえば、AとBを接続するshort_channel_idで指定されたチャネルが転送時に十分な帯域幅を持たない場合、
Aはそれとは異なるチャネルを使用できる。
これにより、short_channel_idの帯域幅制約のためにHTLCが失敗するのを防止し、
送信者がAとBの間のチャネルでのみ異なる同じルートを試みるようにすることで、支払いの待ち時間を短縮できる。

Non-strict forwarding allows nodes to make use of private channels connecting
them to the receiving node, even if the channel is not known in the public
channel graph.

厳密でない転送は、チャネルがパブリックチャネルグラフで知られていなくても、
ノードが受信ノードにそれらを接続するプライベートチャネルを利用することを可能にする。

### Recommendation

Implementations using non-strict forwarding should consider applying the same
fee schedule to all channels with the same peer, as senders are likely to select
the channel which results in the lowest overall cost. Having distinct policies
may result in the forwarding node accepting fees based on the most optimal fee
schedule for the sender, even though they are providing aggregate bandwidth
across all channels with the same peer.

厳密でない転送を使用する実装では、
同じピアを持つすべてのチャネルに同じ料金スケジュールを適用することを検討する必要がある。
送信者は、全体のコストが最も低いチャネルを選択する可能性が高いからである。
別個のポリシーを有することにより、
転送ノードは、同じピアを有するすべてのチャネルにわたって総帯域幅を提供しているにもかかわらず、
送信者にとって最も最適な料金スケジュールに基づいて料金を受諾することになる可能性がある。
（XXX: 総帯域幅が大きさにも関わらず、一番すくないchannelのfeeになる可能性がある）

Alternatively, implementations may choose to apply non-strict forwarding only to
like-policy channels to ensure their expected fee revenue does not deviate by
using an alternate channel.

代わりに、実装は、同様のポリシーチャネルにのみ非厳密な転送を適用することを選択して、
代理チャネルを使用して予想される料金収入が逸脱しないようにすることができる。

## Payload for the Last Node

When building the route, the origin node MUST use a payload for
the final node with the following values:

* `outgoing_cltv_value`: set to the final expiry specified by the recipient (e.g.
  `min_final_cltv_expiry` from a [BOLT #11](11-payment-encoding.md) payment invoice)
* `amt_to_forward`: set to the final amount specified by the recipient (e.g. `amount`
  from a [BOLT #11](11-payment-encoding.md) payment invoice)

ルートを構築する場合、origin nodeは、次の値を持つfinal nodeのペイロードを使用しなければならない：
outgoing_cltv_value：受信者によって指定されたfinal expiryに設定される
（e.g. BOLT #11 payment invoiceのmin_final_cltv_expiry)
amt_to_forward：受信者によって指定された最終金額に設定される
（e.g. BOLT #11 payment invoiceのamount）

This allows the final node to check these values and return errors if needed,
but it also eliminates the possibility of probing attacks by the second-to-last
node. Such attacks could, otherwise, attempt to discover if the receiving peer is the
last one by re-sending HTLCs with different amounts/expiries.
The final node will extract its onion payload from the HTLC it has received and
compare its values against those of the HTLC. See the
[Returning Errors](#returning-errors) section below for more details.

これにより、final nodeはこれらの値をチェックし、必要に応じてエラーを返すことができるが、
最後から2番目のノードによる探査攻撃の可能性もなくなる。（XXX: TODO: なぜ？）
そのような攻撃は、そうでなければ、受信ピアが異なるamounts/expiriesを持つHTLCを再送することによって、
最後のピアであるかどうかを発見しようとする可能性がある。
final nodeは、受け取ったHTLCからそのonionのペイロードを抽出し、その値をHTLCの値と比較する。
詳細については、 以下の「Returning Errors」を参照。

If not for the above, since it need not forward payments, the final node could
simply discard its payload.

上記に該当しない場合、支払いの転送を必要としないので、
final nodeはそのペイロードを単に破棄することができる。
（XXX: update_add_htlcをエラーにする？）

# Shared Secret

The origin node establishes a shared secret with each hop along the route using
Elliptic-curve Diffie-Hellman between the sender's ephemeral key at that hop and
the hop's node ID key. The resulting curve point is serialized to the
DER-compressed representation and hashed using `SHA256`. The hash output is used
as the 32-byte shared secret.

origin nodeは、ルート上の各hopとのshared secretを、
送信者のそのhopにおけるephemeral keyとhopのnode ID keyとの間のElliptic-curve Diffie-Hellmanを用いて確立する。
得られたcurve pointは、DER-compressed表現にシリアライズされ、SHA256を使用してハッシュ化される。
ハッシュ出力は、32バイトのshared secretとして使用される。

Elliptic-curve Diffie-Hellman (ECDH) is an operation on an EC private key and
an EC public key that outputs a curve point. For this protocol, the ECDH
variant implemented in `libsecp256k1` is used, which is defined over the
`secp256k1` elliptic curve. During packet construction, the sender uses the
ephemeral private key and the hop's public key as inputs to ECDH, whereas
during packet forwarding, the hop uses the ephemeral public key and its own
node ID private key. Because of the properties of ECDH, they will both derive
the same value.

Elliptic-curve Diffie-Hellman（ECDH）は、EC private keyとEC public keyを操作して、curve pointを出力する。
このプロトコルでは、libsecp256k1で実装されたECDHの変形が使用され、これはsecp256k1 elliptic curve上で定義される。
パケット構築中、送信者はephemeral private keyとhopのpublic keyをECDHへの入力として使用するが、
パケット転送ではephemeral public keyと、
それ自身のnode ID private keyを使用する。
ECDHの特性のために、彼らは両方とも同じ値を導き出すであろう。

（XXX: ephemeral private/public keys、node_id private/public keysの組みあわせ）

# Blinding Ephemeral Keys

In order to ensure multiple hops along the route cannot be linked by the
ephemeral public keys they see, the key is blinded at each hop. The blinding is
done in a deterministic way that the allows the sender to compute the
corresponding blinded private keys during packet construction.

ルートに沿った複数のhopsが、
彼らが見るephemeral public keysによってリンクすることができないようにすることを保証するため、
鍵は各hopでブラインドされている。
ブラインドは、
送信者がパケット構築中に対応するblinded private keysを計算することを可能にする決定論的な方法で行われる。

The blinding of an EC public key is a single scalar multiplication of
the EC point representing the public key with a 32-byte blinding factor. Due to
the commutative property of scalar multiplication, the blinded private key is
the multiplicative product of the input's corresponding private key with the
same blinding factor.

EC public keyのブラインドは、
public keyを表すEC pointと32バイトのblinding factorとの１つのスカラ乗算である。
スカラ乗算の交換法則のため、blinded private keyは、
入力の対応するprivate keyと同じblinding factorとの乗算の量である。
（XXX: b*P = (b*x)G。b*xがblinded private key）

The blinding factor itself is computed as a function of the ephemeral public key
and the 32-byte shared secret. Concretely, is the `SHA256` hash value of the
concatenation of the public key serialized in its compressed format and the
shared secret.

blinding factor自体は、ephemeral public keyと32バイトのshared secretの関数として計算される。
具体的には、圧縮形式でシリアライズされたpublic keyとshared secretとの連結のSHA256ハッシュ値である。

# Packet Construction

In the following example, it's assumed that a _sending node_ (origin node),
`n_0`, wants to route a packet to a _receiving node_ (final node), `n_r`.
First, the sender computes a route `{n_0, n_1, ..., n_{r-1}, n_r}`, where `n_0`
is the sender itself and `n_r` is the final recipient. All nodes `n_i` and
`n_{i+1}` MUST be peers in the overlay network route. The sender then gathers the
public keys for `n_1` to `n_r` and generates a random 32-byte `sessionkey`.
Optionally, the sender may pass in _associated data_, i.e. data that the
packet commits to but that is not included in the packet itself. Associated
data will be included in the HMACs and must match the associated data provided
during integrity verification at each hop.

以下の例では、
sending node（origin node）n_0がreceiving node（final node）n_rにパケットをルーティングしたいと仮定している。
まず、送信者がルート{n_0, n_1, ..., n_{r-1}, n_r}を計算する場合、n_0は送信者自身であり、n_rは最終的な受信者である。
すべてのノードn_iとn_{i+1}はオーバレイネットワークルートのピアでなければならない。
次に、送信側はn_1からn_rのpublic keysを収集し、ランダムな32バイトsessionkeyを生成する。
任意に、送信者は、associated dataすなわち、パケットがコミットするがパケット自体には含まれないデータを渡すことができる。
associated dataはHMACsに含まれ、各hopで完全性検証中に提供されるassociated dataと一致しなければならない。
（XXX: HTLCではpayment_hashがassociated dataになる）

To construct the onion, the sender initializes the ephemeral private key for the
first hop `ek_1` to the `sessionkey` and derives from it the corresponding
ephemeral public key `epk_1` by multiplying with the `secp256k1` base point. For
each of the `k` hops along the route, the sender then iteratively computes the
shared secret `ss_k` and ephemeral key for the next hop `ek_{k+1}` as follows:

onionを構築するために、送信者は、最初のhopのためのephemeral private key ek_1をsessionkeyで初期化すると、
そこから対応するephemeral public key epk_1をsecp256k1 base pointを乗算して導出する。
ルートに沿ったk hopsのそれぞれについて、
送信側はそれから、
shared secret ss_kと
次のhopのephemeral key　ek_{k+1}
を繰り返し計算する。

 - The sender executes ECDH with the hop's public key and the ephemeral private
 key to obtain a curve point, which is hashed using `SHA256` to produce the
 shared secret `ss_k`.
 - The blinding factor is the `SHA256` hash of the concatenation between the
 ephemeral public key `epk_k` and the shared secret `ss_k`.
 - The ephemeral private key for the next hop `ek_{k+1}` is computed by
 multiplying the current ephemeral private key `ek_k` by the blinding factor.
 - The ephemeral public key for the next hop `epk_{k+1}` is derived from the
 ephemeral private key `ek_{k+1}` by multiplying with the base point.

（XXX: 区切り）

 - 送信者は、
 curve pointを得るために、hopのpublic keyとephemeral private key（XXX: ek_k）でECDHを実行する、
 これはshared secret ss_kを生成するためにSHA256でハッシュされる。
 - blinding factorは、ephemeral public key epk_kとshared secret ss_kとの連結のSHA256ハッシュである。
 - 次のhopのephemeral private key ek_{k+1}は、現在のephemeral private key ek_kにblinding factorを掛けることによって計算される。
 - 次のhopのephemeral public key epk_{k+1}は、ephemeral private key ek_{k+1}にbase pointを乗算することにより導出される。
 （XXX:<br>
 SHA256(pk_k * ek_k) => ss_k。<br>
 SHA256(epk_k || ss_k) => bf_k。<br>
 ek_k * bf_k => ek_{k+1}。<br>
 ek_{k+1} * G => epk_{k+1}。<br>
 ）

Once the sender has all the required information above, it can construct the
packet. Constructing a packet routed over `r` hops requires `r` 32-byte
ephemeral public keys, `r` 32-byte shared secrets, `r` 32-byte blinding factors,
and `r` variable length `hop_payload` payloads.
The construction returns a single 1366-byte packet along with the first receiving peer's address.

送信者が上記の必要な情報をすべて取得するとパケットを構築できる。
r個のhops経由でルーティングされるパケットを構築するには、
r個の32バイトephemeral public keys（XXX: epk_1-epk_r）、
r個の32バイトshared secrets（XXX: ss_1-ss_r）、
r個の32バイトblinding factors（XXX: bf_1-bf_r）、
およびr個の可変長hop_payloadペイロードが必要である。
この構成では、最初の受信ピアのアドレスとともに1つの1366バイトのパケットが返される。
（XXX: onion_packet）

The packet construction is performed in the reverse order of the route, i.e.
the last hop's operations are applied first.

パケット構成は、経路の逆順で実行される。
すなわち、最後のhopの操作が最初に適用される。

The packet is initialized with 1366 `0x00`-bytes.

パケットは1366 0x00-bytesで初期化される。

A filler is generated (see [Filler Generation](#filler-generation)) using the shared secret.

shared secretを使用して、fillerが生成される（Filler Generation参照）。

For each hop in the route, in reverse order, the sender applies the
following operations:

ルート内の各hopに対して、逆順で、送信側は次の操作を適用する。

 - The _rho_-key and _mu_-key are generated using the hop's shared secret.
 - `shift_size` is defined as the length of the `hop_payload` plus the varint encoding of the length and the length of that HMAC. Thus if the payload length is `l` then the `shift_size` is `1 + l + 32` for `l < 253`, otherwise `3 + l + 32` due to the varint encoding of `l`.
 - The `hop_payload` field is right-shifted by `shift_size` bytes, discarding the last `shift_size`
 bytes that exceed its 1300-byte size.
 - The varint-serialized length, serialized `hop_payload` and `HMAC` are copied into the following `shift_size` bytes.
 - The _rho_-key is used to generate 1300 bytes of pseudo-random byte stream
 which is then applied, with `XOR`, to the `hop_payloads` field.
 - If this is the last hop, i.e. the first iteration, then the tail of the
 `hop_payloads` field is overwritten with the routing information `filler`.
 - The next HMAC is computed (with the _mu_-key as HMAC-key) over the
 concatenated `hop_payloads` and associated data.

（XXX: 区切り）

 - rho-keyとmu-keyはhopのshared secretを使用して生成される。
 - shift_sizeは、hop_payloadの長さに、lengthのvarint符号化とそのHMACの長さを加えたものとして定義される。
 したがって、ペイロード長がlの場合、shift_sizeはl < 253に対して1 + l + 32となる。
 それ以外の場合は、varint符号化がlであるため、3 + l + 32となる。（XXX: 一旦BOLT4を全部見直す）
 - hop_payloadフィールドはshift_sizeバイト右シフトされ、その1300バイトサイズを超える最後のshift_sizeバイトが破棄される。
 - varintにシリアル化されたlength、シリアル化されたhop_payload、およびHMACは、続くshift_sizeバイトにコピーされる。
 - rho-keyは1300バイトのpseudo-random byte streamを生成するために使用される、
 これはその後hop_payloadsフィールドとXORされる。
 - これが最後のhop、すなわち最初のイテレーションである場合、
hop_payloadsフィールドの末尾にはルーティング情報（XXX: ？）の詰め物が上書きされる。
 - 次のHMACは（mu-keyをHMAC-keyとして）hop_payloadsとassociated dataを連結したもので計算される。

The resulting final HMAC value is the HMAC that will be used by the first
receiving peer in the route.

結果の最後のHMAC値は、ルート内の最初の受信ピアによって使用されるHMACである。

The packet generation returns a serialized packet that contains the `version`
byte, the ephemeral pubkey for the first hop, the HMAC for the first hop, and
the obfuscated `hop_payloads`.

パケット生成は、versionバイト、最初のhopのephemeral pubkey（XXX: epk_1）、最初のhopのHMAC、
および難読化されたhop_payloadsを含む、シリアル化されたパケットを返す。

（XXX:<br>
origin:<br>
SHA256(pk_k * ek_k) => ss_k。<br>
SHA256(epk_k || ss_k) => bf_k。<br>
ek_k * bf_k => ek_{k+1}。<br>
ek_{k+1} * G => epk_{k+1}。<br>
hop:<br>
SHA256(k_k * epk_k) => ss_k。<br>
SHA256(epk_k || ss_k) => bf_k。<br>
epk_k+1 * bf_k => epk_{k+1}。<br>
）

The following Go code is an example implementation of the packet construction:

次のGoコードは、パケット構成の実装例である：
（XXX: TODO: まだ完全に理解していない）

```Go
func NewOnionPacket(paymentPath []*btcec.PublicKey, sessionKey *btcec.PrivateKey,
	hopsData []HopData, assocData []byte) (*OnionPacket, error) {

//（XXX: paymentPath: pk_1-pk_k）
//（XXX: sessionKey: ek_1）
//（XXX: hopsData: per_hop_1-per_hop_k）
//（XXX: assocData: payment_hash）

//（XXX: 長さと領域の確保）
	numHops := len(paymentPath)
	hopSharedSecrets := make([][sha256.Size]byte, numHops)

	// Initialize ephemeral key for the first hop to the session key.
	var ephemeralKey big.Int
	ephemeralKey.Set(sessionKey.D)

	for i := 0; i < numHops; i++ {
//（XXX: SHA256(pk_k * ek_k) => ss_k）
		// Perform ECDH and hash the result.
		ecdhResult := scalarMult(paymentPath[i], ephemeralKey)
		hopSharedSecrets[i] = sha256.Sum256(ecdhResult.SerializeCompressed())

//（XXX: ek_k * G => epk_k）
		// Derive ephemeral public key from private key.
		ephemeralPrivKey := btcec.PrivKeyFromBytes(btcec.S256(), ephemeralKey.Bytes())
		ephemeralPubKey := ephemeralPrivKey.PubKey()

//（XXX: SHA256(epk_k || ss_k) => bf_k）
		// Compute blinding factor.
		sha := sha256.New()
		sha.Write(ephemeralPubKey.SerializeCompressed())
		sha.Write(hopSharedSecrets[i])

		var blindingFactor big.Int
		blindingFactor.SetBytes(sha.Sum(nil))

//（XXX: ek_k * bf_k => ek_{k+1}）
		// Blind ephemeral key for next hop.
		ephemeralKey.Mul(&ephemeralKey, &blindingFactor)
		ephemeralKey.Mod(&ephemeralKey, btcec.S256().Params().N)
	}

//（XXX: ？？？）
//（XXX: 埋める分だけ生成）
//（XXX: これは後述のgenerateFiller）
	// Generate the padding, called "filler strings" in the paper.
	filler := generateHeaderPadding("rho", numHops, hopDataSize, hopSharedSecrets)

//（XXX: routingInfoSize　== 1300 bytes、hmacSize == 32 bytes）
//（XXX: mixHeaderはhop_payloads）
	// Allocate and initialize fields to zero-filled slices
	var mixHeader [routingInfoSize]byte
	var nextHmac [hmacSize]byte

	// Compute the routing information for each hop along with a
	// MAC of the routing information using the shared key for that hop.
	for i := numHops - 1; i >= 0; i-- {
//（XXX: HMAC("rho", ss_k) => rho_key_k）
//（XXX: HMAC("mu", ss_k) => mu_key_k）
		rhoKey := generateKey("rho", hopSharedSecrets[i])
		muKey := generateKey("mu", hopSharedSecrets[i])

//（XXX: final nodeはzeros、hop_payloadsとpayment_hashのHMAC）
		hopsData[i].HMAC = nextHmac

//（XXX: the pseudo-random byte stream、numStreamBytes　== 1365）
		// Shift and obfuscate routing information
		streamBytes := generateCipherStream(rhoKey, numStreamBytes)

//（XXX: hop_payloadsを、hopDataSize == 65 bytes、シフト）
//（XXX: per_hop_kをhop_payloadsの先頭にコピー）
//（XXX: 1300 bytesだけxor）
		rightShift(mixHeader[:], hopDataSize)
		buf := &bytes.Buffer{}
		hopsData[i].Encode(buf)
		copy(mixHeader[:], buf.Bytes())
		xor(mixHeader[:], mixHeader[:], streamBytes[:routingInfoSize])

//（XXX: 最初だけ後ろをfillerで埋める？）
		// These need to be overwritten, so every node generates a correct padding
		if i == numHops-1 {
			copy(mixHeader[len(mixHeader)-len(filler):], filler)
		}

//（XXX: HMAC生成。mixHeader全部とpayment_hashから。mu_key_kで）
		packet := append(mixHeader[:], assocData...)
		nextHmac = calcMac(muKey, packet)
	}

	packet := &OnionPacket{
		Version:      0x00,
		EphemeralKey: sessionKey.PubKey(),
		RoutingInfo:  mixHeader,
		HeaderMAC:    nextHmac,
	}
	return packet, nil
}
```

# Packet Forwarding

This specification is limited to `version` `0` packets; the structure
of future versions may change.

この仕様はversion 0パケットに限定されている；将来のバージョンの構造が変更される可能性がある。

Upon receiving a packet, a processing node compares the version byte of the
packet with its own supported versions and aborts the connection if the packet
specifies a version number that it doesn't support.
For packets with supported version numbers, the processing node first parses the
packet into its individual fields.

パケットを受信すると、processing nodeは、パケットのversionバイトをそれ自身のサポートされているversionsと比較し、
パケットがサポートしていないversion番号を指定している場合、接続を打ち切る。
サポートされているversion番号を持つパケットの場合、processing nodeは最初にパケットを個々のフィールドに分析する。

Next, the processing node computes the shared secret using the private key
corresponding to its own public key and the ephemeral key from the packet, as
described in [Shared Secret](#shared-secret).

次に、processing nodeは、自身のpublic keyに対応するprivate keyと、
パケットからのephemeral keyを使用してshared secretを計算する、
Shared Secretで説明されるように。
（XXX: SHA256(k_k * epk_k) => ss_k）

The above requirements prevent any hop along the route from retrying a payment
multiple times, in an attempt to track a payment's progress via traffic
analysis. Note that disabling such probing could be accomplished using a log of
previous shared secrets or HMACs, which could be forgotten once the HTLC would
not be accepted anyway (i.e. after `outgoing_cltv_value` has passed). Such a log
may use a probabilistic data structure, but it MUST rate-limit commitments as
necessary, in order to constrain the worst-case storage requirements or false
positives of this log.

上記の要件は、ルートに沿ったどんなhopsも、トラフィック分析を介して支払いの進捗を追跡するために、
支払いを複数回再試行するのを防ぎます。
そのような探査を無効にすることは、以前のshared secretsまたはHMACのログを使用して達成できることに注意すること。
それらはHTLCが受け入れられないと忘れる可能性がある（つまり、outgoing_cltv_valueが経過した後に）。
（XXX: もうタイムアウトしているということか？）
そのようなログは、probabilistic data structureを使用するかもしれないが、
このログのワーストケースのストレージ要件または偽陽性を制限するために、必要に応じてコミットメントをレート制限する必要がある。
（XXX: shard secretsかHMACのログを取っておかないと探査を防ぐことはできないということか？）

Next, the processing node uses the shared secret to compute a _mu_-key, which it
in turn uses to compute the HMAC of the `hop_payloads`. The resulting HMAC is then
compared against the packet's HMAC.

次に、processing nodeは、shared secretを使用してmu-keyを計算し、
それは順番に（XXX: 次に？）hop_payloadsのHMACの計算に使用する。
得られたHMACはそれから、パケットのHMACと比較される。

Comparison of the computed HMAC and the packet's HMAC MUST be
time-constant to avoid information leaks.

計算されたHMACとパケットのHMACとの比較は、情報漏洩を避けるために時定数でなければならない。
（XXX: ？）

At this point, the processing node can generate a _rho_-key and a _gamma_-key.

この時点で、processing nodeはrho-keyとgamma-keyを生成することができる。
（XXX: gammaは使わない？ammag？）

The routing information is then deobfuscated, and the information about the
next hop is extracted.
To do so, the processing node copies the `hop_payloads` field, appends 1300 `0x00`-bytes,
generates `2*1300` pseudo-random bytes (using the _rho_-key), and applies the result, using `XOR`, to the copy of the `hop_payloads`.
The first few bytes correspond to the varint-encoded length `l` of the `hop_payload`, followed by `l` bytes of the resulting routing information become the `hop_payload`, and the 32 byte HMAC.
The next 1300 bytes are the `hop_payloads` for the outgoing packet.

その後、ルーティング情報は逆難読化され、次のhopに関する情報が抽出される。
これを行うには、processing nodeはhop_payloadsフィールドをコピーし、1300個の0x00-bytesを追加し、
（XXX: 0x00を追加する意味は、次のhop_payloadsをそのまま作成するため）
（rho-keyを使用して）2*1300個のpseudo-random bytesを生成し、そしてその結果をhop_payloadsのコピーにXORを使用して適用する。
最初の数バイトは、hop_payloadの可変長符号化された長さlに対応し、
その後に、結果として得られるルーティング情報のlバイトがhop_payloadとなり、
32バイトのHMACとなる。
次の1300バイトは出力パケット用のhop_payloadsである。

A special `HMAC` value of 32 `0x00`-bytes indicates that the currently processing hop is the intended recipient and that the packet should not be forwarded.

特別なHMAC値の32個の0x00-bytesは、現在のprocessing hopが意図された受信者であり、
パケットが転送されるべきでないことを示す。

If the HMAC does not indicate route termination, and if the next hop is a peer of the
processing node; then the new packet is assembled. Packet assembly is accomplished
by blinding the ephemeral key with the processing node's public key, along with the
shared secret, and by serializing the `hop_payloads`.
The resulting packet is then forwarded to the addressed peer.

HMACが経路終端を示さず（XXX: all zeroではない）、次のhopがprocessing nodeのpeerである場合、
新しいパケットが組み立てられる。
パケットの組み立ては、ephemeral keyをprocessing nodeのpublic keyとshared secretでブラインドし、
hops_payloadsをシリアライズすることによって達成される。
（XXX: ？）
結果として得られたパケットは、次に、アドレス指定されたpeerに転送される。
（XXX:<br>
SHA256(epk_k || ss_k) => bf_k。<br>
ek_k * bf_k => ek_{k+1}。<br>
）

## Requirements

The processing node:
  - if the ephemeral public key is NOT on the `secp256k1` curve:
    - MUST abort processing the packet.
    - MUST report a route failure to the origin node.
  - if the packet has previously been forwarded or locally redeemed, i.e. the
  packet contains duplicate routing information to a previously received packet:
    - if preimage is known:
      - MAY immediately redeem the HTLC using the preimage.
    - otherwise:
      - MUST abort processing and report a route failure.
  - if the computed HMAC and the packet's HMAC differ:
    - MUST abort processing.
    - MUST report a route failure.
  - if the `realm` is unknown:
    - MUST drop the packet.
    - MUST signal a route failure.
  - MUST address the packet to another peer that is its direct neighbor.
  - if the processing node does not have a peer with the matching address:
    - MUST drop the packet.
    - MUST signal a route failure.

processing node：
  - ephemeral public keyがsecp256k1 curve上にない場合：
    - パケットの処理を中断しなければならない。
    - origin nodeに経路障害を報告しなければならない。
  - パケットが以前に転送されたか、またはローカルに償還された場合、
  すなわち、パケットが前に受信したパケットに重複したルーティング情報を含む場合：
    - preimageがわかっている場合：
      - preimageを使用してHTLCをすぐに償還して良い。
    - そうでなければ：
      - 処理を中断し、経路障害を報告しなければならない。
      （XXX: Base AMPでここの条件が変わるか？）
  - 計算されたHMACとパケットのHMACが異なる場合：
    - 処理を中断しなければならない。
    - 経路障害を報告しなければならない。
    （XXX: onionのhashを返すので「report」という表現か）
  - realmが不明の場合：
    - パケットをドロップしなければならない。
    - 経路障害を通知しなければならない。
  - その直接の隣人である別のピアにパケットをアドレスしなければなりません。
  - processing nodeが一致するアドレスを持つピアを持たない場合：
    - パケットをドロップしなければならない。
    - 経路障害を通知しなければならない。

# Filler Generation

Upon receiving a packet, the processing node extracts the information destined
for it from the route information and the per-hop payload.
The extraction is done by deobfuscating and left-shifting the field.
This would make the field shorter at each hop, allowing an attacker to deduce the
route length. For this reason, the field is pre-padded before forwarding.
Since the padding is part of the HMAC, the origin node will have to pre-generate an
identical padding (to that which each hop will generate) in order to compute the
HMACs correctly for each hop.
The filler is also used to pad the field-length, in the case that the selected
route is shorter than 1300 bytes.

（XXX: Filler、詰め物）
パケットを受信すると、processing nodeは、
routing informationとper-hopのペイロードから、それに向けられた情報を抽出する。
抽出は、フィールドの逆難読化と左シフトによって実行される。
これにより、各hopでフィールドが短くなり、攻撃者がルートの長さを推測できるようになる。
このため、フィールドは転送前に事前にパディングされる。
パディングはHMACの一部であるので、origin nodeは、各hopに対してHMACを正しく計算するために、
（各hopが生成する）パディングを前もって生成しなければならない。
詰め物は、選択されたルートが1300バイトよりも短い場合にも、フィールド長を埋めるためにも使用される。

Before deobfuscating the `hop_payloads`, the processing node pads it with 1300
`0x00`-bytes, such that the total length is `2*1300`.
It then generates the pseudo-random byte stream, of matching length, and applies
it with `XOR` to the `hop_payloads`.
This deobfuscates the information destined for it, while simultaneously
obfuscating the added `0x00`-bytes at the end.

hop_payloadsを逆難読化する前に、processing nodeは1300個の0x00-bytesでそれをパディングし、
その合計の長さは2*1300になる。
次に、一致する長さのpseudo-random byte streamを生成し、それをhop_payloadsにXORで適用する。
これにより、最後に追加された0x00-bytesを難読化すると同時に、それに向けられた情報が逆難読化される。

In order to compute the correct HMAC, the origin node has to pre-generate the
`hop_payloads` for each hop, including the incrementally obfuscated padding added
by each hop. This incrementally obfuscated padding is referred to as the
`filler`.

正しいHMACを計算するために、origin nodeは、
各hopによって追加された段階的に難読化されたパディングを含む、
各hopに対するhop_payloadsを前もって生成しなければならない。
この段階的に難読化されたパディングをfillerと呼ばれる。

The following example code shows how the filler is generated in Go:

次のコード例は、詰め物がGoでどのように生成されるかを示している：

```Go
func generateFiller(key string, numHops int, hopSize int, sharedSecrets [][sharedSecretSize]byte) []byte {
	fillerSize := uint((numMaxHops + 1) * hopSize)
	filler := make([]byte, fillerSize)

  //（XXX: key == "rho"）
  //（XXX: numHops 1-20）
  //（XXX: hopSize == 65 bytes）
  //（XXX: sharedSecrets、numHops個）
  //（XXX: fillerSize == 1365）
  //（XXX: filler 1365 bytes）

	// The last hop does not obfuscate, it's not forwarding anymore.
	for i := 0; i < numHops-1; i++ {

		// Left-shift the field
		copy(filler[:], filler[hopSize:])

		// Zero-fill the last hop
		copy(filler[len(filler)-hopSize:], bytes.Repeat([]byte{0x00}, hopSize))

		// Generate pseudo-random byte stream
		streamKey := generateKey(key, sharedSecrets[i])
		streamBytes := generateCipherStream(streamKey, fillerSize)

		// Obfuscate
		xor(filler, filler, streamBytes)
	}

  //（XXX: numHopsが1だったらhopSizeを21飛ばして0個）
  //（XXX: numHopsが20だったらhopSizeを2つ飛ばして19個）
	// Cut filler down to the correct length (numHops+1)*hopSize
	// bytes will be prepended by the packet generation.
	return filler[(numMaxHops-numHops+2)*hopSize:]
}
```

Note that this example implementation is for demonstration purposes only; the
`filler` can be generated much more efficiently.
The last hop need not obfuscate the `filler`, since it won't forward the packet
any further and thus need not extract an HMAC either.

この実装例はデモンストレーションの目的でのみ使用されていることに注意すること。
詰め物はより効率的に生成することができる。
最後のホップは、パケットをそれ以上転送しないので、HMACを抽出する必要もないので、詰め物を難読化する必要はない。

# Returning Errors

The onion routing protocol includes a simple mechanism for returning encrypted
error messages to the origin node.
The returned error messages may be failures reported by any hop, including the
final node.
The format of the forward packet is not usable for the return path, since no hop
besides the origin has access to the information required for its generation.
Note that these error messages are not reliable, as they are not placed on-chain
due to the possibility of hop failure.

onion routing protocolは、暗号化されたエラーメッセージをorigin nodeに返すための単純なメカニズムを含む。
返されるエラーメッセージは、final nodeを含む任意のhopによって報告された障害である可能性がある。
転送パケットのフォーマットは、origin以外のどのhopも、その生成に必要な情報へのアクセスを有していないので、戻りパスには使用できない。
これらのエラーメッセージは、hop障害の可能性のためにチェーン上に配置されないため、信頼性がないことに注意。
（XXX: ？）

Intermediate hops store the shared secret from the forward path and reuse it to
obfuscate any corresponding return packet during each hop.
In addition, each node locally stores data regarding its own sending peer in the
route, so it knows where to return-forward any eventual return packets.
The node generating the error message (_erring node_) builds a return packet
consisting of the following fields:

中間hopsは、順方向パスからのshared secretを格納し（XXX: 格納しておかないといけない）、
それを再利用して、各hop中に対応する戻りパケットを難読化する。（XXX: update_fail_htlcのreason）
さらに、各nodeはルート内に自身の送信ピアに関するデータをローカルに格納するため、最終的な戻りパケットをどこに返すかを知っている。
エラーメッセージを生成しているノード（erring node）は、次のフィールドで構成される戻りパケットを生成する。

1. data:
   * [`32*byte`:`hmac`]
   * [`u16`:`failure_len`]
   * [`failure_len*byte`:`failuremsg`]
   * [`u16`:`pad_len`]
   * [`pad_len*byte`:`pad`]

Where `hmac` is an HMAC authenticating the remainder of the packet, with a key
generated using the above process, with key type `um`, `failuremsg` as defined
below, and `pad` as the extra bytes used to conceal length.

ここでhmacは、上記のプロセスを使用して生成された鍵を用いて、パケットの残りの部分を認証するHMACであり、
キータイプumとともに、
failuremsgは以下に定義されるように、
及びpadは長さを隠すために使用される余分なバイトとする。

（XXX: ammagでa pseudo-random stream）
（XXX: umでHMAC）

The erring node then generates a new key, using the key type `ammag`.
This key is then used to generate a pseudo-random stream, which is in turn
applied to the packet using `XOR`.

erring nodeはそのとき、キータイプammagを使用して新しいキーを生成する。
このキーは次に、pseudo-random streamを生成するために使用される。
これは順番に、パケットにXORで適用される。

The obfuscation step is repeated by every hop along the return path.
Upon receiving a return packet, each hop generates its `ammag`, generates the
pseudo-random byte stream, and applies the result to the return packet before
return-forwarding it.

難読化ステップは、戻りパスに沿ったすべてのhopによって繰り返される。
リターンパケットを受信すると、各hopはそのammagを生成し、pseudo-random byte streamを生成し、
その結果を戻りパケットに適用したあと、転送して戻す。

The origin node is able to detect that it's the intended final recipient of the
return message, because of course, it was the originator of the corresponding
forward packet.
When an origin node receives an error message matching a transfer it initiated
(i.e. it cannot return-forward the error any further) it generates the `ammag`
and `um` keys for each hop in the route.
It then iteratively decrypts the error message, using each hop's `ammag`
key, and computes the HMAC, using each hop's `um` key.
The origin node can detect the sender of the error message by matching the
`hmac` field with the computed HMAC.

origin nodeは、返信メッセージの意図された最終受信者であることを検出することができる、
もちろん、対応する転送パケットの発信者であったからである。
origin nodeが開始した転送に一致するエラーメッセージを受信すると（つまり、それ以上エラーを戻すことはできない）、
ルート内の各hopのammagとumキーを生成する。
次に、各hopのammagキーを使用してエラーメッセージを反復的に復号し、各hopのumキーを使用してHMACを計算する。
origin nodeは、hmacフィールドを計算されたHMACと照合することによってエラーメッセージの送信者を検出することができる。
（XXX: 一致したら完了）

The association between the forward and return packets is handled outside of
this onion routing protocol, e.g. via association with an HTLC in a payment
channel.

転送パケットと返送パケットとの間の関連付けは、このonion routing protocolの外部で処理される、
例えば、payment channel内のHTLCとの関連付けを介して。

### Requirements

The _erring node_:
  - SHOULD set `pad` such that the `failure_len` plus `pad_len` is equal to 256.
    - Note: this value is 118 bytes longer than the longest currently-defined
    message.

erring node：
  - padを、failure_lenプラスpad_lenが256に等しくなるように設定すべきである。
    - 注：この値は、現在定義されている最長のメッセージよりも118バイト長くなる。
    （XXX: なぜそうした？）

The _origin node_:
  - once the return message has been decrypted:
    - SHOULD store a copy of the message.
    - SHOULD continue decrypting, until the loop has been repeated 20 times.
    - SHOULD use constant `ammag` and `um` keys to obfuscate the route length.

origin node：
（XXX: errorのoriginではなくadd_htlcのorigin）
  - 返信メッセージが復号されると：
    - メッセージのコピーを格納すべきである。
    - ループが20回繰り返されるまで、復号化を継続すべきである。
    - ルートの長さを難読化するための定数ammagとumキーを使うべきである。（XXX: ？）

## Failure Messages

The failure message encapsulated in `failuremsg` has an identical format as
a normal message: a 2-byte type `failure_code` followed by data applicable
to that type. Below is a list of the currently supported `failure_code`
values, followed by their use case requirements.

failuremsgにカプセル化された障害メッセージは、通常のメッセージと同じフォーマットである.
2バイトのタイプfailure_codeに続いて、そのタイプに適用可能なデータが続く。
以下は、現在サポートされているfailure_codeの値のリストとそのユースケースの要件である。

Notice that the `failure_code`s are not of the same type as other message types,
defined in other BOLTs, as they are not sent directly on the transport layer
but are instead wrapped inside return packets.
The numeric values for the `failure_code` may therefore reuse values, that are
also assigned to other message types, without any danger of causing collisions.

failure_codeは他のBOLTで定義されている他のメッセージタイプと同じタイプではないことに注意すること、
トランスポートレイヤーで直接送信されるのではなく、戻りパケットでラップされる。
したがって、failure_codeの数値は、衝突を引き起こす危険なしに、
他のメッセージタイプにも割り当てられた値を再利用する可能性がある。

The top byte of `failure_code` can be read as a set of flags:
* 0x8000 (BADONION): unparsable onion encrypted by sending peer
* 0x4000 (PERM): permanent failure (otherwise transient)
* 0x2000 (NODE): node failure (otherwise channel)
* 0x1000 (UPDATE): new channel update enclosed

failure_codeの一番上のバイトは、一連のフラグとして読み取ることができる。
* 0x8000（BADONION）：送信側ピアによって暗号化された解析不能なonion
* 0x4000（PERM）：永続的な障害（それ以外の場合は一時的な障害）
* 0x2000（NODE）：ノード障害（それ以外の場合はチャネル障害）
* 0x1000（UPDATE）：新しいchannel_updateが封入されている

Please note that the `channel_update` field is mandatory in messages whose
`failure_code` includes the `UPDATE` flag.

channel_updateフィールドは、
UPDATEフラグを含むfailure_codeのメッセージでは必須であることに留意すること。

The following `failure_code`s are defined:

次failure_codeが定義されている。

1. type: PERM|1 (`invalid_realm`)

The `realm` byte was not understood by the processing node.

realmバイトは、processing nodeによって理解されなかった。

1. type: NODE|2 (`temporary_node_failure`)

General temporary failure of the processing node.

processing nodeの一般的な一時的な障害。

1. type: PERM|NODE|2 (`permanent_node_failure`)

General permanent failure of the processing node.

processing nodeの一般的な永続的な障害。

1. type: PERM|NODE|3 (`required_node_feature_missing`)

The processing node has a required feature which was not in this onion.

processing nodeには、このonionにない機能を必要とする。

1. type: BADONION|PERM|4 (`invalid_onion_version`)
2. data:
   * [`sha256`:`sha256_of_onion`]

The `version` byte was not understood by the processing node.

versionバイトは、processing nodeによって理解されなかった。

1. type: BADONION|PERM|5 (`invalid_onion_hmac`)
2. data:
   * [`sha256`:`sha256_of_onion`]

The HMAC of the onion was incorrect when it reached the processing node.

onionのHMACは、processing nodeに到達したとき正しくなかった。

1. type: BADONION|PERM|6 (`invalid_onion_key`)
2. data:
   * [`sha256`:`sha256_of_onion`]

The ephemeral key was unparsable by the processing node.

ephemeral keyは、processing nodeによって解析できなかった。

1. type: UPDATE|7 (`temporary_channel_failure`)
2. data:
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

The channel from the processing node was unable to handle this HTLC,
but may be able to handle it, or others, later.

processing nodeからのチャネルは、このHTLCを処理することができなかったが、
後で処理することができるかもしれない。

1. type: PERM|8 (`permanent_channel_failure`)

The channel from the processing node is unable to handle any HTLCs.

processing nodeからのチャネルは、どのHTLCも処理できない。

1. type: PERM|9 (`required_channel_feature_missing`)

The channel from the processing node requires features not present in
the onion.

processing nodeからのチャネルは、このonionにない機能を必要とする。

1. type: PERM|10 (`unknown_next_peer`)

The onion specified a `short_channel_id` which doesn't match any
leading from the processing node.

onionが指定するshort_channel_idは、processing nodeから導かれるものと一致しない。

1. type: UPDATE|11 (`amount_below_minimum`)
2. data:
   * [`u64`:`htlc_msat`]
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

The HTLC amount was below the `htlc_minimum_msat` of the channel from
the processing node.

HTLCの量は、processing nodeからのチャネルのhtlc_minimum_msatよりも少なかった。

1. type: UPDATE|12 (`fee_insufficient`)
2. data:
   * [`u64`:`htlc_msat`]
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

The fee amount was below that required by the channel from the
processing node.

feeは、processing nodeからのチャネルによって要求されたものよりも少なかった。

1. type: UPDATE|13 (`incorrect_cltv_expiry`)
2. data:
   * [`u32`:`cltv_expiry`]
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

The `cltv_expiry` does not comply with the `cltv_expiry_delta` required by
the channel from the processing node: it does not satisfy the following
requirement:

cltv_expiryは、processing nodeからのチャネルによって要求されたcltv_expiry_deltaに準拠していない：
次の要件を満たしていない。

        cltv_expiry - cltv_expiry_delta >= outgoing_cltv_value

1. type: UPDATE|14 (`expiry_too_soon`)
2. data:
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

The CLTV expiry is too close to the current block height for safe
handling by the processing node.

processing nodeによる安全な処理のために、CLTV expiryが現在のブロックの高さに近すぎる。

1. type: PERM|15 (`incorrect_or_unknown_payment_details`)
2. data:
   * [`u64`:`htlc_msat`]

The `payment_hash` is unknown to the final node or the amount for that
`payment_hash` is incorrect.

payment_hashがfinal nodeに知られていないか、そのpayment_hashのための金額が正しくない。

Note: Originally PERM|16 (`incorrect_payment_amount`) was
used to differentiate incorrect final amount from unknown payment
hash. Sadly, sending this response allows for probing attacks whereby a node
which receives an HTLC for forwarding can check guesses as to its final
destination by sending payments with the same hash but much lower values to
potential destinations and check the response.

注意： もともとPERM|16（incorrect_payment_amount）は、
不正確な最終金額と未知のpayment hashを区別するために使用されていた。
残念なことに、この応答を送ることでプロービング攻撃が可能になり、
それによって転送のためにHTLCを受け取るノードは、
同じハッシュだがもっと低い値で支払いを潜在的な宛先に送ってレスポンスをチェックすることによって、
その最終宛先に関しての推測をチェックすることができる。
（XXX: このような攻撃を行うのはfinal nodeのひとつまえのnodeであろう。
しかし、この対応でそれを防ぐことができるのであろうか？
通常未知のpayment_hashエラーも不正確な最終金額エラーも起きない。
そう考えると、意図的に最終金額を少なくしたときにincorrect_or_unknown_payment_detailsが
発生した場合、最終金額が不正確であったためにこのエラーが起きたことは自明であり、
そうであればプロービング攻撃は可能であると思う）。

1. type: 17 (`final_expiry_too_soon`)

The CLTV expiry is too close to the current block height for safe
handling by the final node.

final nodeによる安全な処理のために、CLTV expiryは現在のブロックの高さに近すぎる。

1. type: 18 (`final_incorrect_cltv_expiry`)
2. data:
   * [`u32`:`cltv_expiry`]

The CLTV expiry in the HTLC doesn't match the value in the onion.

HTLCのCLTV expiryはonionの値と一致しない。
（XXX: イコールではなく条件に一致しないということか？）

1. type: 19 (`final_incorrect_htlc_amount`)
2. data:
   * [`u64`:`incoming_htlc_amt`]

The amount in the HTLC doesn't match the value in the onion.

HTLCの量がonionの値と一致しない。

1. type: UPDATE|20 (`channel_disabled`)
2. data:
   * [`u16`: `flags`]
   * [`u16`:`len`]
   * [`len*byte`:`channel_update`]

The channel from the processing node has been disabled.

processing nodeからのチャネルは無効になっている。

1. type: 21 (`expiry_too_far`)

The CLTV expiry in the HTLC is too far in the future.

HTLCにおけるCLTV expiryは、将来的には遠すぎる。

### Requirements

An _erring node_:
  - MUST select one of the above error codes when creating an error message.
  - MUST include the appropriate data for that particular error type.
  - if there is more than one error:
    - SHOULD select the first error it encounters from the list above.

erring node：
  - エラーメッセージを作成するときは、上記のエラーコードの1つを選択しなければならない。
  - その特定のエラータイプに適切なデータを含めなけらばならない。
  - 複数のエラーがある場合：
    - 上記のリストから遭遇する最初のエラーを選択すべきである。

Any _erring node_ MAY:
  - if the `realm` byte is unknown:
    - return an `invalid_realm` error.
  - if an otherwise unspecified transient error occurs for the entire node:
    - return a `temporary_node_failure` error.
  - if an otherwise unspecified permanent error occurs for the entire node:
    - return a `permanent_node_failure` error.
  - if a node has requirements advertised in its `node_announcement` `features`,
  which were NOT included in the onion:
    - return a `required_node_feature_missing` error.

erring nodeはしてよい：
  - realmバイトが不明の場合：
    - invalid_realmエラーを返す。
  - そうでなければ、指定されていない一時的なエラーがノード全体に対して発生した場合：
    - temporary_node_failureエラーを返す。
  - そうでなければ、指定されていない永続的なエラーがノード全体に対して発生した場合：
    - permanent_node_failureエラーを返す。
  - ノードが、それのnode_announcementのfeaturesで広告されている要件を持つ場合に、
  それらがonionに含まれていなかった場合：
    - required_node_feature_missingエラーを返す。

A _forwarding node_ MAY, but a _final node_ MUST NOT:
  - if the onion `version` byte is unknown:
    - return an `invalid_onion_version` error.
  - if the onion HMAC is incorrect:
    - return an `invalid_onion_hmac` error.
  - if the ephemeral key in the onion is unparsable:
    - return an `invalid_onion_key` error.
  - if during forwarding to its receiving peer, an otherwise unspecified,
  transient error occurs in the outgoing channel (e.g. channel capacity reached,
  too many in-flight HTLCs, etc.):
    - return a `temporary_channel_failure` error.
  - if an otherwise unspecified, permanent error occurs during forwarding to its
  receiving peer (e.g. channel recently closed):
    - return a `permanent_channel_failure` error.
  - if the outgoing channel has requirements advertised in its
  `channel_announcement`'s `features`, which were NOT included in the onion:
    - return a `required_channel_feature_missing` error.
  - if the receiving peer specified by the onion is NOT known:
    - return an `unknown_next_peer` error.
  - if the HTLC amount is less than the currently specified minimum amount:
    - report the amount of the outgoing HTLC and the current channel setting for
    the outgoing channel.
    - return an `amount_below_minimum` error.
  - if the HTLC does NOT pay a sufficient fee:
    - report the amount of the incoming HTLC and the current channel setting for
    the outgoing channel.
    - return a `fee_insufficient` error.
 -  if the incoming `cltv_expiry` minus the `outgoing_cltv_value` is below the
    `cltv_expiry_delta` for the outgoing channel:
    - report the `cltv_expiry` of the outgoing HTLC and the current channel setting for the outgoing
    channel.
    - return an `incorrect_cltv_expiry` error.
  - if the `cltv_expiry` is unreasonably near the present:
    - report the current channel setting for the outgoing channel.
    - return an `expiry_too_soon` error.
  - if the `cltv_expiry` is unreasonably far in the future:
    - return an `expiry_too_far` error.
  - if the channel is disabled:
    - report the current channel setting for the outgoing channel.
    - return a `channel_disabled` error.

forwarding nodeは良いが、final nodeはだめである：
  - onionのversionバイトが不明な場合：
    - invalid_onion_versionエラーを返す。
  - onionのHMACが正しくない場合：
    - invalid_onion_hmacエラーを返す。
  - onionのephemeral keyが解析できない場合：
    - invalid_onion_keyエラーを返す。
  - receiving peerへの転送中に、別に指定がなければ、一時的なエラーが出力チャネルで発生した場合
  （例えば、チャネル容量に到達、途中のHTLCsが多すぎるなど）：
    - temporary_channel_failureエラーを返す。
  - そうでなければ、不特定な永続的なエラーがreceiving peer転送中に発生した場合（例えば最近閉じられたチャネル）：
    - permanent_channel_failureエラーを返す。
  - 出力チャネルは、そのchannel_announcementのfeaturesで宣伝された要件があり、それらがonionに含まれない場合：
    - required_channel_feature_missingエラーを返す。
  - onionによって指定されたreceiving peerが知られていない場合：
    - unknown_next_peerエラーを返す。
  - HTLCの量が現在指定されている最小量よりも少ない場合：
    - 出力HTLCの量と出力チャネルの現在のチャネル設定を報告する。
    - amount_below_minimumエラーを返す。
  - HTLCが十分なfeeを支払っていない場合：
    - 入力HTLCの量と出力チャネルの現在のチャネル設定を報告する。
    - fee_insufficientエラーを返す。
  - 入力のcltv_expiryマイナスoutgoing_cltv_valueが、
  出力チャネルのためのcltv_expiry_deltaを下回っている：
    - cltv_expiryと出力HTLCと現在の出力チャネルのチャネル設定を報告する。
    - incorrect_cltv_expiryエラーを返す。
  - cltv_expiryが現在に不当に近い場合：
    - 出力チャネルの現在のチャネル設定を報告する。
    - expiry_too_soonエラーを返す。
  - cltv_expiryが将来に不当に遠い場合：
    - expiry_too_farエラーを返す。
  - チャネルが無効になっている場合：
    - 出力チャネルの現在のチャネル設定を報告する。
    - channel_disabledエラーを返す。

An _intermediate hop_ MUST NOT, but the _final node_:
  - if the payment hash has already been paid:
    - MAY treat the payment hash as unknown.
    - MAY succeed in accepting the HTLC.
  - if the amount paid is less than the amount expected:
    - MUST fail the HTLC.
    - MUST return an `incorrect_or_unknown_payment_details` error.
  - if the payment hash is unknown:
    - MUST fail the HTLC.
    - MUST return an `incorrect_or_unknown_payment_details` error.
  - if the amount paid is more than twice the amount expected:
    - SHOULD fail the HTLC.
    - SHOULD return an `incorrect_or_unknown_payment_details` error.
      - Note: this allows the origin node to reduce information leakage by
      altering the amount while not allowing for accidental gross overpayment.
  - if the `cltv_expiry` value is unreasonably near the present:
    - MUST fail the HTLC.
    - MUST return a `final_expiry_too_soon` error.
  - if the `outgoing_cltv_value` does NOT correspond with the `cltv_expiry` from
  the final node's HTLC:
    - MUST return `final_incorrect_cltv_expiry` error.
  - if the `amt_to_forward` is greater than the `incoming_htlc_amt` from the
  final node's HTLC:
    - MUST return a `final_incorrect_htlc_amount` error.

intermediate hopはだめだが、final nodeはよい：
（XXX: TODO: この前にforwarding nodeと言っているのにintermediate hop。
どの表現に統一すべきか？）
  - payment hashがすでに支払済みの場合：
    - payment hashを未知として扱ってよい。
    - HTLCを受け入れることに成功してもよい。
  - 支払い金額が期待される額を下回る場合：
    - HTLCに失敗しなければならない。
    - incorrect_or_unknown_payment_detailsエラーを返さなければならない。
  - 支払いハッシュが不明な場合：
    - HTLCに失敗しなければならない。
    - incorrect_or_unknown_payment_detailsエラーを返さなければならない。
  - 支払い額が予想額の2倍を超える場合：
    - HTLCは失敗するべきである。
    - incorrect_payment_amountエラーを返すべきである。
      - 注：これにより、origin nodeは、偶発的な総額超過を許さずに、量を変更することによる情報漏洩を減らすことができる。
      （XXX: 請求額ちょうどを払わないことにより支払い先を特定しにくく、
      また誤って多すぎる量を払い過ぎないようにする。
      量がちょうどだとfinal nodeの１つ前のnodeにfinal nodeが漏洩する可能性が高くなる。）
  - cltv_expiryの値が不当に現在に近い場合：
    - HTLCに失敗しなければならない。
    - final_expiry_too_soonエラーを返さなければならない。
  - outgoing_cltv_valueが、final nodeのHTLCのcltv_expiryと一致しない場合：
    - final_incorrect_cltv_expiryエラーを返さなければならない。
  - amt_to_forwardが、final nodeのHTLCのincoming_htlc_amtよりも大きい場合：
    - final_incorrect_htlc_amountエラーを返さなければならない。

## Receiving Failure Codes

### Requirements

The _origin node_:
  - MUST ignore any extra bytes in `failuremsg`.
  - if the _final node_ is returning the error:
    - if the PERM bit is set:
      - SHOULD fail the payment.
    - otherwise:
      - if the error code is understood and valid:
        - MAY retry the payment. In particular, `final_expiry_too_soon` can
        occur if the block height has changed since sending, and in this case
        `temporary_node_failure` could resolve within a few seconds.
  - otherwise, an _intermediate hop_ is returning the error:
    - if the NODE bit is set:
      - SHOULD remove all channels connected with the erring node from
      consideration.
    - if the PERM bit is NOT set:
      - SHOULD restore the channels as it receives new `channel_update`s.
    - otherwise:
      - if UPDATE is set, AND the `channel_update` is valid and more recent
      than the `channel_update` used to send the payment:
        - if `channel_update` should NOT have caused the failure:
          - MAY treat the `channel_update` as invalid.
        - otherwise:
          - SHOULD apply the `channel_update`.
        - MAY queue the `channel_update` for broadcast.
      - otherwise:
        - SHOULD eliminate the channel outgoing from the erring node from
        consideration.
        - if the PERM bit is NOT set:
          - SHOULD restore the channel as it receives new `channel_update`s.
    - SHOULD then retry routing and sending the payment.
  - MAY use the data specified in the various failure types for debugging
  purposes.

origin node：
  - failuremsgの余分なバイトを無視しなければならない。
  - final nodeがエラーを返す場合：
    - PERMビットがセットされている場合：
      - 支払いに失敗するべきである。
    - そうでなければ：
      - エラーコードが理解され、有効である場合：
        - 支払いを再試行してよい。
        特に、final_expiry_too_soonはブロックの高さが送信後に変更された場合に発生する可能性があり、
        この場合 temporary_node_failureが数秒以内に解決できる可能性がある。
  - そうではなく、intermediate hopがエラーを返している：
    - NODEビットがセットされている場合：
      - 誤りのあるノードに接続されているすべてのチャネルを考慮から削除すべきである。
    - PERMビットがセットされていない場合：
      - 新しいchannel_updateを受け取ったときにチャネルを回復すべきである。
    - そうでなければ：
      - UPDATEが設定されていて、channel_updateが有効で、
      channel_updateが支払いを送信するために使用されたものより新しい場合：
        - このchannel_updateがこの失敗を引き起こすはずではない場合：
          - channel_updateを無効として扱って良い。
        - そうでなければ：
          - channel_updateを適用すべきである。
        - そのchannel_updateをブロードキャストの待ち行列に入れてもよい。
      - そうでなければ：
        - erring nodeからの出力チャネルを考慮から削除すべきである。
        - PERMビットがセットされていない場合：
          - 新しいchannel_updateを受け取ったときにチャネルを回復しなければならない。
    - ルーティングを再試行し、支払いを送信すべきである。
  - さまざまな障害タイプで指定されたデータをデバッグ目的で使用してよい。

# Test Vector

## Returning Errors

The test vectors use the following parameters:

	pubkey[0] = 0x02eec7245d6b7d2ccb30380bfbe2a3648cd7a942653f5aa340edcea1f283686619
	pubkey[1] = 0x0324653eac434488002cc06bbfb7f10fe18991e35f9fe4302dbea6d2353dc0ab1c
	pubkey[2] = 0x027f31ebc5462c1fdce1b737ecff52d37d75dea43ce11c74d25aa297165faa2007
	pubkey[3] = 0x032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991
	pubkey[4] = 0x02edabbd16b41c8371b92ef2f04c1185b4f03b6dcd52ba9b78d9d7c89c8f221145

	nhops = 5/20
	sessionkey = 0x4141414141414141414141414141414141414141414141414141414141414141
	associated data = 0x4242424242424242424242424242424242424242424242424242424242424242

The following is an in-depth trace of an example of error message creation:

	# node 4 is returning an error
	failure_message = 2002
	# creating error message
	shared_secret = b5756b9b542727dbafc6765a49488b023a725d631af688fc031217e90770c328
	payload = 0002200200fe0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
	um_key = 4da7f2923edce6c2d85987d1d9fa6d88023e6c3a9c3d20f07d3b10b61a78d646
	raw_error_packet = 4c2fc8bc08510334b6833ad9c3e79cd1b52ae59dfe5c2a4b23ead50f09f7ee0b0002200200fe0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
	# forwarding error packet
	shared_secret = b5756b9b542727dbafc6765a49488b023a725d631af688fc031217e90770c328
	 ammag_key = 2f36bb8822e1f0d04c27b7d8bb7d7dd586e032a3218b8d414afbba6f169a4d68
	stream = e9c975b07c9a374ba64fd9be3aae955e917d34d1fa33f2e90f53bbf4394713c6a8c9b16ab5f12fd45edd73c1b0c8b33002df376801ff58aaa94000bf8a86f92620f343baef38a580102395ae3abf9128d1047a0736ff9b83d456740ebbb4aeb3aa9737f18fb4afb4aa074fb26c4d702f42968888550a3bded8c05247e045b866baef0499f079fdaeef6538f31d44deafffdfd3afa2fb4ca9082b8f1c465371a9894dd8c243fb4847e004f5256b3e90e2edde4c9fb3082ddfe4d1e734cacd96ef0706bf63c9984e22dc98851bcccd1c3494351feb458c9c6af41c0044bea3c47552b1d992ae542b17a2d0bba1a096c78d169034ecb55b6e3a7263c26017f033031228833c1daefc0dedb8cf7c3e37c9c37ebfe42f3225c326e8bcfd338804c145b16e34e4
	error packet for node 4: a5e6bd0c74cb347f10cce367f949098f2457d14c046fd8a22cb96efb30b0fdcda8cb9168b50f2fd45edd73c1b0c8b33002df376801ff58aaa94000bf8a86f92620f343baef38a580102395ae3abf9128d1047a0736ff9b83d456740ebbb4aeb3aa9737f18fb4afb4aa074fb26c4d702f42968888550a3bded8c05247e045b866baef0499f079fdaeef6538f31d44deafffdfd3afa2fb4ca9082b8f1c465371a9894dd8c243fb4847e004f5256b3e90e2edde4c9fb3082ddfe4d1e734cacd96ef0706bf63c9984e22dc98851bcccd1c3494351feb458c9c6af41c0044bea3c47552b1d992ae542b17a2d0bba1a096c78d169034ecb55b6e3a7263c26017f033031228833c1daefc0dedb8cf7c3e37c9c37ebfe42f3225c326e8bcfd338804c145b16e34e4
	# forwarding error packet
	shared_secret = 21e13c2d7cfe7e18836df50872466117a295783ab8aab0e7ecc8c725503ad02d
	ammag_key = cd9ac0e09064f039fa43a31dea05f5fe5f6443d40a98be4071af4a9d704be5ad
	stream = 617ca1e4624bc3f04fece3aa5a2b615110f421ec62408d16c48ea6c1b7c33fe7084a2bd9d4652fc5068e5052bf6d0acae2176018a3d8c75f37842712913900263cff92f39f3c18aa1f4b20a93e70fc429af7b2b1967ca81a761d40582daf0eb49cef66e3d6fbca0218d3022d32e994b41c884a27c28685ef1eb14603ea80a204b2f2f474b6ad5e71c6389843e3611ebeafc62390b717ca53b3670a33c517ef28a659c251d648bf4c966a4ef187113ec9848bf110816061ca4f2f68e76ceb88bd6208376460b916fb2ddeb77a65e8f88b2e71a2cbf4ea4958041d71c17d05680c051c3676fb0dc8108e5d78fb1e2c44d79a202e9d14071d536371ad47c39a05159e8d6c41d17a1e858faaaf572623aa23a38ffc73a4114cb1ab1cd7f906c6bd4e21b29694
	error packet for node 3: c49a1ce81680f78f5f2000cda36268de34a3f0a0662f55b4e837c83a8773c22aa081bab1616a0011585323930fa5b9fae0c85770a2279ff59ec427ad1bbff9001c0cd1497004bd2a0f68b50704cf6d6a4bf3c8b6a0833399a24b3456961ba00736785112594f65b6b2d44d9f5ea4e49b5e1ec2af978cbe31c67114440ac51a62081df0ed46d4a3df295da0b0fe25c0115019f03f15ec86fabb4c852f83449e812f141a9395b3f70b766ebbd4ec2fae2b6955bd8f32684c15abfe8fd3a6261e52650e8807a92158d9f1463261a925e4bfba44bd20b166d532f0017185c3a6ac7957adefe45559e3072c8dc35abeba835a8cb01a71a15c736911126f27d46a36168ca5ef7dccd4e2886212602b181463e0dd30185c96348f9743a02aca8ec27c0b90dca270
	forwarding error packet
	shared_secret = 3a6b412548762f0dbccce5c7ae7bb8147d1caf9b5471c34120b30bc9c04891cc
	ammag_key = 1bf08df8628d452141d56adfd1b25c1530d7921c23cecfc749ac03a9b694b0d3
	stream = 6149f48b5a7e8f3d6f5d870b7a698e204cf64452aab4484ff1dee671fe63fd4b5f1b78ee2047dfa61e3d576b149bedaf83058f85f06a3172a3223ad6c4732d96b32955da7d2feb4140e58d86fc0f2eb5d9d1878e6f8a7f65ab9212030e8e915573ebbd7f35e1a430890be7e67c3fb4bbf2def662fa625421e7b411c29ebe81ec67b77355596b05cc155755664e59c16e21410aabe53e80404a615f44ebb31b365ca77a6e91241667b26c6cad24fb2324cf64e8b9dd6e2ce65f1f098cfd1ef41ba2d4c7def0ff165a0e7c84e7597c40e3dffe97d417c144545a0e38ee33ebaae12cc0c14650e453d46bfc48c0514f354773435ee89b7b2810606eb73262c77a1d67f3633705178d79a1078c3a01b5fadc9651feb63603d19decd3a00c1f69af2dab259593
	error packet for node 2: a5d3e8634cfe78b2307d87c6d90be6fe7855b4f2cc9b1dfb19e92e4b79103f61ff9ac25f412ddfb7466e74f81b3e545563cdd8f5524dae873de61d7bdfccd496af2584930d2b566b4f8d3881f8c043df92224f38cf094cfc09d92655989531524593ec6d6caec1863bdfaa79229b5020acc034cd6deeea1021c50586947b9b8e6faa83b81fbfa6133c0af5d6b07c017f7158fa94f0d206baf12dda6b68f785b773b360fd0497e16cc402d779c8d48d0fa6315536ef0660f3f4e1865f5b38ea49c7da4fd959de4e83ff3ab686f059a45c65ba2af4a6a79166aa0f496bf04d06987b6d2ea205bdb0d347718b9aeff5b61dfff344993a275b79717cd815b6ad4c0beb568c4ac9c36ff1c315ec1119a1993c4b61e6eaa0375e0aaf738ac691abd3263bf937e3
	# forwarding error packet
	shared_secret = a6519e98832a0b179f62123b3567c106db99ee37bef036e783263602f3488fae
	ammag_key = 59ee5867c5c151daa31e36ee42530f429c433836286e63744f2020b980302564
	stream = 0f10c86f05968dd91188b998ee45dcddfbf89fe9a99aa6375c42ed5520a257e048456fe417c15219ce39d921555956ae2ff795177c63c819233f3bcb9b8b28e5ac6e33a3f9b87ca62dff43f4cc4a2755830a3b7e98c326b278e2bd31f4a9973ee99121c62873f5bfb2d159d3d48c5851e3b341f9f6634f51939188c3b9ff45feeb11160bb39ce3332168b8e744a92107db575ace7866e4b8f390f1edc4acd726ed106555900a0832575c3a7ad11bb1fe388ff32b99bcf2a0d0767a83cf293a220a983ad014d404bfa20022d8b369fe06f7ecc9c74751dcda0ff39d8bca74bf9956745ba4e5d299e0da8f68a9f660040beac03e795a046640cf8271307a8b64780b0588422f5a60ed7e36d60417562938b400802dac5f87f267204b6d5bcfd8a05b221ec2
	error packet for node 1: aac3200c4968f56b21f53e5e374e3a2383ad2b1b6501bbcc45abc31e59b26881b7dfadbb56ec8dae8857add94e6702fb4c3a4de22e2e669e1ed926b04447fc73034bb730f4932acd62727b75348a648a1128744657ca6a4e713b9b646c3ca66cac02cdab44dd3439890ef3aaf61708714f7375349b8da541b2548d452d84de7084bb95b3ac2345201d624d31f4d52078aa0fa05a88b4e20202bd2b86ac5b52919ea305a8949de95e935eed0319cf3cf19ebea61d76ba92532497fcdc9411d06bcd4275094d0a4a3c5d3a945e43305a5a9256e333e1f64dbca5fcd4e03a39b9012d197506e06f29339dfee3331995b21615337ae060233d39befea925cc262873e0530408e6990f1cbd233a150ef7b004ff6166c70c68d9f8c853c1abca640b8660db2921
	# forwarding error packet
	shared_secret = 53eb63ea8a3fec3b3cd433b85cd62a4b145e1dda09391b348c4e1cd36a03ea66
	ammag_key = 3761ba4d3e726d8abb16cba5950ee976b84937b61b7ad09e741724d7dee12eb5
	stream = 3699fd352a948a05f604763c0bca2968d5eaca2b0118602e52e59121f050936c8dd90c24df7dc8cf8f1665e39a6c75e9e2c0900ea245c9ed3b0008148e0ae18bbfaea0c711d67eade980c6f5452e91a06b070bbde68b5494a92575c114660fb53cf04bf686e67ffa4a0f5ae41a59a39a8515cb686db553d25e71e7a97cc2febcac55df2711b6209c502b2f8827b13d3ad2f491c45a0cafe7b4d8d8810e805dee25d676ce92e0619b9c206f922132d806138713a8f69589c18c3fdc5acee41c1234b17ecab96b8c56a46787bba2c062468a13919afc18513835b472a79b2c35f9a91f38eb3b9e998b1000cc4a0dbd62ac1a5cc8102e373526d7e8f3c3a1b4bfb2f8a3947fe350cb89f73aa1bb054edfa9895c0fc971c2b5056dc8665902b51fced6dff80c
	error packet for node 0: 9c5add3963fc7f6ed7f148623c84134b5647e1306419dbe2174e523fa9e2fbed3a06a19f899145610741c83ad40b7712aefaddec8c6baf7325d92ea4ca4d1df8bce517f7e54554608bf2bd8071a4f52a7a2f7ffbb1413edad81eeea5785aa9d990f2865dc23b4bc3c301a94eec4eabebca66be5cf638f693ec256aec514620cc28ee4a94bd9565bc4d4962b9d3641d4278fb319ed2b84de5b665f307a2db0f7fbb757366067d88c50f7e829138fde4f78d39b5b5802f1b92a8a820865af5cc79f9f30bc3f461c66af95d13e5e1f0381c184572a91dee1c849048a647a1158cf884064deddbf1b0b88dfe2f791428d0ba0f6fb2f04e14081f69165ae66d9297c118f0907705c9c4954a199bae0bb96fad763d690e7daa6cfda59ba7f2c8d11448b604d12d

# References

[sphinx]: http://www.cypherpunks.ca/~iang/pubs/Sphinx_Oakland09.pdf
[RFC2104]: https://tools.ietf.org/html/rfc2104
[fips198]: http://csrc.nist.gov/publications/fips/fips198-1/FIPS-198-1_final.pdf
[sec2]: http://www.secg.org/sec2-v2.pdf
[rfc7539]: https://tools.ietf.org/html/rfc7539

# Authors

[ FIXME: ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
