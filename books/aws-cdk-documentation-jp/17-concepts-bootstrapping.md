---
title: "Concepts - Bootstrapping"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html)

## tl;dt

CDK を使うときに最初に唱える呪文。ここに書かれている内容が必要になったことはないけど、大量の環境に bootstrap 唱えたいときとかは（CFn で Stack Sets 使う話とか）参考になりそう。

## 本文

AWS CDK アプリを AWS [environment](./05-concepts-environments)（AWS アカウントとリージョンの組み合わせ）にデプロイするとき、デプロイを実行するために AWS CDK が必要とするリソースをプロビジョニングすることが必要になる場合があります。これらのリソースには、ファイルを格納するための Amazon S3 バケットと、デプロイを実行するために必要なパーミッションを付与する IAM ロールが含まれます。これらの初期リソースをプロビジョニングするプロセスは、bootstrap と呼ばれています。

以下のいずれかに該当する場合、environment を bootstrap する必要があります。

- デプロイされる AWS CDK スタックが [Assets](./11-concepts-assets) を使用している。
- アプリで生成された AWS CloudFormation template が 50 キロバイトを超える。

必要なリソースは、bootstrap スタックと呼ばれる AWS CloudFormation スタックで定義されており、通常 `CDKToolkit` という名前になっています。他の AWS CloudFormation スタックと同様に、デプロイされると AWS CloudFormation コンソールに表示されます。

:::message
**Note**
CDK v2 は Modern template と呼ばれる bootstrap template を使用しています。CDK v1 の Legacy template は v2 ではサポートされていません。
:::

environment は独立しているので、複数の environment（異なる AWS アカウントや同一アカウント内の異なるリージョン）にデプロイする場合は、それぞれの environment を別々に bootstrap する必要があります。

:::message alert
**Important**
bootstrap されたリソースに保存されたデータに対して、AWS の料金が発生する場合があります。
:::

:::message
**Note**
bootstrap template の古いバージョンでは、デフォルトで各 bootstrap environment にカスタマーマスターキー（CMK）が作成されます。CMK の課金を避けるには、これらの environment を `--no-bootstrap-customer-key` を使用して再 bootstrap してください。現在のデフォルトでは、CMK を使用しないことで、これらの費用を回避することができます。
:::

If you attempt to deploy an AWS CDK application that requires bootstrap resources into an environment that does not have them, you receive an error message telling you that you need to bootstrap the environment.

If you are using CDK Pipelines to deploy into another account's environment, and you receive a message like the following:

bootstrap リソースを必要とする AWS CDK アプリケーションを、bootstrap リソースがない environment にデプロイしようとすると、environment を bootstrap する必要があることを伝えるエラーメッセージが表示されます。

CDK Pipelines を使用して他のアカウントの environment にデプロイしているときに、次のようなメッセージが表示された場合、

```
Policy contains a statement with one or more invalid principals
```

このエラーメッセージは、適切な IAM ロールが他の environment に存在しないことを意味し、bootstrap が行われていないことが原因である可能性が高いです。

:::message
Note
CDK Pipelines を使用してそのアカウントにデプロイしている場合は、そのアカウントの bootstrap スタックの削除、再作成はしないでください。Pipelines が動作しなくなります。bootstrap スタックを新しいバージョンに更新するには、代わりに `cdk bootstrap` を再実行して、bootstrap スタックを所定の位置に更新してください。
:::

## How to bootstrap

