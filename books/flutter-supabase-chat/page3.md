---
title: "Flutter create & Flutter と Supabase の紐付け・準備"
---

## 空のFlutterアプリを作成

まず最初に空のFlutterアプリを作りましょう。

ターミナルから以下のコマンドを打ってください。

```bash
flutter create --platforms=web,ios,android my_chat_app
```

それが終わったら下記コマンドで実行してみましょう。エミュレーターでもWeb上でも大丈夫です。

```bash
cd my_chat_app
flutter run
```

Flutterがデフォルトで用意しているカウンターアプリが立ち上がっているかと思います。一旦ここまで来たら、普段使っているコードエディターを開いてコーディングに入りましょう！

## パッケージのインストール

pubspec.yamlを開いて以下のパッケージを追加しましょう！

```yaml
supabase_flutter: ^1.0.0
```
`supabase_flutter`はSupabase上でログインしたり、データの読み書きをしたりする際に使います。

`flutter pub get`を実行してパッケージのインストールを完了させましょう。先ほどローカルでFlutterのアプリを実行していましたが、そちらも一度閉じて再実行する必要があります。

![アプリのディレクトリー構造]()

## Supabaseとの紐付け

Supabaseを使うにはmain関数で[initialize](https://supabase.com/docs/reference/dart/initializing#flutter-initialize)してあげる必要があります。
`main.dart`を編集してSupabaseをinitializeしてあげましょう。

```dart:lib/main.dart
import 'package:flutter/material.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await Supabase.initialize(
    // TODO: ここにSupabaseのURLとAnon Keyを入力
    url: 'SUPABASE_URL',
    anonKey: 'SUPABASE_ANON_KEY',
  );
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'チャットアプリ',
      home: // TODO: 後ほど初期ページに変更
    );
  }
}
```

ここでSupabase URLとSupabase Anon Keyが必要になるのですが、これらはSupabaseダッシュボードの`settings -> API`から探すことができます。これらの情報は外部に漏れても全く問題ないものなのでそのままGitにコミットしてしまっても大丈夫です！

![SupabaseのAPI関連情報の探し場所](/images/flutter-supabase-chat/supabase-credentials.png)

ついでに最後にアプリの要所要所で使う細かな便利な変数や関数を定義したconstantsファイルを作りましょう。

```dart:lib/utils/constants.dart
import 'package:flutter/material.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

/// Supabase にアクセスするためのクライアントインスタンス
final supabase = Supabase.instance.client;

/// シンプルなプリローダー
const preloader =
    Center(child: CircularProgressIndicator(color: Colors.orange));

/// ちょっとした隙間を作るのに便利なウィジェット
const formSpacer = SizedBox(width: 16, height: 16);

/// フォームのパディング
const formPadding = EdgeInsets.symmetric(vertical: 20, horizontal: 16);

/// 予期せぬエラーが起きた際のエラーメッセージ
const unexpectedErrorMessage = '予期せぬエラーが起きました';

/// Snackbarを楽に表示させるための拡張メソッド
extension ShowSnackBar on BuildContext {
  /// 標準的なSnackbarを表示
  void showSnackBar({
    required String message,
    Color backgroundColor = Colors.white,
  }) {
    ScaffoldMessenger.of(this).showSnackBar(SnackBar(
      content: Text(message),
      backgroundColor: backgroundColor,
    ));
  }

  /// エラーが起きた際のSnackbarを表示
  void showErrorSnackBar({required String message}) {
    showSnackBar(
      message: message,
      backgroundColor: Theme.of(this).colorScheme.error,
    );
  }
}
```