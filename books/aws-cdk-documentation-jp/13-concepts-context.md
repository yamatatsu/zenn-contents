---
title: "Concepts - Context"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/context.html)

## 著者の感想

僕は、TS を使うなら、`cdk.json`や`cdk.context.json`ではなく`settings.ts`とか`enviroment.ts`みたいなファイルに TS で書けばいいと思います。環境名(環境情報を引くキー)と秘匿情報以外は。

## 本文

コンテキスト値は、アプリ、スタック、またはコンストラクトに関連付けることができるキーと値のペアです。AWS CDK は、アカウント内の Availability Zone やインスタンスの起動に使用される Amazon Machine Image (AMI) ID など、AWS アカウントからの情報をキャッシュするためにコンテキストを使用します。[Feature flags](./14-concepts-feature-flags)もまた、コンテキスト値です。アプリやコンストラクトで使用するために、独自のコンテキスト値を作成することができます。

コンテキストのキーは文字列で、値は JSON でサポートされている任意の型（数値、文字列、配列、オブジェクト）を使用することができます。

独自のコンテキスト値を作成する場合は、他のパッケージのコンテキスト値と競合しないように、ライブラリのパッケージ名をキーに含めるようにします。

ほとんどのコンテキスト値は、特定の AWS 環境に関連付けられており、特定の CDK アプリは複数の環境にデプロイされる可能性があるため、各環境のコンテキスト値を設定できるようにすることが重要です。これは、異なる環境からの値が競合しないように、コンテキストキーに AWS アカウントとリージョンを含めることで達成されます。

以下のコンテキストキーは、アカウントとリージョンを含む AWS CDK で使用される形式を表しています。

```
availability-zones:account=123456789012:region=eu-central-1
```

:::message alert
**Important**
コンテキスト値は、AWS CDK とそのコンストラクタ（あなたが書いたコンストラクタも含む）によって管理されます。一般的に、手動でファイルを編集してコンテキスト値を追加または変更するべきではありません。どのような値がキャッシュされているかを確認するために、`cdk.context.json` を確認することは有用です。
:::

### Construct context

コンテキスト値は、6 つの方法で AWS CDK アプリに提供することができます。

- 現在の AWS アカウントから自動的に提供
- `cdk`コマンドの`--context`オプションで指定(値は常に文字列)
- プロジェクトの`cdk.context.json`ファイルに記述
- プロジェクトの`cdk.json`ファイルの`context`キー
- `~/.cdk.json` ファイルの`context`キー
- AWS CDK アプリで`construct.node.setContext()`メソッドを使用した場合

プロジェクトファイルの `cdk.context.json` は、AWS CDK が AWS アカウントから取得したコンテキスト値をキャッシュしている場所です。このプラクティスは、例えば、新しい Amazon Linux AMI がリリースされ、Auto Scaling グループが変更されたときに、デプロイメントへの予期しない変更を回避します。AWS CDK は、リストアップされた他のどのファイルにもコンテキストデータを書き込みません。

`cdk.context.json` は、git にコミットすることをお勧めします。これらの情報は、アプリケーションの状態の一部であり、一貫して合成とデプロイができることが重要だからです。また、コンテキスト値に依存するスタックの自動デプロイを成功させるためにも重要です（たとえば、CDK Pipelines を使用する）。

コンテキスト値は、それを作成したコンストラクトにスコープされ、子コンストラクトからは見えますが、兄弟コンストラクトからは見えません。AWS CDK Toolkit（cdk コマンド）で設定されたコンテキスト値は、自動、ファイル、`--context` オプションのいずれであっても、暗黙的に App コンストラクトに設定され、アプリ内のすべてのコンストラクトから見えるようになっています。

コンテキストの値は、`construct.node.tryGetContext` メソッドを使用して取得できます。要求されたエントリが現在のコンストラクトやその親に見つからない場合、結果は未定義（または Python では None などの言語に対応するもの）です。

### Context methods

