---
version: 1.0.2
title: カスタム Mix タスク
---

Elixir プロジェクトのためのカスタム Mix タスクの作成

{% include toc.html %}

## 導入

カスタムの Mix タスクを追加して Elixir アプリケーションの機能を拡張したいと思うのは珍しいことではありません。
私たちのプロジェクト特有の Mix タスクを作成する方法を学ぶ前に、既存のタスクを見てみましょう:

```shell
$ mix phx.new my_phoenix_app

* creating my_phoenix_app/config/config.exs
* creating my_phoenix_app/config/dev.exs
* creating my_phoenix_app/config/prod.exs
* creating my_phoenix_app/config/prod.secret.exs
* creating my_phoenix_app/config/test.exs
* creating my_phoenix_app/lib/my_phoenix_app.ex
* creating my_phoenix_app/lib/my_phoenix_app/endpoint.ex
* creating my_phoenix_app/test/views/error_view_test.exs
...
```

上のシェルコマンドから分かる通り、 Phoenix Framework は新しいプロジェクトを作成するカスタムの Mix タスクを持っています。
私たちのプロジェクトのために、同様のものを作るにはどうすればいいでしょうか？幸いなことに、カスタム Mix タスクの作成が可能であるだけでなく、 Elixir はそれをとても簡単にしてくれます。

## セットアップ

基本的な Mix アプリケーションをセットアップしましょう。

```shell
$ mix new hello

* creating README.md
* creating .gitignore
* creating mix.exs
* creating config
* creating config/config.exs
* creating lib
* creating lib/hello.ex
* creating test
* creating test/test_helper.exs
* creating test/hello_test.exs

Your Mix project was created successfully.
You can use "mix" to compile it, test it, and more:

cd hello
mix test

Run "mix help" for more commands.
```

それでは、 Mix が生成した **lib/hello.ex** に、 "Hello, World!" を出力する簡単な関数を作成しましょう。

```elixir
defmodule Hello do
  @doc """
  Outputs `Hello, World!` every time.
  """
  def say do
    IO.puts("Hello, World!")
  end
end
```

## カスタム Mix タスク

カスタムの Mix タスクを作成しましょう。
新しいディレクトリとファイル、 **hello/lib/mix/tasks/hello.ex** を作成してください。
このファイルに以下の 7 行の Elixir コードを追加しましょう。

```elixir
defmodule Mix.Tasks.Hello do
  use Mix.Task

  @shortdoc "Simply runs the Hello.say/0 function"
  def run(_) do
    # calling our Hello.say() function from earlier
    Hello.say()
  end
end
```

defmodule 文が `Mix.Tasks` と、そしてコマンドラインから呼び出したい名前から始まっているのに気付いてください。
二行目では名前空間に `Mix.Task` ビヘイビアをもたらす `use Mix.Task` を実行しています。
それから、今のところ全ての引数を無視する run 関数を宣言します。
この関数の中では、 `Hello` モジュールの `say` 関数を呼び出しています。

## Mix タスクの実行

私たちの Mix タスクを確かめてみましょう。
アプリケーションのディレクトリにいる限り、これは上手くいきます。
コマンドラインから `mix hello` を実行すると以下のようになるはずです:

```shell
$ mix hello
Hello, World!
```

Mix はデフォルトでかなり親切です。
誰もが時々スペルミスをするのを知っているので、ファジーマッチングと呼ばれる技術を用いて候補を提案します:

```shell
$ mix hell
** (Mix) The task "hell" could not be found. Did you mean "hello"?
```

また、前に新しいモジュール属性である `@shortdoc` を紹介したのに気がついたでしょうか？これは `mix help` コマンドをターミナルで実行したときのように私たちのアプリケーションを使用するときに便利です。

```shell
$ mix help

mix app.start         # Starts all registered apps
...
mix hello             # Simply calls the Hello.say/0 function.
...
```

注: 新しいタスクが `mix help` の出力に表示されるには、コードが事前にコンパイルされてなければいけません。
`mix compile` を直接実行するか、先ほどと同じように `mix hello` を実行してコンパイルのトリガーを作り出すことでコンパイルできます。
