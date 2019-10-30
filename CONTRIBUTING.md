# How to Modify the Specification

Welcome!  This document is a meta-discussion of how the specifications
should be safely altered when you want to include some amazing new
functionality.

ようこそ!このドキュメントでは、驚くべき新機能を追加する場合に、
仕様を安全に変更する方法についてメタ・ディスカッションします。

Please remember that we're all trying to Make Things Better.  Respect,
consideration, kindness and humor have made this process
[fun](00-introduction.md#theme-song) and rewarding and we'd like to keep it
that way.  We're nice!

私たちはより良いものを作ろうとしていることを覚えておいてください。
尊敬、思いやり、優しさ、ユーモアがこのプロセスを楽しく、やりがいのあるものにしているので、
私たちはそれを維持したいと思っています。いいね!

## Extension Design

There are several extension mechanisms in the spec; you should seek to use
them, or introduce new ones if necessary.

仕様にはいくつかの拡張メカニズムがあります;
あなたはそれらを使用するように努めるべきです。
あるいは必要に応じて新しいものを導入する必要があります。

### Adding New Inter-Peer Messages

Unknown odd inter-peer messages are ignored, aka "it's OK to be odd!"
which makes more sense as you get to know me.

未知の奇数のピア・メッセージ「it's OK to be odd!」は無視されます。

If your message is an enhancement, and you don't need to know if the other
side supports it, you should give it an odd number.  If it would be broken
if the other side doesn't support it (ie. Should Never Happen) give it an
even number.  Mistakes happen, and future versions of the software may well
not be tested against ancient versions.

メッセージが拡張機能であり、相手側がそれをサポートしているかどうかを知る必要がない場合は、奇数を指定する必要があります。
反対側がサポートしていなければ中断されるようにする（つまり、起こるべきではない）場合は、偶数にしてください。
間違いは起こりうるし、ソフトウェアの将来のバージョンは、古いバージョンに対してはテストされないかもしれない。

If you want to experiment with new [message types](01-messaging.md#lightning-message-format) internally, I recommend
using 32768 and above (use even, so it will break if these accidentally
escape into the wild).

新しいメッセージ・タイプを内部で試してみたい場合には、
32768以上を使用することをお勧めします
（偶数を使うこと。それらが誤って野生に逃れた場合、それは中断するように）。

### Adding New Feature Bits

[Feature bits](01-messaging.md#the-init-message) are how you know a message is legal to send (see above), and
also they can be used to find appropriate peers.

機能ビットは、送信するメッセージが正当であること（上記参照）を知る方法であり、適切なピアを見つけるためにも使用できます。

Feature bits are always assigned in pairs, even if it doesn't make sense
for them to ever be compulsory.

機能ビットは、たとえそれらが強制的であることに意味がないとしても、常にペアで割り当てられます。
（XXX: 偶数と奇数）

Almost every spec change should have a feature bit associated; in the past
we have grouped feature bits, then we couldn't disable a single feature
when implementations turned out to be broken.

ほとんどすべての仕様変更には、機能ビットを関連付ける必要があります；
過去には機能ビットをグループ化していましたが、実装が壊れていると判明したときには、1つの機能を無効にすることはできませんでした。

Usually feature bits are odd when first deployed, then some become even
when deployment is almost universal.  This often allows legacy code to be
removed, since you'll never talk to peers who can't deal with the feature.

通常、機能ビットは最初に展開されたときに奇数で、展開がほとんど普遍的であるときに偶数になるものもあります。
この機能を扱えない仲間と話すことはないので、これによってレガシーコードが削除されることがよくあります。

If you want to experiment with new feature bits internally, I recommend
using 100 and above.

新しい機能を内部的に試してみたい場合は、100以降を使用することをお勧めします。

### Extending Inter-Peer Messages

The spec says that additional data in messages is ignored, which is another
way we can extend in future.  For BOLT 1.0, optional fields were appended,
and their presence flagged by feature bits.

仕様によると、メッセージ内の追加データは無視されますが、これは将来拡張できるもう1つの方法です。
BOLT 1.0では、オプションフィールドが追加され、フィーチャビットによってその存在がフラグ付けされました。

The modern way to do this is to add a TLV to the end of a message.  This
contains optional fields: again, even means you will only send it if a
feature bit indicates support, odd means it's OK to send to old peers
(often making implementation easier, since peers can send them
unconditionally).

これを行う最近の方法は、メッセージの最後にTLVを追加することです。
これにはオプションのフィールドが含まれています。
再度、偶数は機能ビットがサポートすることを示す場合にのみ送信することを意味し、
奇数は古いピアに送信してもよいことを意味します
（しばしば、ピアが無条件に送信できるため、実装が容易になることが多い）。

## Writing The Spec

The specification is supposed to be readable in text form, readable once
converted to HTML, and digestible by [tools/extract-formats.py].  In
particular, fields should use the correct type and have as much of their
structure as possible described explicitly (avoid 100*byte fields).

この仕様は、テキスト形式で読め、HTMLに変換されると読め、
[tools/extract-formats.py]で要約できるようになっています。
特に、フィールドは正しい型を使用し、明示的に記述されたできるだけ多くの構造を持つべきです
（100*バイトフィールドを避ける）。（XXX: ？）

If necessary, you can modify that tool if you need strange formatting
changes.

書式を変更する必要がある場合は、必要に応じてツールを変更できます。

The output of this tool is used to generate code for several
implementations, and it's also recommended that implementations quote the
spec liberally and have automated testing that the quotes are correct, as
[c-lightning
does](https://github.com/ElementsProject/lightning/blob/master/tools/check-bolt.c).

このツールの出力は、いくつかの実装のためのコードを生成するために使用されます。
また、c-lightningのように、実装が仕様を自由に引用し、引用が正しいことを自動テストすることもお勧めします。

If your New Thing replaces the existing one, be sure to move the existing
one to a Legacy subsection: new readers will want to go straight to the
modern version.  Don't emulate the classic Linux snprintf 1.27 man page:

新しいもので既存のものを置き換える場合は、必ず既存のものをレガシーのサブセクションに移動してください。
古典的なLinux snprintf 1.27のマニュアルページをエミュレートしないでください。

    RETURN VALUE
       If the output was truncated, the return value is -1, otherwise it is the
       number of characters stored, not including the terminating null.   (Thus
       until  glibc  2.0.6.  Since glibc 2.1 these functions return the  number
       of characters (excluding the trailing null) which would have been  writ‐
       ten to the final string if enough space had been available.)

Imagine the bitterness of someone who only reads the first sentence
assuming they have the answer they're looking for!  Someone who still
remembers it with bitterness 20 years on and digs it out of prehistory
to use it as an example of how not to write.  Yep, that'd be sad.

自分が求めている答えを持っていると思い込んで、最初の文しか読まない人の苦味を想像してみてください！
20年前の苦い思い出を今でも覚えている人が、それを先史時代から掘り起こして、書き方の見本にしているのです。
そう、それは悲しいことだ。

There's a [detailed style guide](.copy-edit-stylesheet-checklist.md) if you
want to know how to format things, and we run a spellchecker in our [CI
system](.travis.yml) as well so you may need to add lines to
[.aspell.en.pws].

書式設定の方法を知りたい場合は、詳細なスタイルガイドがあります。
また、CIシステムでもスペルチェックを実行しますので、[.aspell.en.pws]に行を追加する必要があります。

### Writing The Requirements

Some requirements are obvious, some are subtle.  They're designed to walk
an implementer through the code they have to write, so write them as YOU
develop YOUR implementation.  Stick with `MUST`/`SHOULD`/`MAY` and `NOT`:
see [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt)

明らかな要件もあれば、微妙な要件もあります。
これらは、作成する必要があるコードを実装者に説明するように設計されているので、実装の開発時に作成してください。
MUST/SHOULD/MAYとNOTに従う：RFC2119参照

Requirements are grouped into writer and reader, just as implementations
are.  Make sure you define exactly what a writer must do, and exactly what
a reader must do if the writer doesn't do that!  A developer should
never have to intuit reader requirements from writer ones.

要件は、実装と同様に書き手と読み手にグループ化されます。
書き手がしなければならないことと、書き手がしなかった場合に読み手がしなければならないことを正確に定義してください。
開発者は、書き手の要求から読み手の要求を直感する必要はありません。

Note that the data doesn't have requirements: don't say `foo MUST be 0`,
say `The writer MUST set foo to 0` and `The reader MUST fail the connection
if foo is not 0`.

データには要件がないことに注意してください：
`foo MUST is 0` とは言わないでください。
`The writer MUST set foo to 0` そして `The reader MUST fail the connection
if foo is not 0` と言ってください。

Avoid the term `MUST check`: use `MUST fail the connection if` or `MUST
fail the channel if` or `MUST send an error message if`.

`MUST check` という用語を使用しないでください：
`MUST fail the connection if` または
`MUST fail the channel if` または
`MUST send an error message if` を使用してください。

There's a subtle art here for future extensions: you might say `a writer
MUST set foo to 0` and not mention it in the reader requirements, but it's
better to say `a reader MUST ignore foo`.  A future version of the spec
might define when a writer sets `foo` to `1` and we know that old readers
will ignore it.

ここには、将来の拡張のための巧妙な技があります：
あなたは `a writer MUST set foo to 0` と言うかもしれず、読み手の要件ではそれに言及しないかもしれない、
しかし `a reader MUST ignore foo` と言うのがよりよいでしょう。
この仕様の将来のバージョンでは、作成者が `foo` を `1` に設定しても、古い読み手はそれを無視することがわかっています。

`MAY` is a hint as to what something is for: an implementation may do
anything not written in the spec anyway.  `MUST` is when not doing
something will break the protocol or security.

`MAY` は、何かが何のためにあるかについてのヒントです：
実装は、いずれにしても、仕様に書かれていないことをするかもしれません。
`MUST` は、何かをしないことがプロトコルやセキュリティを破るときです。

Requirements can be vague (eg. "in a timely manner"), but only as a last
resort admission of defeat.  If you don't know, what hope has the poor
implementer?

要件はあいまいなこともあるが（例：「タイムリーな方法で」）、
敗北を認める最後の手段としてのみである。
知らないとしたら、貧しい実装者にはどんな希望があるのでしょうか。

### Creating Test Vectors

For new low-level protocol constructions, test vectors are necessary.
These have traditionally been lines within the spec itself, but the modern
trend is to use JSON and separate files.  The intent is that they be
machine-readable by implementations.

新しい低レベルプロトコル構築のために、テストベクタが必要です。
これらは従来、仕様自体の中の並べられましたが、最近のトレンドはJSONと別々のファイルを使うことです。
目的は、それらが実装によって機械可読であることです。

For new inter-peer messages, a test framework is in development to simulate
entire conversations.

新しいピア間メッセージについては、会話全体をシミュレートするテストフレームワークを開発中です。

## Specification Modification Process

There is a [mailing
list](https://lists.linuxfoundation.org/mailman/listinfo/lightning-dev)
for larger feature discussion, a [GitHub
repository](https://github.com/lightningnetwork/lightning-rfc) for
explicit issues and pull requests, and a bi-weekly IRC meeting on
#lightning-dev on Freenode, currently held at 5:30am Tuesday, 
Adelaide/Australia timezone (eg. Tuesday 23rd July 2019 05:30 == Mon, 22
Jul 2019 20:00 UTC).

もっと大きな機能の議論のためのメーリングリストや、明示的な問題やプルリクエストのためのGitHubリポジトリ、
Freenodeでの#lightning-devに関する隔週のIRCミーティングなどがあり、
現在火曜日の5:30AM、アデレード/オーストラリア時間
（例:。2019年7月23日火曜日05:30　==　2019年7月22日20:00UTC）に開催されている。

Spelling, typo and formatting changes are accepted once two contributors
ack and there are no nacks.  All other changes get approved and minuted at
the IRC meeting.  Protocol changes require two independent implementations
which successfully inter-operate; be patient as spec changes are hard to
fix later, so agreement can take some time.

スペル、タイプミス、書式の変更は、2人の投稿者が了解し、nacksがなければ受け付けられます。
他のすべての変更はIRC会議で承認され、議事録に記録されます。
プロトコル変更は、相互運用する2つの独立した実装を必要とする；
仕様の変更は後で修正するのが難しいため我慢してください、合意には時間がかかることがあります。

In addition, there are occasional face-to-face invitation-only Summits
where broad direction is established.  These are amazing, and you should
definitely join us sometime.

さらに、幅広い方向性が確立されている場合には、招待制の会合が時折開催されます。
これらは素晴らしいので、いつかぜひ参加してください。

We look forward to you joining us!
Your Friendly Lightning Developers.

皆様のご参加をお待ちしております！
フレンドリーなLightning開発者。
