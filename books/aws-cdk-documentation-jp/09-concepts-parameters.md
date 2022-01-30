---
title: "Concepts - Parameters"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/parameters.html)

## tl;dt

[CFn の Parameters](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html)。CFn と同じように使えるけど、環境変数やコンテキストの方を使うことが推奨されている。つまりあんまり読まなくても OK。

## 本文

AWS CloudFormation のテンプレートには、デプロイ時に渡されテンプレートに組み込まれるカスタム値である[パラメータ](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html)を含めることができます。AWS CDK は AWS CloudFormation のテンプレートを合成するので、デプロイメント時のパラメータもサポートしています。

AWS CDK を使用すると、パラメータを定義して、作成したコンストラクトのプロパティで使用することができ、また、パラメータを含むスタックをデプロイすることができます。

AWS CDK Toolkit を使用して AWS CloudFormation テンプレートをデプロイする場合、コマンドライン上でパラメータ値を渡します。AWS CloudFormation コンソールを介してテンプレートをデプロイする場合、パラメータ値の入力を求められます。

一般的に、AWS CDK で AWS CloudFormation のパラメータを使用しないことをお勧めします。ハードコーディングせずに AWS CDK アプリに値を渡す通常の方法である[コンテキスト値](./13-concepts-context)や環境変数とは異なり、パラメータ値は合成時に利用できないため、AWS CDK アプリの他の部分、特に制御フローで簡単に使用することができません。

:::message
パラメータを使った制御フローを行うには、[CfnCondition 構文](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.CfnCondition.html)を使いますが、これはネイティブの if 文と比べると厄介です。
:::

パラメータを使うには、合成時だけでなく、デプロイ時に自分の書いているコードがどのように動作するかを意識する必要があります。これは、AWS CDK アプリケーションを理解し推論することを難しくし、多くの場合、ほとんど利益をもたらしません。

一般的には、CDK アプリがユーザーから必要な情報を受け取り、CDK アプリのコンストラクトを宣言するためにそれを直接使用する方がよいでしょう。AWS CDK が生成する AWS CloudFormation のテンプレートは、デプロイ時に指定する値が残っていない具体的なものが理想的です。

しかし、AWS CloudFormation のパラメータが独自に適しているユースケースもあります。例えば、インフラを定義するチームとデプロイするチームが分かれている場合、生成されたテンプレートをより広く使えるようにするためにパラメータを使用することができます。さらに、AWS CDK の AWS CloudFormation パラメータのサポートにより、AWS CloudFormation テンプレートを使用する AWS サービス（AWS Service Catalog など）で AWS CDK を使用することができ、これらはパラメータを使用して展開されるテンプレートを構成することができます。

### Defining parameters

[`CfnParameter` クラス](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.CfnParameter.html)を使用して、パラメータを定義します。ほとんどのパラメータでは、少なくとも`type`と`description`を指定する必要がありますが、技術的にはどちらも任意です。`description`は、AWS CloudFormation コンソールでユーザーがパラメータ値を入力するプロンプトが表示されるときに表示されます。利用可能な`type`の詳細については、[Types](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html#parameters-section-structure-properties-type)を参照してください。

:::message
パラメータはどのスコープでも定義できますが、コードをリファクタリングしても論理 ID が変わらないように、スタックレベルでパラメータを定義することをお勧めします。
:::

```ts
const uploadBucketName = new CfnParameter(this, "uploadBucketName", {
  type: "String",
  description:
    "The name of the Amazon S3 bucket where uploaded files will be stored.",
});
```

### Using parameters

A CfnParameter instance exposes its value to your AWS CDK app via a token. Like all tokens, the parameter's token is resolved at synthesis time, but it resolves to a reference to the parameter defined in the AWS CloudFormation template, which will be resolved at deploy time, rather than to a concrete value.

You can retrieve the token as an instance of the Token class, or in string, string list, or numeric encoding, depending on the type of value required by the class or method you want to use the parameter with.

`CfnParameter` インスタンスは、[`token`](./08-concepts-tokens)を介して AWS CDK アプリにその値を公開します。他の`token`と同様に、パラメータの`token`は合成時に解決されますが、具体的な値ではなく、デプロイ時に解決される AWS CloudFormation テンプレートで定義されたパラメータへの参照に解決されます。

`token`は、Token クラスのインスタンスとして、またはパラメータを使用したいクラスやメソッドが必要とする値の種類に応じて、文字列、文字列リスト、数値エンコーディングで取得することができます。

| Property      | kind of value                          |
| ------------- | -------------------------------------- |
| value         | Token class instance                   |
| valueAsList   | The token represented as a string list |
| valueAsNumber | The token represented as a number      |
| valueAsString | The token represented as a string      |

例えば、Bucket の定義でパラメータを使用する場合は以下のように記述します。

```ts
const bucket = new Bucket(this, "myBucket", {
  bucketName: uploadBucketName.valueAsString,
});
```

### Deploying with parameters

パラメータを含む生成されたテンプレートは、AWS CloudFormation コンソールから通常の方法でデプロイすることができ、各パラメータの値の入力を求められます。

AWS CDK Toolkit（cdk コマンドラインツール）も、デプロイ時のパラメータ指定に対応しています。コマンドラインで `--parameters` フラグの後に、これらのパラメータを指定することができます。`uploadBucketName` パラメータを使用するスタックをデプロイするには、次のようにします。

```sh
cdk deploy MyStack --parameters uploadBucketName=UploadBucket
```

複数のパラメータを定義するには、複数の `--parameters` フラグを使用します。

```sh
cdk deploy MyStack --parameters uploadBucketName=UpBucket --parameters downloadBucketName=DownBucket
```

複数のスタックをデプロイする場合、パラメータ名の前にスタック名とコロンを付けることで、スタックごとに異なるパラメータ値を指定することができます。

```sh
cdk deploy MyStack YourStack --parameters MyStack:uploadBucketName=UploadBucket --parameters YourStack:uploadBucketName=UpBucket
```

デフォルトでは、AWS CDK は以前のデプロイメントのパラメータ値を保持し、それらが明示的に指定されていない場合、以降のデプロイメントでそれらを使用します。すべてのパラメータを指定する必要がある場合は、`--no-previous-parameters` フラグを使用します。