bootstrap とは、AWS CloudFormation の template を特定の AWS environment（アカウントとリージョン）に配備することです。bootstrap template は、bootstrap されたリソースのいくつかの側面をカスタマイズするパラメータを受け付けます（[Customizing bootstrapping](#customizing-bootstrapping)を参照）。したがって、2 つの方法のうちの 1 つで bootstrap を行うことができます。

- AWS CDK Toolkit の `cdk bootstrap` コマンドを使用します。これは最もシンプルな方法で、bootstrap する environment が数少ない場合に有効です。
- AWS CDK Toolkit が提供する template を、他の AWS CloudFormation デプロイツールを使ってデプロイする。これにより、AWS CloudFormation コンソールや AWS CLI だけでなく、AWS CloudFormation Stack Sets や AWS Control Tower を使用することができます。デプロイ前に template に小さな修正を加えることもできます。この方法はより柔軟で、大規模なデプロイメントに適しています。

environment を複数回 bootstrap することはエラーではありません。bootstrap する environment がすでに bootstrap されている場合、その bootstrap スタックは必要に応じてアップグレードされます。

### Bootstrapping with the AWS CDK Toolkit

`cdk bootstrap` コマンドは、1 つまたは複数の AWS environment を bootstrap するために使用します。このコマンドの基本形は、指定した 1 つ以上の AWS environment（この例では 2 つ）を bootstrap することです。

```sh
cdk bootstrap aws://ACCOUNT-NUMBER-1/REGION-1 aws://ACCOUNT-NUMBER-2/REGION-2 ...
```

次の例は、それぞれ 1 つ、2 つの environment の bootstrap を示しています。(どちらも同じ AWS アカウントを使用しています。) 2 番目の例で示されているように、environment を指定する際に `aws://`というプレフィックスは任意です。

```sh
cdk bootstrap aws://123456789012/us-east-1
cdk bootstrap 123456789012/us-east-1 123456789012/us-west-1
```

`cdk bootstrap` コマンドで少なくとも 1 つの environment を指定しない場合、AWS CDK Toolkit はカレントディレクトリにある AWS CDK アプリを合成し、アプリで参照されるすべての environment を bootstrap します。スタックが environment に依存しない（つまり、`env` プロパティを持っていない）場合、CDK の environment（例えば、`--profile` を使用して指定されたもの、またはそれ以外のデフォルトの AWS environment）は、スタックを environment 固有にするために適用され、その environment が次に bootstrap されます。

例えば、次のコマンドは `prod` AWS プロファイルを使用して現在の AWS CDK アプリを合成し、その environment を bootstrap します。

```sh
cdk bootstrap --profile prod
```

### Bootstrapping from the AWS CloudFormation template

AWS CDK の bootstrap は、AWS CloudFormation の template によって行われます。この template のコピーをファイル `bootstrap-template.yaml` で取得するには、以下のコマンドを実行します。

```sh
cdk bootstrap --show-template > bootstrap-template.yaml
```

この template は、[AWS CDK GitHub repository](https://github.com/aws/aws-cdk/blob/master/packages/aws-cdk/lib/api/bootstrap/bootstrap-template.yaml)でも公開されています。

AWS CloudFormation template のお好みのデプロイメントメカニズムを使用して、この template をデプロイしてください。例えば、次のコマンドは、AWS CLI を使用して template をデプロイします。

```sh
aws cloudformation create-stack \
 --stack-name CDKToolkit \
 --template-body file://bootstrap-template.yaml
```

## Bootstrapping template

前述の通り、AWS CDK v1 では Legacy と Modern の 2 つの bootstrap template をサポートしていました。CDK v2 は Modern template のみをサポートしています。参考までに、これら 2 つの template のハイレベルな違いを以下に示します。

| Feature                            | Legacy (v1 only)                                                                          | Modern (v1 and v2)                                                                                                                                       |
| ---------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cross-account deployments**      | Not allowed                                                                               | Allowed                                                                                                                                                  |
| **AWS CloudFormation Permissions** | 現在のユーザーの権限でデプロイする（AWS プロファイルや environment 変数などで決定される） | bootstrap スタックのプロビジョニング時に指定された権限を使用してデプロイする（例:`--trust` を使用する）                                                  |
| **Versioning**                     | bootstrap スタックのバージョンは 1 つだけ                                                 | Bootstrap スタックはバージョン管理されており、将来のバージョンで新しいリソースが追加されたり、AWS CDK アプリが最小バージョンを要求したりすることができる |
| **Resources**\*                    | Amazon S3 bucket                                                                          | - Amazon S3 bucket<br/>- AWS KMS key<br/>- IAM roles<br/>- Amazon ECR repository<br/>- SSM parameter for versioning                                      |
| **Resource naming**                | Automatically generated                                                                   | Deterministic                                                                                                                                            |
| **Bucket encryption**              | Default key                                                                               | Customer-managed key                                                                                                                                     |

\* _必要に応じて、bootstrap template にリソースを追加します。_

Legacy template を使用して bootstrap された environment は、CDK v2 で使用するために再 bootstrap することで Modern template にアップグレードすることができます（そして、そうしなければなりません）。Legacy バケットを削除する前に、少なくとも一度は environment 内のすべての AWS CDK アプリケーションを再デプロイしてください。

## Customizing bootstrapping

bootstrap リソースをカスタマイズする方法は 2 つあります。

- `cdk bootstrap` コマンドでコマンドラインパラメータを使用する。これにより、template のいくつかの側面を変更することができます。
- デフォルトの bootstrap template を変更し、それを自分でデプロイする。これにより、bootstrap リソースを無制限に制御することができます。

以下のコマンドラインオプションは、CDK Toolkit の `cdk bootstrap` と一緒に使用すると、bootstrap template に一般的に必要とされる調整を提供します。

- `--bootstrap-bucket-name` は、Amazon S3 バケットの名前をオーバーライドします。CDK アプリの変更が必要な場合があります（[Stack synthesizers](#bootstrapping_synthesizers) を参照）。
- `--bootstrap-kms-key-id` は、S3 バケットを暗号化するために使用される AWS KMS キーをオーバーライドします。
- `--cloudformation-execution-policies` は、スタックのデプロイ時に AWS CloudFormation が想定するデプロイロールにアタッチされるべき、マネージドポリシーの ARN を指定します。少なくとも 1 つのポリシーが必要で、そうでない場合、AWS CloudFormation はパーミッションなしでデプロイを試み、デプロイは失敗します。

:::message
**Tip**
ポリシーは、以下のように、ポリシーの ARN をカンマで区切り、単一の文字列引数として渡す必要があります。

```
--cloudformation-execution-policies "arn:aws:iam::aws:policy/AWSLambda_FullAccess,arn:aws:iam::aws:policy/AWSCodeDeployFullAccess".
```

:::

- `--qualifier` bootstrap スタック内のすべてのリソースの名前に追加される文字列です。qualifier を使用すると、同じ environment で 2 つの bootstrap スタックをプロビジョニングするときに、名前の衝突を避けることができます。デフォルトは `hnb659fds` です（この値には意味がありません）。qualifier を変更するには、AWS CDK アプリを変更する必要があります（[Stack synthesizers](#bootstrapping_synthesizers) を参照）
- `--tags` は、bootstrap スタックに 1 つ以上の AWS CloudFormation タグを追加します。
- `--trust` は、bootstrap される environment にデプロイすることができる AWS アカウントをリストアップします。他の environment にある CDK パイプラインがデプロイする environment を bootstrap するときに、このフラグを使用します。bootstrap を行うアカウントは常に信頼されます。
- `--trust-for-lookup` は、bootstrap されている environment からコンテキスト情報を検索することができる AWS アカウントをリストアップします。このフラグを使用すると、アカウントに直接スタックをデプロイする権限を与えずに、environment にデプロイされるスタックを合成する権限を与えることができます。trust で指定されたアカウントは、コンテキスト検索において常に信頼されます。
- `--termination-protection` は bootstrap スタックが削除されないようにします（AWS CloudFormation User Guide の [Protecting a stack from being deleted](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-protect-stacks.html) を参照）。

:::message alert
**Important**
Modern bootstrap template は、`--trust` リスト内の任意の AWS アカウントに`--cloudformation-execution-policies` によって暗示されるパーミッションを効果的に付与し、デフォルトでは bootstrap アカウントの任意のリソースへの読み取りおよび書き込みのパーミッションを拡張します。bootstrap スタックに、自分が使いやすいポリシーと信頼できるアカウントを設定することを確認してください。
:::

## Customizing the template

AWS CDK Toolkit のスイッチで提供できる以上のカスタマイズが必要な場合、bootstrap template を必要に応じて変更することができます。template は `--show-template` フラグで取得できることを覚えておいてください。

```
export CDK_NEW_BOOTSTRAP=1
cdk bootstrap --show-template
```

変更する場合は、[bootstrapping template contract](#the_bootstrapping_template_contract)を遵守する必要があります。

[AWS CloudFormation template からの bootstrap](#bootstrapping_from_the_aws_cloudformation_template) で説明したように、または `cdk bootstrap --template` を使用して、修正した template をデプロイしてください。

```sh
cdk bootstrap --template bootstrap-template.yaml
```

## Stack synthesizers

AWS CDK アプリは、デプロイ可能なスタックを正常に合成するために、利用可能な bootstrap リソースについて知っておく必要があります。stack synthesizer は、スタックの template が合成される方法を制御する AWS CDK クラスで、bootstrap リソースの使用方法（例えば、bootstrap バケットに格納されているアセットを参照する方法）を含みます。

AWS CDK の組み込みスタック合成器は、`DefaultStackSynthesizer` と呼ばれています。これは、クロスアカウントのデプロイメントと [CDK Pipelines](https://docs.aws.amazon.com/cdk/v2/guide/cdk_pipeline.html) のデプロイメントのための機能を含んでいます。

`synthesizer` プロパティを使用してスタックをインスタンス化するときに、スタック合成器を渡すことができます。

```ts
new MyStack(this, "MyStack", {
  // stack properties
  synthesizer: new DefaultStackSynthesizer({
    // synthesizer properties
  }),
});
```

`synthesizer` property を指定しない場合、`DefaultStackSynthesizer` が使用されます。

## Customizing synthesis

bootstrap template に加えた変更に応じて、合成もカスタマイズする必要があるかもしれません。`DefaultStackSynthesizer` は、以下に説明するプロパティを使用してカスタマイズすることができます。これらのプロパティーが必要なカスタマイズを提供しない場合、`IStackSynthesizer` を実装するクラスとして synthesizer を書くことができる（おそらく `DefaultStackSynthesizer` から派生する）。

### Changing the qualifier

qualifier は、別々の bootstrap スタックのリソースを区別するために、bootstrap リソースの名前に追加されます。同じ environment（AWS アカウントとリージョン）で 2 つの異なるバージョンの bootstrap スタックをデプロイするには、スタックに異なる qualifier を付ける必要があります。この機能は、CDK 自体の自動テスト間の名前分離のために意図されています。AWS CloudFormation の実行ロールに与えられた IAM 権限を非常に正確にスコープダウンできない限り、1 つのアカウントで 2 つの異なる bootstrap スタックを持つことによる特権分離のメリットはないので、通常この値を変更する必要はありません。

qualifier を変更するには、プロパティで合成器をインスタンス化するか、`DefaultStackSynthesizer` を構成する。

```ts
new MyStack(this, "MyStack", {
  synthesizer: new DefaultStackSynthesizer({
    qualifier: "MYQUALIFIER",
  }),
});
```

または、qualifier を `cdk.json` のコンテキストキーとして設定することでも対応可能です。

```json
{
  "app": "...",
  "context": {
    "@aws-cdk/core:bootstrapQualifier": "MYQUALIFIER"
  }
}
```

### Changing the resource names

他のすべての `DefaultStackSynthesizer` プロパティは、bootstrap template 内のリソースの名前に関連しています。あなたが bootstrap template を修正し、リソース名または命名方式を変更した場合にのみ、これらのプロパティのいずれかを提供する必要があります。

全てのプロパティは、特別なプレースホルダー `${Qualifier}`, `${AWS::Partition}`, `${AWS::AccountId}`, 及び `${AWS::Region}` を受け付けます。これらのプレースホルダーは、それぞれ `qualifier` パラメータの値、スタック environment の AWS パーティション、アカウント ID、リージョンの値で置き換えられます。

次の例では、シンセサイザーをインスタンス化するように、`DefaultStackSynthesizer` で利用可能なすべてのプロパティをデフォルト値とともに示しています。

```ts
new DefaultStackSynthesizer({
  // ファイルアセット用のS3バケット名
  fileAssetsBucketName:
    "cdk-${Qualifier}-assets-${AWS::AccountId}-${AWS::Region}",
  bucketPrefix: "",

  // Dockerイメージアセット用ECRリポジトリ名
  imageAssetsRepositoryName:
    "cdk-${Qualifier}-container-assets-${AWS::AccountId}-${AWS::Region}",

  // CLIとPipelineがここでデプロイするために想定しているロールのARN
  deployRoleArn:
    "arn:${AWS::Partition}:iam::${AWS::AccountId}:role/cdk-${Qualifier}-deploy-role-${AWS::AccountId}-${AWS::Region}",
  deployRoleExternalId: "",

  // ファイルアセット公開に使用するロールのARN（deployロールから想定）
  fileAssetPublishingRoleArn:
    "arn:${AWS::Partition}:iam::${AWS::AccountId}:role/cdk-${Qualifier}-file-publishing-role-${AWS::AccountId}-${AWS::Region}",
  fileAssetPublishingExternalId: "",

  // Dockerアセットパブリッシングに使用されるロールのARN（deployロールから仮定）
  imageAssetPublishingRoleArn:
    "arn:${AWS::Partition}:iam::${AWS::AccountId}:role/cdk-${Qualifier}-image-publishing-role-${AWS::AccountId}-${AWS::Region}",
  imageAssetPublishingExternalId: "",

  // デプロイメントを実行するためにCloudFormationに渡されるロールのARN
  cloudFormationExecutionRole:
    "arn:${AWS::Partition}:iam::${AWS::AccountId}:role/cdk-${Qualifier}-cfn-exec-role-${AWS::AccountId}-${AWS::Region}",

  // 起動スタックのバージョン番号を記述したSSMパラメータ名
  bootstrapStackVersionSsmParameter: "/cdk-bootstrap/${Qualifier}/version",

  // すべてのtemplateに、必要なbootstrapスタックのバージョンを確認するルールを追加します。
  generateBootstrapVersionRule: true,
});
```

## The bootstrapping template contract

bootstrap スタックの要件は、使用するスタックシンセサイザに依存します。もしあなたが独自のスタック合成器を書くなら、あなたは合成器が必要とする bootstrap リソースと、合成器がそれを見つける方法を完全に制御することができます。このセクションでは、`DefaultStackSynthesizer` が bootstrap template に持つ期待について説明します。

### Versioning

template には、よく知られた名前の SSM パラメータを作成するためのリソースと、template のバージョンを反映させるためのアウトプットが含まれている必要があります。

```yaml
Resources:
  CdkBootstrapVersion:
    Type: AWS::SSM::Parameter
    Properties:
      Type: String
      Name:
        Fn::Sub: "/cdk-bootstrap/${Qualifier}/version"
      Value: 4
Outputs:
  BootstrapVersion:
    Value:
      Fn::GetAtt: [CdkBootstrapVersion, Value]
```

### Roles

`DefaultStackSynthesizer` は、5 つの目的のために 5 つの IAM ロールを必要とします。デフォルトのロールを使用しない場合、シンセサイザーは使用したいロールの ARN を伝える必要があります。ロールは以下の通りです。

- deployment ロールは、AWS CDK Toolkit と AWS CodePipeline が environment にデプロイするために想定しているロールです。その `AssumeRolePolicy` は、environment にデプロイできる人を制御します。このロールが必要とする権限は、template で見ることができます。
- lookup ロールは、AWS CDK Toolkit が environment 内のコンテキスト検索を実行するために想定されています。その `AssumeRolePolicy` は、誰が environment にデプロイできるかを制御します。このロールが必要とする権限は、template で見ることができます。
- file publishing ロールと画像公開ロールは、AWS CDK Toolkit と AWS CodeBuild プロジェクトが environment にアセットを公開するために想定しているもので、それぞれ S3 バケットと ECR リポジトリに書き込むためのものです。これらのロールは、これらのリソースへの書き込みアクセスを必要とします。
- AWS CloudFormation execution ロールは、AWS CloudFormation に渡され、実際のデプロイメントを実行します。そのパーミッションは、デプロイが実行されるパーミッションです。パーミッションは、管理されたポリシー ARN をリストアップするパラメータとしてスタックに渡されます。

### Outputs

AWS CDK Toolkit では、起動用スタックに以下の CloudFormation 出力が存在することが必要です。

- `BucketName`: ファイルアセットバケットの名前
- `BucketDomainName`: ドメイン名形式のファイルアセットバケット
- `BootstrapVersion`: bootstrap スタックの現在のバージョン

### Template history

bootstrap template はバージョン管理され、AWS CDK 自体とともに進化していきます。もしあなたが独自の bootstrap template を提供する場合、あなたのがすべての CDK の機能で動作し続けることを確実にするために、標準のデフォルト template でそれを最新に保つ。このセクションには、各バージョンで行われた変更のリストが含まれています。

| Template version AWS | CDK version | Changes                                                                                                                                                               |
| -------------------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1                    | 1.40.0      | Bucket、Key、Repository、Roles を含む template の初期バージョン                                                                                                       |
| 2                    | 1.45.0      | アセット公開ロールをファイル公開と画像公開のロールに分割                                                                                                              |
| 3                    | 1.46.0      | FileAssetKeyArn のエクスポートを追加し、アセットコンシューマーに復号化権限を追加できるように                                                                          |
| 4                    | 1.61.0      | CdkBootstrapVersion SSM パラメータを追加し、スタック名を知らなくても bootstrap スタックのバージョンを確認できるように                                                 |
| 5                    | 1.87.0      | デプロイメントロールは、SSM パラメータを読み取れるように                                                                                                              |
| 6                    | 1.108.0     | デプロイメントロールとは別に、ルックアップロールを追加                                                                                                                |
| 6                    | 1.109.0     | aws-cdk:bootstrap-role タグをデプロイメント、ファイルパブリッシング、およびイメージパブリッシングロールにアタッチ                                                     |
| 7                    | 1.110.0     | デプロイロールは、ターゲットアカウントの Bucket を直接読めなくなる（ただし、このロールは事実上管理者であり、AWS CloudFormation 権限を使えばいつでも Bucket を読める） |
| 8                    | 1.114.0     | lookup ロールはターゲット environment に対する完全な読み取り専用権限を持ち、同様に aws-cdk:bootstrap-role タグも持つ                                                  |
| 9                    | 2.1.0       | S3 アセットのアップロードが、一般的に参照される暗号化 SCP によって拒否されるのを修正                                                                                  |
