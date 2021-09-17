---
title: "GitHub Actions Token ID で AWS CDK する"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws, github, githubactions]
published: true
---

こればな ↓
https://dev.classmethod.jp/articles/github-actions-without-permanent-credential/

GitHub Secrets に埋め込まれた忌まわしき Access Key を捨てれるとのことで試してみました。

できたのがこれ ↓

https://github.com/yamatatsu/github-actions-id-sample

GitHub Actions の結果がこれ ↓

https://github.com/yamatatsu/github-actions-id-sample/runs/3628343114?check_suite_focus=true

# 動機

CDK 使ってると
「これで lambda デプロイできるやん」
「これで s3 デプロイできるやん」
「これで ECR デプロイできるやん」
ってなると思います。

こうなってくると GitHub Actions で動かしたくてたまらなくなります。

# やりかた

## AWS 編

[雑に CDK で書きました。](https://github.com/yamatatsu/github-actions-id-sample/blob/main/index.ts)

やってることは[元記事の CFn](https://dev.classmethod.jp/articles/github-actions-without-permanent-credential/#toc-3)と基本的に変わらないです。

## GitHub Actions 編

こんな感じで動くと思います。

```yaml
jobs:
  invoke:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - run: sleep 5 # there's still a race condition for now

      - name: Configure AWS
        run: |
          export AWS_ROLE_ARN=arn:aws:iam::407172421073:role/ExampleGithubRole
          export AWS_WEB_IDENTITY_TOKEN_FILE=/tmp/awscreds
          export AWS_REGION=ap-northeast-1

          echo AWS_WEB_IDENTITY_TOKEN_FILE=$AWS_WEB_IDENTITY_TOKEN_FILE >> $GITHUB_ENV
          echo AWS_ROLE_ARN=$AWS_ROLE_ARN >> $GITHUB_ENV
          echo AWS_REGION=$AWS_REGION >> $GITHUB_ENV

          curl -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sigstore" | jq -r '.value' > $AWS_WEB_IDENTITY_TOKEN_FILE

      - uses: actions/checkout@v2
      - run: yarn
      - run: yarn cdk deploy
```

[元記事の yaml](https://dev.classmethod.jp/articles/github-actions-without-permanent-credential/#toc-4)と違うのは、 `AWS_DEFAULT_REGION` ではなく `AWS_REGION` を使ってます。
aws-cdk の中では AWS SDK が使われててるので、aws cli とはちょっとだけ使ってる環境変数が違うみたい。（じつはめっちゃハマった）

# まとめ

[CodeCov のインシデントによって、GitHub Actions 上の環境変数からさまざまな認証情報や個人情報が流出した事件](https://about.codecov.io/security-update/)は記憶に新しいと思います。

このような悲劇を繰り返さないためにも、今回の GitHub の神アプデ（まだ公式の発表はないけど）によって、永続的なクレデンシャルを駆逐していきましょう。

# 苦労話

雑に一瞬で完了すると思ったけど、CLI と SDK で region を渡す環境変数名が違うことを知らず、無限に時間を消耗した。。。

とりあえずこれが出ます。

```
Error: Need to perform AWS calls for account XXXXXXXXXXXX, but no credentials have been configured
```

`--varbose` をつけて実行すると、region が設定されていない旨がエラーで出てます。

```
Unable to determine the default AWS account: Error [ConfigError]: Missing region in config
    at Request.optInRegionalEndpoint (/home/runner/work/github-actions-id-sample/github-actions-id-sample/node_modules/aws-sdk/lib/services/sts.js:75:30)
    at Request.callListeners (/home/runner/work/github-actions-id-sample/github-actions-id-sample/node_modules/aws-sdk/lib/sequential_executor.js:106:20)
    at Request.emit (/home/runner/work/github-actions-id-sample/github-actions-id-sample/node_modules/aws-sdk/lib/sequential_executor.js:78:10)
    at Request.emit (/home/runner/work/github-actions-id-sample/github-actions-id-sample/node_modules/aws-sdk/lib/request.js:688:14)
    at Request.transition (/home/runner/work/github-actions-id-sample/github-actions-id-sample/node_modules/aws-sdk/lib/request.js:22:10)
    at AcceptorStateMachine.runTo (/home/runner/work/github-actions-id-sample/github-actions-id-sample/node_modules/aws-sdk/lib/state_machine.js:14:12)
    at Request.runTo (/home/runner/work/github-actions-id-sample/github-actions-id-sample/node_modules/aws-sdk/lib/request.js:408:15)
    at Request.send (/home/runner/work/github-actions-id-sample/github-actions-id-sample/node_modules/aws-sdk/lib/request.js:372:10)
    at features.constructor.makeUnauthenticatedRequest (/home/runner/work/github-actions-id-sample/github-actions-id-sample/node_modules/aws-sdk/lib/service.js:230:31)
    at features.constructor.assumeRoleWithWebIdentity (/home/runner/work/github-actions-id-sample/github-actions-id-sample/node_modules/aws-sdk/lib/services/sts.js:44:17) {
  code: 'ConfigError',
  time: 2021-09-16T12:02:44.650Z
}
```

おー SDK でやっとるんかー、ってなって Doc みると設定する環境変数が `AWS_REGION` であることがわかると。
https://docs.aws.amazon.com/ja_jp/sdk-for-javascript/v2/developer-guide/setting-region.html#setting-region-environment-variable

ぎゅぎゅっと要約すると上記のとおりですが、人類（おもに僕）は紆余曲折する生き物なので、もう無駄にコード読んだよ。まずドキュメント見ろよって話ですよ。あー、泣いた。
