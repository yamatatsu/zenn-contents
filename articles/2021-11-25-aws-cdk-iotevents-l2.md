---
title: "@aws-cdk/aws-iotevents の L2 の設計を考えてみる"
emoji: "📖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws, awscdk, awsiotevents]
published: true
---

考え中の公開ノート。

## Design

### Input

reference: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iotevents-input.html

```mermaid
classDiagram
  class Input {
    inputName?: string;
    description?: string;
    attributeJsonPaths: string[];
  }
```

### DetectorModel

reference: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iotevents-detectormodel.html

DetectorModel:

```mermaid
classDiagram
  DetectorModel --> DetectorModelProps
  DetectorModelProps ..> EvaluationMethod
  DetectorModelProps ..> Definition

  class DetectorModel {
    +constructor(props: DetectorModelProps)
  }
  class DetectorModelProps {
    detectorModelName?: string;
    description?: string;
    key?: string;
    evaluationMethod?: EvaluationMethod;
    role?: IRole;
    definition: Definition;
  }
  <<Interface>> DetectorModelProps

  class Definition {
  }

  class EvaluationMethod{
    BATCH
    SERIAL
  }
  <<enumeration>> EvaluationMethod

```

Definition:

```mermaid
classDiagram
  Definition ..> State
  State ..> StateProps
  State ..> TransitionEvent
  State ..|> IState
  StateProps ..> Event
  TransitionEvent ..> IState
  Event ..|> IEvent
  TransitionEvent ..|> IEvent

  class Definition {
    -initialState: State
    -states: Set<State>
    +constructor(startState: State)
    +toCfn()
  }

  class State {
    +constructor(props: StateProps)
    +addTransition(transitionEvent: TransitionEvent)
    +getGraphStates(stateSet?: Set<State>)
  }
  class StateProps {
    stateName: string;
    onEnterEvents?: Event[];
    onInputEvents?: Event[];
    onExitEvents?: Event[];
  }
  <<Interface>> StateProps
  class Event {
    condition?: string;
  }
  <<Interface>> Event
  class TransitionEvent {
    condition: string;
    nextState: IState;
  }
  <<Interface>> TransitionEvent
  class IState {
    stateName: string;
    transitionEvents: TransitionEvent[];
  }
  <<Interface>> IState
  class IEvent {
    eventName: string;
    actions?: IAction[];
  }
  <<Interface>> IEvent
```

usage:

```ts
const timer = new Timer('heartbeatTimer', 60)

// Define nodes of the state machine
const onlineState = new iotevents.State({
  stateName: 'online',
  onEnterEvents: [{
    eventName: 'setTimer',
    actions: [timer.set()], // `timer.set()` return `SetTimerAction`
  }],
  onInputEvents: [{
    eventName: 'resetTimer',
    condition: 'currentInput("HeartbeatInputData")',
    actions: [timer.reset()], // `timer.reset()` return `ResetTimerAction`
  }],
});
const offlineState = new iotevents.State({
  stateName: 'offline',
});

// Define edges of the state machine
onlineState.addTransition({
  eventName: 'to_offline',
  condition: timer.timeout(), // `timer.timeout()` return just string
  nextState: offlineState,
})
offlineState.addTransition({
  eventName: 'to_online',
  condition: 'currentInput("HeartbeatInputData")',
  nextState: onlineState,
})

// Define the state machine
new iotevents.DetectorModel(this, 'DetectorModel', {
  definition: new iotevents.Definition(onlineState);
})
```

Definition:

```ts
class Definition {
  public definition: CfnDetectorModel.DetectorModelDefinitionProperty;

  constructor(private readonly initialState: State) {}

  toCfn() {
    const stateSet = initialState.getGraphStates();

    return {
      initialState: initialState.stateName,
      states: Array.from(stateSet).map((state) => state.toCfn()),
    };
  }
}
```

State:

```ts
class State {
  constructor(private readonly props: StateProps) {}

  /**
   * get states recursively
   */
  getGraphStates(stateSet: Set<State> = new Set<State>()): Set<State> {
    if (stateSet.has(this)) {
      return stateSet;
    }
    stateSet.add(this);

    initialState.transitionEvents.forEach((te) => {
      te.nextState.getGraphStates(stateSet);
    });

    return stateSet;
  }
}
```

## Roadmap

1. implement `DetectorModel`, `Definition` and `State` with only required properties
   - It will not be able to have multiple states yet.
2. implement `state.addTransition()`
   - It will be able to have multiple states and to transit.
   - It will not be able to have events that is without transition.
   - It will not be able to perform actions.
3. implement `IAction` and a action `SetTimer`
   - Note: AWS IoT Events has many actions like AWS IoT Core. https://docs.aws.amazon.com/iotevents/latest/apireference/API_Action.html
4. implement some actions (separate PRs for each actions)
   - It will be able to perform actions.
5. implement `Event` and use it in `State`
   - It will be able to have events that is without transition.
6. implement remaining actions (separate PRs for each actions)
