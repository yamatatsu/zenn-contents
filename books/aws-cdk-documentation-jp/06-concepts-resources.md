---
title: "Concepts - Resources"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/resources.html)

## tl;dt

AWS CDK を書いてて、ハマるケースの多くがこの章に書かれていると思う。
コードブロックだけでも良いので眺めていって、気になるところを文章呼んでみてもいいかも。
とにかく読んでほしい。

## 本文

[Constructs](./02-concepts-constructs)で説明したように、AWS CDK はすべての AWS リソースを表現する AWS constructs と呼ばれる豊富なクラスライブラリを提供します。このセクションでは、これらのコンストラストを使用する方法について、いくつかの一般的なパターンとベストプラクティスを説明します。

CDK app で AWS リソースを定義するのは、他の construct の定義と全く同じです。construct クラスのインスタンスを作成し、最初の引数としてスコープ、construct の論理 ID、そして設定プロパティ（props）のセットを渡します。例えば、AWS Construct Library の [`sqs.Queue`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_sqs.Queue.html) コンストラストを使用して、KMS 暗号化された Amazon SQS キューを作成する方法を示します。

```ts
import * as sqs from "aws-cdk-lib/aws-sqs";

new sqs.Queue(this, "MyQueue", {
  encryption: sqs.QueueEncryption.KMS_MANAGED,
});
```

いくつかの設定 props はオプションであり、多くの場合、デフォルト値があります。場合によっては、すべての props がオプションとなり、最後の引数は完全に省略することができます。

### Resource attributes

AWS Construct Library のほとんどのリソースは attributes を公開しており、AWS CloudFormation によってデプロイ時に解決されます。attributes は、リソースクラスのプロパティという形で、型名をプレフィックスとして公開されます。次の例では、`queueUrl` プロパティを使用して Amazon SQS キューの URL を取得する方法を示しています。

