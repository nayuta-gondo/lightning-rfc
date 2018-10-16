# BOLT #11: Invoice Protocol for Lightning Payments

（ほとんどGoogle翻訳まま。意味が通らないところだけ修正。） （ところどころコメント有。）

A simple, extendable QR-code-ready protocol for requesting payments
over Lightning.

Lightning上の支払いを要求するためのシンプルで拡張可能なQRコード対応プロトコル

# Table of Contents

  * [Encoding Overview](#encoding-overview)
  * [Human-Readable Part](#human-readable-part)
  * [Data Part](#data-part)
    * [Tagged Fields](#tagged-fields)
  * [Payer / Payee Interactions](#payer--payee-interactions)
    * [Payer / Payee Requirements](#payer--payee-requirements)
  * [Implementation](#implementation)
  * [Examples](#examples)
  * [Authors](#authors)

# Encoding Overview

The format for a Lightning invoice uses
[bech32 encoding](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki),
which is already used for Bitcoin Segregated Witness. It can be
simply reused for Lightning invoices even though its 6-character checksum is optimized
for manual entry, which is unlikely to happen often given the length
of Lightning invoices.

Lightning invoiceのフォーマットは、Bitcoin Segregated Witnessですでに使用されているbech32 encodingを使用する。
6桁のchecksumの手入力用の最適化さえも、Lightning invoicesのために再利用することができるが、
Lightning invoicesの長さを考慮してそのようにすることが頻繁に発生する可能性は低い。

If a URI scheme is desired, the current recommendation is to either
use 'lightning:' as a prefix before the BOLT-11 encoding (note: not
'lightning://'), or for fallback to bitcoin payments to use 'bitcoin:',
as per BIP-21, with the key 'lightning' and the value equal to the BOLT-11
encoding.

URI schemeが望まれる場合、現在推奨されているのは、BOLT-11 encodingの前にprefixとして 'lightning：'を使用することである（注： 'lightning：//'ではない）。
または、'bitcoin:'を使ったビットコインの支払いの代用として、
キー'lightning'とBOLT-11 encodingに等しい値とともに、BIP-21のとおりにする。

## Requirements

A writer MUST encode the payment request in Bech32 as specified in
BIP-0173, with the exception that the Bech32 string MAY be longer than
the 90 characters specified there. A reader MUST parse the address as
Bech32 as specified in BIP-0173 (also without the character limit),
and MUST fail if the checksum is incorrect.

BIP-0173で指定されているように、Bech32 stringがそこに指定された90文字より長いかもしれないという例外を除いて、
作者は支払い要求をBech32でencodeしなければならない。
読者はBIP-0173で指定されているようにBech32としてアドレスをparseしなければならず（同じく文字の制限なしで）、
checksumが正しくなければ失敗しなければならない。

# Human-Readable Part

The human-readable part of a Lightning invoice consists of two sections:
1. `prefix`: `ln` + BIP-0173 currency prefix (e.g. `lnbc` for bitcoin mainnet, `lntb` for bitcoin testnet and `lnbcrt` for bitcoin regtest)
1. `amount`: optional number in that currency, followed by an optional
   `multiplier` letter

Lightning invoiceの可読部分は、次の2つのsectionsで構成されている：
1. prefix：ln + BIP-0173 currency prefix
（
bitcoin mainnetの場合は lnbc、
bitcoin testnetの場合は lntb、
bitcoin regtestの場合は lnbcrt
）
1. amount：その通貨でのoptionalな数にoptionalなmultiplier文字が続く

The following `multiplier` letters are defined:

次のmultiplier文字が定義されている：

* `m` (milli): multiply by 0.001
* `u` (micro): multiply by 0.000001
* `n` (nano): multiply by 0.000000001
* `p` (pico): multiply by 0.000000000001

（区切り）

* `m`（milli）：0.001を掛ける
* `u`（micro）：0.000001を掛ける
* `n`（nano）：0.000000001を掛ける
* `p`（pico）：0.000000000001を掛ける

## Requirements

A writer:
  - MUST encode `prefix` using the currency it requires for successful payment
  - If it requires a specific minimum amount for successful payment:
	- MUST include that `amount`
	- MUST encode `amount` as a positive decimal integer with no leading zeroes
	- SHOULD use the shortest representation possible by using the largest
	  multiplier or omitting the multiplier

作者：    
  - 成功裏に支払うために必要な通貨を使用してprefixを符号化しなければならない
  - 支払いに必要な最小額が必要な場合：
    - amountを含めなければならない
    - amountを先行ゼロのない正の10進整数として符号化しなければならない
    - 可能な限り最短の表現を用いるべきである、最大の乗数を使用するか、乗数を省略して

A reader:
  - MUST fail if it does not understand the `prefix`
  - If the `amount` is empty:
	- SHOULD indicate if amount is unspecified
  - Otherwise:
	- MUST fail if `amount` contains a non-digit or is followed by
      anything except a `multiplier` in the table above
    - If the `multiplier` is present:
	  - MUST multiply `amount` by the `multiplier` value to derive the
        amount required for payment

読者：
  - prefixが理解できなければ失敗しなければならない
  - amountが空の場合：
    - amountが指定されていないかどうかを示すべきである
  - そうでなければ：
    - amountに数字以外の数字が含まれている場合、
    または上記の表のmultiplier以外の文字が続く場合は失敗しなければならない
    - multiplierが存在する場合：
      - 支払いに必要な金額を導出するためにamountにmultiplier値を掛けなければならない

## Rationale

The `amount` is encoded into the human readable part, as it's fairly
readable and a useful indicator of how much is being requested.

amountは、可読部分にencodeされる。
非常に読みやすく、どれくらいの量が要求されているかを示す有用な指標である。

Donation addresses often don't have an associated amount, so `amount`
is optional in that case. Usually a minimum payment is required for
whatever is being offered in return.

寄付アドレスには関連するamountがないことが多いので、そのケースではamountはoptionalである。
おつりで提供されるものは、通常最低限の支払いが必要である。

# Data Part

The data part of a Lightning invoice consists of multiple sections:

Lightning invoiceのデータ部分は、複数のsectionsで構成されている：

1. `timestamp`: seconds-since-1970 (35 bits, big-endian)
1. zero or more tagged parts
1. `signature`: bitcoin-style signature of above (520 bits)

（区切り）

1. timestamp：1970年からの秒（35 bits, big-endian）
1. ゼロ個以上のタグ付けされた部品
1. signature：上記のbitcoin-style signature（520 bits）

## Requirements

A writer MUST set `timestamp` to
the number of seconds since Midnight 1 January 1970, UTC in
big-endian. A writer MUST set `signature` to a valid
512-bit secp256k1 signature of the SHA2 256-bit hash of the
human-readable part, represented as UTF-8 bytes, concatenated with the
data part (excluding the signature) with zero bits appended to pad the
data to the next byte boundary, with a trailing byte containing
the recovery ID (0, 1, 2 or 3).

作者はtimestampに、big-endianのUTCで1970年1月1日の真夜中0時からの秒数を設定しなければならない。
作者はsignatureに、UTF-8のバイト列で表現された可読部分のSHA2 256-bit hashの、
有効な512-bit secp256k1 signature（UTF-8バイトで表される）設定する。
可読部分には、データ部分（signatureを除く）にデータを次のバイト境界にパディングするためにゼロビットを付加し、
recovery ID（0、1、2または3）を含む末尾バイトを付加される。

A reader MUST check that the `signature` is valid (see the `n` tagged
field specified below).

読者は、signatureが有効であることをチェックしなければならない（下記のnタグ付きフィールドを参照）。

## Rationale

`signature` covers an exact number of bytes even though the SHA-2
standard actually supports hashing in bit boundaries, because it's not widely
implemented. The recovery ID allows public-key recovery, so the
identity of the payee node can be implied.

SHA-2標準が実際にはビット境界でのハッシュをサポートしていても、それは広範には実装されていないため、
signatureはバイトの整数をカバーする。
recovery IDは被支払人nodeの身元を暗示することができるためpublic-keyの回復を可能にする。
（？？？）

## Tagged Fields

Each Tagged Field is of the form:

各タグ付きフィールドの形式は次のとおりである：
（typeがタグ。Bech32。32進数表現）

1. `type` (5 bits)
1. `data_length` (10 bits, big-endian)
1. `data` (`data_length` x 5 bits)

Currently defined tagged fields are:

現在定義されているタグ付きフィールドは次のとおりである：

* `p` (1): `data_length` 52. 256-bit SHA256 payment_hash. Preimage of this provides proof of payment
* `d` (13): `data_length` variable. Short description of purpose of payment (UTF-8),  e.g. '1 cup of coffee' or 'ナンセンス 1杯'
* `n` (19): `data_length` 53. 33-byte public key of the payee node
* `h` (23): `data_length` 52. 256-bit description of purpose of payment (SHA256). This is used to commit to an associated description that is over 639 bytes, but the transport mechanism for the description in that case is transport specific and not defined here.
* `x` (6): `data_length` variable. `expiry` time in seconds (big-endian). Default is 3600 (1 hour) if not specified.
* `c` (24): `data_length` variable. `min_final_cltv_expiry` to use for the last HTLC in the route. Default is 9 if not specified.
* `f` (9): `data_length` variable, depending on version. Fallback on-chain address: for bitcoin, this starts with a 5-bit `version` and contains a witness program or P2PKH or P2SH address.
* `r` (3): `data_length` variable. One or more entries containing extra routing information for a private route; there may be more than one `r` field
   * `pubkey` (264 bits)
   * `short_channel_id` (64 bits)
   * `fee_base_msat` (32 bits, big-endian)
   * `fee_proportional_millionths` (32 bits, big-endian)
   * `cltv_expiry_delta` (16 bits, big-endian)

（区切り）

* p（1）：data_length 52. 256-bit SHA256 payment_hash。これのpreimageは支払いの証拠を提供する
* d（13）：data_length variable。支払い目的の簡単な説明（UTF-8）。 「1 cup of coffee」または「ナンセンス 1杯」
* n（19）：data_length 53. 支払先nodeの33-byte public key
* h（23）：data_length 52.支払い目的の256-bit記述（SHA256）。これは、639バイトを超える関連する説明を約束するために使用される、その場合の説明のトランスポートメカニズムはトランスポート固有であり、ここでは定義されない。
* x（6）：data_length variable。expiry 期限（秒）（big-endian）。指定されていない場合、デフォルトは3600（1時間）である。
* c（24）：data_length variable。ルートの最後のHTLCに使用するmin_final_cltv_expiry。指定されていない場合、デフォルトは9。
* f（9）：バージョンに応じてdata_length variable。代用のfallback address：bitcoinの場合、これは5-bit versionから始まり、witness programまたはP2PKHまたはP2SH addressを含む。
* r（3）：data_length variable。private routeに関する追加のルーティング情報を含む1つまたは複数のエントリ。複数のrフィールドが存在する可能性がある
（以下を全部含めてrフィールドとなるのであろう）
   * pubkey（264 bits）
   * short_channel_id（64 bits）
   * fee_base_msat（32 bits, big-endian）
   * fee_proportional_millionths（32 bits, big-endian）
   * cltv_expiry_delta（16 bits, big-endian）

### Requirements

A writer MUST include exactly one `p` field, and set `payment_hash` to
the SHA-2 256-bit hash of the `payment_preimage` that will be given
in return for payment.

作者はちょうど1つのpフィールドを含めなければならず、
支払いの代償として与えられるpayment_preimageのSHA-2 256-bit hashにpayment_hashを設定しなければならない。

A writer MUST include either exactly one `d` or exactly one `h` field. If included, a
writer SHOULD make `d` a complete description of
the purpose of the payment, and MUST use a valid UTF-8 string. If included, a writer MUST make the preimage
of the hashed description in `h` available through some unspecified means,
which SHOULD be a complete description of the purpose of the payment.

作者は、ちょうど1つのdまたはちょうど1つのhフィールドのいずれかを含める必要がある。
含まれている場合、作者は支払いの目的を完全に記述するべきで、有効なUTF-8文字列を使用しなければならない。
もし含まれていれば、作者はhに不特定の手段によって利用可能なハッシュされた記述の原像を作成しなければならない。
これは支払い目的の完全な記述であるべきである。
（なんかちょっとおかしい？？？hに入れるのはハッシュ）

A writer MAY include one `x` field.

作者は、1つのxフィールドを含めることができる。

A writer MAY include one `c` field, which MUST be set to the minimum `cltv_expiry` it
will accept for the last HTLC in the route.

作者は、1つのcフィールドを含めることができる。
ルートの最後のHTLCで受け入れる最小のcltv_expiryを設定しなければならない。
（ない場合は9）

A writer SHOULD use the minimum `data_length` possible for `x` and `c` fields.

作者は、xとcのフィールドに可能な限り最小のdata_lengthを使うべきである。

A writer MAY include one `n` field, which MUST be set to the public key
used to create the `signature`.

作者は、1つのnフィールドを含めることができる。
signatureを作成するために使用されたpublic keyに設定されなければならない。

A writer MAY include one or more `f` fields. For bitcoin payments, a writer MUST set an
`f` field to a valid witness version and program, or `17` followed by
a public key hash, or `18` followed by a script hash.

作者は、1つ以上のfフィールドを含めることができる。
bitcoinの支払いの場合、作成者は有効なwitness versionとprogramにfフィールドを設定するか、
17に続けてpublic key hashを指定するか、18に続いてscript hashを設定しなければならない。

A writer MUST include at least one `r` field if there is not a
public channel associated with its public key. The `r` field MUST contain
one or more ordered entries, indicating the forward route from a
public node to the final destination. For each entry, the `pubkey` is the
node ID of the start of the channel; `short_channel_id` is the short channel ID
field to identify the channel; and `fee_base_msat`, `fee_proportional_millionths`, and `cltv_expiry_delta` are as specified in [BOLT #7](07-routing-gossip.md#the-channel_update-message). A writer MAY include more than one `r` field to
provide multiple routing options.

作者は、public keyに関連付けられたpublic channelがない場合、少なくとも1つのrフィールドを含めなければならない。
rフィールドは、public nodeから最終的な宛先への順方向ルートを示す、1つ以上の順序付けられたエントリを含めなければならない。
各エントリについて、
pubkeyはchannelの始点のnode IDである；
short_channel_idは、channelを識別する短いchannel IDフィールドである；
fee_base_msat、fee_proportional_millionths、およびcltv_expiry_deltaは、BOLT＃7で指定されたとおりである。
作者は、複数のルーティングオプションを提供するために複数のフィールドを含めることができる。
（channel_updateのフィールドを含めるのであろう）

A writer MUST pad field data to a multiple of 5 bits, using zeroes.

作者は、フィールドデータを0を使って5ビットの倍数にパディングしなければならない。

If a writer offers more than one of any field type, it MUST specify
the most-preferred field first, followed by less-preferred fields in
order.

作者が複数のフィールドタイプを提供する場合は、最も優先順位の高いフィールドを最初に指定しなければならず、
優先順位の低いフィールドを順番に指定しなければならない。

A reader MUST skip over unknown fields, an `f` field with unknown
`version`, or a `p`, `h`, or `n` field that does not have `data_length` 52,
52, or 53 respectively.

読者は、未知のフィールド、未知のバージョンを持つfフィールド、
またはdata_length 52、52または53をそれぞれ持たないp、h、またはnフィールドをスキップしなければならない。

A reader MUST check that the SHA-2 256 in the `h` field exactly
matches the hashed description.

読者は、hフィールドのSHA-2 256がハッシュされた記述と正確に一致することをチェックしなければならない。

A reader MUST use the `n` field to validate the signature instead of
performing signature recovery if a valid `n` field is provided.

有効なnフィールドが提供されている場合、署名回復を実行するのではなく、
署名を検証するために読者はnフィールドを使用しなければならない。
（signature recoveryってなに？？？）

### Rationale

The type-and-length format allows future extensions to be backward
compatible. `data_length` is always a multiple of 5 bits, for easy
encoding and decoding. For fields that we expect may change, readers
also ignore ones of different length.

type-and-lengthフォーマットにより、将来の拡張機能に下位互換性を持たせることができる。
data_lengthは、エンコードとデコードを容易にするために、常に5ビットの倍数である。
変更が予想されるフィールドについて、読者は異なる長さのものも無視する。

The `p` field supports the current 256-bit payment hash, but future
specs could add a new variant of different length, in which case
writers could support both old and new, and old readers would ignore
the one not the correct length.

pフィールドは現在の256-bit payment hashをサポートしているが、
将来の仕様では異なる長さの新しい別形を追加できる。
その場合、作者は古いものと新しいものの両方をサポートすることができ、
古い読者は正しい長さではないものを無視する。

The `d` field allows inline descriptions, but may be insufficient for
complex orders; thus the `h` field allows a summary, though the method
by which the description is served is as-yet unspecified and will
probably be transport dependent. The `h` format could change in future
by changing the length, so readers ignore it if it's not 256 bits.

dフィールドはインラインの記述を可能にするが、複雑な注文には不十分であろう。
従って、hフィールドは要約を可能にするが、記述が提供される方法はまだ指定されておらず、おそらくトランスポート依存である。
長さを変えることによって将来hのフォーマットが変わる可能性があるので、読者はそれが256ビットでなければそれを無視する。

The `n` field can be used to explicitly specify the destination node ID,
instead of requiring signature recovery.

nフィールドは、署名回復を要求するのではなく、宛先node IDを明示的に指定するために使用できる。

The `x` field gives warning as to when a payment will be
refused; this is mainly to avoid confusion. The default was chosen
to be reasonable for most payments and to allow sufficient time for
on-chain payment if necessary.

xフィールドは、支払いが拒否される時期を警告する；これは主に混乱を避けるためである。
デフォルトは、ほとんどの支払いに対して合理的であり、
必要に応じてon-chainの支払いに十分な時間を与えるように選択された。
（LNを使わずに支払う）

The `c` field gives a way for the destination node to require a specific
minimum CLTV expiry for its incoming HTLC. Destination nodes may use this
to require a higher, more conservative value than the default one, depending
on their fee estimation policy and their sensitivity to time locks. Note
that remote nodes in the route specify their required `cltv_expiry_delta`
in the `channel_update` message, which they can update at all times.

cフィールドは、着信nodeが着信HTLCに対して特定のminimum CLTV expiryを要求する方法を与える。
宛先nodeは、fee estimation policyとtime locksに対する感応度に応じて、これを使用してデフォルトのものより高い、
より控えめな値を要求することがある。（厳しくないという意味で控えめ）
注：ルート内のremote nodesはchannel_updateメッセージに必要なcltv_expiry_deltaを指定する。
彼らはいつでも更新する可能性がある。

The `f` field allows on-chain fallback. This may not make sense for
tiny or time-sensitive payments, however. It's possible that new
address forms will appear, and so multiple `f` fields in an implied
preferred order help with transition, and `f` fields with versions 19-31
will be ignored by readers.

fフィールドはon-chainの代用を可能にする。
しかし、これは小額または時間に敏感な支払いには意味をなさないかもしれない。
新しいアドレスフォームが現れる可能性があるため、暗黙の優先順の複数のfフィールドが移行に役立ち、
バージョン19-31のfフィールドは読者によって無視される。

The `r` field allows limited routing assistance: as specified it only
allows minimum information to use private channels, but it could also
assist in future partial-knowledge routing.

rフィールドは、限定されたルーティング支援を可能にする：
指定されたように、private channelsを使用するための最小限の情報のみを許すが、
将来の部分知識ルーティングを支援することもできる。

### Security Considerations for Payment Descriptions

Payment descriptions are user-defined and provide a potential avenue for
injection attacks, both in the process of rendering and persistence.

Payment descriptionsはユーザ定義であり、描画処理中と永続化中の両方で、注入攻撃の可能性
を与える。

Payment descriptions should always be sanitized before being displayed in
HTML/Javascript contexts, or any other dynamically interpreted rendering
frameworks. Implementers should be extra perceptive to the possibility of
reflected XSS attacks when decoding and displaying payment descriptions. Avoid
optimistically rendering the contents of the payment request until all
validation, verification, and sanitization have been successfully completed.

Payment descriptionsは、HTML / Javascriptコンテキストまたはその他の動的に解釈されるレンダリングフレームワークで表示される前に、
必ずサニタイズする必要がある。
実装者は、Payment descriptionsをデコードして表示する際に、reflected XSS attacksの可能性に注意を払う必要がある。
すべてのvalidation、verification、およびsanitizationが正常に完了するまで、支払い要求の内容を楽観的に表示しないこと。

Furthermore, consider using prepared statements, input validation, and/or
escaping to protect against injection vulnerabilities against persistence
engines that support SQL or other dynamically interpreted querying languages.

さらに、prepared statements、input validation、
および/または、SQLやその他のdynamically interpreted querying languagesをサポートする永続性エンジンに対する脆弱性注入から保護するために、escapingを使用することを検討しなさい。

* [Stored and Reflected XSS Prevention](https://www.owasp.org/index.php/XSS_%28Cross_Site_Scripting%29_Prevention_Cheat_Sheet)
* [DOM-based XSS Prevention](https://www.owasp.org/index.php/DOM_based_XSS_Prevention_Cheat_Sheet)
* [SQL Injection Prevention](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet)

（上記リンクのURL表記がMarkdownの表記とコンフリクトするので少し変更した）

Don't be like the school of [Little Bobby Tables](https://xkcd.com/327/).

Little Bobby Tablesの学校のようにしないでください。
（SQL Injectionの例。）

# Payer / Payee Interactions

These are generally defined by the rest of the Lightning BOLT series,
but it's worth noting that [BOLT #5](05-onchain.md) specifies that the payee SHOULD
accept up to twice the expected `amount`, so the payer can make
payments harder to track by adding small variations.

これらは通常Lightning BOLTシリーズの他の部分で定義されているが、
注目すべきは、
BOLT＃5が、受取人が期待されるamountの２倍まで受け入れるべきであると指定しており、
支払人は小額のバリエーションを追加することで支払いの追跡を困難にすることができる。
（どういうこと？？？）

The intent is that the payer recover the payee's node ID from the
signature, and after checking that conditions such as fees,
expiry, and block timeout are acceptable, attempt a payment. It can use `r` fields to
augment its routing information if necessary to reach the final node.

その意図は、支払人が署名から受取人のnode IDを復元し、手数料、有効期限、ブロックのタイムアウトなどの条件が満たされていることを確認した後、
支払いを試みることである。（？？？）
rフィールドを使用して、必要に応じて最終nodeに到達するためにルーティング情報を増やすことができる。

If the payment succeeds but there is a later dispute, the payer can
prove both the signed offer from the payee and the successful
payment.

支払いが成功したが、その後の紛争がある場合、支払人は、受取人からの署名付き申し出と成功した支払いの両方を証明することができる。

## Payer / Payee Requirements

A payer SHOULD NOT attempt a payment after the `timestamp` plus
`expiry` has passed. Otherwise, if a Lightning payment fails, a payer
MAY attempt to use the address given in the first `f` field that it
understands for payment. A payer MAY use the sequence of channels
specified by the `r` field to route to the payee. A payer SHOULD consider the
fee amount and payment timeout before initiating payment. A payer
SHOULD use the first `p` field that it did not skip as the payment hash.

支払人は、timestamp + expiryが過ぎた後に支払いを試みるべきではない。
そうでなければ、Lightningの支払いが失敗した場合、
支払人は支払いを理解している最初のfフィールドで与えられたアドレスを使用することができる。
（ここのOtherwiseがわからない？？？期限が切れたあとにやるの？？？）
支払人は、rフィールドで指定されたchannelのシーケンスを使用して受取人にルーティングすることができる。
支払人は支払いを開始する前に手数料と支払いのタイムアウトを考慮すべきである。
支払人は、payment hashとしてスキップしなかった最初のpフィールドを使用すべきである。

A payee SHOULD NOT accept a payment after `timestamp` plus `expiry`.

受取人は、timestamp + expiry満了後に支払いを受け入れるべきではない。

# Implementation

https://github.com/rustyrussell/lightning-payencode

# Examples

NB: all the following examples are signed with `priv_key`=`e126f68f7eafcc8b74f54d269fe206be715000f94dac067d1c04a8ca3b2db734`.

> ### Please make a donation of any amount using payment_hash 0001020304050607080900010203040506070809000102030405060708090102 to me @03e7156ae33b0a208d0744199163177e909e80176e55d97a2f221ede0f934dd9ad
> lnbc1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpl2pkx2ctnv5sxxmmwwd5kgetjypeh2ursdae8g6twvus8g6rfwvs8qun0dfjkxaq8rkx3yf5tcsyz3d73gafnh3cax9rn449d9p5uxz9ezhhypd0elx87sjle52x86fux2ypatgddc6k63n7erqz25le42c4u4ecky03ylcqca784w

Breakdown:

* `lnbc`: prefix, lightning on bitcoin mainnet
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `p`: payment hash
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypq`: payment hash 0001020304050607080900010203040506070809000102030405060708090102
* `d`: short description
  * `pl`: `data_length` (`p` = 1, `l` = 31; 1 * 32 + 31 == 63)
  * `2pkx2ctnv5sxxmmwwd5kgetjypeh2ursdae8g6twvus8g6rfwvs8qun0dfjkxaq`: 'Please consider supporting this project'
* `8rkx3yf5tcsyz3d73gafnh3cax9rn449d9p5uxz9ezhhypd0elx87sjle52x86fux2ypatgddc6k63n7erqz25le42c4u4ecky03ylcq`: signature
* `ca784w`: Bech32 checksum
* Signature breakdown:
  * `38ec6891345e204145be8a3a99de38e98a39d6a569434e1845c8af7205afcfcc7f425fcd1463e93c32881ead0d6e356d467ec8c02553f9aab15e5738b11f127f` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e62630b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404081a1fa83632b0b9b29031b7b739b4b232b91039bab83837b93a34b733903a3434b990383937b532b1ba0` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `c3d4e83f646fa79a393d75277b1d858db1d1f7ab7137dcb7835db2ecd518e1c9` hex of SHA256 of the preimage

> ### Please send $3 for a cup of coffee to the same peer, within 1 minute
> lnbc2500u1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdq5xysxxatsyp3k7enxv4jsxqzpuaztrnwngzn3kdzw5hydlzf03qdgm2hdq27cqv3agm2awhz5se903vruatfhq77w3ls4evs3ch9zw97j25emudupq63nyw24cg27h2rspfj9srp

Breakdown:

* `lnbc`: prefix, lightning on bitcoin mainnet
* `2500u`: amount (2500 micro-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `p`: payment hash...
* `d`: short description
  * `q5`: `data_length` (`q` = 0, `5` = 20; 0 * 32 + 20 == 20)
  * `xysxxatsyp3k7enxv4js`: '1 cup coffee'
* `x`: expiry time
  * `qz`: `data_length` (`q` = 0, `z` = 2; 0 * 32 + 2 == 2)
  * `pu`: 60 seconds (`p` = 1, `u` = 28; 1 * 32 + 28 == 60)
* `aztrnwngzn3kdzw5hydlzf03qdgm2hdq27cqv3agm2awhz5se903vruatfhq77w3ls4evs3ch9zw97j25emudupq63nyw24cg27h2rsp`: signature
* `fj9srp`: Bech32 checksum
* Signature breakdown:
  * `e89639ba6814e36689d4b91bf125f10351b55da057b00647a8dabaeb8a90c95f160f9d5a6e0f79d1fc2b964238b944e2fa4aa677c6f020d466472ab842bd750e` hex of signature data (32-byte r, 32-byte s)
  * `1` (int) recovery flag contained in `signature`
  * `6c6e626332353030750b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404081a0a189031bab81031b7b33332b2818020f00` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `3cd6ef07744040556e01be64f68fd9e1565fb47d78c42308b1ee005aca5a0d86` hex of SHA256 of the preimage

> ### Please send 0.0025 BTC for a cup of nonsense (ナンセンス 1杯) to the same peer, within 1 minute
> lnbc2500u1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpquwpc4curk03c9wlrswe78q4eyqc7d8d0xqzpuyk0sg5g70me25alkluzd2x62aysf2pyy8edtjeevuv4p2d5p76r4zkmneet7uvyakky2zr4cusd45tftc9c5fh0nnqpnl2jfll544esqchsrny

Breakdown:

* `lnbc`: prefix, lightning on bitcoin mainnet
* `2500u`: amount (2500 micro-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `p`: payment hash...
* `d`: short description
  * `pq`: `data_length` (`p` = 1, `q` = 0; 1 * 32 + 0 == 32)
  * `uwpc4curk03c9wlrswe78q4eyqc7d8d0`: 'ナンセンス 1杯'
* `x`: expiry time
  * `qz`: `data_length` (`q` = 0, `z` = 2; 0 * 32 + 2 == 2)
  * `pu`: 60 seconds (`p` = 1, `u` = 28; 1 * 32 + 28 == 60)
* `yk0sg5g70me25alkluzd2x62aysf2pyy8edtjeevuv4p2d5p76r4zkmneet7uvyakky2zr4cusd45tftc9c5fh0nnqpnl2jfll544esq`: signature
* `chsrny`: Bech32 checksum
* Signature breakdown:
  * `259f04511e7ef2aa77f6ff04d51b4ae9209504843e5ab9672ce32a153681f687515b73ce57ee309db588a10eb8e41b5a2d2bc17144ddf398033faa49ffe95ae6` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e626332353030750b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404081a1071c1c571c1d9f1c15df1c1d9f1c15c9018f34ed798020f0` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `197a3061f4f333d86669b8054592222b488f3c657a9d3e74f34f586fb3e7931c` hex of SHA256 of the preimage

> ### Now send $24 for an entire list of things (hashed)
> lnbc20m1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqscc6gd6ql3jrc5yzme8v4ntcewwz5cnw92tz0pc8qcuufvq7khhr8wpald05e92xw006sq94mg8v2ndf4sefvf9sygkshp5zfem29trqq2yxxz7

Breakdown:

* `lnbc`: prefix, lightning on bitcoin mainnet
* `20m`: amount (20 milli-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `p`: payment hash...
* `h`: tagged field: hash of description
  * `p5`: `data_length` (`p` = 1, `5` = 20; 1 * 32 + 20 == 52)
  * `8yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqs`: SHA256 of 'One piece of chocolate cake, one icecream cone, one pickle, one slice of swiss cheese, one slice of salami, one lollypop, one piece of cherry pie, one sausage, one cupcake, and one slice of watermelon'
* `cc6gd6ql3jrc5yzme8v4ntcewwz5cnw92tz0pc8qcuufvq7khhr8wpald05e92xw006sq94mg8v2ndf4sefvf9sygkshp5zfem29trqq`: signature
* `2yxxz7`: Bech32 checksum
* Signature breakdown:
  * `c63486e81f8c878a105bc9d959af1973854c4dc552c4f0e0e0c7389603d6bdc67707bf6be992a8ce7bf50016bb41d8a9b5358652c4960445a170d049ced4558c` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e626332306d0b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404082e1a1c92db7b3f161a001b7689049eea2701b46f8db7513629edf2408fac7eaedc60800` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `b6025e8a10539dddbcbe6840a9650707ae3f147b8dcfda338561ada710508916` hex of SHA256 of the preimage

> ### The same, on testnet, with a fallback address mk2QpYatsKicvFVuTAQLBryyccRXMUaGHP
> lntb20m1pvjluezhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfpp3x9et2e20v6pu37c5d9vax37wxq72un98kmzzhznpurw9sgl2v0nklu2g4d0keph5t7tj9tcqd8rexnd07ux4uv2cjvcqwaxgj7v4uwn5wmypjd5n69z2xm3xgksg28nwht7f6zspwp3f9t

Breakdown:

* `lntb`: prefix, lightning on bitcoin testnet
* `20m`: amount (20 milli-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `h`: tagged field: hash of description...
* `p`: payment hash...
* `f`: tagged field: fallback address
  * `pp`: `data_length` (`p` = 1; 1 * 32 + 1 == 33)
  * `3` = 17, so P2PKH address
  * `x9et2e20v6pu37c5d9vax37wxq72un98`: 160 bit P2PKH address
* `kmzzhznpurw9sgl2v0nklu2g4d0keph5t7tj9tcqd8rexnd07ux4uv2cjvcqwaxgj7v4uwn5wmypjd5n69z2xm3xgksg28nwht7f6zsp`: signature
* `wp3f9t`: Bech32 checksum
* Signature breakdown:
  * `b6c42b8a61e0dc5823ea63e76ff148ab5f6c86f45f9722af0069c7934daff70d5e315893300774c897995e3a7476c8193693d144a36e2645a0851e6ebafc9d0a` hex of signature data (32-byte r, 32-byte s)
  * `1` (int) recovery flag contained in `signature`
  * `6c6e746232306d0b25fe64570d0e496dbd9f8b0d000dbb44824f751380da37c6dba89b14f6f92047d63f576e304021a000081018202830384048000810182028303840480008101820283038404808102421898b95ab2a7b341e47d8a34ace9a3e7181e5726538` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `00c17b39642becc064615ef196a6cc0cce262f1d8dde7b3c23694aeeda473abe` hex of SHA256 of the preimage

> ### On mainnet, with fallback address 1RustyRX2oai4EYYDpQGWvEL62BBGqN9T with extra routing info to go via nodes 029e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255 then 039e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255
> lnbc20m1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqsfpp3qjmp7lwpagxun9pygexvgpjdc4jdj85fr9yq20q82gphp2nflc7jtzrcazrra7wwgzxqc8u7754cdlpfrmccae92qgzqvzq2ps8pqqqqqqpqqqqq9qqqvpeuqafqxu92d8lr6fvg0r5gv0heeeqgcrqlnm6jhphu9y00rrhy4grqszsvpcgpy9qqqqqqgqqqqq7qqzqj9n4evl6mr5aj9f58zp6fyjzup6ywn3x6sk8akg5v4tgn2q8g4fhx05wf6juaxu9760yp46454gpg5mtzgerlzezqcqvjnhjh8z3g2qqdhhwkj

Breakdown:

* `lnbc`: prefix, lightning on bitcoin mainnet
* `20m`: amount (20 milli-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `p`: payment hash...
* `h`: tagged field: hash of description...
* `f`: tagged field: fallback address
  * `pp`: `data_length` (`p` = 1; 1 * 32 + 1 == 33)
  * `3` = 17, so P2PKH address
  * `qjmp7lwpagxun9pygexvgpjdc4jdj85f`: 160 bit P2PKH address
* `r`: tagged field: route information
  * `9y`: `data_length` (`9` = 5, `y` = 4; 5 * 32 + 4 = 164)
    * `q20q82gphp2nflc7jtzrcazrra7wwgzxqc8u7754cdlpfrmccae92qgzqvzq2ps8pqqqqqqpqqqqq9qqqvpeuqafqxu92d8lr6fvg0r5gv0heeeqgcrqlnm6jhphu9y00rrhy4grqszsvpcgpy9qqqqqqgqqqqq7qqzq`:
      * pubkey: `029e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255`
      * `short_channel_id`: 0102030405060708
      * `fee_base_msat`: 1 millisatoshi
      * `fee_proportional_millionths`: 20
      * `cltv_expiry_delta`: 3
      * pubkey: `039e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255`
      * `short_channel_id`: 030405060708090a
      * `fee_base_msat`: 2 millisatoshi
      * `fee_proportional_millionths`: 30
      * `cltv_expiry_delta`: 4
* `j9n4evl6mr5aj9f58zp6fyjzup6ywn3x6sk8akg5v4tgn2q8g4fhx05wf6juaxu9760yp46454gpg5mtzgerlzezqcqvjnhjh8z3g2qq`: signature
* `dhhwkj`: Bech32 checksum
* Signature breakdown:
  * `91675cb3fad8e9d915343883a49242e074474e26d42c7ed914655689a8074553733e8e4ea5ce9b85f69e40d755a55014536b12323f8b220600c94ef2b9c51428` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e626332306d0b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404082e1a1c92db7b3f161a001b7689049eea2701b46f8db7513629edf2408fac7eaedc60824218825b0fbee0f506e4ca122326620326e2b26c8f448ca4029e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255010203040506070800000001000000140003039e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255030405060708090a000000020000001e00040` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `ff68246c5ad4b48c90cf8ff3b33b5cea61e62f08d0e67910ffdce1edecade71b` hex of SHA256 of the preimage

> ### On mainnet, with fallback (P2SH) address 3EktnHQD7RiAE6uzMj2ZifT9YgRrkSgzQX
> lnbc20m1pvjluezhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfppj3a24vwu6r8ejrss3axul8rxldph2q7z9kmrgvr7xlaqm47apw3d48zm203kzcq357a4ls9al2ea73r8jcceyjtya6fu5wzzpe50zrge6ulk4nvjcpxlekvmxl6qcs9j3tz0469gq5g658y

Breakdown:

* `lnbc`: prefix, lightning on bitcoin mainnet
* `20m`: amount (20 milli-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `h`: tagged field: hash of description...
* `p`: payment hash...
* `f`: tagged field: fallback address
  * `pp`: `data_length` (`p` = 1; 1 * 32 + 1 == 33)
  * `j` = 18, so P2SH address
  * `3a24vwu6r8ejrss3axul8rxldph2q7z9`:  160 bit P2SH address
* `kmrgvr7xlaqm47apw3d48zm203kzcq357a4ls9al2ea73r8jcceyjtya6fu5wzzpe50zrge6ulk4nvjcpxlekvmxl6qcs9j3tz0469gq`: signature
* `5g658y`: Bech32 checksum
* Signature breakdown:
  * `b6c6860fc6ff41bafba1745b538b6a7c6c2c0234f76bf817bf567be88cf2c632492c9dd279470841cd1e21a33ae7ed59b25809bf9b3366fe81881651589f5d15` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e626332306d0b25fe64570d0e496dbd9f8b0d000dbb44824f751380da37c6dba89b14f6f92047d63f576e304021a000081018202830384048000810182028303840480008101820283038404808102421947aaab1dcd0cf990e108f4dcf9c66fb437503c228` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `64f1ff500bcc62a1b211cd6db84a1d93d1f77c6a132904465b6ff912420176be` hex of SHA256 of the preimage

> ### On mainnet, with fallback (P2WPKH) address bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4
> lnbc20m1pvjluezhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfppqw508d6qejxtdg4y5r3zarvary0c5xw7kepvrhrm9s57hejg0p662ur5j5cr03890fa7k2pypgttmh4897d3raaq85a293e9jpuqwl0rnfuwzam7yr8e690nd2ypcq9hlkdwdvycqa0qza8

* `lnbc`: prefix, lightning on bitcoin mainnet
* `20m`: amount (20 milli-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `h`: tagged field: hash of description...
* `p`: payment hash...
* `f`: tagged field: fallback address
  * `pp`: `data_length` (`p` = 1; 1 * 32 + 1 == 33)
  * `q`: 0, so witness version 0
  * `w508d6qejxtdg4y5r3zarvary0c5xw7k`: 160 bits = P2WPKH.
* `epvrhrm9s57hejg0p662ur5j5cr03890fa7k2pypgttmh4897d3raaq85a293e9jpuqwl0rnfuwzam7yr8e690nd2ypcq9hlkdwdvycq`: signature
* `a0qza8`: Bech32 checksum
* Signature breakdown:
  * `c8583b8f65853d7cc90f0eb4ae0e92a606f89caf4f7d65048142d7bbd4e5f3623ef407a75458e4b20f00efbc734f1c2eefc419f3a2be6d51038016ffb35cd613` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e626332306d0b25fe64570d0e496dbd9f8b0d000dbb44824f751380da37c6dba89b14f6f92047d63f576e304021a00008101820283038404800081018202830384048000810182028303840480810242103a8f3b740cc8cb6a2a4a0e22e8d9d191f8a19deb0` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `b3df27aaa01d891cc9de272e7609557bdf4bd6fd836775e4470502f71307b627` hex of SHA256 of the preimage

> ### On mainnet, with fallback (P2WSH) address bc1qrp33g0q5c5txsp9arysrx4k6zdkfs4nce4xj0gdcccefvpysxf3qccfmv3
> lnbc20m1pvjluezhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfp4qrp33g0q5c5txsp9arysrx4k6zdkfs4nce4xj0gdcccefvpysxf3q28j0v3rwgy9pvjnd48ee2pl8xrpxysd5g44td63g6xcjcu003j3qe8878hluqlvl3km8rm92f5stamd3jw763n3hck0ct7p8wwj463cql26ava

* `lnbc`: prefix, lightning on bitcoin mainnet
* `20m`: amount (20 milli-bitcoin)
* `1`: Bech32 separator
* `pvjluez`: timestamp (1496314658)
* `h`: tagged field: hash of description...
* `p`: payment hash...
* `f`: tagged field: fallback address
  * `p4`: `data_length` (`p` = 1, `4` = 21; 1 * 32 + 21 == 53)
  * `q`: 0, so witness version 0
  * `rp33g0q5c5txsp9arysrx4k6zdkfs4nce4xj0gdcccefvpysxf3q`: 260 bits = P2WSH.
* `28j0v3rwgy9pvjnd48ee2pl8xrpxysd5g44td63g6xcjcu003j3qe8878hluqlvl3km8rm92f5stamd3jw763n3hck0ct7p8wwj463cq`: signature
* `l26ava`: Bech32 checksum
* Signature breakdown:
  * `51e4f6446e410a164a6da9f39507e730c26241b4456ab6ea28d1b12c71ef8ca20c9cfe3dffc07d9f8db671ecaa4d20beedb193bda8ce37c59f85f82773a55d47` hex of signature data (32-byte r, 32-byte s)
  * `0` (int) recovery flag contained in `signature`
  * `6c6e626332306d0b25fe64570d0e496dbd9f8b0d000dbb44824f751380da37c6dba89b14f6f92047d63f576e304021a00008101820283038404800081018202830384048000810182028303840480810243500c318a1e0a628b34025e8c9019ab6d09b64c2b3c66a693d0dc63194b02481931000` hex of data for signing (prefix + data after separator up to the start of the signature)
  * `399a8b167029fda8564fd2e99912236b0b8017e7d17e416ae17307812c92cf42` hex of SHA256 of the preimage

# Authors

[ FIXME: ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
