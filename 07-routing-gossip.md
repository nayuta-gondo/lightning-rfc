# BOLT #7: P2P Node and Channel Discovery

（ほとんどGoogle翻訳まま。意味が通らないところだけ修正。）
（ところどころコメント有。）

This specification describes simple node discovery, channel discovery, and channel update mechanisms that do not rely on a third-party to disseminate the information.

この仕様では、情報を普及させるために第三者に依存しない単純なnode discovery、channel discovery、およびchannel updateのメカニズムについて説明する。

Node and channel discovery serve two different purposes:

nodeとchannelのdiscoveryには、次の2つの目的がある。：

 - Channel discovery allows the creation and maintenance of a local view of the network's topology, so that a node can discover routes to desired destinations.

 - channel discoveryは、ネットワークのトポロジのローカルビューの作成および保守を可能にするので、nodeは所望の宛先へのルートをdiscoveryすることができる。
（local view？全世界の全てのchannelではなく一部であるという意味か？）

 - Node discovery allows nodes to broadcast their ID, host, and port, so that other nodes can open connections and establish payment channels with them.

 - node discoveryにより、nodeはID、ホスト、およびポートをブロードキャストでき、他のnodesが接続を開いて支払いchannelsを確立することができる。

To support channel discovery, peers in the network exchange
`channel_announcement` messages containing information regarding new
channels between the two nodes. They can also exchange `channel_update`
messages, which update information about a channel. There can only be
one valid `channel_announcement` for any channel, but at least two
`channel_update` messages are expected.

channel discoveryをサポートするために、ネットワーク内のpeersは、2つのnodes間の新しいchannelsに関する情報を含むchannel_announcement messagesを交換する。
（peersは隣接する2つのnodesを強調した表現か？？？）
また、channelに関する情報をupdateするchannel_update messagesを交換することもできる。
任意のchannelに対して有効なchannel_announcementは1つだけですが、少なくとも2つのchannel_update messagesが必要である。

To support node discovery, peers exchange `node_announcement`
messages, which supply additional information about the nodes. There may be
multiple `node_announcement` messages, in order to update the node information.

node discoveryをサポートするために、peersはnodesに関する追加情報を提供するnode_announcement messagesを交換する。
nodes情報をupdateするために、複数のnode_announcement messagesが存在する可能性がある。