この attributes は CloudFormation でいうところの[Return Values](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-sqs-queues.html#aws-properties-sqs-queues-return-values)を返すため、これらを使用することでリソースとリソースに依存関係をもたせることができます。逆に言うと、自前で ARN などを文字結合で生成すると依存関係を CloudFormation に伝達できないため、デプロイ時のエラーとなる場合があります。

```ts
import * as sqs from "aws-cdk-lib/aws-sqs";

const queue = new sqs.Queue(this, "MyQueue");
const url = queue.queueUrl; // => A string representing a deploy-time value
```

AWS CDK が deploy-time 属性を文字列としてエンコードする方法については、[Tokens](./08-concepts-tokens) を参照してください。

### Referencing resources

多くの AWS CDK クラスは、AWS CDK リソースオブジェクト（リソース）であるプロパティを必要とします。これらの要件を満たすために、以下の 2 つの方法でリソースを参照することができます。

- リソースをそのまま渡す
- リソースの一意情報を渡す（ARN、ID、name など）

例えば、Amazon ECS サービスでは、それが動作するクラスタへの参照が必要であり、Amazon CloudFront distribution では、ソースコードが含まれる S3 buckt への参照が必要です。

Construct のプロパティが他の AWS Construct を表す場合、その型はその Construct のインターフェイスの型となります。例えば、Amazon ECS サービスは `ecs.ICluster` 型のプロパティ cluster を取り、CloudFront distribution は `s3.IBucket` 型のプロパティ `sourceBucket` を取ります。

すべてのリソースは対応するインターフェースを実装しているので、同じ AWS CDK アプリで定義しているリソースオブジェクトを直接渡すことができます。次の例では、Amazon ECS クラスタを定義し、それを使って Amazon ECS サービスを定義しています。

```ts
const cluster = new ecs.Cluster(this, 'Cluster', { /_..._/ });

const service = new ecs.Ec2Service(this, 'Service', { cluster: cluster });
```

### Accessing resources in a different stack

同じアカウントと AWS リージョンであれば、異なる Stack のリソースにアクセスすることができます。次の例では、stack1 を定義し、Amazon S3 バケットを定義しています。次に、2 つ目の Stack である stack2 を定義し、stack1 からのバケットをコンストラクタのプロパティとして受け取ります。

```ts
const prod = { account: "123456789012", region: "us-east-1" };

const stack1 = new StackThatProvidesABucket(app, "Stack1", { env: prod });

// stack2 will take a property { bucket: IBucket }
const stack2 = new StackThatExpectsABucket(app, "Stack2", {
  bucket: stack1.bucket,
  env: prod,
});
```

リソースが同じアカウントとリージョンにあるが、異なる Stack にあると AWS CDK が判断した場合、リソースを提供する側の Stack （`StackThatProvidesABucket`）（以下 producing stack）では AWS CloudFormation [exports](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html) を、リソースを使用する側の Stack （`StackThatExpectsABucket`）（以下 consuming stack）では [`Fn::ImportValue`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-importvalue.html) を自動的に合成してその情報をある Stack から別の Stack に転送します。

ある Stack のリソースを別の Stack で参照すると、2 つの Stack 間に依存関係が生じます。この依存関係が確立されると、consuming stack から共有リソースの使用を削除すると、AWS CDK Toolkit が producing stack を consuming stack より先にデプロイした場合、予期しないデプロイの失敗を引き起こす可能性があります。これは、2 つの Stack 間に別の依存関係がある場合に起こりますが、producing stack が AWS CDK Toolkit によって最初にデプロイされるように選択されることも起こり得ます。AWS CloudFormation のエクスポートは不要になったのでプロデュース Stack から削除されますが、エクスポートされたリソースはその更新がまだデプロイされていないので consuming stack でまだ使用されており、producing stack のデプロイは失敗しています。

このデッドロックを解消するには、consuming stack から共有リソースの使用を削除し（これにより、producing stack から自動エクスポートが削除されます）、自動生成されたエクスポートとまったく同じ論理 ID を使用して、同じエクスポートを producing stack に手動で追加してください。consuming Stack の共有リソースの使用を削除し、両方の Stack をデプロイします。次に、手動エクスポート（および不要になった場合は共有リソース）を削除し、両方の Stack を再びデプロイします。Stack の [`exportValue()`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.Stack.html#exportwbrvalueexportedvalue-options) メソッドは、この目的のために手動エクスポートを作成する便利な方法です (リンク先のメソッドリファレンスの例を参照してください)。

### Physical names

AWS CloudFormation のリソースの論理名は、AWS CloudFormation がリソースをデプロイした後に AWS Management Console に表示されるリソースの名前とは異なります。AWS CDK では、これらの最終的な名前を Physical names と呼んでいます。

例えば、AWS CloudFormation は、前の例の論理 ID `Stack2MyBucket4DD88B4F` の Amazon S3 バケットを Physical names `stack2mybucket4dd88b4f-iuv1rbv9z3to` として作成するかもしれません。

リソースを表す Construct を作成する際に、`${resourceType}Name` のように命名されたプロパティを使用すると、Physical names を指定することができます。次の例では、Physical names `my-bucket-name` で Amazon S3 バケットを作成しています。

```ts
const bucket = new s3.Bucket(this, "MyBucket", {
  bucketName: "my-bucket-name",
});
```

リソースに Physical names を割り当てることは、AWS CloudFormation ではいくつかのデメリットがあります。最も重要なのは、作成後に不変であるリソースのプロパティへの変更など、リソースの置き換えを必要とするデプロイ済みリソースへの変更は、リソースに Physical names が割り当てられていると失敗することです。もしそのような状態になってしまったら、唯一の解決策は AWS CloudFormation Stack を削除し、再度 AWS CDK アプリをデプロイすることです。詳しくは [AWS CloudFormation のドキュメント](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-name.html)を参照してください。

クロス環境参照で AWS CDK アプリを作成する場合など、AWS CDK が正しく機能するために Physical names が必要な場合があります。そのような場合、Physical names を自分で考えるのが面倒であれば、以下のように特別な値 `PhysicalName.GENERATE_IF_NEEDED` を使って AWS CDK に名前を付けてもらうことができます。

```ts
const bucket = new s3.Bucket(this, "MyBucket", {
  bucketName: cdk.PhysicalName.GENERATE_IF_NEEDED,
});
```

### Passing unique identifiers

可能な限り、前のセクションで説明したように、リソースを参照渡しする必要があります。しかし、リソースの属性のひとつを参照する以外に選択肢がないケースもあります。例えば、低レベルの AWS CloudFormation リソースを使用しているときや、環境変数を通じて Lambda 関数を参照するときなど、AWS CDK アプリケーションのランタイムコンポーネントにリソースを公開する必要がある場合です。

これらの識別子は、以下のようなリソースの属性として利用できます。

```ts
bucket.bucketName;
lambdaFunc.functionArn;
securityGroup.groupArn;
```

次の例は、生成されたバケット名を AWS Lambda 関数に渡す方法です。

```ts
const bucket = new s3.Bucket(this, "Bucket");

new lambda.Function(this, "MyLambda", {
  // ...
  environment: {
    BUCKET_NAME: bucket.bucketName,
  },
});
```

### Importing existing external resources

例えば、コンソールや AWS SDK、AWS CloudFormation で直接定義したリソースや、別の AWS CDK アプリケーションで定義したリソースなど、既に AWS アカウントにあるリソースを AWS CDK アプリで使いたい場合があります。リソースの ARN（または他の識別属性、または属性グループ）を、リソースのクラスで静的ファクトリーメソッドを呼び出すことで、現在の Stack で AWS CDK オブジェクトに変換することができます。

以下の例では、ARN `arn:aws:s3:::my-bucket-name` を持つ既存のバケットを基にバケットを定義し、特定の ID を持つ既存の VPC を基に Amazon Virtual Private Cloud を定義する方法を示しています。

```ts
// Construct a resource (bucket) just by its name (must be same account)
s3.Bucket.fromBucketName(this, "MyBucket", "my-bucket-name");

// Construct a resource (bucket) by its full ARN (can be cross account)
s3.Bucket.fromBucketArn(this, "MyBucket", "arn:aws:s3:::my-bucket-name");

// Construct a resource by giving attribute(s) (complex resources)
ec2.Vpc.fromVpcAttributes(this, "MyVpc", {
  vpcId: "vpc-1234567890abcde",
});
```

`ec2.Vpc` の構成は複雑で、VPC 本体、サブネット、セキュリティグループ、ルーティングテーブルなど多くの AWS リソースで構成されているため、属性を用いてこれらのリソースをインポートすることが困難な場合があります。これに対処するため、VPC コンストラクトには `fromLookup` メソッドがあり、[context method](./context#context-methods)を使用して合成時に必要なすべての属性を解決し、将来使用するためにその値を `cdk.context.json` にキャッシュしています。

AWS アカウントで VPC を一意に識別するのに十分な属性を提供する必要があります。たとえば、デフォルトの VPC は 1 つしかないので、デフォルトとしてマークされた VPC をインポートすることを指定すれば十分です。

```ts
ec2.Vpc.fromLookup(this, "DefaultVpc", {
  isDefault: true,
});
```

`tags` プロパティを使用すると、タグでクエリーを実行できます。タグは AWS CloudFormation または AWS CDK を使用して作成時に VPC に追加でき、作成後は AWS Management Console、AWS CLI、または AWS SDK を使用していつでも編集することが可能です。自分で追加したタグに加えて、AWS CDK は自動的に以下のタグを作成したすべての VPC に追加します。

- Name – The name of the VPC.
- aws-cdk:subnet-name – The name of the subnet.
- aws-cdk:subnet-type – The type of the subnet: Public, Private, or Isolated.

```ts
ec2.Vpc.fromLookup(this, "PublicVpc", {
  tags: { "aws-cdk:subnet-type": "Public" },
});
```

`Vpc.fromLookup()` は、env プロパティにアカウントと地域を明示的に指定して定義された Stack でのみ機能します。AWS CDK が [environment-agnostic stack](./stacks#stack-api) から Amazon VPC を検索しようとした場合、CLI は VPC を検索するためにどの環境を照会すればよいのかわかりません。

`Vpc.fromLookup()` の結果は、プロジェクトの `cdk.context.json` ファイルにキャッシュされます。[CDK Pipelines](https://docs.aws.amazon.com/cdk/v2/guide/cdk_pipeline.html) のように、VPC を定義する AWS アカウントにアクセスできない環境で Stack をデプロイする場合は、このファイルをバージョンコントロールにコミットしてください。

インポートしたリソースはどこでも使用できますが、インポートしたリソースを変更することはできません。例えば、インポートした `s3.Bucket` に対して `addToResourcePolicy` を呼び出しても、何も起こりません。

### Permission grants

AWS のコンストラストは、許可要件を表現するためのシンプルな intent-based API を提供することで、最小権限原則の準拠を簡単に実現することができます。多くの AWS コンストラクトは、IAM ロールやユーザーのようなエンティティに、リソースで作業する許可を簡単に付与できる grant メソッドを提供しており、1 つ以上の IAM 許可文を手動で作成する必要はありません。

次の例では、Lambda 関数の実行ロールが特定の Amazon S3 バケットにオブジェクトを読み書きできるようにするためのパーミッションを作成しています。Amazon S3 バケットが AWS KMS キーを使用して暗号化されている場合、このメソッドはまた、このキーを使用して復号化するために Lambda 関数の実行ロールパーミッションを付与します。

```ts
if (bucket.grantReadWrite(func).success) {
  // ...
}
```

grant メソッドは、`iam.Grant` オブジェクトを返します。`Grant` オブジェクトの `success` 属性を使用して、 grant が効果的に適用されたかどうかを判断します (たとえば、インポートされたリソースに適用されていない可能性があります)。また、`Grant` オブジェクトの `assertSuccess`メソッドを使用すると、grant の適用が成功したことを強制することができます。

特定のユースケースで特定の grant メソッドが使用できない場合は、 汎用的な grant メソッドを使用して、指定したアクションのリストを持つ新しい grant を定義することができます。

次の例は、Amazon DynamoDB CreateBackup アクションへのアクセスを Lambda 関数に許可する方法を示しています。

```ts
table.grant(func, "dynamodb:CreateBackup");
```

Lambda 関数などの多くのリソースでは、コードの実行時にロールを引き受ける必要があります。設定プロパティで、`iam.IRole` を指定することができます。ロールが指定されていない場合、この関数はこの用途に特化したロールを自動的に作成します。その後、リソースの grant メソッドを使用して、ロールにステートメントを追加することができます。

grant メソッドは、IAM ポリシーで処理するための低レベルの API を使用して構築されています。ポリシーは [PolicyDocument](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_iam.PolicyDocument.html) オブジェクトとしてモデル化されています。`addToRolePolicy` メソッドを使用してロール (または construct に付属するロール) に直接文を追加するか、 `addToResourcePolicy` メソッドを使用してリソースのポリシー (`Bucket policy` など) に Statement を追加できます。

### Metrics and alarms

多くのリソースは、監視ダッシュボードやアラームを設定するために使用できる CloudWatch メトリクスを発行します。AWS のコンストラクトには metric メソッドがあり、使用する正しい名前を調べることなく、簡単にメトリックにアクセスできるようになっています。

次の例では、Amazon SQS キューの `ApproximateNumberOfMessagesNotVisible` が 100 を超えたときにアラームを定義する方法を示しています。

```ts
import _ as cw from 'aws-cdk-lib/aws-cloudwatch';
import _ as sqs from 'aws-cdk-lib/aws-sqs';
import { Duration } from 'aws-cdk-lib';

const queue = new sqs.Queue(this, 'MyQueue');

const metric = queue.metricApproximateNumberOfMessagesNotVisible({
label: 'Messages Visible (Approx)',
period: Duration.minutes(5),
// ...
});
metric.createAlarm(this, 'TooManyMessagesAlarm', {
comparisonOperator: cw.ComparisonOperator.GREATER_THAN_THRESHOLD,
threshold: 100,
// ...
});
```

特定のメトリックに対応するメソッドがない場合は、一般的なメトリックメソッドを使用してメトリック名を手動で指定することができます。

メトリクスは CloudWatch のダッシュボードに追加することもできます。[CloudWatch](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_cloudwatch-readme.html) を参照してください。

### Network traffic

多くの場合、アプリケーションを動作させるためには、ネットワーク上のパーミッションを有効にする必要があります。例えば、compute インフラストラクチャが永続化層にアクセスする必要がある場合などです。接続を確立したりリッスンしたりするリソースは、セキュリティ・グループ・ルールやネットワーク ACL の設定など、トラフィックフローを有効にするメソッドを公開します。

[IConnectable](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ec2.IConnectable.html) リソースには、ネットワークトラフィックのルール設定へのゲートウェイとなる `connection` プロパティがあります。

`allow` メソッドを使用すると、指定したネットワークパスでデータを流せるようになります。次の例では、Web への HTTPS 接続と、Amazon EC2 Auto Scaling グループ `fleet2` からの着信接続を有効にしています。

```ts
import _ as asg from 'aws-cdk-lib/aws-autoscaling';
import _ as ec2 from 'aws-cdk-lib/aws-ec2';

const fleet1: asg.AutoScalingGroup = asg.AutoScalingGroup(/_..._/);

// Allow surfing the (secure) web
fleet1.connections.allowTo(new ec2.Peer.anyIpv4(), new ec2.Port({ fromPort: 443, toPort: 443 }));

const fleet2: asg.AutoScalingGroup = asg.AutoScalingGroup(/_..._/);
fleet1.connections.allowFrom(fleet2, ec2.Port.AllTraffic());
```

例えば、ロードバランサーのリスナーはパブリックポート、データベースエンジンは Amazon RDS データベースのインスタンスの接続を受け付けるポートなど、特定のリソースにはデフォルトのポートが関連付けられています。このような場合、`allowDefaultPortFrom` および `allowToDefaultPort` メソッドを使用すると、ポートを手動で指定しなくても、厳密なネットワーク制御を行うことができます。

次の例は、任意の IPV4 アドレスからの接続と、Auto Scaling グループからの接続を有効にして、データベースにアクセスする方法を示しています。

```ts
listener.connections.allowDefaultPortFromAnyIpv4("Allow public access");

fleet.connections.allowToDefaultPort(rdsDatabase, "Fleet can access database");
```

### Event handling

リソースによっては、イベントソースとして機能するものもあります。リソースが発する特定のイベントタイプにイベントターゲットを登録するには `addEventNotification` メソッドを使ってください。これに加えて、`addXxxNotification` メソッドは、一般的なイベントタイプに対するハンドラを登録する簡単な方法を提供します。

次の例は、Amazon S3 バケットにオブジェクトが追加されたときに Lambda 関数をトリガーする方法を示しています。

```ts
import * as s3nots from 'aws-cdk-lib/aws-s3-notifications';

const handler = new lambda.Function(this, 'Handler', { /_…_/ });
const bucket = new s3.Bucket(this, 'Bucket');
bucket.addObjectCreatedNotification(new s3nots.LambdaDestination(handler));
```

### Removal policies

データベースや Amazon S3 バケット、さらには Amazon ECR レジストリなど、永続的なデータを保持するリソースは、それらを含む AWS CDK スタックが破壊されたときに永続オブジェクトを削除するかどうかを示す削除ポリシーを持っています。削除ポリシーを指定する値は、`aws-cdk-lib` モジュールの `RemovalPolicy` enum を通じて利用可能です。

:::message
**Note**
データを永続的に保存するリソース以外にも、別の目的で使用される removalPolicy を持つ場合があります。例えば、Lambda 関数のバージョンは、新しいバージョンがデプロイされたときに、与えられたバージョンを保持するかどうかを決定するために removePolicy 属性を使用します。これらは、Amazon S3 バケットや DynamoDB テーブルの削除ポリシーと比較して、異なる意味とデフォルトを持っています。
:::

| Value                 | meaning                                                                                                                                                                                                                              |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| RemovalPolicy.RETAIN  | スタック破棄時にリソースの内容を保持する（デフォルト）。リソースはスタックから孤立し、手動で削除する必要があります。リソースがまだ存在する状態でスタックを再デプロイしようとすると、名前の衝突によるエラーメッセージが表示されます。 |
| RemovalPolicy.DESTROY | リソースはスタックと一緒に破棄されます。                                                                                                                                                                                             |

AWS CloudFormation は、削除ポリシーが `DESTROY` に設定されていても、ファイルを含む Amazon S3 バケットを削除しない。これを行おうとすると、AWS CloudFormation のエラーになります。バケットを破棄する前に AWS CDK にバケットからすべてのファイルを削除させるには、バケットの `autoDeleteObjects` プロパティを `true` に設定します。

以下は、`RemovementPolicy` が `DESTROY`、`autoDeleteOjbects` が `true` に設定された Amazon S3 バケットを作成する場合の例です。

```ts
import _ as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import _ as s3 from 'aws-cdk-lib/aws-s3';

export class CdkTestStack extends cdk.Stack {
constructor(scope: Construct, id: string, props?: cdk.StackProps) {
  super(scope, id, props);

  const bucket = new s3.Bucket(this, 'Bucket', {
    removalPolicy: cdk.RemovalPolicy.DESTROY,
    autoDeleteObjects: true
  });

  }
}
```

`applyRemovalPolicy()`メソッドにより、基盤となる AWS CloudFormation リソースに直接削除ポリシーを適用することも可能です。このメソッドは、AWS CloudFormation スタック、Amazon Cognito ユーザープール、Amazon DocumentDB データベースインスタンス、Amazon EC2 ボリューム、Amazon OpenSearch Service ドメイン、Amazon FSx ファイルシステム、Amazon SQS キューなど、L2 リソースのプロップに `removalPolicy` プロパティがない一部のステートフルリソースで利用可能です。

```ts
const resource = bucket.node.findChild("Resource") as cdk.CfnResource;
resource.applyRemovalPolicy(cdk.RemovalPolicy.DESTROY);
```

:::message
**Note**
AWS CDK の `RemovementPolicy` は AWS CloudFormation の `DeletionPolicy` に変換されますが、AWS CDK のデフォルトはデータの保持であり、AWS CloudFormation のデフォルトと逆になっています。
:::
