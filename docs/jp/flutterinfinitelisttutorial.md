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

Our `PostBloc` will `yield` whenever there is a new state because it returns a `Stream<PostState>`. Check out [core concepts](https://bloclibrary.dev/#/coreconcepts?id=streams) for more information about `Streams` and other core concepts.

Now every time a `PostEvent` is added, if it is a `Fetch` event and there are more posts to fetch, our `PostBloc` will fetch the next 20 posts.

The API will return an empty array if we try to fetch beyond the maximum number of posts (100), so if we get back an empty array, our bloc will `yield` the currentState except we will set `hasReachedMax` to true.

If we cannot retrieve the posts, we throw an exception and `yield` `PostError()`.

If we can retrieve the posts, we return `PostLoaded()` which takes the entire list of posts.

One optimization we can make is to `debounce` the `Events` in order to prevent spamming our API unnecessarily. We can do this by overriding the `transform` method in our `PostBloc`.

?> **Note:** Overriding transform allows us to transform the Stream<Event> before mapEventToState is called. This allows for operations like distinct(), debounceTime(), etc... to be applied.

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

Our finished `PostBloc` should now look like this:

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

Don't forget to update `bloc/bloc.dart` to include our `PostBloc`!

```dart
export './post_bloc.dart';
export './post_event.dart';
export './post_state.dart';
```

Great! Now that we’ve finished implementing the business logic all that’s left to do is implement the presentation layer.

## Presentation Layer

In our `main.dart` we can start by implementing our main function and calling `runApp` to render our root widget.

In our `App` widget, we use `BlocProvider` to create and provide an instance of `PostBloc` to the subtree. Also, we add a `Fetch` event so that when the app loads, it requests the initial batch of Posts.

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

Next, we need to implement our `HomePage` widget which will present our posts and hook up to our `PostBloc`.

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

?> `HomePage` is a `StatefulWidget` because it will need to maintain a `ScrollController`. In `initState`, we add a listener to our `ScrollController` so that we can respond to scroll events. We also access our `PostBloc` instance via `BlocProvider.of<PostBloc>(context)`.

Moving along, our build method returns a `BlocBuilder`. `BlocBuilder` is a Flutter widget from the [flutter_bloc package](https://pub.dev/packages/flutter_bloc) which handles building a widget in response to new bloc states. Any time our `PostBloc` state changes, our builder function will be called with the new `PostState`.

!> We need to remember to clean up after ourselves and dispose of our `ScrollController` when the StatefulWidget is disposed.

Whenever the user scrolls, we calculate how far away from the bottom of the page they are and if the distance is ≤ our `_scrollThreshold` we add a `Fetch` event in order to load more posts.

Next, we need to implement our `BottomLoader` widget which will indicate to the user that we are loading more posts.

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

Lastly, we need to implement our `PostWidget` which will render an individual Post.

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

At this point, we should be able to run our app and everything should work; however, there’s one more thing we can do.

One added bonus of using the bloc library is that we can have access to all `Transitions` in one place.

> The change from one state to another is called a `Transition`.

?> A `Transition` consists of the current state, the event, and the next state.

Even though in this application we only have one bloc, it's fairly common in larger applications to have many blocs managing different parts of the application's state.

If we want to be able to do something in response to all `Transitions` we can simply create our own `BlocDelegate`.

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

?> All we need to do is extend `BlocDelegate` and override the `onTransition` method.

In order to tell Bloc to use our `SimpleBlocDelegate`, we just need to tweak our main function.

```dart
void main() {
  BlocSupervisor.delegate = SimpleBlocDelegate();
  runApp(MyApp());
}
```

Now when we run our application, every time a Bloc `Transition` occurs we can see the transition printed to the console.

?> In practice, you can create different `BlocDelegates` and because every state change is recorded, we are able to very easily instrument our applications and track all user interactions and state changes in one place!

That’s all there is to it! We’ve now successfully implemented an infinite list in flutter using the [bloc](https://pub.dev/packages/bloc) and [flutter_bloc](https://pub.dev/packages/flutter_bloc) packages and we’ve successfully separated our presentation layer from our business logic.

Our `HomePage` has no idea where the `Posts` are coming from or how they are being retrieved. Conversely, our `PostBloc` has no idea how the `State` is being rendered, it simply converts events into states.

The full source for this example can be found [here](https://github.com/felangel/Bloc/tree/master/examples/flutter_infinite_list).
