---

copyright:
  years: 2015, 2017
lastupdated: "2017-01-06"

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}

# Cloudant での関連文書のグループ化

従来の方法では、e-コマース・システムは、リレーショナル・データベースを使用して構築されます。これらのデータベースは、通常、結合された多くのテーブルを使用して、売上、顧客の詳細情報、購入された商品、および配達追跡の情報を記録します。リレーショナル・データベースは高い整合性を提供します。つまり、アプリケーション開発者は、コレクション間の結合、オブジェクトの状態を記録するための列挙、およびアトミック操作を保証するためのデータベース・トランザクションの使用を含め、データベースの強さに合わせてアプリケーションを構築することができます。

Cloudant では、整合性よりも可用性が優先されます。Cloudant は、結果整合性を実現する、高可用性を備えたフォールト・トレラントの分散データベースです。これには、お客様のショッピング・サービスで、複数のユーザーの同時購入を処理するのに十分な可用性とスケーラビリティーが常に提供されるという利点があります。つまり、アプリケーションは、Cloudant の強さを利用できるが、リレーショナル・データベースのようには扱えません。

このトピックでは、以下のように、他の多くのドメインに適用できる概念を使用して、Cloudant の強さを利用する e-コマース・システムの構築に関連したいくつかの要因の概要を提供します。

-   1 つの文書を頻繁に更新するのではなく、複数の文書を使用して購入の状態を示す。
-   関連するオブジェクトのコピーを、別のコレクションに結合するのではなく、順番に保管する。
-   `order_id` によって文書を照合するためのビューを作成して、購入の現在の状態を反映する。

例えば、注文されたアイテム、カスタマー情報、コスト、および配達情報などの詳細を含む `purchase` 文書を作成する場合があります。

_購入を記述する文書の例:_

```json
{
    "_id": "023f7a21dbe8a4177a2816e4ad1ea27e",
    "type": "purchase",
    "order_id": "320afa89017426b994162ab004ce3383",
    "basket": [
        {
            "product_id": "A56",
            "title": "Adele - 25",
            "category": "Audio CD",
            "price": 8.33,
            "tax": 0.2,
            "quantity": 2
        },
        {
            "product_id": "B32",
            "title": "The Lady In The Van - Alan Bennett",
            "category": "Paperback book",
            "price": 3.49,
            "tax": 0,
            "quantity": 2
        }
    ],
    "account_id": "985522332",
    "delivery": {
        "option": "Next Day",
        "price": 2.99,
        "address": {
            "street": "17 Front Street",
            "town": "Middlemarch",
            "postcode": "W1A 1AA"
        }
    },
    "pretax" : 20.15,
    "tax" : 3.32,
    "total": 26.46
}
```
{:codeblock}

この文書は、追加レコードを取り出さずに、Web ページまたは E メールでオーダーの要約をレンダリングするのに十分なデータを購入レコードに提供します。特に以下に示すような、オーダーについての主な詳細に注目してください。

-   買物かごは、他の場所に保管されている商品のデータベースの参照 ID (`product_id`) を含んでいる。
-   買物かごは、このレコード内の一部の商品データを、購入されたアイテムの販売時点の状態を記録するのに十分なだけ重複している。
-   文書は、オーダーの状況にマークを付けるフィールドを含んでいない。支払いと配達を記録するために、後で追加文書が追加されます。
-   データベースは、文書をデータベースに挿入する際に文書 `_id` を自動的に生成する。
-   後でオーダーを参照するために、固有 ID (`order_id`) が各購入レコードに提供されている。 
 
お客様が注文する時 (通常、Web サイトで「チェックアウト」段階に入った時点) に、前述の例に似た購入オーダー・レコードが作成されます。 

## 独自の固有 ID (UUID) の生成

リレーショナル・データベースでは、順次「自動増分」番号がよく使用されますが、サーバー・クラスターにデータが分散される分散データベースでは、文書がそれぞれ独自の固有 ID によって保管されることを確実にするために、長い UUID が使用されます。

