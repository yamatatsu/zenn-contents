---
title: "Concepts - Tokens"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/tokens.html)

## 著者の感想

CDK で`console.log()`とかするとたまに出てくる`${TOKEN[Bucket.Name.1234]}`みたいな文字列についての説明。Token に対してやってはいけないこととかが書いてある。

初見だと「？？？」ってなるので知っておいてもいいと思う。

要は「CloudFormation テンプレートを生成するときに一緒に作るから仮置きで。。。」というマーク。なので TypeScript のランタイムで処理しても`${TOKEN[Bucket.Name.1234]}`みたいなほぼ意味を持たない文字列しか取り出せない。

## 本文

Token は、アプリのライフサイクルの後の時点でのみ解決できる値を表します（[App lifecycle](./03-concepts-apps#App_lifecycle)を参照）。例えば、AWS CDK アプリで定義した Amazon S3 バケットの名前は、AWS CloudFormation のテンプレートが合成されたときにのみ割り当てられます。文字列である`bucket.bucketName`属性を出力すると、以下のような内容が含まれていることがわかります。

```
${TOKEN[Bucket.Name.1234]}
```

このようにして、AWS CDK は、構築時にはまだ知られていないが、後で利用可能になるであろう値をエンコードします。AWS CDK はこれらのプレースホルダーを Token と呼んでいます。上記の場合、文字列としてエンコードされた Token です。

以下の例のように、バケット名を環境変数として AWS Lambda 関数に指定することで、この文字列をバケット名であるかのように渡すことができます。

```ts
const bucket = new s3.Bucket(this, "MyBucket");

const fn = new lambda.Function(stack, "MyLambda", {
  // ...
  environment: {
    BUCKET_NAME: bucket.bucketName,
  },
});
```

最終的に AWS CloudFormation テンプレートが合成されると、Token は AWS CloudFormation でおなじみの `{ "Ref": "MyBucket" }` としてレンダリングされます。デプロイ時に、AWS CloudFormation はこの値を実際に作成されたバケットの名前に置き換えます。

### Tokens and token encodings

Token は [IResolvable](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.IResolvable.html) インターフェースを実装したオブジェクトで、1 つの resolve メソッドを含んでいます。AWS CDK は合成時にこのメソッドを呼び出し、AWS CloudFormation テンプレートの最終的な値を生成します。Token は合成プロセスに参加し、任意の型の任意の値を生成します。

:::message
**Note**
`IResolvable` インタフェースを直接操作することはほとんどないでしょう。Token を文字列エンコードしたものしか目にすることはないでしょう。
:::

他の関数は一般的に文字列や数値のような基本的な型の引数しか受け取りません。このような場合に Token を使用するには、[cdk.Token](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.Token.html) クラスのスタティックメソッドを使用して、Token を 3 つのタイプのいずれかにエンコードします。

- [`Token.asString`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.Token.html#static-aswbrstringvalue-options) は文字列エンコーディングを生成します（または Token オブジェクトの`.toString()`を呼び出します）。

- [`Token.asList`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.Token.html#static-aswbrlistvalue-options) はリストエンコーディングを生成します。

- [`Token.asNumber`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.Token.html#static-aswbrnumbervalue) は数値エンコーディングを生成します。

これらは、`IResolvable` である任意の値を取り、指定された型のプリミティブ値にエンコードします。

:::message alart
**Important**
前のタイプのどれかがエンコードされた Token になる可能性があるため、解析するときやその内容を読もうとするときは注意が必要です。例えば、文字列を解析して値を取り出そうとしたときに、その文字列がエンコード Token であった場合、解析は失敗します。同様に、配列の長さを問い合わせようとしたり、数値を使って計算を実行しようとしたりする場合は、まずそれらが符号化された Token でないことを確認する必要があります。
:::

ある値に未解決の Token が含まれているかどうかを確認するには、`Token.isUnresolved` メソッドを呼び出します。

次の例では、Token である可能性のある文字列の値が 10 文字以下であることを検証しています。

```ts
if (!Token.isUnresolved(name) && name.length > 10) {
  throw new Error(`Maximum length for name is 10 characters`);
}
```

`name` が Token の場合、検証は行われず、デプロイ時などライフサイクルの後段でエラーが発生する可能性があります。

:::message
**Note**
Token・エンコーディングは、型システムから逃れるために使用することができます。例えば、合成時に数値値を生成する Token を文字列エンコードすることができます。これらの関数を使用する場合、合成後にテンプレートが使用可能な状態に解決されることを保証するのは、あなたの責任です。
:::

### String-encoded tokens

文字列エンコードされた Token は以下のようになります。

```
${TOKEN[Bucket.Name.1234]}
```

これらは通常の文字列と同様に渡すことができ、次の例に示すように連結することも可能です。

```ts
const functionName = bucket.bucketName + "Function";
```

また、お使いの言語でサポートされている場合は、次の例のように文字列補間を使用することができます。

```ts
const functionName = `${bucket.bucketName}Function`;
```

その他の方法で文字列を操作することは避けてください。例えば、文字列の部分文字列を取ると、文字列 Token が壊れる可能性が高いです。

### List-encoded tokens

リストエンコードされた Token は以下のようになります。

```
["#{TOKEN[Stack.NotificationArns.1234]}"]
```

これらのリストで唯一安全なのは、他のコンストラクタに直接渡すことです。文字列リスト形式の Token は連結することができませんし、Token から要素を取り出すこともできません。これらを操作する唯一の安全な方法は、[`Fn.select`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-select.html) のような AWS CloudFormation の組み込み関数を使用することです。

### Number-encoded tokens

ナンバーエンコード Token は、次のような小さな負の浮動小数点数の集合である。

```
-1.8881545897087626e+289
```

リスト Token の場合と同様に、数値の値を変更することはできません。そうすると、数値 Token が壊れてしまう可能性が高いからです。唯一許される操作は、その値を他の構成要素に渡すことです。

### Lazy values

AWS CloudFormation のパラメータのようなデプロイ時の値に加えて、Token は合成時の遅延値を表すのにもよく使われます。これは、合成が完了する前に最終的な値が決定される値であり、値が構築される時点では決定されない。Token を使って、文字列や数値のリテラル値を他のコンストラクトに渡しますが、合成時の実際の値は、まだ行われていない計算に依存する可能性があります。

合成時の遅延値を表す Token は、Lazy クラスの静的メソッド（Lazy.string や Lazy.number など）を使って構築することができます。これらのメソッドは、produce プロパティがコンテキスト引数を受け取り、呼び出されたときに最終的な値を返す関数であるオブジェクトを受け取ります。

次の例では、作成後に容量が決定されるオートスケーリンググループを作成しています。

```ts
let actualValue: number;

new AutoScalingGroup(this, "Group", {
  desiredCapacity: Lazy.numberValue({
    produce(context) {
      return actualValue;
    },
  }),
});

// At some later point
actualValue = 10;
```

### Converting to JSON

任意のデータの JSON 文字列を生成したいが、そのデータに Token が含まれているかどうかわからない場合がある。Token を含むかどうかにかかわらず、任意のデータ構造を適切に JSON エンコードするには、次の例に示すように、stack.toJsonString メソッドを使用します。

```ts
const stack = Stack.of(this);
const str = stack.toJsonString({
  value: bucket.bucketName,
});
```
