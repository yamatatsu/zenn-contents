---
title: "Concepts - Identifiers"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/identifiers.html)

## 著者の感想

[Construct IDs](#construct-ids)は読んでおいて損はないかと思う。第 2 引数に渡される`id`の説明。重複が怒られるケースと怒られないケース（scope の概念）がわかる。

それ以降の項は全く知らなくても何ら問題ないと思う。

## 本文

AWS CDK は、多くの種類の Identifier と名前を扱います。AWS CDK を効果的に使用し、エラーを回避するためには、Identifier を理解する必要があります。

Identifier は、作成された Scope 内で一意でなければならない。AWS CDK アプリケーションでは、グローバルに一意である必要はない。

同じ Scope で同じ値を持つ Identifier を作成しようとすると、AWS CDK は例外をスローします。

### Construct IDs

最も一般的な Identifier は id で、construct オブジェクトのインスタンス化の際に第 2 引数として渡される識別子です。この識別子も他の識別子と同様、作成されたスコープ内で一意であればよく、これは construct オブジェクトのインスタンスを作成する際の最初の引数です。

:::message
**Note**
Stack の id は、AWS CDK Toolkit（cdk コマンド）で Stack を参照するための識別子にもなっています。
:::

例として、アプリ内に `MyBucket` という識別子を持つ 2 つの construct がある場合を見てみましょう。しかし、1 つ目は識別子 `Stack1` を持つ Stack のスコープ、2 つ目は識別子 `Stack2` を持つ Stack のスコープと、異なるスコープで定義されているため、何らかの衝突が起こるわけではなく、同じアプリ内で問題なく共存することができます。

```ts
import { App, Stack, StackProps } from "aws-cdk-lib";
import { Construct } from "constructs";
import * as s3 from "aws-cdk-lib/aws-s3";

class MyStack extends Stack {
  constructor(scope: Construct, id: string, props: StackProps = {}) {
    super(scope, id, props);

    new s3.Bucket(this, "MyBucket");
  }
}

const app = new App();
new MyStack(app, "Stack1");
new MyStack(app, "Stack2");
```

### Paths

AWS CDK アプリケーションの construct は、`App` クラスをルートとする階層を形成しています。ある construct、その親 construct、その祖父母など、construct ツリーのルートまでの ID の集合を Path と呼びます。

AWS CDK は通常、テンプレート内の Path を、ルート App インスタンスの直下の note（通常は Stack）から、各レベルの ID をスラッシュで区切った文字列として表示します。例えば、先ほどのコード例にある 2 つの Amazon S3 バケットリソースの Path は、`Stack1/MyBucket` と `Stack2/MyBucket` です。

次の例では、`myConstruct`の Path を取得していますが、プログラムによって任意の construct の Path にアクセスすることができます。ID は作成されたスコープ内で一意でなければならないため、その Path は AWS CDK アプリケーション内で常に一意となります。

```ts
const path: string = myConstruct.node.path;
```

### Unique IDs

AWS CloudFormation はテンプレート内の全ての論理 ID が一意であることを要求しているため、AWS CDK はアプリケーション内の各構成に対して一意の識別子を生成できる必要があります。リソースはグローバルに一意である Path（上の項で説明されたやつ）を持っているので、AWS CDK は Path の要素を連結し、8 桁のハッシュを追加することによって必要な一意の識別子を生成する。(AWS CloudFormation の識別子は英数字で、スラッシュや他の区切り文字を含むことができないため、A/B/C や A/BC といった異なる Path を区別して、同じ AWS CloudFormation 識別子になるようにハッシュは必要です)。AWS CDK では、この文字列を construct の unique ID と呼んでいます。

一般的に、AWS CDK アプリはユニーク ID について知る必要はないはずです。しかし、次の例に示すように、プログラム上で任意の construct のユニーク ID にアクセスすることができます。

```ts
const uid: string = Names.uniqueId(myConstruct);
```

address は、CDK リソースを一意に区別するためのもう一つのユニークな識別子である。Path の SHA-1 ハッシュから派生したもので、人間が読むことはできませんが、一定で比較的短い長さ（常に 16 進数で 42 文字）なので、「従来の」unique ID では長すぎるような状況で役に立ちます。construct によっては、unique ID の代わりに合成された AWS CloudFormation テンプレート内の address を使用することもあります。この場合も、一般的にはアプリがその construct の address を知る必要はありませんが、以下のように construct の address を取得することができます。

```ts
const addr: string = myConstruct.node.addr;
```

### Logical IDs

unique ID は、AWS リソースを表す construct のために生成された AWS CloudFormation テンプレートにおいて、リソースの Logical ID（論理名と呼ばれることもある）として機能します。

例えば、先ほどの Amazon S3 バケットを `Stack2` 内に作成した場合、生成される AWS CloudFormation テンプレートでは、論理 ID が `Stack2MyBucket4DD88B4F` の `AWS::S3::Bucket` リソースが生成されます。

construct ID は、construct の公開された契約の一部とお考えください。construct tree で construct の ID を変更すると、AWS CloudFormation はその construct の展開されたリソースインスタンスを置き換え、サービスの中断やデータ損失を引き起こす可能性があります。

### Logical ID stability

デプロイメント間でリソースの論理 ID を変更しないようにします。AWS CloudFormation は論理 ID によってリソースを識別するので、リソースの論理 ID を変更すると、AWS CloudFormation は既存のリソースを削除し、新しい論理 ID で新しいリソースを作成します。
