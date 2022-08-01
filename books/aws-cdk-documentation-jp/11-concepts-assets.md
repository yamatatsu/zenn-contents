---
title: "Concepts - Assets"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/assets.html)

## 著者の感想

CDK の Assets を知らない場合は是非読んで欲しい。インフラデプロイとアプリケーションデプロイの境目を消して、CD の構成を容易にし、フルサイクルエンジニアが DevOps で働くのを大いに助ける、CDK における重要な機能の説明。

## 本文

アセットとは、AWS CDK のライブラリやアプリにバンドルできるローカルファイル、ディレクトリ、Docker イメージのことで、例えば、AWS Lambda 関数のハンドラーコードを含むディレクトリがこれにあたります。アセットとは、アプリが動作するために必要なあらゆるアーティファクトを表すことができる。

あなたは、特定の AWS 構成によって公開されている API を介してアセットを追加します。例えば、[lambda.Function](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda.Function.html) コンストラクトを定義すると、[code](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda.Function.html#code) プロパティで [asset](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda.Code.html#static-fromwbrassetpath-options)（ディレクトリ）を渡すことができます。`Function` はアセットを使ってディレクトリの内容をバンドルし、それを関数のコードに使用します。同様に、[ecs.ContainerImage.fromAsset](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs.ContainerImage.html#static-fromwbrassetdirectory-props) は、Amazon ECS のタスク定義時にローカルディレクトリから構築された Docker イメージを使用します。

### Assets in detail

アプリ内でアセットを参照すると、アプリケーションから合成された[Cloud assemblies](./03-concepts-apps#cloud_assemblies)には、ローカルディスクのどこでアセットを見つけるか、圧縮（ZIP）するディレクトリやビルドする Docker イメージなど、アセットの種類に応じてどのようなバンドル処理を行うか、AWS CDK CLI への指示が書かれたメタデータ情報が含まれています。

AWS CDK は、アセットのソースハッシュを生成し、構築時にこれを使用してアセットの内容が変更されたかどうかを判断することができます。

デフォルトでは、AWS CDK は、ソースハッシュの下に、Cloud assemblies ディレクトリ（デフォルトは `cdk.out`）にアセットのコピーを作成します。これは、Cloud assemblies が自己完結しており、デプロイのために別のホストに移動できるようにするためです。詳細については、Cloud assemblies を参照してください。

AWS CDK は、デプロイ時に AWS CDK CLI が指定する AWS CloudFormation のパラメータも合成します。AWS CDK は、これらのパラメータを使用して、アセットのデプロイ時の値を参照します。

AWS CDK がアセットを参照するアプリをデプロイするとき（アプリコードによって直接、またはライブラリを通して）、AWS CDK CLI は最初にそれらを準備し、Amazon S3 または Amazon ECR に公開し、その後のみスタックをデプロイする。AWS CDK は、公開されたアセットの場所を AWS CloudFormation のパラメータとして関連するスタックに指定し、その情報を使用して AWS CDK アプリ内でこれらの場所を参照できるようにします。

このセクションでは、フレームワークで利用可能な低レベルの API について説明します。

### Asset types

AWS CDK は、以下の種類のアセットをサポートしています。

##### Amazon S3 Assets

これらは、AWS CDK が Amazon S3 にアップロードするローカルファイルおよびディレクトリです。

##### Docker Image

AWS CDK が Amazon ECR にアップロードする Docker イメージです。

これらのアセットタイプは、以下のセクションで説明します。

#### Amazon S3 assets

ローカルのファイルやディレクトリをアセットとして定義すると、AWS CDK は [aws-s3-assets](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3_assets-readme.html) モジュールを通じて、それらをパッケージ化して Amazon S3 へアップロードすることが可能です。

以下の例では、ローカルディレクトリのアセットとファイルのアセットを定義しています。

```ts
import { Asset } from "aws-cdk-lib/aws-s3-assets";

// Archived and uploaded to Amazon S3 as a .zip file
const directoryAsset = new Asset(this, "SampleZippedDirAsset", {
  path: path.join(__dirname, "sample-asset-directory"),
});

// Uploaded to Amazon S3 as-is
const fileAsset = new Asset(this, "SampleSingleFileAsset", {
  path: path.join(__dirname, "file-asset.txt"),
});
```

ほとんどの場合、`aws-s3-assets` モジュールの API を直接使用する必要はありません。`aws-lambda` などのアセットをサポートするモジュールには、アセットを利用するための便利なメソッドが用意されています。Lambda 関数の場合、[fromAsset()](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda.Code.html#static-fromwbrassetpath-options)という静的メソッドで、ローカルファイルシステム内のディレクトリや.zip ファイルを指定することが可能です。

#### Lambda function example

一般的なユースケースは、関数のエントリポイントであるハンドラーコードを Amazon S3 アセットとして AWS Lambda 関数を作成することです。

以下の例では、Amazon S3 アセットを使ってローカルディレクトリの`handler`に Python ハンドラを定義し、ローカルディレクトリアセットを `code` プロパティとして Lambda 関数を作成しています。以下は、ハンドラの Python コードです。

```python
def lambda_handler(event, context):
  message = 'Hello World!'
  return {
    'message': message
  }
```

AWS CDK のメインアプリのコードは、以下のようになります。

```ts
import * as cdk from "aws-cdk-lib";
import { Constructs } from "constructs";
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as path from "path";

export class HelloAssetStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new lambda.Function(this, "myLambdaFunction", {
      code: lambda.Code.fromAsset(path.join(__dirname, "handler")),
      runtime: lambda.Runtime.PYTHON_3_6,
      handler: "index.lambda_handler",
    });
  }
}
```

Function では、アセットを使ってディレクトリの内容をバンドルし、それを関数のコードに使用します。

#### Deploy-time attributes example

Amazon S3 のアセットタイプは、AWS CDK のライブラリやアプリで参照可能な[deploy-time attributes](./06-concepts-resources#resource_attributes)も公開しています。AWS CDK CLI コマンドの `cdk synth` は、アセットのプロパティを AWS CloudFormation のパラメータとして表示します。

次の例では、deploy-time attributes を使用して、画像アセットの場所を環境変数として Lambda 関数に渡しています。(ファイルの種類は関係ありません。ここで使用されている PNG 画像は単なる例です)。

```ts
import { Asset } from "aws-cdk-lib/aws-s3-assets";
import * as path from "path";

const imageAsset = new Asset(this, "SampleAsset", {
  path: path.join(__dirname, "images/my-image.png"),
});

new lambda.Function(this, "myLambdaFunction", {
  code: lambda.Code.asset(path.join(__dirname, "handler")),
  runtime: lambda.Runtime.PYTHON_3_6,
  handler: "index.lambda_handler",
  environment: {
    S3_BUCKET_NAME: imageAsset.s3BucketName,
    S3_OBJECT_KEY: imageAsset.s3ObjectKey,
    S3_URL: imageAsset.s3Url,
  },
});
```

#### Permissions

[aws-s3-assets](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3_assets-readme.html) モジュール、IAM ロール、ユーザー、またはグループを通じて Amazon S3 アセットを直接使用し、実行時にアセットを読み取る必要がある場合は、[asset.grantRead](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3_assets.Asset.html#grantwbrreadgrantee) メソッドを通じてそれらのアセットに IAM パーミッションを付与してください。

次の例では、IAM グループにファイルアセットの読み取り権限を付与しています。

```ts
import { Asset } from "aws-cdk-lib/aws-s3-assets";
import * as path from "path";

const asset = new Asset(this, "MyFile", {
  path: path.join(__dirname, "my-image.png"),
});

const group = new iam.Group(this, "MyUserGroup");
asset.grantRead(group);
```

#### Docker image assets

AWS CDK は [aws-ecr-assets](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecr_assets-readme.html) モジュールにより、ローカルの Docker イメージをアセットとしてバンドルすることをサポートしています。

次の例では、ローカルでビルドして Amazon ECR にプッシュする Docker イメージを定義しています。イメージはローカルの Docker コンテキストディレクトリから（Dockerfile で）ビルドされ、AWS CDK CLI またはアプリの CI/CD パイプラインによって Amazon ECR にアップロードされ、AWS CDK アプリで自然に参照することができます。

```ts
import { DockerImageAsset } from "aws-cdk-lib/aws-ecr-assets";

const asset = new DockerImageAsset(this, "MyBuildImage", {
  directory: path.join(__dirname, "my-image"),
});
```

`my-image` ディレクトリには、Dockerfile を含める必要があります。AWS CDK CLI は `my-image` から Docker イメージを構築し、Amazon ECR リポジトリにプッシュし、スタックに AWS CloudFormation パラメータとしてリポジトリの名前を指定します。Docker イメージのアセットタイプは、AWS CDK のライブラリやアプリで参照できる[deploy-time attributes](./06-concepts-resources#resource_attributes)を公開します。AWS CDK CLI コマンドの `cdk synth` は、アセットプロパティを AWS CloudFormation のパラメータとして表示します。

#### Amazon ECS task definition example

一般的なユースケースは、Docker コンテナを実行するために Amazon ECS [TaskDefinition](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs.TaskDefinition.html) を作成することです。次の例は、AWS CDK がローカルでビルドし、Amazon ECR にプッシュする Docker イメージアセットの場所を指定します。

```ts
import * as ecs from "aws-cdk-lib/aws-ecs";
import * as path from "path";

const taskDefinition = new ecs.FargateTaskDefinition(this, "TaskDef", {
  memoryLimitMiB: 1024,
  cpu: 512,
});

taskDefinition.addContainer("my-other-container", {
  image: ecs.ContainerImage.fromAsset(path.join(__dirname, "..", "demo-image")),
});
```

#### Deploy-time attributes example

次の例は、deploy-time attributes の `repository` と `imageUri` を使用して、AWS Fargate 起動タイプで Amazon ECS タスク定義を作成する方法を示しています。Amazon ECR のレポ検索は、URI ではなくイメージのタグを必要とするので、アセットの URI の末尾からそれを切り取ることに注意してください。

```ts
import * as ecs from "aws-cdk-lib/aws-ecs";
import * as path from "path";
import { DockerImageAsset } from "aws-cdk-lib/aws-ecr-assets";

const asset = new DockerImageAsset(this, "my-image", {
  directory: path.join(__dirname, "..", "demo-image"),
});

const taskDefinition = new ecs.FargateTaskDefinition(this, "TaskDef", {
  memoryLimitMiB: 1024,
  cpu: 512,
});

taskDefinition.addContainer("my-other-container", {
  image: ecs.ContainerImage.fromEcrRepository(
    asset.repository,
    asset.imageUri.split(":").pop()
  ),
});
```

#### Build arguments example

AWS CDK CLI がデプロイ時にイメージを構築する際、`buildArgs` プロパティオプションを通じて、Docker 構築ステップにカスタマイズした構築引数を提供することができます。

```ts
const asset = new DockerImageAsset(this, "MyBuildImage", {
  directory: path.join(__dirname, "my-image"),
  buildArgs: {
    HTTP_PROXY: "http://10.20.30.2:1234",
  },
});
```

#### Permissions

[aws-ecs](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs-readme.html) などの Docker イメージアセットをサポートするモジュールを利用する場合、アセットを直接利用する場合と [`ContainerImage.fromEcrRepository`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecs.ContainerImage.html#static-fromwbrecrwbrrepositoryrepository-tag) を通じて利用する場合の権限を AWS CDK が管理します。Docker イメージアセットを直接使用する場合、消費するプリンシパルがイメージを引き出すための権限を持っていることを確認する必要があります。

多くの場合、[`asset.repository.grantPull`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecr.Repository.html#grantwbrpullgrantee) メソッドを使用する必要があります。これは、プリンシパルの IAM ポリシーを変更し、このリポジトリからイメージを引き出せるようにします。イメージを pull するプリンシパルが同じアカウントでない場合や、AWS CodeBuild などのアカウントでロールを想定していない AWS サービスの場合は、プリンシパルのポリシーではなく、リソースポリシーで pull 権限を付与しなければなりません。[`asset.repository.addToResourcePolicy`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ecr.Repository.html#addwbrtowbrresourcewbrpolicystatement) メソッドを使用して、適切なプリンシパルパーミッションを付与してください。

### AWS CloudFormation resource metadata

:::message
**Note**
このセクションは、コンストラクトの作成者にのみ関係します。特定の状況では、特定の CFN リソースがローカルアセットを使用していることをツールが知る必要があります。例えば、AWS SAM CLI を使用して、デバッグのためにローカルで Lambda 関数を呼び出すことができます。詳しくは [SAM CLI](https://docs.aws.amazon.com/cdk/v2/guide/sam.html) を参照してください。
:::

このようなユースケースを可能にするために、外部ツールは AWS CloudFormation リソースのメタデータエントリのセットを参照する。

- `aws:asset:path` - アセットのローカルパスを指定する。

- `aws:asset:property` - アセットが使用されるリソースプロパティの名前。

これらの 2 つのメタデータエントリを使用すると、ツールはアセットが特定のリソースで使用されていることを識別し、高度なローカルエクスペリエンスを有効にすることができます。

これらのメタデータエントリをリソースに追加するには、`asset.addResourceMetadata` メソッドを使用します。
