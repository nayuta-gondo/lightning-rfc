# BOLT #1: Base Protocol

## Overview

This protocol assumes an underlying authenticated and ordered transport mechanism that takes care of framing individual messages.
[BOLT #8](08-transport.md) specifies the canonical transport layer used in Lightning, though it can be replaced by any transport that fulfills the above guarantees.

このプロトコルは、
フレーム化された独立したメッセージを処理する、認証され順序付けられたトランスポートメカニズムの下層を前提としている。
BOLT＃8は、Lightningで使用される標準的なトランスポート層を指定するが、
上記の保証を満たすトランスポートで置き換えることができる。

The default TCP port is 9735. This corresponds to hexadecimal `0x2607`: the Unicode code point for LIGHTNING.<sup>[1](#reference-1)</sup>

デフォルトのTCPポートは9735である。
これは16進数の0x2607対応する：LIGHTNINGのUnicode code point。

All data fields are unsigned big-endian unless otherwise specified.

特に明記しない限り、すべてのデータフィールドは符号なしビッグエンディアンである。

## Table of Contents

  * [Connection Handling and Multiplexing](#connection-handling-and-multiplexing)
  * [Lightning Message Format](#lightning-message-format)
  * [Type-Length-Value Format](#type-length-value-format)
  * [Fundamental Types](#fundamental-types)
  * [Setup Messages](#setup-messages)
    * [The `init` Message](#the-init-message)
    * [The `error` Message](#the-error-message)
  * [Control Messages](#control-messages)
    * [The `ping` and `pong` Messages](#the-ping-and-pong-messages)
  * [Acknowledgments](#acknowledgments)
  * [References](#references)
  * [Authors](#authors)

## Connection Handling and Multiplexing

Implementations MUST use a single connection per peer; channel messages (which include a channel ID) are multiplexed over this single connection.

実装は、peerごとに単一の接続を使用しなければならない。
（channel IDを含む）channel messageは、この単一の接続を介して多重化される。

## Lightning Message Format

After decryption, all Lightning messages are of the form:

1. `type`: a 2-byte big-endian field indicating the type of message
2. `payload`: a variable-length payload that comprises the remainder of
   the message and that conforms to a format matching the `type`

解読後、すべてのLightning messagesの形式は次のとおりである：

1. type：messageのタイプを示す2バイトのビッグエンディアンのフィールド
2. payload：messageの残りの部分から成る、それはtypeに一致するフォーマットに従う、可変長ペイロード

The `type` field indicates how to interpret the `payload` field.
The format for each individual type is defined by a specification in this repository.
The type follows the _it's ok to be odd_ rule, so nodes MAY send _odd_-numbered types without ascertaining that the recipient understands it.

typeフィールドはpayloadフィールドをどのように解釈するかを示す。
個々のtypeの形式は、このリポジトリ内の仕様によって定義される。
型はそれがit's ok to be oddルールに従っているので、
nodeは受信者がそれを理解していることを確認せずに奇数のtypesを送信することができる。

A sending node:
  - MUST NOT send an evenly-typed message not listed here without prior negotiation.

送信node：
  - 事前交渉なしにここにリストされていない偶数typeのmessageを送信してはいけない。
  （XXX: negotiationとは？フィーチャービット？
  奇数typeはnegotiationとやらはいらない？）

A receiving node:
  - upon receiving a message of _odd_, unknown type:
    - MUST ignore the received message.
  - upon receiving a message of _even_, unknown type:
    - MUST fail the channels.

受信node：
  - 未知の奇数typeのmessageを受信すると：
    - 受信したmessageを無視しなければならない。
  - 未知の偶数typeのmessageを受信すると：
    - channelに失敗しなければならない。

The messages are grouped logically into four groups, ordered by the most significant bit that is set:

  - Setup & Control (types `0`-`31`): messages related to connection setup, control, supported features, and error reporting (described below)
  - Channel (types `32`-`127`): messages used to setup and tear down micropayment channels (described in [BOLT #2](02-peer-protocol.md))
  - Commitment (types `128`-`255`): messages related to updating the current commitment transaction, which includes adding, revoking, and settling HTLCs as well as updating fees and exchanging signatures (described in [BOLT #2](02-peer-protocol.md))
  - Routing (types `256`-`511`): messages containing node and channel announcements, as well as any active route exploration (described in
  [BOLT #7](07-routing-gossip.md))

messagesは論理的に4つのグループにグループ化され、設定されているmost significant bitによって順序付けられる。

  - Setup ＆ Control（types 0 - 31）：
  接続のセットアップ、コントロール、サポートされている機能、エラー報告に関連するmessages（後述）
  - Channel（types 32 - 127）：
  micropayment channelsのセットアップと解除に使用されるmessages（BOLT＃2で説明）
  - Commitment（types 128 - 255）：
  HTLCの追加、取り消しおよび決済、同様にfeesの更新、署名の交換など、
  現在のcommitment transactionの更新に関するmessages（BOLT＃2で説明）
  - Routing（types 256 - 511）：
  nodeとchannel announcements、およびアクティブなルート探索を含むメッセージ（BOLT＃7で説明）

The size of the message is required by the transport layer to fit into a 2-byte unsigned int; therefore, the maximum possible size is 65535 bytes.

messageのサイズは、トランスポート層が2バイトの符号なし整数に収まるように要求される。
したがって、可能な最大サイズは65535バイトである。
（XXX: トランスポート層とはNoise Protocolのこと）

A node:
  - MUST ignore any additional data within a message beyond the length that it expects for that type.
  - upon receiving a known message with insufficient length for the contents:
    - MUST fail the channels.
  - that negotiates an option in this specification:
    - MUST include all the fields annotated with that option.

node：
  - そのtypeに期待される長さを超える、メッセージ内のadditional dataを無視しなければならない。
  - コンテンツの長さが不十分な既知のmessageを受信すると、
    - channelに失敗しなければならない。
  - この仕様書でオプションを交渉する：
    - そのオプションで注釈が付けられたすべてのフィールドを含める必要がある。
    （XXX: そのオプションを使う場合は、この仕様書に従って、そのオプションに必要なフィールドを含める必要がある）

### Rationale

By default `SHA2` and Bitcoin public keys are both encoded as
big endian, thus it would be unusual to use a different endian for
other fields.

デフォルトでは、SHA2とBitcoin public keysはどちらもビッグエンディアンとしてエンコードされているため、
他のフィールドには異なるエンディアンを使用することは珍しいことである。

Length is limited to 65535 bytes by the cryptographic wrapping, and
messages in the protocol are never more than that length anyway.

長さは暗号のラッピングによって65535バイトに制限され、
プロトコルのmessageは決してその長さを超えることはない。

The _it's ok to be odd_ rule allows for future optional extensions
without negotiation or special coding in clients. The "ignore
additional data" rule similarly allows for future expansion.

it's ok to be oddルールは、交渉やクライアントに特別なコーディングなしで、
将来のオプションの拡張が可能になる。
「ignore additional data」ルールは、同様に将来の拡張を可能にする。

Implementations may prefer to have message data aligned on an 8-byte
boundary (the largest natural alignment requirement of any type here);
however, adding a 6-byte padding after the type field was considered
wasteful: alignment may be achieved by decrypting the message into
a buffer with 6-bytes of pre-padding.

実装では、messageデータを8バイト境界に整列させることが望ましいかもしれない
（ここでは任意のtypeの最大整数アライメント要件）。
しかし、typeフィールドの後に6バイトのパディングを追加することは無駄であると考えた。
6バイトのプレパディングを持つバッファでmessageを復号化することによって、アライメントを達成できる。
（XXX: 先頭６バイトを気にせず復号？）

## Type-Length-Value Format

Throughout the protocol, a TLV (Type-Length-Value) format is used to allow for
the backwards-compatible addition of new fields to existing message types.

プロトコル全体で、TLV (Type-Length-Value)形式は、
既存のメッセージタイプに後方互換性のある新しいフィールドの追加を可能にするために使用される。

A `tlv_record` represents a single field, encoded in the form:

`tlv_record`は、以下の形式でエンコードされた単一のフィールドを表す：

* [`varint`: `type`]
* [`varint`: `length`]
* [`length`: `value`]

A `tlv_stream` is a series of (possibly zero) `tlv_record`s, represented as the
concatenation of the encoded `tlv_record`s. When used to extend existing
messages, a `tlv_stream` is typically placed after all currently defined fields.

`tlv_stream`は、エンコードされた`tlv_record`の連結として表現される一連の(0の可能性がある)`tlv_record'である。
（XXX: zeroという表現はよくないだろう。）
既存のメッセージを拡張するために使用される場合、
`tlv_stream`は通常、現在定義されているすべてのフィールドの後に配置される。

The `type` is a varint encoded using the bitcoin CompactSize format. It
functions as a message-specific, 64-bit identifier for the `tlv_record`
determining how the contents of `value` should be decoded.

`type`はビットコインCompactSizeフォーマットを用いて符号化されたvarintである。
`value`の内容をデコードする方法を決定する `tlv_record`のメッセージ固有の64ビット識別子として機能する。

The `length` is a varint encoded using the bitcoin CompactSize format
signaling the size of `value` in bytes.

`length`は、ビットコインCompactSizeフォーマットを使用して符号化されたvarintであり、
バイト単位で`value'のサイズを示す。

The `value` depends entirely on the `type`, and should be encoded or decoded
according to the message-specific format determined by `type`.

`value`は完全に`type`に依存し、
`type`によって決定されるメッセージ固有のフォーマットに従ってエンコードまたはデコードされるべきである。

### Requirements

The sending node:
 - MUST order `tlv_record`s in a `tlv_stream` by monotonically-increasing `type`.
 - MUST minimally encode `type` and `length`.
 - SHOULD NOT use redundant, variable-length encodings in a `tlv_record`.

The sending node:
 - `tlv_stream`s内の`tlv_record`を単調に増加する`type`で順序付けしなければならない。
 - `type`と`length`は最低限の長さでエンコードしなければならない。
 - 冗長な可変長符号化を`tlv_record`で使用すべきではありません。

The receiving node:
 - if zero bytes remain before parsing a `type`:
   - MUST stop parsing the `tlv_stream`.
 - if a `type` or `length` is not minimally encoded:
   - MUST fail to parse the `tlv_stream`.
 - if decoded `type`s are not monotonically-increasing:
   - MUST fail to parse the `tlv_stream`.
 - if `type` is known:
   - MUST decode the next `length` bytes using the known encoding for `type`.
 - otherwise, if `type` is unknown:
   - if `type` is even:
     - MUST fail to parse the `tlv_stream`.
   - otherwise, if `type` is odd:
     - MUST discard the next `length` bytes.

The receiving node:
 - `type`を解析する前に0バイトが残っている場合:
   - `tlv_stream`の解析を停止しなければならない。
 - `type`または`length`のエンコードが最小でない場合:
   - `tlv_stream`の解析に失敗しなければならない。
 - デコードされた`type`sが単調に増加していない場合:
   - `tlv_stream`の解析に失敗しなければならない。
 - `type`がわかっている場合:
   - 次の`length`バイトを`type`の既知のエンコーディングを使ってデコードしなければならない。
 - そうではなく、`type`が未知の場合:
   - `type`が偶数の場合:
     - `tlv_stream`の解析に失敗しなければならない。
   - そうではなく、`type`が奇数ならば:
     - 次の`length`バイトを破棄しなければならない。
     (XXX: `length`と`value`フィールドを破棄、であろう)

### Rationale

The primary advantage in using TLV is that a reader is able to ignore new fields
that it does not understand, since each field carries the exact size of the
encoded element. Without TLV, even if a node does not wish to use a particular
field, the node is forced to add parsing logic for that field in order to
determine the offset of any fields that follow.

TLVを使用する主な利点は、各フィールドがエンコードされた要素の正確なサイズを保持するため、
理解できない新しいフィールドを無視できることである。
TLVを使用しない場合、ノードが特定のフィールドを使用しない場合でも、
後続のフィールドのオフセットを判断するために、そのフィールドの解析ロジックを追加する必要がある。

The monotonicity constraint ensures that all `type`s are unique and can appear
at most once. Fields that map to complex objects, e.g. vectors, maps, or
structs, should do so by defining the encoding such that the object is
serialized within a single `tlv_record`. The uniqueness constraint, among other
things, enables the following optimizations:
 - canonical ordering is defined independent of the encoded `value`s.
 - canonical ordering can be known at compile-time, rather that being determined
   dynamically at the time of encoding.
 - verifying canonical ordering requires less state and is less-expensive.
 - variable-size fields can reserve their expected size up front, rather than
   appending elements sequentially and incurring double-and-copy overhead.

単調性制約は、すべての`type`が一意であり、一度だけ現れることを保証する。
ベクトル、マップ、または構造体などの複雑なオブジェクトにマップするフィールドは、
オブジェクトが単一の`tlv_record`内で直列化されるようにエンコーディングを定義することによって、
そのようにする必要がある。一意性制約は、特に次の最適化を可能にする。
 - 正規順序は、エンコードされた`value`sとは無関係に定義される。
 - 正規順序は、エンコード時に動的に決定されるのではなく、コンパイル時に認識される。
 - 正規順序を検証は、必要な状態が少なくなり、コストが削減される。
 - 可変サイズのフィールドは、要素を順番に追加して二重コピーのオーバーヘッドを発生させるのではなく、
 前もって期待されるサイズを予約することができる。

The use of a varint for `type` and `length` permits a space savings for small
`type`s or short `value`s. This potentially leaves more space for application
data over the wire or in an onion payload.

`type`と`length`にvarintを使用することで、小さな`type`や短い`value`sのスペースを節約できる。
これにより、wire上またはオニオン・ペイロード内にアプリケーション・データのためのスペースが増える可能性がある。

All `type`s must appear in increasing order to create a canonical encoding of
the underlying `tlv_record`s. This is crucial when computing signatures over a
`tlv_stream`, as it ensures verifiers will be able to recompute the same message
digest as the signer. Note that the canonical ordering over the set of fields
can be enforced even if the verifier does not understand what the fields
contain.

すべての`type`は、基本となる`tlv_record`の正規エンコーディングを作成するために、昇順に出現する必要がある。
これは、検証者が署名者と同じメッセージダイジェストを再計算できるようにするため、
`tlv_stream`上で署名を計算するときに重要である。
検証者がフィールドが何を含んでいるかを理解していない場合でも、
フィールドの集合に対する標準的な順序付けを強制できることに注意されたい。

Writers should avoid using redundant, variable-length encodings in a
`tlv_record` since this results in encoding the length twice and complicates
computing the outer length. As an example, when writing a variable length byte
array, the `value` should contain only the raw bytes and forgo an additional
internal length since the `tlv_record` already carries the number of bytes that
follow. On the other hand, if a `tlv_record` contains multiple, variable-length
elements then this would not be considered redundant, and is needed to allow the
receiver to parse individual elements from `value`.

`tlv_record`で冗長な可変長符号化を使用することは、結果として長さを2回符号化することになり、
外側の長さの計算を複雑にするので、ライタは避けるべきである。
たとえば、可変長のバイト配列を書き込む場合、`tlv_record`はすでに後続のバイト数を保持しているため、
`value`にはrawバイトのみを含め、内部の長さを追加する必要はない。
一方、`tlv_record`が複数の可変長要素を含む場合、これは冗長とは見なされず、
受信側が`value`から個々の要素を解析できるようにするために必要である。
（XXX: writerに対比させるならreceiverじゃなくてreaderだろ）

## Fundamental Types

Various fundamental types are referred to in the message specifications:

メッセージ仕様では、さまざまな基本的なタイプが参照される：

* `byte`: an 8-bit byte
* `u16`: a 2 byte unsigned integer
* `u32`: a 4 byte unsigned integer
* `u64`: an 8 byte unsigned integer

The following convenience types are also defined:

次のコンビニエンス・タイプも定義されている：

* `chain_hash`: a 32-byte chain identifier (see [BOLT #0](00-introduction.md#glossary-and-terminology-guide))
* `channel_id`: a 32-byte channel_id (see [BOLT #2](02-peer-protocol.md#definition-of-channel-id)
* `sha256`: a 32-byte SHA2-256 hash
* `signature`: a 64-byte bitcoin Elliptic Curve signature
* `point`: a 33-byte Elliptic Curve point (compressed encoding as per [SEC 1 standard](http://www.secg.org/sec1-v2.pdf#subsubsection.2.3.3))
* `pubkey`: a `point` explicitly for use as a public key
* `preimage`: a 32 byte value used as a preimage for a hash
* `short_channel_id`: an 8 byte value identifying a channel (see [BOLT #7](07-routing-gossip.md#definition-of-short-channel-id))
* `secret`: a 32 byte secret being revealed to the peer

（XXX: 区切り）

* `chain_hash`: 32バイトのチェイン識別子(BOLT#0を参照)
* `channel_id`: 32バイトのchannel_id(BOLT#2を参照)
* `sha256`: 32バイトのSHA2-256ハッシュ
* `signature`: 64バイトのbitcoin楕円曲線署名
* `point`: 33バイトの楕円曲線の点(SEC1規格による圧縮エンコーディング)
* `pubkey`: 公開鍵として明示的に使用する`point`
* `preimage`: ハッシュのプリイメージとして使用される32バイトの値
* `short_channel_id`: チャンネルを識別する8バイトの値(BOLT#7を参照)
* `secret`: ピアに公開される32バイトの秘密

## Setup Messages

### The `init` Message

Once authentication is complete, the first message reveals the features supported or required by this node, even if this is a reconnection.

認証が完了すると、このノードがサポートしているかまたは必要としている機能が、たとえこれが再接続であっても、最初のメッセージで明らかになる。
（XXX: これは接続時毎回？）

[BOLT #9](09-features.md) specifies lists of global and local features. Each feature is generally represented in `globalfeatures` or `localfeatures` by 2 bits. The least-significant bit is numbered 0, which is _even_, and the next most significant bit is numbered 1, which is _odd_.

BOLT＃9は、グローバルおよびローカル機能のリストを指定する。
各機能は通常、globalfeaturesか、localfeaturesの2ビットで表される。
最下位ビットは0であり、これは偶数であり、次の最上位ビットは1であり、これは奇数である。

Both fields `globalfeatures` and `localfeatures` MUST be padded to bytes with 0s.

globalfeaturesとlocalfeatures、両方のフィールドは0のバイトにパディングされなければならない。

1. type: 16 (`init`)
2. data:
   * [`u16`:`gflen`]
   * [`gflen*byte`:`globalfeatures`]
   * [`u16`:`lflen`]
   * [`lflen*byte`:`localfeatures`]

The 2-byte `gflen` and `lflen` fields indicate the number of bytes in the immediately following field.

2バイトのgflenおよびlflenフィールドは、直後のフィールドのバイト数を指定する。

#### Requirements

The sending node:
  - MUST send `init` as the first Lightning message for any connection.
  - MUST set feature bits as defined in [BOLT #9](09-features.md).
  - MUST set any undefined feature bits to 0.
  - SHOULD use the minimum lengths required to represent the feature fields.

送信ノード：
  - 任意の接続のための最初のLightningメッセージとして、initを送らなければならない。
  - BOLT＃9で定義されている、機能ビットを設定しなければならない。
  - 未定義の機能ビットを0に設定しなければならない。
  - フィーチャフィールドを表現するのに必要な最小長を使用すべきである。

The receiving node:
  - MUST wait to receive `init` before sending any other messages.
  - MUST respond to known feature bits as specified in [BOLT #9](09-features.md).
  - upon receiving unknown _odd_ feature bits that are non-zero:
    - MUST ignore the bit.
  - upon receiving unknown _even_ feature bits that are non-zero:
    - MUST fail the connection.

受信ノード：
  - 他のメッセージを送信する前にinitの受信を待つ必要がある。
  - BOLT＃9で指定された既知の機能ビットに応答しなければならない。
  - 非ゼロである未知の奇数機能ビットを受信した場合：
    - ビットを無視しなければならない。
  - 非ゼロである未知の偶数機能ビットを受信した場合：
    - 接続に失敗しなければならない。

#### Rationale

This semantic allows both future incompatible changes and future backward compatible changes. Bits should generally be assigned in pairs, in order that optional features may later become compulsory.

このセマンティックは、将来の互換性のない変更と、将来の下位互換性のある変更の両方を可能にする。
オプションの機能が後で強制になるように、ビットは通常ペアで割り当てられるべきである。

Nodes wait for receipt of the other's features to simplify error
diagnosis when features are incompatible.

ノードは、機能が互換性がないときのエラー診断を単純化するために、
他の機能の受信を待機する。（XXX: initを受信するまで）

The feature masks are split into local features (which only affect the
protocol between these two nodes) and global features (which can affect
HTLCs and are thus also advertised to other nodes).

フィーチャマスクは、local features（これら2つのノード間のプロトコルにのみ影響する）と、
global features（HTLCに影響を及ぼし、他のノードにもアドバタイズされる）に分割される。

### The `error` Message

For simplicity of diagnosis, it's often useful to tell a peer that something is incorrect.

診断を簡単にするために、何かが間違っていることをピアに伝えることはしばしば有用である。
（XXX: これはピア用）

1. type: 17 (`error`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`data`]

The 2-byte `len` field indicates the number of bytes in the immediately following field.

2バイトのlenフィールドは、直後のフィールドのバイト数を示す。

#### Requirements

The channel is referred to by `channel_id`, unless `channel_id` is 0 (i.e. all bytes are 0), in which case it refers to all channels.

チャネルはchannel_idによって参照される、
channel_idが0でない限りは（すなわち、すべてのバイトは0である）、
その場合には全てのチャンネルを指す。

The funding node:
  - for all error messages sent before (and including) the `funding_created` message:
    - MUST use `temporary_channel_id` in lieu of `channel_id`.

fundingノード：
  - funding_createdメッセージの前（それを含む）に送信されたすべてのエラーメッセージ：
    - channel_idの代わりにtemporary_channel_idを使用しなければならない。

The fundee node:
  - for all error messages sent before (and not including) the `funding_signed` message:
    - MUST use `temporary_channel_id` in lieu of `channel_id`.

fundeeノード：
  - funding_signedメッセージの前（それを含まない）に送信されたすべてのエラーメッセージ：
    - channel_idの代わりにtemporary_channel_idを使用しなければならない。
    （XXX: funding_signedにchannel_idが乗っている）

A sending node:
  - when sending `error`:
    - MUST fail the channel referred to by the error message.
  - SHOULD send `error` for protocol violations or internal errors that make channels unusable or that make further communication unusable.
  - SHOULD send `error` with the unknown `channel_id` in reply to messages of type `32`-`255` related to unknown channels.
  - MAY send an empty `data` field.
  - when failure was caused by an invalid signature check:
    - SHOULD include the raw, hex-encoded transaction in reply to a `funding_created`, `funding_signed`, `closing_signed`, or `commitment_signed` message.
  - when `channel_id` is 0:
    - MUST fail all channels with the receiving node.
    - MUST close the connection.
  - MUST set `len` equal to the length of `data`.

送信ノード：
  - error送信時：
    - エラーメッセージによって参照されるチャネルを失敗しなければならない。
  - プロトコルの違反や内部エラーのためのerrorを送信して、チャネルを使用不能にしたり、それ以上の通信を使用不能にすべきである。
  - 未知のチャネルに関連するタイプ32-255のメッセージに応答して、
  未知のchannel_idでエラーを送信すべきである。（XXX: メッセージで送られて来たchannel_idをそのまま入れてerrorを返す）
  - 空のdataフィールドを送っても良い。
  - 不正な署名チェックによって失敗した場合：
    - 応答に生の、16進エンコードトランザクションを含むべきである、funding_created、funding_signed、closing_signed、またはcommitment_signedメッセージには。
  - channel_idが0のとき：
    - 受信ノードのすべてのチャネルを失敗にしなければならない。
    - 接続を閉じなければならない。
  - lenはdataの長さに等しく設定しなければならない。

The receiving node:
  - upon receiving `error`:
    - MUST fail the channel referred to by the error message, if that channel is with the sending node.
  - if no existing channel is referred to by the message:
    - MUST ignore the message.
  - MUST truncate `len` to the remainder of the packet (if it's larger).
  - if `data` is not composed solely of printable ASCII characters (For reference: the printable character set includes byte values 32 through 126, inclusive):
    - SHOULD NOT print out `data` verbatim.

受信ノード：
  - error受信時：
    - そのチャネルが送信ノードとの間にある場合は、エラーメッセージが参照するチャネルを失敗しなければならない。
  - そのメッセージによって既存のチャネルが参照されていない場合：
    - メッセージを無視しなければならない。
  - パケットの残りの部分はlenに切り詰めるなければならない（より大きければ）。
  - dataが印字可能なASCII文字のみで構成されていない場合（参考：印刷可能な文字セットには32〜126のバイト値が含まれる）
    - dataを文字通りに印字すべきではない。

#### Rationale

There are unrecoverable errors that require an abort of conversations;
if the connection is simply dropped, then the peer may retry the
connection. It's also useful to describe protocol violations for
diagnosis, as this indicates that one peer has a bug.

会話の中断が必要な回復不可能なエラーがある。
単に接続が切断された場合、ピアは接続を再試行するであろう。
1つのピアにバグがあることを示すので、診断のためにプロトコル違反を記述することも有益である。

It may be wise not to distinguish errors in production settings, lest
it leak information — hence, the optional `data` field.

それによって情報が漏れないようにするために、
生産設定でのエラーを区別しないようにすることが賢明と思われる。
したがって、dataフィールドはオプションである。
（XXX: 生産設定とデバッグ設定を分けるなということ？
例えばデバッグ設定がプロダクションで出ていったときのために？
例えデバッグ設定でも秘密鍵を送ったりしないようにか）

## Control Messages

### The `ping` and `pong` Messages

In order to allow for the existence of long-lived TCP connections, at
times it may be required that both ends keep alive the TCP connection at the
application level. Such messages also allow obfuscation of traffic patterns.

長時間のTCP接続を可能にするために、
両端でアプリケーションレベルでのTCP接続を維持が必要な場合がある。
そのようなメッセージはまた、トラフィックパターンの難読化を可能にする。

1. type: 18 (`ping`)
2. data:
    * [`u16`:`num_pong_bytes`]
    * [`u16`:`byteslen`]
    * [`byteslen*byte`:`ignored`]

The `pong` message is to be sent whenever a `ping` message is received. It
serves as a reply and also serves to keep the connection alive, while
explicitly notifying the other end that the receiver is still active. Within
the received `ping` message, the sender will specify the number of bytes to be
included within the data payload of the `pong` message.

pingメッセージが受信されたときいつでも、pongメッセージは送信される。（XXX: 実際の実装はそうなってない？）
これは応答として機能し、受信側がまだアクティブであることを明示的に他方の側に通知しながら、接続を維持する役割も果たす。
受信されるpingメッセージの内に、（XXX: pingの）送信者はpongメッセージのデータペイロード内に含めるバイト数を指定する。

1. type: 19 (`pong`)
2. data:
    * [`u16`:`byteslen`]
    * [`byteslen*byte`:`ignored`]

（XXX: ignoredってなに？受け取っても無視するってこと？）

#### Requirements

A node sending a `ping` message:
  - SHOULD set `ignored` to 0s.
  - MUST NOT set `ignored` to sensitive data such as secrets or portions of initialized
memory.
  - if it doesn't receive a corresponding `pong`:
    - MAY terminate the network connection,
      - and MUST NOT fail the channels in this case.
  - SHOULD NOT send `ping` messages more often than once every 30 seconds.

pingメッセージを送信するノード：
  - ignoredを0に設定すべきである。
    - ignoredにsecretsや初期化されたメモリの一部などの機密データを設定してはならない。
  - それに対応するpongを受け取らない場合：
    - ネットワーク接続を終了してもよい、
      - この場合、チャネルを失敗してはならない。
  - 30秒ごとに1回以上pingメッセージを送信すべきではない。

A node sending a `pong` message:
  - SHOULD set `ignored` to 0s.
  - MUST NOT set `ignored` to sensitive data such as secrets or portions of initialized
 memory.

pongメッセージを送信するノード：
  - ignoredを0に設定すべきである。
    - ignoredにsecretsや初期化されたメモリの一部などの機密データを設定してはならない。

A node receiving a `ping` message:
  - SHOULD fail the channels if it has received significantly in excess of one `ping` per 30 seconds.
  - if `num_pong_bytes` is less than 65532:
    - MUST respond by sending a `pong` message, with `byteslen` equal to `num_pong_bytes`.
  - otherwise (`num_pong_bytes` is **not** less than 65532):
    - MUST ignore the `ping`.

pingメッセージを受信したノード：
  - 30秒間に1回のpingを大幅に超過して受信した場合、そのチャネルは失敗するべきである。
  - num_pong_bytesが65532未満の場合：
    - pongメッセージを送信することによって応答しなければならない、byteslenをnum_pong_bytesに等しく。
  - そうでない場合（num_pong_bytesが65532未満ではない）：
    - pingを無視しなければならない。

A node receiving a `pong` message:
  - if `byteslen` does not correspond to any `ping`'s `num_pong_bytes` value it has sent:
    - MAY fail the channels.

pongメッセージを受信したノード：
  - byteslenが、送信されたどのpingのnum_pong_bytes値にも対応していない場合：
    - チャンネルに失敗してもよい。

### Rationale

The largest possible message is 65535 bytes; thus, the maximum sensible `byteslen`
is 65531 — in order to account for the type field (`pong`) and the `byteslen` itself. This allows
a convenient cutoff for `num_pong_bytes` to indicate that no reply should be sent.

可能な最大のメッセージは65535バイトである。
したがって、最大の意味のあるbyteslenは65531である。
typeフィールド（pong）とbyteslenそれ自身を計上するため。
これは返信を送信すべきでないことを示す、num_pong_bytesの簡易なカットオフを可能にする。

Connections between nodes within the network may be long lived, as payment
channels have an indefinite lifetime. However, it's likely that
no new data will be
exchanged for a
significant portion of a connection's lifetime. Also, on several platforms it's possible that Lightning
clients will be put to sleep without prior warning. Hence, a
distinct `ping` message is used, in order to probe for the liveness of the connection on
the other side, as well as to keep the established connection active.

ペイメントチャネルの寿命が不定であるため、ネットワーク内のノード間の接続は長寿命になる可能性がある。
ただし、接続の生存期間のかなりの部分で新しいデータが交換されない可能性がある。
また、いくつかのプラットフォームでは、Lightningクライアントが事前の警告なしにスリープ状態になる可能性がある。
したがって、確立された接続をアクティブに保つだけでなく、相手側の接続の有効性を調べるために、別のpingメッセージが使用される。
（XXX: keep aliveだけではなく、通常のpingのような目的で、ということか）

Additionally, the ability for a sender to request that the receiver send a
response with a particular number of bytes enables nodes on the network to
create _synthetic_ traffic. Such traffic can be used to partially defend
against packet and timing analysis — as nodes can fake the traffic patterns of
typical exchanges without applying any true updates to their respective
channels.

さらに、送信側が受信側に特定のバイト数の応答を送信するよう要求する機能によって、
ネットワーク上のノードは合成（synthetic）トラフィックを作成できる。
このようなトラフィックは、ノードがそれぞれのチャネルに実際の更新を適用せずに、
典型的な交換のトラフィックパターンを偽装できることにより、
パケットとタイミング解析に対して部分的に防御するために使用することができる。

When combined with the onion routing protocol defined in
[BOLT #4](04-onion-routing.md),
careful statistically driven synthetic traffic can serve to further bolster the
privacy of participants within the network.

BOLT＃4で定義されるオニオンルーティングプロトコルと組み合わされた場合、
統計的に駆動される合成トラフィックは、ネットワーク内の参加者のプライバシをさらに強化するのに役立つ。

Limited precautions are recommended against `ping` flooding, however some
latitude is given because of network delays. Note that there are other methods
of incoming traffic flooding (e.g. sending _odd_ unknown message types, or padding
every message maximally).

pingフラッディングに対して、制限（XXX: 30秒に一回）の注意事項が推奨されている。
しかしながら、ネットワークの遅延のために、ある程度の自由度が与えられる。
流入トラフィックのフラッディングの他の方法があることに注意すること
（たとえば、奇数の未知のメッセージタイプを送信する、またはすべてのメッセージを最大限にパディングする）。

Finally, the usage of periodic `ping` messages serves to promote frequent key
rotations as specified within [BOLT #8](08-transport.md).

最後に、周期的なpingメッセージの使用は、BOLT＃8で指定されているように、頻繁なキーローテーションを促進する。

## Acknowledgments

[ TODO: (roasbeef); fin ]

## References

1. <a id="reference-2">http://www.unicode.org/charts/PDF/U2600.pdf</a>

## Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
