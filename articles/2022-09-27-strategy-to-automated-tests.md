---
title: "[メモ]「組織に自動テストを根付かせる戦略」の記事を読んで"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [test, devops]
published: true
---

t-wada さんの[組織に自動テストを根付かせる戦略](https://www.publickey1.jp/blog/22/12022.html)
がとても重要そうだったので暗唱できるようになりたい。

「自動テストがあることが良いこと」ではない「プロダクトコードと一緒にテストコートを書いていることが良いこと」だ。

## 分からないものをどう作るか？

未知の未知とかリーンとかの話。

リリースした製品は必ず間違っている。

それをどう fit させていくか。

## ソフトウェアの開発力を測る 4 つのメトリクス

4 つのキーメトリクス

- リードタイム
- デプロイ頻度
- MTTR（平均修復時間）
- 変更失敗率

「リリースした製品は必ず間違っている。」という前提に立った上で、高速に改善していくための指標。

## スピードの速い企業は品質が高く、スピードの遅い企業は品質が低い

スピードの速い企業は品質が高く、スピードの遅い企業は品質が低い。

各クラスターの配分も変わっていて、エリートとハイパーフォーマーの数が増えてきている一方で、ミディアムやローパフォーマーからなかなか抜けられない企業もいる、という傾向。

## 組み込みなのか基幹系なのかはパフォーマンスに関係ない

システムのタイプ、つまり組み込みなのか基幹系なのかパッケージなのか、といったものと、先ほどの 4 つのキーメトリクスには相関関係がない。

## 「テスト容易性」と「デプロイ独立性」が重要

システムのタイプよりも、システムの 2 つのアーキテクチャの特性がパフォーマンスの差になり、つまりそれが業績に繋がっている。

「テスト容易性」と「デプロイ容易性（デプロイ独立性）」

一つ目は「テストの大半を統合環境を必要とせずに実施できる」
統合環境というのはステージング環境みたいなものです。テスト用に用意された実機とか、本番環境に近い環境のことです。

二つ目は「アプリケーションを、それが依存する他のアプリケーションサービスからは独立した形でデプロイまたはリリースできる」
つまり自分たちが作ってるシステムを、それが依存してるとか依存されているところとは独立して開発し、デプロイし、リリースできること。

## ソフトウェアを外注している企業はローパフォーマーになりやすい

「構築中のソフトウェア（あるいは利用する必要のある一群のサービス）は、他社（外部委託先など）が開発したカスタムソフトウェアである」

ソフトウェア開発を外注していると答えた企業は、有意にローパフォーマーになると。

## 継続的デリバリーの 5 つの基本原則

- 「品質」の概念を生産工程の最初から組み込む
- 作業は（小さい）バッチ処理で進める
- 反復作業はコンピュータに任せ人間は問題解決に当たる
- 徹底した改善努力を継続的に行う
- 全員が責任を担う

## 継続的デリバリーの 3 つの技術的な基盤

- 包括的な構成管理
- 継続的インテグレーション
- 自動テスト

## 開発チームと分かれた QA チームではパフォーマンスは向上しない

受け入れテストや End-to-End テストなどの開発を外部に委託しても、業績とは関係がない。
むしろ外部に委託する場合は、先ほど言ったようにパフォーマンスとしては落ちる傾向にあります。

## 信頼性の高い自動テストを備えること

これが今自分にないものだな、と感じた。
「自動テスト通ったからデプロイできる」ってならないと 1 日に数回デプロイすることはできない。

> 現場を見回してみると「前々任者と前任者が書いたテストコードがどうも内容は重複していそうだし同時に失敗するんだけども、2 人とももういないし、そのテストコードを消す度胸もないから歯を食いしばってメンテナンスしていくしかない」といったような、保守性や可読性、理解容易性や変更容易性などに問題を抱えたテストコードがたくさん現場を苦しめています。

たまによく見る光景な気がする。テストコードが悪いのか、モジュールの理解容易性が低いのか。
