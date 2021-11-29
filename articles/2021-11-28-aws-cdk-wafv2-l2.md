---
title: "@aws-cdk/aws-wafv2 の L2 の設計を考えてみる"
emoji: "📖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws, awscdk, awswafv2]
published: true
---

考え中の公開ノート。

## WebACL

### CFn

[webacl](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-wafv2-webacl.html)

CloudFormation はこんな感じ。
![](/images/2021-11-28-aws-cdk-wafv2-l2-1.jpg)
![](/images/2021-11-28-aws-cdk-wafv2-l2-2.jpg)

[Statement](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-wafv2-webacl-statement.html#cfn-wafv2-webacl-statement-ipsetreferencestatement)はめんどいやつ。別パッケージ切るパターンか。

### Design

#### WebACL

```mermaid
classDiagram
  WebACL ..> WebACLProps
  WebACLProps o.. CustomResponseBody
  WebACLProps o.. Scope
  WebACLProps o.. DefaultAction
  WebACLProps o.. Rule
  DefaultAction ..> DefaultActionConfig
  WebACLProps o.. VisibilityConfig
  Rule o.. VisibilityConfig

  class WebACL {
    +constructor(props: WebACLProps)
  }
  class WebACLProps {
    name?: string;
    description?: string;
    scope: Scope;
    customResponseBodies?: Record<string, CustomResponseBody>;
    defaultAction: DefaultAction;
    rules?: Rule[];
    visibilityConfig?: VisibilityConfig;
  }
  <<Interface>> WebACLProps

  class CustomResponseBody {
    content: string;
    contentType: string;
  }
  <<Interface>> CustomResponseBody

  class Scope {
    REGIONAL
    CLOUDFRONT
  }
  <<enumerate>> Scope

  class DefaultAction {
    allow()$ DefaultAction
    block()$ DefaultAction
    bind()* DefaultActionConfig
  }
  <<abstract>> DefaultAction
  class DefaultActionConfig {
    configuration: CfnWebACL.DefaultActionProperty;
  }
  <<Interface>> DefaultActionConfig

  class Rule {
    name: string;
    action: RuleAction;
    overrideAction: OverrideAction;
    priority: number;
    statement: Statement;
    visibilityConfig?: VisibilityConfig;
    ruleLabels?: Label[];
  }
  <<Interface>> Rule
  class VisibilityConfig {
    cloudWatchMetricsEnabled: boolean;
    metricName: string;
    sampledRequestsEnabled: boolean;
  }
  <<Interface>> VisibilityConfig
```

#### Rule

```mermaid
classDiagram
  Rule o.. RuleAction
  RuleAction ..> RuleActionConfig
  Rule o.. OverrideAction
  OverrideAction ..> OverrideActionConfig
  Rule o.. IStatement
  IStatement ..> StatementConfig

  class Rule {
    name: string;
    action: RuleAction;
    overrideAction: OverrideAction;
    priority?: number;
    statement: Statement;
    visibilityConfig: VisibilityConfig;
    ruleLabels?: string[];
  }
  <<Interface>> Rule

  class RuleAction {
    allow()$ RuleAction
    block()$ RuleAction
    count()$ RuleAction
    bind()* RuleActionConfig
  }
  <<abstract>> RuleAction
  class RuleActionConfig {
    configuration: CfnRuleGroup.RuleActionProperty
  }
  <<Interface>> RuleActionConfig

  class OverrideAction {
    count()$ OverrideAction
    none()$ OverrideAction
    bind()* OverrideActionConfig
  }
  <<abstract>> OverrideAction
  class OverrideActionConfig {
    count?: Json;
    none?: Json;
  }

  class IStatement {
    bind() StatementConfig
  }
  <<Interface>> IStatement
  class StatementConfig {
    configuration: CfnRuleGroup.StatementProperty;
  }
  <<Interface>> StatementConfig
```

Statements

- Logical
  - AndStatement
  - NotStatement
  - OrStatement
- Depends on other resources
  - IPSetReferenceStatement
  - RegexPatternSetReferenceStatement
  - RuleGroupReferenceStatement
- Others
  - ByteMatchStatement
  - GeoMatchStatement
  - LabelMatchStatement
  - ManagedRuleGroupStatement
  - RateBasedStatement
  - SizeConstraintStatement
  - SqliMatchStatement
  - XssMatchStatement

### Usage

```ts
new wafv2.WebACL(this, "WebACL", {
  scope: wafv2.Scope.REGIONAL,
  defaultAction: wafv2.DefaultAction.block(),
  rules: [
    {
      name: "IPSetAllow",
      action: wafv2.RuleAction.allow(),
      statement: new wafv2Statement.IPSetReferenceStatement(ipSet),
    },
    {
      name: "OWASP",
      overrideAction: wafv2.OverrideAction.count(),
      statement: new wafv2Statement.ManagedRuleGroupStatement({
        vendorName: "AWS",
        name: "AWSManagedRulesCommonRuleSet",
      }),
    },
  ],
});
```

> Note: `priority` of the `rules` is automatically numbered according to the order of the `rules`.
> If `priority` is set specifically, all rules must be set `priority`.

> Note: `visibilityConfig` have default value.
> If `WebACLProps.visibilityConfig` is set, Rules inherit it.

### Roadmap

1. implement `WebACL` with only required properties
   - It will not be able to use Rules
1. implement `Rule` with one `Statement`(LabelMatchStatement)
1. implement other remaining properties
1. implement Statements
