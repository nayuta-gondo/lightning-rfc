# BOLT #7: P2P Node and Channel Discovery

This specification describes simple node discovery, channel discovery, and channel update mechanisms that do not rely on a third-party to disseminate the information.

この仕様では、情報を普及させるために第三者に依存しない単純なnode discovery、channel discovery、およびchannel updateのメカニズムについて説明する。

Node and channel discovery serve two different purposes:

 - Channel discovery allows the creation and maintenance of a local view of the network's topology, so that a node can discover routes to desired destinations.

 - Node discovery allows nodes to broadcast their ID, host, and port, so that other nodes can open connections and establish payment channels with them.

nodeとchannelのdiscoveryには、次の2つの目的がある：

  - channel discoveryは、ネットワークのトポロジのローカルビューの作成および保守を可能にするので、nodeは所望の宛先へのルートをdiscoveryすることができる。

  - node discoveryにより、nodeはID、ホスト、およびポートをブロードキャストでき、他のnodesが接続を開いて支払いchannelsを確立することができる。

To support channel discovery, three *gossip messages* are supported.   Peers in the network exchange
`channel_announcement` messages containing information regarding new
channels between the two nodes. They can also exchange `channel_update`
messages, which update information about a channel. There can only be
one valid `channel_announcement` for any channel, but at least two
`channel_update` messages are expected.

channel discoveryをサポートするために、3つのgossip messagesがサポートされている。
ネットワーク内のpeersは、2つのnodes間の新しいchannelsに関する情報を含むchannel_announcement messagesを交換する。
また、channelに関する情報をupdateするchannel_update messagesを交換することもできる。
任意のchannelに対して有効なchannel_announcementは1つだけだが、少なくとも2つのchannel_update messagesが必要である。

To support node discovery, peers exchange `node_announcement`
messages, which supply additional information about the nodes. There may be
multiple `node_announcement` messages, in order to update the node information.

node discoveryをサポートするために、peersはnodesに関する追加情報を提供するnode_announcement messagesを交換する。
nodes情報をupdateするために、複数のnode_announcement messagesが存在する可能性がある。

