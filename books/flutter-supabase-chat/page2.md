---
title: "Supabase のプロジェクト作成"
---

## Supabaseプロジェクトの作成

今度はSupabase側の設定に入っていきましょう。「まだSupabaseプロジェクト作ったことがないよ」という方もご心配なく！Githubアカウントさえあれば誰でも無料で簡単にプロジェクトが作れます！まずは[こちら](https://app.supabase.com/)にアクセス。

Githubアカウントでログインすることを促されるので緑のボタンを押してログインしてしまいましょう。あとはGithub側でもろもろ許可してしまえばログインできます！ログインが完了してSupabaseの画面に戻ってきたら左上の「New Project」ボタンを押しましょう！

![Supabaseの新規プロジェクト作成](https://supabase.com/images/blog/flutter-chat/create-new-supabase-project.png)

このボタンを押したあとプロジェクト名などを設定します。プロジェクト名はとりあえず「chat」とでも呼んでおきましょう。Detabaseのパスワードに関しては特に今回は使いませんし、後々何かで必要になっても上書きはできるので`Generate a password`ボタンを押してランダムでセキュアなパスワードを自動生成しちゃいましょう。Pricing planはデフォルトの無料版でOKです。ここまで済んだら「Create new Project」ボタンを押しましょう。Supabaseが裏側で新しいプロジェクトを１、2分ほどでセットアップしてくれます。

プロジェクトのセットアップが完了したら実際に設定に入っていきましょう！

## Supabase内でテーブルの作成

今回のアプリで使うテーブルは以下の二つです。
- profiles - ユーザーのプロフィールデータを保存する
- messages - 送信されたチャットデータを保存する

それぞれのメッセージは外部キーによってユーザーのプロフィールに紐づけられています。

![profilesテーブルとmessagesテーブルの関係](https://supabase.com/images/blog/flutter-chat/entity-relations.png)

こちらのSQLをSupabaseダッシュボード内のSQLエディターから実行しましょう。

![SQLエディター](https://supabase.com/images/blog/flutter-chat/sql-editor.png)

```sql
create table if not exists public.profiles (
    id uuid references auth.users on delete cascade not null primary key,
    username varchar(24) not null unique,
    created_at timestamp with time zone default timezone('utc' :: text, now()) not null,

    -- ユーザー名にRegexを使って制限をかける
    constraint username_validation check (username ~* '^[A-Za-z0-9_]{3,24}$')
);
comment on table public.profiles is 'ユーザー名などのユーザー情報を保持する';

create table if not exists public.messages (
    id uuid not null primary key default uuid_generate_v4(),
    profile_id uuid default auth.uid() references public.profiles(id) on delete cascade not null,
    content varchar(500) not null,
    created_at timestamp with time zone default timezone('utc' :: text, now()) not null
);
comment on table public.messages is 'アプリ内で送られたチャットを保持する';
```

実行が完了したらテーブルエディターに行って実際に作成されたテーブルを確認してみましょう。空のテーブルが二つ作成されているはずです。

![テーブルエディターでテーブルを確認](https://supabase.com/images/blog/flutter-chat/table-editor.png)

Supabaseにはリアルタイムにデータを引っ張ってくる機能があるのですが、デフォルトでこちらの機能はオフになっており、テーブル単位でオンにしてあげる必要があります。ダッシュボードから`messages`テーブルのリアルタイム機能をオンにしてあげましょう。

![リアルタイム機能をオンにする](/images/flutter-supabase-chat/turn-on-realtime.png)

ここまできたら今度はFlutter側の作業に入っていきます！