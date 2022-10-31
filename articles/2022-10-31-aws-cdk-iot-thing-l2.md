---
title: "@aws-cdk/aws-iot の Thing の L2 の設計を考えてみる"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [test, devops]
published: false
---

[この issue](https://github.com/aws/aws-cdk/issues/18872)を受けて。

[これ](https://zenn.dev/yamatatsu/articles/2021-09-19-aws-cdk-iot-l2)のスピンオフ的な。

# 方針

一旦 Thing 関連だけ考える。

Certificate と Policy もやりたいことあるけど、ややこしくなるし切り離して考えても問題無さそうなので切り離す。

# 問題

CFn は Thing しか扱ってない

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iot-thing.html

でも[この Issue](https://github.com/aws/aws-cdk/issues/18872)は ThingGroup と ThingType も扱ってくれよ。って Issue。

ThingType と ThingGroup は API から扱うしかない。

APIs

- Thing
  - [CreateThing](https://docs.aws.amazon.com/iot/latest/apireference/API_CreateThing.html)
  - [UpdateThing](https://docs.aws.amazon.com/iot/latest/apireference/API_UpdateThing.html)
  - [DeleteThing](https://docs.aws.amazon.com/iot/latest/apireference/API_DeleteThing.html)
  - [RegisterThing](https://docs.aws.amazon.com/iot/latest/apireference/API_RegisterThing.html)
- ThingType
  - [CreateThingType](https://docs.aws.amazon.com/iot/latest/apireference/API_CreateThingType.html)
  - [DeleteThingType](https://docs.aws.amazon.com/iot/latest/apireference/API_DeleteThingType.html)
  - [DeprecateThingType](https://docs.aws.amazon.com/iot/latest/apireference/API_DeprecateThingType.html)
- ThingGroup
  - [CreateThingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_CreateThingGroup.html)
  - [UpdateThingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_UpdateThingGroup.html)
  - [DeleteThingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_DeleteThingGroup.html)
  - [AddThingToThingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_AddThingToThingGroup.html)
  - [RemoveThingFromThingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_RemoveThingFromThingGroup.html)
- DynamicThingGroup
  - [CreateDynamicThingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_CreateDynamicThingGroup.html)
  - [UpdateDynamicThingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_UpdateDynamicThingGroup.html)
  - [DeleteDynamicThingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_DeleteDynamicThingGroup.html)
- BillingGroup
  - [CreateBillingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_CreateBillingGroup.html)
  - [UpdateBillingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_RemoveThingFromBillingGroup.html)
  - [DeleteBillingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_DeleteBillingGroup.html)
  - [AddThingToBillingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_AddThingToBillingGroup.html)
  - [RemoveThingFromBillingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_RemoveThingFromBillingGroup.html)

# 作戦

1. まずは Thing を作る
1. ThingType
   1. ThingType を作る
   1. Thing と紐付けれるようにする
1. ThingGroup
   1. ThingGroup を作る
   1. Thing と紐付けれるようにする
1. DynamicThingGroup
   1. DynamicThingGroup を作る
   1. Thing と紐付けれるようにする
1. BillingGroup
   1. BillingGroup を作る
   1. Thing と紐付けれるようにする

integ-test を駆使する感じになりそう。

# usage example

```ts
import * as iot from "aws-cdk-lib/aws-iot";

const thingType = new iot.ThingType(this, "ThingType", {
  thingTypeName: "example-thingTypeName",
  thingTypeDescription: "example-thingTypeDescription",
  searchableAttributes: ["xxx", "yyy"],
});

const thing = new iot.Thing(this, "Thing", {
  thingName: "example-thingName",
  attributes: {
    location: "xxxx",
  },
  thingType,
});

const thingGroupL1 = new iot.Thing(this, "ThingGroupLayer1", {
  thingGroupName: "example-thingGroupName",
  thingGroupDescription: "example-thingGroupDescription",
  parentGroup:
  attributes: {
    location: "xxxx",
  },
  mergeAttributes: true,
});

const thingGroupL2 = new iot.Thing(this, "ThingGroupLayer2", {
  thingGroupName: "example-thingGroupName",
  thingGroupDescription: "example-thingGroupDescription",
  parentGroup: thingGroupL1,
  attributes: {
    location: "xxxx",
  },
  mergeAttributes: true,
});

thing.addGroup(thingGroupL2);
```
