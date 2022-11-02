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
    - Provisioning なので、 `merge` は使わなそう
    - Provisioning なので、 `expectedVersion` は使わなそう
    - deprecated thing type に紐付けることはできない
  - [DeleteThing](https://docs.aws.amazon.com/iot/latest/apireference/API_DeleteThing.html)
- ThingType
  - [CreateThingType](https://docs.aws.amazon.com/iot/latest/apireference/API_CreateThingType.html)
  - [DeprecateThingType](https://docs.aws.amazon.com/iot/latest/apireference/API_DeprecateThingType.html)
    - 紐づく thing がいても deprecate 可能
  - [DeleteThingType](https://docs.aws.amazon.com/iot/latest/apireference/API_DeleteThingType.html)
    - deprecated してないと削除できない（どうしよう）
    - deprecated にしてから 5 分経たないと削除できない（どうしよう）
    - 紐づく thing がいると削除不可能
- ThingGroup
  - [CreateThingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_CreateThingGroup.html)
  - [UpdateThingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_UpdateThingGroup.html)
  - [DeleteThingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_DeleteThingGroup.html)
    - 子 group がいると削除できない
    - thing が紐付いていても消せる
  - [AddThingToThingGroup](https://docs.aws.amazon.com/iot/latest/apireference/API_AddThingToThingGroup.html)
    - 1 つの thing が、同じ木の中の 2 つの group に紐づくことはできない
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

# 調査用コマンド

```sh
# Thing
aws iot create-thing --thing-name 'test-thing' --attribute-payload 'attributes={KeyName1=foo,KeyName2=bar}' # --billing-group-name
aws iot create-thing --thing-name 'test-thing' --attribute-payload 'attributes={KeyName1=foo,KeyName2=bar}' --thing-type-name 'test-thing-type'
aws iot update-thing --thing-name 'test-thing' --attribute-payload 'attributes={KeyName3=buz},merge=false'
aws iot update-thing --thing-name 'test-thing' --attribute-payload 'attributes={KeyName1=foo,KeyName2=bar},merge=true'
aws iot update-thing --thing-name 'test-thing' --attribute-payload 'attributes={KeyName3=""},merge=true'
aws iot update-thing --thing-name 'test-thing' --attribute-payload 'attributes={KeyName3=foobar},merge=true' --expected-version 1
aws iot update-thing --thing-name 'test-thing' --remove-thing-type
aws iot update-thing --thing-name 'test-thing' --thing-type-name 'test-thing-type'
aws iot delete-thing --thing-name 'test-thing'
aws iot list-things
aws iot list-thing-groups-for-thing --thing-name 'test-thing'
# ThingType
aws iot create-thing-type --thing-type-name 'test-thing-type'
aws iot create-thing-type --thing-type-name 'test-thing-type' --thing-type-properties 'thingTypeDescription=foo,searchableAttributes=bar,buz'
aws iot deprecate-thing-type --thing-type-name 'test-thing-type'
aws iot deprecate-thing-type --thing-type-name 'test-thing-type' --undo-deprecate
aws iot delete-thing-type --thing-type-name 'test-thing-type'
aws iot list-thing-types
# ThingGroup
aws iot create-thing-group --thing-group-name 'test-thing-group-l1-1' --thing-group-properties 'thingGroupDescription=foo,attributePayload={attributes={KeyName1=bar,KeyName2=buz}}'
aws iot create-thing-group --thing-group-name 'test-thing-group-l1-2'
aws iot create-thing-group --thing-group-name 'test-thing-group-l2-1' --parent-group-name 'test-thing-group-l1-1'
aws iot create-thing-group --thing-group-name 'test-thing-group-l2-2' --parent-group-name 'test-thing-group-l1-1'
aws iot update-thing-group --thing-group-name 'test-thing-group-l1-1' --thing-group-properties 'thingGroupDescription=foobar'
aws iot update-thing-group --thing-group-name 'test-thing-group-l1-1' --thing-group-properties 'attributePayload={attributes={KeyName3=foobar}}'
aws iot delete-thing-group --thing-group-name 'test-thing-group-l1-1'
aws iot delete-thing-group --thing-group-name 'test-thing-group-l1-2'
aws iot delete-thing-group --thing-group-name 'test-thing-group-l2-1'
aws iot delete-thing-group --thing-group-name 'test-thing-group-l2-2'
aws iot add-thing-to-thing-group --thing-group-name 'test-thing-group-l1-1' --thing-name 'test-thing' # --override-dynamic-groups
aws iot add-thing-to-thing-group --thing-group-name 'test-thing-group-l1-2' --thing-name 'test-thing' # --override-dynamic-groups
aws iot add-thing-to-thing-group --thing-group-name 'test-thing-group-l2-1' --thing-name 'test-thing' # --override-dynamic-groups
aws iot add-thing-to-thing-group --thing-group-name 'test-thing-group-l2-2' --thing-name 'test-thing' # --override-dynamic-groups
aws iot remove-thing-from-thing-group --thing-group-name 'test-thing-group-l1-1' --thing-name 'test-thing'
aws iot remove-thing-from-thing-group --thing-group-name 'test-thing-group-l1-2' --thing-name 'test-thing'
aws iot remove-thing-from-thing-group --thing-group-name 'test-thing-group-l2-1' --thing-name 'test-thing'
aws iot remove-thing-from-thing-group --thing-group-name 'test-thing-group-l2-2' --thing-name 'test-thing'
aws iot list-thing-groups
aws iot describe-thing-group --thing-group-name 'test-thing-group-l1-1'
aws iot describe-thing-group --thing-group-name 'test-thing-group-l2-1'
# DynamicThingGroup
aws iot create-dynamic-thing-group --thing-group-name 'test-dynamic-thing-group'
aws iot update-dynamic-thing-group --thing-group-name 'test-dynamic-thing-group'
aws iot delete-dynamic-thing-group --thing-group-name 'test-dynamic-thing-group'
# BillingGroup
aws iot create-billing-group help
aws iot update-billing-group help
aws iot delete-billing-group help
aws iot add-thing-to-billing-group help
aws iot remove-thing-from-billing-group help
```

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

## Thing, Thing Type, Thing Group

```ts
import * as iot from "aws-cdk-lib/aws-iot";

const thingType = new iot.ThingType(this, "ThingType", {
  thingTypeName: "example-thingTypeName",
  thingTypeDescription: "example-thingTypeDescription",
  searchableAttributes: ["xxx", "yyy"],
});

const thingGroup = new iot.Thing(this, "ThingGroup", {
  thingGroupName: "example-thingGroupName",
  thingGroupDescription: "example-thingGroupDescription",
  attributes: {
    location: "xxxx",
  },
  mergeAttributes: true,
});

const thing = new iot.Thing(this, "Thing", {
  thingName: "example-thingName",
  attributes: {
    location: "xxxx",
  },
  thingType,
  thingGroup,
});
```

## Add thing to thing group

```ts
import * as iot from "aws-cdk-lib/aws-iot";

const thingGroup = new iot.Thing(this, "ThingGroup", {
  thingGroupName: "example-thingGroupName",
});

const thing = new iot.Thing(this, "Thing", {
  thingName: "example-thingName",
});

thing.addToGroup(thingGroup);
```

## Nested Group

```ts
import * as iot from "aws-cdk-lib/aws-iot";

const thingGroupL1 = new iot.Thing(this, "ThingGroupLayer1", {
  thingGroupName: "example-thingGroupName",
});

const thingGroupL2 = new iot.Thing(this, "ThingGroupLayer2", {
  thingGroupName: "example-thingGroupName",
  parentGroup: thingGroupL1,
});
```
