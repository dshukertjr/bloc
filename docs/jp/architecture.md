# アーキテクチャー

![Bloc Architecture](../assets/bloc_architecture.png)

Bloc はアプリケーションを３つのレイヤーに分けてくれます：

- データ
  - Data Provider
  - Repository
- Business Logic (Bloc)
- プレゼンテーション

ここではまずユーザーインターフェイスから一番遠いレイヤーから解説していきます。

## データレイヤー

> データレイヤーは様々なデータソースとデータのやり取りをするレイヤーです。

データレイヤーは２つに分けることができます：

- Repository
- Data Provider

データレイヤーはアプリケーション内でもっともローレベルなレイヤーで、ネットワークリクエストやデータベースなどとのやり取りを行います。

### Data Provider

> Data Provider の役割は生のデータを返すことです。Data Provider は汎用的で多用途である必要があります。

Data Provider は複雑な処理はせず、シンプルに[CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)用の API を公開するだけです。
例えば、`createData`, `readData`, `updateData`, `deleteData` のようなメソッドをデータレイヤーに作ったりするでしょう。

[data_provider.dart](../_snippets/architecture/data_provider.dart.md ':include')

### Repository

> Repository レイヤーは１つ、もしくは複数の data provider のラッパーとなり、Bloc レイヤーと交信するためにあります。

[repository.dart](../_snippets/architecture/repository.dart.md ':include')

みてわかる通り、repository レイヤーは複数の data provider と更新することができ、bloc にデータを渡す前の変換なども行います。

## Bloc (Business Logic) レイヤー

> Bloc レイヤーの役割はプレゼンテーションレイヤーから来た event を受け、プレゼンテーションレイヤーに新しい state を返すことです。Bloc レイヤーは state に返すデータを取得するために複数の repository に依存することがあります。

Bloc レイヤーはプレゼンテーションレイヤーとデータレイヤーの架け橋となるような存在です。ユーザーアクションなどから発せられた event を受け取り、repository と交信をし新しい state を生成してプレゼンテーションレイヤーに返してあげています。

[business_logic_component.dart](../_snippets/architecture/business_logic_component.dart.md ':include')

### Bloc 間のコミュニケーション

> 全ての bloc は他の bloc が受け取り、その状態に応じて反応できる state の stream を持っています。

Bloc は他の bloc に依存することでその bloc の state の変化に反応することができます。下記の例では`MyBloc`は`OtherBloc`に依存していて、`OtherBloc`の state が変わった時に event を`add`することができます。また、メモリーリークを防ぐために`StreamSubscription`は`MyBloc`の中の`close`メソッド内で close します。

[bloc_to_bloc_communication.dart](../_snippets/architecture/bloc_to_bloc_communication.dart.md ':include')

## プレゼンテーションレイヤー

> プレゼンテーションレイヤーは bloc から流れてくる state を受け取りそれに応じて UI を構築するレイヤーです。加えて、タップなどのユーザーアクションやウィジェットのライフサイクルに反応して event を bloc に流してあげる役割もあります。

ほとんどのアプリは`AppStart`event から始まりデータを引っ張ってきて初期画面をユーザーに見せるところから始まります。

この場合では`AppStart` event をプレゼンテーションレイヤーの中から add します。

これに加えて、プレゼンテーションレイヤーはその state がきた時にどの UI を表示するかを決めないといけません。

[presentation_component.dart](../_snippets/architecture/presentation_component.dart.md ':include')

ここまでの話は多少のコーディングはあったものの、ほとんどがコンセプトの話でした。チュートリアルではこれらの知識を組み合わせ実際に複数のアプリを作っていく中でこれらがどのように組み合わさっていくのかをみていきます。
