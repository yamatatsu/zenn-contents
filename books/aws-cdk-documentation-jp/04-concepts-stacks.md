---
title: "Concepts - Stacks"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/stacks.html)

## 著者の感想

この章では、全体的に、CloudFormation に慣れた人が CDK を使うときに覚えておくと便利なことが記載されています。特に[Stack API](#stack-api)は、「CFn でのあの記法、CDK ではどう使うんだろ？」に対応できる API が紹介されています。

## 本文

AWS CDK でのデプロイの単位は、Stack と呼ばれます。Stack の Scope 内で直接または間接的に定義されたすべての AWS リソースは、単一のユニットとしてプロビジョニングされます。

Stack は、AWS CloudFormation Stack を介して実装されているので、[AWS CloudFormation と同じ制限](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html)に従います。

app では任意の数の Stack を定義できます。`Stack` Construct のインスタンスは Stack を表し、前に示した`MyFirstStack`の例のように app の scope 内で直接定義することも、ツリー内の Construct によって間接的に定義することもできます。

たとえば、次のコードは 2 つの Stack を持つ app を定義します。

```ts
const app = new App();

new MyFirstStack(app, "stack1");
new MySecondStack(app, "stack2");

app.synth();
```

app のすべての Stack を一覧表示するには、`cdk ls` コマンドを実行します。これは、上記の app の場合は次の出力になります。

```
stack1
stack2
```

複数の Stack を持つ app に対して `cdk synth` コマンドを実行すると、Cloud assemblies には、Stack インスタンスごとに個別のテンプレートが含まれます。2 つの Stack が同じクラスのインスタンスであっても、AWS CDK はそれらを 2 つの個別のテンプレートとして出力します。

`cdk synth` コマンドで Stack 名を指定することにより、各テンプレートを合成できます。次の例では、`stack1` のテンプレートを合成します。

```
cdk synth stack1
```

このアプローチ（プログラミングによってプロビジョニング内容を合成するアプローチ）は、AWS CloudFormation テンプレートが通常使用される方法とは概念的に異なっています。AWS CloudFormation テンプレートの一般的な使用では、[AWS CloudFormation パラメータ](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html)を変えながらテンプレートを再利用して、複数回デプロイされるます。CDK でも AWS CloudFormation パラメータを定義できることはできますが、AWS CloudFormation パラメータはデプロイ時にのみ解決される（つまり app のプログラム中でパラメータの値を参照できない）ため、一般的には推奨されません。例えば、パラメータの値に基づいて app に条件付きでリソースを含めるには、[AWS CloudFormation condition](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/conditions-section-structure.html) を設定し、この condition でリソースにタグ付けする必要があります。AWS CDK は具体的なテンプレートが合成時に決定されるアプローチを取っているため、if 文を使って値をチェックし、リソースを定義すべきか、何らかの動作を適用すべきかを判断することができます。

:::message
**Note**
AWS CDK は可能な限り合成時に解決することで、プログラミング言語のイディオム的で自然な利用（if 文や for 文や抽象化など）を可能にします。
:::

他の Construct と同様に、Stack も一緒にグループとして構成することができます。次のコードは、3 つの Stack（`ControlPlane`, `DataPlane`, `Monitoring`）から構成されるサービスの例です。このサービス構成要素は、ベータ環境と本番環境の 2 回定義されています。

```ts
import { App, Stack } from "aws-cdk-lib";
import { Construct } from "constructs";

interface EnvProps {
  prod: boolean;
}

// imagine these stacks declare a bunch of related resources
class ControlPlane extends Stack {}
class DataPlane extends Stack {}
class Monitoring extends Stack {}

class MyService extends Construct {
  constructor(scope: Construct, id: string, props?: EnvProps) {
    super(scope, id);

    // we might use the prod argument to change how the service is configured
    new ControlPlane(this, "cp");
    new DataPlane(this, "data");
    new Monitoring(this, "mon");
  }
}

const app = new App();
new MyService(app, "beta");
new MyService(app, "prod", { prod: true });

app.synth();
```

この app は、最終的に環境ごとに 3 つ、つまり 6 つの Stack で構成されます。

```
$ cdk ls

betacpDA8372D3
betadataE23DB2BA
betamon632BD457
prodcp187264CE
proddataF7378CE5
prodmon631A1083
```

AWS CloudFormation stack の物理名は、ツリー内の Stack の Construct path パスに基づいて AWS CDK によって自動的に決定されます。デフォルトでは、Stack の名前は `Stack` オブジェクトの construct ID から派生しますが、次のように `stackName` prop を使用して明示的な名前を指定できます。

```ts
new MyStack(this, "not:a:stack:name", { stackName: "this-is-stack-name" });
```

### Stack API

The Stack object provides a rich API, including the following:

[Stack](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.Stack.html) object は、以下を含む、豊富な API を提供します

- `Stack.of(construct)` – Construct が定義されている Stack を返す静的メソッド。これは、再利用可能な Construct 内から Stack と対話する必要がある場合に役立ちます。scope から Stack が見つからない場合、呼び出しは失敗します。

- `stack.stackName` – Stack の physical name を返します。前述のように、すべての AWS CDK Stack は physical name を持ち、これは AWS CDK が合成中に解決されます。

- `stack.region` and `stack.account` – この Stack がデプロイされる AWS リージョンとアカウントをそれぞれ返します。これらのプロパティは、app 内で明示的に示されている場合はそれを返し、示されていない場合は AWS CloudFormation 疑似パラメーターとして扱われる文字列エンコードトークンを返します。 Stack の環境がどのように決定されるかについては、[Environments](https://docs.aws.amazon.com/cdk/v2/guide/environments.html) を参照してください。

- `stack.addDependency(stack)` – 2 つの Stack 間の依存関係の順序を明示的に定義するために使用できます。この順序は、複数の Stack を一度にデプロイするときに `cdk deploy` コマンドによって尊重されます。

- `stack.tags` – Stack レベルのタグを追加または削除するために使用できる [TagManager](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.TagManager.html) を返します。このタグマネージャーは、Stack 内のすべてのリソースにタグを付け、AWS CloudFormation を介して作成されたときに Stack 自体にもタグを付けます。

- `stack.partition`, `stack.urlSuffix`, `stack.stackId`, and `stack.notificationArn` – `{ "Ref": "AWS::Partition" }` のような AWSCloudFormation 疑似パラメーターとして解決されるトークンを返します。これらのトークンは特定の Stack オブジェクトに関連付けられているため、AWS CDK フレームワークは Stack を跨いで参照できます。

- `stack.availabilityZones` – この Stack がデプロイされている環境で使用可能なアベイラビリティーゾーンのセットを返します。環境に依存しない Stack の場合、これは常に 2 つのアベイラビリティーゾーンの配列を返しますが、環境固有の Stack の場合、AWS CDK は環境にクエリを実行し、指定したリージョンで利用可能なアベイラビリティーゾーンの正確なセットを返します。

- `stack.parseArn(arn)` and `stack.formatArn(comps)` – Amazon リソース名（ARN）の操作に使用できます。

- `stack.toJsonString(obj)` – AWS CloudFormation テンプレートに埋め込むことができる JSON 文字列として任意のオブジェクトをフォーマットするために使用できます。オブジェクトには、展開中にのみ解決されるトークン、属性、および参照を含めることができます。

- `stack.templateOptions` – Stack の変換、説明、メタデータなどの AWS CloudFormation テンプレートオプションを指定できます。

### Nested Stacks

NestedStack construct は、Stack のために AWS CloudFormation 500-リソース制限（CFn の Stack はクォータによりリソース数を 500 までしか持てない）を回避する方法を提供しています。nested stack は、それを含む Stack 内の 1 つのリソースとしてのみカウントされますが、それ自体には、追加の nested stack を含め、最大 500 のリソースを含めることができます。

nested stack のスコープは、`Stack` または `NestedStack` construct である必要があります。nested stack をインスタンス化するときに、最初のパラメーター（scope）として親 Stack を渡すだけ利用することができます。この制約を除けば、nested stack で構成を定義することは、通常の Stack とまったく同じように機能します。

合成時に、nested stack は独自の AWS CloudFormation テンプレートに合成され、デプロイ時に AWS CDK ステージングバケットにアップロードされます。nested stack は親 Stack にバインドされ、独立したデプロイメントアーティファクトとしては扱われません。それらは`cdk list`によってリストされず、`cdk deploy`によって展開することもできません 。

親 Stack と nested stack 間の参照は、他のクロス Stack 参照と同様に、生成された AWS CloudFormation テンプレートの Stack パラメーターと出力に自動的に変換されます 。

:::message alert
**Warning**
nested stack の展開前は、security posture の変更は表示されません。この情報は、最上位 Stack に対してのみ表示されます。
:::
