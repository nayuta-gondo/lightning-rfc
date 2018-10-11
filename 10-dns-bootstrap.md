# BOLT #10: DNS Bootstrap and Assisted Node Location

This specification describes a node discovery mechanism based on the Domain Name System (DNS).
Its purpose is twofold:

この仕様は、ドメインネームシステム（DNS）に基づくノード発見メカニズムについて説明する。
その目的は2つある：

 - Bootstrap: providing the initial node discovery for nodes that have no known contacts in the network
 - Assisted Node Location: supporting nodes in discovery of the current network address of previously known peers

（XXX: 区切り）

 - ブートストラップ：ネットワークに既知の連絡先がないノードの初期ノード検出を提供する
 - ノード位置のアシスト：既知のピアの現在のネットワークアドレスを発見するノードをサポートする

A domain name server implementing this specification is called a _DNS Seed_, and answers incoming DNS queries of type `A`, `AAAA`, or `SRV` as specified in RFCs 1035<sup>[1](#ref-1)</sup>, 3596<sup>[2](#ref-2)</sup> and 2782<sup>[3](#ref-3)</sup> respectively.
The DNS server is authoritative for a subdomain, called a _seed root domain_, and clients may query it for subdomains.

この仕様を実装するドメインネームサーバはDNS Seedと呼ばれ、
RFC 1035、3596および2782で指定されるA、AAAAまたはSRVタイプの着信DNSクエリに応答する。
DNSサーバーは、seed root domainと呼ばれるサブドメインに対して権限を持ち、クライアントはサブドメインに対してクエリを実行する。

（XXX: A: IPv4 IPアドレスレコード、AAAA: IPv6 IPアドレスレコード、SRV: サービスレコード（`_Service._Proto.Name  TTL Class  SRV Priority  Weight  Port  Target`））

The subdomains consist of a number of dot-separated _conditions_ that further narrow the desired results.

サブドメインは、ドットで区切られた複数の条件で構成され、目的の結果をさらに絞り込む。

## DNS Seed Queries

A client MAY issue queries using the `A`, `AAAA`, or `SRV` query types, specifying conditions for the desired results that the seed should return.

クライアントは、使用してクエリを発行することができる
A、AAAAまたはSRVクエリタイプを使って、クエリを発行できる、
seedが返すべき期待する結果を得るための条件を指定して。

### Query Semantics

The conditions are key-value pairs with a single-letter key; the remainder of the key-value pair is the value.
The following key-value pairs MUST be supported by a DNS seed:

条件は、1文字のキーを持つkey-value pairsである。
key-value pairの残りの部分が値になる。
次のkey-value pairsは、DNSシードがサポートしなければならない。

 - `r`: realm byte, used to specify what realm the returned nodes must support (default value: 0, Bitcoin)
 - `a`: address types, used to specify what address types should be returned for `SRV` queries. This is a bitfield that uses the types from [BOLT #7](07-routing-gossip.md) as bit index. This condition MAY only be used for `SRV` queries. (default value: 6, i.e. `2 || 4`, since bit 1 and bit 2 are set for IPv4 and IPv6, respectively)
 - `l`: `node_id`, the bech32-encoded `node_id` of a specific node, used to ask for a single node instead of a random selection. (default: null)
 - `n`: the number of desired reply records (default: 25)

（XXX: 区切り）

 - r：realm byte、返されるノードがサポートしなければならないrealm（XXX: 領域）を指定するために使用される（デフォルト値：0、Bitcoin）
 - a：address types、SRVクエリに対して返されるべきアドレス型を指定するために使用される。
 これは、ビットインデックスとしてBOLT＃7のタイプを使用するビットフィールドである。
 この条件は、SRVクエリにのみ使用することができる。
 （デフォルト値：6、つまり2 || 4、それぞれIPv4とIPv6にビット1とビット2が設定されているため）
 - l：node_id、特定のノードのbech32-encodedのnode_id、ランダムな選択の代わりに単一のノードを要求するために使用される。
 （デフォルト：null）
 - n：期待する応答レコードの数（デフォルト：25）

Conditions are passed in the DNS seed query as individual, dot-separated subdomain components.

条件は、個々が、ドット区切りのサブドメインコンポーネントとしてDNS seedクエリで渡される。

A query for `r0.a2.n10.lseed.bitcoinstats.com` would mean: Return 10 (`n10`) IPv4 (`a2`) records  for nodes supporting Bitcoin (`r0`).

r0.a2.n10.lseed.bitcoinstats.comのクエリは意味する：Bitcoin（r0）をサポートするノードを10件（n10）IPv4（a2）のレコードを返す。

The DNS seed MUST evaluate the conditions from the _seed root domain_ and going up-the-tree, meaning right-to-left in a fully qualified domain name. In the example above, that would be: `n10`, then `a2`, then `r0`.
If a condition (key) is specified more than once, the DNS seed MUST discard any earlier value for that condition and use the new value instead. For `n5.r0.a2.n10.lseed.bitcoinstats.com`, the result is then: ~~`n10`~~, `a2`, `r0`, `n5`.
Results returned by the DNS seed SHOULD match all conditions.
If the DNS seed does not implement filtering by a given condition it MAY ignore the condition altogether (i.e. the seed filtering is best effort only).
Clients MUST NOT rely on any given condition being met by the results.

DNS seedは、seed root domainの条件がツリーの上の方へ、つまり完全修飾ドメイン名の右から左へ評価しなければならない。
上の例では、それはn10、それからa2、それからr0である。
条件（キー）が2回以上指定されている場合、DNSシードはその条件の以前の値を破棄して、代わりに新しい値を使用しなければならない。

n5.r0.a2.n10.lseed.bitcoinstats.com場合、結果は次のようになる：~~`n10`~~、`a2`、`r0`、`n5`。

DNS seedによって返された結果はすべての条件と一致する。
DNSシードが所定の条件によるフィルタリングを実装していない場合、条件を完全に無視して良い。
（つまり、シードフィルタリングはベストエフォートのみである）。
クライアントは、結果で満たされる任意の条件に依存してはならない。

Queries distinguish between _wildcard_ queries and _node_ queries, depending on whether the `l`-key is set or not.

クエリは、l-keyが設定されているかどうかによって、ワイルドカードクエリとノードクエリを区別される。

Upon receiving a wildcard query, the DNS seed MUST select a random subset of up to `n` IPv4 or IPv6 addresses of nodes that are listening for incoming connections.
For `A` and `AAAA` queries, only nodes listening on the default port 9735, as defined in [BOLT #1](01-messaging.md), MUST be returned.
Since `SRV` records return a _(hostname,port)_-tuple, nodes that are listening on non-default ports MAY be returned.

ワイルドカードクエリを受け取るとDNS seedは、
着信接続をリッスンしているノードのIPv4またはIPv6アドレスの、nまでのランダムサブセットを選択しなければならない。
AとAAAAクエリでは、BOLT＃1で定義されているように、デフォルトのポート9735をリッスンするノードだけが返されなければならない。
SRVレコードは（ホスト名、ポート）のタプルを返すので、デフォルト以外のポートでリッスンしているノードを返すことができる。

Upon receiving a node query, the seed MUST select the record matching the `node_id`, if any, and return all addresses associated with that node.

ノードクエリを受信すると、シードは、もしあれば、node_idに一致するレコードを選択し、そのノードに関連するすべてのアドレスを返さなければならない。

### Reply Construction

The results are serialized in a reply with a query type matching the client's query type, i.e. `A` queries result in `A` replies, `AAAA` queries result in `AAAA` replies, and `SRV` queries result in `SRV` replies, but they may be augmented with additional records (e.g. to add `A` or `AAAA` records matching the returned `SRV` records).

結果は、クライアントのクエリタイプに一致するクエリタイプの応答の中にシリアライズされる、
すなわち、AクエリはA応答を結果とし、AAAAクエリはAAAA応答を結果とし、およびSRVクエリはSRV応答を結果とし、
ただし、彼らはadditional recordsを追加できる（例えば、返されたSRVレコードに一致するAかAAAAレコードを追加）。

For `A` and `AAAA` queries, the reply contains the domain name and the IP address of the results.
The domain name MUST match the domain in the query in order not to be filtered by intermediate resolvers.

AとAAAAクエリの場合、返信にはドメイン名と結果のIPアドレスが含まれる。
中間のリゾルバによってフィルタリングされないように、ドメイン名はクエリのドメインと一致しなければならない。

For `SRV` queries, the reply consists of (_virtual hostnames_, port)-tuples.
A virtual hostname is a subdomain of the seed root domain that uniquely identifies a node in the network.
It is constructed by prepending the `node_id` condition to the seed root domain.
The DNS seed MAY additionally return the corresponding `A` and `AAAA` records that indicate the IP address for the `SRV` entries in the Extra section of the reply.
Due to the large size of the resulting reply, the reply may be dropped by intermediate resolvers, hence the DNS seed MAY omit these additional records upon detecting a repeated query.

SRVクエリの場合、応答は（仮想ホスト名、ポート）のタプルで構成される。
仮想ホスト名は、ネットワーク内のノードを一意に識別する、seed root domainのサブドメインである。
seed root domainの先頭にnode_idの条件を付加することで構築される。
DNS seedは、応答の追加セクションで、SRVエントリのIPアドレスを示す、対応するAおよびAAAAレコードをさらに返すことができる。
返される結果のサイズが大きいために、中間のリゾルバによって応答が破棄される可能性があるため、
DNSシードは繰り返しのクエリを検出したときにこれらの追加レコードを省略してもよい。
（XXX: a repeated queryって具体的にどんな条件？）

Should no entries match all the conditions then an empty reply MUST be returned.

すべての条件に一致するエントリがない場合は、空の応答を返さなければならない。

## Policies

The DNS seed MUST NOT return replies with a TTL lower than 60 seconds.
The DNS seed MAY filter nodes from its local views for various reasons, including faulty nodes, flaky nodes, or spam prevention.
In accordance with the Bitcoin DNS Seed policy<sup>[4](#ref-4)</sup>, replies to random queries (i.e. queries to the seed root domain and to the `_nodes._tcp.` alias for `SRV` queries) MUST be random samples from the set of all known good nodes and MUST NOT be biased.

DNS seedは、60秒未満のTTLを返信してはならない。
DNSシードは、障害のあるノード、信頼できないノード、またはスパム防止を含むさまざまな理由で、
ローカルビューからノードをフィルタリングしてもよい。
Bitcoin DNS Seedポリシーに従って、
無作為なクエリ（すなわち、seed root domainへのクエリとSRVクエリのための`_nodes._tcp.`エイリアスへのクエリ）に対する応答は、
すべての既知の正常なノードのセットからのランダムサンプルでなければならず、偏ってはいけない。
（XXX: なにこのエイリアスって？）

## Examples

Querying for `AAAA` records:

	$ dig lseed.bitcoinstats.com AAAA
	lseed.bitcoinstats.com. 60      IN      AAAA    2a02:aa16:1105:4a80:1234:1234:37c1:9c9

Querying for `SRV` records:

	$ dig lseed.bitcoinstats.com SRV
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 6331 ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 9735 ln1qv2w3tledmzczw227nnkqrrltvmydl8gu4w4d70g9td7avke6nmz2tdefqp.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 9735 ln1qtynyymv99pqf0r9cuexvvqtxrlgejuecf8myfsa96vcpflgll5cqmr2xsu.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 4280 ln1qdfvlysfpyh96apy3w3qdwlu8jjkdhnuxa689ka540tnde6gnx86cf7ga2d.lseed.bitcoinstats.com.
	lseed.bitcoinstats.com. 59   IN      SRV     10 10 4281 ln1qwf789tlcpe4n34649xrqllxt97whsvfk5pm07ggqms3vrjwdj3cu6332zs.lseed.bitcoinstats.com.

Querying for the `A` for the first virtual hostname from the previous example:

	$ dig ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com A
	ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com. 60 IN A 139.59.143.87

Querying for only IPv4 nodes (`a2`) via seed filtering:

	$dig a2.lseed.bitcoinstats.com SRV
	a2.lseed.bitcoinstats.com. 59	IN	SRV	10 10 9735 ln1q2jy22cg2nckgxttjf8txmamwe9rtw325v4m04ug2dm9sxlrh9cagrrpy86.lseed.bitcoinstats.com.
	a2.lseed.bitcoinstats.com. 59	IN	SRV	10 10 9735 ln1qfrkq32xayuq63anmc2zp5vtd2jxafhdzzudmuws0hvxshtgd2zd7jsqv7f.lseed.bitcoinstats.com.

Querying for only IPv6 nodes (`a4`) supporting Bitcoin (`r0`) via seed filtering:

	$dig r0.a4.lseed.bitcoinstats.com SRV
	r0.a4.lseed.bitcoinstats.com. 59 IN	SRV	10 10 9735 ln1qwx3prnvmxuwsnaqhzwsrrpwy4pjf5m8fv4m8kcjkdvyrzymlcmj5dakwrx.lseed.bitcoinstats.com.
	r0.a4.lseed.bitcoinstats.com. 59 IN	SRV	10 10 9735 ln1qwr7x7q2gvj7kwzzr7urqq9x7mq0lf9xn6svs8dn7q8gu5q4e852znqj3j7.lseed.bitcoinstats.com.

## References
- <a id="ref-1">[RFC 1035 - Domain Names](https://www.ietf.org/rfc/rfc1035.txt)</a>
- <a id="ref-2">[RFC 3596 - DNS Extensions to Support IP Version 6](https://tools.ietf.org/html/rfc3596)</a>
- <a id="ref-3">[RFC 2782 - A DNS RR for specifying the location of services (DNS SRV)](https://www.ietf.org/rfc/rfc2782.txt)</a>
- <a id="ref-4">[Expectations for DNS Seed operators](https://github.com/bitcoin/bitcoin/blob/master/doc/dnsseed-policy.md)</a>
