# Flutter ログインチュートリアル

![intermediate](https://img.shields.io/badge/level-intermediate-orange.svg)

> 今回のチュートリアルでは Flutter と Bloc ライブラリーを使ってログインフローを作っていきます。

![demo](../../assets/gifs/flutter_login.gif)

## 準備

まずは新規の Flutter プロジェクトを作りましょう。

```bash
flutter create flutter_login
```

それができたら`pubspec.yaml`の中身を下記の通り書き換えましょう。

```yaml
name: flutter_login
description: A new Flutter project.
version: 1.0.0+1

environment:
  sdk: '>=2.6.0 <3.0.0'

dependencies:
  flutter:
    sdk: flutter
  flutter_bloc: ^3.2.0
  meta: ^1.1.6
  equatable: ^1.0.0

dev_dependencies:
  flutter_test:
    sdk: flutter

flutter:
  uses-material-design: true
```

そしてこれらをインストールしましょう

```bash
flutter packages get
```

## User レポジトリー

まずはユーザーデータを管理する`UserRepository`を作りましょう。

```dart
class UserRepository {
  Future<String> authenticate({
    @required String username,
    @required String password,
  }) async {
    await Future.delayed(Duration(seconds: 1));
    return 'token';
  }

  Future<void> deleteToken() async {
    /// keystore/keychainから削除
    await Future.delayed(Duration(seconds: 1));
    return;
  }

  Future<void> persistToken(String token) async {
    /// keystore/keychainに追加
    await Future.delayed(Duration(seconds: 1));
    return;
  }

  Future<bool> hasToken() async {
    /// keystore/keychainから読み込み
    await Future.delayed(Duration(seconds: 1));
    return false;
  }
}
```

?> **メモ**: 今回のアプリはそれぞれの実装をモック的に作っていますが、実際は[HttpClient](https://pub.dev/packages/http)や[Flutter Secure Storage](https://pub.dev/packages/flutter_secure_storage)を使ってトークンをリクエストした上で keystore/keychain に読み書きするのが一般的です。

## Authentication State

次に今回のアプリの state をどのように管理していくかを考えつつ bloc (business logic components)を作っていきましょう。

ハイレベルなところでユーザーの Authenticatoin state を管理する必要があります。Authentication state は以下のどれかになります：

- uninitialized - アプリを立ち上げた直後でまだユーザーのログイン状態が不明な状態
- loading - トークンの読み書きをしている最中
- authenticated - ログイン成功
- unauthenticated - ログインされていない

これらの state それぞれに対応しているアプリがユーザーに見せるべきものがあります。

例えば:

- もし state が uninitialized ならユーザーにはスプラッシュスクリーンを見せます
- もし state が loading ならユーザーには progress indicator を見せます
- もし state が authenticated ならユーザーにはホーム画面を見せます
- もし state が unauthenticated ならユーザーにはログインフォームを見せます

> 詳細な実装に入って行く前に bloc がどのような state を持つことができるのかを整理しておくことはとても重要です。

Authentication state 一覧ができたので`AuthenticationState`クラスの実装に入っていきましょう。

```dart
import 'package:equatable/equatable.dart';

abstract class AuthenticationState extends Equatable {
  @override
  List<Object> get props => [];
}

class AuthenticationUninitialized extends AuthenticationState {}

class AuthenticationAuthenticated extends AuthenticationState {}

class AuthenticationUnauthenticated extends AuthenticationState {}

class AuthenticationLoading extends AuthenticationState {}
```

?> **メモ**: [`Equatable`](https://pub.dev/packages/equatable)パッケージは二つの`AuthenticationState`を比較するために使用しています。Dart はデフォルトでは二つの Object を`==`で比較すると二つのインスタンつが同一の場合のみ true を返します。

## Authentication Event

`AuthenticationState`が実装できたら次は`AuthenticationBloc`に情報を送る`AuthenticationEvent`を実装していきましょう。

必要な event は:

- `AppStarted` アプリの開始時に呼ばれ、ユーザーのログイン状態をチェックさせるための event
- `LoggedIn` ユーザーが無事ログインできたことを知らせる event
- `LoggedOut` ユーザーがログアウトしたことを知らせる event

```dart
import 'package:meta/meta.dart';
import 'package:equatable/equatable.dart';

abstract class AuthenticationEvent extends Equatable {
  const AuthenticationEvent();

  @override
  List<Object> get props => [];
}

class AppStarted extends AuthenticationEvent {}

class LoggedIn extends AuthenticationEvent {
  final String token;

  const LoggedIn({@required this.token});

  @override
  List<Object> get props => [token];

  @override
  String toString() => 'LoggedIn { token: $token }';
}

class LoggedOut extends AuthenticationEvent {}
```

?> **メモ**: `meta`パッケージは`AuthenticationEvent`のパラメーターを`@required`にするために必要です。これにより必須パラメーターをインスタンスに渡さなかった場合に IDE が警告してくれるようになります。

## Authentication Bloc

`AuthenticationState`と`AuthenticationEvents`が実装できたのでここからは`AuthenticationBloc`を作っていきましょう。`AuthenticationBloc`は`AuthenticationEvents`を受け取り必要な処理をしてから`AuthenticationState`を返してくれます。

まずは`AuthenticationBloc`クラスを作りましょう。

```dart
class AuthenticationBloc extends Bloc<AuthenticationEvent, AuthenticationState> {
  final UserRepository userRepository;

  AuthenticationBloc({@required this.userRepository}): assert(userRepository != null);
}
```

?> **メモ**: 上記の定義を見ただけでこの bloc は`AuthenticationEvents`を`AuthenticationStates`に変換していることがわかりますね。

?> **メモ**: `AuthenticationBloc`は`UserRepository`に依存しています。

We can start by overriding `initialState` to the `AuthenticationUninitialized()` state.

```dart
@override
AuthenticationState get initialState => AuthenticationUninitialized();
```

これが完了したら`mapEventToState`を実装しましょう。

```dart
@override
Stream<AuthenticationState> mapEventToState(
  AuthenticationEvent event,
) async* {
  if (event is AppStarted) {
    final bool hasToken = await userRepository.hasToken();

    if (hasToken) {
      yield AuthenticationAuthenticated();
    } else {
      yield AuthenticationUnauthenticated();
    }
  }

  if (event is LoggedIn) {
    yield AuthenticationLoading();
    await userRepository.persistToken(event.token);
    yield AuthenticationAuthenticated();
  }

  if (event is LoggedOut) {
    yield AuthenticationLoading();
    await userRepository.deleteToken();
    yield AuthenticationUnauthenticated();
  }
}
```

やりましたね！最終的な`AuthenticationBloc`はこのようになっているはずです

```dart
import 'dart:async';

import 'package:meta/meta.dart';
import 'package:bloc/bloc.dart';
import 'package:user_repository/user_repository.dart';

import 'package:flutter_login/authentication/authentication.dart';

class AuthenticationBloc
    extends Bloc<AuthenticationEvent, AuthenticationState> {
  final UserRepository userRepository;

  AuthenticationBloc({@required this.userRepository})
      : assert(userRepository != null);

  @override
  AuthenticationState get initialState => AuthenticationUninitialized();

  @override
  Stream<AuthenticationState> mapEventToState(
    AuthenticationEvent event,
  ) async* {
    if (event is AppStarted) {
      final bool hasToken = await userRepository.hasToken();

      if (hasToken) {
        yield AuthenticationAuthenticated();
      } else {
        yield AuthenticationUnauthenticated();
      }
    }

    if (event is LoggedIn) {
      yield AuthenticationLoading();
      await userRepository.persistToken(event.token);
      yield AuthenticationAuthenticated();
    }

    if (event is LoggedOut) {
      yield AuthenticationLoading();
      await userRepository.deleteToken();
      yield AuthenticationUnauthenticated();
    }
  }
}
```

これで`AuthenticationBloc`の実装が完了しました。次はプレゼンテーションレイヤーに取り掛かっていきましょう。

## スプラッシュ画面

プレゼンテーションレイヤーで最初に作るのは`SplashPage`です。`SplashPage`は`AuthenticationBloc`ユーザーのログイン状態を確認している最中に表示されるウィジェットになります。

```dart
import 'package:flutter/material.dart';

class SplashPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Text('Splash Screen'),
      ),
    );
  }
}
```

## ホーム画面

次に無事ログインをすることができたユーザーを送る`HomePage`を作りましょう。

```dart
import 'package:flutter/material.dart';

import 'package:flutter_bloc/flutter_bloc.dart';

import 'package:flutter_login/authentication/authentication.dart';

class HomePage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Home'),
      ),
      body: Container(
        child: Center(
            child: RaisedButton(
          child: Text('logout'),
          onPressed: () {
            BlocProvider.of<AuthenticationBloc>(context).add(LoggedOut());
          },
        )),
      ),
    );
  }
}
```

?> **メモ**: ここで初めて`flutter_bloc`を使ったウィジェットが出てきました。後ほど`BlocProvider.of<AuthenticationBloc>(context)`については解説しますが、`HomePage`から`AuthenticationBloc`にアクセスするために必要だということだけ覚えていてください。

?> **メモ**: ユーザーがログアウトボタンを押した時には`LoggedOut` event を `AuthenticationBloc` に送ります。

次に`LoginPage`と`LoginForm`を作ります。

`LoginForm`はユーザーからのインプット(ログインボタンが押されるなど)や、ビジネスロジック(ユーザー名とパスワードに対応するトークンを取得する)を処理する必要があるため、`LoginBloc`を作る必要があります。

`AuthenticationBloc`でも行ったように`LoginState`と`LoginEvents`を定義していきましょう。まずは`LoginState`からです。

## Login State

```dart
import 'package:meta/meta.dart';
import 'package:equatable/equatable.dart';

abstract class LoginState extends Equatable {
  const LoginState();

  @override
  List<Object> get props => [];
}

class LoginInitial extends LoginState {}

class LoginLoading extends LoginState {}

class LoginFailure extends LoginState {
  final String error;

  const LoginFailure({@required this.error});

  @override
  List<Object> get props => [error];

  @override
  String toString() => 'LoginFailure { error: $error }';
}
```

`LoginInitial`は LoginForm の初期状態です。

`LoginLoading`はログイン情報を認証しているときの状態です。

`LoginFailure`はログインが失敗したときの状態です。

`LoginState`が揃ったら今度は`LoginEvent`クラスを作っていきましょう。

## Login Event

```dart
import 'package:meta/meta.dart';
import 'package:equatable/equatable.dart';

abstract class LoginEvent extends Equatable {
  const LoginEvent();
}

class LoginButtonPressed extends LoginEvent {
  final String username;
  final String password;

  const LoginButtonPressed({
    @required this.username,
    @required this.password,
  });

  @override
  List<Object> get props => [username, password];

  @override
  String toString() =>
      'LoginButtonPressed { username: $username, password: $password }';
}
```

`LoginButtonPressed`はユーザーがログインボタンをタップした時のイベントです。`LoginBloc`に対してユーザーのログイン情報に対応したトークンをリクエストするように指示します。

これで`LoginBloc`の実装に入れます。

## Login Bloc

```dart
import 'dart:async';

import 'package:meta/meta.dart';
import 'package:bloc/bloc.dart';
import 'package:user_repository/user_repository.dart';

import 'package:flutter_login/authentication/authentication.dart';
import 'package:flutter_login/login/login.dart';

class LoginBloc extends Bloc<LoginEvent, LoginState> {
  final UserRepository userRepository;
  final AuthenticationBloc authenticationBloc;

  LoginBloc({
    @required this.userRepository,
    @required this.authenticationBloc,
  })  : assert(userRepository != null),
        assert(authenticationBloc != null);

  LoginState get initialState => LoginInitial();

  @override
  Stream<LoginState> mapEventToState(LoginEvent event) async* {
    if (event is LoginButtonPressed) {
      yield LoginLoading();

      try {
        final token = await userRepository.authenticate(
          username: event.username,
          password: event.password,
        );

        authenticationBloc.add(LoggedIn(token: token));
        yield LoginInitial();
      } catch (error) {
        yield LoginFailure(error: error.toString());
      }
    }
  }
}
```

?> **メモ**: `LoginBloc`は`UserRepository`にユーザーをログインするためのロジックを依存しています。

?> **メモ**: `LoginBloc`はユーザー情報を元にユーザーをログインさせられるように`AuthenticationBloc`に依存しています。

`LoginBloc`が完成したらあとは`LoginPage`と`LoginForm`に取り掛かりましょう。

## Login Page

`LoginPage`ウィジェットは`LoginForm`のラッパーとなり`LoginBloc`と`AuthenticationBloc`を`LoginForm`に渡してくれます。

```dart
import 'package:flutter/material.dart';

import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:user_repository/user_repository.dart';

import 'package:flutter_login/authentication/authentication.dart';
import 'package:flutter_login/login/login.dart';

class LoginPage extends StatelessWidget {
  final UserRepository userRepository;

  LoginPage({Key key, @required this.userRepository})
      : assert(userRepository != null),
        super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Login'),
      ),
      body: BlocProvider(
        create: (context) {
          return LoginBloc(
            authenticationBloc: BlocProvider.of<AuthenticationBloc>(context),
            userRepository: userRepository,
          );
        },
        child: LoginForm(),
      ),
    );
  }
}
```

?> **メモ**: `LoginPage`は`StatelessWidget`です。`LoginPage`は`BlocProvider`を使って子孫ウィジェットに`LoginBloc`を渡しています。

?> **メモ**: `LoginBloc`を作る時に`UserRepository`を渡しています。

?> **メモ**: ここでは`BlocProvider.of<AuthenticationBloc>(context)`という記法を使って`LoginPage`から`AuthenticationBloc`にアクセスします。

次は`LoginForm`を作っていきましょう。

## Login Form

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_login/login/login.dart';

class LoginForm extends StatefulWidget {
  @override
  State<LoginForm> createState() => _LoginFormState();
}

class _LoginFormState extends State<LoginForm> {
  final _usernameController = TextEditingController();
  final _passwordController = TextEditingController();

  @override
  Widget build(BuildContext context) {
    _onLoginButtonPressed() {
      BlocProvider.of<LoginBloc>(context).add(
        LoginButtonPressed(
          username: _usernameController.text,
          password: _passwordController.text,
        ),
      );
    }

    return BlocListener<LoginBloc, LoginState>(
      listener: (context, state) {
        if (state is LoginFailure) {
          Scaffold.of(context).showSnackBar(
            SnackBar(
              content: Text('${state.error}'),
              backgroundColor: Colors.red,
            ),
          );
        }
      },
      child: BlocBuilder<LoginBloc, LoginState>(
        builder: (context, state) {
          return Form(
            child: Column(
              children: [
                TextFormField(
                  decoration: InputDecoration(labelText: 'username'),
                  controller: _usernameController,
                ),
                TextFormField(
                  decoration: InputDecoration(labelText: 'password'),
                  controller: _passwordController,
                  obscureText: true,
                ),
                RaisedButton(
                  onPressed:
                      state is! LoginLoading ? _onLoginButtonPressed : null,
                  child: Text('Login'),
                ),
                Container(
                  child: state is LoginLoading
                      ? CircularProgressIndicator()
                      : null,
                ),
              ],
            ),
          );
        },
      ),
    );
  }
}
```

?> **メモ**: `LoginForm`は内部で`BlocBuilder`を使って`LoginState`が変更されるごとに UI を再描画します。`BlocBuilder`は Flutter のウィジェットで
、builder メソッドに定義した通りに state が変わるごとに UI を再描画してくれます。`BlocBuilder`は`StreamBuilder`に似ていますが、よりシンプルに使えて、様々なパフォーマンス改善用の機能も用意されています。

現段階では`LoginForm`には大して何も実装されていないので、ローディング中あることを示すローディングインジケーターウィジェットを実装してみましょう。

## ローディングインジケーター

```dart
import 'package:flutter/material.dart';

class LoadingIndicator extends StatelessWidget {
  @override
  Widget build(BuildContext context) => Center(
        child: CircularProgressIndicator(),
      );
}
```

最後に`main.dart`内で今まで作ってきたものを全て組み合わせましょう。

## 組み立て

```dart
import 'package:flutter/material.dart';

import 'package:bloc/bloc.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:user_repository/user_repository.dart';

import 'package:flutter_login/authentication/authentication.dart';
import 'package:flutter_login/splash/splash.dart';
import 'package:flutter_login/login/login.dart';
import 'package:flutter_login/home/home.dart';
import 'package:flutter_login/common/common.dart';

class SimpleBlocDelegate extends BlocDelegate {
  @override
  void onEvent(Bloc bloc, Object event) {
    super.onEvent(bloc, event);
    print(event);
  }

  @override
  void onTransition(Bloc bloc, Transition transition) {
    super.onTransition(bloc, transition);
    print(transition);
  }

  @override
  void onError(Bloc bloc, Object error, StackTrace stacktrace) {
    super.onError(bloc, error, stacktrace);
    print(error);
  }
}

void main() {
  BlocSupervisor.delegate = SimpleBlocDelegate();
  final userRepository = UserRepository();
  runApp(
    BlocProvider<AuthenticationBloc>(
      create: (context) {
        return AuthenticationBloc(userRepository: userRepository)
          ..add(AppStarted());
      },
      child: App(userRepository: userRepository),
    ),
  );
}

class App extends StatelessWidget {
  final UserRepository userRepository;

  App({Key key, @required this.userRepository}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: BlocBuilder<AuthenticationBloc, AuthenticationState>(
        builder: (context, state) {
          if (state is AuthenticationUninitialized) {
            return SplashPage();
          }
          if (state is AuthenticationAuthenticated) {
            return HomePage();
          }
          if (state is AuthenticationUnauthenticated) {
            return LoginPage(userRepository: userRepository);
          }
          if (state is AuthenticationLoading) {
            return LoadingIndicator();
          }
        },
      ),
    );
  }
}
```

?> **メモ**: ここでは再び`BlocBuilder`を使って`AuthenticationState`が変わるたびに`SplashPage`, `LoginPage`, `HomePage`, `LoadingIndicator` のの表示の出し分けを行なっています。

?> **メモ**: 今回のアプリはアプリ全体が`BlocProvider`で包まれていて、`AuthenticationBloc`がアプリのどこからでもアクセスできるようになっています。おさらいになりますが、`BlocProvider`は子孫要素のどこからでも`BlocProvider.of(context)`という記法を使って Bloc にアクセスできるようにするウィジェットです。こうすることで一つ Bloc インスタンスを全ての子孫要素で共有することができます。

これで`BlocProvider.of<AuthenticationBloc>(context)`が`HomePage`や`LoginPage`に書かれている理由がわかったのではないでしょうか？

`App`を`BlocProvider<AuthenticationBloc>`で囲ったので`AuthenticationBloc`のインスタンスへは`BlocProvider.of<AuthenticationBloc>(BuildContext context)`を使うことで子孫要素のどこからでも呼ぶことができます。

ここまでくればロジックと UI が綺麗に分かれたアプリを Bloc を使って作ることができたと言えるでしょう。

コードを全部見たい方は[こちら](https://github.com/felangel/Bloc/tree/master/examples/flutter_login)から見れます。
