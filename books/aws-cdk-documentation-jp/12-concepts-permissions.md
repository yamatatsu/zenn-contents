---
title: "Concepts - Permissions"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/permissions.html)

## tl;dt

CDK を始める方は読んでおくことを強く推奨します。
CDK では L2 以上のコンストラクト（`Cfn`で始まってるクラス以外）を使っていれば、勝手に最小限の権限が与えられた Role が生成されて割り当てられるので、Terraform や CFn と違って Role をいくつも宣言する必要がありません。それでもリソースからリソースへ権限を渡すのに`grant`メソッドをよく使うので、読んでおくとコーディングが捗ります。

## 本文

AWS Construct Library は、アクセスやパーミッションを管理するために、いくつかの一般的で広く実装されているイディオムを使用します。IAM モジュールは、これらのイディオムを使用するために必要なツールを提供します。

### Principals

IAM プリンシパルは、ユーザー、サービス、アプリケーションなど、AWS リソースにアクセスするために認証されるエンティティです。AWS Construct Library は、以下のような多くの種類のプリンシパルをサポートしています。

1. IAM resources such as [`Role`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.Role.html), [`User`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.User.html), and [`Group`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.Group.html)

1. Service principals ([`new iam.ServicePrincipal('service.amazonaws.com')`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.ServicePrincipal.html))

1. Federated principals ([`new iam.FederatedPrincipal('cognito-identity.amazonaws.com')`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.FederatedPrincipal.html))

1. Account principals ([`new iam.AccountPrincipal('0123456789012')`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.AccountPrincipal.html))

1. Canonical user principals ([`new iam.CanonicalUserPrincipal('79a59d[...]7ef2be')`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.CanonicalUserPrincipal.html))

1. AWS organizations principals ([`new iam.OrganizationPrincipal('org-id')`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.OrganizationPrincipal.html))

1. Arbitrary ARN principals ([`new iam.ArnPrincipal(res.arn)`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.ArnPrincipal.html))

1. 複数プリンシパルを信頼するために使う [`iam.CompositePrincipal(principal1, principal2, ...)`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.CompositePrincipal.html)

### Grants

