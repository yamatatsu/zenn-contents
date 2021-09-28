---
title: "@aws-cdk/aws-iot の L2 の設計を考えてみる"
emoji: "📖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws, awscdk, awsiot]
published: false
---

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
  IAction <|.. CloudwatchAlarmAction
  IAction <|.. CloudwatchLogsAction
  IAction <|.. CloudwatchMetricAction
  IAction <|.. DynamoDBAction
  IAction <|.. DynamoDBv2Action
  IAction <|.. ElasticsearchAction
  IAction <|.. FirehoseAction
  IAction <|.. HttpAction
  IAction <|.. IotAnalyticsAction
  IAction <|.. IotEventsAction
  IAction <|.. IotSiteWiseAction
  IAction <|.. KafkaAction
  IAction <|.. KinesisAction
  IAction <|.. LambdaAction
  IAction <|.. RepublishAction
  IAction <|.. S3Action
  IAction <|.. SnsAction
  IAction <|.. SqsAction
  IAction <|.. StepFunctionsAction
  IAction <|.. TimestreamAction

  class IAction{
    bind() ActionConfig
  }
  <<Interface>> IAction

```

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
