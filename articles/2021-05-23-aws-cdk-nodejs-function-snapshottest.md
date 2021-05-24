---
title: "[AWS-CDK] NodejsFunction の snapshot test がCI環境で落ちる"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["awscdk", "docker", "lambda"]
published: false
---

# 課題

aws-cdk の `NodejsFunction` の snapshot test が、GitHub Actions 上でコケる。

# tl;dr

解決方法は以下の通り。

- package.json の dependencies に `esbuild` を入れる
- props の `bundling.forceDockerBundling` を `false` にする

```js
bundling: {
  forceDockerBundling: false,
},
```

# 原因

aws-cdk の `NodejsFunction` では、気を使っていないと docker を用いてビルドが行われる。(以下コード)
https://github.com/aws/aws-cdk/blob/4c8e938e01b87636390a4f04de63bcd4dfe44cf8/packages/@aws-cdk/aws-lambda-nodejs/lib/bundling.ts#L82-L92

この docker image の tag を決めているのが以下のコードであるが、
https://github.com/aws/aws-cdk/blob/4c8e938e01b87636390a4f04de63bcd4dfe44cf8/packages/@aws-cdk/core/lib/bundling.ts#L257
hash 計算が `node_modules/aws-cdk-lib/lib/aws-lambda-nodejs/lib` の絶対パスに依存するため、実行環境が変わると hash がずれる。

この結果、ここ
https://github.com/aws/aws-cdk/blob/4c8e938e01b87636390a4f04de63bcd4dfe44cf8/packages/@aws-cdk/core/lib/asset-staging.ts#L514
で計算される fingerprint が変わってしまい、bundle 後の zip ファイル名が変わるため、snapshot test の結果が変わってしまう。

# 解決策

package.json の dependencies に `esbuild` を入れ、かつ `NodejsFunction` の props の `bundling.forceDockerBundling` を `false` にする。

こうすることでビルドに docker が使われなくなるため、CI 環境上でも結果がズレなくなる。

# 調査の背景

aws-cdk で typescript の lambda をデプロイするのはめっちゃ簡単。 `NodejsFunction` を使うだけ。
`NodejsFunction` は中で `esbuild` を使っていて、compile, bundle, deploy まで全部やってくれる。尊い。

個人的には aws-cdk のテストは snapshot test を必ず入れるようにしている。
これがあると aws-cdk のアップデートも怖くない。尊い。
snapshot test があれば [v2 へのアップデート](https://docs.aws.amazon.com/cdk/latest/guide/work-with-cdk-v2.html)も怖くない。尊い。

aws-cdk のアウトプット(CloudFormation テンプレート)は hash 値が使われている場合がある。(lambda や s3-deploy など)
このような場合では入力が一定になるようにテスト用のダミーファイルをインプットにする必要があるが、今回はなかなか hash を固定できなかったため、調べた。
