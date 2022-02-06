---
title: "Concepts - Aspects"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/aspects.html)

## tl;dt

これを読んで初めて知った。S3 bucket をうっかり暗号化設定無しで作成してしまって、あとから Security Hub とかで怒られてから設定しようとして、S3 bucket 作り直しになってしまう、とか防げるのかな。やってみたい。

## 本文

Aspect は、与えられたスコープ内のすべての構成要素に操作を適用するための方法です。Aspect は、タグの追加など construct を変更したり、すべてのバケットが暗号化されているかなど construct の状態を確認したりすることができます。

ある構成と同じスコープ内のすべての構成に Aspect を適用するには、次の例に示すように、新しい Aspect を指定して [`Aspects.of(SCOPE).add()`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.Aspects.html#static-ofscope) を呼び出します。

```ts
Aspects.of(myConstruct).add(new SomeAspect(...));
```

AWS CDK では現在、Aspect をリソースへのタグ付けにのみ使用しているが、フレームワークは拡張可能であり、他の目的にも使用できる。例えば、上位の construct で定義された AWS CloudFormation のリソースを検証したり変更したりするのに使うことができます。

### Aspects in detail

Aspect は、ビジターパターンを採用しています。Aspect は、以下のインターフェイスを実装したクラスです。

```ts
interface IAspect {
  visit(node: IConstruct): void;
}
```

`Aspects.of(SCOPE).add(...)` を呼び出すと、この construct は、Aspect を内部の Aspect のリストに追加します。このリストは `Aspects.of(SCOPE)`で取得することができます。

準備フェーズでは、AWS CDK は construct のオブジェクトとその子オブジェクトの `visit` メソッドを上から順に呼び出します。

`visit`メソッドは、construct 内の何かを自由に変更することができます。強い静的型付け言語では、受け取った construct を特定の型にキャストしてから、 construct 固有のプロパティやメソッドにアクセスします。

`Stage`は定義後、自己完結型で不変なので、Aspect は`Stage`の境界を越えて伝搬されません。`Stage`内の construct に Aspect を適用する場合は、`Stage`の construct 自身（またはそれ以下）に適用します。

### Example

次の例は、スタック内に作成されたすべてのバケットがバージョニングを有効にしていることを検証するものです。この Aspect は、検証に失敗した construct にエラーアノテーションを追加し、その結果、合成操作が失敗し、結果のクラウドアセンブリのデプロイができなくなります。

```ts
class BucketVersioningChecker implements IAspect {
  public visit(node: IConstruct): void {
    // See that we're dealing with a CfnBucket
    if (node instanceof s3.CfnBucket) {
      // Check for versioning property, exclude the case where the property
      // can be a token (IResolvable).
      if (
        !node.versioningConfiguration ||
        (!Tokenization.isResolvable(node.versioningConfiguration) &&
          node.versioningConfiguration.status !== "Enabled")
      ) {
        Annotations.of(node).addError("Bucket versioning is not enabled");
      }
    }
  }
}

// Later, apply to the stack
Aspects.of(stack).add(new BucketVersioningChecker());
```
