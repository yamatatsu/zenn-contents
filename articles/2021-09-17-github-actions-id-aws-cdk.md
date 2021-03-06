---
title: "GitHub Actions Token ID ใง AWS CDK ใใ"
emoji: "๐"
type: "tech" # tech: ๆ่ก่จไบ / idea: ใขใคใใข
topics: [aws, github, githubactions]
published: true
---

ใใใฐใช โ
https://dev.classmethod.jp/articles/github-actions-without-permanent-credential/

GitHub Secrets ใซๅใ่พผใพใใๅฟใพใใใ Access Key ใๆจใฆใใใจใฎใใจใง่ฉฆใใฆใฟใพใใใ

ใงใใใฎใใใ โ

https://github.com/yamatatsu/github-actions-id-sample

GitHub Actions ใฎ็ตๆใใใ โ

https://github.com/yamatatsu/github-actions-id-sample/runs/3628343114?check_suite_focus=true

# ๅๆฉ

CDK ไฝฟใฃใฆใใจ
ใใใใง lambda ใใใญใคใงใใใใใ
ใใใใง s3 ใใใญใคใงใใใใใ
ใใใใง ECR ใใใญใคใงใใใใใ
ใฃใฆใชใใจๆใใพใใ

ใใใชใฃใฆใใใจ GitHub Actions ใงๅใใใใใฆใใพใใชใใชใใพใใ

# ใใใใ

## AWS ็ทจ

[้ใซ CDK ใงๆธใใพใใใ](https://github.com/yamatatsu/github-actions-id-sample/blob/main/index.ts)

ใใฃใฆใใใจใฏ[ๅ่จไบใฎ CFn](https://dev.classmethod.jp/articles/github-actions-without-permanent-credential/#toc-3)ใจๅบๆฌ็ใซๅคใใใชใใงใใ

## GitHub Actions ็ทจ

ใใใชๆใใงๅใใจๆใใพใใ

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

[ๅ่จไบใฎ yaml](https://dev.classmethod.jp/articles/github-actions-without-permanent-credential/#toc-4)ใจ้ใใฎใฏใ `AWS_DEFAULT_REGION` ใงใฏใชใ `AWS_REGION` ใไฝฟใฃใฆใพใใ
aws-cdk ใฎไธญใงใฏ AWS SDK ใไฝฟใใใฆใฆใใฎใงใaws cli ใจใฏใกใใฃใจใ?ใไฝฟใฃใฆใ็ฐๅขๅคๆฐใ้ใใฟใใใ๏ผใใคใฏใใฃใกใใใใฃใ๏ผ

# ใพใจใ

[CodeCov ใฎใคใณใทใใณใใซใใฃใฆใGitHub Actions ไธใฎ็ฐๅขๅคๆฐใใใใพใใพใช่ช่จผๆๅ?ฑใๅไบบๆๅ?ฑใๆตๅบใใไบไปถ](https://about.codecov.io/security-update/)ใฏ่จๆถใซๆฐใใใจๆใใพใใ

ใใฎใใใชๆฒๅใ็นฐใ่ฟใใชใใใใซใใไปๅใฎ GitHub ใฎ็ฅใขใใ๏ผใพใ?ๅฌๅผใฎ็บ่กจใฏใชใใใฉ๏ผใซใใฃใฆใๆฐธ็ถ็ใชใฏใฌใใณใทใฃใซใ้ง้ใใฆใใใพใใใใ

# ่ฆๅด่ฉฑ

้ใซไธ็ฌใงๅฎไบใใใจๆใฃใใใฉใCLI ใจ SDK ใง region ใๆธกใ็ฐๅขๅคๆฐๅใ้ใใใจใ็ฅใใใ็ก้ใซๆ้ใๆถ่ใใใใใ

ใจใใใใใใใๅบใพใใ

```
Error: Need to perform AWS calls for account XXXXXXXXXXXX, but no credentials have been configured
```

`--varbose` ใใคใใฆๅฎ่กใใใจใregion ใ่จญๅฎใใใฆใใชใๆจใใจใฉใผใงๅบใฆใพใใ

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

ใใผ SDK ใงใใฃใจใใใใผใใฃใฆใชใฃใฆ Doc ใฟใใจ่จญๅฎใใ็ฐๅขๅคๆฐใ `AWS_REGION` ใงใใใใจใใใใใจใ
https://docs.aws.amazon.com/ja_jp/sdk-for-javascript/v2/developer-guide/setting-region.html#setting-region-environment-variable

ใใใใใฃใจ่ฆ็ดใใใจไธ่จใฎใจใใใงใใใไบบ้ก๏ผใใใซๅ๏ผใฏ็ดไฝๆฒๆใใ็ใ็ฉใชใฎใงใใใ็ก้งใซใณใผใ่ชญใใ?ใใใพใใใญใฅใกใณใ่ฆใใใฃใฆ่ฉฑใงใใใใใผใๆณฃใใใ
