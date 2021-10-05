---
title: "@aws-cdk/aws-iot の L2 の設計を考えてみる"
emoji: "📖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws, awscdk, awsiot]
published: true
---

考え中の公開ノート。

# 考え中

## TopicRule

方針

- Actions のあたりは EventBridge の Target と CFn の構造が似てるのでリスペクトするのが良さそう。

### TopicRule

```mermaid
classDiagram
  TopicRule ..> IAction
  IAction ..> ActionConfig

  class TopicRule {
    -Array<IAction>: actions
    +constructor(TopicRuleProps)
    +addAction(IAction) void
  }

  class IAction{
    bind() ActionConfig
  }
  <<Interface>> IAction

  class ActionConfig{
    CloudwatchAlarmActionProperty?: cloudwatchAlarm
    CloudwatchLogsActionProperty?: cloudwatchLogs
    CloudwatchMetricActionProperty?: cloudwatchMetric
    DynamoDBActionProperty?: dynamoDb
    DynamoDBv2ActionProperty?: dynamoDBv2
    ElasticsearchActionProperty?: elasticsearch
    FirehoseActionProperty?: firehose
    HttpActionProperty?: http
    IotAnalyticsActionProperty?: iotAnalytics
    IotEventsActionProperty?: iotEvents
    IotSiteWiseActionProperty?: iotSiteWise
    KafkaActionProperty?: kafka
    KinesisActionProperty?: kinesis
    LambdaActionProperty?: lambda
    RepublishActionProperty?: republish
    S3ActionProperty?: s3
    SnsActionProperty?: sns
    SqsActionProperty?: sqs
    StepFunctionsActionProperty?: stepFunctions
    TimestreamActionProperty?: timestream
  }
  <<Interface>> ActionConfig

```

### TopicRuleProps

```mermaid
classDiagram
  TopicRuleProps o.. TopicRulePayloadProperty

  class TopicRuleProps {
    string?: ruleName
    TopicRulePayloadProperty: topicRulePayload
  }
  class TopicRulePayloadProperty {
    string: sql
    Array<IAction>: actions
    string?: awsIotSqlVersion
    string?: description
    IAction?: errorAction
    boolean?: ruleDisabled
  }

```

### TopicRuleActions

package として分離している。aws-events-targets と同じイメージ。

```mermaid
classDiagram
  IAction <|.. DynamoDBAction
  IAction <|.. LambdaAction
  IAction <|.. S3Action
  IAction <|.. SnsAction
  IAction <|.. SqsAction

  class IAction{
    bind() ActionConfig
  }
  <<Interface>> IAction

```

作るべきActionクラスは以下の通り。多い。。。

- [On Going] CloudwatchAlarmAction
- [On Going] CloudwatchLogsAction
- [On Going] CloudwatchMetricAction
- [On Going] DynamoDBAction
- [On Going] DynamoDBv2Action
- [On Going] LambdaAction
- [On Going] RepublishAction
- [On Going] S3Action
- [On Going] SnsAction
- [On Going] SqsAction
- [To Be Developed] ElasticsearchAction
- [To Be Developed] FirehoseAction
- [To Be Developed] HttpAction
- [To Be Developed] IotAnalyticsAction
- [To Be Developed] IotEventsAction
- [To Be Developed] IotSiteWiseAction
- [To Be Developed] KafkaAction
- [To Be Developed] KinesisAction
- [To Be Developed] StepFunctionsAction
- [To Be Developed] TimestreamAction

# 一旦考えない

#### Destination 関連

```mermaid
classDiagram
      class TopicRuleDestination{
      }

```

#### Thing 関連

```mermaid
classDiagram
      Certificate <.. Policy
      Certificate <.. Thing
      class Certificate{
        +fromArn()$ Certificate
      }
      class Policy{
        -Array<Certificate>: principals
      }
      class Thing{
        -Array<Certificate>: principals
      }

```

#### Authorizer とか Domain 設定とか

```mermaid
classDiagram
      Authorizer <.. DomainConfiguration
      class Authorizer{
      }
      class DomainConfiguration{
      }

```

#### Device Defender 関連

```mermaid
classDiagram
      class CustomMetric{
      }
      class AccountAuditConfiguration{
      }
      class Dimension{
      }
      class MitigationAction{
      }
      class ScheduledAudit{
      }
      class SecurityProfile{
      }

```

#### Device provisioning 関連

```mermaid
classDiagram
      class ProvisioningTemplate{
      }

```

#### Fleet indexing service 関連

```mermaid
classDiagram
      class FleetMetric{
      }

```

# ナレッジ、苦しんだこと

もしかしたら人のためになるかもしれないことも書いてみる。

## packageをビルドするとき
コミットしたいpackageがあったとして、そのpackageが依存しているすべてのpackageを依存グラフに基づいてビルドしなければいけない。

以下scriptsでトポロジカルソートした順序でビルドしていってくれる。
でも全部再ビルドするから効率は良くない。。。

```
scripts/buildup
```

## ゴミビルドが残っているとき
リポジトリを久しぶりにpullしたり、別のpackageを開発した直後だったりすると、存在してはいけない`*.js`や`*.d.ts`が残っている場合がある。これらがビルドを邪魔する時がある。

以下scriptでいらん成果物を削除してくれる。
```
scripts/clean-stale-files.sh
```

## lintで怒られる。
頑張るしかない。「doc書いてよ」系は自分で作文するより公式Documentの文をオマージュする感じのほうが安全と思う。

この作業が一番ボリュームあるかもしれない。

## CodeBuildが通らない
いやこれが一番時間食った。

https://github.com/aws/aws-cdk/pull/16681#issuecomment-929944810

flakyに`Out of memory`がでる。辛い。

`Out of memory`について、未だ根本的な解決してない。

加えて、flakyな箇所の先で、`individual-packages`上で行われるテストがコケてて辛かった。再現方法がわからんnow。`scripts/transform.sh`で`individual-packages`の中身を作ってからbuildすれば良いんだと思われるがなかなかビルドが通らんnow。

更新したい。