Amazon S3 バケットや Amazon DynamoDB テーブルなど、アクセス可能なリソースを表す construct には、他のエンティティにアクセスを許可するメソッドが用意されています。このようなメソッドはすべて `grant` で始まる名前を持っています。例えば、Amazon S3 バケットには [`grantRead`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3.Bucket.html#grantwbrreadidentity-objectskeypattern) と [`grantReadWrite`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3.Bucket.html#grantwbrreadwbrwriteidentity-objectskeypattern) というメソッドがあり、あるエンティティからバケットへの読み込みと読み込み/書き込みのアクセスを、これらの操作に必要な Amazon S3 IAM パーミッションを正確に知らなくとも可能にすることができます。

`grant` メソッドの第一引数は、常に [`IGrantable`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.IGrantable.html) 型です。このインターフェースは、パーミッションを付与できるエンティティ、つまり、IAM オブジェクトの [`Role`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.Role.html)、[`User`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.User.html)、[`Group`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.User.html) のようなロールを持つリソースを表します。

他のエンティティもパーミッションを付与することができます。例えば、このトピックの後半で、CodeBuild プロジェクトに Amazon S3 バケットへのアクセスを許可する方法を紹介します。一般に、関連するロールは、アクセスを許可されるエンティティの`role`プロパティによって取得されます。権限を付与することができる他のエンティティは、Amazon EC2 インスタンスと CodeBuild プロジェクトです。

[`lambda.Function`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda.Function.html) のような実行ロールを使用するリソースも `IGrantable` を実装しているので、ロールにアクセス権を付与するのではなく、直接アクセス権を付与することができます。例えば、`bucket` が Amazon S3 バケット、`function` が Lambda 関数である場合、以下のコードでは関数にバケットへの読み取り権限を付与しています。

```ts
bucket.grantRead(function);
```

スタックのデプロイ中にパーミッションを適用する必要がある場合があります。AWS CloudFormation のカスタムリソースに、他のリソースへのアクセスを許可する場合がその 1 つです。カスタムリソースはデプロイ中に呼び出されるため、デプロイ時に指定されたパーミッションを持っている必要があります。もう一つのケースは、渡されたロールに正しいポリシーが適用されているか、サービスが検証するときです（多くの AWS サービスは、あなたがポリシーを設定するのを忘れていないことを確認するために、これを行います）。このような場合、パーミッションの適用が遅れるとデプロイに失敗することがあります。

別のリソースが作成される前に grant のパーミッションを強制的に適用させるには、ここに示すように grant 自体に依存関係を追加します。grant メソッドの戻り値は一般的に破棄されますが、実際には、すべての grant メソッドは `iam.Grant` オブジェクトを返します。

```ts
const grant = bucket.grantRead(lambda);
const custom = new CustomResource(...);
custom.node.addDependency(grant);
```

### Roles

The IAM package contains a Role construct that represents IAM roles. The following code creates a new role, trusting the Amazon EC2 service.

IAM パッケージには、IAM ロールを表す [`Role`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.Role.html) コンストラクトが含まれています。次のコードは、Amazon EC2 サービスを信頼する、新しいロールを作成します。

```ts
import * as iam from "aws-cdk-lib/aws-iam";

const role = new iam.Role(this, "Role", {
  assumedBy: new iam.ServicePrincipal("ec2.amazonaws.com"), // required
});
```

Role の [`addToPolicy`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.Role.html#addwbrtowbrpolicystatement) メソッドを呼び出し、追加するルールを定義した [`PolicyStatement`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.PolicyStatement.html) を渡すことで、ロールにパーミッションを追加することができます。ステートメントはロールのデフォルトポリシーに追加されます; もしロールに何もなければ、1 つが作成されます。

次の例では、リソース Bucket と `otherRole` 上のアクション `ec2:SomeAction` と `s3:AnotherAction` に対して、認可サービスが AWS CodeBuild であるという条件で、Deny ポリシー文がロールに追加されています。

```ts
role.addToPolicy(
  new iam.PolicyStatement({
    effect: iam.Effect.DENY,
    resources: [bucket.bucketArn, otherRole.roleArn],
    actions: ["ec2:SomeAction", "s3:AnotherAction"],
    conditions: {
      StringEquals: {
        "ec2:AuthorizedService": "codebuild.amazonaws.com",
      },
    },
  })
);
```

上記の例では、[`addToPolicy`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.Role.html#addwbrtowbrpolicystatement) の呼び出しと同時に、新しい [`PolicyStatement`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.PolicyStatement.html) を作成しました。また、既存のポリシーステートメントや変更したステートメントを渡すこともできます。[`PolicyStatement`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.PolicyStatement.html) オブジェクトは、プリンシパル、リソース、条件、アクションを追加するための[多数のメソッド](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.PolicyStatement.html#methods)を持っています。

正しく機能するためにロールを必要とする構成を使用する場合、構成オブジェクトのインスタンスを作成する際に既存のロールを渡すか、構成に新しいロールを作成させ、適切なサービスプリンシパルを信頼させることができます。次の例では、このような CodeBuild プロジェクトを使用しています。

```ts
import * as codebuild from "aws-cdk-lib/aws-codebuild";

// imagine roleOrUndefined is a function that might return a Role object
// under some conditions, and undefined under other conditions
const someRole: iam.IRole | undefined = roleOrUndefined();

const project = new codebuild.Project(this, "Project", {
  // if someRole is undefined, the Project creates a new default role,
  // trusting the codebuild.amazonaws.com service principal
  role: someRole,
});
```

オブジェクトが作成されると、ロール (渡されたロールか、コンストラクトで作成されたデフォルトのロールか) をプロパティ `role` として使用できるようになります。しかし、このプロパティはインポートリソースでは利用できないため、そのようなコンストラクトには `addToRolePolicy` メソッドがあります。これは、コンストラクトがインポートリソースである場合は何もせず、それ以外はロールプロパティの `addToPolicy` メソッドを呼び出し、未定の場合を明確に処理する手間が省かれます。次の例はその例です。

```ts
// project is imported into the CDK application
const project = codebuild.Project.fromProjectName(
  this,
  "Project",
  "ProjectName"
);

// project is imported, so project.role is undefined, and this call has no effect
project.addToRolePolicy(
  new iam.PolicyStatement({
    effect: iam.Effect.ALLOW, // ... and so on defining the policy
  })
);
```

### Resource policies

Amazon S3 バケットや IAM ロールなど、AWS のいくつかのリソースは、リソースポリシーも持っています。これらのコンストラクトには `addToResourcePolicy` メソッドがあり、`PolicyStatement` を引数として受け取ります。リソースポリシーに追加されるすべてのポリシーステートメントは、少なくとも 1 つのプリンシパルを指定する必要があります。

以下の例では、Amazon S3 バケット `bucket` が `s3:SomeAction` 権限を持つロールを自分自身に付与しています。

```ts
bucket.addToResourcePolicy(
  new iam.PolicyStatement({
    effect: iam.Effect.ALLOW,
    actions: ["s3:SomeAction"],
    resources: [bucket.bucketArn],
    principals: [role],
  })
);
```
