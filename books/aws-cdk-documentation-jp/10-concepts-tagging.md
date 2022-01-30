---
title: "Concepts - Tagging"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/tagging.html)

## tl;dt

CDK でのタグの付け方。オプションとかも含めて学びが多い。タグ付けが必要になったときには読み直したい。

## 本文

タグは、AWS CDK アプリの construct に追加できる、informational key-value 要素です。ある construct に適用されたタグは、そのタグ付け可能な子 construct 全てに適用されます。タグは、アプリから合成された AWS CloudFormation テンプレートに含まれ、それがデプロイされる AWS リソースに適用されます。タグを使用してリソースを識別および分類することで、管理の簡素化、コスト配分、アクセス制御、およびその他の目的に使用することができます。

:::message
**Tip**
AWS リソースでタグを使用する方法の詳細については、ホワイトペーパー[Tagging Best Practices](https://d1.awsstatic.com/whitepapers/aws-tagging-best-practices.pdf)を参照してください。
:::

[Tags class](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.Tags.html) には静的メソッド of() があり、これを用いて指定した構成要素にタグを追加したり、タグを削除したりすることができます。

- [`Tags.of(SCOPE).add()`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.Tags.html#addkey-value-props) は新しいタグを、指定された construct とそのすべての子 construct に適用します。

- [`Tags.of(SCOPE).remove()`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.Tags.html#removekey-props) は指定された construct とその子 construct からタグを削除します。

:::message
**Note**
タグ付けは[aspects](./15-concepts-aspects)を使用して実装されます。aspects は、指定されたスコープ内のすべての construct に操作（タグ付けなど）を適用するための方法です。
:::

次の例では、タグの key と value の組み合わせを construct に適用しています。

```ts
Tags.of(myConstruct).add("key", "value");
```

次の例では、construct から key を削除しています。

```ts
Tags.of(myConstruct).remove("key");
```

`Stage` 構成を使用している場合は、`Stage` レベル以下でタグを適用してください。`Stage` の境界を越えてタグが適用されることはありません。

### Tag priorities

AWS CDK は、再帰的にタグの適用と削除を行う。競合が発生した場合、最も高い優先度を持つタグ付け操作が優先されます。(2 つの操作の優先順位が同じ場合、構成ツリーの一番下に近いタグ付け操作が優先されます。デフォルトでは、タグの適用には 100 の優先度があり（AWS CloudFormation リソースに直接追加されたタグは例外で、優先度は 50）、タグの削除には 200 の優先度があります。

以下は、優先度 300 のタグをコンストラクトに適用したものです。

```ts
Tags.of(myConstruct).add("key", "value", {
  priority: 300,
});
```

### Optional properties

タグは、リソースへのタグの適用や削除の方法を微調整するためのプロパティをサポートしています。すべてのプロパティはオプションです。

`applyToLaunchedInstance`
`add()` のみで使用可能。デフォルトでは、タグは Auto Scaling グループで起動されたインスタンスに適用されます。このプロパティを false に設定すると、Auto Scaling グループで起動されたインスタンスは無視されます。

`includeResourceTypes/excludeResourceTypes`
AWS CloudFormation のリソースタイプに基づき、リソースのサブセットに対してのみタグを操作するために使用します。デフォルトでは、操作は構成サブツリー内のすべてのリソースに適用されますが、これは特定のリソースタイプを含むか除外することで変更できます。両方が指定されている場合は、include が優先される。

`priority`
他の `Tags.add()`および `Tags.remove()`操作に対するこの操作の優先度を設定するために使用します。より高い値がより低い値より優先されます。デフォルトは、add 操作で 100（AWS CloudFormation リソースに直接適用されるタグは 50）、remove 操作で 200 です。

次の例では、値 value、優先度 100 の tagname を、構成内のタイプ `AWS::Xxx::Yyy` のリソースに適用しますが、Amazon EC2 Auto Scaling グループで起動したインスタンス、タイプ `AWS::Xxx::Zzz` のリソースには適用されません。(これらは任意の 2 つの異なる AWS CloudFormation リソースタイプのプレースホルダーです)。

```ts
Tags.of(myConstruct).add("tagname", "value", {
  applyToLaunchedInstances: false,
  includeResourceTypes: ["AWS::Xxx::Yyy"],
  excludeResourceTypes: ["AWS::Xxx::Zzz"],
  priority: 100,
});
```

以下の例では、優先度 200 のタグ name を、構成内の `AWS::Xxx::Yyy` タイプのリソースから削除しますが、`AWS::Xxx::Zzz` タイプのリソースからは削除されません。

```ts
Tags.of(myConstruct).remove("tagname", {
  includeResourceTypes: ["AWS::Xxx::Yyy"],
  excludeResourceTypes: ["AWS::Xxx::Zzz"],
  priority: 200,
});
```

### Example

次の例では、MarketingSystem という名前のスタック内に作成されたすべてのリソースに、StackType というタグキーと TheBest という値を追加しています。そして、Amazon EC2 VPC サブネットを除くすべてのリソースから、それを再び削除します。その結果、サブネットにのみタグが適用されます。

```ts
import { App, Stack, Tags } from "aws-cdk-lib";

const app = new App();
const theBestStack = new Stack(app, "MarketingSystem");

// Add a tag to all constructs in the stack
Tags.of(theBestStack).add("StackType", "TheBest");

// Remove the tag from all resources except subnet resources
Tags.of(theBestStack).remove("StackType", {
  excludeResourceTypes: ["AWS::EC2::Subnet"],
});
```

次のコードでも同じ結果が得られます。どちらのアプローチ（包含と除外）があなたの意図を明確にするか考えてみてください。

```ts
Tags.of(theBestStack).add("StackType", "TheBest", {
  includeResourceTypes: ["AWS::EC2::Subnet"],
});
```
