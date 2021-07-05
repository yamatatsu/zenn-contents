---
title: "[AWS-CDK] lambda@edge のログが見つからないときはカリフォルニアにあるかも"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["awscdk", "cloudfront"]
published: true
---

# 課題

aws-cdk で建てた Cloud Front の lambda@edge のログがどこにも見つからない。悲しい。

# tl;dr

- カリフォルニア(`us-west-1`)の CloudWatch Losgs にあるかもよ。カリフォルニアになかったら、カナダとかアメリカとかヨーロッパのリージョンで CloudWatch Logs を確認してみよう。
- エッジを利かすためにも `priceClass: cloudfront.PriceClass.PRICE_CLASS_200` を設定しよう。

# 何が起きたの？

lambda@edge のログがカリフォルニアに勝手に飛ばされた？いいえ。lambda@edge がカリフォルニアで実行されたのです。

lambda@edge のログは[公式のドキュメント](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-cloudwatch-metrics-logging.html#lambda-cloudwatch-logs)の記載通り、実行されたエッジロケーションが所属するリージョンの CloudWatch Losgs に出力されます。

# なんで lambda@edge はカリフォルニアで実行されたの？

これが割とハマりポイントかもしれない。
AWS CDK で CloudFront Distribution を作成すると、デフォルトでは最も安い PriceClass である `Price Class 100` で作成されます。
この `Price Class 100` というのは、 `United States, Mexico, & Canada` または `Europe & Israel` のリージョンに所属するエッジロケーションを利用します。つまり東京のエッジロケーションは使われません。
https://aws.amazon.com/jp/cloudfront/pricing/

`Price Class 100` で使えるキャッシュサーバーのうち、僕がアクセスしたのがたまたまカリフォルニアだったのです。

つまりせっかく CDN 建てても、遠くアメリカのキャッシュが使われているということ、悲しい。

# PriceClass とは？

CloudFront では、利用されるエッジロケーションによって利用料金が変わります。ついうっかり、高額なエッジロケーションがドカドカ利用されると高額な請求になってしまう場合があります。（聞いたこと無いけど。理論上は。）
マネジメントコンソールや Cloud Formation の PriceClass はパフォーマンスを優先して`Price Class All`がデフォルトになっているのですが、AWS CDK は安さを優先してか、 `Price Class 100` がデフォルトになっています。

# どうすればいい？

PriceClass を`Price Class 200` か `Price Class All` に設定すれば、東京のエッジロケーションで実行されるようになるので、ログが東京の CloudWatch Logs に出力されるようになります。
Happy!!

# そんなことより

CloudFront Functions はどこで実行されても `us-east-1` にログが残ります。
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/monitoring-functions.html#monitoring-functions-logs
人類に優しいね！CloudFront Functions！

ただし、PriceClass は CloudFront 自体の設定であるため、CloudFront Functions についても、`Price Class 100` だと日本国内のエッジコンピューティングは利用できないと思われます。
`Price Class 200` にしよう！
