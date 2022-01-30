---
title: "Concepts - Environments"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/environments.html)

## tl;dt

デプロイ先の AWS アカウントやリージョンを切り替える方法や、それぞれの方法のメリットが説明されています。
最後に TS における個人的な考えを書き足しました。

## 本文

app の各 Stack インスタンスは、明示的または暗黙的に環境（`env`）に関連付けられています。環境は、スタックがデプロイされる予定のターゲット AWS アカウントとリージョンです。

:::message
**Note**
最も単純な展開を除くすべての場合、展開先の各環境をブートストラップする必要があります。デプロイには特定の AWS リソースが利用可能である必要があり、これらのリソースは[bootstrapping](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html)によってプロビジョニングされます。
:::

Stack をインスタンス化するときに環境を指定しない場合、Stack は _environment-agnostic（環境非依存）_ と言われます。このような environment-agnostic Stack から合成された AWS CloudFormation テンプレートは、`stack.account`, `stack.region`, `stack.availabilityZones` などの環境関連の属性について、デプロイ時の解決しようとします。

:::message
**Tip**
標準的な AWS CDK 開発テンプレートを使用している場合、Stack は App オブジェクトをインスタンス化するのと同じファイルにインスタンス化されます。

プロジェクトの bin フォルダにある、プロジェクトの名前を冠したファイル（例：hello-cdk.ts）。
:::

Environment-agnostic Stack では、アベイラビリティゾーンを使用するすべての Construct には 2 つのゾーンが割り当てられ、Stack を任意のリージョンにデプロイすることが可能になります。