AWS CDK は、AWS CDK アプリがコンテキスト情報を取得できるように、いくつかのコンテキストメソッドをサポートしています。例えば、指定した AWS アカウントとリージョンで利用可能な Availability Zone の一覧を [`stack.availabilityZones`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.Stack.html#availabilityzones) メソッドで取得することが可能です。

以下は、コンテキストメソッドです。

- [`HostedZone.fromLookup`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_route53.HostedZone.html#static-fromwbrlookupscope-id-query)
  - アカウント内のホストされたゾーンを取得します。
- [`stack.availabilityZones`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.Stack.html#availabilityzones)
  - サポートされているアベイラビリティゾーンを取得します。
- [`StringParameter.valueFromLookup`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ssm.StringParameter.html#static-valuewbrfromwbrlookupscope-parametername)
  - 現在のリージョンの Amazon EC2 Systems Manager Parameter Store から値を取得します。
- [`Vpc.fromLookup`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ec2.Vpc.html#static-fromwbrlookupscope-id-options)
  - アカウント内の既存の Amazon Virtual Private Clouds を取得します。
- [`LookupMachineImage`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ec2.LookupMachineImage.html)
  - Amazon Virtual Private Cloud の NAT インスタンスで使用するためのマシンイメージを検索します。

必要なコンテキスト値が利用できない場合、AWS CDK アプリは AWS CDK CLI にコンテキスト情報がないことを通知します。CLI は、現在の AWS アカウントに情報を問い合わせ、結果のコンテキスト情報を `cdk.context.json` ファイルに格納し、コンテキスト値で AWS CDK アプリを再実行します。

### Viewing and managing context

`cdk.context.json` ファイル内の情報を表示、管理するには、`cdk context` コマンドを使用します。この情報を確認するには、`cdk context` コマンドをオプションなしで使用します。出力は次のようなものになるはずです。

```
Context found in cdk.json:

┌───┬─────────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────┐
│ # │ Key                                                         │ Value                                                   │
├───┼─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
│ 1 │ availability-zones:account=123456789012:region=eu-central-1 │ [ "eu-central-1a", "eu-central-1b", "eu-central-1c" ]   │
├───┼─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
│ 2 │ availability-zones:account=123456789012:region=eu-west-1    │ [ "eu-west-1a", "eu-west-1b", "eu-west-1c" ]            │
└───┴─────────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────┘

Run
cdk context --reset KEY_OR_NUMBER
 to remove a context key. If it is a cached value, it will be refreshed on the next
cdk synth
.
```

コンテキスト値を削除するには、値の対応するキーまたは番号を指定して、`cdk context --reset` を実行します。次の例では、前の例の 2 番目のキーに対応する値（アイルランド地域のアベイラビリティゾーンのリスト）を削除します。

```sh
cdk context --reset 2
```

```
Context value
availability-zones:account=123456789012:region=eu-west-1
reset. It will be refreshed on the next SDK synthesis run.
```

したがって、Amazon Linux AMI の最新バージョンに更新したい場合は、前述の例でコンテキスト値の制御更新を行い、リセットしてから、再度アプリを合成してデプロイすることが可能です。

```sh
cdk synth
```

保存されているアプリのコンテキスト値をすべてクリアするには、次のように`cdk context --clear`を実行します。

```sh
cdk context --clear
```

`cdk.context.json`に格納されているコンテキスト値のみ、リセットまたはクリアすることができます。AWS CDK は他のコンテキスト値には触れません。これらのコマンドを使用してコンテキスト値がリセットされないようにするには、その値を`cdk.json`にコピーする必要があります。

### AWS CDK Toolkit --context flag

合成時やデプロイ時に CDK アプリにランタイムコンテキスト値を渡すには、`--context`（`-c`）オプションを使用します。

```sh
cdk synth --context key=value MyStack
```

To specify multiple context values, repeat the --context option any number of times, providing one key-value pair each time.

複数のコンテキスト値を指定するには、`--context`オプションを何度でも繰り返し、その都度 1 つのキーと値のペアを指定します。

```sh
cdk synth --context key1=value1 --context key2=value2 MyStack
```

複数のスタックをデプロイする場合、指定したコンテキスト値は通常、すべてのスタックに渡されます。必要であれば、コンテキスト値の前にスタック名を付けることで、各スタックに異なる値を指定することができます。

```sh
cdk synth --context Stack1:key=value --context Stack2:key=value Stack1 Stack2
```

### Example

以下は、AWS CDK コンテキストを使用して、既存の Amazon VPC をインポートする例です。

```ts
import _ as cdk from 'aws-cdk-lib';
import _ as ec2 from 'aws-cdk-lib/aws-ec2';
import { Construct } from 'constructs';

export class ExistsVpcStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpcid = this.node.tryGetContext('vpcid');
    const vpc = ec2.Vpc.fromLookup(this, 'VPC', {
      vpcId: vpcid,
    });

    const pubsubnets = vpc.selectSubnets({subnetType: ec2.SubnetType.PUBLIC});

    new cdk.CfnOutput(this, 'publicsubnets', {
      value: pubsubnets.subnetIds.toString(),
    });
  }
}
```

`cdk diff` を使って、コマンドラインでコンテキスト値を渡したときの効果を確認することができます。

```sh
cdk diff -c vpcid=vpc-0cb9c31031d0d3e22
```

```
Stack ExistsvpcStack
Outputs
[+] Output publicsubnets publicsubnets: {"Value":"subnet-06e0ea7dd302d3e8f,subnet-01fc0acfb58f3128f"}
```

その結果、コンテキストの値は次のように表示されます。

```
cdk context -j
```

```json
{
  "vpc-provider:account=123456789012:filter.vpc-id=vpc-0cb9c31031d0d3e22:region=us-east-1": {
    "vpcId": "vpc-0cb9c31031d0d3e22",
    "availabilityZones": ["us-east-1a", "us-east-1b"],
    "privateSubnetIds": [
      "subnet-03ecfc033225be285",
      "subnet-0cded5da53180ebfa"
    ],
    "privateSubnetNames": ["Private"],
    "privateSubnetRouteTableIds": [
      "rtb-0e955393ced0ada04",
      "rtb-05602e7b9f310e5b0"
    ],
    "publicSubnetIds": ["subnet-06e0ea7dd302d3e8f", "subnet-01fc0acfb58f3128f"],
    "publicSubnetNames": ["Public"],
    "publicSubnetRouteTableIds": [
      "rtb-00d1fdfd823c82289",
      "rtb-04bb1969b42969bcb"
    ]
  }
}
```
