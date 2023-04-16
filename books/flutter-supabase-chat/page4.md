---
title: "登録・ログインページ"
---

## 登録ページの作成

一通り下準備が整ったのでページの作成に入っていきましょう！まずは登録ページに取り掛かります。今回はシンプルにメールアドレスとパスワード、そしてユーザーネームを設定して登録する形にしましょう。ユーザー名はアプリ内でそのユーザーのアイコンとして表示されます。

```dart:lib/pages/register.dart
import 'package:flutter/material.dart';
import 'package:my_chat_app/utils/constants.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

class RegisterPage extends StatefulWidget {
  const RegisterPage({Key? key}) : super(key: key);

  static Route<void> route({bool isRegistering = false}) {
    return MaterialPageRoute(
      builder: (context) => const RegisterPage(),
    );
  }

  @override
  State<RegisterPage> createState() => _RegisterPageState();
}

class _RegisterPageState extends State<RegisterPage> {
  final bool _isLoading = false;

  final _formKey = GlobalKey<FormState>();

  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  final _usernameController = TextEditingController();

  Future<void> _signUp() async {
    final isValid = _formKey.currentState!.validate();
    if (!isValid) {
      return;
    }
    final email = _emailController.text;
    final password = _passwordController.text;
    final username = _usernameController.text;
    try {
      await supabase.auth.signUp(
          email: email, password: password, data: {'username': username});
      // TODO: チャットページ実装後に下記コードを追加
      // Navigator.of(context)
      //     .pushAndRemoveUntil(ChatPage.route(), (route) => false);
    } on AuthException catch (error) {
      context.showErrorSnackBar(message: error.message);
    } catch (error) {
      context.showErrorSnackBar(message: unexpectedErrorMessage);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('登録'),
      ),
      body: Form(
        key: _formKey,
        child: ListView(
          padding: formPadding,
          children: [
            TextFormField(
              controller: _emailController,
              decoration: const InputDecoration(
                label: Text('メールアドレス'),
              ),
              validator: (val) {
                if (val == null || val.isEmpty) {
                  return '必須';
                }
                return null;
              },
              keyboardType: TextInputType.emailAddress,
            ),
            formSpacer,
            TextFormField(
              controller: _passwordController,
              obscureText: true,
              decoration: const InputDecoration(
                label: Text('パスワード'),
              ),
              validator: (val) {
                if (val == null || val.isEmpty) {
                  return '必須';
                }
                if (val.length < 6) {
                  return '6文字以上';
                }
                return null;
              },
            ),
            formSpacer,
            TextFormField(
              controller: _usernameController,
              decoration: const InputDecoration(
                label: Text('ユーザー名'),
              ),
              validator: (val) {
                if (val == null || val.isEmpty) {
                  return '必須';
                }
                final isValid = RegExp(r'^[A-Za-z0-9_]{3,24}$').hasMatch(val);
                if (!isValid) {
                  return '3~24文字のアルファベットか文字で入力してください';
                }
                return null;
              },
            ),
            formSpacer,
            ElevatedButton(
              onPressed: _isLoading ? null : _signUp,
              child: const Text('登録'),
            ),
            formSpacer,
            TextButton(
              onPressed: () {
                // TODO: ログインページが実装できたらコメントを外す
                // Navigator.of(context).push(LoginPage.route());
              },
              child: const Text('すでにアカウントをお持ちの方はこちら'),
            )
          ],
        ),
      ),
    );
  }
}
```

`RegisterPage`が作れたら一度UIを確認してみましょう。main.dartの`MyApp`をこちらのように書き換えて`RegisterPage`を表示させてみます。

```dart:lib/main.dart
class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'チャットアプリ',
      home: RegisterPage(), // RegisterPageを表示するように変更
    );
  }
}
```

![登録ページのUI](/images/flutter-supabase-chat/register-page.png)

いい感じに登録ページが表示されていますね。

ユーザー名の`TextFormField`の`validation`の部分を見ていただくと、テーブル定義でユーザー名のフィールドに使っていたものと同じ正規表現を使ってユーザー名のフォーマットを制限していることがわかると思います。テーブルを定義した際にもテーブルのユーザー名の箇所に同じバリデーションロジックをかけています。一般的にアプリのユーザー名にはこのように制限がかかっていることが多いので今回もそれっぽく実装してみました。

さらに、`_signup()`メソッドを見てみると、ユーザー名をここでは`userMetadata`としてSupabaseに保存していることがわかるかと思います。この`userMetadata`とはSupabaseがデフォルトで用意してくれている`auth.users`テーブル内に存在する`jsonb`型のカラムで、今回はこのユーザー名を他のユーザーもロードしてきて閲覧できるようにしたいので`profiles`テーブルにコピーしてあげる必要があります。

