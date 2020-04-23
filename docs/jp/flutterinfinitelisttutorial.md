# Flutter 無限リストチュートリアル

![intermediate](https://img.shields.io/badge/level-intermediate-orange.svg)

> このチュートリアルではユーザーがリストをスクロールして下に行くたびにネットワークからデータをロードしてきてリストに追加していくアプリを作成します。

![demo](../../assets/gifs/flutter_infinite_list.gif)

## 準備

まずは新規の Flutter プロジェクトを作成するところから始めましょう。

```bash
flutter create flutter_infinite_list
```

pubspec.yaml の中身をこのように書き換えましょう：

```yaml
name: flutter_infinite_list
description: A new Flutter project.

version: 1.0.0+1

environment:
  sdk: '>=2.6.0 <3.0.0'

dependencies:
  flutter:
    sdk: flutter
  flutter_bloc: ^3.2.0
  http: ^0.12.0
  equatable: ^1.0.0
  rxdart: ^0.23.1

flutter:
  uses-material-design: true
```

それができたらパッケージのインストールをします。

```bash
flutter packages get
```

## REST API

このアプリでは[jsonplaceholder](http://jsonplaceholder.typicode.com)をデータソースとして使います。

?> jsonplaceholder とはオンライン上のデモデータを吐き出してくれる REST API で、プロトタイプを作るときには最適です。

ブラウザで https://jsonplaceholder.typicode.com/posts?_start=0&_limit=2 にアクセスしてみてこの API がどんな値を返してくれるのかをみてみましょう。

```json
[
  {
    "userId": 1,
    "id": 1,
    "title": "sunt aut facere repellat provident occaecati excepturi optio reprehenderit",
    "body": "quia et suscipit\nsuscipit recusandae consequuntur expedita et cum\nreprehenderit molestiae ut ut quas totam\nnostrum rerum est autem sunt rem eveniet architecto"
  },
  {
    "userId": 1,
    "id": 2,
    "title": "qui est esse",
    "body": "est rerum tempore vitae\nsequi sint nihil reprehenderit dolor beatae ea dolores neque\nfugiat blanditiis voluptate porro vel nihil molestiae ut reiciendis\nqui aperiam non debitis possimus qui neque nisi nulla"
  }
]
```

?> **メモ:** 今アクセスした URL では start と limit をクエリーパラメターで指定しました。

API の返してるデータがどんなものかわかったので、これを元にモデルを作っていきましょう。

## データモデル

`post.dart`を作成し Post モデルを作りましょう。

```dart
import 'package:equatable/equatable.dart';

class Post extends Equatable {
  final int id;
  final String title;
  final String body;

  const Post({this.id, this.title, this.body});

  @override
  List<Object> get props => [id, title, body];

  @override
  String toString() => 'Post { id: $id }';
}
```

`Post` はただ `id`、`title`、`body`の三つのプロパティがあるだけのクラスです。

?> `toString`関数を上書きすることで`Post`クラスに独自の String 表示をさせることができます。

?> [`Equatable`](https://pub.dev/packages/equatable)を継承することで`Posts`のプロパティたちが一致するかどうかを検証できます。デフォルトではオブジェクトは同じインスタンスでないと == が true になりません。

`Post`モデルが完成したので Business Logic Component (bloc) に取り掛かりましょう。

## Post Event

実装に入る前に`PostBloc`が何をするのかを定義しなくてはなりません。

ハイレベルなところでは、`PostBloc`はユーザーインプット（スクロール）に対して反応し、追加で post をロードして表示するのが役割です。まずは`Event`を作るところから始めましょう。

`PostBloc`は一つの event しか持ちません。`Fetch` event はプレゼンテーションレイヤーがもっと post をロードして欲しいときに呼ばれるでしょう。 `Fetch` event は `PostEvent` の一種なので `bloc/post_event.dart` を作成しこのように実装することができます。

```dart
import 'package:equatable/equatable.dart';

abstract class PostEvent extends Equatable {
  @override
  List<Object> get props => [];
}

class Fetch extends PostEvent {}
```

?> 繰り返しになりますが、今回も`toString`を上書きしています。そして、[`Equatable`](https://pub.dev/packages/equatable)を継承し楽に == で二つの post インスタンスを比較できるようにしています。

おさらいをすると、`PostBloc`は`PostEvent`を受け取り、それを`PostState`に変換します。ここまでで全ての`PostEvent`は定義し終わりました。次に`PostState`を定義します。

## Post State

このアプリのプレゼンテーションレイヤーがレイアウトを作るためにいくつか送らないといけない情報があります:

- `PostUninitialized`- プレゼンテーションレイヤーに最初のコンテンツをロードする間ローディングインジケーターを表示することを知らせる
- `PostLoaded`- プレゼンテーションレイヤーに表示するコンテンツがあることを知らせる
  - `posts`- `List<Post>`型の表示する投稿一覧
  - `hasReachedMax`- ロードできる投稿を全てロードし終え、これ以上ロードするコンテンツがないことを知らせる
- `PostError`- 投稿をロードする最中にエラーが起きたことを知らせる

これが理解できたら`bloc/post_state.dart`をこのように作っていきましょう。

```dart
import 'package:equatable/equatable.dart';

import 'package:flutter_infinite_list/models/models.dart';

abstract class PostState extends Equatable {
  const PostState();

  @override
  List<Object> get props => [];
}

class PostUninitialized extends PostState {}

class PostError extends PostState {}

class PostLoaded extends PostState {
  final List<Post> posts;
  final bool hasReachedMax;

  const PostLoaded({
    this.posts,
    this.hasReachedMax,
  });

  PostLoaded copyWith({
    List<Post> posts,
    bool hasReachedMax,
  }) {
    return PostLoaded(
      posts: posts ?? this.posts,
      hasReachedMax: hasReachedMax ?? this.hasReachedMax,
    );
  }

  @override
  List<Object> get props => [posts, hasReachedMax];

  @override
  String toString() =>
      'PostLoaded { posts: ${posts.length}, hasReachedMax: $hasReachedMax }';
}
```

?> `copyWith`を実装しておくと`PostLoaded`のプロパティいくつか簡単に変化させることができます（あとで役に立つ時がきます）。

`Event`と`State`が実装できたらあとは`PostBloc`を作りましょう。

## Post Bloc

シンプルに作るために今回は`PostBloc`から`http client`を呼び出しますが、実際にプロダクションレベルのアプリを作る際にはレポジトリーパターンなどを使うようにすることをお勧めします[ドキュメント](jp/architecture.md)。

`post_bloc.dart`を作成し空の`PostBloc`を作りましょう。

```dart
import 'package:bloc/bloc.dart';
import 'package:meta/meta.dart';
import 'package:http/http.dart' as http;

import 'package:flutter_infinite_list/bloc/bloc.dart';
import 'package:flutter_infinite_list/post.dart';

class PostBloc extends Bloc<PostEvent, PostState> {
  final http.Client httpClient;

  PostBloc({@required this.httpClient});

  @override
  // TODO: 初期 state を定義する
  PostState get initialState => null;

  @override
  Stream<PostState> mapEventToState(PostEvent event) async* {
    // TODO: mapEventToState を定義する
    yield null;
  }
}
```

?> **メモ:** クラスの定義を見ただけで PostBloc は PostEvent を受け取り PostState を返してくれることがわかります。

まずなんの stete もロードされる前の`PostBloc`の state になる`initialState`を定義するところから始めましょう。

```dart
@override
get initialState => PostUninitialized();
```

次に`PostEvent`が追加されるたびに呼ばれる`mapEventToState`を定義しましょう。

```dart
@override
Stream<PostState> mapEventToState(PostEvent event) async* {
  final currentState = state;
  if (event is Fetch && !_hasReachedMax(currentState)) {
    try {
      if (currentState is PostUninitialized) {
        final posts = await _fetchPosts(0, 20);
        yield PostLoaded(posts: posts, hasReachedMax: false);
        return;
      }
      if (currentState is PostLoaded) {
        final posts =
            await _fetchPosts(currentState.posts.length, 20);
        yield posts.isEmpty
            ? currentState.copyWith(hasReachedMax: true)
            : PostLoaded(
                posts: currentState.posts + posts,
                hasReachedMax: false,
              );
      }
    } catch (_) {
      yield PostError();
    }
  }
}

bool _hasReachedMax(PostState state) =>
    state is PostLoaded && state.hasReachedMax;

Future<List<Post>> _fetchPosts(int startIndex, int limit) async {
  final response = await httpClient.get(
      'https://jsonplaceholder.typicode.com/posts?_start=$startIndex&_limit=$limit');
  if (response.statusCode == 200) {
    final data = json.decode(response.body) as List;
    return data.map((rawPost) {
      return Post(
        id: rawPost['id'],
        title: rawPost['title'],
        body: rawPost['body'],
      );
    }).toList();
  } else {
    throw Exception('error fetching posts');
  }
}
```

`PostBloc`は`Stream<PostState>`を返すので新しい state を返すたびに`yield`します。`Streams`や他のコアなコンセプトについては[core concepts](https://bloclibrary.dev/#/coreconcepts?id=streams)を確認しましょう。

これで`PostEvent`が追加され、それが`Fetch`であり、まだロードする投稿がある場合は`PostBloc`は次の 20 件をロードします。

今回使っている API は用意されているデータ数（100 件）を超えてデータをロードしようとすると空の配列を返すようになっているので、もしからの配列を受け取った場合は`hasReachedMax`を true にして`yield`します。

もし、投稿をロードする際にエラーが発生したら`PostError()`を`yield`します。

もし、投稿が取得できたら投稿が全て含まれている`PostLoaded()`を返します。

一つ改善の余地があるならば、`Event`が API を必要以上に何回も呼ばないように`debounce`するようにすることでしょう。これは`PostBloc`の`transform`メソッドを上書きすることでできます。

?> **メモ:** Transform を上書きすることで mapEventToState が呼ばれる前に Stream<Event> を返還することができます。これにより distinct()や debounceTime()などの処理をかけることができるようになります。

```dart
@override
Stream<Transition<PostEvent, PostState>> transformEvents(
  Stream<PostEvent> events,
  TransitionFunction<PostEvent, PostState> transitionFn,
) {
  return super.transformEvents(
    events.debounceTime(const Duration(milliseconds: 500)),
    transitionFn,
  );
}
```

完成した`PostBloc`はこのようになっているはずです:

```dart
import 'dart:async';
import 'dart:convert';

import 'package:meta/meta.dart';
import 'package:rxdart/rxdart.dart';
import 'package:http/http.dart' as http;
import 'package:bloc/bloc.dart';
import 'package:flutter_infinite_list/bloc/bloc.dart';
import 'package:flutter_infinite_list/models/models.dart';

class PostBloc extends Bloc<PostEvent, PostState> {
  final http.Client httpClient;

  PostBloc({@required this.httpClient});

  @override
  get initialState => PostUninitialized();

  @override
  Stream<Transition<PostEvent, PostState>> transformEvents(
    Stream<PostEvent> events,
    TransitionFunction<PostEvent, PostState> transitionFn,
  ) {
    return super.transformEvents(
      events.debounceTime(const Duration(milliseconds: 500)),
      transitionFn,
    );
  }

  @override
  Stream<PostState> mapEventToState(PostEvent event) async* {
    final currentState = state;
    if (event is Fetch && !_hasReachedMax(currentState)) {
      try {
        if (currentState is PostUninitialized) {
          final posts = await _fetchPosts(0, 20);
          yield PostLoaded(posts: posts, hasReachedMax: false);
          return;
        }
        if (currentState is PostLoaded) {
          final posts = await _fetchPosts(currentState.posts.length, 20);
          yield posts.isEmpty
              ? currentState.copyWith(hasReachedMax: true)
              : PostLoaded(
                  posts: currentState.posts + posts,
                  hasReachedMax: false,
                );
        }
      } catch (_) {
        yield PostError();
      }
    }
  }

  bool _hasReachedMax(PostState state) =>
      state is PostLoaded && state.hasReachedMax;

  Future<List<Post>> _fetchPosts(int startIndex, int limit) async {
    final response = await httpClient.get(
        'https://jsonplaceholder.typicode.com/posts?_start=$startIndex&_limit=$limit');
    if (response.statusCode == 200) {
      final data = json.decode(response.body) as List;
      return data.map((rawPost) {
        return Post(
          id: rawPost['id'],
          title: rawPost['title'],
          body: rawPost['body'],
        );
      }).toList();
    } else {
      throw Exception('error fetching posts');
    }
  }
}
```

完璧です！ビジネスロジックが実装し終わったのであとはプレゼンテーションレイヤーを作って終わります。

## プレゼンテーションレイヤー

`App`ウィジェットの中で`PostBloc`のインスタンスを子孫要素に渡すために`BlocProvider`を使います。加えて、最初の投稿データを引っ張ってきてくれるように`Fetch` event を渡しておきます。

```dart
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'package:flutter_bloc/flutter_bloc.dart';

import 'package:flutter_infinite_list/bloc/bloc.dart';

void main() {
  runApp(App());
}

class App extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Infinite Scroll',
      home: Scaffold(
        appBar: AppBar(
          title: Text('Posts'),
        ),
        body: BlocProvider(
          create: (context) =>
              PostBloc(httpClient: http.Client())..add(Fetch()),
          child: HomePage(),
        ),
      ),
    );
  }
}
```

次に`HomePage`ウィジェットを作り、`PostBloc`を紐づけます。

```dart
class HomePage extends StatefulWidget {
  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  final _scrollController = ScrollController();
  final _scrollThreshold = 200.0;
  PostBloc _postBloc;

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(_onScroll);
    _postBloc = BlocProvider.of<PostBloc>(context);
  }

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<PostBloc, PostState>(
      builder: (context, state) {
        if (state is PostUninitialized) {
          return Center(
            child: CircularProgressIndicator(),
          );
        }
        if (state is PostError) {
          return Center(
            child: Text('failed to fetch posts'),
          );
        }
        if (state is PostLoaded) {
          if (state.posts.isEmpty) {
            return Center(
              child: Text('no posts'),
            );
          }
          return ListView.builder(
            itemBuilder: (BuildContext context, int index) {
              return index >= state.posts.length
                  ? BottomLoader()
                  : PostWidget(post: state.posts[index]);
            },
            itemCount: state.hasReachedMax
                ? state.posts.length
                : state.posts.length + 1,
            controller: _scrollController,
          );
        }
      },
    );
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  void _onScroll() {
    final maxScroll = _scrollController.position.maxScrollExtent;
    final currentScroll = _scrollController.position.pixels;
    if (maxScroll - currentScroll <= _scrollThreshold) {
      _postBloc.add(Fetch());
    }
  }
}
```

?> `HomePage`は`ScrollController`を持たないといけないので`StatefulWidget`を継承させます。`initState`の中でユーザーが画面をスクロールすることに対してコードを実行できるように`ScrollController`にリスナーを貼ります。さらに、`BlocProvider.of<PostBloc>(context)`を使って`PostBloc`インスタンスにアクセスします。

build メソッドの中では`BlocBuilder`を使っています。`BlocBuilder`は[flutter_bloc package](https://pub.dev/packages/flutter_bloc)の中のウィジェットで、state に合わせて UI を変更することができます。`PostBloc`の state が変わるたびに builder 関数が新しい`PostState`で呼ばれます。

!> 忘れてはならないのが、StatefulWidget が dispose されるときに`ScrollController`も dispose することです。

ユーザーが画面をスクロールするたびに画面の下から今のポジション前の距離を計算し、それが`_scrollThreshold`以下であれば `Fetch` event を送り追加で投稿をロードします。

次に、ユーザーに現在投稿をロード中であることを知らせる`BottomLoader`ウィジェットを作成します。

```dart
class BottomLoader extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(
      alignment: Alignment.center,
      child: Center(
        child: SizedBox(
          width: 33,
          height: 33,
          child: CircularProgressIndicator(
            strokeWidth: 1.5,
          ),
        ),
      ),
    );
  }
}
```

最後に、一つ一つの投稿を描画する`PostWidget`ウィジェットを実装します。

```dart
class PostWidget extends StatelessWidget {
  final Post post;

  const PostWidget({Key key, @required this.post}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: Text(
        '${post.id}',
        style: TextStyle(fontSize: 10.0),
      ),
      title: Text(post.title),
      isThreeLine: true,
      subtitle: Text(post.body),
      dense: true,
    );
  }
}
```

ここまできたら、アプリを実行すると全てうまく動くはずです。ただ、もう一つやるべきことがあります。

一つ Bloc ライブラリーを使うことでついてくるおまけが、一箇所で全ての`Transition`にアクセスできるということです。

> 1 つの state から別の state に変化することを`Transition`と呼びます。

?> `Transition`には今の state, 呼ばれた event, そして次の state が含まれます。

今回作成したアプリでは 1 つの bloc しか使いませんでしたが、一般的なアプリでは複数の bloc を使いアプリの様々な状態を管理することがよくあります。

もし、全ての`Transitions`に対して何かを実行したい時は`BlocDelegate`を作成します。

```dart
import 'package:bloc/bloc.dart';

class SimpleBlocDelegate extends BlocDelegate {
  @override
  void onTransition(Bloc bloc, Transition transition) {
    super.onTransition(bloc, transition);
    print(transition);
  }
}
```

?> やることはただ`BlocDelegate`を継承し`onTransition`メソッドを上書きするだけです。

作成した Bloc に`SimpleBlocDelegate`を使用するようにするには main 関数を書き換えます。

```dart
void main() {
  BlocSupervisor.delegate = SimpleBlocDelegate();
  runApp(MyApp());
}
```

これでこのアプリの中の Bloc が新しい`Transition` を発するたびにコンソールにその旨を print できます。

?> 実際のアプリではただコンソールに print するだけでなく、アナリティクスツールなどにイベントを記録するために使います。

これで終わりです！Flutter と[flutter_bloc](https://pub.dev/packages/flutter_bloc)を使ってプレゼンテーションレイヤーと Business Logic が綺麗に分かれた無限リストを表示するアプリを作ることができました。

`HomePage`はどこからどうやって`Posts`がやってきているのかを知る由もありません。同じように`PostBloc`はどこでどのように`State`が使われているかを知る由もありません。ただ event を state に変換しているだけです。

このアプリのソースコードは[こちら](https://github.com/felangel/Bloc/tree/master/examples/flutter_infinite_list)から見ることができます。
