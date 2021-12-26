---
title: "What is the AWS CDK?"
---

[元ドキュメント](https://docs.aws.amazon.com/cdk/v2/guide/home.html)

The AWS CDK を使うと reliable, scalable, cost-effective な applications をビルドできます。

AWS CDK の利点

- 適切で安全なデフォルト値が設定されている。少ないコードで必要なリソース群を構築できる。
- パラメータ、条件、ループ、構成、継承などを使ってモデリングできる。
- インフラもアプリケーションも、一つの一緒くたに管理できる。
- インフラについて、コードレビュー、単体テスト、ソース管理ができる。
- シンプルで直感的な API で（ときには stack も跨いで）リソースを連携さることができる（参照を与えたりパーミションを与えたり）
- CFn テンプレートをインポートできる。
- CFn の機能により、AWS インフラに対して冪等なプロビジョニングを行える。
- インフラ設計パターンを簡単に共有できる。

## Why use the AWS CDK?

以下の CDK コードは、500 行を超える CFn テンプレートを生成し、50 を超えるリソースがデプロイされます。

```ts
export class MyEcsConstructStack extends Stack {
  constructor(scope: App, id: string, props?: StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, "MyVpc", {
      maxAzs: 3, // Default is all AZs in region
    });

    const cluster = new ecs.Cluster(this, "MyCluster", {
      vpc: vpc,
    });

    // Create a load-balanced Fargate service and make it public
    new ecs_patterns.ApplicationLoadBalancedFargateService(
      this,
      "MyFargateService",
      {
        cluster: cluster, // Required
        cpu: 512, // Default is 256
        desiredCount: 6, // Default is 1
        taskImageOptions: {
          image: ecs.ContainerImage.fromRegistry("amazon/amazon-ecs-sample"),
        },
        memoryLimitMiB: 2048, // Default is 512
        publicLoadBalancer: true, // Default is false
      }
    );
  }
}
```

## Developing with the AWS CDK

`yarn cdk diff`によってデプロイ済みの構成と、手元の CDK の構成との差分を表示できます。

## The Construct Programming Model

The Construct Programming Model (CPM) は、AWS CDK の背後にある概念を追加のドメインに拡張します。CPM を使用するその他のツールは次のとおりです。

- CDK for Terraform (CDKtf)
- CDK for Kubernetes (CDK8s)
- Projen, for building project configurations

Construct Hub で公開されてます。