ここで役立つのが[`Postgresトリガー`](https://www.youtube.com/watch?v=0N6M5BBe9AE)と[`Postgres Function`](https://supabase.com/docs/guides/database/functions)です。Postgres Functionはデータベース内に定義できる関数のことで、任意で引数を渡してあげて特定のSQL、を実行させることができるものになっています。Postgresトリガーはデータベース内に任意の変更があった際に特定のPostgres Functionを実行する機能になっております。この二つを組み合わせて、`auth.users`テーブルにユーザーが新しく追加された際にその中身を`profiles`テーブルにコピーしてあげることができます。

下記のSQLを実行してトリガーとFunctionを定義してあげましょう！その際便利なのが、`profiles`テーブルの`username`カラムには`unique`な制限をかけてあげているので、Flutterのアプリ側でユーザーが選んだユーザー名が既に登録済みの場合はエラーが出て登録が失敗し、ユーザーに違うユーザー名を選ぶことを促すことができる点です。データベースレベルでユニークさが定義されているので、アプリを作る際はあまりそこらへんに神経を使うことなく簡単に裏側のデータをきれいに保つことができます。


```sql
-- ユーザー名をprofilesテーブルにコピーするDatabase Functionを定義
create or replace function public.handle_new_user() returns trigger as $$
    begin
        insert into public.profiles(id, username)
        values(new.id, new.raw_user_meta_data->>'username');

        return new;
    end;
$$ language plpgsql security definer;

-- ユーザー作成時に`handle_new_user`を呼ぶためのトリガーを定義
create trigger on_auth_user_created
    after insert on auth.users
    for each row
    execute function handle_new_user();
```

もう一つSupabaseの設定を変える必要がある点があります。SupabaseはデフォルトでEmailで登録した際にそのメールアドレスに確認メールを送り、その確認が済まないときちんと登録完了したことにならない仕様になっているのですが、今回は簡単なサンプルアプリということで一旦こちらはオフにしてしまいましょう。後々続編の記事で認証認可についてはもう少し深堀するので、その際にここら辺はカバーさせてください。ということで、Supabase管理画面の`authentication -> settings`から`Confirm Email`のスイッチをオフにしてください。

![メールアドレスの確認をオフにする](https://supabase.com/images/blog/flutter-chat/turn-off-email-confirmation.png)

ここまでできたら一度実際にユーザー登録をしてみましょう。登録ページからメールアドレス、パスワード、そしてユーザー名を入力して登録ボタンを押します。アプリ側では特に何も起こらないですが、Supabaseの管理画面から`Auth`のページに行くとユーザーが作成されていることが確認できるかと思います。

## ログインページ

ログインページはシンプルにメールアドレスとパスワードを入力する`TextFormField`があるくらいで、特に捻りはないです。登録ページと同じく、ログインが完了したらチャットページに飛ぶ形になっていますが、一旦そこの箇所はコメントアウトしてあります。

```dart:lib/pages/login_page.dart
import 'package:flutter/material.dart';
import 'package:my_chat_app/utils/constants.dart';
import 'package:supabase_flutter/supabase_flutter.dart';

class LoginPage extends StatefulWidget {
  const LoginPage({Key? key}) : super(key: key);

  static Route<void> route() {
    return MaterialPageRoute(builder: (context) => const LoginPage());
  }

  @override
  LoginPageState createState() => LoginPageState();
}

class LoginPageState extends State<LoginPage> {
  bool _isLoading = false;
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();

  Future<void> _signIn() async {
    setState(() {
      _isLoading = true;
    });
    try {
      await supabase.auth.signInWithPassword(
        email: _emailController.text,
        password: _passwordController.text,
      );
      // TODO: チャットページ実装後に下記コードを追加
      // ログイン後にチャットページに飛ぶ
      // Navigator.of(context)
      //     .pushAndRemoveUntil(ChatPage.route(), (route) => false);
    } on AuthException catch (error) {
      context.showErrorSnackBar(message: error.message);
    } catch (_) {
      context.showErrorSnackBar(message: unexpectedErrorMessage);
    }
    if (mounted) {
      setState(() {
        _isLoading = false;
      });
    }
  }

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('ログイン')),
      body: ListView(
        padding: formPadding,
        children: [
          TextFormField(
            controller: _emailController,
            decoration: const InputDecoration(labelText: 'メールアドレス'),
            keyboardType: TextInputType.emailAddress,
          ),
          formSpacer,
          TextFormField(
            controller: _passwordController,
            decoration: const InputDecoration(labelText: 'パスワード'),
            obscureText: true,
          ),
          formSpacer,
          ElevatedButton(
            onPressed: _isLoading ? null : _signIn,
            child: const Text('ログイン'),
          ),
        ],
      ),
    );
  }
}
```

再びUIを確認してみましょう。`RegisterPage`の一番下にある`TextButton`内の`// TODO: ログインページが実装できたらコメントを外す`下のコメントを外すと、そのボタンを押した時にログインページに飛ぶようになります。一度そのボタンを押してログインページに飛んでみましょう。試しにログインをしてみてください。アプリのUI上は特に何も起こらないですが、デバッグコンソール上で無事にログインできたことが確認できるかと思います。

![ログインページのUI](/images/flutter-supabase-chat/login-page.png)

次のページからいよいよメインのチャットページを作っていきます！