アプリケーションで使用する固有 ID (例えば、`order_id`) を作成するには、Cloudant API で [`GET _uuids` エンドポイント](../api/advanced.html#-get-_uuids-)を呼び出します。データベースが、ユーザーのために ID を生成します。`count` パラメーター (例えば、`/_uuids?count=10`) を追加することにより、同じエンドポイントを使用して複数の ID を生成できます。

## 支払いの記録

お客様がアイテムの支払いを正常に行うと、オーダーを記録するための追加レコードがデータベースに追加されます。

_支払いレコードの例:_

```json
{
    "_id": "bf70c30ea5d8c3cd088fef98ad678e9e",
    "type": "payment",
    "account_id": "985522332",
    "order_id": "320afa89017426b994162ab004ce3383",
    "value": 6.46,
    "method": "credit card",
    "payment_reference": "AB9977G244FF2F667"
}
...
{
    "_id": "12c0ea6cd3d2c6e3b1d34442aea6a2d9",
    "type": "payment",
    "account_id": "985522332",
    "order_id": "320afa89017426b994162ab004ce3383",
    "value": 20.00,
    "method": "voucher",
    "payment_reference": "Q88775662377224"
}
```
{:codeblock}

前の例で、お客様は、クレジットカードの提供とプリペイド・バウチャーの交換によって支払いを行いました。2 つの支払いの合計はオーダーの金額分となりました。それぞれの支払いは、別々の文書として Cloudant に書き込まれました。

ある `order_id` について知っているすべてのことのビューを作成して、オーダーの状況を表示することができます。このビューは、以下の情報を含む台帳を有効にします。 

-   購入代金の合計 (正数として)。
-   アカウントに対する支払い (負数として)。

マップ関数を使用して必要な値を識別することができます。

_購入代金の合計と支払いの値を見つけるための マップ関数の例:_ 

```javascript
function(doc) {
    if (doc.type === 'purchase') {
        emit(doc.order_id, doc.total);
    } else {
        if (doc.type === 'payment') {
            emit(doc.order_id, -doc.value);
        }
    }
}
```
{:codeblock}

組み込まれている [`_sum` reducer](../api/creating_views.html#built-in-reduce-functions) を使用すると、支払いイベントの台帳として出力を生成することができます。

_組み込まれている `_sum` reducer を使用して `?reduce=false` で照会する例:_

```json
{
    "total_rows":3,"offset":0,"rows":[
        {
            "id":"320afa89017426b994162ab004ce3383",
            "key":"985522332",
            "value":26.46
        },
        {
            "id":"320afa89017426b994162ab004ce3383",
            "key":"985522332",
            "value":-20
        },
        {
            "id":"320afa89017426b994162ab004ce3383",
            "key":"985522332",
            "value":-6.46
        }
    ]
}
```
{:codeblock}

あるいは、`order_id` でグループ化された合計額を作成することもできます。

_`?group_level=1` を使用して、`order_id` でグループ化された合計額の例:_

```json
{
    "rows": [
          {
              "key":"320afa89017426b994162ab004ce3383",
            "value":0
        }
    ]
}
```
{:codeblock}

前の例のビューはオーダー値として 0 を返しているので、この結果は、オーダーが全額支払われていることを示しています。この理由は、正の購入オーダーの合計が、負の支払い金額を相殺しているためです。イベントを別々の文書 (オーダー用の文書とそれぞれの支払い用の文書) として記録することは、Cloudant では良い慣習です。なぜなら、それにより、複数のプロセスが同じ文書を同時に変更した時に競合が発生する可能性が回避されるからです。

## 追加文書の追加

他の別々の文書をデータベースに追加して、オーダーがプロビジョンおよび発送された時に以下の状態変更を記録することができます。

-   発送通知。
-   配達の受け取り。
-   払い戻しレコード。

データが到着すると Cloudant はそれぞれの文書に別々に書き込みます。したがって、コアとなる購入文書を変更する必要はありません。

## Cloudant に購入オーダーを保管することの利点

Cloudant を使用して購入オーダー情報を保管することにより、オーダー・システムの可用性およびスケーラビリティーを高め、大量のボリュームのデータの処理および高い同時アクセス率に対処することが可能になります。1 回のみ書き込まれる別々の文書内にデータをモデル化することにより、別々のプロセスによって同一文書への同時アクセスが行われた時などに、文書の競合が発生しないことを確実にできます。

さらに、文書には、外部キーを使用したデータの結合 (に依存するのではなく) を表す、他のコレクション内に存在するデータのコピーを含めることができます。例えば、購入時の買物かごの状態を記録する時などです。これにより、`order_id` によって関連付けられた文書をグループ化する Cloudant のビューへの 1 回の呼び出しでオーダーの状態を取り出すことができます。