`cdk deploy` を使用して環境を問わない Stack をデプロイする場合、AWS CDK CLI は指定された AWS CLI プロファイル（または何も指定されていない場合はデフォルトプロファイル）を使用して、デプロイ先を決定する。AWS CDK CLI は、AWS CLI と同様のプロトコルに従って、AWS アカウントで操作を実行するときに使用する AWS credentials を決定します。詳細は [AWS CDK Toolkit (cdk command)](https://docs.aws.amazon.com/cdk/v2/guide/cli.html)を参照してください。

本番用 Stack では、env プロパティを使用してアプリ内の各 Stack の環境を明示的に指定することをお勧めします。次の例では、その 2 つの異なる Stack に対して異なる環境を指定しています。

```ts
const envEU = { account: "2383838383", region: "eu-west-1" };
const envUSA = { account: "8373873873", region: "us-west-2" };

new MyFirstStack(app, "first-stack-us", { env: envUSA });
new MyFirstStack(app, "first-stack-eu", { env: envEU });
```

上記のようにターゲットアカウントとリージョンをハードコードした場合、Stack は常にその特定のアカウントとリージョンにデプロイされます。Stack を別のターゲットにデプロイできるようにし、合成時にターゲットを決定するためには、AWS CDK CLI が提供する 2 つの環境変数、 `CDK_DEFAULT_ACCOUNT` と `CDK_DEFAULT_REGION` を使用します。これらの変数は、`--profile` オプションで指定した AWS プロファイル、または指定しない場合はデフォルトの AWS プロファイルに基づいて設定されます。

以下のコードでは、AWS CDK CLI から渡されたアカウントとリージョンに Stack でアクセスする方法を示しています。Node.js の `process` オブジェクトを介して環境変数にアクセスします。

```ts
new MyDevStack(app, "dev", {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
});
```

:::message
**Note**
TypeScript で`process`使用するには DefinitelyTyped モジュールが必要です。

```
npm install @types/node
```

:::

CDK は `CDK_DEFAULT_ACCOUNT` 、 `CDK_DEFAULT_REGION`が使われていない場合と使われている場合を区別します。前者では合成結果として environment-agnostic テンプレートを出力すべきです。このような Stack の中で定義された Construct はデプロイ先の環境に関する情報の一切を使用することができません。例えば、`if (stack.region === 'us-east-1')`や[`VPC.fromLookup`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ec2.Vpc.html#static-fromwbrlookupscope-id-options)は、AWS アカウントに対してクエリを発行する必要があるため、使えません。これらの機能は環境を明示しないと使えません。

`CDK_DEFAULT_ACCOUNT`, `CDK_DEFAULT_REGION` の環境変数を指定する場合、Stack は指定されたアカウントとリージョンにデプロイされます。これは CDK CLI によって合成時に決定されます。これにより、環境依存コードはうまく動作することができますが、合成されたテンプレートが、マシン、ユーザー、セッションによって異なる可能性があることも意味します。この動作は、開発時にはしばしば許容され、望ましいとさえ言えますが、実運用時にはおそらくアンチパターンとなるでしょう。

何かしらの式を使うことで env を如何様にでも指定することができます。たとえば、合成時にアカウントとリージョンを書き換えることで、2 つ環境変数するように書けます。書き換えるアカウントとリージョンの環境変数を、`CDK_DEPLOY_ACCOUNT`, `CDK_DEPLOY_REGION`と呼ぶことにしましょう。この名前はどのような名前でも問題ありません。以下の例では、設定したら優先して使用される環境変数とフォールバック先のデフォルト環境変数を Stack に与えています。

```ts
new MyDevStack(app, "dev", {
  env: {
    account: process.env.CDK_DEPLOY_ACCOUNT || process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEPLOY_REGION || process.env.CDK_DEFAULT_REGION,
  },
});
```

このように Stack の環境が宣言されていれば、次のような短いスクリプトやバッチファイルを書いて、コマンドライン引数から変数を設定し、`cdk deploy` を呼び出すことができます。最初の 2 つ以外の引数は、`cdk deploy` に渡され、コマンドラインオプションや Stack を指定するために使用されます。

```bash
#!/usr/bin/env bash
if [[ $# -ge 2 ]]; then
    export CDK_DEPLOY_ACCOUNT=$1
    export CDK_DEPLOY_REGION=$2
    shift; shift
    npx cdk deploy "$@"
    exit $?
else
    echo 1>&2 "Provide account and region as first two args."
    echo 1>&2 "Additional args are passed through to cdk deploy."
    exit 1
fi
```

スクリプトを `cdk-deploy-to.sh` という名前で保存し、`chmod +x cdk-deploy-to.sh` を実行して実行可能な状態にします。

あとは，この deploy-to スクリプトを呼び出すスクリプトを追加で書けば，特定の環境にデプロイできます（スクリプトごとに複数の環境でも可）．

```bash
#!/usr/bin/env bash
# cdk-deploy-to-test.sh
./cdk-deploy-to.sh 123457689 us-east-1 "$@"
```

複数の環境にデプロイする場合、デプロイに失敗した後に他の環境へのデプロイを継続するかどうかを検討します。次の例では、最初のデプロイが成功しなかった場合、2 番目の本番環境へのデプロイを回避しています。

```bash
#!/usr/bin/env bash
# cdk-deploy-to-prod.sh
./cdk-deploy-to.sh 135792468 us-west-1 "$@" || exit
./cdk-deploy-to.sh 246813579 eu-west-1 "$@"
```

開発者は、通常の cdk deploy コマンドを使用して、開発用の自分の AWS 環境にデプロイすることも可能です。

## ヤマタツ追記

秘匿情報でない限り、TS を使う場合には環境値も TS 内で定義するほうが一貫性があって扱いやすいと、個人的には思っています。その場合、デプロイ先の環境名をキーにして環境固有値を引けるようにすると扱いやすいと思います。

```ts
const ENV_NAMES = ["dev", "stg", "prd"] as const;
type EnvName = typeof ENV_NAMES[number];
type EnvValues = {
  envName: string;
  domainName: string;
  externalServiceEndpoint: string;
};

export function getEnv(): EnvValues {
  const envName = (process.env.ENV_NAME || "dev") as EnvName;
  switch (envName) {
    case "dev":
      return {
        envName,
        domainName: "dev.example.com",
        externalServiceEndpoint: "https://dev.external.example.com",
      };
    case "stg":
      return {
        envName,
        domainName: "stg.example.com",
        externalServiceEndpoint: "https://stg.external.example.com",
      };
    case "prd":
      return {
        envName,
        domainName: "example.com",
        externalServiceEndpoint: "https://external.example.com",
      };
    default:
      const _: never = envName;
      throw new Error(
        `Invalid environment variable ENV_NAME has given. ${envName}`
      );
  }
}
```

キーとなる環境名や秘匿情報は環境変数で渡します。

```sh
ENV_NAME=prd cdk deploy
```

[CDK の context](https://docs.aws.amazon.com/cdk/v2/guide/context.html) を使って渡してもいいと思います。

```sh
cdk deploy -c ENV_NAME=prd
```

でも、多言語ならいざしらず、使い勝手が同じなら、Node.js が純粋にできること（環境変数の取得）をわざわざ CDK 独自機能で解決する必要はない、というのが個人的な考え方ではあります。
