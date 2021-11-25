---
title: "@aws-cdk/aws-iot-events の L2 の設計を考えてみる"
emoji: "📖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws, awscdk, awsiotevents]
published: true
---

考え中の公開ノート。

## Input

reference: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iotevents-input.html

```mermaid
classDiagram
  class Input {
    inputName?: string;
    description?: string;
    attributeJsonPaths: string[];
  }
```

## DetectorModel

reference: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iotevents-detectormodel.html

```mermaid
classDiagram
  DetectorModel --> EvaluationMethod
  DetectorModel --> State
  State --> OnEnter
  State --> OnInput
  State --> OnExit
  OnEnter --> Event
  OnInput --> Event
  OnInput --> TransitionEvent
  OnExit --> Event

  class DetectorModel {
    detectorModelName?: string;
    description?: string;
    key?: string;
    evaluationMethod?: EvaluationMethod;
    role?: IRole;
    initialState: State;
    states: State[];
  }

  class EvaluationMethod{
    BATCH
    SERIAL
  }
  <<enumeration>> EvaluationMethod

  class State {
    stateName: string;
    onEnter?: OnEnter;
    onInput?: OnInput;
    onExit?: OnExit;
  }
  class OnEnter {
    events?: Event[];
  }
  class OnInput {
    events?: Event[];
    transitionEvents?: TransitionEvent[];
  }
  class OnExit {
    events?: Event[];
  }
  class Event {
    eventName: string;
    condition?: string;
    actions?: IAction[];
  }
  class TransitionEvent {
    eventName: string;
    condition: string;
    actions?: IAction[];
    nextState: string;
  }
```
