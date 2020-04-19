# Flutter タイマーチュートリアル

![beginner](https://img.shields.io/badge/level-beginner-green.svg)

> このチュートリアルではキッチンタイマーのようなタイマーアプリを bloc ライブラリーを使ってどのように開発するかを見ていきます。完成品はこのような形になります。

![demo](../assets/gifs/flutter_timer.gif)

## 準備

まずは新規の Flutter プロジェクトを作りましょう

```sh
flutter create flutter_timer
```

pubspec.yaml の中身を下記のように書き換えます：

```yaml
name: flutter_timer
description: A new Flutter project.

version: 1.0.0+1

environment:
  sdk: '>=2.6.0 <3.0.0'

dependencies:
  flutter:
    sdk: flutter
  flutter_bloc: ^3.2.0
  equatable: ^1.0.0
  wave: ^0.0.8

flutter:
  uses-material-design: true
```

?> **メモ:** [flutter_bloc](https://pub.dev/packages/flutter_bloc)と[equatable](https://pub.dev/packages/equatable)そして[wave](https://pub.dev/packages/wave)パッケージをこのアプリでは使います。

次に`flutter packages get`を実行しパッケージをインストールします。

## Ticker

> Ticker がこのアプリのデータソースとなります。毎秒時間のデータが乗った Stream を公開し、それに subscribe することで時間の経過に応じてコードを実行できます。

まずは`ticker.dart`を作成しましょう。

```dart
class Ticker {
  Stream<int> tick({int ticks}) {
    return Stream.periodic(Duration(seconds: 1), (x) => ticks - x - 1)
        .take(ticks);
  }
}
```

`Ticker`クラスは何秒間カウントダウンしたいか（何回 tick したいか）を引数に取り、１秒間に１度、残り時間を乗せた Stream を公開しているだけです。

次にこの`Ticker`を使って`TimerBloc`を作りましょう。

## Timer Bloc

### TimerState

まずは`TimerBloc`が取れる`TimerStates`を定義しましょう。

`TimerBloc`の state はこれらのいずれかになります：

- Ready — カウントダウンの準備ができている状態。
- Running —  特定の時間からカウントダウンをしている真っ最中。
- Paused —  特定の時間でカウントダウンを一時停止している状態。
- Finished —  残り時間が 0 になりカウントダウンを完了した状態。

Each of these states will have an implication on what the user sees. For example:
これらの state は全て固有のアプリに対するアクションを持っています。例えば:

- アプリが“ready”の時はユーザーはタイマーをスタートさせることができます。
- アプリが“running”の時はユーザーはアプリを一時停止やリセットでき、残り時間を確認することができます。
- アプリが“paused”の時はユーザーはカウントダウンを再開したり、リセットしたりできます。
- アプリが“finished”の時はユーザーはタイマーをリセットすることができます。

Bloc 系のファイルを一箇所にまとめられるように bloc というディレクトリーを作り、そこにこのように`bloc/timer_state.dart`入れましょう。

?> **豆知識:** [IntelliJ](https://plugins.jetbrains.com/plugin/12129-bloc-code-generator)か[VSCode](https://marketplace.visualstudio.com/items?itemName=FelixAngelov.bloc)のエクステンションを使うことで関連ファイルを一括で自動生成することができます。

```dart
import 'package:equatable/equatable.dart';

abstract class TimerState extends Equatable {
  final int duration;

  const TimerState(this.duration);

  @override
  List<Object> get props => [duration];
}

class Ready extends TimerState {
  const Ready(int duration) : super(duration);

  @override
  String toString() => 'Ready { duration: $duration }';
}

class Paused extends TimerState {
  const Paused(int duration) : super(duration);

  @override
  String toString() => 'Paused { duration: $duration }';
}

class Running extends TimerState {
  const Running(int duration) : super(duration);

  @override
  String toString() => 'Running { duration: $duration }';
}

class Finished extends TimerState {
  const Finished() : super(0);
}
```

注目して欲しいのは全ての`TimerState`はベースクラスである`TimerState`を継承しているので全て duration プロパティを持っています。これは`TimerBloc`がどの state だったとしてもタイマーの残り時間は見れるようにしておきたいからです。

次に`TimerBloc`に渡される`TimerEvent`を定義していきましょう。

### TimerEvent

Our `TimerBloc` will need to know how to process the following events:
`TimerBloc`は下記の event を処理する必要があるでしょう：

- Start — TimerBloc にカウントダウンの開始を知らせる。
- Pause — TimerBloc にカウントダウンの一時停止を知らせる。
- Resume — TimerBloc にカウントダウンの再開を知らせる。
- Reset — TimerBloc に元の状態にリセットするように知らせる。
- Tick — TimerBloc に tick が起きた（カウントダウンが発生し残り時間が減少した）ことを知らせ state をアップデートさせる。

もし[IntelliJ](https://plugins.jetbrains.com/plugin/12129-bloc-code-generator)か[VSCode](https://marketplace.visualstudio.com/items?itemName=FelixAngelov.bloc)のエクステンションを使っていない場合は`bloc/timer_event.dart`を作成してください。

Event を定義していきましょう。

```dart
import 'package:equatable/equatable.dart';
import 'package:meta/meta.dart';

abstract class TimerEvent extends Equatable {
  const TimerEvent();

  @override
  List<Object> get props => [];
}

class Start extends TimerEvent {
  final int duration;

  const Start({@required this.duration});

  @override
  String toString() => "Start { duration: $duration }";
}

class Pause extends TimerEvent {}

class Resume extends TimerEvent {}

class Reset extends TimerEvent {}

class Tick extends TimerEvent {
  final int duration;

  const Tick({@required this.duration});

  @override
  List<Object> get props => [duration];

  @override
  String toString() => "Tick { duration: $duration }";
}
```

次に`TimerBloc`を実装していきましょう！

### TimerBloc

もしまだ作っていない場合は`bloc/timer_bloc.dart`を作り、からの`TimerBloc`を作りましょう。

```dart
import 'package:bloc/bloc.dart';
import 'package:flutter_timer/bloc/bloc.dart';

class TimerBloc extends Bloc<TimerEvent, TimerState> {
  @override
  TimerState get initialState => // TODO: 初期 state を作る

  @override
  Stream<TimerState> mapEventToState(
    TimerEvent event,
  ) async* {
    // TODO: EventToState メソッドを定義する
  }
}
```

一番最初にするのは`TimerBloc`の`initialState`を定義することです。今回の場合は`TimerBloc`に最初に 1 分間（60 秒間）がセットされた`Ready`state を初期 state として与えましょう。

```dart
import 'package:bloc/bloc.dart';
import 'package:flutter_timer/bloc/bloc.dart';

class TimerBloc extends Bloc<TimerEvent, TimerState> {
  final int _duration = 60;

  @override
  TimerState get initialState => Ready(_duration);

  @override
  Stream<TimerState> mapEventToState(
    TimerEvent event,
  ) async* {
    // TODO: EventToState メソッドを定義する
  }
}
```

次に`Ticker`を引数として受け取れるようにしましょう。

```dart
import 'dart:async';
import 'package:meta/meta.dart';
import 'package:bloc/bloc.dart';
import 'package:flutter_timer/bloc/bloc.dart';
import 'package:flutter_timer/ticker.dart';

class TimerBloc extends Bloc<TimerEvent, TimerState> {
  final Ticker _ticker;
  final int _duration = 60;

  StreamSubscription<int> _tickerSubscription;

  TimerBloc({@required Ticker ticker})
      : assert(ticker != null),
        _ticker = ticker;

  @override
  TimerState get initialState => Ready(_duration);

  @override
  Stream<TimerState> mapEventToState(
    TimerEvent event,
  ) async* {
    // TODO: EventToState メソッドを定義する
  }
}
```

さらに`Ticker`の状態を持っておくために`StreamSubscription`も定義します。

ここまできたらあとは`mapEventToState`を完成させるだけです。可読性を高めるために私は一つ一つの event ハンドラーを別々のヘルパーメソッドに切り分けるのが好きです。まずは`Start` event から作りましょう。

```dart
import 'dart:async';
import 'package:meta/meta.dart';
import 'package:bloc/bloc.dart';
import 'package:flutter_timer/bloc/bloc.dart';
import 'package:flutter_timer/ticker.dart';

class TimerBloc extends Bloc<TimerEvent, TimerState> {
  final Ticker _ticker;
  final int _duration = 60;

  StreamSubscription<int> _tickerSubscription;

  TimerBloc({@required Ticker ticker})
      : assert(ticker != null),
        _ticker = ticker;

  @override
  TimerState get initialState => Ready(_duration);

  @override
  Stream<TimerState> mapEventToState(
    TimerEvent event,
  ) async* {
    if (event is Start) {
      yield* _mapStartToState(event);
    }
  }

  @override
  Future<void> close() {
    _tickerSubscription?.cancel();
    return super.close();
  }

  Stream<TimerState> _mapStartToState(Start start) async* {
     yield Running(start.duration);
    _tickerSubscription?.cancel();
    _tickerSubscription = _ticker
        .tick(ticks: start.duration)
        .listen((duration) => add(Tick(duration: duration)));
  }
}
```

もし`TimerBloc`が`Start` event を受け取ったら開始時点での時間を入れて`Running` state を返します。さらに、もうすでに`_tickerSubscription`が存在する場合は一度 subscription をキャンセルしメモリーを解放してあげる必要があります。加えて`TimerBloc`が閉じられた時に`_tickerSubscription`を cancel してメモリーを解放できるように`TimerBloc`の`close`メソッドを上書きする必要があります。最後に、`_ticker.tick`にリスナーを貼り、tick （毎秒のカウントダウン）に対してその時点での残り時間を含んだ`Tick` event を発する必要があります。

次に`Tick` event ハンドラーを作成しましょう。

```dart
import 'dart:async';
import 'package:meta/meta.dart';
import 'package:bloc/bloc.dart';
import 'package:flutter_timer/bloc/bloc.dart';
import 'package:flutter_timer/ticker.dart';

class TimerBloc extends Bloc<TimerEvent, TimerState> {
  final Ticker _ticker;
  final int _duration = 60;

  StreamSubscription<int> _tickerSubscription;

  TimerBloc({@required Ticker ticker})
      : assert(ticker != null),
        _ticker = ticker;

  @override
  TimerState get initialState => Ready(_duration);

  @override
  Stream<TimerState> mapEventToState(
    TimerEvent event,
  ) async* {
    if (event is Start) {
      yield* _mapStartToState(event);
    } else if (event is Tick) {
      yield* _mapTickToState(event);
    }
  }

  @override
  Future<void> close() {
    _tickerSubscription?.cancel();
    return super.close();
  }

  Stream<TimerState> _mapStartToState(Start start) async* {
     yield Running(start.duration);
    _tickerSubscription?.cancel();
    _tickerSubscription = _ticker
        .tick(ticks: start.duration)
        .listen((duration) => add(Tick(duration: duration)));
  }

  Stream<TimerState> _mapTickToState(Tick tick) async* {
    yield tick.duration > 0 ? Running(tick.duration) : Finished();
  }
}
```

毎秒`Tick` event を受け取り、もし残り時間が 0 以上なら `Running` state を新しい残り時間を含めて返します。それ以外の場合、つまり残り時間が 0 の場合は`Finished` state を返します。

それができたら今度は`Pause` event ハンドラーを作りましょう。

```dart
import 'dart:async';
import 'package:meta/meta.dart';
import 'package:bloc/bloc.dart';
import 'package:flutter_timer/bloc/bloc.dart';
import 'package:flutter_timer/ticker.dart';

class TimerBloc extends Bloc<TimerEvent, TimerState> {
  final Ticker _ticker;
  final int _duration = 60;

  StreamSubscription<int> _tickerSubscription;

  TimerBloc({@required Ticker ticker})
      : assert(ticker != null),
        _ticker = ticker;

  @override
  TimerState get initialState => Ready(_duration);

  @override
  Stream<TimerState> mapEventToState(
    TimerEvent event,
  ) async* {
    if (event is Start) {
      yield* _mapStartToState(event);
    } else if (event is Pause) {
      yield* _mapPauseToState(event);
    } else if (event is Tick) {
      yield* _mapTickToState(event);
    }
  }

  @override
  Future<void> close() {
    _tickerSubscription?.cancel();
    return super.close();
  }

  Stream<TimerState> _mapStartToState(Start start) async* {
     yield Running(start.duration);
    _tickerSubscription?.cancel();
    _tickerSubscription = _ticker
        .tick(ticks: start.duration)
        .listen((duration) => add(Tick(duration: duration)));
  }

  Stream<TimerState> _mapPauseToState(Pause pause) async* {
    if (state is Running) {
      _tickerSubscription?.pause();
      yield Paused(state.duration);
    }
  }

  Stream<TimerState> _mapTickToState(Tick tick) async* {
    yield tick.duration > 0 ? Running(tick.duration) : Finished();
  }
}
```

`_mapPauseToState`の中ではもし`TimerBloc`の`state`が`Running`だったら`_tickerSubscription`を一時停止し、今の時間を含んだ`Paused` state を返します。

次はタイマーを再開できるように`Resume` event ハンドラーを作っていきましょう。

```dart
import 'dart:async';
import 'package:meta/meta.dart';
import 'package:bloc/bloc.dart';
import 'package:flutter_timer/bloc/bloc.dart';
import 'package:flutter_timer/ticker.dart';

class TimerBloc extends Bloc<TimerEvent, TimerState> {
  final Ticker _ticker;
  final int _duration = 60;

  StreamSubscription<int> _tickerSubscription;

  TimerBloc({@required Ticker ticker})
      : assert(ticker != null),
        _ticker = ticker;

  @override
  TimerState get initialState => Ready(_duration);

  @override
  Stream<TimerState> mapEventToState(
    TimerEvent event,
  ) async* {
    if (event is Start) {
      yield* _mapStartToState(event);
    } else if (event is Pause) {
      yield* _mapPauseToState(event);
    } else if (event is Resume) {
      yield* _mapResumeToState(event);
    } else if (event is Tick) {
      yield* _mapTickToState(event);
    }
  }

  @override
  Future<void> close() {
    _tickerSubscription?.cancel();
    return super.close();
  }

  Stream<TimerState> _mapStartToState(Start start) async* {
     yield Running(start.duration);
    _tickerSubscription?.cancel();
    _tickerSubscription = _ticker
        .tick(ticks: start.duration)
        .listen((duration) => add(Tick(duration: duration)));
  }

  Stream<TimerState> _mapPauseToState(Pause pause) async* {
    if (state is Running) {
      _tickerSubscription?.pause();
      yield Paused(state.duration);
    }
  }

  Stream<TimerState> _mapResumeToState(Resume resume) async* {
    if (state is Paused) {
      _tickerSubscription?.resume();
      yield Running(state.duration);
    }
  }

  Stream<TimerState> _mapTickToState(Tick tick) async* {
    yield tick.duration > 0 ? Running(tick.duration) : Finished();
  }
}
```

`Resume` event ハンドラーは`Pause` event ハンドラーととてもよく似た構造になっています。もし`TimerBloc`の`state`が`Paused`で`Resume` event を受け取ると、一時停止してた`_tickerSubscription`を再開し、その時点での残り時間を含んだ`Running` state を返します。

最後に`Reset` event ハンドラーを実装しましょう。

```dart
import 'dart:async';
import 'package:meta/meta.dart';
import 'package:bloc/bloc.dart';
import 'package:flutter_timer/bloc/bloc.dart';
import 'package:flutter_timer/ticker.dart';

class TimerBloc extends Bloc<TimerEvent, TimerState> {
  final Ticker _ticker;
  final int _duration = 60;

  StreamSubscription<int> _tickerSubscription;

  TimerBloc({@required Ticker ticker})
      : assert(ticker != null),
        _ticker = ticker;

  @override
  TimerState get initialState => Ready(_duration);

  @override
  void onTransition(Transition<TimerEvent, TimerState> transition) {
    super.onTransition(transition);
    print(transition);
  }

  @override
  Stream<TimerState> mapEventToState(
    TimerEvent event,
  ) async* {
    if (event is Start) {
      yield* _mapStartToState(event);
    } else if (event is Pause) {
      yield* _mapPauseToState(event);
    } else if (event is Resume) {
      yield* _mapResumeToState(event);
    } else if (event is Reset) {
      yield* _mapResetToState(event);
    } else if (event is Tick) {
      yield* _mapTickToState(event);
    }
  }

  @override
  Future<void> close() {
    _tickerSubscription?.cancel();
    return super.close();
  }

  Stream<TimerState> _mapStartToState(Start start) async* {
     yield Running(start.duration);
    _tickerSubscription?.cancel();
    _tickerSubscription = _ticker
        .tick(ticks: start.duration)
        .listen((duration) => add(Tick(duration: duration)));
  }

  Stream<TimerState> _mapPauseToState(Pause pause) async* {
    final state = currentState;
    if (state is Running) {
      _tickerSubscription?.pause();
      yield Paused(state.duration);
    }
  }

  Stream<TimerState> _mapResumeToState(Resume resume) async* {
    final state = currentState;
    if (state is Paused) {
      _tickerSubscription?.resume();
      yield Running(state.duration);
    }
  }

  Stream<TimerState> _mapResetToState(Reset reset) async* {
    _tickerSubscription?.cancel();
    yield Ready(_duration);
  }

  Stream<TimerState> _mapTickToState(Tick tick) async* {
    yield tick.duration > 0 ? Running(tick.duration) : Finished();
  }
}
```

`TimerBloc`が`Reset` event を受け取ると、残り時間を毎秒返すのをやめるために`_tickerSubscription`を cancel し、最初に設定されていた残り時間を含んだ`Ready` state を返します。

`TimerBloc`はこれで完成です。今度は UI 関係の実装を行いましょう。

## アプリの UI

### MyApp

まずは`main.dart`の中身を全部消去し、今回のアプリの大元となる`MyApp`を作りましょう。

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_timer/bloc/bloc.dart';
import 'package:flutter_timer/ticker.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(
        primaryColor: Color.fromRGBO(109, 234, 255, 1),
        accentColor: Color.fromRGBO(72, 74, 126, 1),
        brightness: Brightness.dark,
      ),
      title: 'Flutter Timer',
      home: BlocProvider(
        create: (context) => TimerBloc(ticker: Ticker()),
        child: Timer(),
      ),
    );
  }
}
```

`MyApp`は`StatelessWidget`で、`TimerBloc`の初期化をしています。加えて、`BlocProvider`を使って`TimerBloc`のインスタンスを子孫ウィジェットたちにに渡しています。

次に`Timer`の実装をしていきましょう。

### Timer

`Timer`は現在の残り時間を表示するだけでなく、state に合わせて開始、停止、リセットボタンを表示するウィジェットです。

```dart
class Timer extends StatelessWidget {
  static const TextStyle timerTextStyle = TextStyle(
    fontSize: 60,
    fontWeight: FontWeight.bold,
  );

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Flutter Timer')),
      body: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        crossAxisAlignment: CrossAxisAlignment.center,
        children: <Widget>[
          Padding(
            padding: EdgeInsets.symmetric(vertical: 100.0),
            child: Center(
              child: BlocBuilder<TimerBloc, TimerState>(
                builder: (context, state) {
                  final String minutesStr = ((state.duration / 60) % 60)
                      .floor()
                      .toString()
                      .padLeft(2, '0');
                  final String secondsStr =
                      (state.duration % 60).floor().toString().padLeft(2, '0');
                  return Text(
                    '$minutesStr:$secondsStr',
                    style: Timer.timerTextStyle,
                  );
                },
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

ここまでは`TimerBloc`にアクセスするために`BlocProvider`を使い、`TimerState`の変化に応じて UI を変化させるために`BlocBuilder`を使っています。

次に state に合わせて表示するボタンを変える`Actions`ウィジェットを実装します。

### Actions

```dart
class Actions extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
      children: _mapStateToActionButtons(
        timerBloc: BlocProvider.of<TimerBloc>(context),
      ),
    );
  }

  List<Widget> _mapStateToActionButtons({
    TimerBloc timerBloc,
  }) {
    final TimerState currentState = timerBloc.state;
    if (currentState is Ready) {
      return [
        FloatingActionButton(
          child: Icon(Icons.play_arrow),
          onPressed: () =>
              timerBloc.add(Start(duration: currentState.duration)),
        ),
      ];
    }
    if (currentState is Running) {
      return [
        FloatingActionButton(
          child: Icon(Icons.pause),
          onPressed: () => timerBloc.add(Pause()),
        ),
        FloatingActionButton(
          child: Icon(Icons.replay),
          onPressed: () => timerBloc.add(Reset()),
        ),
      ];
    }
    if (currentState is Paused) {
      return [
        FloatingActionButton(
          child: Icon(Icons.play_arrow),
          onPressed: () => timerBloc.add(Resume()),
        ),
        FloatingActionButton(
          child: Icon(Icons.replay),
          onPressed: () => timerBloc.add(Reset()),
        ),
      ];
    }
    if (currentState is Finished) {
      return [
        FloatingActionButton(
          child: Icon(Icons.replay),
          onPressed: () => timerBloc.add(Reset()),
        ),
      ];
    }
    return [];
  }
}
```

`Actions`ウィジェットは`BlocProvider`を使って`TimerBloc`にアクセスしているだけでなんの変哲も無い`StatelessWidget`ウィジェットです。`TimerBloc`の state に応じて違う種類の`FloatingActionButton`を返してくれます。それぞれの`FloatingActionButton`は`onPressed`のなかに`TimerBloc`に対して event を送っています。

次に`Actions`と`Timer`ウィジェットを紐づけていきましょう。

```dart
class Timer extends StatelessWidget {
  static const TextStyle timerTextStyle = TextStyle(
    fontSize: 60,
    fontWeight: FontWeight.bold,
  );

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Flutter Timer')),
      body: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        crossAxisAlignment: CrossAxisAlignment.center,
        children: <Widget>[
          Padding(
            padding: EdgeInsets.symmetric(vertical: 100.0),
            child: Center(
              child: BlocBuilder<TimerBloc, TimerState>(
                builder: (context, state) {
                  final String minutesStr = ((state.duration / 60) % 60)
                      .floor()
                      .toString()
                      .padLeft(2, '0');
                  final String secondsStr =
                      (state.duration % 60).floor().toString().padLeft(2, '0');
                  return Text(
                    '$minutesStr:$secondsStr',
                    style: Timer.timerTextStyle,
                  );
                },
              ),
            ),
          ),
          BlocBuilder<TimerBloc, TimerState>(
            condition: (previousState, state) =>
                state.runtimeType != previousState.runtimeType,
            builder: (context, state) => Actions(),
          ),
        ],
      ),
    );
  }
}
```

We added another `BlocBuilder` which will render the `Actions` widget; however, this time we’re using a newly introduced [flutter_bloc](https://pub.dev/packages/flutter_bloc) feature to control how frequently the `Actions` widget is rebuilt (introduced in `v0.15.0`).

If you want fine-grained control over when the `builder` function is called you can provide an optional `condition` to `BlocBuilder`. The `condition` takes the previous bloc state and current bloc state and returns a `boolean`. If `condition` returns `true`, `builder` will be called with `state` and the widget will rebuild. If `condition` returns `false`, `builder` will not be called with `state` and no rebuild will occur.

In this case, we don’t want the `Actions` widget to be rebuilt on every tick because that would be inefficient. Instead, we only want `Actions` to rebuild if the `runtimeType` of the `TimerState` changes (Ready => Running, Running => Paused, etc...).

As a result, if we randomly colored the widgets on every rebuild, it would look like:

![BlocBuilder condition demo](https://cdn-images-1.medium.com/max/1600/1*YyjpH1rcZlYWxCX308l_Ew.gif)

?> **Notice:** Even though the `Text` widget is rebuilt on every tick, we only rebuild the `Actions` if they need to be rebuilt.

Lastly, we need to add the super cool wave background using the [wave](https://pub.dev/packages/wave) package.

### Waves Background

```dart
import 'package:flutter/material.dart';
import 'package:wave/wave.dart';
import 'package:wave/config.dart';

class Background extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return WaveWidget(
      config: CustomConfig(
        gradients: [
          [
            Color.fromRGBO(72, 74, 126, 1),
            Color.fromRGBO(125, 170, 206, 1),
            Color.fromRGBO(184, 189, 245, 0.7)
          ],
          [
            Color.fromRGBO(72, 74, 126, 1),
            Color.fromRGBO(125, 170, 206, 1),
            Color.fromRGBO(172, 182, 219, 0.7)
          ],
          [
            Color.fromRGBO(72, 73, 126, 1),
            Color.fromRGBO(125, 170, 206, 1),
            Color.fromRGBO(190, 238, 246, 0.7)
          ],
        ],
        durations: [19440, 10800, 6000],
        heightPercentages: [0.03, 0.01, 0.02],
        gradientBegin: Alignment.bottomCenter,
        gradientEnd: Alignment.topCenter,
      ),
      size: Size(double.infinity, double.infinity),
      waveAmplitude: 25,
      backgroundColor: Colors.blue[50],
    );
  }
}
```

### Putting it all together

Our finished, `main.dart` should look like:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_timer/bloc/bloc.dart';
import 'package:flutter_timer/ticker.dart';
import 'package:wave/wave.dart';
import 'package:wave/config.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(
        primaryColor: Color.fromRGBO(109, 234, 255, 1),
        accentColor: Color.fromRGBO(72, 74, 126, 1),
        brightness: Brightness.dark,
      ),
      title: 'Flutter Timer',
      home: BlocProvider(
        create: (context) => TimerBloc(ticker: Ticker()),
        child: Timer(),
      ),
    );
  }
}

class Timer extends StatelessWidget {
  static const TextStyle timerTextStyle = TextStyle(
    fontSize: 60,
    fontWeight: FontWeight.bold,
  );

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Flutter Timer')),
      body: Stack(
        children: [
          Background(),
          Column(
            mainAxisAlignment: MainAxisAlignment.center,
            crossAxisAlignment: CrossAxisAlignment.center,
            children: <Widget>[
              Padding(
                padding: EdgeInsets.symmetric(vertical: 100.0),
                child: Center(
                  child: BlocBuilder<TimerBloc, TimerState>(
                    builder: (context, state) {
                      final String minutesStr = ((state.duration / 60) % 60)
                          .floor()
                          .toString()
                          .padLeft(2, '0');
                      final String secondsStr = (state.duration % 60)
                          .floor()
                          .toString()
                          .padLeft(2, '0');
                      return Text(
                        '$minutesStr:$secondsStr',
                        style: Timer.timerTextStyle,
                      );
                    },
                  ),
                ),
              ),
              BlocBuilder<TimerBloc, TimerState>(
                condition: (previousState, currentState) =>
                    currentState.runtimeType != previousState.runtimeType,
                builder: (context, state) => Actions(),
              ),
            ],
          ),
        ],
      ),
    );
  }
}

class Actions extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
      children: _mapStateToActionButtons(
        timerBloc: BlocProvider.of<TimerBloc>(context),
      ),
    );
  }

  List<Widget> _mapStateToActionButtons({
    TimerBloc timerBloc,
  }) {
    final TimerState currentState = timerBloc.state;
    if (currentState is Ready) {
      return [
        FloatingActionButton(
          child: Icon(Icons.play_arrow),
          onPressed: () =>
              timerBloc.add(Start(duration: currentState.duration)),
        ),
      ];
    }
    if (currentState is Running) {
      return [
        FloatingActionButton(
          child: Icon(Icons.pause),
          onPressed: () => timerBloc.add(Pause()),
        ),
        FloatingActionButton(
          child: Icon(Icons.replay),
          onPressed: () => timerBloc.add(Reset()),
        ),
      ];
    }
    if (currentState is Paused) {
      return [
        FloatingActionButton(
          child: Icon(Icons.play_arrow),
          onPressed: () => timerBloc.add(Resume()),
        ),
        FloatingActionButton(
          child: Icon(Icons.replay),
          onPressed: () => timerBloc.add(Reset()),
        ),
      ];
    }
    if (currentState is Finished) {
      return [
        FloatingActionButton(
          child: Icon(Icons.replay),
          onPressed: () => timerBloc.add(Reset()),
        ),
      ];
    }
    return [];
  }
}

class Background extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return WaveWidget(
      config: CustomConfig(
        gradients: [
          [
            Color.fromRGBO(72, 74, 126, 1),
            Color.fromRGBO(125, 170, 206, 1),
            Color.fromRGBO(184, 189, 245, 0.7)
          ],
          [
            Color.fromRGBO(72, 74, 126, 1),
            Color.fromRGBO(125, 170, 206, 1),
            Color.fromRGBO(172, 182, 219, 0.7)
          ],
          [
            Color.fromRGBO(72, 73, 126, 1),
            Color.fromRGBO(125, 170, 206, 1),
            Color.fromRGBO(190, 238, 246, 0.7)
          ],
        ],
        durations: [19440, 10800, 6000],
        heightPercentages: [0.03, 0.01, 0.02],
        gradientBegin: Alignment.bottomCenter,
        gradientEnd: Alignment.topCenter,
      ),
      size: Size(double.infinity, double.infinity),
      waveAmplitude: 25,
      backgroundColor: Colors.blue[50],
    );
  }
}
```

That’s all there is to it! At this point we have a pretty solid timer application which efficiently rebuilds only widgets that need to be rebuilt.

The full source for this example can be found [here](https://github.com/felangel/Bloc/tree/master/examples/flutter_timer).
