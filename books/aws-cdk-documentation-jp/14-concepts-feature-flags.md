---
title: "Concepts - Falure Flags"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/featureflags.html)

## 著者の感想

今から CDK を使い始める場合、v2 を使うことになると思うので、知らなくても大丈夫と思います。

## 本文

AWS CDK は、リリースにおいて壊れる可能性のある動作を有効にするために、Falure Flags を使用します。フラグは[Runtime context](./13-concepts-context)の値として`cdk.json`（または`~/.cdk.json`）に格納されます。`cdk context --reset`や`cdk context --clear`コマンドで削除されることはありません。

Falure Flags はデフォルトで無効化されるため、フラグを指定しない既存のプロジェクトは、それ以降の AWS CDK リリースでも期待通りに動作し続けます。`cdk init` を使用して作成された新しいプロジェクトは、プロジェクトを作成したリリースで利用可能なすべての機能を有効にするフラグを含んでいます。AWS CDK のアップグレード後に、古い動作を好むフラグを無効にしたり、新しい動作を有効にするフラグを追加するために、`cdk.json` を編集してください。

:::message
**Note**
現在、CDK v2 には新しい動作を可能にするための機能フラグがありません。
:::

CDK v2 では、特定の動作を v1 のデフォルトに戻すために、機能フラグも使用されます。以下に示すフラグを false に設定すると、特定の v1 AWS CDK v1 の動作に戻されます。`cdk diff` コマンドを使用して、合成したテンプレートの変更を確認し、これらのフラグが必要であるかどうかを確認します。

- `@aws-cdk/aws-apigateway:usagePlanKeyOrderInsensitiveId`

  - アプリケーションが複数の Amazon API Gateway API キーを使用し、それらを利用プランに関連付ける場合

- `@aws-cdk/aws-rds:lowercaseDbIdentifier`

  - アプリケーションが Amazon RDS データベースインスタンスまたはデータベースクラスタを使用し、これらの識別子を明示的に指定する場合。

- `@aws-cdk/aws-cloudfront:defaultSecurityPolicyTLSv1.2_2021`

  - Amazon CloudFront 配信で TLS_V1_2_2019 のセキュリティポリシーを使用するアプリケーションの場合。CDK v2 は、デフォルトでセキュリティポリシー TLSv1.2_2021 を使用します。

- `@aws-cdk/core:stackRelativeExports`
  - アプリケーションが複数のスタックを使用し、あるスタックのリソースを別のスタックで参照する場合、AWS CloudFormation のエクスポートを構築するために絶対パスと相対パスのどちらを使用するかを決定する。

`cdk.json`でこれらのフラグを元に戻すための構文は以下のとおりです。

```json
{
  "context": {
    "@aws-cdk/aws-apigateway:usagePlanKeyOrderInsensitiveId": false,
    "@aws-cdk/aws-cloudfront:defaultSecurityPolicyTLSv1.2_2021": false,
    "@aws-cdk/aws-rds:lowercaseDbIdentifier": false,
    "@aws-cdk/core:stackRelativeExports": false
  }
}
```