# Table of Contents

  * [The `announcement_signatures` Message](#the-announcement_signatures-message)
  * [The `channel_announcement` Message](#the-channel_announcement-message)
  * [The `node_announcement` Message](#the-node_announcement-message)
  * [The `channel_update` Message](#the-channel_update-message)
  * [Initial Sync](#initial-sync)
  * [Rebroadcasting](#rebroadcasting)
  * [HTLC Fees](#htlc-fees)
  * [Pruning the Network View](#pruning-the-network-view)
  * [Recommendations for Routing](#recommendations-for-routing)
  * [References](#references)

## The `announcement_signatures` Message

This is a direct message between the two endpoints of a channel and serves as an opt-in mechanism to allow the announcement of the channel to the rest of the network.
It contains the necessary signatures, by the sender, to construct the `channel_announcement` message.

これは、channelの2つのendpoints間のdirect messageで、channelのannouncementをネットワークの他の部分に許可するオプトインメカニズムを提供する。
（送られなければオプトインされないということか？？？なんでchannel_flagsだけで表さないのか？？？）
これには送信者がchannel_announcement messageを作成するために必要なsignaturesが含まれています。

1. type: 259 (`announcement_signatures`)
2. data:
    * [`32`:`channel_id`]
    * [`8`:`short_channel_id`]
    * [`64`:`node_signature`]
    * [`64`:`bitcoin_signature`]

The willingness of the initiating node to announce the channel is signaled during channel opening by setting the `announce_channel` bit in `channel_flags` (see [BOLT #2](02-peer-protocol.md#the-open_channel-message)).

開始nodeがchannelをannounceする意思は、channel_flagsのannounce_channelビットをセットすることによってchannel開通中に通知される（BOLT＃2参照）。

### Requirements

The `announcement_signatures` message is created by constructing a `channel_announcement` message, corresponding to the newly established channel, and signing it with the secrets matching an endpoint's `node_id` and `bitcoin_key`. After it's signed, the
`announcement_signatures` message may be sent.

announcement_signatures messageはchannel_announcement messageを構築することにより作成され、
（あとあとわかるだろう？？？）
新しく設立されたchannelに対応し、endpointのnode_idとbitcoin_keyにマッチングしたsecretsで署名される。
署名された後、 announcement_signatures messageを送信することができる。

The `short_channel_id` is the unique description of the funding transaction.
It is constructed as follows:
  1. the most significant 3 bytes: indicating the block height
  2. the next 3 bytes: indicating the transaction index within the block
  3. the least significant 2 bytes: indicating the output index that pays to the channel.

short_channel_idはfunding transactionの一意な記述である。
それは次のように構築されます：
  1. 最上位3バイト：ブロックの高さを示す
  2. 次の3バイト：ブロック内のtransactionインデックスを示す
  3. 最下位2バイト：channelに支払うoutputインデックスを示す。

A node:
node：
  - if the `open_channel` message has the `announce_channel` bit set AND a `shutdown` message has not been sent:
    - MUST send the `announcement_signatures` message.
      - MUST NOT send `announcement_signatures` messages until `funding_locked`
      has been sent AND the funding transaction has at least six confirmations.

  - open_channel messageのannounce_channelビットが設定され、shutdown messageが送信されていない場合：
    - announcement_signatures messageを送信しなければならない。
      - funding_lockedが送られ、かつfunding transactionが6つの確認を持つまで、announcement_signatures messagesを送信してはいけない。

  - otherwise:
    - MUST NOT send the `announcement_signatures` message.

  - そうでなければ：
    - announcement_signatures messageを送信してはいけない。

  - upon reconnection:
    - MUST respond to the first `announcement_signatures` message with its own
    `announcement_signatures` message.
    - if it has NOT received an `announcement_signatures` message:
      - SHOULD retransmit the `announcement_signatures` message.

  - 再接続時：
    - 最初のannouncement_signatures messageにそれ自身のannouncement_signatures messageで応答しなければならない。
    （the firstというのが、そのchannelの最初ではなく、再接続後の最初だと思われる？？？）
    - announcement_signatures messageを受信していない場合：
      - announcement_signatures messageを再送すべきである。

A recipient node:
  - if the `node_signature` OR the `bitcoin_signature` is NOT correct:
    - MAY fail the channel.
  - if it has sent AND received a valid `announcement_signatures` message:
    - SHOULD queue the `channel_announcement` message for its peers.

受信node：
  - node_signatureもしくはbitcoin_signatureが正しくない場合：
    - channelを失敗させてよい。
  - 有効なannouncement_signatures messageを送受信した場合：
    - channel_announcement messageをそのpeersのためにキューイングすべきである。
    （？？？）

## The `channel_announcement` Message

This message contains ownership information regarding a channel. It ties
each on-chain Bitcoin key to the associated Lightning node key, and vice-versa.
The channel is not practically usable until at least one side has announced
its fee levels and expiry, using `channel_update`.

このmessageには、channelに関する所有権情報が含まれています。
それは各オンチェーンのBitcoin keyと関連するLightning node keyを結び付け、逆もまた同様である。
このchannelは、channel_updateを使用して少なくとも片側がfeeレベルとexpiryをannounceするまで、実際には使用できません。

Proving the existence of a channel between `node_1` and `node_2` requires:

node_1とnode_2間のchannelの存在証明が必要とするのは、：

1. proving that the funding transaction pays to `bitcoin_key_1` and
   `bitcoin_key_2`
2. proving that `node_1` owns `bitcoin_key_1`
3. proving that `node_2` owns `bitcoin_key_2`

（区切り）

1. funding transactionがbitcoin_key_1とbitcoin_key_2に支払っていることを証明
2. node_1がbitcoin_key_1を所有していることを証明
3. node_2がbitcoin_key_2を所有していることを証明

Assuming that all nodes know the unspent transaction outputs, the first proof is
accomplished by a node finding the output given by the `short_channel_id` and
verifying that it is indeed a P2WSH funding transaction output for those keys
specified in [BOLT #3](03-transactions.md#funding-transaction-output).

すべてのnodesがunspent transaction outputsを知っていると仮定すると、
（？？？）
nodeによる、short_channel_idから与えられるoutputの発見と、それが実際に[BOLT #3]で指定されたように、
それらのkeysのP2WSH funding transaction outputであることの確認により、最初の証明は達成される。

The last two proofs are accomplished through explicit signatures:
`bitcoin_signature_1` and `bitcoin_signature_2` are generated for each
`bitcoin_key` and each of the corresponding `node_id`s are signed.

最後の二つの証明は、明示的なsignaturesを介して達成される：
bitcoin_signature_1とbitcoin_signature_2が、それぞれのbitcoin_keyとそれぞれの対応するnode_idが署名されることにより、
それぞれ生成される。

It's also necessary to prove that `node_1` and `node_2` both agree on the
announcement message: this is accomplished by having a signature from each
`node_id` (`node_signature_1` and `node_signature_2`) signing the message.

また、node_1とnode_2が共にannouncement messageに合意していることを証明する必要がある：
各node_idがmessageにしたsignature（node_signature_1とnode_signature_2）を持つことによって達成されます。

1. type: 256 (`channel_announcement`)
2. data:
    * [`64`:`node_signature_1`]
    * [`64`:`node_signature_2`]
    * [`64`:`bitcoin_signature_1`]
    * [`64`:`bitcoin_signature_2`]
    * [`2`:`len`]
    * [`len`:`features`]
    * [`32`:`chain_hash`]
    * [`8`:`short_channel_id`]
    * [`33`:`node_id_1`]
    * [`33`:`node_id_2`]
    * [`33`:`bitcoin_key_1`]
    * [`33`:`bitcoin_key_2`]

### Requirements

The origin node:
起点node：
（messageの起点ということであろう？？？）

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

  - short_channel_idに、[BOLT＃2]で規定される確認済みのfunding transactionを参照するように設定しなければならない。
    - 注：対応するoutputは、[BOLT＃3]に記述されているように、P2WSHでなければならない。

  - MUST set `node_id_1` and `node_id_2` to the public keys of the two nodes
  operating the channel, such that `node_id_1` is the numerically-lesser of the
  two DER-encoded keys sorted in ascending numerical order.

  - node_id_1とnode_id_2に、channelを操作する2つのnodesの公開鍵を、node_id_1は、2つのDERエンコードされたkeysを数字の小さい順にソートし、数値的に小さいものになるように設定する。

  - MUST set `bitcoin_key_1` and `bitcoin_key_2` to `node_id_1` and `node_id_2`'s
  respective `funding_pubkey`s.

  - bitcoin_key_1とbitcoin_key_2に、node_id_1とnode_id_2のそれぞれのfunding_pubkeyを設定する。

  - MUST compute the double-SHA256 hash `h` of the message, beginning at offset
  256, up to the end of the message.
    - Note: the hash skips the 4 signatures but hashes the rest of the message,
    including any future fields appended to the end.

  - 位置256から始まり、messageの最後までのdouble-SHA256ハッシュhを計算しなければならない。
    - 注：ハッシュは4つのsignaturesをスキップしますが、最後に追加される将来のフィールドも含め、残りのmessageをハッシュする。

  - MUST set `node_signature_1` and `node_signature_2` to valid
    signatures of the hash `h` (using `node_id_1` and `node_id_2`'s respective
    secrets).

  - node_signature_1とnode_signature_2に、（node_id_1とnode_id_2のそれぞれのsecretsを使って）ハッシュhの正しいsignaturesを設定しなければなりません。

  - MUST set `bitcoin_signature_1` and `bitcoin_signature_2` to valid
  signatures of the hash `h` (using `bitcoin_key_1` and `bitcoin_key_2`'s
  respective secrets).

  - bitcoin_signature_1とbitcoin_signature_2に、（bitcoin_key_1とbitcoin_key_2のそれぞれのsecretsを使って）ハッシュhの正しいsignaturesを設定しなければなりません。

  - SHOULD set `len` to the minimum length required to hold the `features` bits
  it sets.

  - lenに、featuresビットを保持するために必要な最小長を設定すべきである。

The final node:
終点node：
（受け取ったnode？？？originの対義でfinalってなっているのかな？？？下にreceiverという表現があるので受け取ったnodeで間違いないだろう）

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
    - announcementを転送すべきでない。

  - if the `short_channel_id`'s output does NOT correspond to a P2WSH (using
    `bitcoin_key_1` and `bitcoin_key_2`, as specified in
    [BOLT #3](03-transactions.md#funding-transaction-output)) OR the output is
    spent:
    - MUST ignore the message.

  - short_channel_idのoutputがP2WSH（[BOLT＃3]で規定されているように、bitcoin_key_1とbitcoin_key_2を使用した）に対応していない場合、またはoutputが消費された場合：
    - messageを無視しなければならない。

  - if the specified `chain_hash` is unknown to the receiver:
    - MUST ignore the message.

  - 指定されたchain_hashがレシーバーに知られていない場合：
    - messageを無視しなければならない。

  - otherwise:
  - そうでなければ：

    - if `bitcoin_signature_1`, `bitcoin_signature_2`, `node_signature_1` OR
    `node_signature_2` are invalid OR NOT correct:
      - SHOULD fail the connection.

    - bitcoin_signature_1、bitcoin_signature_2、node_signature_1もしくはnode_signature_2が無効または正しく無い場合。
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
          - 再放送のためにmessagesをキューに入れるべきである。
          （もしかするとブロックチェーンのリオーガニゼーションが発生して、参照されたtransactionが以前にchannelとして発表されているものと一致する可能性があるからそれまで）
          - 期待される最小の長さより長いmessagesに対してはNOTを選択することができる。

      - if it has previously received a valid `channel_announcement`, for the
      same transaction, in the same block, but for a different `node_id_1` or
      `node_id_2`:
        - SHOULD blacklist the previous message's `node_id_1` and `node_id_2`,
        as well as this `node_id_1` and `node_id_2` AND forget any channels
        connected to them.

      - 以前に同じtransaction、同じブロック内ではあるが、別のnode_id_1かnode_id_2のものとして有効なchannel_announcementを受け取っていれば：
      （これはshort_channel_idで判断されるのであろう。）
        - 前のmessageのnode_id_1とnode_id_2をブラックリストに載せ、このnode_id_1とnode_id_2も同様にし、それらに接続された任意のchannelsも忘れるべきだ。
        （鍵が漏洩したと考えられる。後述。）

      - otherwise:
        - SHOULD store this `channel_announcement`.

      - そうでなければ：
        - このchannel_announcementを格納すべきだ。

  - once its funding output has been spent OR reorganized out:
    - SHOULD forget a channel.

  - 一旦そのfunding outputが費やされたか、またはブロックチェーンのリオーガニゼーションが発生した場合：
    - channelを忘れるべきだ。

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

channelsは十分に深くなる前にadvertiseすべきではありませんが、再ブロードキャストに対する要件は、
transactionが別のブロックに移動していない場合にのみ適用されます。
（送信者が確認したものとは異なるブロックで確定した、つまりリオーガニゼーションが発生して確定してしまった場合には再ブロードキャストをする機会が失われたということか？？？short_channel_idが合うことがないので）

In order to avoid storing excessively large messages, yet still allow for
reasonable future expansion, nodes are permitted to restrict rebroadcasting
(perhaps statistically).

過度に大きなmessagesを格納することを避けるために、将来の合理的な拡張を可能にするために、
nodesは再ブロードキャスティングを制限する（おそらく統計的に）ことが許可されています。

New channel features are possible in the future: backwards compatible (or
optional) features will have _odd_ feature bits, while incompatible features
will have _even_ feature bits
(["It's OK to be odd!"](00-introduction.md#glossary-and-terminology-guide)).
Incompatible features will result in the announcement not being forwarded by
nodes that do not understand them.

将来的には新しいchannel機能が可能です：後方互換性のある（またはオプションの）機能は奇数のfeatureビットを持ち、
互換性のないfeatureは偶数のfeatureビットを持つ（「奇妙であることは問題ない！」）。
互換性のない機能は、そのannouncementが、それらを理解していないnodesによって転送されない結果になる。

## The `node_announcement` Message

This message allows a node to indicate extra data associated with it, in
addition to its public key. To avoid trivial denial of service attacks,
nodes not associated with an already known channel are ignored.

このmessageは、nodeが公開鍵に加えて、それに関連する追加のデータを示すことを可能にする。
DoS攻撃を回避するために、既知のchannelに関連付けられていないnodesは無視されます。

1. type: 257 (`node_announcement`)
2. data:
   * [`64`:`signature`]
   * [`2`:`flen`]
   * [`flen`:`features`]
   * [`4`:`timestamp`]
   * [`33`:`node_id`]
   * [`3`:`rgb_color`]
   * [`32`:`alias`]
   * [`2`:`addrlen`]
   * [`addrlen`:`addresses`]

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
それはnodeに接続するための一連のaddress descriptorを含みます。
最初のバイトはアドレスタイプを記述し、そのタイプの適切なバイト数が続きます。

The following `address descriptor` types are defined:

次のaddress descriptorタイプが定義されています：

   * `0`: padding; data = none (length 0)
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

   * 0：パディング; データ = なし（長さ0）
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
起点node：

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
    注：rgb_colorの最初のバイトは赤の値、2番目のバイトは緑の値、最後のバイトは青の値である。

  - MUST set `alias` to a valid UTF-8 string, with any `alias` trailing-bytes
  equal to 0.

  - aliasは、有効なUTF-8文字列に設定する必要がある。末尾に0を指定する。

  - SHOULD fill `addresses` with an address descriptor for each public network
  address that expects incoming connections.

  - addressesは、接続を待つパブリックネットワークアドレスごとにaddress descriptorを記入すべきである。

  - MUST set `addrlen` to the number of bytes in `addresses`.

  - addrlenは、addressesのバイト数に設定する必要がある。

  - MUST place non-zero typed address descriptors in ascending order.

  - ゼロ以外のタイプのaddress descriptorを昇順に配置する必要がある。

  - MAY place any number of zero-typed address descriptors anywhere.

  - 任意の数のタイプゼロのaddress descriptorを任意の場所に配置することができる。

  - SHOULD use placement only for aligning fields that follow `addresses`.

  - addressesの次のフィールドを整列させるためだけに配置を使用すべきである。
  （任意の場所にOKではないのか？？？）

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

The final node:
終点node：

  - if `node_id` is NOT a valid compressed public key:
    - SHOULD fail the connection.
    - MUST NOT process the message further.

  - node_idが有効な圧縮された公開鍵でない場合：
    - 接続に失敗すべきである。
    - messageをさらに処理してはならない。

  - if `signature` is NOT a valid signature (using `node_id` of the
  double-SHA256 of the entire message following the `signature` field, including
  unknown fields following `alias`):
    - SHOULD fail the connection.
    - MUST NOT process the message further.

  - signatureが有効なsignatureではない場合（node_id（の秘密鍵）を使用して、aliasの次の未知のフィールドを含む、signatureフィールドの後に続くmessage全体のdouble-SHA256（への署名））。
  （なんで最後でないaliasに言及？？？）
    - 接続に失敗すべきである。
    - messageをさらに処理してはならない。

  - if `features` field contains _unknown even bits_:
    - MUST NOT parse the remainder of the message.
    - MAY discard the message altogether.
    - SHOULD NOT connect to the node.

  - featuresフィールドが未知の偶数ビットを含んでいる場合：
    - messageの残りの部分を解析してはならない。
    - messageを完全に破棄してもよい。
    - nodeに接続してはならない。

  - MAY forward `node_announcement`s that contain an _unknown_ `features` _bit_,
  regardless of if it has parsed the announcement or not.

  - それ（終点node）がannouncementを解析したかどうかに関係なく、
  未知のfeaturesビット含むnode_announcementを転送することができる。

  - SHOULD ignore the first `address descriptor` that does NOT match the types
  defined above.

  - 上で定義したタイプと一致しない最初のaddress descriptorを無視すべきである。
  （the first？？？）

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
    （なんで同じ条件（の反対）がここでまた出てくる？？？）
      - 再放送のためにmessagesをキューに入れるべきである。
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

nodeエイリアスは、HTML / Javascriptコンテキストまたはその他の動的に解釈されるレンダリングフレームワークで表示される前に、
必ずサニタイズする必要がある。
同様に、SQLやその他の動的に解釈されるクエリ言語をサポートする永続性エンジンを、注入脆弱性から保護するために、
プリペアドステートメント、入力検証、エスケープを使用することを検討しなさい。
（？？？そのまま訳せなかった）

* [Stored and Reflected XSS Prevention](https://www.owasp.org/index.php/XSS_%28Cross_Site_Scripting%29_Prevention_Cheat_Sheet)
* [DOM-based XSS Prevention](https://www.owasp.org/index.php/DOM_based_XSS_Prevention_Cheat_Sheet)
* [SQL Injection Prevention](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet)

（上記リンクのURL表記がMarkdownの表記とコンフリクトするので少し変更した）

Don't be like the school of [Little Bobby Tables](https://xkcd.com/327/).

Little Bobby Tablesの学校のようにしないでください。
（SQL Injectionの例。）

## The `channel_update` Message

After a channel has been initially announced, each side independently
announces the fees and minimum expiry delta it requires to relay HTLCs
through this channel. Each uses the 8-byte channel shortid that matches the
`channel_announcement` and the 1-bit `flags` field to indicate which end of the
channel it's on (origin or final). A node can do this multiple times, in
order to change fees.

channelが最初にannounceされた後、各側は、このchannelを介してHTLCを中継するために必要なfeeとminimum expiry deltaを独立して公表する。
それぞれはchannel_announcementに一致する8バイトのshortidと、
channelのどちらの終わり（起点または終点）を示すために1ビットのflagsフィールドを使用する。
nodeはこれを複数回行うことができ、feeを変更することができる。

Note that the `channel_update` message is only useful in the context
of *relaying* payments, not *sending* payments. When making a payment
 `A` -> `B` -> `C` -> `D`, only the `channel_update`s related to channels
 `B` -> `C` (announced by `B`) and `C` -> `D` (announced by `C`) will
 come into play. When building the route, amounts and expiries for HTLCs need
 to be calculated backward from the destination to the source. The exact initial
 value for `amount_msat` and the minimal value for `cltv_expiry`, to be used for
 the last HTLC in the route, are provided in the payment request
 (see [BOLT #11](11-payment-encoding.md#tagged-fields)).

このchannel_update messageが有用なのは、支払いを中継するコンテキストであり、
支払いを送信するコンテキストではないことに注意してください。
（送信（send）というのはfunderからの隣接するnodeへの支払い？？？）
A -> B -> C -> Dの支払いを作るとき、B -> C（Bによって発表）とC -> D （Cによって発表）のchannelに関係するchannel_updateのみが関わります。
（最後のC->Dも関係ないのでは？？？）
ルートを構築する際には、HTLCの金額と有効期限をdestinationからsourceまで逆方向に計算する必要がある。
ルートの最後のHTLCに使用される正確なamount_msatのための初期値とcltv_expiryのための最小値は、支払い要求で提供されます（BOLT＃11を参照）。

1. type: 258 (`channel_update`)
2. data:
    * [`64`:`signature`]
    * [`32`:`chain_hash`]
    * [`8`:`short_channel_id`]
    * [`4`:`timestamp`]
    * [`2`:`flags`]
    * [`2`:`cltv_expiry_delta`]
    * [`8`:`htlc_minimum_msat`]
    * [`4`:`fee_base_msat`]
    * [`4`:`fee_proportional_millionths`]

The `flags` bitfield is used to indicate the direction of the channel: it
identifies the node that this update originated from and signals various options
concerning the channel. The following table specifies the meaning of its
individual bits:

flagsビットフィールドは、channelの方向を示すために使用されます：
このビットフィールドは、このアップデートが発生したnodeを識別し、
channelに関するさまざまなオプションを通知する。
次の表は、個々のビットの意味を示しています。
（channelを無効にできる！）

| Bit Position  | Name        | Meaning                          |
| ------------- | ----------- | -------------------------------- |
| 0             | `direction` | Direction this update refers to. |
| 1             | `disable`   | Disable the channel.             |

| ビット位置      | 名前         | 意味                             |
| ------------- | ----------- | -------------------------------- |
| 0             | `direction` | このアップデートが参照する方向        |
| 1             | `disable`   | channelを無効にする                 |

The `node_id` for the signature verification is taken from the corresponding
`channel_announcement`: `node_id_1` if the least-significant bit of flags is 0
or `node_id_2` otherwise.

signature検証のためのnode_idは対応するchannel_announcementから取得される：
flagsの最下位ビットが0の場合はnode_id_1で、そうでない場合はnode_id_2。

### Requirements

The origin node:
起点node：

  - MAY create a `channel_update` to communicate the channel parameters to the
  final node, even though the channel has not yet been announced (i.e. the
  `announce_channel` bit was not set).
    - MUST NOT forward such a `channel_update` to other peers, for privacy
    reasons.
    - Note: such a `channel_update`, one not preceded by a
    `channel_announcement`, is invalid to any other peer and would be discarded.

  - channelがまだannounceされていなくてもchannel_updateを終点nodeにchannelパラメータを伝えるために作成できる（すなわち、announce_channelビットがセットされていなくても）。
  （yetがわからない？？？channel announcementなしでchannel_updateのみというのがありそう。隣接するnodeへのみ。）
    - プライバシーの理由から、そのようなchannel_updateを他のpeersに転送してはいけません。
    - 注：channel_announcementが前に無いようなchannel_updateは、他のpeerにとって無効であり破棄されます。

  - MUST set `signature` to the signature of the double-SHA256 of the entire
  remaining packet after `signature`, using its own `node_id`.

  - signatureに、signatureの後の残りのパケット全体のdouble-SHA256に、それ自身のnode_idを使用したsignatureを、設定しなければならない。

  - MUST set `chain_hash` AND `short_channel_id` to match the 32-byte hash AND
  8-byte channel ID that uniquely identifies the channel specified in the
  `channel_announcement` message.

  - chain_hashとshort_channel_idに、channel_announcement messageに指定されたchannelを一意に識別する32バイトのハッシュと8バイトのchannel IDを一致させるように設定する必要がある。

  - if the origin node is `node_id_1` in the message:
    - MUST set the `direction` bit of `flags` to 0.

  - 起点nodeがmessage内のnode_id_1である場合：
    - flagsのdirectionビットを0に設定しなければならない。

  - otherwise:
    - MUST set the `direction` bit of `flags` to 1.

  - そうでなければ：
    - flagsのdirectionビットを1に設定しなければならない。

  - MUST set bits that are not assigned a meaning to 0.

  - 意味が割り当てられていないビットを0に設定しなければならない。

  - MAY create and send a `channel_update` with the `disable` bit set to 1, to
  signal a channel's temporary unavailability (e.g. due to a loss of
  connectivity) OR permanent unavailability (e.g. prior to an on-chain
  settlement).
    - MAY sent a subsequent `channel_update` with the `disable` bit  set to 0 to
    re-enable the channel.

  - channelの一時的な利用不可能性（例えば、接続性の欠如による）または永久的な利用不可能性（例えば、オンチェーン上の確定前）を示すために、disableビットを1に設定してchannel_updateを作成して送信することができる。
    - channelを再びenableにするためにビットを0に設定して後続のchannel_updateを送っても良い。

  - MUST set `timestamp` to greater than 0, AND to greater than any
  previously-sent `channel_update` for this `short_channel_id`.
    - SHOULD base `timestamp` on a UNIX timestamp.

  - timestampに、0より大きく、かつ、このshort_channel_idのために以前に送信されたchannel_updateより大きく設定しなくてはならない。
    - timestampをUNIX timestampベースにすべきである。

  - MUST set `cltv_expiry_delta` to the number of blocks it will subtract from
  an incoming HTLC's `cltv_expiry`.

  - cltv_expiry_deltaに、着信HTLCのcltv_expiryから減算するブロック数を設定する必要がある。

  - MUST set `htlc_minimum_msat` to the minimum HTLC value (in millisatoshi)
  that the final node will accept.

  - htlc_minimum_msatに、最終nodeが受け入れる最小のHTLC値（millisatoshi単位で）を設定しなければならない。

  - MUST set `fee_base_msat` to the base fee (in millisatoshi) it will charge
  for any HTLC.

  - fee_base_msatに、任意のHTLCのために課金されるであろう基本fee（millisatoshi単位で）を設定しなければならない。

  - MUST set `fee_proportional_millionths` to the amount (in millionths of a
  satoshi) it will charge per transferred satoshi.

  - fee_proportional_millionthsに、転送されたsatoshi当たりに課金される金額（100万satoshi分）を設定しなければならない。

The final node:
終点node：

  - if the `short_channel_id` does NOT match a previous `channel_announcement`,
  OR if the channel has been closed in the meantime:
    - MUST ignore `channel_update`s that do NOT correspond to one of its own
    channels.

  - short_channel_idが先立つchannel_announcementに一致しないか、channelがその間に閉じられている場合：
    - それ自身のchannelsの1つに対応しないchannel_updateを無視しなければならない。

  - SHOULD accept `channel_update`s for its own channels (even if non-public),
  in order to learn the associated origin nodes' forwarding parameters.

  - 関連する起点nodesの転送パラメータを知るために、（たとえ非公開であっても）それ自身のchannelsに対してのchannel_updateを受け入れるべきである。

  - if `signature` is not a valid signature, using `node_id` of the
  double-SHA256 of the entire message following the `signature` field (including
  unknown fields following `fee_proportional_millionths`):
    - MUST NOT process the message further.
    - SHOULD fail the connection.

  - signatureが有効なsignatureではない場合（node_id（の秘密鍵）を使用して、fee_proportional_millionthsの次の未知のフィールドを含む、signatureフィールドの後に続くmessage全体のdouble-SHA256（への署名））。
    - messageをさらに処理してはならない。
    - 接続に失敗すべきである。

  - if the specified `chain_hash` value is unknown (meaning it isn't active on
  the specified chain):
    - MUST ignore the channel update.

  - 指定されたchain_hash値が不明な場合（指定されたチェーン上でアクティブでないことを意味する）
    - channel updateを無視しなければならない。

  - if `timestamp` is NOT greater than that of the last-received
  `channel_announcement` for this `short_channel_id` AND for `node_id`:
    - SHOULD ignore the message.

  - timestampが、最後に受信したこのshort_channel_idとnode_idのchannel_announcementよりも大きくない場合：
    - messageを無視すべきである。
    （channel_announcementにはtimestampがないのでは？？？channel_updateの間違い？？？）

  - otherwise:
    - if the `timestamp` is equal to the last-received `channel_announcement`
    AND the fields (other than `signature`) differ:
    （channel_updateの間違い？？？）
      - MAY blacklist this `node_id`.
      - MAY forget all channels associated with it.

  - そうでなければ：
    - timestampが、最後に受信にしたchannel_announcementに等しく、 かつ（signature以外の）フィールドが異なる場合：
    （channel_updateの間違い？？？）
      - このnode_idをブラックリストに入れてよい。
      - それに関連する全てのchannelsを忘れてよい。

  - if the `timestamp` is unreasonably far in the future:
    - MAY discard the `channel_announcement`.
    （channel_updateの間違い？？？）

  - timestampが不合理に未来に遠い場合は：
    - そのchannel_announcementを捨てても良い。
    （channel_updateの間違い？？？）

  - otherwise:
    - SHOULD queue the message for rebroadcasting.
    - MAY choose NOT to for messages longer than the minimum expected length.

  - そうでなければ：
    - 再放送のためにmessagesをキューに入れるべきである。
    - 期待される最小の長さ以上のmessagesを待ち行列に入れないことを選択してもよい。

### Rationale

The `timestamp` field is used by nodes for pruning `channel_update`s that are
either too far in the future or have not been updated in two weeks; so it
makes sense to have it be a UNIX timestamp (i.e. seconds since UTC
1970-01-01). This cannot be a hard requirement, however, given the possible case
of two `channel_update`s within a single second.

このtimestampフィールドは、未来にあまりにも遠すぎるかまたは2週間updateされていないchannel_updateのプルーニングのために、
nodesによって使用されます。
UNIX timestamp（つまり、UTC 1970-01-01以降の秒数）にすることは理にかなっています。
1秒間に2つchannel_updateがあるとしても、これは厳しい要件ではありません。

## Initial Sync

### Requirements

An endpoint node:
endpoint node：
（？？？）

  - upon establishing a connection:
    - SHOULD set the `init` message's `initial_routing_sync` flag to 1, to
    negotiate an initial sync.

  - 接続の確立時に：
    - init messageのinitial_routing_syncフラグを1に設定して初期同期をネゴシエートすべきである。

  - if it requires a full copy of the other endpoint's routing state:
    - SHOULD set the `initial_routing_sync` flag to 1.

  - 他のendpointのルーティング状態の完全なコピーが必要な場合：
    - initial_routing_syncフラグを1に設定すべきである。

  - upon receiving an `init` message with the `initial_routing_sync` flag set to
  1:
    - SHOULD send `channel_announcement`s, `channel_update`s and
    `node_announcement`s for all known channels and nodes, as if they were just
    received.

  - initial_routing_syncフラグが1にセットされたinit messageを受信すると、
    - すべての既知のchannelsとnodesのためのchannel_announcementとchannel_updateとnode_announcementを、それらをちょうど受信したように送信すべきである。（送信順番を崩さないということ？？？）

  - if the `initial_routing_sync` flag is set to 0, OR if the initial sync was
  completed:
    - SHOULD resume normal operation, as specified in the following
    [Rebroadcasting](#rebroadcasting) section.

  - initial_routing_syncフラグが0に設定されているか、または初期同期が完了した場合：
    次の[Rebroadcasting]セクションで指定されているように、normal operationを再開すべきである。
    （Initial Sync以降をnormal operationと言っているのであろう。）

## Rebroadcasting

### Requirements

The final node:
終端node：
（final nodeとendpoint nodeの違いがわからない？？？）

  - upon receiving a new `channel_announcement` or a `channel_update` or
  `node_announcement` with an updated `timestamp`:
    - SHOULD update its local view of the network's topology accordingly.

  - updateしてtimestampを持つ、新しいchannel_announcementかchannel_updateかnode_announcementを受信すると：
    - それに従って、ネットワークのトポロジのローカルビューをupdateすべきである。

  - after applying the changes from the announcement:
    - if there are no channels associated with the corresponding origin node:
      - MAY purge the origin node from its set of known nodes.
    - otherwise:
      - SHOULD update the appropriate metadata AND store the signature
      associated with the announcement.
        - Note: this will later allow the final node to rebuild the announcement
        for its peers.

  - announcementからの変更を適用した後：
    - 対応する起点nodeに関連付けられたchannelがない場合は、
      - 起点nodeをその既知のnodesのセットからパージしてもよい。
    - そうでなければ：
      - 適切なメタデータをupdateし、announcementに関連付けられたsignatureを格納すべきである。
        - 注：これにより、後で終端nodeがそれのpeersのannouncementを再構築できるようになる。
        （？？？）

An endpoint node:
endpoint node：

  - SHOULD flush outgoing announcements once every 60 seconds, independently of
  the arrival times of announcements.
    - Note: this results in staggered announcements that are unique (not
    duplicated).

  - announcementsの到着時刻とは無関係に、60秒おきに、発信announcementsをフラッシュすべきである。
  （一度reboroadcastingで送ったannouncementsは以降同じnodesには送らない？？？）
    注：これにより、staggeredなannouncementsが一意に（重複しないように）なる。
    （staggeredって重複しているのに意味があるのでは？？？）

  - MAY re-announce its channels regularly.
    - Note: this is discouraged, in order to keep the resource requirements low.

  - 定期的にchannelsをre-announceしてもよい。
    - 注：リソース要件を低く抑えるため、これはお勧めしない。

  - upon connection establishment:
    - SHOULD send all `channel_announcement` messages, followed by the latest
    `node_announcement` AND `channel_update` messages.

  - 接続確立時：
    - すべてのchannel_announcement messageを送信し、続けて最新のnode_announcementとchannel_update messagesを送信すべきである。
    (Initial Syncを行う？？？)

### Rationale

Once the announcement has been processed, it's added to a list of outgoing
announcements, destined for the processing node's peers, replacing any older
updates from the origin node. This list of announcements will be flushed at
regular intervals: such a store-and-delayed-forward broadcast is called a
_staggered broadcast_. Also, such batching of announcements forms a natural rate
limit with low overhead.

announcementが処理されると、処理nodeのpeers向けの発信announcementsのリストに追加され、
元の起点nodeからの古いupdatesが置き換えられます。
このannouncementsのリストは定期的にフラッシュされる。
そのようなstore-and-delayed-forward broadcastは、staggered broadcastと呼ばれる。
（ちょっと違くない？？？[staggered broadcast](https://pdfs.semanticscholar.org/0888/3486a96150da7664d8c4dd932f27272c0d7f.pdf
)特定のpeers間でのことでなくて、ネットワーク全体でこんな動きになるということ？？？）
また、そのようなannouncementsのバッチ処理は、低いオーバヘッドで自然なレート制限を形成する。

The sending of all announcements on reconnection is naive, but simple,
and allows bootstrapping for new nodes as well as updating for nodes that
have been offline for some time.

再接続時にすべてのannouncementsを送信するのは素朴ですが、簡単で、新しいnodesのブートストラップと、
しばらくの間オフラインだったnodesのupdateが可能である。

## HTLC Fees

### Requirements

The origin node:
起点ノード：

  - SHOULD accept HTLCs that pay a fee equal to or greater than:
    - fee_base_msat + ( amount_msat * fee_proportional_millionths / 1000000 )
  - SHOULD accept HTLCs that pay an older fee, for some reasonable time after
  sending `channel_update`.
    - Note: this allows for any propagation delay.

  - 次のfee以上のfeeを支払うHTLCを受け入れるべきである：
    - fee_base_msat +（amount_msat * fee_proportional_millionths / 1000000）
    （(fee_proportional_millionths / 1000000)は1satoshi分にしている。）
    （fee_base_msat、fee_proportional_millionthsは自分の（channel_update）のもの）
  - channel_update送信後、適切な時間、古いfeeを支払うHTLCを受け入れるべきである。
    - 注：これは伝搬遅延を許容する。

## Pruning the Network View

### Requirements

A node:
ノード：

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

An endpoint node:
endpoint node：

  - if a channel's latest `channel_update`s `timestamp` is older than two weeks
  (1209600 seconds):
    - MAY prune the channel.
    - MAY ignore the channel.
    - Note: this is an endpoint node policy and MUST NOT be enforced by
    forwarding peers, e.g. by closing channels when receiving outdated gossip
    messages. [ FIXME: is this intended meaning? ]

  - channelの最新channel_updateのtimestampが2週間（1209600秒）より古い場合：
    - channelをプルーニングしてよい。
    - channelを無視してよい。
    - 注：これはendpoint nodeのポリシーであり、転送peersによって強制されてはならない
    （例えば、古くなったgossip messagesを受信したときにchannlesを閉じるなど）。
    [FIXME：これは意図した意味ですか？]
    （？？？）

#### Rationale

Several scenarios may result in channels becoming unusable and its endpoints
becoming unable to send updates for these channels. For example, this occurs if
both endpoints lose access to their private keys and can neither sign
`channel_update`s nor close the channel on-chain. In this case, the channels are
unlikely to be part of a computed route, since they would be partitioned off
from the rest of the network; however, they would remain in the local network
view would be forwarded to other peers indefinitely.

いくつかのシナリオでは、channelsが使用不能になり、そのendpointがこれらのchannelsのupdatesを送信できなくなるだろう。
（endpoints？？？）
たとえば、両方のendpointが秘密鍵へのアクセスを失い、channel_updateに署名することも、オンチェーンのchannelを閉じることもできません 。
この場合、このchannelsはネットワークの他の部分から分離されるので、計算された経路の一部になる可能性は低い。
ただし、ローカルネットワークビューに残っていると、他のpeersに無期限に転送されます。
（peers？？？）

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
HTLCのCLTV、周囲のネットワークトポロジ、およびcltv_expiry_deltasを知ることで、攻撃者は意図された受信者を推測することができる。
したがって、意図された受信者が受信するCLTVにランダムオフセットを追加することが非常に望ましく、これはルートに沿ってすべてのCLTVを押し上げます。

In order to create a plausible offset, the origin node MAY start a limited
random walk on the graph, starting from the intended recipient and summing the
`cltv_expiry_delta`s, and use the resulting sum as the offset.
This effectively creates a _shadow route extension_ to the actual route and
provides better protection against this attack vector than simply picking a
random offset would.

尤もらしいオフセットを作成するために、起点nodeはグラフ上の限定されたランダムウォークを開始し、目的の受信者から開始してcltv_expiry_deltaを合計し、その結果の合計をオフセットとして使用してよい。
（限定されたランダムウォーク？？？合計ではなく平均？？？）
これにより、実際のルートへのshadow route extensionが効果的に作成され、単純にランダムオフセットを選択するよりも、この攻撃ベクタに対する保護が向上する。（shadow route extensionは、あたかも追加のルートがあるように計算する。）

Other more advanced considerations involve diversification of route selection,
to avoid single points of failure and detection, and balancing of local
channels.

他の高度な考慮事項には、ルート選択の多様化、単一障害点の回避と検出、およびローカルchannelsのバランシングが含まれます。
（local？？？）

### Routing Example

Consider four nodes:
4つのnodesを考えます：

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

それぞれは、すべてのチャンネルの最後に、次のcltv_expiry_deltaをadvetiseする。

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

ネットワークには8つのchannel_update messagesが見えます。

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

B→C。Bが4,999,999millisatoshiを直接Cに送っても、それはfeeをチャージすることも、それ自体のcltv_expiry_deltaを追加することもないので、Cの9のmin_final_cltv_expiryの要求を使用することになる。
おそらく42の余分なCLTVを与えるshadow routeも追加するだろう。
さらに、これらの値は最小値を表すので、他のホップで余分なCLTVデルタを追加することができるが、ここでは単純化のために選択しない。

   * `amount_msat`: 4999999
   * `cltv_expiry`: current-block-height + 9 + 42
   * `onion_routing_packet`:
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 9 + 42

**A->B->C.** If A were to send 4,999,999 millisatoshi to C via B, it needs to
pay B the fee it specified in the B->C `channel_update`, calculated as
per [HTLC Fees](#htlc_fees):

A→B→C。もしAが4,999,999millisatoshiをB経由でCに送るなら、
[HTLC Fees]として計算した、B→Cのchannel_updateで指定したfeeをBに支払う必要がある：

        fee_base_msat + ( amount_msat * fee_proportional_millionths / 1000000 )

	200 + ( 4999999 * 2000 / 1000000 ) = 10199

Similarly, it would need to add B->C's `channel_update` `cltv_expiry` (20), C's
requested `min_final_cltv_expiry` (9), and the cost for the _shadow route_ (42).
Thus, A->B's `update_add_htlc` message would be:

同様に、B→Cのchannel_updateのcltv_expiry（20）、Cが要求するmin_final_cltv_expiry（9）、およびshadow route（42）のコストを追加する必要がある。
したがって、A→Bのupdate_add_htlc messageは次のようになる：

   * `amount_msat`: 5010198
   * `cltv_expiry`: current-block-height + 20 + 9 + 42
   * `onion_routing_packet`:
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 9 + 42

B->C's `update_add_htlc` would be the same as B->C's direct payment above.

B→Cのupdate_add_htlcは上記のB→Cのdirect paymentと同じである。

**A->D->C.** Finally, if for some reason A chose the more expensive route via D,
A->D's `update_add_htlc` message would be:

A→D→C。最後に、何らかの理由でAがD経由でより高価なルートを選択した場合、A→Dのupdate_add_htlc messageは次のようになる：

   * `amount_msat`: 5020398
   * `cltv_expiry`: current-block-height + 40 + 9 + 42
   * `onion_routing_packet`:
	 * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 9 + 42

And D->C's `update_add_htlc` would again be the same as B->C's direct payment
above.

そして、D→Cのupdate_add_htlcは、上記のB→Cのdirect paymentと同じである。

## References

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
