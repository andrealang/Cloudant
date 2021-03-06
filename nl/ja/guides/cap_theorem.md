---

copyright:
  years: 2015, 2017
lastupdated: "2017-01-24"

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}

<!-- Acrolinx: 2017-01-24 -->

<div id="cap_theorem"></div>

<div id="consistency"></div>

# CAP 定理

Cloudant は、[「結果整合性」![外部リンク・アイコン](../images/launch-glyph.svg "外部リンク・アイコン")](http://en.wikipedia.org/wiki/Eventual_consistency){:new_window} モデルを使用しています。
{:shortdesc}

このモデルがどのように機能し、なぜ Cloudant を使用するうえで不可欠な部分なのかを理解するために、整合性が何を意味しているかを考えてみます。

整合性は、データベース内のトランザクションを確実に処理および報告するために必要な 4 つの[「ACID」![外部リンク・アイコン](../images/launch-glyph.svg "外部リンク・アイコン")](https://en.wikipedia.org/wiki/ACID){:new_window}プロパティーの 1 つです。

さらに、整合性は、
<a href="http://en.wikipedia.org/wiki/CAP_Theorem" target="_blank">「CAP」<img src="../images/launch-glyph.svg" alt="外部リンク・アイコン" title="外部リンク・アイコン"></a>定理における 3 つの属性の 1 つでもあります。
これらの属性は、**C**onsistency (整合性)、**A**vailability (可用性)、および **P**artition tolerance (分断耐性) です。この定理は、Cloudant などの分散コンピューター・システムが次の 3 つの属性を_同時に_保証することは不可能だと述べています。
-   Consistency (整合性)。すべてのノードで同じデータが同時に表示される。
-   Availability (可用性)。すべての要求が、成功したか失敗したかに関する応答を受信することを保証する。
-   Partition tolerance (分断耐性)。システムのどこか一部が破損または故障してもシステムは稼働し続ける。

3 つの属性をすべて同時に保証することが不可能だということは、Cloudant が Consistency (整合性) 属性を保証していないことを意味します。Cloudant のような結果整合性 (eventually consistent) モデルでは、システムのある部分に対して行われた更新は、システムの他の部分によって_最終的に (eventually) _確認されます。更新が伝搬するにしたがって、システムは完全な整合性へと収束していくと言われています。

結果整合性は、パフォーマンスには有益です。強整合性モデルでは、システムは、書き込み要求または更新要求を完了する前に、すべての更新が完全および正常に伝搬するのを待機しなければなりません。結果整合性モデルでは、システム全体の伝搬が「裏で」で継続している間、書き込み要求または更新要求をほぼ即時に返すことができます。

理論的理由と実際的理由の両方により、データベースはこれらの 3 つの属性のうち 2 つだけを示すことができます。整合性と可用性を優先するデータベースは単純です。1 台のノードがデータの単一コピーを保管します。しかし、このモデルは拡大が困難になります。なぜなら、より良いパフォーマンスを得るには、追加のノードを使用するのではなく、ノードのアップグレードが必要になるからです。また、小さなシステム障害でも単一ノード・システムをシャットダウンする可能性があり、いかなるメッセージ損失も大きなデータ損失を意味します。これらの問題を許容するためには、システムがより洗練される必要があります。

## 分断耐性でのトレードオフ

整合性と分断耐性を優先するデータベースは、通常、<a href="http://en.wikipedia.org/wiki/Master/slave_(technology)" target="_blank">「マスター - スレーブ」<img src="../images/launch-glyph.svg" alt="外部リンク・アイコン" title="外部リンク・アイコン"></a>のセットアップを採用しています。このセットアップでは、システム内の多くのノードのうちの 1 台がリーダーとして選出されます。そのリーダーだけがデータ書き込みを承認でき、2 次ノードはすべて、リーダーからデータを複製して読み取りを処理します。リーダーがネットワークへの接続を失ったり、そのシステムのノードの多くと通信できなくなったりすると、残りのノードは新しいリーダーを選出します。この選出プロセスはシステム間で異なり、[重大な問題![外部リンク・アイコン](../images/launch-glyph.svg "外部リンク・アイコン")](http://aphyr.com/posts/284-call-me-maybe-mongodb){:new_window}の原因となる可能性があります。
Cloudant は、「マスター - マスター」セットアップを採用して、可用性と分断耐性を優先しています。そのため、すべてのノードが、そのノードのデータ割り当て部分に対する書き込みと読み取りの両方を受け入れることができます。複数のノードに、データの各割り当て部分のコピーが含まれています。各ノードは、他のノードのデータをコピーします。あるノードがアクセス不能になると、ネットワークが回復する間、他のノードが代わりにサービスを提供できます。このようにして、システムは、不定のノード障害が発生してもタイムリーにデータを返し、[結果整合性![外部リンク・アイコン](../images/launch-glyph.svg "外部リンク・アイコン")](http://en.wikipedia.org/wiki/Eventual_consistency){:new_window}を維持します。絶対的整合性の優先順位を下げた場合のトレードオフは、すべてのノードで同じデータが表示されるまでに時間がかかることです。その結果として、新しいデータがシステムで伝搬する間に、一部の応答に古いデータが含まれることがあります。

## アプローチの変更

1 つの整合したデータ・ビューを維持することは理にかなっており、容易に理解できます。なぜなら、リレーショナル・データベースはユーザーの代わりにこの作業を実行しているからです。データベース・システムと対話する、Web ベースのサービスはこのように振る舞うことが期待されています。しかし、その期待は、必ずこのように動作することを意味しているわけではありません。整合性は当たり前のものではなく、アプローチを変えるためには若干の作業が必要になります。

実際、整合性は、多くのエンタープライズ・クラウド・サービスにとって、必ずしも不可欠なものではありません。大規模で、使用頻繁が高いシステムには、システムの一部で障害が発生するかもしれない高い確率が伴います。可用性と結果整合性を優先させる必要性に合わせて設計されたデータベースは、アプリケーションの接続を維持することに、より適しています。アプリケーション・データの整合性の対処は事後に行うことができます。MIT の Seth Gilbert および Nancy Lynch は、「今日の現実世界の多くのシステムは、『ほとんどの場合、ほとんどのデータ』を返すということで解決せざるを得えない」という[結論![外部リンク・アイコン](../images/launch-glyph.svg "外部リンク・アイコン")](http://www.glassbeam.com/sites/all/themes/glassbeam/images/blog/10.1.1.67.6951.pdf){:new_window}を出しています。

## 企業におけるアプリケーションの可用性対整合性

人気のある Web ベースのサービスを見ると、人々が既に高可用性を期待しており、多くの場合そうしていることに気付かずに、喜んでこの可用性を結果整合データと交換していることが示されています。

多くのアプリケーションは、可用性のためにユーザーを誤った方向に導いています。ATM について考えてみましょう。不整合の銀行データが原因で、今も、知らないうちに口座の預金残高を超えてお金を引き出してしまうことが可能です。操作を続行する前にネットワーク内のすべてのノードを停止して口座の預金残高の金額を記録しなければならない場合、銀行システム全体でお客様の口座の預金残高の整合したビューを表示するというのは現実的ではありません。システムの可用性を高めた方が良いのです。

銀行業界は、1980 年代にこのことを理解しましたが、多くの IT 組織はまだ、可用性のために整合性を犠牲にすることに懸念を持っています。販売チームが CRM アプリにアクセスできない時にかかってくるサポート・コールの数について考えてみてください。次に、データベース更新をアプリケーション全体に伝搬するのに数秒間かかるということに彼らが気付きさえするかどうかについて考えてみてください。

可用性は、予想されるよりもずっと整合性に勝っています。その他のいくつかの例としては、オンライン・ショッピングのカート・システム、HTTP キャッシュ、および DNS があります。組織は、ユーザーの不満、生産性の損失、および逃した好機などのダウン時間のコストについて考える必要があります。

## 理論から実装へ

高可用性についての取り組みは、クラウド・アプリケーションにとって不可欠です。この取り組みを行わなければ、事業が拡大していくにつれて、全体的なデータベース整合性は大きなボトルネックとして残ります。高可用性アプリケーションは、たとえデータが最新でなくても、データとの定期的な接触を維持する必要があります。それこそが結果整合性の概念であり、怖れるものではありません。大きな規模では、何の答えも提供しないよりは、完全に正解でなくても答えを提供した方が良いということがあります。

データベース・システムは、さまざまな方法で可用性対整合性の複雑さを隠していますが、複雑さは常にそこにあります。CouchDB データベースおよび NoSQL データベースと共に Cloudant Database as a Service の見解は、設計プロセスの早い段階でこれらの複雑さに取り組むよう開発者に要求した方が良いというものです。最初に困難な作業を行うことにより、アプリケーションは最初から拡大する準備ができるので、サプライズの数が減ります。
