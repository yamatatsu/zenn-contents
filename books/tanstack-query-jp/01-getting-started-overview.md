---
title: "Overview"
---

TanStack Query (FKA React Query) is often described as the missing data-fetching library for web applications, but in more technical terms, it makes **fetching, caching, synchronizing and updating server state** in your web applications a breeze.

<!-- 上記文章の和訳 -->
TanStack Query は、Web アプリケーションのための data-fetching library としてしばしば言及されます。しかし、より技術的な言葉で言えば、TanStack Query は **server state の取得、キャッシュ、同期、更新**を簡単に行うためのものです。

## Motivation

Most core web frameworks **do not** come with an opinionated way of fetching or updating data in a holistic way. Because of this developers end up building either meta-frameworks which encapsulate strict opinions about data-fetching, or they invent their own ways of fetching data. This usually means cobbling together component-based state and side-effects, or using more general purpose state management libraries to store and provide asynchronous data throughout their apps.

<!-- 上記文章の和訳 -->
ほとんどのコアWebフレームワークには、データの取得や更新についての意見が統一された方法が**付属していません**。そのため、開発者はデータ取得に関する厳格な意見をカプセル化したメタフレームワークを構築するか、独自のデータ取得方法を考案します。これは、コンポーネントベースのステートと副作用を組み合わせるか、より一般的なステート管理ライブラリを使用してアプリケーション全体で非同期データを格納および提供することを意味します。

While most traditional state management libraries are great for working with client state, they are **not so great at working with async or server state**. This is because server state is totally different. For starters, server state:

<!-- 上記文章の和訳 -->
ほとんどの伝統的なステート管理ライブラリは、 client state と一緒に使うのに適していますが、**非同期または server state と一緒に使うのには適していません**。これは、**server state が client state とはまったく異なるものである**からです。まず、 server state は次のようなものです。

- Is persisted remotely in a location you may not control or own
- Requires asynchronous APIs for fetching and updating
- Implies shared ownership and can be changed by other people without your knowledge
- Can potentially become "out of date" in your applications if you're not careful

<!-- 上記箇条書きのの和訳 -->
- あなたが制御または所有していない場所にリモートで永続化されています
- 取得および更新のために非同期APIが必要です
- shared ownership を前提とし、他の人によってあなたの知識なしに変更される可能性があります
- 慎重でないと、アプリケーション内で "out of date" になる可能性があります

Once you grasp the nature of server state in your application, **even more challenges will arise** as you go, for example:

<!-- 上記文章の和訳 -->
アプリケーション内の server state の性質を理解すると、**さらに多くの課題が発生します**。たとえば：

- Caching... (possibly the hardest thing to do in programming)
- Deduping multiple requests for the same data into a single request
- Updating "out of date" data in the background
- Knowing when data is "out of date"
- Reflecting updates to data as quickly as possible
- Performance optimizations like pagination and lazy loading data
- Managing memory and garbage collection of server state
- Memoizing query results with structural sharing

<!-- 上記箇条書きのの和訳 -->
- cache...（おそらくプログラミングで最も難しいこと）
- 同じデータに対する複数のリクエストを単一のリクエストにまとめる
- バックグラウンドで "out of date" データを更新する
- データが "out of date" かどうかを知る
- データの更新をできるだけ早く反映する
- ページネーションやデータの lazy loading などのパフォーマンス最適化
- server state のメモリ管理とガベージコレクション
- structural sharingを使用したクエリ結果のメモ化

If you're not overwhelmed by that list, then that must mean that you've probably solved all of your server state problems already and deserve an award. However, if you are like a vast majority of people, you either have yet to tackle all or most of these challenges and we're only scratching the surface!

<!-- 上記文章の和訳 -->
もしあなたがこのリストを読んでも圧倒されていない場合、すでにすべての server state の問題を解決しているか、解決に値するということです。しかし、ほとんどの人と同じように、これらの課題のすべてまたはほとんどをまだ解決していない場合、私たちはまだ表面をかいているだけです！（英語の慣用句かな）

React Query is hands down one of the best libraries for managing server state. It works amazingly well **out-of-the-box, with zero-config, and can be customized** to your liking as your application grows.

<!-- 上記文章の和訳 -->
React Query は、server state を管理するための最高のライブラリの1つです。**ゼロコンフィグ**で驚くほどうまく機能し、アプリケーションが成長するにつれて好みに合わせて**カスタマイズ**できます。

React Query allows you to defeat and overcome the tricky challenges and hurdles of server state and control your app data before it starts to control you.

<!-- 上記文章の和訳 -->
React Query は、server state の難しい課題や障害を克服し、アプリケーションデータを制御することができます。

On a more technical note, React Query will likely:

<!-- 上記文章の和訳 -->
技術的な観点から言えば、React Query はおそらく次のようなことをします：

- Help you remove **many** lines of complicated and misunderstood code from your application and replace with just a handful of lines of React Query logic.
- Make your application more maintainable and easier to build new features without worrying about wiring up new server state data sources
- Have a direct impact on your end-users by making your application feel faster and more responsive than ever before.
- Potentially help you save on bandwidth and increase memory performance

<!-- 上記箇条書きのの和訳 -->
- アプリケーションから多くの複雑で誤解されたコードを削除し、React Query ロジックの数行だけで置き換えるのを助けます。
- 新しい server state データソースを接続することなく、アプリケーションをより保守可能で新機能の構築が容易になります。
- アプリケーションを以前よりも速く、より反応性の高いものにすることで、エンドユーザーに直接的な影響を与えます。
- バンド幅を節約し、メモリパフォーマンスを向上させるのに役立つかもしれません。

## Enough talk, show me some code already!

In the example below, you can see React Query in its most basic and simple form being used to fetch the GitHub stats for the React Query GitHub project itself:

<!-- 上記文章の和訳 -->
以下の例では、React Query を最も基本的でシンプルな形で使用して、React Query GitHub プロジェクト自体の GitHub の統計情報を取得しています。

```tsx

import {
  QueryClient,
  QueryClientProvider,
  useQuery,
} from '@tanstack/react-query'

const queryClient = new QueryClient()

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Example />
    </QueryClientProvider>
  )
}

function Example() {
  const { isPending, error, data } = useQuery({
    queryKey: ['repoData'],
    queryFn: () =>
      fetch('https://api.github.com/repos/TanStack/query').then((res) =>
        res.json(),
      ),
  })

  if (isPending) return 'Loading...'

  if (error) return 'An error has occurred: ' + error.message

  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.description}</p>
      <strong>👀 {data.subscribers_count}</strong>{' '}
      <strong>✨ {data.stargazers_count}</strong>{' '}
      <strong>🍴 {data.forks_count}</strong>
    </div>
  )
}
```

## You talked me into it, so what now?

Consider taking the official React Query Course (or buying it for your whole team!)
Learn React Query at your own pace with our amazingly thorough Walkthrough Guide and API Reference

<!-- 上記文章の和訳 -->
公式の React Query コースを受講することを検討してください（またはチーム全体で購入してください！）
驚くほど詳細なウォークスルーガイドと API リファレンスで自分のペースで React Query を学びましょう
