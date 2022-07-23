---
title: "[読解メモ]How to model one-to-many relationships in DynamoDB"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [aws, dynamodb]
published: true
---

[How to model one-to-many relationships in DynamoDB](https://www.alexdebrie.com/posts/dynamodb-one-to-many/) というブログがめっちゃ勉強になったので、それの読会メモをおいておく。

これは、RDB のデータモデルを DynamoDB のシングルテーブル（マルチスキーマ）設計に落とし込むパターンについて記載されたブログである。

このブログでは 1 対 n を表現する 5 つの手法が紹介されている。

- Denormalization by using a complex attribute
- Denormalization by duplicating data
- Composite primary key + the Query API action
- Secondary index + the Query API action
- Composite sort keys with hierarchical data

# Denormalization by using a complex attribute

Set や Map を用いた非正規化。
MongoDB だと「embedded data」って名前で正規化と対比させて説明されていたりする。（MongoDB の話を出してしまったが、MongoDB は embedded data に対してもクエリをかけれるので、DynamoDB の記事で名前を出すのは結構違うかもしれない。）

この手法は以下の 3 つの条件を満たす場合に使える（元のブログでは 2 つと紹介されている。）

### 検索キーに使われないこと

Set や Map として保存された Attribute は GSI を指定しても主キー検索に用いることはできません。

### データサイズが大きすぎないこと

DynamoDB では Item のデータサイズは 400KB を超えることはできません。

### embedded data の部分的な変更が要件にないこと

これは元記事にはなかった条件。僕が MongoDB 畑から来たから、どうしてもこれを言いたかった。

DynamoDB では Set や Map の部分的な変更はサポートされていません。

ただし、集約指向 DB として DynamoDB を扱う場合、そして、ヴァーン・ヴァーノンの IDDD 本に習って設計された集約が設計されている場合、Attribute に保存された Map や Set の部分的な変更は発生しないため、問題はなくなる。  
なぜなら IDDD で紹介されている集約の設計では、集約は sub entity を持たず、VO のみを持つためである。

もし、集約が sub entity を持つ必要がある場合は、後述する`Composite primary key + the Query API action`パターンを用いて sub entity を別の Item に保存するのが有効であると思う。

# Denormalization by duplicating data

同一の事柄を示すデータを複数の Item で同時に保持する非正規化パターン。

上述した`Denormalization by using a complex attribute`に比べて、GSI などを用いて検索キーに使える点が大きく異る。

この手法は以下の 1 つの条件を満たす場合に使える。（元ブログでは 2 つだけど、一つにまとめて説明する）

### 複製されるデータが不変であること

これに尽きるし、相当な覚悟を持ってこれを確実にしなければならない。アプリケーションからの Item 追加や編集とかも考えると、この複製データを変更するのは相当難しいと言えるからである。

この「不変である」というのは「モデルとして基本的に不変である」だけでは足りないと僕は思う。なぜならアプリケーションからの誤入力が考慮されるべきだからである。
生年月日のような相当に不変に思えるデータですら「誤入力の訂正」というパターンで、変更が求められることがある。

つまりこのパターンは「現実世界の事象の記録」には使えないと思う。「システムの中で発生した事象の記録」には使えるはずだ。例えば「購入日時」など、キャンセルはあっても訂正はありえない。
（元ブログでは、本の著者の誕生日などの「現実世界の事象」が例に挙げられているが、これは誤入力の更新を考えるとそれなりのリスクがあるものと思う。）

元ブログでも記載されているが「検索容易性の向上」というメリットが、この手法以外では受けづらく、そしてその価値が十分に高く、そしてデータの変更が必要ないデータにだけ用いられるべきだろう。

# Composite primary key + the Query API action

pk のみを指定して検索することで、複数の sk に紐付けられた、複数の異なる entity を取得する手法。  
最も DynamoDB らしくて、正直言ってかっこいい。かっこいいよ。。。

この手法と後述する 2 つの手法は「複数の異なる entity が取得可能である」という点で前述の 2 とは異なる。（`Denormalization by using a complex attribute`で取得できるのは VO であると仮定する）

# Secondary index + the Query API action

上記の`Composite primary key + the Query API action`の GSI 版。かっこいい。。。

この`gsi1pk`と`gsi1sk`を他の Attribute とは別で持つ手法、公式のベストプラクティスにはなかった気がする。更新処理ミスる（gsi\d[ps]k の更新を忘れる）のが怖いけど、そこさえ気をつければ少ない GSI でいろんなことができるのが魅力。

# Composite sort keys with hierarchical data

上記の 2 つ`Composite primary key + the Query API action`, `Composite sort keys with hierarchical data`に合わせて sk に対する `begins_with` を使うパターン。
はー、かっこいい。

この hierarchical data を見ると、DynamoDB も MongoDB と同じように木のデータとして設計する必要がある、ということな気がする。
GSI という複製飛び道具が使えるタイプの木だ。
