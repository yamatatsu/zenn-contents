---
title: "TypeScript"
---

React Query is now written in **TypeScript** to make sure the library and your projects are type-safe!

<!-- 上記文章の和訳 -->
React Query は今や **TypeScript** で書かれており、ライブラリとあなたのプロジェクトが型安全であることを確認しています！

Things to keep in mind:

<!-- 上記文章の和訳 -->
注意すべきこと：

- Types currently require using TypeScript **v4.7** or greater
- Changes to types in this repository are considered **non-breaking** and are usually released as patch semver changes (otherwise every type enhancement would be a major version!).
- It is **highly recommended that you lock your react-query package version to a specific patch release and upgrade with the expectation that types may be fixed or upgraded between any release**
- The non-type-related public API of React Query still follows semver very strictly.

<!-- 上記箇条書きのの和訳 -->
- 現在の型は TypeScript **v4.7** 以上が必要です
- このリポジトリの型の変更は **非破壊的** と見なされ、通常はパッチのsemver変更としてリリースされます（さもなければ、すべての型の強化がメジャーバージョンになってしまいます！）。
- **react-query パッケージのバージョンを特定のパッチリリースにロックし、型が修正されたりアップグレードされたりする可能性があることを前提にアップグレードすることを強くお勧めします**
- React Query の非型関連のパブリックAPIは引き続きsemverを非常に厳密に遵守しています。

## Type Inference

Types in React Query generally flow through very well so that you don't have to provide type annotations for yourself

<!-- 上記文章の和訳 -->
React Query の型は一般的に非常にうまく推量されるため、自分で型注釈を提供する必要はありません

```tsx
const { data } = useQuery({
  //    ^? const data: number | undefined
  queryKey: ['test'],
  queryFn: () => Promise.resolve(5),
})
```

```tsx
const { data } = useQuery({
  //      ^? const data: string | undefined
  queryKey: ['test'],
  queryFn: () => Promise.resolve(5),
  select: (data) => data.toString(),
})
```

This works best if your `queryFn` has a well-defined returned type. Keep in mind that most data fetching libraries return `any` per default, so make sure to extract it to a properly typed function:

<!-- 上記文章の和訳 -->
これは、`queryFn` がよく定義された戻り型を持っている場合に最もうまく機能します。ほとんどのデータ取得ライブラリはデフォルトで `any` を返すため、適切に型付けされた関数に抽出することを忘れないでください：

```tsx
const fetchGroups = (): Promise<Group[]> =>
  axios.get('/groups').then((response) => response.data)

const { data } = useQuery({ queryKey: ['groups'], queryFn: fetchGroups })
//      ^? const data: Group[] | undefined
```

## Type Narrowing

