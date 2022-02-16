---
title: "Concepts - Escape Hatches"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/cfn_layer.html)

## tl;dt

「このクラス、Props にあの機能生えてないじゃん！CloudFormation ならできるのに！」ってときに読む。

## 本文

高レベルのコンストラクトと低レベルの CFN Resource コンストラクトのいずれにも、あなたが探している特定の機能がない可能性があります。このような機能がない理由として、3 つの可能性があります。

- AWS サービス機能は AWS CloudFormation で利用できるが、そのサービス用の Construct クラスがない。
- AWS サービス機能は AWS CloudFormation を通じて利用可能で、サービス用の Construct クラスもありますが、Construct クラスはまだその機能を公開していない。
- この機能は、AWS CloudFormation を通じてまだ利用できない。

AWS CloudFormation を通じて機能が利用可能かどうかを判断するには、 [AWS Resource and Property Types Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html) を参照してください。

### Using AWS CloudFormation constructs directly

サービスに利用可能な Construct クラスがない場合、自動的に生成される CFN リソースに頼ることができます。これは、利用可能なすべての AWS CloudFormation リソースとプロパティに 1 対 1 でマッピングされます。これらのリソースは、 `CfnBucket` や `CfnRole` など、`Cfn` で始まるクラス名で表されます。これらのリソースは、同等の AWS CloudFormation リソースを使用するのとまったく同じようにインスタンス化することができます。詳細については、 [AWS Resource and Property Types Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)を参照してください。

例えば、分析を有効にした低レベルの Amazon S3 バケット CFN Resource をインスタンス化するには、次のように書きます。

```ts
new s3.CfnBucket(this, "MyBucket", {
  analyticsConfigurations: [
    {
      id: "Config",
      // ...
    },
  ],
});
```

AWS CloudFormation のリソース仕様にまだ公開されていない新しいリソースタイプなど、対応する `CfnXxx` クラスがないリソースを定義したい場合は、 `cdk.CfnResource` を直接インスタンス化して、リソースタイプやプロパティを指定することが可能です。これは、次の例で示されています。

```ts
new cdk.CfnResource(this, "MyBucket", {
  type: "AWS::S3::Bucket",
  properties: {
    // Note the PascalCase here! These are CloudFormation identifiers.
    AnalyticsConfigurations: [
      {
        Id: "Config",
        // ...
      },
    ],
  },
});
```

For more information, see [AWS Resource and Property Types Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html).

### Modifying the AWS CloudFormation resource behind AWS constructs

Construct に機能が不足している場合、または問題を回避しようとする場合、Construct によってカプセル化されている CFN Resource を修正することができます。

全ての Construct は、対応する CFN Resource を内包しています。例えば、高レベルの `Bucket` コンストラクトは、低レベルの `CfnBucket` コンストラクトを包んでいます。`CfnBucket` は AWS CloudFormation リソースに直接対応しているため、AWS CloudFormation を通じて利用できる全ての機能にアクセスすることができます。

CFN Resource クラスにアクセスする基本的な方法は、`construct.node.defaultChild` を使い、正しい型にキャストし（必要な場合）、そのプロパティを変更することです。ここでも、`Bucket` を例に挙げて説明します。

```ts
// Get the CloudFormation resource
const cfnBucket = bucket.node.defaultChild as s3.CfnBucket;

// Change its properties
cfnBucket.analyticsConfiguration = [
  {
    id: "Config",
    // ...
  },
];
```

また、このオブジェクトを使用して、`Metadata` や `UpdatePolicy` などの AWS CloudFormation のオプションを変更することができます。

```ts
cfnBucket.cfnOptions.metadata = {
  MetadataKey: "MetadataValue",
};
```

### Raw overrides

CFN リソースに不足しているプロパティがある場合、raw オーバーライドを使用してすべての型付けを回避することができます。これは合成されたプロパティを削除することも可能にします。

次の例に示すように、`addOverride` メソッドのいずれかを使用します。

```ts
// Get the CloudFormation resource
const cfnBucket = bucket.node.defaultChild as s3.CfnBucket;

// Use dot notation to address inside the resource template fragment
cfnBucket.addOverride("Properties.VersioningConfiguration.Status", "NewStatus");
cfnBucket.addDeletionOverride("Properties.VersioningConfiguration.Status");

// use index (0 here) to address an element of a list
cfnBucket.addOverride("Properties.Tags.0.Value", "NewValue");
cfnBucket.addDeletionOverride("Properties.Tags.0");

// addPropertyOverride is a convenience function for paths starting with "Properties."
cfnBucket.addPropertyOverride("VersioningConfiguration.Status", "NewStatus");
cfnBucket.addPropertyDeletionOverride("VersioningConfiguration.Status");
cfnBucket.addPropertyOverride("Tags.0.Value", "NewValue");
cfnBucket.addPropertyDeletionOverride("Tags.0");
```

### Custom resources

If the feature isn't available through AWS CloudFormation, but only through a direct API call, the only solution is to write an AWS CloudFormation Custom Resource to make the API call you need. Don't worry, the AWS CDK makes it easier to write these, and wrap them up into a regular construct interface, so from another user's perspective the feature feels native.

Building a custom resource involves writing a Lambda function that responds to a resource's CREATE, UPDATE and DELETE lifecycle events. If your custom resource needs to make only a single API call, consider using the AwsCustomResource. This makes it possible to perform arbitrary SDK calls during an AWS CloudFormation deployment. Otherwise, you should write your own Lambda function to perform the work you need to get done.

The subject is too broad to completely cover here, but the following links should get you started:

- Custom Resources
- Custom-Resource Example
- For a more fully fledged example, see the DnsValidatedCertificate class in the CDK standard library. This is implemented as a custom resource.

もしその機能が AWS CloudFormation を通してではなく、直接の API コールを通してのみ利用できる場合、唯一の解決策は、必要な API コールを行うために AWS CloudFormation Custom Resource を書くことです。AWS CDK はこれらを書くことを容易にし、それらを通常のコンストラクトインターフェイスにラップするので、他のユーザーの観点から、機能はネイティブに感じられるのです。

カスタムリソースを構築するには、リソースの CREATE、UPDATE、DELETE ライフサイクルイベントに応答する Lambda 関数を記述する必要があります。カスタムリソースが単一の API コールのみを行う必要がある場合、 [`AwsCustomResource`](https://github.com/awslabs/aws-cdk/tree/master/packages/%40aws-cdk/custom-resources) の使用を検討してください。これにより、AWS CloudFormation のデプロイ中に任意の SDK コールを実行することが可能になる。そうでない場合は、必要な作業を実行するために独自の Lambda 関数を記述する必要があります。

このテーマは広すぎてここで完全にカバーすることはできませんが、以下のリンクはあなたが始めるべきものです。

- [Custom Resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)
- [Custom-Resource Example](https://github.com/aws-samples/aws-cdk-examples/tree/master/typescript/custom-resource/)
- より本格的な例として、CDK 標準ライブラリの [`DnsValidatedCertificate`](https://github.com/awslabs/aws-cdk/blob/master/packages/@aws-cdk/aws-certificatemanager/lib/dns-validated-certificate.ts) クラスを参照してください。これはカスタムリソースとして実装されています。