# Table of Contents

  * [Definition of `short_channel_id`](#definition-of-short-channel-id)
  * [The `announcement_signatures` Message](#the-announcement_signatures-message)
  * [The `channel_announcement` Message](#the-channel_announcement-message)
  * [The `node_announcement` Message](#the-node_announcement-message)
  * [The `channel_update` Message](#the-channel_update-message)
  * [Query Messages](#query-messages)
  * [Initial Sync](#initial-sync)
  * [Rebroadcasting](#rebroadcasting)
  * [HTLC Fees](#htlc-fees)
  * [Pruning the Network View](#pruning-the-network-view)
  * [Recommendations for Routing](#recommendations-for-routing)
  * [References](#references)

## Definition of `short_channel_id`

The `short_channel_id` is the unique description of the funding transaction.
It is constructed as follows:
  1. the most significant 3 bytes: indicating the block height
  2. the next 3 bytes: indicating the transaction index within the block
  3. the least significant 2 bytes: indicating the output index that pays to the channel.

short_channel_idはfunding transactionの一意な記述である。
それは次のように構築される：
  1. 最上位3バイト：ブロックの高さを示す
  2. 次の3バイト：ブロック内のtransactionインデックスを示す
  3. 最下位2バイト：channelに支払うoutputインデックスを示す。

（XXX: channel_idはTXIDとoutputインデックスのXOR）

The standard human readable format for `short_channel_id` is created
by printing the above components, in the order:
block height, transaction index, and output index.
Each component is printed as a decimal number,
and separated from each other by the small letter `x`.
For example, a `short_channel_id` might be written as `539268x845x1`,
indicating a channel on the output 1 of the transaction at index 845
of the block at height 539268.

short_channel_idのThe standard human readable formatは、
block height、 transaction index、 およびoutput indexの順番で
上記のコンポーネントを表記することによって作成される。
各構成要素は10進数で表記され、小文字のxで区切られている。
たとえば、short_channel_idは、539268x845x1と書かれていて、
高さ539268のブロックのインデックス845にあるトランザクションの出力1のチャネルを示す。

### Rationale

The `short_channel_id` human readable format is designed
so that double-clicking or double-tapping it will select the entire ID
on most systems.
Humans prefer decimal when reading numbers,
so the ID components are written in decimal.
The small letter `x` is used since on most fonts,
the `x` is visibly smaller than decimal digits,
making it easy to visibly group each component of the ID.

人間が読める形式のshort_channel_idは、ダブルクリックまたはダブルタップすることにより、
ほとんどのシステムでID全体が選択されるように設計されている。
人間は数字を読むときに10進数を使うので、IDコンポーネントは10進数で書かれる。
ほとんどのフォントでは、xが10進数より視覚的に小さいので、
IDの各コンポーネントを視覚的に簡単にグループ化できるので、小さな文字xが使用される。

## The `announcement_signatures` Message

This is a direct message between the two endpoints of a channel and serves as an opt-in mechanism to allow the announcement of the channel to the rest of the network.
It contains the necessary signatures, by the sender, to construct the `channel_announcement` message.

これは、channelの2つのendpoints間のdirect messageで、
channelのannouncementをネットワークの他の部分に許可するオプトインメカニズムを提供する。
これには送信者がchannel_announcement messageを作成するために必要なsignaturesが含まれている。
（XXX: ここでオプトインという表現は不適切だと思う。
意味としてはchannel_announcementを交換したらこのchannelを公開することができるようになる。
このタイミングで公開することを「決める」という意図のものではない）

1. type: 259 (`announcement_signatures`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`short_channel_id`:`short_channel_id`]
    * [`signature`:`node_signature`]
    * [`signature`:`bitcoin_signature`]

The willingness of the initiating node to announce the channel is signaled during channel opening by setting the `announce_channel` bit in `channel_flags` (see [BOLT #2](02-peer-protocol.md#the-open_channel-message)).

開始nodeがchannelをannounceする意思は、
channel_flagsのannounce_channelビットをセットすることによってchannel開通中に通知される（BOLT＃2参照）。

### Requirements

The `announcement_signatures` message is created by constructing a `channel_announcement` message, corresponding to the newly established channel, and signing it with the secrets matching an endpoint's `node_id` and `bitcoin_key`. After it's signed, the
`announcement_signatures` message may be sent.

announcement_signatures messageはchannel_announcement messageを構築することにより作成され、
新しく設立されたchannelに対応し、endpointのnode_idとbitcoin_key（XXX: 後でわかる）にマッチングしたsecretsで署名される。
署名された後、 announcement_signatures messageを送信することができる。

A node:
  - if the `open_channel` message has the `announce_channel` bit set AND a `shutdown` message has not been sent:
    - MUST send the `announcement_signatures` message.
      - MUST NOT send `announcement_signatures` messages until `funding_locked`
      has been sent and received AND the funding transaction has at least six confirmations.

  - open_channel messageのannounce_channelビットが設定され、
  shutdown messageが送信されていない場合：
    - announcement_signatures messageを送信しなければならない。
      - funding_lockedが送受信され、かつfunding transactionが少なくとも6つの確認を持つまで、
      announcement_signatures messagesを送信してはいけない。

  - otherwise:
    - MUST NOT send the `announcement_signatures` message.

  - そうでなければ：
    - announcement_signatures messageを送信してはいけない。

  - upon reconnection (once the above timing requirements have been met):
    - MUST respond to the first `announcement_signatures` message with its own
    `announcement_signatures` message.
    - if it has NOT received an `announcement_signatures` message:
      - SHOULD retransmit the `announcement_signatures` message.

  - 再接続時（上記のタイミング要件が満たされると）：
  （XXX: TODO: 条件が足りない。クロスするケースがある。
  再接続時のannouncement_signatures messageの送信は一度だけという条件が必要）
    - 最初のannouncement_signatures messageに対して、それ自身のannouncement_signatures messageで応答しなければならない。
    （XXX: TODO: the firstというのは再接続後の最初。）
    - announcement_signatures messageを受信していない場合：
      - announcement_signatures messageを再送すべきである。

A recipient node:
  - if the `short_channel_id` is NOT correct:
    - SHOULD fail the channel.
  - if the `node_signature` OR the `bitcoin_signature` is NOT correct:
    - MAY fail the channel.
  - if it has sent AND received a valid `announcement_signatures` message:
    - SHOULD queue the `channel_announcement` message for its peers.
  - if it has not sent funding_locked:
    - MAY defer handling the announcement_signatures until after it has sent funding_locked
    - otherwise:
      - MUST ignore it.

A recipient node:
  - short_channel_idが正しくない場合:
    - channelに失敗すべきである。
  - node_signatureもしくはbitcoin_signatureが正しくない場合：
  （XXX: これらはchannel_announcementに入るもの）
    - channelを失敗させてよい。
  - 有効なannouncement_signatures messageを送受信した場合：
    - channel_announcement messageをそのpeersのためにキューイングすべきである。
    （XXX: ？）
  - funding_lockedを送信していない場合：
    - funding_lockedを送信した後まで、announcement_signaturesの処理を延期してもよい
  - そうでなければ：
      - それ（XXX: 受信したannouncement_signatures）を無視しなければならない。
      （XXX: なぜchannel失敗ではないのか？）

### Rationale

The reason for allowing deferring of a premature announcement_signatures is
that an earlier version of the spec did not require waiting for receipt of
funding locked: deferring rather than ignoring it allows compatibility with
this behavior.

時期尚早なannouncement_signaturesの延期を許可する理由は、
以前のバージョンの仕様は、funding lockedの受領を待つ必要がなく：
無視するのではなく延期することにより、この動作との互換性が可能になるからである。

## The `channel_announcement` Message

This gossip message contains ownership information regarding a channel. It ties
each on-chain Bitcoin key to the associated Lightning node key, and vice-versa.
The channel is not practically usable until at least one side has announced
its fee levels and expiry, using `channel_update`.

このgossip messageには、channelに関する所有権情報が含まれている。
それは各オンチェーンのBitcoin keyと関連するLightning node keyを結び付け、逆もまた同様である。
このchannelは、channel_updateを使用して少なくとも片側がfeeレベルとexpiryをannounceするまで、実際には使用できない。

Proving the existence of a channel between `node_1` and `node_2` requires:

node_1とnode_2間のchannelの存在証明が必要とするのは：

1. proving that the funding transaction pays to `bitcoin_key_1` and
   `bitcoin_key_2`
2. proving that `node_1` owns `bitcoin_key_1`
3. proving that `node_2` owns `bitcoin_key_2`

（区切り）

1. funding transactionがbitcoin_key_1とbitcoin_key_2に支払っていることの証明
2. node_1がbitcoin_key_1を所有していることの証明
3. node_2がbitcoin_key_2を所有していることの証明

Assuming that all nodes know the unspent transaction outputs, the first proof is
accomplished by a node finding the output given by the `short_channel_id` and
verifying that it is indeed a P2WSH funding transaction output for those keys
specified in [BOLT #3](03-transactions.md#funding-transaction-output).

全てのnodesがunspent transaction outputsを知っていると仮定すると、
nodeによる、short_channel_idから与えられるoutputの発見と、それが実際にBOLT #3で指定されたように、
それらのkeysのP2WSH funding transaction outputであることの確認により、最初の証明は達成される。

The last two proofs are accomplished through explicit signatures:
`bitcoin_signature_1` and `bitcoin_signature_2` are generated for each
`bitcoin_key` and each of the corresponding `node_id`s are signed.

最後の二つの証明は、明示的なsignaturesを介して達成される：
bitcoin_signature_1とbitcoin_signature_2が、
それぞれのbitcoin_keyとそれぞれの対応するnode_idが署名されることにより、
それぞれ生成される。
（XXX: bitcoin_keyは冗長？）

It's also necessary to prove that `node_1` and `node_2` both agree on the
announcement message: this is accomplished by having a signature from each
`node_id` (`node_signature_1` and `node_signature_2`) signing the message.

また、node_1とnode_2が共にannouncement messageに合意していることを証明する必要がある：
各node_idがmessageにしたsignature（node_signature_1とnode_signature_2）を持つことによって達成される。

1. type: 256 (`channel_announcement`)
2. data:
    * [`signature`:`node_signature_1`]
    * [`signature`:`node_signature_2`]
    * [`signature`:`bitcoin_signature_1`]
    * [`signature`:`bitcoin_signature_2`]
    * [`u16`:`len`]
    * [`len*byte`:`features`]
    * [`chain_hash`:`chain_hash`]
    * [`short_channel_id`:`short_channel_id`]
    * [`point`:`node_id_1`]
    * [`point`:`node_id_2`]
    * [`point`:`bitcoin_key_1`]
    * [`point`:`bitcoin_key_2`]

### Requirements

The origin node:

  - MUST set `chain_hash` to the 32-byte hash that uniquely identifies the chain
  that the channel was opened within:
    - for the _Bitcoin blockchain_:
      - MUST set `chain_hash` value (encoded in hex) equal to `6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000`.

  - chain_hashに、channelが開かれたチェーンを一意に識別する32バイトのハッシュを設定する必要がある：
    - _Bitcoin blockchain_ では：
      - chain_hash値（16進数で符号化された）を以下に等しく設定しなければなりません。
      6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000。

  - MUST set `short_channel_id` to refer to the confirmed funding transaction,
  as specified in [BOLT #2](02-peer-protocol.md#the-funding_locked-message).
    - Note: the corresponding output MUST be a P2WSH, as described in [BOLT #3](03-transactions.md#funding-transaction-output).

  - short_channel_idに、BOLT＃2で規定される確認済みのfunding transactionを参照するように設定しなければならない。
    - 注：対応するoutputは、BOLT＃3に記述されているように、P2WSHでなければならない。

  - MUST set `node_id_1` and `node_id_2` to the public keys of the two nodes
  operating the channel, such that `node_id_1` is the numerically-lesser of the
  two DER-encoded keys sorted in ascending numerical order.

  - node_id_1とnode_id_2に、channelを操作する2つのnodesの公開鍵を、
  node_id_1は、2つのDERエンコードされたkeysを数字の小さい順にソートし、数値的に小さいものになるように設定する。

  - MUST set `bitcoin_key_1` and `bitcoin_key_2` to `node_id_1` and `node_id_2`'s
  respective `funding_pubkey`s.

  - bitcoin_key_1とbitcoin_key_2に、node_id_1とnode_id_2のそれぞれのfunding_pubkeyを設定する。

  - MUST compute the double-SHA256 hash `h` of the message, beginning at offset
  256, up to the end of the message.
    - Note: the hash skips the 4 signatures but hashes the rest of the message,
    including any future fields appended to the end.

  - 位置256から始まり、messageの最後までのdouble-SHA256ハッシュhを計算しなければならない。
    - 注：ハッシュは4つのsignaturesをスキップするが、最後に追加される将来のフィールドも含め、残りのmessageをハッシュする。

  - MUST set `node_signature_1` and `node_signature_2` to valid
    signatures of the hash `h` (using `node_id_1` and `node_id_2`'s respective
    secrets).

  - node_signature_1とnode_signature_2に、
  （node_id_1とnode_id_2のそれぞれのsecretsを使って）ハッシュhの正しいsignaturesを設定しなければならない。

  - MUST set `bitcoin_signature_1` and `bitcoin_signature_2` to valid
  signatures of the hash `h` (using `bitcoin_key_1` and `bitcoin_key_2`'s
  respective secrets).

  - bitcoin_signature_1とbitcoin_signature_2に、
  （bitcoin_key_1とbitcoin_key_2のそれぞれのsecretsを使って）ハッシュhの正しいsignaturesを設定しならない。

  - SHOULD set `len` to the minimum length required to hold the `features` bits
  it sets.

  - lenに、featuresビットを保持するために必要な最小長を設定すべきである。

The receiving node:
  - MUST verify the integrity AND authenticity of the message by verifying the
  signatures.

  - signaturesを検証することによってmessageの完全性と真正性を検証しなければならない。

  - if there is an unknown even bit in the `features` field:
    - MUST NOT parse the remainder of the message.
    - MUST NOT add the channel to its local network view.
    - SHOULD NOT forward the announcement.

  - featuresフィールドに未知の偶数ビットがある場合：
    - messageの残りの部分を解析してはならない。
    - そのnodeのローカルネットワークビューにchannelを追加してはいけない。
    （XXX: 使用できない）
    - announcementを転送すべきでない。

  - if the `short_channel_id`'s output does NOT correspond to a P2WSH (using
    `bitcoin_key_1` and `bitcoin_key_2`, as specified in
    [BOLT #3](03-transactions.md#funding-transaction-output)) OR the output is
    spent:
    - MUST ignore the message.

  - short_channel_idのoutputがP2WSH
  （BOLT＃3で規定されているように、bitcoin_key_1とbitcoin_key_2を使用した）
  に対応していない場合、またはoutputが消費された場合：
    - messageを無視しなければならない。

  - if the specified `chain_hash` is unknown to the receiver:
    - MUST ignore the message.

  - 指定されたchain_hashを受信者が知らない場合：
    - messageを無視しなければならない。

  - otherwise:
  - そうでなければ：

    - if `bitcoin_signature_1`, `bitcoin_signature_2`, `node_signature_1` OR
    `node_signature_2` are invalid OR NOT correct:
      - SHOULD fail the connection.

    - bitcoin_signature_1、bitcoin_signature_2、node_signature_1もしくはnode_signature_2
    が無効または正しく無い場合。
      - 接続に失敗すべきである。

    - otherwise:
    - そうでなければ

      - if `node_id_1` OR `node_id_2` are blacklisted:
        - SHOULD ignore the message.

      - node_id_1かnode_id_2がブラックリストに載っている場合：
        - messageを無視すべきである。

      - otherwise:
        - if the transaction referred to was NOT previously announced as a
        channel:
          - SHOULD queue the message for rebroadcasting.
          - MAY choose NOT to for messages longer than the minimum expected
          length.

      - そうでなければ：
        - 参照されたtransactionが以前にchannelとしてannounceされているものではない場合：
          - rebroadcastingのためにmessagesをキューに入れるべきである。
          - 期待される長さの最小値より長いmessagesに対してはNOTを選択することができる。

      - if it has previously received a valid `channel_announcement`, for the
      same transaction, in the same block, but for a different `node_id_1` or
      `node_id_2`:
        - SHOULD blacklist the previous message's `node_id_1` and `node_id_2`,
        as well as this `node_id_1` and `node_id_2` AND forget any channels
        connected to them.

      - 以前に同じtransaction、同じブロック内ではあるが、
      別のnode_id_1かnode_id_2のものとして有効なchannel_announcementを受け取っていれば：
        - 前のmessageのnode_id_1とnode_id_2をブラックリストに載せ、
        このnode_id_1とnode_id_2も同様にし、それらに接続された任意のchannelsも忘れるべきだ。
        （XXX: 鍵が漏洩したと考えられる）

      - otherwise:
        - SHOULD store this `channel_announcement`.

      - そうでなければ：
        - このchannel_announcementを格納すべきである。

  - once its funding output has been spent OR reorganized out:
    - SHOULD forget a channel.

  - 一旦そのfunding outputが費やされたか、またはブロックチェーンのリオーガニゼーションが発生した場合：
    - channelを忘れるべきである。

### Rationale

Both nodes are required to sign to indicate they are willing to route other
payments via this channel (i.e. be part of the public network); requiring their
Bitcoin signatures proves that they control the channel.

両方のnodesは、このchannelを介して他の支払いをルーティングする意思があることを示すために署名する必要がある
（つまり、パブリックネットワークの一部であること）。
彼らのBitcoin signaturesを必要とすることは、彼らがchannelを制御していることを証明する。

The blacklisting of conflicting nodes disallows multiple different
announcements. Such conflicting announcements should never be broadcast by any
node, as this implies that keys have leaked.

競合するnodesをブラックリストに登録すると、複数の異なるannouncementsが許可されなくなる。
このような競合するannouncementsは、鍵が漏洩したことを意味するので、どのnodeによってもブロードキャストされるべきではない。

While channels should not be advertised before they are sufficiently deep, the
requirement against rebroadcasting only applies if the transaction has not moved
to a different block.

channelsは十分に深くなる前にadvertiseすべきではなく、
rebroadcastingに対する要件は、transactionが別のブロックに移動していない場合にのみ適用される。

In order to avoid storing excessively large messages, yet still allow for
reasonable future expansion, nodes are permitted to restrict rebroadcasting
(perhaps statistically).

過度に大きなmessagesを格納することを避けるために、将来の合理的な拡張を可能にするために、
nodesはrebroadcastingを制限する（おそらく統計的に）ことが許可されている。

New channel features are possible in the future: backwards compatible (or
optional) features will have _odd_ feature bits, while incompatible features
will have _even_ feature bits
(["It's OK to be odd!"](00-introduction.md#glossary-and-terminology-guide)).
Incompatible features will result in the announcement not being forwarded by
nodes that do not understand them.

将来的には新しいchannel機能が可能である：後方互換性のある（またはオプションの）機能は奇数のfeatureビットを持ち、
互換性のないfeatureは偶数のfeatureビットを持つ（It's OK to be odd!）。
互換性のない機能は、そのannouncementが、それらを理解していないnodesによって転送されない結果になる。

## The `node_announcement` Message

This gossip message allows a node to indicate extra data associated with it, in
addition to its public key. To avoid trivial denial of service attacks,
nodes not associated with an already known channel are ignored.

このgossip messageは、nodeが公開鍵に加えて、それに関連する追加のデータを示すことを可能にする。
DoS攻撃を回避するために、既知のchannelに関連付けられていないnodesは無視される。

1. type: 257 (`node_announcement`)
2. data:
   * [`signature`:`signature`]
   * [`u16`:`flen`]
   * [`flen*byte`:`features`]
   * [`u32`:`timestamp`]
   * [`point`:`node_id`]
   * [`3*byte`:`rgb_color`]
   * [`32*byte`:`alias`]
   * [`u16`:`addrlen`]
   * [`addrlen*byte`:`addresses`]

`timestamp` allows for the ordering of messages, in the case of multiple
announcements. `rgb_color` and `alias` allow intelligence services to assign
nodes colors like black and cool monikers like 'IRATEMONK' and 'WISTFULTOLL'.

複数のannounceの場合、timestampはmessagesの順序付けを可能にする。
rgb_colorとaliasは、インテリジェントなサービスがnodesに、黒のような色や
「IRATEMONK」や「WISTFULTOLL」のようなクールな愛称を割り当てることができるようにする。

`addresses` allows a node to announce its willingness to accept incoming network
connections: it contains a series of `address descriptor`s for connecting to the
node. The first byte describes the address type and is followed by the
appropriate number of bytes for that type.

addressesはnodeに、ネットワーク接続を受け入れる意思をannounceすることができ：
それはnodeに接続するための一連のaddress descriptorを含む。
最初のバイトはアドレスタイプを記述し、そのタイプの適切なバイト数が続く。

The following `address descriptor` types are defined:

   * `1`: ipv4; data = `[4:ipv4_addr][2:port]` (length 6)
   * `2`: ipv6; data = `[16:ipv6_addr][2:port]` (length 18)
   * `3`: Tor v2 onion service; data = `[10:onion_addr][2:port]` (length 12)
       * version 2 onion service addresses; Encodes an 80-bit, truncated `SHA-1`
       hash of a 1024-bit `RSA` public key for the onion service (a.k.a. Tor
       hidden service).
   * `4`: Tor v3 onion service; data = `[35:onion_addr][2:port]` (length 37)
       * version 3 ([prop224](https://gitweb.torproject.org/torspec.git/tree/proposals/224-rend-spec-ng.txt))
         onion service addresses; Encodes:
         `[32:32_byte_ed25519_pubkey] || [2:checksum] || [1:version]`, where
         `checksum = sha3(".onion checksum" | pubkey || version)[:2]`.

次のaddress descriptorタイプが定義されている：

   * 1：ipv4; データ = [4:ipv4_addr][2:port]（長さ6）
   * 2：ipv6; データ = [16:ipv6_addr][2:port]（長さ18）
   * 3：Tor v2 オニオンサービス; データ = [10:onion_addr][2:port]（長さ12）
       * バージョン2のオニオンサービスアドレス；
       オニオンサービス（別名、Tor hiddenサービス）用の1024ビットのRSA公開鍵のSHA-1ハッシュが切り捨てられた、80ビットをエンコードする。
   * 4：Tor v3 オニオンサービス; データ = [35:onion_addr][2:port]（長さ37）
       * バージョン3（prop224）オニオンサービスアドレス；
       以下のようにエンコードされる：
       [32:32_byte_ed25519_pubkey] || [2:checksum] || [1:version]、
       checksum = sha3(".onion checksum" | pubkey || version)[:2]。

### Requirements

The origin node:

  - MUST set `timestamp` to be greater than that of any previous
  `node_announcement` it has previously created.
    - MAY base it on a UNIX timestamp.

  - timestampは、以前に作成されたnode_announcementのものよりも大きくなるように設定しなければならない。
    - UNIX timestampに基づいてもよい。

  - MUST set `signature` to the signature of the double-SHA256 of the entire
  remaining packet after `signature` (using the key given by `node_id`).

  - signatureは、signatureの後のパケットの残り全体のdouble-SHA256のsignatureに設定する必要がある
  （node_idによって与えられた鍵を使用して）。

  - MAY set `alias` AND `rgb_color` to customize its appearance in maps and
  graphs.
    - Note: the first byte of `rgb_color` is the red value, the second byte is the
    green value, and the last byte is the blue value.

  - aliasとrgb_colorは、地図やグラフでその外観をカスタマイズするために設定してよい。
    - 注：rgb_colorの最初のバイトは赤の値、2番目のバイトは緑の値、最後のバイトは青の値である。

  - MUST set `alias` to a valid UTF-8 string, with any `alias` trailing-bytes
  equal to 0.

  - aliasは、有効なUTF-8文字列に設定する必要がある。末尾に0を指定する。

  - SHOULD fill `addresses` with an address descriptor for each public network
  address that expects incoming connections.

  - addressesは、接続を待つパブリックネットワークアドレスごとにaddress descriptorを記入すべきである。

  - MUST set `addrlen` to the number of bytes in `addresses`.

  - addrlenは、addressesのバイト数に設定する必要がある。

  - MUST place address descriptors in ascending order.

  - address descriptorを昇順に配置する必要がある。

  - SHOULD NOT place any zero-typed address descriptors anywhere.

  - ゼロタイプのaddress descriptorを任意の場所に配置すべきでない。

  - SHOULD use placement only for aligning fields that follow `addresses`.

  - addressesの次のフィールドを整列させるためだけに配置を使用すべきである。
  （XXX: ゼロタイプのこと？）

  - MUST NOT create a `type 1` OR `type 2` address descriptor with `port` equal
  to 0.

  - portが0に等しいtype 1かtype 2のaddress descriptorを作成してはならない。

  - SHOULD ensure `ipv4_addr` AND `ipv6_addr` are routable addresses.

  - ipv4_addrとipv6_addrはルーティング可能なアドレスであることを保証すべきである。

  - MUST NOT include more than one `address descriptor` of the same type.

  - 同じタイプのaddress descriptorを複数含んではならない。

  - SHOULD set `flen` to the minimum length required to hold the `features`
  bits it sets.

  - flenには、featuresビットを保持するために必要な最小長に設定すべきである。

The receiving node:
  - if `node_id` is NOT a valid compressed public key:
    - SHOULD fail the connection.
    - MUST NOT process the message further.

  - node_idが有効な圧縮された公開鍵でない場合：
    - 接続に失敗すべきである。
    - messageをさらに処理してはならない。

  - if `signature` is NOT a valid signature (using `node_id` of the
  double-SHA256 of the entire message following the `signature` field, including
any future fields appended to the end):
    - SHOULD fail the connection.
    - MUST NOT process the message further.

  - signatureが有効なsignatureではない場合
  （node_id（XXX: の秘密鍵）を使用して、最後に追加される将来のフィールドも含め、signatureフィールドの後に続くmessage全体のdouble-SHA256（XXX: への署名））。
    - 接続に失敗すべきである。
    - messageをさらに処理してはならない。

  - if `features` field contains _unknown even bits_:
    - MUST NOT parse the remainder of the message.
    - MAY discard the message altogether.
    - SHOULD NOT connect to the node.

  - featuresフィールドが未知の偶数ビットを含んでいる場合：
    - messageの残りの部分を解析してはならない。
    - messageを完全に破棄してもよい。
    - nodeに接続すべきでない。

  - MAY forward `node_announcement`s that contain an _unknown_ `features` _bit_,
  regardless of if it has parsed the announcement or not.

  - それ（XXX: node）がannouncementを解析したかどうかに関係なく、
  未知のfeaturesビット含むnode_announcementを転送することができる。

  - SHOULD ignore the first `address descriptor` that does NOT match the types
  defined above.

  - 上で定義したタイプと一致しない最初のaddress descriptorを無視すべきである。
  （XXX: the first？マッチしない最初のtypeが来たら、以降は全て無視するということであろう）

  - if `addrlen` is insufficient to hold the address descriptors of the
  known types:
    - SHOULD fail the connection.

  - addrlenが既知のタイプのaddress descriptorsを保持するには不十分な場合：
    - 接続に失敗すべきである。

  - if `port` is equal to 0:
    - SHOULD ignore `ipv6_addr` OR `ipv4_addr`.

  - portが0に等しい場合：
    - ipv6_addrもしくはipv4_addrを無視すべきである。

  - if `node_id` is NOT previously known from a `channel_announcement` message,
  OR if `timestamp` is NOT greater than the last-received `node_announcement`
  from this `node_id`:
    - SHOULD ignore the message.

  - node_idが過去にchannel_announcement messageから知らされていないか、
  またはtimestampが最後に受信したこのnode_idのnode_announcementよりも大きくない場合：
    - messageを無視すべきである。

  - otherwise:
    - if `timestamp` is greater than the last-received `node_announcement` from
    this `node_id`:
      - SHOULD queue the message for rebroadcasting.
      - MAY choose NOT to queue messages longer than the minimum expected length.

  - そうでなければ：
    - timestampが最後に受信したこのnode_idのnode_announcementよりも大きい場合：
      - rebroadcastingのためにmessagesをキューに入れるべきである。
      - 期待される最小の長さ以上のmessagesを待ち行列に入れないことを選択してもよい。

  - MAY use `rgb_color` AND `alias` to reference nodes in interfaces.
    - SHOULD insinuate their self-signed origins.

  - インタフェースにおいてrgb_colorかaliasを使用してのnodesを参照してよい。
    - それらの自己署名された起源に誘導すべきである。

### Rationale

New node features are possible in the future: backwards compatible (or
optional) ones will have _odd_ `feature` _bits_, incompatible ones will have
_even_ `feature` _bits_. These may be propagated by nodes even if they
cannot process the announcements themselves.

新しいnodeの機能は、将来的に可能である：
後方互換性（またはオプション）のものは奇数のfeatureビットをもつだろうし、
互換性のないものは偶数のfeatureビットを持つだろう。
これらは、nodes自身がannounceを処理できない場合であっても、nodesによって伝搬されることがある。

New address types may be added in the future; as address descriptors have
to be ordered in ascending order, unknown ones can be safely ignored.
Additional fields beyond `addresses` may also be added in the future—with
optional padding within `addresses`, if they require certain alignment.

新しいアドレスタイプが将来追加される可能性がある。
address descriptorは昇順に並べ替える必要があるため、未知のものは安全に無視できる。
将来addressesの向こうに追加のフィールドを追加することができる。
もしそれらが特定の位置合わせを必要とする場合は、addressesの中のオプショナルのパディングとともに。

### Security Considerations for Node Aliases

Node aliases are user-defined and provide a potential avenue for injection
attacks, both during the process of rendering and during persistence.

nodeエイリアスはユーザ定義であり、描画処理中と永続化中の両方で、注入攻撃の可能性を与える。

Node aliases should always be sanitized before being displayed in
HTML/Javascript contexts or any other dynamically interpreted rendering
frameworks. Similarly, consider using prepared statements, input validation,
and escaping to protect against injection vulnerabilities and persistence
engines that support SQL or other dynamically interpreted querying languages.

nodeエイリアスは、HTML/Javascriptコンテキストまたはその他の動的に解釈されるレンダリングフレームワークで表示される前に、
必ずサニタイズする必要がある。
同様に、SQLやその他の動的に解釈されるクエリ言語をサポートする永続性エンジンを、注入脆弱性から保護するために、
プリペアドステートメント、入力検証、エスケープを使用することを検討しなさい。

* [Stored and Reflected XSS Prevention](https://www.owasp.org/index.php/XSS_%28Cross_Site_Scripting%29_Prevention_Cheat_Sheet)
* [DOM-based XSS Prevention](https://www.owasp.org/index.php/DOM_based_XSS_Prevention_Cheat_Sheet)
* [SQL Injection Prevention](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet)

（XXX: 上記リンクのURL表記がMarkdownの表記とコンフリクトするので少し変更した）

Don't be like the school of [Little Bobby Tables](https://xkcd.com/327/).

Little Bobby Tablesの学校のようにしないでください。
（XXX: SQL Injectionの例。）

## The `channel_update` Message

After a channel has been initially announced, each side independently
announces the fees and minimum expiry delta it requires to relay HTLCs
through this channel. Each uses the 8-byte channel shortid that matches the
`channel_announcement` and the 1-bit `channel_flags` field to indicate which end of the
channel it's on (origin or final). A node can do this multiple times, in
order to change fees.

channelが最初にannounceされた後、
各側は、このchannelを介してHTLCを中継するために必要なfeeとminimum expiry deltaを独立して公表する。
それぞれはchannel_announcementに一致する8バイトのshortidと、
channelのどちらの終わり（起点または終点）を示すために1ビットのchannel_flagsフィールドを使用する。
nodeはこれを複数回行うことができ、feeを変更することができる。

Note that the `channel_update` gossip message is only useful in the context
of *relaying* payments, not *sending* payments. When making a payment
 `A` -> `B` -> `C` -> `D`, only the `channel_update`s related to channels
 `B` -> `C` (announced by `B`) and `C` -> `D` (announced by `C`) will
 come into play. When building the route, amounts and expiries for HTLCs need
 to be calculated backward from the destination to the source. The exact initial
 value for `amount_msat` and the minimal value for `cltv_expiry`, to be used for
 the last HTLC in the route, are provided in the payment request
 (see [BOLT #11](11-payment-encoding.md#tagged-fields)).

このchannel_update gossip messageが有用なのは、支払いを中継する（XXX: 受信側）コンテキストであり、
支払いを送信するコンテキストではないことに注意してください。
A -> B -> C -> Dの支払いを作るとき、
B -> C（Bによって発表）とC -> D （Cによって発表）のchannelに関係するchannel_updateのみが関わる。
（XXX: 要するにBとCが中継を行うときに関わる）
ルートを構築する際には、HTLCの金額と有効期限をdestinationからsourceまで逆方向に計算する必要がある。
（XXX: feeとcltv_expiry_deltaをそれぞれ足し合わせていく）
ルートの最後のHTLCに使用される正確なamount_msatのための初期値とcltv_expiryのための最小値は、支払い要求で提供される（BOLT＃11を参照）。
（XXX: 逆に、最後のルートのcltv_expiryはinvoiceの値に下駄を履かせる）

1. type: 258 (`channel_update`)
2. data:
    * [`signature`:`signature`]
    * [`chain_hash`:`chain_hash`]
    * [`short_channel_id`:`short_channel_id`]
    * [`u32`:`timestamp`]
    * [`byte`:`message_flags`]
    * [`byte`:`channel_flags`]
    * [`u16`:`cltv_expiry_delta`]
    * [`u64`:`htlc_minimum_msat`]
    * [`u32`:`fee_base_msat`]
    * [`u32`:`fee_proportional_millionths`]
    * [`u64`:`htlc_maximum_msat`] (option_channel_htlc_max)

The `channel_flags` bitfield is used to indicate the direction of the channel: it
identifies the node that this update originated from and signals various options
concerning the channel. The following table specifies the meaning of its
individual bits:

channel_flagsビットフィールドは、channelの方向を示すために使用される：
このビットフィールドは、このアップデートが発生したnodeを識別し、
channelに関するさまざまなオプションを通知する。
次の表は、個々のビットの意味を示している：

| Bit Position  | Name        | Meaning                          |
| ------------- | ----------- | -------------------------------- |
| 0             | `direction` | Direction this update refers to. |
| 1             | `disable`   | Disable the channel.             |

| ビット位置      | 名前         | 意味                             |
| ------------- | ----------- | -------------------------------- |
| 0             | `direction` | このアップデートが参照する方向        |
| 1             | `disable`   | channelを無効にする                 |

The `message_flags` bitfield is used to indicate the presence of optional
fields in the `channel_update` message:

message_flagsビットフィールドはchannel_updateメッセージのオプショナルフィールドの存在を示す：

| Bit Position  | Name                      | Field                            |
| ------------- | ------------------------- | -------------------------------- |
| 0             | `option_channel_htlc_max` | `htlc_maximum_msat`              |

Note that the `htlc_maximum_msat` field is static in the current
protocol over the life of the channel: it is *not* designed to be
indicative of real-time channel capacity in each direction, which
would be both a massive data leak and uselessly spam the network (it
takes an average of 30 seconds for gossip to propagate each hop).

htlc_maximum_msatフィールドは、現在のプロトコルではチャネルの寿命にわたって静的であることに注意すること：
各方向のリアルタイムなチャネル容量を示すようには設計されていない、
それは大量のデータ漏洩とネットワークの無駄なスパムになりうる（gossipが各hopを伝播する平均は30秒）。

The `node_id` for the signature verification is taken from the corresponding
`channel_announcement`: `node_id_1` if the least-significant bit of flags is 0
or `node_id_2` otherwise.

signature検証のためのnode_idは対応するchannel_announcementから取得される：
flagsの最下位ビットが0の場合はnode_id_1で、そうでない場合はnode_id_2。

### Requirements

The origin node:

  - MUST NOT send a created `channel_update` before `funding_locked` has been received.
  - MAY create a `channel_update` to communicate the channel parameters to the
  channel peer, even though the channel has not yet been announced (i.e. the
  `announce_channel` bit was not set).
    - MUST NOT forward such a `channel_update` to other peers, for privacy
    reasons.
    - Note: such a `channel_update`, one not preceded by a
    `channel_announcement`, is invalid to any other peer and would be discarded.

  - funding_lockedを受信する前に、作成したchannel_updateを送信してはならない。
  - channelがまだannounceされていなくてもchannel_updateを
  channel peerにchannelパラメータを伝えるために作成してよい（すなわち、announce_channelビットがセットされていなくても）。
    - プライバシーの理由から、そのようなchannel_updateを他のpeersに転送してはいけない。
    - 注：channel_announcementが前に無いようなchannel_updateは、他のpeerにとって無効であり破棄される。

  - MUST set `signature` to the signature of the double-SHA256 of the entire
  remaining packet after `signature`, using its own `node_id`.

  - signatureに、signatureの後の残りのパケット全体のdouble-SHA256に、
  それ自身のnode_idを使用したsignatureを、設定しなければならない。

  - MUST set `chain_hash` AND `short_channel_id` to match the 32-byte hash AND
  8-byte channel ID that uniquely identifies the channel specified in the
  `channel_announcement` message.

  - chain_hashとshort_channel_idに、
  32バイトのハッシュと8バイトのchannel IDに一致させるように設定する必要がある、
  channel_announcement messageに指定されたchannelを一意に識別する。

  - if the origin node is `node_id_1` in the message:
    - MUST set the `direction` bit of `channel_flags` to 0.

  - origin nodeがmessage内のnode_id_1である場合：
    - channel_flagsのdirectionビットを0に設定しなければならない。

  - otherwise:
    - MUST set the `direction` bit of `channel_flags` to 1.

  - そうでなければ：
    - channel_flagsのdirectionビットを1に設定しなければならない。

  - if the `htlc_maximum_msat` field is present:
	- MUST set the `option_channel_htlc_max` bit of `message_flags` to 1.
	- MUST set `htlc_maximum_msat` to the maximum value it will send through this channel for a single HTLC.
		- MUST set this to less than or equal to the channel capacity.
		- MUST set this to less than or equal to `max_htlc_value_in_flight_msat`
		  it received from the peer.
      - for channels with `chain_hash` identifying the Bitcoin blockchain:
        - MUST set this to less than 2^32.

  - htlc_maximum_msatフィールドが存在する場合：
    - message_flagsのoption_channel_htlc_maxビットを1に設定しなければならない。
    - htlc_maximum_msatを、1つのHTLCに対してこのチャネルを通じて送信される最大値に設定する必要がある。
      - チャネル容量以下に設定しなければならない。
      - ピアから受信したmax_htlc_value_in_flight_msat以下に設定しなければならない。
        - Bitcoinブロックチェーンを識別するchain_hashを持つチャンネルの場合：
          - これを2^32未満に設定しなければならない。

  - otherwise:
	- MUST set the `option_channel_htlc_max` bit of `message_flags` to 0.

  - そうでなければ：
    - message_flagsのoption_channel_htlc_maxビットを0に設定しなければならない。

  - MUST set bits in `channel_flags` and `message_flags `that are not assigned a meaning to 0.

  - 意味の割り当てられていないchannel_flagsとmessage_flagsのビットに0を設定しなければならない。

  - MAY create and send a `channel_update` with the `disable` bit set to 1, to
  signal a channel's temporary unavailability (e.g. due to a loss of
  connectivity) OR permanent unavailability (e.g. prior to an on-chain
  settlement).
    - MAY sent a subsequent `channel_update` with the `disable` bit set to 0 to
    re-enable the channel.

  - channelの一時的な利用不可能性（例えば、接続性の欠如による）または永久的な利用不可能性（例えば、オンチェーン上の確定前）を示すために、
  disableビットを1に設定してchannel_updateを作成して送信することができる。
    - channelを再びenableにするためにビットを0に設定して後続のchannel_updateを送って良い。

  - MUST set `timestamp` to greater than 0, AND to greater than any
  previously-sent `channel_update` for this `short_channel_id`.
    - SHOULD base `timestamp` on a UNIX timestamp.

  - timestampに、0より大きく、かつ、このshort_channel_idのために
  以前に送信されたchannel_updateより大きく設定しなくてはならない。
    - timestampをUNIX timestampベースにすべきである。

  - MUST set `cltv_expiry_delta` to the number of blocks it will subtract from
  an incoming HTLC's `cltv_expiry`.

  - cltv_expiry_deltaに、着信HTLCのcltv_expiryから減算するブロック数を設定する必要がある。

  - MUST set `htlc_minimum_msat` to the minimum HTLC value (in millisatoshi)
  that the channel peer will accept.

  - htlc_minimum_msatに、
  channel peerが受け入れる最小のHTLC値（millisatoshi単位で）を設定しなければならない。

  - MUST set `fee_base_msat` to the base fee (in millisatoshi) it will charge
  for any HTLC.

  - fee_base_msatに、任意のHTLCのために課金されるであろう基本fee（millisatoshi単位で）を設定しなければならない。

  - MUST set `fee_proportional_millionths` to the amount (in millionths of a
  satoshi) it will charge per transferred satoshi.
  - SHOULD NOT create redundant `channel_update`s

  - fee_proportional_millionthsに、転送されたsatoshi当たりに課金される金額（100万satoshi分）を設定しなければならない。
  - 冗長なchannel_updateを生成すべきでない

The receiving node:
  - if the `short_channel_id` does NOT match a previous `channel_announcement`,
  OR if the channel has been closed in the meantime:
    - MUST ignore `channel_update`s that do NOT correspond to one of its own
    channels.

  - short_channel_idが先立つchannel_announcementに一致しないか、channelがその間に閉じられている場合：
    - それ自身のchannelsの1つに対応しないchannel_updateを無視しなければならない。

  - SHOULD accept `channel_update`s for its own channels (even if non-public),
  in order to learn the associated origin nodes' forwarding parameters.

  - 関連するorigin nodesの転送パラメータを知るために、（たとえ非公開であっても）
  それ自身のchannelsに対してのchannel_updateを受け入れるべきである。

  - if `signature` is not a valid signature, using `node_id` of the
  double-SHA256 of the entire message following the `signature` field (including
  unknown fields following `fee_proportional_millionths`):
    - MUST NOT process the message further.
    - SHOULD fail the connection.

  - signatureが有効なsignatureではない場合
  （node_id（XXX: の秘密鍵）を使用して、fee_proportional_millionthsの次の未知のフィールドを含む、
  signatureフィールドの後に続くmessage全体のdouble-SHA256（XXX: への署名））。
    - messageをさらに処理してはならない。
    - 接続に失敗すべきである。

  - if the specified `chain_hash` value is unknown (meaning it isn't active on
  the specified chain):
    - MUST ignore the channel update.

  - 指定されたchain_hash値が不明な場合（指定されたチェーン上で（XXX: そのノードが）アクティブでないことを意味する）
    - channel updateを無視しなければならない。

  - if the `timestamp` is equal to the last-received `channel_update` for this
    `short_channel_id` AND `node_id`:
  - if the fields below `timestamp` differ:
    - MAY blacklist this `node_id`.
    - MAY forget all channels associated with it.
  - if the fields below `timestamp` are equal:
    - SHOULD ignore this message

  - timestampがこのshort_channel_idとnode_idの最後に受信したchannel_updateと等しい場合：
    - timestampの下のフィールドが異なる場合:
      - このnode_idをブラックリストに載せても良い。
      - 関連付けられているすべてのチャネルを忘れて良い。
    - timestampの下のフィールドが等しい場合:
      - このメッセージを無視すべきである

  - if `timestamp` is lower than that of the last-received
  `channel_update` for this `short_channel_id` AND for `node_id`:
    - SHOULD ignore the message.

  - timestampが、このshort_channel_idのnode_idの最後に受信したchannel_updateのtimestampよりも小さい場合:
    - このメッセージを無視すべきである

  - otherwise:
    - if the `timestamp` is unreasonably far in the future:
      - MAY discard the `channel_update`.
    - otherwise:
      - SHOULD queue the message for rebroadcasting.
      - MAY choose NOT to for messages longer than the minimum expected length.

  - そうでなければ：
    - timestampが不合理に未来に遠い場合は：
      - そのchannel_updateを捨てても良い。
    - そうでなければ：
      - rebroadcastingのためにmessagesをキューに入れるべきである。
      - 期待される最小の長さ以上のmessagesを待ち行列に入れないことを選択してもよい。

  - if the `option_channel_htlc_max` bit of `message_flags` is 0:
    - MUST consider `htlc_maximum_msat` not to be present.
  - otherwise:
    - if `htlc_maximum_msat` is not present or greater than channel capacity:
	  - MAY blacklist this `node_id`
	  - SHOULD ignore this channel during route considerations.
	- otherwise:
	  - SHOULD consider the `htlc_maximum_msat` when routing.

  - message_flagsのoption_channel_htlc_maxビットが0の場合：
    - htlc_maximum_msatは存在しないと考えなければならない。
  - そうでなければ：
    - htlc_maximum_msatが存在しないかチャネル容量よりも大きくない場合：
    （XXX: なぜここはこんなに厳しい？）
      - このnode_idをブラックリストに入れてよい
      - ルートの検討中にこのチャネルを無視すべきである。
    - そうでなければ：
      - htlc_maximum_msatをルーティングの際に考慮すべきである。

### Rationale

The `timestamp` field is used by nodes for pruning `channel_update`s that are
either too far in the future or have not been updated in two weeks; so it
makes sense to have it be a UNIX timestamp (i.e. seconds since UTC
1970-01-01). This cannot be a hard requirement, however, given the possible case
of two `channel_update`s within a single second.

このtimestampフィールドは、
未来にあまりにも遠すぎるかまたは2週間updateされていないchannel_updateのプルーニングのために、
nodesによって使用される。
UNIX timestamp（つまり、UTC 1970-01-01以降の秒数）にすることは理にかなっている。
1秒間に2つchannel_updateがあるとしても、これは厳しい要件ではありません。

It is assumed that more than one `channel_update` message changing the channel
parameters in the same second may be a DoS attempt, and therefore, the node responsible
for signing such messages may be blacklisted. However, a node may send a same
`channel_update` message with a different signature (changing the nonce in signature
signing), and hence fields apart from signature are checked to see if the channel
parameters have changed for the same timestamp. It is also important to note that
ECDSA signatures are malleable. So, an intermediate node who received the `channel_update`
message can rebroadcast it just by changing the `s` component of signature with `-s`.
This should however not result in the blacklist of the `node_id` from where
the message originated.

同じ秒でチャネルパラメータを変更する複数のchannel_updateメッセージはDoSの試みであると想定され
したがって、そのようなメッセージに署名する責任のあるノードはブラックリストに載せられる可能性がある。
しかし、ノードが異なる署名（署名のナンスの変更）で同じchannel_updateメッセージを送信する可能性があるため、
署名以外のフィールドがチェックされて、同じtimestampでチャネルパラメータが変更されたかどうかが確認される。
（XXX: 決定性、RFC6979ではないかもしれない）
ECDSAの署名は展性があることにも留意することが重要である。
したがって、channel_updateメッセージを受信した中間ノードは、-sで署名のsコンポーネントを変更するだけで、
メッセージを再ブロードキャストできる。
ただし、これによって、メッセージの発信元であるnode_idがブラックリストに載せられることはない。

The explicit `option_channel_htlc_max` flag to indicate the presence
of `htlc_maximum_msat` (rather than having `htlc_maximum_msat` implied
by the message length) allows us to extend the `channel_update`
with different fields in future.  Since channels are limited to 2^32-1
millisatoshis in Bitcoin, the `htlc_maximum_msat` has the same restriction.

htlc_maximum_msatの存在を示すための明示的なoption_channel_htlc_maxフラグ
（メッセージの長さでhtlc_maximum_msatを暗示的に持つのではなく）は、
channel_updateを将来異なるフィールドで拡張することを可能にする。
Bitcoinではチャンネルが2^32-1 millisatoshis に制限されているので、
htlc_maximum_msatには同じ制限がある。

The recommendation against redundant `channel_update`s minimizes spamming the network,
however it is sometimes inevitable.  For example, a channel with a
peer which is unreachable will eventually cause a `channel_update` to
indicate that the channel is disabled, with another update re-enabling
the channel when the peer reestablishes contact.  Because gossip
messages are batched and replace previous ones, the result may be a
single seemingly-redundant update.

冗長なchannel_updateに対する推奨は、ネットワークのスパムを最小限に抑えるが、時には不可避である。
たとえば、到達不能なピアを持つチャネルは、結局はチャネルが無効であることを示すchannel_updateを生成し、
ピアが再接続したときのチャネルの再有効化の更新も伴う。
gossipメッセージはバッチ処理されて前のバージョンと置き換えられるため、
一見重複した更新が1つの結果になる可能性がある。

## Query Messages

Negotiating the `gossip_queries` option via `init` enables a number
of extended queries for gossip synchronization.  These explicitly
request what gossip should be received.

initを通して、このgossip_queriesオプションをネゴシエートすることにより、gossip同期のための多数の拡張クエリが可能になる。
これらは、どのようなgossipを受け取るべきかを明示的に要求する。

There are several messages which contain a long array of
`short_channel_id`s (called `encoded_short_ids`) so we utilize a
simple compression scheme: the first byte indicates the encoding, the
rest contains the data.

short_channel_idsの長い配列（encoded_short_idsと呼ばれる）を含むいくつかのmessagesがあり、
簡単な圧縮スキームを使用する。
最初のバイトはエンコーディングを示し、残りはデータを含む。

Encoding types:
* `0`: uncompressed array of `short_channel_id` types, in ascending order.
* `1`: array of `short_channel_id` types, in ascending order, compressed with zlib deflate<sup>[1](#reference-1)</sup>

エンコードタイプ：
* 0：short_channel_idタイプの非圧縮配列、昇順。
* 1：short_channel_idタイプの配列、昇順、zlib deflateで圧縮

This encoding is also used for arrays of other types (timestamps, flags, ...), and specified with an `encoded_` prefix. For example, `encoded_timestamps` is an array of timestamps than can be either compressed (with a `1` prefix) or uncompressed (with a `0` prefix).

このエンコーディングは、他の型（タイムスタンプ、フラグ、...）の配列にも使用され、encoded_prefixで指定される。
たとえば、encoded_timestampsはタイムスタンプの配列で、圧縮（先頭に1を付けて）
または非圧縮（先頭に0を付けて）のいずれかにできる。

Note that a 65535-byte zlib message can decompress into 67632120
bytes<sup>[2](#reference-2)</sup>, but since the only valid contents
are unique 8-byte values, no more than 14 bytes can be duplicated
across the stream: as each duplicate takes at least 2 bits, no valid
contents could decompress to more than 3669960 bytes.

注：65535バイトのzlib messageは67632120バイトに展開されるが、有効な内容は一意の8バイト値だけなので、
ストリームを通して重複するのは14バイト未満である：
各重複は少なくとも2ビットを用し、有効なコンテンツは3669960バイト以上に展開されることはない。
（XXX: short_channel_idは8バイト）

Query messages can be extended with optional fields that can help reduce the number of messages needed to synchronize routing tables by enabling:

クエリメッセージは、オプションフィールドを使用して拡張できる。
オプションフィールドを使用すると、次の機能を有効にすることで、ルーティングテーブルの同期に必要なメッセージ数を減らすことができる。

- timestamp-based filtering of `channel_update` messages: only ask for `channel_update` messages that are newer than the ones you already have.
- checksum-based filtering of `channel_update` messages: only ask for `channel_update` messages that carry different information from the ones you already have.

- タイムスタンプベースのchannel_updateメッセージのフィルタリング：
既存のメッセージよりも新しいchannel_updateメッセージだけを要求する。
（XXX: そもそもタイムスタンプベースではなかったか）
- channel_updateメッセージのチェックサムに基づくフィルタリング：
既に持っている情報とは異なる情報を持つchannel_updateメッセージだけを要求する。

Nodes can signal that they support extended gossip queries with the `gossip_queries_ex` feature bit.

ノードは、gossip_queries_ex機能ビットを使用して、拡張ゴシップクエリをサポートしていることを示すことができる。

### The `query_short_channel_ids`/`reply_short_channel_ids_end` Messages

1. type: 261 (`query_short_channel_ids`) (`gossip_queries`)
2. data:
    * [`chain_hash`:`chain_hash`]
    * [`u16`:`len`]
    * [`len*byte`:`encoded_short_ids`]
    * [`query_short_channel_ids_tlvs`:`tlvs`]

1. tlvs: `query_short_channel_ids_tlvs`
2. types:
    1. type: 1 (`query_flags`)
    2. data:
        * [`u8`:`encoding_type`]
        * [`...*byte`:`encoded_query_flags`]

（XXX: たぶん思った通りの番号になっていない）<br>

`encoded_query_flags` is an array of bitfields, one varint per bitfield, one bitfield for each `short_channel_id`. Bits have the following meaning:

encoded_query_flagsはビットフィールドの配列で、
ビットフィールドごとに1つのvarint、
short_channel_idごとに1つのビットフィールドがある。
ビットの意味は次のとおり。

| Bit Position  | Meaning                                  |
| ------------- | ---------------------------------------- |
| 0             | Sender wants `channel_announcement`      |
| 1             | Sender wants `channel_update` for node 1 |
| 2             | Sender wants `channel_update` for node 2 |
| 3             | Sender wants `node_announcement` for node 1 |
| 4             | Sender wants `node_announcement` for node 2 |

| Bit Position  | Meaning                                  |
| ------------- | ---------------------------------------- |
| 0             | 送信者は `channel_announcement` を望む        |
| 1             | 送信者は mode 1 の `channel_update` を望む    |
| 2             | 送信者は node 2 の `channel_update` を望む    |
| 3             | 送信者は node 1 の `node_announcement` を望む |
| 4             | 送信者は node 2 の `node_announcement` を望む |

Query flags must be minimally encoded, which means that one flag will be encoded with a single byte.

Query flagsは最小限に符号化されなければならない。
これは、1つのフラグが1バイトで符号化されることを意味する。<br>
（XXX: varintだと<= 0xfcだとそのままの値）

1. type: 262 (`reply_short_channel_ids_end`) (`gossip_queries`)
2. data:
    * [`chain_hash`:`chain_hash`]
    * [`byte`:`complete`]

This is a general mechanism which lets a node query for the
`channel_announcement` and `channel_update` messages for specific channels
(identified via `short_channel_id`s). This is usually used either because
a node sees a `channel_update` for which it has no `channel_announcement` or
because it has obtained previously unknown `short_channel_id`s
from `reply_channel_range`.

これはノードが特定のchannels（short_channel_idで同定される）の
channel_announcementとchannel_updateを問い合わせるための一般的なメカニズムである；
これは通常、channel_announcementのないchannel_updateを見る場合か
（XXX: channel_updateだけ受け取っていて、channel_announcementが欲しい。
逆にchannel_announcementだけ受け取っているケースは考えられないのか？
新しいデータだけ持っているということか）、
またはreply_channel_rangeから以前に未知のshort_channel_idを取得しているためである。

#### Requirements

The sender:

  - MUST NOT send `query_short_channel_ids` if it has sent a previous `query_short_channel_ids` to this peer and not received `reply_short_channel_ids_end`.

  - このpeerに前回query_short_channel_idsを送信し、
  reply_short_channel_ids_endを受信していない場合は、
  query_short_channel_idsを送信してはならない。

  - MUST set `chain_hash` to the 32-byte hash that uniquely identifies the chain
  that the `short_channel_id`s refer to.

  - chain_hashに、short_channel_idが参照するチェーンを一意に識別する32バイトのハッシュを設定しなければならない。

  - MUST set the first byte of `encoded_short_ids` to the encoding type.

  - encoded_short_idsの最初のバイトを符号化タイプに設定しなければならない。

  - MUST encode a whole number of `short_channel_id`s to `encoded_short_ids`

  - encoded_short_idsに、short_channel_idの全数をエンコードしなければならない

  - MAY send this if it receives a `channel_update` for a
   `short_channel_id` for which it has no `channel_announcement`.

  - channel_announcementを持たないshort_channel_idのchannel_updateを受信した場合は、それを送ってよい。
  （XXX: encoded_short_idsにshort_channel_idを含めて送って良い）

  - SHOULD NOT send this if the channel referred to is not an unspent output.

  - 参照されたchannelがUTXOでない場合、これを送信すべきではない。
  （XXX: encoded_short_idsにshort_channel_idを含めて送ってはいけない）

  - MAY include an optional `query_flags`. If so:
    - MUST set `encoding_type`, as for `encoded_short_ids`.
    - Each query flag is a minimally-encoded varint.
    - MUST encode one query flag per `short_channel_id`.

  - オプションのquery_flagsを含めてもよい。その場合：
    - encoded_short_idsと同様に、encoding_typeを設定しなければならない。
    - 各query flagは最小符号化されたvarintである。
    - short_channel_idごとに1つのquery flagをエンコードしなければならない。  

The receiver:

  - if the first byte of `encoded_short_ids` is not a known encoding type:
    - MAY fail the connection

  - encoded_short_idsの最初のバイトが既知のエンコーディングタイプでない場合：
    - 接続に失敗して良い

  - if `encoded_short_ids` does not decode into a whole number of `short_channel_id`:
    - MAY fail the connection.

  - encoded_short_idsをshort_channel_idの全数にデコードしない場合：
  （XXX: encoded_short_idsはshort_channel_id8バイトの配列なので、
  デコードしたときにそのようなものに展開できなかったらということであろう）
    - 接続に失敗して良い。

  - if it has not sent `reply_short_channel_ids_end` to a previously received `query_short_channel_ids` from this sender:
    - MAY fail the connection.

  - この送信者から前に受信したquery_short_channel_idに対してreply_short_channel_ids_endを送信していない場合：
    - 接続に失敗して良い。    

  - if the incoming message includes `query_short_channel_ids_tlvs`:
    - if `encoding_type` is not a known encoding type:
      - MAY fail the connection
    - if `encoded_query_flags` does not decode to exactly one flag per `short_channel_id`:
      - MAY fail the connection.

  - 着信メッセージにquery_short_channel_ids_tlvsが含まれる場合：
    - encoding_typeが既知の符号化タイプでない場合：
      - 接続に失敗して良い。
    - encoded_query_flagsがshort_channel_idごとに1つのフラグにデコードされない場合：
      - 接続に失敗して良い

  - MUST respond to each known `short_channel_id`:
    - if the incoming message does not include `encoded_query_flags`:
      - with a `channel_announcement` and the latest `channel_update` for each end
      - MUST follow with any `node_announcement`s for each `channel_announcement`
    - otherwise:
      - We define `query_flag` for the Nth `short_channel_id` in
        `encoded_short_ids` to be the Nth varint of the decoded
        `encoded_query_flags`.
      - if bit 0 of `query_flag` is set:
        - MUST reply with a `channel_announcement`
      - if bit 1 of `query_flag` is set and it has received a `channel_update` from `node_id_1`:
        - MUST reply with the latest `channel_update` for `node_id_1`
      - if bit 2 of `query_flag` is set and it has received a `channel_update` from `node_id_2`:

        - MUST reply with the latest `channel_update` for `node_id_2`
      - if bit 3 of `query_flag` is set and it has received a `node_announcement` from `node_id_1`:
        - MUST reply with the latest `node_announcement` for `node_id_1`
      - if bit 4 of `query_flag` is set and it has received a `node_announcement` from `node_id_2`:
        - MUST reply with the latest `node_announcement` for `node_id_2`

  - それぞれの既知のshort_channel_idに応答しなければならない：
    - 着信メッセージにencoded_query_flagsが含まれていない場合：
      - channel_announcementと各端の最新のchannel_updateを以って
      - channel_announcementごとにnode_announcementを続けなければならない
    - そうでなければ:
      - encoded_short_idsのN番目のshort_channel_idのquery_flagを、デコードされたencoded_query_flagsのN番目のvarintとして定義する。
      - query_flagのビット0が設定されている場合：
        - channel_announcementで応答しなければならない。（XXX: これも受信している場合、という条件が必要）
      - query_flagのビット1が設定され、node_id_1からchannel_updateを受信している場合：
        - node_id_1の最新のchannel_updateで応答しなければならない。
      - query_flagのビット2が設定され、node_id_2からchannel_updateを受信している場合：
        - node_id_2の最新のchannel_updateで応答しなければならない。
      - query_flagのビット3が設定され、node_id_1からnode_announcementを受信している場合：
        - node_id_1の最新のnode_announcementで応答しなければならない。
      - query_flagのビット4が設定され、node_id_2からnode_announcementを受信している場合：
        - node_id_2の最新のnode_announcementで応答しなければならない。

	- SHOULD NOT wait for the next outgoing gossip flush to send these.

  - これらを送信するために次の発信gossipのフラッシュを待つべきではない。
  (XXX: すぐに応答すべき)

  - SHOULD avoid sending duplicate `node_announcements` in response to a single `query_short_channel_ids`.

  - 単一のquery_short_channel_idsに応答して、重複したnode_announcementを送信することを避けるべきである。

  - MUST follow these responses with `reply_short_channel_ids_end`.

  - これらの応答に続けてreply_short_channel_ids_endを送らなければならない。

  - if does not maintain up-to-date channel information for `chain_hash`:
	  - MUST set `complete` to 0.

  - chain_hashの最新のチャンネル情報を保持しない場合：
  （XXX: Rationale参照）
    - completeに、0を設定すべきである。

  - otherwise:
	  - SHOULD set `complete` to 1.

  - そうでなければ：
    - completeに、1を設定すべきである。

#### Rationale

Future nodes may not have complete information; they certainly won't have
complete information on unknown `chain_hash` chains.  While this `complete`
field cannot be trusted, a 0 does indicate that the sender should search
elsewhere for additional data.

将来のnodesには完全な情報がないかもしれない：
彼らは確かに未知のchain_hashチェーンに関する完全な情報を持っていないであろう。
このcompleteフィールドは信頼できないが、0は送信者が追加のデータを他の場所で検索する必要があることを示している。

The explicit `reply_short_channel_ids_end` message means that the receiver can
indicate it doesn't know anything, and the sender doesn't need to rely on
timeouts.  It also causes a natural rate limiting of queries.

明示的なreply_short_channel_ids_end messageは、受信者が何も知らないことを示すことができ、
送信者はタイムアウトに頼る必要がないことを意味する。それはまた、クエリの自然なレート制限をもたらす。（XXX: reply_short_channel_ids_endが来るまで次のクエリは行えない）

### The `query_channel_range` and `reply_channel_range` Messages

1. type: 263 (`query_channel_range`) (`gossip_queries`)
2. data:
    * [`chain_hash`:`chain_hash`]
    * [`u32`:`first_blocknum`]
    * [`u32`:`number_of_blocks`]
    * [`query_channel_range_tlvs`:`tlvs`]

1. tlvs: `query_channel_range_tlvs`
2. types:
    1. type: 1 (`query_option`)
    2. data:
        * [`varint`:`query_option_flags`]

`query_option_flags` is a bitfield represented as a minimally-encoded varint. Bits have the following meaning:

query_option_flagsは、最小限にエンコードされたvarintで表されるビットフィールドである。
ビットの意味は次のとおり：

| Bit Position  | Meaning                 |
| ------------- | ----------------------- |
| 0             | Sender wants timestamps |
| 1             | Sender wants checksums  |

Though it is possible, it would not be very useful to ask for checksums without asking for timestamps too: the receiving node may have an older `channel_update` with a different checksum, asking for it would be useless. And if a `channel_update` checksum is actually 0 (which is quite unlikely) it will not be queried.

それは可能ですが、タイムスタンプを要求せずにチェックサムを要求することはあまり有用ではない：
受信ノードが異なるチェックサムを持つ古いchannel_updateを持つかもしれず、それを要求することは役に立たないだろう。
また、channel_updateのチェックサムが実際には0（それはとてもありそうにない）の場合、クエリは行われない。
（XXX: これはなに？）

1. type: 264 (`reply_channel_range`) (`gossip_queries`)
2. data:
    * [`chain_hash`:`chain_hash`]
    * [`u32`:`first_blocknum`]
    * [`u32`:`number_of_blocks`]
    * [`byte`:`complete`]
    * [`u16`:`len`]
    * [`len*byte`:`encoded_short_ids`]
    * [`reply_channel_range_tlvs`:`tlvs`]

1. tlvs: `reply_channel_range_tlvs`
2. types:
    1. type: 1 (`timestamps_tlv`)
    2. data:
	    * [`u8`:`encoding_type`]
        * [`...*byte`:`encoded_timestamps`]
    1. type: 3 (`checksums_tlv`)
    2. data:
        * [`...*channel_update_checksums`:`checksums`]

For a single `channel_update`, timestamps are encoded as:

channel_updateが1つの場合、タイムスタンプは次のようにエンコードされる：
（XXX: ２つの場合は？）

1. subtype: `channel_update_timestamps`
2. data:
    * [`u32`:`timestamp_node_id_1`]
    * [`u32`:`timestamp_node_id_2`]

（XXX:
[`...*byte`:`encoded_timestamps`]
これは、
[`...*channel_update_timestamps`:`encoded_timestamps`]
と書くべき？
)

Where:
* `timestamp_node_id_1` is the timestamp of the `channel_update` for `node_id_1`, or 0 if there was no `channel_update` from that node.
* `timestamp_node_id_2` is the timestamp of the `channel_update` for `node_id_2`, or 0 if there was no `channel_update` from that node.

ここで：
* timestamp_node_id_1はnode_id_1のchannel_updateのタイムスタンプで、そのノードからchannel_updateがなかった場合は0である。
* timestamp_node_id_2はnode_id_2のchannel_updateのタイムスタンプで、そのノードからchannel_updateがなかった場合は0である。

For a single `channel_update`, checksums are encoded as:

channel_updateが1つの場合、チェックサムは次のようにエンコードされる：
（XXX: ２つの場合は？）

1. subtype: `channel_update_checksums`
2. data:
    * [`u32`:`checksum_node_id_1`]
    * [`u32`:`checksum_node_id_2`]

Where:
* `checksum_node_id_1` is the checksum of the `channel_update` for `node_id_1`, or 0 if there was no `channel_update` from that node.
* `checksum_node_id_2` is the checksum of the `channel_update` for `node_id_2`, or 0 if there was no `channel_update` from that node.

ここで：
* checksum_node_id_1はnode_id_1のchannel_updateのチェックサムで、そのノードからchannel_updateがなかった場合は0である。
* checksum_node_id_2はnode_id_2のchannel_updateのチェックサムで、そのノードからchannel_updateがなかった場合は0である。

The checksum of a `channel_update` is the CRC32C checksum as specified in [RFC3720](https://tools.ietf.org/html/rfc3720#appendix-B.4) of this `channel_update` without its `signature` and `timestamp` fields.

channel_updateのチェックサムは、RFC3720で指定されているCRC32Cチェックサムで、
このchannel_updateの署名フィールドとタイムスタンプフィールドを除く。

This allows to query for channels within specific blocks.

これにより、特定のブロック内のchannelsのクエリが可能になる。

#### Requirements

The sender of `query_channel_range`:

  - MUST NOT send this if it has sent a previous `query_channel_range` to this peer and not received all `reply_channel_range` replies.

  - 前のquery_channel_rangeがこのpeerに送られ、全てのreply_channel_range応答を受け取っていないなら、これを送ってはいけない。
  （XXX: reply_channel_rangeは複数回に分けて全部送る）

  - MUST set `chain_hash` to the 32-byte hash that uniquely identifies the chain
  that it wants the `reply_channel_range` to refer to

  - chain_hashに、それが必要とするreply_channel_rangeが参照するチェーンを一意に識別する32バイトのハッシュに設定しなければならない

  - MUST set `first_blocknum` to the first block it wants to know channels for

  - first_blocknumは、channlesを知りたい最初のブロックを設定しなければならない

  - MUST set `number_of_blocks` to 1 or greater.

  - number_of_blocksは、1以上に設定しなければならない。

  - MAY append an additional `query_channel_range_tlv`, which specifies the type of extended information it would like to receive.  

  - 受信したい拡張情報のタイプを指定するquery_channel_range_tlvを追加してよい。

The receiver of `query_channel_range`:

  - if it has not sent all `reply_channel_range` to a previously received `query_channel_range` from this sender:
    - MAY fail the connection.

  - この送信者からの前に受信したquery_channel_rangeに対する全てのreply_channel_rangeを送信していない場合：
    - 接続に失敗して良い。

  - MUST respond with one or more `reply_channel_range` whose combined range
	cover the requested `first_blocknum` to `first_blocknum` plus
	`number_of_blocks` minus one.

  - リクエストされたfirst_blocknumから first_blocknum + number_of_blocks - 1 を、
  その組み合わせられた範囲がカバーする、
  1つ以上のreply_channel_rangeで応答しなければならない。

  - For each `reply_channel_range`:
    - MUST set with `chain_hash` equal to that of `query_channel_range`,
    - MUST encode a `short_channel_id` for every open channel it knows in blocks `first_blocknum` to `first_blocknum` plus `number_of_blocks` minus one.
    - MUST limit `number_of_blocks` to the maximum number of blocks whose
      results could fit in `encoded_short_ids`
    - if does not maintain up-to-date channel information for `chain_hash`:
      - MUST set `complete` to 0.
    - otherwise:
      - SHOULD set `complete` to 1.

  - それぞれのreply_channel_rangeついて：
    - chain_hashに、query_channel_rangeのそれと同じに設定しなければならない。
    - short_channel_idに、first_blocknumからfirst_blocknum + number_of_blocks - 1のブロックの中で知っている、
    全てのオープンなchannelをエンコードしなければならない。
    - number_of_blocksを、ブロックの結果（XXX: ブロックに含まれるshort_channel_id）がencoded_short_idsに収まるように、
    ブロックの最大数に制限しなければならない
    （XXX: channelが存在しないブロックもあることを考慮する。例えば、<br>
    block=998にはchannelがある<br>
    block=999にはない<br>
    block=1000にはchannelがあるがそれを加えるとreply_channel_rangeメッセージに収まらない<br>
    このときnumber_of_blocksには998までではなく最大の999まで入れた値にする。
    block=1000は次のreply_channel_rangeメッセージで送る。
    そうすると最後のreply_channel_rangeは必ずfirst_blocknum + number_of_blocks - 1までのnumber_of_blocksとなり最後がわかる。もし最後の方のブロックにchannelがなくても。
    ）
    - chain_hashの最新のチャンネル情報を保持しない場合：
      - completeに、0を設定すべきである。
    - そうでなければ：
      - completeに、1を設定すべきである。

If the incoming message includes `query_option`, the receiver MAY append additional information to its reply:
- if bit 0 in `query_option_flags` is set, the receiver MAY append a `timestamps_tlv` that contains `channel_update` timestamps for all `short_chanel_id`s in `encoded_short_ids`
- if bit 1 in `query_option_flags` is set, the receiver MAY append a `checksums_tlv` that contains `channel_update` checksums for all `short_chanel_id`s in `encoded_short_ids`

もし着信メッセージがquery_optionを含んでいるなら、受信者は応答に追加情報を追加してもよい：
- query_option_flagsのビット0が設定されている場合、受信者はencoded_short_idsのすべてのshort_chanel_idsのchannel_updateタイムスタンプを含むtimestamps_tlvを追加してもよい。
- query_option_flagsのビット1がセットされている場合、受信者はencoded_short_idsのすべてのshort_chanel_idsのchannel_updateチェックサムを含むchecksums_tlvを追加してもよい。

#### Rationale

A single response might be too large for a single packet, and also a peer can
store canned results for (say) 1000-block ranges, and simply offer each reply
which overlaps the ranges of the request.

1つのパケットに対して1つの応答が大きすぎる可能性があるので、
peerは1000ブロックの範囲で封じた結果を格納し、
単純に要求の範囲と重複する各応答を提供する可能性がある。（XXX: say？）
（XXX: TODO: 最初と末尾でrangeをオーバーしてもいいのか？だめだろう？）
（XXX: gossip_timestamp_filterがあるため複数のノードに対して応答の使い回しはできない？）

The addition of timestamp and checksum fields allow a peer to omit querying for redundant updates.

タイムスタンプフィールドとチェックサムフィールドの追加により、ピアは冗長な更新のクエリを省略できる。

### The `gossip_timestamp_filter` Message

1. type: 265 (`gossip_timestamp_filter`) (`gossip_queries`)
2. data:
    * [`chain_hash`:`chain_hash`]
    * [`u32`:`first_timestamp`]
    * [`u32`:`timestamp_range`]

This message allows a node to constrain future gossip messages to
a specific range.  A node which wants any gossip messages would have
to send this, otherwise `gossip_queries` negotiation means no gossip
messages would be received.

このmessageは、nodeが将来のgossip messagesを特定の範囲に制限することを可能にする。
gossip messagesを必要とするnodeは、これを送信しなければならず、
さもなければ、gossip_queriesネゴシエーションは、gossip messagesが受信されないことを意味する。
（XXX: TODO: このmessageはgossip messagesを受信する場合にはMUST？
「将来の」というのはどういうことか？Rebroadcastingだけか？全てか？）

Note that this filter replaces any previous one, so it can be used
multiple times to change the gossip from a peer.

注：このフィルタは以前のフィルタを置き換えるので、複数回使用してpeerからのgossipを変更することができる。
（XXX: これがクエリの応答の途中で行われた場合、どこから有効になるのか？
まあ次からだろう）

#### Requirements

The sender:

  - MUST set `chain_hash` to the 32-byte hash that uniquely identifies the chain
  that it wants the gossip to refer to.

  - chain_hashは、gossipが参照するチェーンを一意に識別する32バイトのハッシュに設定しなければならない。

The receiver:

  - SHOULD send all gossip messages whose `timestamp` is greater or
    equal to `first_timestamp`, and less than `first_timestamp` plus
    `timestamp_range`.

  - timestampがfirst_timestamp以上でかつfirst_timestamp + timestamp_range未満の全てのgossip messagesを送るべきである。
  。

	- MAY wait for the next outgoing gossip flush to send these.

  - これら（XXX: gossip messages）を送るために次の送信gossipフラッシュを待ってよい。
  (XXX: これはRebroadcastingにも適用されるのであろう。
  gossip_timestamp_filterを受信したタイミングで送る必要はない。
  timestampが変わると新たに送られる条件になるものが存在する可能性がある）

  - SHOULD restrict future gossip messages to those whose `timestamp`
    is greater or equal to `first_timestamp`, and less than
    `first_timestamp` plus `timestamp_range`.

  - 将来のgossip messagesをそれらのtimestampがfirst_timestamp以上でかつfirst_timestamp + timestamp_range未満に制限すべきである。

  - If a `channel_announcement` has no corresponding `channel_update`s:
    - MUST NOT send the `channel_announcement`.

  - channel_announcementに対応するchannel_updatesがない場合：
    - channel_announcementを送信してはいけない。

  - Otherwise:
	  - MUST consider the `timestamp` of the `channel_announcement` to be the `timestamp` of a corresponding `channel_update`.
	  - MUST consider whether to send the `channel_announcement` after receiving the first corresponding `channel_update`.

  - そうでなければ：
    - channel_announcementのtimestampは、対応するのchannel_updateのtimestampであると見なさなければならない。
    （XXX: TODO: ２つあるけどどうするの？）
    - channel_announcementを送信するかどうかは、最初の対応するchannel_updateを受信した後に考えなければならない。

  - If a `channel_announcement` is sent:
	  - MUST send the `channel_announcement` prior to any corresponding `channel_update`s and `node_announcement`s.

  - channel_announcementが送信される場合：
    - 対応するchannel_updateとnode_announcementに先立ってchannel_announcementを送らなければならない。

#### Rationale

Since `channel_announcement` doesn't have a timestamp, we generate a likely
one.  If there's no `channel_update` then it is not sent at all, which is most
likely in the case of pruned channels.

channel_announcementにはtimestampがないので、可能性のあるものを生成する。
channel_updateがない場合はまったく送信されないが、これは刈り取られたchannelsの場合に最も可能性が高い。
（XXX: どのような理由でchannel_updateが刈り取られたのか？古い？）

Otherwise the `channel_announcement` is usually followed immediately by a
`channel_update`. Ideally we would specify that the first (oldest) `channel_update`'s
timestamp is to be used as the time of the `channel_announcement`, but new nodes on
the network will not have this, and further would require the first `channel_update`
timestamp to be stored. Instead, we allow any update to be used, which
is simple to implement.

そうでなければ、
channel_announcementは通常直後にchannel_updateが続く。
理想的には、我々は最初の（一番古い）channel_updateのtimestampを
channel_announcementの時間として使用することを指定するが、
ネットワーク上の新しいnodesはこれを持たず、
さらに、最初のchannel_updateタイムスタンプを保存する必要がある。
代わりに、任意のupdate（XXX: channel_update）を使用することができ、これは実装が簡単である。
（XXX: なぜ最古のものがいいのか？）
（XXX: 任意と書いているが実際は持っているものの最新であろう）

In the case where the `channel_announcement` is nonetheless missed,
`query_short_channel_ids` can be used to retrieve it.

それでもなおchannel_announcementが欠けている場合、
それを取得するためにquery_short_channel_idsを使用することができる。

## Initial Sync

If a node requires an initial sync of gossip messages, it will be flagged
in the `init` message, via a feature flag ([BOLT #9](09-features.md#assigned-localfeatures-flags)).

a nodeがgossip messagesのan initial syncを必要とする場合、
ノードは機能フラグBOLT#9を介して initメッセージにフラグが立てられる。

Note that the `initial_routing_sync` feature is overridden (and should
be considered equal to 0) by the `gossip_queries` feature if the
latter is negotiated via `init`.

注：initial_routing_sync featureはgossip_queries featureによって、
もし後者がinitを介してネゴシエートされている場合、
オーバーライドされる（そして、0に等しいと見なす必要がある）。

Note that `gossip_queries` does not work with older nodes, so the
value of `initial_routing_sync` is still important to control
interactions with them.

注：gossip_queriesは古いノードとは動作しないので、
initial_routing_syncの値はまだ、彼らとの相互作用を制御するために重要である。

### Requirements

A node:
  - if the `gossip_queries` feature is negotiated:
	- MUST NOT relay any gossip messages unless explicitly requested.

  - gossip_queries機能がネゴシエートされている場合：
    - 明示的に要求されない限り、gossip messagesを中継してはならない。

  - otherwise:
    - if it requires a full copy of the peer's routing state:
      - SHOULD set the `initial_routing_sync` flag to 1.
    - upon receiving an `init` message with the `initial_routing_sync` flag set to
    1:
      - SHOULD send gossip messages for all known channels and nodes, as if they were just
      received.
    - if the `initial_routing_sync` flag is set to 0, OR if the initial sync was
    completed:
      - SHOULD resume normal operation, as specified in the following
      [Rebroadcasting](#rebroadcasting) section.

  - そうでなければ：
    - peerのルーティング状態の完全なコピーが必要な場合は次のようにする：
      - initial_routing_syncフラグを1に設定すべきである。
    - initial_routing_syncフラグが1にセットされたinit messageを受信すると：
      - 全ての既知のchannelsとnodesについて、受信したばかりのようにgossip messagesを送信すべきである。
    - initial_routing_syncフラグが0に設定されているか、または初期同期が完了した場合：
      - 次のRebroadcastingセクションで指定されているように、normal operationを再開すべきである。

## Rebroadcasting

### Requirements

A receiving node:
  - upon receiving a new `channel_announcement` or a `channel_update` or
  `node_announcement` with an updated `timestamp`:
    - SHOULD update its local view of the network's topology accordingly.

  - 新しいchannel_announcementか、
  updateしたtimestampを持つ、
  channel_updateかnode_announcementを受信すると：
    - それに従って、ネットワークのトポロジのローカルビューをupdateすべきである。

  - after applying the changes from the announcement:
    - if there are no channels associated with the corresponding origin node:
      - MAY purge the origin node from its set of known nodes.
    - otherwise:
      - SHOULD update the appropriate metadata AND store the signature
      associated with the announcement.
        - Note: this will later allow the node to rebuild the announcement
        for its peers.

  - announcementからの変更を適用した後：
    - 対応する（XXX: node_announcementの？）
    origin nodeに関連付けられたchannelがない場合は、
      - origin nodeをその既知のnodesのセットからパージしてもよい。
    - そうでなければ：
      - 適切なメタデータをupdateし、announcementに関連付けられたsignatureを格納すべきである。
        - 注：これにより、後でnodeがそれのpeersのannouncementを再構築できるようになる。
        （XXX: ？）

A node:
  - if the `gossip_queries` feature is negotiated:
	- MUST not send gossip until it receives `gossip_timestamp_filter`.

  - gossip_queries機能がネゴシエートされている場合：
    - gossip_timestamp_filterが受信されるまでgossipを送信してはならない。

  - SHOULD flush outgoing gossip messages once every 60 seconds, independently of
  the arrival times of the messages.
    - Note: this results in staggered announcements that are unique (not
    duplicated).

  - messagesの到着時刻とは無関係に、60秒おきに、出力gossip messagesをフラッシュすべきである。
    - 注：これにより、staggeredな（XXX: ？）announcementsが一意になる（重複しない）。

  - MAY re-announce its channels regularly.
    - Note: this is discouraged, in order to keep the resource requirements low.

  - 定期的にchannelsをre-announceしてもよい。
    - 注：リソース要件を低く抑えるため、これはお勧めしない。

  - upon connection establishment:
    - SHOULD send all `channel_announcement` messages, followed by the latest
    `node_announcement` AND `channel_update` messages.

  - 接続確立時：
    - 全てのchannel_announcement messageを送信し、
    （XXX: これは切断時に受信してたものを全部送る？）
    続けて最新のnode_announcementとchannel_update messagesを送信すべきである。

### Rationale

Once the gossip message has been processed, it's added to a list of outgoing
messages, destined for the processing node's peers, replacing any older
updates from the origin node. This list of gossip messages will be flushed at
regular intervals; such a store-and-delayed-forward broadcast is called a
_staggered broadcast_. Also, such batching forms a natural rate
limit with low overhead.

gossip messageが処理されると、
processing nodeのpeers向けの出力messagesのリストに追加され、
origin nodeからの古いupdatesが置き換えられる。
このgossip messagesのリストは定期的にフラッシュされる。
そのようなstore-and-delayed-forward broadcastは、staggered broadcastと呼ばれる。
（XXX: 違くない？[staggered broadcast]
(https://pdfs.semanticscholar.org/0888/3486a96150da7664d8c4dd932f27272c0d7f.pdf)）
また、そのようなバッチ処理は、低いオーバヘッドで自然なレート制限を形成する。

The sending of all gossip on reconnection is naive, but simple,
and allows bootstrapping for new nodes as well as updating for nodes that
have been offline for some time.  The `gossip_queries` option
allows for more refined synchronization.

再接続時に全てのgossipを送信するのは素朴であるが、簡単で、新しいnodesのブートストラップと、
しばらくの間オフラインだったnodesのupdateが可能である。
gossip_queriesオプションにより、より洗練された同期が可能になる。

## HTLC Fees

（XXX: HTLC Feesはupdate_add_htlcの差分として中継nodeに渡される）

### Requirements

The origin node:

  - SHOULD accept HTLCs that pay a fee equal to or greater than:
    - fee_base_msat + ( amount_to_forward * fee_proportional_millionths / 1000000 )
  - SHOULD accept HTLCs that pay an older fee, for some reasonable time after
  sending `channel_update`.
    - Note: this allows for any propagation delay.

  - 次のfee以上のfeeを支払うHTLCを受け入れるべきである：
    - fee_base_msat +（amount_msat * fee_proportional_millionths / 1000000）
    （XXX: (fee_proportional_millionths / 1000000)は1satoshi分にしている。）
    （XXX: fee_base_msat、fee_proportional_millionthsは自分の（channel_update）のもの）
  - channel_update送信後、適切な時間、古いfeeを支払うHTLCを受け入れるべきである。
    - 注：これは伝搬遅延を許容する。

## Pruning the Network View

### Requirements

A node:

  - SHOULD monitor the funding transactions in the blockchain, to identify
  channels that are being closed.

  - 閉鎖されているchannelsを識別するために、ブロックチェーン内のfunding transactionsを監視すべきである。

  - if the funding output of a channel is being spent:
    - SHOULD be removed from the local network view AND be considered closed.

  - channelのfunding outputが使われている場合：
    - ローカルネットワークビューから削除し、閉じているとみなすべきである。

  - if the announced node no longer has any associated open channels:
    - MAY prune nodes added through `node_announcement` messages from their
    local view.
      - Note: this is a direct result of the dependency of a `node_announcement`
      being preceded by a `channel_announcement`.

  - announceされたnodeに関連する開いているchannelsがなくなった場合：
    - ローカルビューから、node_announcement messagesを通じて追加されたnodesをプルーニングしてよい。
      - 注：これは、node_announcementの前にchannel_announcementがあるという依存関係の直接の結果である。

### Recommendation on Pruning Stale Entries

#### Requirements

A node:

  - if a channel's latest `channel_update`s `timestamp` is older than two weeks
  (1209600 seconds):
    - MAY prune the channel.
    - MAY ignore the channel.
    - Note: this is an individual node policy and MUST NOT be enforced by
    forwarding peers, e.g. by closing channels when receiving outdated gossip
    messages.

  - channelの最新channel_updateのtimestampが2週間（1209600秒）より古い場合：
    - channelをプルーニングしてよい。
    - channelを無視してよい。
    - 注：これは個々のnodeのポリシーであり、転送peersによって強制されてはならない
    例えば、古くなったgossip messagesを受信したときにchannlesを閉じるなど。
    （XXX: ？）

#### Rationale

Several scenarios may result in channels becoming unusable and its endpoints
becoming unable to send updates for these channels. For example, this occurs if
both endpoints lose access to their private keys and can neither sign
`channel_update`s nor close the channel on-chain. In this case, the channels are
unlikely to be part of a computed route, since they would be partitioned off
from the rest of the network; however, they would remain in the local network
view would be forwarded to other peers indefinitely.

いくつかのシナリオでは、channelsが使用不能になり、
そのendpointがこれらのchannelsのupdatesを送信できなくなるだろう。
たとえば、両方のendpointが秘密鍵へのアクセスを失い、
channel_updateに署名することも、オンチェーンのchannelを閉じることもできない 。
この場合、このchannelsはネットワークの他の部分から分離されるので、
計算された経路の一部になる可能性は低い。
ただし、ローカルネットワークビューに残っていると、他のpeersに無期限に転送される。
（XXX: ？）

## Recommendations for Routing

When calculating a route for an HTLC, both the `cltv_expiry_delta` and the fee
need to be considered: the `cltv_expiry_delta` contributes to the time that
funds will be unavailable in the event of a worst-case failure. The relationship
between these two attributes is unclear, as it depends on the reliability of the
nodes involved.

HTLCのルートを計算する際には、cltv_expiry_deltaとfeeの両方を考慮する必要がある：
cltv_expiry_deltaは、最悪な失敗の場合にファンドが利用できなくなる時間に貢献する。
関係するnodesの信頼性に依存するため、これらの2つの属性の関係は不明である。

If a route is computed by simply routing to the intended recipient and summing
the `cltv_expiry_delta`s, then it's possible for intermediate nodes to guess
their position in the route. Knowing the CLTV of the HTLC, the surrounding
network topology, and the `cltv_expiry_delta`s gives an attacker a way to guess
the intended recipient. Therefore, it's highly desirable to add a random offset
to the CLTV that the intended recipient will receive, which bumps all CLTVs
along the route.

目的の受信者にルーティングしてcltv_expiry_deltaを合計するだけでルートが計算された場合、
中間nodesはルート内の位置を推測することができる。
HTLCのCLTV、周囲のネットワークトポロジ、およびcltv_expiry_deltasを知ることで、
攻撃者は意図された受信者を推測することができる。
したがって、意図された受信者が受信するCLTVにランダムオフセットを追加することが非常に望ましく、
これはルートに沿って全てのCLTVを押し上げる。

In order to create a plausible offset, the origin node MAY start a limited
random walk on the graph, starting from the intended recipient and summing the
`cltv_expiry_delta`s, and use the resulting sum as the offset.
This effectively creates a _shadow route extension_ to the actual route and
provides better protection against this attack vector than simply picking a
random offset would.

尤もらしいオフセットを作成するために、origin nodeはグラフ上の限定されたランダムウォークを開始し、
目的の受信者から開始してcltv_expiry_deltaを合計し、その結果の合計をオフセットとして使用してよい。
（XXX: 受信者から先にルートが延びているように見せかける。
送信者側のシャドウルートは意味がない（cltvは受信者側から足しこまれるので）。
ただし、あまり既存のchannel_updateの値をそのまま足しこむとcltv_expiry_deltaの値が分散しているときにルートが推測できるのでは？
極端な例として全てのノードのcltv_expiry_deltaの値が被らない2^xの値とする。
中継ノードはシャドウルート込みではあるが残りの完全なルートが一意に特定できる。
ランダムな値も足す必要がある）
（XXX: シャドウルートは受信者隠蔽のためであるのに送信者に依存しているのでトラストレスな思想にそぐわない。同様に現行のルーティングでは解決策がない）

Other more advanced considerations involve diversification of route selection,
to avoid single points of failure and detection, and balancing of local
channels.

他の高度な考慮事項には、
ルート選択の多様化、
単一障害点の回避と検出、
およびローカルchannelsのバランシングが含まれる。

### Routing Example

Consider four nodes:
4つのnodesを考える：

```
   B
  / \
 /   \
A     C
 \   /
  \ /
   D
```

Each advertises the following `cltv_expiry_delta` on its end of every
channel:

それぞれは、全てのチャンネルの最後に、次のcltv_expiry_deltaをadvetiseする。

1. A: 10 blocks
2. B: 20 blocks
3. C: 30 blocks
4. D: 40 blocks

C also uses a `min_final_cltv_expiry` of 9 (the default) when requesting
payments.

またCは支払いを要求するときに、9（デフォルト）のmin_final_cltv_expiryを使用する。

Also, each node has a set fee scheme that it uses for each of its
channels:

また、各nodeには、そのchannelsごとに使用する、feeを設定する体系がある。

1. A: 100 base + 1000 millionths
2. B: 200 base + 2000 millionths
3. C: 300 base + 3000 millionths
4. D: 400 base + 4000 millionths

The network will see eight `channel_update` messages:

ネットワークには8つのchannel_update messagesが見える。

1. A->B: `cltv_expiry_delta` = 10, `fee_base_msat` = 100, `fee_proportional_millionths` = 1000
1. A->D: `cltv_expiry_delta` = 10, `fee_base_msat` = 100, `fee_proportional_millionths` = 1000
1. B->A: `cltv_expiry_delta` = 20, `fee_base_msat` = 200, `fee_proportional_millionths` = 2000
1. D->A: `cltv_expiry_delta` = 40, `fee_base_msat` = 400, `fee_proportional_millionths` = 4000
1. B->C: `cltv_expiry_delta` = 20, `fee_base_msat` = 200, `fee_proportional_millionths` = 2000
1. D->C: `cltv_expiry_delta` = 40, `fee_base_msat` = 400, `fee_proportional_millionths` = 4000
1. C->B: `cltv_expiry_delta` = 30, `fee_base_msat` = 300, `fee_proportional_millionths` = 3000
1. C->D: `cltv_expiry_delta` = 30, `fee_base_msat` = 300, `fee_proportional_millionths` = 3000

**B->C.** If B were to send 4,999,999 millisatoshi directly to C, it would
neither charge itself a fee nor add its own `cltv_expiry_delta`, so it would
use C's requested `min_final_cltv_expiry` of 9. Presumably it would also add a
_shadow route_ to give an extra CLTV of 42. Additionally, it could add extra
CLTV deltas at other hops, as these values represent a minimum, but chooses not
to do so here, for the sake of simplicity:

B->C。Bが4,999,999millisatoshiを直接Cに送っても、それはfeeをチャージすることも、
それ自体のcltv_expiry_deltaを追加することもないので、
Cの9のmin_final_cltv_expiryの要求を使用することになる。
おそらく42の余分なCLTVを与えるshadow routeも追加するだろう。
さらに、これらの値は最小値を表すので、他のhopsで余分なCLTVデルタを追加することができるが、
ここでは単純化のために選択しない。

   * `amount_msat`: 4999999
   * `cltv_expiry`: current-block-height + 9 + 42
   * `onion_routing_packet`:
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 9 + 42

（XXX: ここでシャドウルートは中継者が居ないので受信者を隠蔽する意味がないが、
追加しないと受信者に送信者が漏洩する）

**A->B->C.** If A were to send 4,999,999 millisatoshi to C via B, it needs to
pay B the fee it specified in the B->C `channel_update`, calculated as
per [HTLC Fees](#htlc-fees):

A->B->C。もしAが4,999,999millisatoshiをB経由でCに送るなら、
HTLC Feesとして計算した、B->Cのchannel_updateで指定したfeeをBに支払う必要がある：

        fee_base_msat + ( amount_to_forward * fee_proportional_millionths / 1000000 )

	200 + ( 4999999 * 2000 / 1000000 ) = 10199

Similarly, it would need to add B->C's `channel_update` `cltv_expiry` (20), C's
requested `min_final_cltv_expiry` (9), and the cost for the _shadow route_ (42).
Thus, A->B's `update_add_htlc` message would be:

同様に、B->Cのchannel_updateのcltv_expiry（20）、
Cが要求するmin_final_cltv_expiry（9）、
およびshadow route（42）のコストを追加する必要がある。
したがって、A→Bのupdate_add_htlc messageは次のようになる：

   * `amount_msat`: 5010198
   * `cltv_expiry`: current-block-height + 20 + 9 + 42
   * `onion_routing_packet`:
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 9 + 42

B->C's `update_add_htlc` would be the same as B->C's direct payment above.

B->Cのupdate_add_htlcは上記のB->Cのdirect paymentと同じである。

**A->D->C.** Finally, if for some reason A chose the more expensive route via D,
A->D's `update_add_htlc` message would be:

A->D->C。最後に、何らかの理由でAがD経由でより高価なルートを選択した場合、A->Dのupdate_add_htlc messageは次のようになる：

   * `amount_msat`: 5020398
   * `cltv_expiry`: current-block-height + 40 + 9 + 42
   * `onion_routing_packet`:
	 * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 9 + 42

And D->C's `update_add_htlc` would again be the same as B->C's direct payment
above.

そして、D->Cのupdate_add_htlcは、上記のB->Cのdirect paymentと同じである。

## References

1. <a id="reference-1">[RFC 1950 "ZLIB Compressed Data Format Specification version 3.3](https://www.ietf.org/rfc/rfc1950.txt)</a>
2. <a id="reference-2">[Maximum Compression Factor](https://zlib.net/zlib_tech.html)</a>

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