React Query uses a [discriminated union type](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html#discriminated-unions) for the query result, discriminated by the `status` field and the derived status boolean flags. This will allow you to check for e.g. `success` status to make `data` defined:

<!-- 上記文章の和訳 -->
React Query は、`status` フィールドと派生ステータスフラグ（`isSuccess`,`isError`,`isFetching`など）によって区別されたクエリ結果の区別された [discriminated union type](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html#discriminated-unions) を使用しています。これにより、たとえば `success` ステータスをチェックして `data` を定義できます：

```tsx
const { data, isSuccess } = useQuery({
  queryKey: ['test'],
  queryFn: () => Promise.resolve(5),
})

if (isSuccess) {
  data
  //  ^? const data: number
}
```

## Typing the error field

The type for error defaults to `Error`, because that is what most users expect.

<!-- 上記文章の和訳 -->
エラーフィールドの型は、ほとんどのユーザーが期待するものである `Error` にデフォルトで設定されています。

```tsx

const { error } = useQuery({ queryKey: ['groups'], queryFn: fetchGroups })
//      ^? const error: Error
```

If you want to throw a custom error, or something that isn't an `Error` at all, you can specify the type of the error field:

<!-- 上記文章の和訳 -->
カスタムエラーをスローしたい場合、または `Error` ではないものをスローしたい場合は、エラーフィールドの型を指定できます：

```tsx

const { error } = useQuery<Group[], string>(['groups'], fetchGroups)
//      ^? const error: string | null
```

However, this has the drawback that type inference for all other generics of `useQuery` will not work anymore. It is generally not considered a good practice to throw something that isn't an Error, so if you have a subclass like `AxiosError` you can use type narrowing to make the error field more specific:

<!-- 上記文章の和訳 -->
ただし、これには `useQuery` のすべての他のジェネリックの型推論がもはや機能しなくなるという欠点があります。一般的に、`Error` ではないものをスローすることは良い慣行とは考えられていません。そのため、`AxiosError` のようなサブクラスがある場合は、エラーフィールドをより具体的にするために型狭めを使用できます：

```tsx

import axios from 'axios'

const { error } = useQuery({ queryKey: ['groups'], queryFn: fetchGroups })
//      ^? const error: Error | null

if (axios.isAxiosError(error)) {
  error
  // ^? const error: AxiosError
}
```

## Registering a global Error

TanStack Query v5 allows for a way to set a global Error type for everything, without having to specify generics on call-sides, by amending the `Register` interface. This will make sure inference still works, but the error field will be of the specified type:

<!-- 上記文章の和訳 -->
TanStack Query v5 では、`Register` インターフェースを修正することで、呼び出し側でジェネリックを指定する必要なく、すべてのものに対してグローバルなエラータイプを設定する方法が提供されます。これにより、推論が引き続き機能することが保証されますが、エラーフィールドは指定された型になります：

```tsx

import '@tanstack/react-query'

declare module '@tanstack/react-query' {
  interface Register {
    defaultError: AxiosError
  }
}

const { error } = useQuery({ queryKey: ['groups'], queryFn: fetchGroups })
//      ^? const error: AxiosError | null
```

## Typing meta

### Registering global Meta

Similarly to registering a global error type you can also register a global `Meta` type. This ensures the optional meta field on queries and mutations stays consistent and is type-safe. Note that the registered type must extend `Record<string, unknown>` so that meta remains an object.

<!-- 上記文章の和訳 -->
グローバルエラータイプを登録するのと同様に、グローバル `Meta` タイプも登録できます。これにより、クエリとミューテーションのオプションのメタフィールドが一貫していて型安全になります。登録された型は `Record<string, unknown>` を拡張する必要があるため、メタがオブジェクトのままであることに注意してください。

```ts

import '@tanstack/react-query'

interface MyMeta extends Record<string, unknown> {
  // Your meta type definition.
}

declare module '@tanstack/react-query' {
  interface Register {
    queryMeta: MyMeta
    mutationMeta: MyMeta
  }
}
```

## Typing Query Options

If you inline query options into `useQuery`, you'll get automatic type inference. However, you might want to extract the query options into a separate function to share them between `useQuery` and e.g. `prefetchQuery`. In that case, you'd lose type inference. To get it back, you can use `queryOptions` helper:

<!-- 上記文章の和訳 -->
クエリオプションを `useQuery` にインラインで埋め込むと、自動的に型推論が行われます。ただし、クエリオプションを別の関数に抽出して `useQuery` と `prefetchQuery` の間で共有したい場合、型推論が失われる可能性があります。その場合、`queryOptions` ヘルパーを使用して戻すことができます：

```ts
import { queryOptions } from '@tanstack/react-query'

function groupOptions() {
  return queryOptions({
    queryKey: ['groups'],
    queryFn: fetchGroups,
    staleTime: 5 * 1000,
  })
}

useQuery(groupOptions())
queryClient.prefetchQuery(groupOptions())
```

Further, the `queryKey` returned from `queryOptions` knows about the `queryFn` associated with it, and we can leverage that type information to make functions like `queryClient.getQueryData` aware of those types as well:

<!-- 上記文章の和訳 -->
さらに、`queryOptions` から返される `queryKey` はそれに関連付けられた `queryFn` について知っており、その型情報を活用して `queryClient.getQueryData` のような関数もそれらの型を認識できるようにすることができます：

```ts
function groupOptions() {
  return queryOptions({
    queryKey: ['groups'],
    queryFn: fetchGroups,
    staleTime: 5 * 1000,
  })
}

const data = queryClient.getQueryData(groupOptions().queryKey)
//     ^? const data: Group[] | undefined
```

Without `queryOptions`, the type of data would be `unknown`, unless we'd pass a generic to it:

<!-- 上記文章の和訳 -->
`queryOptions` がない場合、データの型は `unknown` になりますが、ジェネリックを渡すと型がわかります：

```ts
const data = queryClient.getQueryData<Group[]>(['groups'])
```

## Further Reading

For tips and tricks around type inference, have a look at React Query and TypeScript from the Community Resources. To find out how to get the best possible type-safety, you can read Type-safe React Query.

<!-- 上記文章の和訳 -->
型推論に関するヒントやトリックについては、コミュニティリソースの React Query と TypeScript を参照してください。最高の型安全性を実現する方法については、Type-safe React Query を読んでください。

## Typesafe disabling of queries using `skipToken`

If you are using TypeScript, you can use the `skipToken` to disable a query. This is useful when you want to disable a query based on a condition, but you still want to keep the query to be type safe. Read more about it in the Disabling Queries guide.

<!-- 上記文章の和訳 -->
TypeScript を使用している場合、`skipToken` を使用してクエリを無効にできます。これは、条件に基づいてクエリを無効にしたい場合に便利ですが、クエリを型安全に保ちたい場合に使用します。詳細については、Disabling Queries ガイドを参照してください。
