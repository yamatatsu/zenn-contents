---
title: "Design Guidelines"
---

[元ドキュメント](https://github.com/aws/aws-cdk/blob/master/docs/DESIGN_GUIDELINES.md)

## 著者の感想

TBD

## 本文

### AWS Construct Library Design Guidelines

AWS Construct Library は、AWS リソースを実装するライブラリである。

このドキュメントの目的は、一貫性のある統合されたエクスペリエンスを保証するために、AWS Construct Library の API を設計するためのガイドラインを提供することです。

#### Preface

AWS Construct Library（および本ガイドライン）の API を設計する際、我々は AWS CDK の信条に従います。

- **Meet developers where they are**: 私たちの API は、ユーザーのメンタルモデルに基づいており、サービス API のメンタルモデルには基づいていません。
- **Full coverage**: いい感じのデフォルトを提供しますが、すべての動作を完全にカスタマイズすることができます。
- **Designed for the CDK**: CDK らしさを出しつつ、必要なら CFn の生の API を使えるようにします。
- **Open**: 誰でも CDK の IF を拡張して再公開できます。
- **Designed for jsii**: jsii してるので使えない TS 表現があります。

#### What's Included

AWS CDK の一部として公開されている AWS Construct Library は、AWS リソースを表現するコンストラクトです。

L1 って level 1 のことだったんだ。layer 1 かと思ってた。

L1 は CFn の定義から自動生成されてる。

L2 が L1 に比べてどう違うか。

- 強く型付けされてる（string じゃなくて Enum にしたり、さらに高度にモデリングされてたり）
- AWS リソース同士を関連付けるためのメソッド（grantXXX, addEvent とか）
- 権限系の操作のモデリング
- メトリクスのモデリング

L2.5 ってのもあるよ。`aws-apigateway.LambdaRestApi`, `aws-lambda-nodejs.NodeJsFunction`, `aws-rds.ServerlessCluster` `eks.FargateCluster`など。

L2.5 は以下の場合に作成が検討される。

- コミュニティの大部分によって使用される一般的な使用シナリオをカバーしている。
- 基本的な L2 よりも大幅に使いやすくなる（用途に応じたデフォルトの便利なメソッドや、改良された strong-typing によって）。
- CDK 内の別の L2 を簡素化または有効化する。

L3 は別のライブラリで出そう。このリポジトリにはもう入れないよ。今ある分も v2 で消すよ。

#### API Design

##### Construct Class

経験則的には、ほとんどのコンストラクタは、他のコンストラクタではなく、 **Construct** または **Resource** を直接拡張すべきです。多相的な振る舞いは、継承ではなく、インターフェースで表現することをお勧めします。

コンストラクタ・クラスは、以下のクラスのうちの一つだけを継承すべきです[awslint:construct-inheritence]。

- AWS リソースを表現するなら **Resource** class
- 抽象的なコンポーネントを表現するなら **Construct** class
- The **XxxBase** class (which, in turn extends **Resource**)

###### Owned vs. Unowned Constructs

CDK では、CDK 内で生成されたリソースの他に、別で作られたリソースを使うことがある。その場合`fromXXX`などを用いてリソースへの参照を手に入れます。これを可能にするために、基本的には`IFoo`などのインターフェースに依存して実装されることが望ましいです。

###### Abstract Base

`XxxBase`を実装することが推奨されています。

```ts
abstract class FooBase extends Resource implements IFoo {
  public abstract fooName: string;
  public abstract fooArn: string;

  // .. concrete implementation of IFoo (grants, metrics, factories),
  // should only rely on "fooName" and "fooArn" theoretically
}
```

##### Props

###### Types

L2 が公開する I/F で `XxxArn` を受け取らないで、`IXxxx`を受け取ること。Token 型を受け取らないこと。

Props は以下の方を受け取れます。

- Primitives (string, number, boolean, date)
- Collections (list, map)
- Structs
- Enums
- Enum-like classes
- Union-like classes
- IXxxx interface
- Integration interfaces (interfaces that have a “**bind**” method)

###### Defaults

`@default`は "-" で始まり、デフォルトの動作の説明を含まなければなりません[awslint:props-default-doc] 。

**デフォルトを制御するのが CDK でない場合でも、デフォルト値またはデフォルトの動作を記述します。** 例えば、存在しない値がテンプレートにレンダリングされず、デフォルトの振る舞いを決定するのが最終的に AWS のサービスである場合でも（つまり CFn が何かしらの初期値を設定してくれる場合にも）、ドキュメントにそれを記述するのです。

Examples:

```ts
// ✅ DO - uses a '-' and describes the behavior

/**
 * External KMS key to use for bucket encryption.
 *
 * @default - if encryption is set to "Kms" and this property is undefined, a
 * new KMS key will be created and associated with this bucket.
 */
encryptionKey?: kms.IEncryptionKey;
```

```ts
/**
 * External KMS key to use for bucket encryption.
 *
 * @default undefined
 *            ❌ DO NOT - that the value is 'undefined' by default is implied. However,
 *                        what will the *behavior* be if the value is left out?
 */
encryptionKey?: kms.IEncryptionKey;
```

```ts
/**
 * Minimum capacity of the AutoScaling resource
 *
 * @default - no minimum capacity
 *             ❌ DO NOT - there most certainly is. It's probably 0 or 1.
 *
 * // OR
 * @default - the minimum capacity is the default minimum capacity
 *             ❌ DO NOT - this is circular and useless to the reader.
 *                         Describe what will actually happen.
 */
minCapacity?: number;
```

###### Flat

Props はできるだけ Flatten する。nest しない。共通の概念グループについてはプレフィックスをつけて区別する。

For example, instead of:

```ts
new Bucket(this, "MyBucket", {
  bucketWebSiteConfiguration: {
    errorDocument: "404.html",
    indexDocument: "index.html",
  },
});
```

Prefer:

```ts
new Bucket(this, "MyBucket", {
  websiteErrorDocument: "404.html",
  websiteIndexDocument: "index.html",
});
```

###### Concise

プロパティ名はできるだけ短く簡潔に。冗長なコンテキストを削除する。`configuration`とか`Settings`とか、とかとか。

For example, prefer “readCapacity” versus “readCapacityUnits”.

###### Naming

突飛な名前をつけるよりは AWS の公式サービスドキュメントで使われている用語の方が好ましいです。

###### Property Documentation

言語とスタイルがサービスにマッチするように、可能であれば英語の AWS 公式ドキュメントからコピーすることが推奨されます。

###### Enums

複数の選択肢を表現するために Enum を使用する。

```ts
export enum MyEnum {
  OPTION1 = "op21",
  OPTION2 = "opt2",
}
```

AWS でよくあるパターンは、あらかじめ定義された共通のオプションのセットから選択できるようにしつつ、ユーザーが独自にカスタマイズした値を提供できるようにすることである。

このような場合、「Enum-like Class」のパターンを使用する。

```ts
export interface MyProps {
  readonly option: MyOption;
}

export class MyOption {
  public static COMMON_OPTION_1 = new MyOption("common.option-1");
  public static COMMON_OPTION_2 = new MyOption("common.option-2");

  public constructor(public readonly customValue: string) {}
}
```

Then usage would be:

```ts
new BoomBoom(this, "Boom", {
  option: MyOption.COMMON_OPTION_1,
});
```

```ts
export class MyOption {
  public static COMMON_OPTION_1 = new MyOption("common.option-1");
  public static COMMON_OPTION_2 = new MyOption("common.option-2");

  public static custom(value: string) {
    return new MyOption(value);
  }

  // 'protected' iso. 'private' so that someone that really wants to can still
  // do subclassing. But maybe might as well be private.
  protected constructor(public readonly value: string) {}
}

// Usage
new BoomBoom(this, "Boom", {
  option: MyOption.COMMON_OPTION_1,
});

new BoomBoom(this, "Boom", {
  option: MyOption.custom("my-value"),
});
```

###### Unions

ユニオン型は使わないで。代わりに Enum 使ったり Enum-like にしたりしましょう。 ↓ こんな感じ。

```ts
new lambda.Function(this, 'MyFunction', {
  code: lambda.Code.asset('/asset/path'), // or
  code: lambda.Code.bucket(myBucket, 'bundle.zip'), // or
  code: lambda.Code.inline('code')
  // etc
}
```

##### Attributes

Every AWS resource has a set of "physical" runtime attributes such as ARN, physical names, URLs, etc. These attributes are commonly late-bound, which means they can only be resolved during deployment, when AWS CloudFormation actually provisions the resource.

AWS constructs must expose all resource attributes defined in the underlying CloudFormation resource as readonly properties of the class _[awslint:resource-attribute]_.

All properties that represent resource attributes must include the JSDoc tag **@attribute** _[awslint:attribute-tag]_.

All attribute names must begin with the type name as a prefix (e.g. ***bucket*Arn** instead of just **arn**) _[awslint:attribute-name]_. This implies that if a property begins with the type name, it must have an **@attribute** tag.

All resource attributes must be represented as readonly properties of the resource interface _[awslint:attribute-readonly]_.

Resource attributes should use a type that corresponds to the resolved AWS CloudFormation type (e.g. **string**, **string[]**) _[awslint:attribute-type]_.

> Resource attributes almost always represent string values (URL, ARN, name). Sometimes they might also represent a list of strings. Since attribute values can either be late-bound ("a promise to a string") or concrete ("a string"), the AWS CDK has a mechanism called "tokens" which allows codifying late-bound values into strings or string arrays. This approach was chosen in order to dramatically simplify the type-system and ergonomics of CDK code. As long as users treat these attributes as opaque values (e.g. not try to parse them or manipulate them), they can be used interchangeably.

If needed, you can query whether an object includes unresolved tokens by using the **Token.unresolved(x)** method.

To ensure users are aware that the value returned by attribute properties should be treated as an opaque token, the JSDoc “@returns” annotation should begin with “**@returns a $token representing the xxxxx**” [_awslint:attribute-doc-returns-token_].

##### Configuration

When an app defines a construct or resource, it specifies its provisioning configuration upon initialization. For example, when an SQS queue is defined, its visibility timeout can be configured.

Naturally, when constructs are imported (unowned), the importing app does not have control over its configuration (e.g. you cannot change the visibility timeout of an SQS queue that you didn't create). Therefore, construct interfaces cannot include methods that require access/mutation of configuration.

One of the problems with configuration mutation is that there could be a race condition between two parts of the app, trying to set contradicting values.

There are a number use cases where you'd want to provide APIs which expose or mutate the construct's configuration. For example, **lambda.Function.addEnvironment** is a useful method that adds an environment variable to the function's runtime environment, and used occasionally to inject dependencies.

> Note that there are APIs that look like they mutate the construct, but in fact they are **factories** (i.e. they define resources on the user's stack). Those APIs _should_ be exposed on the construct interface and not on the construct class.

To help avoid the common mistake of exposing non-configuration APIs on the construct class (versus the construct interface), we require that configuration APIs (methods/properties) defined on the construct class will be annotated with the **@config** jsdoc tag [_awslint:config-explicit_].

```ts
interface IFoo extends IConstruct {
  bar(): void;
}

class Foo extends Construct implements IFoo {
  public bar() {}

  /** @mutating */
  public goo() {}

  public mutateMe() {} // ERROR! missing "@mutating" or missing on IFoo
}
```

###### Prefer Additions

As a rule of thumb, “adding” items to configuration props of type unordered array is normally considered safe as it will unlikely cause race conditions. If the prop is a map (like in **addEnvironment**), write defensive code that will throw if two values are assigned to the same key.

###### Dropped Mutations

Since all references across the library are done through a construct's interface, methods that are only available on the concrete construct class will not be accessible by code that uses the interface type. For example, code that accepts a **lambda.IFunction** will not see the **addEnvironment** method.

In most cases, this is desirable, as it ensures that only the code the owns the construct (instantiated it), will be able to mutate its configuration.

However, there are certain areas in the library, where, for the sake of consistency and interoperability, we allow mutating methods to be exposed on the interface. For example, **grant** methods are exposed on the construct interface and not on the concrete class. In most cases, when you grant a permission on an AWS resource, the _principal's_ policy needs to be updated, which mutates the consumer. However, there are certain cases where a _resource policy_ must be updated. However, if the resource is unowned, it doesn't make sense (or even impossible) to update its policy (there is usually a 1:1 relationship between a resource and a resource policy). In such cases, we decided that grant methods will simply skip any changes to resource policies, but will attach a **permission notice** to the app, which will be printed when the stack is synthesized by the toolkit.

##### Factories

In most AWS services, there are one or more resources which can be referred to as “primary resources” (normally one), while other resources exposed by the service can be considered “secondary resources”.

For example, the AWS Lambda service exposes the **Function** resource, which can be considered the primary resource while **Layer**, **Permission**, **Alias** are considered secondary. For API Gateway, the primary resource is **RestApi**, and there are many secondary resources such as **Method**, **Resource**, **Deployment**, **Authorizer**.

Secondary resources are normally associated with the primary resource (i.e. a reference to the primary resource must be supplied upon initialization).

Users should be able to define secondary resources either by directly instantiating their construct class (like any other construct), and passing in a reference to the primary resource's construct interface _or_ it is recommended to implement convenience methods on the primary resource that will facilitate defining secondary resources. This improves discoverability and ergonomics _[awslint:factory-method]_.

For example, **lambda.Function.addLayer** can be used to add a layer to the function, **apigw.RestApi.addResource** can be used to add to an API.

Methods for defining a secondary resource “Bar” associated with a primary resource “Foo” should have the following signature:

```ts
export interface IFoo {
  addBar(...): Bar;
}
```

Notice that:

- The method has an “add” prefix. It implies that users are adding something to their stack.
- The method is implemented on the construct interface (to allow adding secondary resources to unowned constructs).
- The method returns a “Bar” instance (owned).

In order to reuse the set of props used to configure the secondary resource, define a base interface for **FooProps** called **FooOptions** to allow secondary resource factory methods to reuse props _[awslint:factory-method-options]_:

```ts
export interface LogStreamOptions {
  logStreamName?: string;
}

export interface LogStreamProps extends LogStreamOptions {
  logGroup: ILogGroup;
}

export interface ILogGroup {
  addLogStream(id: string, options?: LogStreamOptions): LogStream;
}
```

##### Imports

Construct classes should expose a set of static factory methods with a “**from**” prefix that will allow users to import _unowned_ constructs into their app.

The signature of all “from” methods should adhere to the following rules _[awslint:from-signature]_:

- First argument must be **scope** of type **Construct**.
- Second argument is a **string**. This string will be used to determine the ID of the new construct. If the import method uses some value that is promised to be unique within the stack scope (such as ARN, export name), this value can be reused as the construct ID.
- Returns an object that implements the construct interface (**IFoo**).

###### “from” Methods

Resource constructs should export static “from” methods for importing unowned resources given one or more of its physical attributes such as ARN, name, etc. All constructs should have at least one `fromXxx` method _[awslint:from-method]_:

```ts
static fromFooArn(scope: Construct, id: string, bucketArn: string): IFoo;
static fromFooName(scope: Construct, id: string, bucketName: string): IFoo;
```

> Since AWS constructs usually export all resource attributes, the logic behind the various “from\<Attribute\>” methods would normally need to convert one attribute to another. For example, given a name, it would need to render the ARN of the resource. Therefore, if **from\<Attribute\>** methods expect to be able to parse their input, they must verify that the input (e.g. ARN, name) doesn't have unresolved tokens (using **Token.unresolved**). Preferably, they can use **Stack.parseArn** to achieve this purpose.

If a resource has an ARN attribute, it should implement at least a **fromFooArn** import method [_awslint:from-arn_].

To implement **fromAttribute** methods, use the abstract base class construct as follows:

<!-- markdownlint-disable MD013 -->

```ts
class Foo {
  static fromArn(scope: Construct, fooArn: string): IFoo {
    class _Foo extends FooBase {
      public get fooArn() {
        return fooArn;
      }
      public get fooName() {
        return this.node.stack.parseArn(fooArn).resourceName;
      }
    }

    return new _Foo(scope, fooArn);
  }
}
```

<!-- markdownlint-enable MD013 -->

###### From-attributes

If a resource has more than a single attribute (“ARN” and “name” are usually considered a single attribute since it's usually possible to convert one to the other), then the resource should provide a static **fromAttributes** method to allow users to explicitly supply values to all resource attributes when they import an external (unowned) resource [_awslint:from-attributes_].

```ts
static fromFooAttributes(scope: Construct, id: string, attrs: FooAttributes): IFoo;
```

##### Roles

If a CloudFormation resource has a **Role** property, it normally represents the IAM role that will be used by the resource to perform operations on behalf of the user.

Constructs that represent such resources should conform to the following guidelines.

An optional prop called **role** of type **iam.IRole**should be exposed to allow users to "bring their own role", and use either an owned or unowned role _[awslint:role-config-prop]_.

```ts
interface FooProps {
  /**
   * The role to associate with foo.
   * @default - a role will be automatically created
   */
  role?: iam.IRole;
}
```

The construct interface should expose a **role** property, and extends **iam.IGrantable** _[awslint:role-property]_:

```ts
interface IFoo extends iam.IGrantable {
  /**
   * The role associated with foo. If foo is imported, no role will be available.
   */
  readonly role?: iam.IRole;
}
```

This property will be `undefined` if this is an unowned construct (e.g. was not defined within the current app).

An **addToRolePolicy** method must be exposed on the construct interface to allow adding statements to the role's policy _[awslint:role-add-to-policy]_:

```ts
interface IFoo {
  addToRolePolicy(statement: iam.Statement): void;
}
```

If the construct is unowned, this method should no-op and issue a **permissions notice** (TODO) to the user indicating that they should ensure that the role of this resource should have the specified permission.

Implementing **IGrantable** brings an implementation burden of **grantPrincipal: IPrincipal**. This property must be set to the **role** if available, or to a new **iam.ImportedResourcePrincipal** if the resource is imported and the role is not available.

##### Resource Policies

Resource policies are IAM policies defined on the side of the resource (as oppose to policies attached to the IAM principal). Different resources expose different APIs for controlling their resource policy. For example, ECR repositories have a **RepositoryPolicyText** prop, SQS queues offer a **QueuePolicy** resource, etc.

Constructs that represents resources with a resource policy should encapsulate the details of how resource policies are created behind a uniform API as described in this section.

When a construct represents an AWS resource that supports a resource policy, it should expose an optional prop that will allow initializing resource with a specified policy _[awslint:resource-policy-prop]_:

```ts
resourcePolicy?: iam.PolicyStatement[]
```

Furthermore, the construct _interface_ should include a method that allows users to add statements to the resource policy _[awslint:resource-policy-add-to-policy]_:

```ts
interface IFoo extends iam.IResourceWithPolicy {
  addToResourcePolicy(statement: iam.PolicyStatement): void;
}
```

For some resources, such as ECR repositories, it is impossible (in CloudFormation) to add a resource policy if the resource is unowned (the policy is coupled with the resource creation). In such cases, the implementation of `addToResourcePolicy` should add a **permission** **notice** to the construct (using `node.addInfo`) indicating to the user that they must ensure that the resource policy of that specified resource should include the specified statement.

##### VPC

Compute resources such as AWS Lambda functions, Amazon ECS clusters, AWS CodeBuild projects normally allow users to specify the VPC configuration in which they will be placed. The underlying CFN resources would normally have a property or a set of properties that accept the set of subnets in which to place the resources.

In most cases, the preferred default behavior is to place the resource in all private subnets available within the VPC.

Props of such constructs should include the following properties _[awslint:vpc-props]_:

```ts
/**
 * The VPC in which to run your CodeBuild project.
 */
vpc: ec2.IVpc; // usually this is required

/**
 * Which VPC subnets to use for your CodeBuild project.
 *
 * @default - uses all private subnets in your VPC
 */
vpcSubnetSelection?: ec2.SubnetSelection;
```

##### Grants

Grants are one of the most powerful concepts in the AWS Construct Library. They offer a higher level, intent-based, API for managing IAM permissions for AWS resources.

**Despite the fact that they may be mutating**, grants should be exposed on the construct interface, and not on the concrete class _[awslint:grants-on-interface]_. See discussion above about mutability for reasoning.

Grants are represented as a set of methods with the “**grant**” prefix.

All constructs that represent AWS resources must have at least one grant method called “**grant**” which can be used to grant a grantee (such as an IAM principal) permission to perform a set of actions on the resource with the following signature. This method is defined as an abstract method on the **Resource** base class (and the **IResource** interface) _[awslint:grants-grant-method]_:

```ts
grant(grantee: iam.IGrantable, ...actions: string[]): iam.Grant;
```

The **iam.Grant** class has a rich API for implementing grants which implements the desired behavior.

Furthermore, resources should also include a set of grant methods for common use cases. For example, **dynamodb.Table.grantPutItem**, **s3.Bucket.grantReadWrite**, etc. In such cases, the signature of the grant method should adhere to the following rules _[awslint:grant-signature]_:

1. Name should have a “grant” prefix
2. Returns an **iam.Grant** object
3. First argument must be **grantee: iam.IGrantable**

```ts
grantXxx(grantee: iam.IGrantable): iam.Grant;
```

It makes sense for some AWS resources to also expose grant methods on all resources in the account. To support such use cases, expose a set of static grant methods on the construct class. For example, **dynamodb.Table.grantAllListStreams**. The signature of static grants should be similar _[awslint:grants-static-all]_.

```ts
export class Table {
  public static grantAll(
    grantee: iam.IGrantable,
    ...actions: string[]
  ): iam.Grant;
  public static grantAllListStreams(grantee: iam.IGrantable): iam.Grant;
}
```

##### Metrics

Almost all AWS resources emit CloudWatch metrics, which can be used with alarms and dashboards.

AWS construct interfaces should include a set of “metric” methods which represent the CloudWatch metrics emitted from this resource _[awslint:metrics-on-interface]_.

At a minimum (and enforced by IResource), all resources should have a single method called **metric**, which returns a **cloudwatch.Metric** object associated with this instance (usually this method will simply set the right metrics namespace and dimensions [_awslint:metrics-generic-method_]:

```ts
metric(metricName: string, options?: cloudwatch.MetricOptions): cloudwatch.Metric;
```

> **Exclusion**: If a resource does not emit CloudWatch metrics, this rule may

    be excluded

Additional metric methods should be exposed with the official metric name as a suffix and adhere to the following rules _[awslint:metrics-method-signature]:_

- Name should be “metricXxx” where “Xxx” is the official metric name
- Accepts a single “options” argument of type **MetricOptions**
- Returns a **Metric** object

```ts
interface IFunction {
  metricDuration(options?: cloudwatch.MetricOptions): cloudwatch.Metric;
  metricInvocations(options?: cloudwatch.MetricOptions): cloudwatch.Metric;
  metricThrottles(options?: cloudwatch.MetricOptions): cloudwatch.Metric;
}
```

It is sometimes desirable to use a metric that applies to all resources of a certain type within the account. To facilitate this, resources should expose a static method called **metricAll** _[awslint:metrics-static-all]_. Additional **metricAll** static methods can also be exposed _[awslint:metrics-all-methods]_.

<!-- markdownlint-disable MD013 -->

```ts
class Function extends Resource implements IFunction {
  public static metricAll(
    metricName: string,
    options?: cloudwatch.MetricOptions
  ): cloudwatch.Metric;
  public static metricAllErrors(
    props?: cloudwatch.MetricOptions
  ): cloudwatch.Metric;
}
```

<!-- markdownlint-enable MD013 -->

##### Events

Many AWS resources emit events to the CloudWatch event bus. Such resources should have a set of “onXxx” methods available on their construct interface _[awslint:events-in-interface]_.

All “on” methods should have the following signature [_awslint:events-method-signature_]:

```ts
onXxx(id: string, target: events.IEventRuleTarget, options?: XxxOptions): cloudwatch.EventRule;
```

When a resource emits CloudWatch events, it should at least have a single generic **onEvent** method to allow users to specify the event name [_awslint:events-generic_]:

```ts
onEvent(event: string, id: string, target: events.IEventRuleTarget): cloudwatch.EventRule
```

##### Connections

AWS resources that use EC2 security groups to manage network security should implement the **connections API** interface by having the construct interface extend **ec2.IConnectable** _[awslint:connectable-interface]_.

##### Integrations

Many AWS services offer “integrations” with other services. For example, AWS CodePipeline has actions that can trigger AWS Lambda functions, ECS tasks, CodeBuild projects and more. AWS Lambda can be triggered by a variety of event sources, AWS CloudWatch event rules can trigger many types of targets, SNS can publish to SQS and Lambda, etc, etc.

> See [aws-cdk#1743](https://github.com/awslabs/aws-cdk/issues/1743) for a discussion on the various design options.

AWS integrations normally have a single **central** service and a set of **consumed** services. For example, AWS CodePipeline is the central service and consumes multiple services that can be used as pipeline actions. AWS Lambda is the central service and can be triggered by multiple event sources.

Integrations are an abstract concept, not necessarily a specific mechanism. For example, each AWS Lambda event source is implemented in a different way (SNS, Bucket notifications, CloudWatch events, etc), but conceptually, _some_ users like to think about AWS Lambda as the “center”. It is also completely legitimate to have multiple ways to connect two services on AWS. To trigger an AWS Lambda function from an SNS topic, you could either use the integration or the SNS APIs directly.

Integrations should be modeled using an **interface** (i.e. **IEventSource**) exported in the API of the central module (e.g. “aws-lambda”) and implemented by classes in the integrations module (“aws-lambda-event-sources”) [_awslint:integrations-interface_].

```ts
// aws-lambda
interface IEventSource {
  bind(fn: IFunction): void;
}
```

A method “addXxx” should be defined on the construct interface and adhere to the
following rules _[awslint:integrations-add-method]:_

- Should accept any object that implements the integrations interface
- Should not return anything (void)
- Implementation should call “bind” on the integration object

```ts
interface IFunction extends IResource {
  public addEventSource(eventSource: IEventSource) {
    eventSource.bind(this);
  }
}
```

An optional array prop should allow declaratively applying integrations (sugar to calling “addXxx”):

```ts
interface FunctionProps {
  eventSources?: IEventSource[];
}
```

Lastly, to ease discoverability and maintain a sane dependency graphs, all integrations for a certain service should be mastered in a single secondary module named aws-_xxx_-_yyy_ (where _xxx_ is the service name and _yyy_ is the integration name). For example, **aws-s3-notifications**, **aws-lambda-event-sources**, **aws-codepipeline-actions**. All implementations of the integration interface should reside in a single module _[awslint:integrations-in-single-module]_.

```ts
// aws-lambda-event-sources
class DynamoEventSource implements IEventSource {
  constructor(table: dynamodb.ITable, options?: DynamoEventSourceOptions) { ... }

  public bind(fn: IFunction) {
    // ...do your magic
  }
}
```

When integration classes define new constructs in **bind**, they should be aware that they are adding into a scope they don't fully control. This means they should find a way to ensure that construct IDs do not conflict. This is a domain-specific problem.

##### State

Persistent resources are AWS resource which hold persistent state, such as databases, tables, buckets, etc.

To make sure stateful resources can be easily identified, all resource constructs must include the **@stateful** or **@stateless** JSDoc annotations at the class level _[awslint:state-annotation]_.

This annotation enables the following linting rules.

```ts
/**
 * @stateful
 */
export class Table {}
```

Persistent resources must have a **removalPolicy** prop, defaults to **Orphan** _[awslint:state-removal-policy-prop]_:

```ts
import { RemovalPolicy } from "@aws-cdk/cdk";

export interface TableProps {
  /**
   * @default ORPHAN
   */
  readonly removalPolicy?: RemovalPolicy;
}
```

Removal policy is applied at the CFN resource level using the **RemovalPolicy.apply(resource)**:

```ts
RemovalPolicy.apply(cfnTable, props.removalPolicy);
```

The **IResource** interface requires that all resource constructs implement a property **stateful** which returns **true** or **false** to allow runtime checks query whether a resource is persistent _[awslint:state-stateful-property]_.

##### Physical Names - TODO

See <https://github.com/awslabs/aws-cdk/issues/2283>

##### Tags

The AWS platform has a powerful tagging system that can be used to tag resources with key/values. The AWS CDK exposes this capability through the **Tag** “aspect”, which can seamlessly tag all resources within a subtree:

```ts
// add a tag to all taggable resource under "myConstruct"
myConstruct.node.apply(new cdk.Tag("myKey", "myValue"));
```

Constructs for AWS resources that can be tagged must have an optional **tags** hash in their props [_awslint:tags-prop_].

##### Secrets

If you expect a secret in your API (such as passwords, tokens), use the **cdk.SecretValue** class to signal to users that they should not include secrets in their CDK code or templates.

If a property is named “password” it must use the **SecretValue** type [_awslint:secret-password_]. If a property has the word “token” in it, it must use the SecretValue type [_awslint:secret-token_].

#### Project Structure

##### Code Organization

- Code should be under `lib/`
- Entry point should be `lib/index.ts` and should only contain “imports” for other files.
- No need to put every class in a separate file. Try to think of a reader-friendly organization of your source files.

#### Implementation

The following guidelines and recommendations apply are related to the implementation of AWS constructs.

##### General Principles

- Do not future proof.
- No fluent APIs.
- Good APIs “speak” in the language of the user. The terminology your API uses should be intuitive and represent the mental model your user brings over, not one that you made up and you force them to learn.
- Multiple ways of achieving the same thing is legitimate.
- Constantly maintain the invariants.
- The fewer “if statements” the better.

##### Construct IDs

Construct IDs (the second argument passed to all constructs when they are defined) are used to formulate resource logical IDs which must be **stable** across updates. The logical ID of a resource is calculated based on the **full path** of its construct in the construct scope hierarchy. This means that any change to a logical ID in this path will invalidate all the logical IDs within this scope. This will result in **replacements of all underlying resources** within the next update, which is extremely undesirable.

As described above, use the ID “**Resource**” for the primary resource of an AWS construct.

Therefore, when implementing constructs, you should treat the construct hierarchy and all construct IDs as part of the **external contract** of the construct. Any change to either should be considered and called out as a breaking change.

There is no need to concatenate logical IDs. If you find yourself needing to that to ensure uniqueness, it's an indication that you may be able to create another abstraction, or even just a **Construct** instance to serve as a namespace:

```ts
const privateSubnets = new Construct(this, "PrivateSubnets");
const publicSubnets = new Construct(this, "PublishSubnets");

for (const az of availabilityZones) {
  new Subnet(privateSubnets, az);
  new Subnet(publicSubnets, az, { public: true });
}
```

##### Errors

###### Avoid Errors If Possible

Always prefer to do the right thing for the user instead of raising an error. Only fail if the user has explicitly specified bad configuration. For example, VPC has **enableDnsHostnames** and **enableDnsSupport**. DNS hostnames _require_ DNS support, so only fail if the user enabled DNS hostnames but explicitly disabled DNS support. Otherwise, auto-enable DNS support for them.

###### Error reporting mechanism

There are three mechanism you can use to report errors:

- Eagerly throw an exception (fails synthesis)
- Attach a (lazy) validator to a construct (fails synthesis)
- Attach errors to a construct (succeeds synthesis, fails deployment)

Between these, the first two fail synthesis, while the latter doesn't. Failing synthesis means that no Cloud Assembly will be produced.

The distinction becomes apparent when you consider multiple stacks in the same Cloud Assembly:

- If synthesis fails due to an error in _one_ stack (either by throwing an exception or by failing validation), the other stack can also not be deployed.
- In contrast, if you attach an error to a construct in one stack, that stack cannot be deployed but the other one still can.

Choose one of the first two methods if the failure is caused by a misuse of the API, which the user should be alerted to and fix as quickly as possible. Choose attaching an error to a construct if the failure is due to environmental factors outside the direct use of the API surface (for example, lack of context provider lookup values).

###### Throwing exceptions

This should be the preferred error reporting method.

Validate input as early as it is passed into your code (ctor, methods, etc) and bail out by throwing an `Error`. No need to create subclasses of Error since all errors in the CDK are unrecoverable.

When validating inputs, don't forget to account for the fact that these values may be `Token`s and not available for inspection at synthesis time.

Example:

```ts
if (!Token.isUnresolved(props.minCapacity) && props.minCapacity < 1) {
  throw new Error(
    `'minCapacity' should be at least 1, got '${props.minCapacity}'`
  );
}
```

###### Never Catch Exceptions

All CDK errors are unrecoverable. If a method wishes to signal a recoverable error, this should be modeled in a return value and not through exceptions.

###### Attaching (lazy) Validators

In the rare case where the integrity of your construct can only be checked after the app has completed its initialization, call the **this.node.addValidation()** method to add a validation object. This will generally only be necessary if you want to produce an error when a certain interaction with your construct did _not_ happen (for example, a property that should have been configured over the lifetime of the construct, wasn't):

Always prefer early input validation over post-validation, as the necessity of these should be rare.

Example:

```ts
this.node.addValidation({
  // 'validate' should return a string[] list of errors
  validate: () => this.rules.length === 0
    ? ['At least one Rule must be added. Call \'addRule()\' to add Rules.']
    : []
  }
});
```

###### Attaching Errors/Warnings

You can also “attach” an error or a warning to a construct via the **Annotations** class. These methods (e.g., `Annotations.of(construct).addWarning`) will attach CDK metadata to your construct, which will be displayed to the user by the toolchain when the stack is deployed.

Errors will not allow deployment and warnings will only be displayed in highlight (unless `--strict` mode is used).

```ts
if (!Token.isUnresolved(subnetIds) && subnetIds.length < 2) {
  Annotations.of(this).addError(
    `Need at least 2 subnet ids, got: ${JSON.stringify(subnetIds)}`
  );
}
```

###### Error messages

Think about error messages from the point of view of the end user of the CDK.
This is not necessarily someone who knows about the internals of your construct library, so try to phrase the message in a way that would make sense to them.

For example, if a value the user supplied gets handed off between a number of functions before finally being validated, phrase the message in terms of the API the user interacted with, not in terms of the internal APIs.

A good error message should include the following components:

- What went wrong, in a way that makes sense to a top-level user
- An example of the incorrect value provided (if applicable)
- An example of the expected/allowed values (if applicable)
- The message should explain the (most likely) cause and change the user can make to rectify the situation

The message should be all lowercase and not end in a period, or contain information that can be obtained from the stack trace.

```ts
// ✅ DO - show the value you got and be specific about what the user should do
`supply at least one of minCapacity or maxCapacity, got ${JSON.stringify(
  action
)}` // ❌ DO NOT - this tells the user nothing about what's wrong or what they should do
`required values are missing` // ❌ DO NOT - this error only makes sense if you know the implementation
`'undefined' is not a number`;
```

##### Tokens

- Do not use FnSub

#### Documentation

Documentation style (copy from official AWS docs) No need to Capitalize Resource Names Like Buckets after they've been defined Stability (@stable, @experimental) Use the word “define” to describe resources/constructs in your stack (instead of“~~create~~”, “configure”, “provision”). Use a summary line and separate the doc body from the summary line using an empty line.

##### Inline Documentation

All public APIs must be documented when first introduced [_awslint:docs-public-apis_].

Do not add documentation on overrides/implementations. The public reference documentation will automatically copy the base documentation to the derived APIs, so it's better to avoid the confusion [_awslint:docs-no-duplicates_].

Use the following JSDoc tags: **@param**, **@returns**, **@default**, **@see**,
**@example.**

##### Readme

- Header should include maturity level.
- Example for the simple use case should be almost the first thing.
- If there are multiple common use cases, provide an example for each one and describe what happens under the hood at a high level (e.g. which resources are created).
- Reference docs are not needed.
- Use literate (`.lit.ts`) integration tests into README file.

#### Testing

##### Unit tests

- Unit test utility functions and object models separately from constructs. If you want them to be “package-private”, just put them in a separate file and import `../lib/my-util` from your unit test code.
- Failing tests should be prefixed with “fails”

##### Integration tests

- Integration tests should be under `test/integ.xxx.ts` and should basically just be CDK apps that can be deployed using “cdk deploy” (in the meantime).

##### Versioning

- Semantic versioning Construct ID changes or scope hierarchy

#### Naming & Style

##### Naming Conventions

- **Class names**: PascalCase
- **Properties**: camelCase
- **Methods (static and non-static)**: camelCase
- **Interfaces** (“behavioral interface”): IMyInterface
- **Structs** (“data interfaces”): MyDataStruct
- **Enums**: PascalCase, **Members**: SNAKE_UPPER

##### Coding Style

- **Indentation**: 2 spaces
- **Line length**: 150
- **String literals**: use single-quotes (`'`) or backticks (```)
- **Semicolons**: at the end of each code statement and declaration (incl. properties and imports).
- **Comments**: start with lower-case, end with a period.
