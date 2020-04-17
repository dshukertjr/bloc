# Flutter カウンターチュートリアル

![beginner](https://img.shields.io/badge/level-beginner-green.svg)

> このチュートリアルでは Bloc ライブラリーを使ってカウンターアプリを作ります。

![demo](../../assets/gifs/flutter_counter.gif)

## セットアップ

まず新規の Flutter プロジェクトを作るところから始めます

[script](../_snippets/flutter_counter_tutorial/flutter_create.sh.md ':include')

`pubspec.yaml`の中身をこちらに書き換えましょう

[pubspec.yaml](../_snippets/flutter_counter_tutorial/pubspec.yaml.md ':include')

それができたらパッケージをインストールしましょう

[script](../_snippets/flutter_counter_tutorial/flutter_packages_get.sh.md ':include')

今回作るカウンターアプリには加算と減算をするための二つのボタンと現在のカウントを表示するための`Text`ウィジェットがあります。まずは`CounterEvents`を作成するところから始めましょう。

## Counter Events

[counter_event.dart](../_snippets/flutter_counter_tutorial/counter_event.dart.md ':include')

## Counter States

今回のカウンターアプリの state はただの int なので、クラスを作る必要はありません。

## Counter Bloc

[counter_bloc.dart](../_snippets/flutter_counter_tutorial/counter_bloc.dart.md ':include')

?> **メモ**: `CounterBloc`の定義を見ただけで`CounterEvents`を入力し int を出力することがわかりますね。

## カウンターアプリ

ここまでくれば`CounterBloc`の実装は完了です。あとは Flutter のアプリを作成しましょう。

[main.dart](../_snippets/flutter_counter_tutorial/main.dart.md ':include')

?> **メモ**: `CounterBloc`のインスタンスを`CounterPage`のどこからでもアクセスできるようにするために`flutter_bloc`パッケージの`BlocProvider`を使います。 `BlocProvider`は`CounterBloc`を自動的に close してくれるので`StatefulWidget`のライフサイクルメソッドを使う必要がありません。

## カウンターページ

あとはカウンターページを作成するだけです。

[counter_page.dart](../_snippets/flutter_counter_tutorial/counter_page.dart.md ':include')

?> **メモ**: `CounterPage`を`BlocProvider`で囲ったので`BlocProvider.of<CounterBloc>(context)`を使って`CounterBloc`インスタンスにアクセスすることができます。

?> **メモ**: `flutter_bloc`パッケージの`BlocBuilder`ウィジェットを使って state の変化（カウンターの値の変化）に応じて UI を変化させられるようにしています。

?> **メモ**: `BlocBuilder` takes an optional `bloc` parameter but we can specify the type of the bloc and the type of the state and `BlocBuilder` will find the bloc automatically so we don't need to explicity use `BlocProvider.of<CounterBloc>(context)`.

!> `BlocBuilder`の bloc 引数を渡すのはそのウィジェットにのみ Bloc を渡したい場合かつその時の`BuildContext`先祖ウィジェットにその bloc 用の`BlocProvider`が無い場合のみにしましょう。

これでプレゼンテーションレイヤーとビジネスロジックレイヤーを分けることに成功しました。`CounterPage`はユーザーがボタンを押した時に何が起こるかなんて知る由もありません。ただ`CounterBloc`に event を送るだけです。逆も然りで、`CounterBloc`は state (カウンターの数値)がどのように使われているかなんて知る由もありません。ただシンプルに受け取った`CounterEvents`を元に int を返しているだけです。

アプリは`flutter run`コマンドを使ってシミュレーター/エミュレーターや実機で動かすことができます。

このアプリのソースコードは[こちら](https://github.com/felangel/Bloc/tree/master/packages/flutter_bloc/example)に置いてあります。
