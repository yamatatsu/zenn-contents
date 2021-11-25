---
title: "step functionsの概念モデルをクラス図にしてみる"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws, stepfunctions]
published: true
---

step functions と仲良くなりたかったのでメモ 📝

## StateMachine

definition に大事な情報がいっぱい詰まっている。JSON 文字列としてな。

```mermaid
classDiagram
  StateMachine "1" o.. "1" LoggingConfiguration
  StateMachine "1" o.. "1" TracingConfiguration

  class StateMachine {
    stateMachineArn: string
    name: string
    status: string
    definition: string
    roleArn: string
    type: string
    creationDate: string
    loggingConfiguration: LoggingConfiguration,
    tracingConfiguration: TracingConfiguration
  }
  class LoggingConfiguration {
    level: string
    includeExecutionData: boolean
  }
  class TracingConfiguration {
    enabled: boolean
  }
```

#### definition

```json
{
  "StartAt": "MyLambdaTask",
  "States": {
    "MyLambdaTask": {
      "Next": "GreetedWorld",
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 6,
          "BackoffRate": 2
        }
      ],
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "arn:aws:lambda:ap-northeast-1:660782280015:function:Plantor-NatureRemo-MyLambdaFunction67CCA873-uQDm1aC9dykQ",
        "Payload.$": "$"
      }
    },
    "GreetedWorld": { "Type": "Succeed" }
  }
}
```

## Executions

StateMathine の実行情報を司る

```mermaid
classDiagram
  StateMachine "1" <.. "0..n" Execution
  Execution "1" o.. "1..n" Event
  Execution o.. Includable

  class StateMachine {

  }
  class Execution {
    executionArn: string
    stateMachineArn: string
    name: string
    status: string
    startDate: string
    stopDate: string
    input: string
    inputDetails: Includable,
    output: string
    outputDetails: Includable
    events: Event[]
  }
  class Event {
    ...
  }
  class Includable {
    included: boolean
  }
```

#### output

```json
{
  "ExecutedVersion": "$LATEST",
  "Payload": "Hello World!",
  "SdkHttpMetadata": {
    "AllHttpHeaders": {
      "X-Amz-Executed-Version": ["$LATEST"],
      "x-amzn-Remapped-Content-Length": ["0"],
      "Connection": ["keep-alive"],
      "x-amzn-RequestId": ["80e2c2c5-15d0-4b37-8d6b-7d1471b3d872"],
      "Content-Length": ["14"],
      "Date": ["Tue, 26 Oct 2021 00:32:12 GMT"],
      "X-Amzn-Trace-Id": ["root=1-61774c8c-080affcb2549b5b946e7aa24;sampled=0"],
      "Content-Type": ["application/json"]
    },
    "HttpHeaders": {
      "Connection": "keep-alive",
      "Content-Length": "14",
      "Content-Type": "application/json",
      "Date": "Tue, 26 Oct 2021 00:32:12 GMT",
      "X-Amz-Executed-Version": "$LATEST",
      "x-amzn-Remapped-Content-Length": "0",
      "x-amzn-RequestId": "80e2c2c5-15d0-4b37-8d6b-7d1471b3d872",
      "X-Amzn-Trace-Id": "root=1-61774c8c-080affcb2549b5b946e7aa24;sampled=0"
    },
    "HttpStatusCode": 200
  },
  "SdkResponseMetadata": {
    "RequestId": "80e2c2c5-15d0-4b37-8d6b-7d1471b3d872"
  },
  "StatusCode": 200
}
```

## Events

Execution の中で何が起こったかを示す

Event は previousEventId を持っているため、Event[] から DAG が書ける。

EventDetails たちをもっと抽象化して書けるかもだけど、まぁいいや。

```mermaid
classDiagram
  Event "1" ..> "0..1" Event
  Event "1" o.. "1" ExecutionStartedEventDetails
  Event "1" o.. "1" StateEnteredEventDetails
  Event "1" o.. "1" TaskScheduledEventDetails
  Event "1" o.. "1" TaskStartedEventDetails
  Event "1" o.. "1" TaskSucceededEventDetails
  Event "1" o.. "1" StateExitedEventDetails
  Event "1" o.. "1" ExecutionSucceededEventDetails
  ExecutionStartedEventDetails "1" o.. "1" Truncatable
  StateEnteredEventDetails "1" o.. "1" Truncatable
  TaskSucceededEventDetails "1" o.. "1" Truncatable
  StateExitedEventDetails "1" o.. "1" Truncatable
  ExecutionSucceededEventDetails "1" o.. "1" Truncatable

  class Event {
    timestamp: Date
    type: string
    id: number
    previousEventId: number
    executionStartedEventDetails?: ExecutionStartedEventDetails
    stateEnteredEventDetails?: StateEnteredEventDetails
    taskScheduledEventDetails?: TaskScheduledEventDetails
    taskStartedEventDetails?: TaskStartedEventDetails
    taskSucceededEventDetails?: TaskSucceededEventDetails
    stateExitedEventDetails?: StateExitedEventDetails
    executionSucceededEventDetails?: ExecutionSucceededEventDetails
  }
  class ExecutionStartedEventDetails {
    input: string
    inputDetails: Truncatable
    roleArn: string
  }
  class StateEnteredEventDetails {
    name: string
    input: string
    inputDetails: Truncatable
  }
  class TaskScheduledEventDetails {
    resourceType: string
    resource: string
    region: string
    parameters: string
  }
  class TaskStartedEventDetails {
    resourceType: string
    resource: string
  }
  class TaskSucceededEventDetails {
    resourceType: string
    resource: string
    output: string
    outputDetails: Truncatable
  }
  class StateExitedEventDetails {
    name: string
    output: string
    outputDetails: Truncatable
  }
  class ExecutionSucceededEventDetails {
    output: string
    outputDetails: Truncatable
  }
  class Truncatable {
    truncated: boolean
  }
```
