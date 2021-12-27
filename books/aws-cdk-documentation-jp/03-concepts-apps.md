---
title: "Concepts - Apps"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/apps.html)

## tl;dt

CDK では `const app = new App();` から書き始めて、「CDK でプロビジョニングするアプリケーションたち」をこの`app`に関連付けていき、そしてこのコードを`cdk`コマンドで実行することでデプロイします。この章では、この動作の全体像を説明しています。

[App lifecycle](#app-lifecycle)と[Cloud assemblies](#cloud-assemblies)の内容は把握していなくても CDK を利用することができるので読み飛ばしても大丈夫だと思います。

## 本文

[Construct](./02-concepts-constructs) で説明されているように、インフラストラクチャリソースをプロビジョニングするには、AWS リソースを表すすべての Construct を、直接または間接的に、Stack の scope 内で定義する必要があります。

次の例 `MyFirstStack` では、単一の S3 バケットを含むという名前の Stack クラスを宣言しています。ただし、これは Stack を宣言するだけです。それをデプロイするには、scope 内で定義（インスタンス化とも呼ばれます）する必要があります。

```ts
class MyFirstStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new s3.Bucket(this, "MyFirstBucket");
  }
}
```

### The app construct

To define the previous stack within the scope of an application, use the App construct. The following example app instantiates a MyFirstStack and produces the AWS CloudFormation template that the stack defined.

アプリケーションの scope 内で前の Stack を定義するには、[App](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.App.html) を使用します。次のサンプルアプリは、を`MyFirstStack`をインスタンス化し、Stack の定義に従って AWSCloudFormation テンプレートを生成します。

```ts
const app = new App();
new MyFirstStack(app, "hello-cdk");
app.synth();
```

`App`は construct tree のルートとして使用することができる唯一の Construct だから、App は、任意の初期化引数を必要としません。App は、Stack インスタンスを定義するための Scope として使用できます。

次のように、App から派生したクラス内で構成を定義することもできます。

```ts
class MyApp extends App {
  constructor() {
    new MyFirstStack(this, "hello-cdk");
  }
}

new MyApp().synth();
```

これらの 2 つの方法は同等です。

### App lifecycle

次の図は、`cd kdeploy`を呼び出したときに AWS CDK が通過するフェーズを示しています 。このコマンドは、アプリが定義するリソースをデプロイします。

![cdkライフサイクル](/images/book-aws-cdk-documentation-jp-lifecycle.png)
_cdk ライフサイクル_

app は、次のライフサイクルフェーズを通過します。

1. Construction (or Initialization) 構築（または初期化）
   定義されたすべての Construct をインスタンス化してから、それらをリンクします。この段階では、すべての Construct（App、Stack、およびそれらの子 Construct）がインスタンス化され、constructor chain が実行されます。ほとんどのアプリコードはこの段階で実行されます。
2. Preparation 準備
   `prepare` メソッドを実装した Construct は、このフェーズに参加します。このフェーズではインスタンスに対して、最終ラウンドの設定が行われます。準備フェーズは自動的に行われます。ユーザーとして、このフェーズからのフィードバックは表示されません。「準備」フックを使用する必要があることはまれであり、通常はお勧めしません。操作の順序が動作に影響を与える可能性があるため、このフェーズで Construct tree を変更する場合は、十分に注意する必要があります。
3. Validation 検証
   `validate` メソッドを実装した Construct このフェーズに参加します。Construct 自身を検証して、正しくデプロイされる状態にあるか確認できます。このフェーズで発生した検証の失敗が通知されます。一般に、検証をできるだけ早く（通常は入力を取得したらすぐに）実行し、できるだけ早く例外をスローすることをお勧めします。検証を早期に実行すると、スタックトレースがより正確になり、コードが安全に実行され続けることができるため、診断の可能性が向上します。要は constructer chain で検証できることは、この`prepare`メソッドに頼らずに constructer chain で検証するべき、ということです。
4. Synthesis 合成
   これは、app の実行の最終段階です。`app.synth()`の呼び出しによってトリガーされ、Construct tree をトラバースして、すべての Construct について`synthesize`メソッドを呼び出します。`synthesize`を実装する Construct は 、Synthesis に参加し、結果のクラウドアセンブリにデプロイアーティファクトを発行できます。これらのアーティファクトには、AWS CloudFormation テンプレート、AWS Lambda アプリケーションバンドル、ファイルと Docker イメージアセット、およびその他のデプロイアーティファクトが含まれます。 クラウドアセンブリは、このフェーズの出力を記述します。ほとんどの場合、synthesize メソッド を実装する必要はありません
5. Deployment 展開
   このフェーズでは、AWS CDK Toolkit は、Synthesis フェーズで生成されたデプロイアーティファクトクラウドアセンブリを取得し、AWS 環境にデプロイします。アセットを S3 と ECR、または必要な場所にアップロードしてから、AWS CloudFormation のデプロイを開始して、アプリケーションをデプロイし、リソースを作成します。

AWS CloudFormation のデプロイフェーズ（ステップ 5）が開始するまでに、app は完了してプロセスを閉じます。これには、次の意味があります。

- app は、リソースの作成やデプロイ全体の終了など、デプロイ中に発生するイベントに応答できません。デプロイフェーズでコードを実行するには、[custom resource](https://docs.aws.amazon.com/cdk/v2/guide/cfn_layer.html#cfn_layer_custom)として AWS CloudFormation テンプレートにコードを挿入する必要があります 。app に custom resource を追加する方法の詳細については、[AWS CloudFormation module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_cloudformation-readme.html)、または [custom-resource](https://github.com/aws-samples/aws-cdk-examples/tree/master/typescript/custom-resource/) を参照してください。

- app は、実行時に知ることができない値を扱う必要がある場合があります。例えば、app が自動生成された名前の S3 バケットを定義し、 `bucket.bucketName` 属性を取得した場合、その値は展開されたバケット名ではありません。代わりに、`Token` 値が取得されます。特定の値が利用可能かどうかを判断するには、 `cdk.isToken`(value) (Python: `is_token`) を呼び出します。詳しくは、[Tokens](./08-concepts-tokens) を参照してください。

### Cloud assemblies

`app.synth()`の呼び出しは、Cloud assemblies を合成するように、app から AWS CDK に 指示するものです。通常、Cloud assemblies を直接操作することはありません。それらは、アプリをクラウド環境にデプロイするために必要なすべてのものを含むファイルです。例えば、app の各 stack 用の AWS CloudFormation テンプレートや、app 内で参照するファイルアセットや Docker イメージのコピーなどが含まれます。

Cloud assemblies のフォーマット方法の詳細については、[Cloud assemblies の仕様](https://github.com/aws/aws-cdk/blob/master/design/cloud-assembly.md)を参照してください。

app が作成する Cloud assemblies とやり取りするには、通常、コマンドラインツールである AWS CDK Toolkit を使用します。しかし任意のツール（Cloud assemblies 形式を読み取ることができる任意のツール）を使用しても、app をデプロイできます。

The CDK Toolkit needs to know how to execute your AWS CDK app. If you created the project from a template using the cdk init command, your app's cdk.json file includes an app key that specifies the necessary command for the language the app is written in. If your language requires compilation, the command line performs this step before running the app, so you can't forget to do it.

CDK Toolkit は、app の実行方法を知っておく必要があります。`cdk init` コマンドを使用してテンプレートからプロジェクトを作成した場合、アプリの `cdk.json` ファイルには、アプリが書かれている言語に必要なコマンドを指定する `app` キーが含まれています。言語によってコンパイルが必要な場合、コマンドラインはアプリを実行する前にこのステップを実行するので、忘れずに実行することができます。

```ts
{
  "app": "npx ts-node --prefer-ts-exts bin/my-app.ts"
}
```

If you did not create your project using the CDK Toolkit, or wish to override the command line given in cdk.json, you can use the --app option when issuing the cdk command.

CDK Toolkit を使用してプロジェクトを作成しなかった場合、または`cdk.json`で指定されたコマンドラインをオーバーライドする場合は、`cdk`コマンドを発行するときに `--app` オプションを使用できます。

```sh
cdk --app 'executable' cdk-command ...
```

上記コマンドの `executable` の部分は、CDK アプリケーションを run するために execute すべきコマンドを示します。このようなコマンドにはスペースが含まれるため、表示されているようにシングルコーテーション`''`を使用します。`cdk-command` は `synth` や `deploy` などのサブコマンドで、CDK Toolkit に対して、アプリで何を行いたいかを指示します。このコマンドの後に、そのサブコマンドに必要なオプションを追加します。

CLI は、すでに合成された Cloud assemblies 形式を読み取ることができる任意のツールと直接対話することもできます。そのためには、Cloud assemblies 形式を読み取ることができる任意のツールが格納されているディレクトリを `--app` に渡します。次の例では、`./my-cloud-assembly` に保存されている Cloud assemblies 形式を読み取ることができる任意のツールで定義されている Stack をリストアップします。

```sh
cdk --app ./my-cloud-assembly ls
```
