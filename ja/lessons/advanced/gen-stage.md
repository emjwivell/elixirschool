---
version: 1.0.2
title: GenStage
---

このレッスンでは、 GenStage をどのように扱うのか、アプリケーションでどのように活用できるのかを詳しく見ていきます。

{% include toc.html %}

## イントロダクション

GenStage とは何でしょうか？ 公式ドキュメントによると「Elixir 向けの仕様と計算フロー」とされていますが、これはいったいどういう意味なのでしょうか？

つまるところ GenStage は、個別のプロセスで独立したステップ（ステージ）によって実行される、処理のパイプラインを定義する方法を提供します。パイプラインを使った処理をした経験がある方なら、こうした概念はよくご存知でしょう。

この仕組みがどうやって動くかをより良く理解するために、単純な生産者-消費者のフローを以下に図示します。

```
[A] -> [B] -> [C]
```

この例では生産者(producer) `A` 、生産者-消費者(producer-consumer) `B` 、そして消費者(consumer) `C` という、3 つのステージがあります。 `A` は `B` により消費される値を生産し、 `B` はなんらかの処理をしたのちに消費者 `C` によって受信される新しい値を返します。次のセクションで説明するようにステージの役割は重要です。

この例では、1 対 1 の生産者対消費者となっていますが、どのステージにおいても、両方とも複数の生産者、複数の消費者を持つことが可能です。

この概念をより詳しく説明するために、GenStage を使ってパイプラインを構築しますが、まずはじめに GenStage の各役割をもう少し深く掘り下げてみましょう。

## 消費者と生産者

これまで見てきたように、ステージに与えられた役割は大切です。GenStage の仕様では以下の 3 つの役割があります。

- `:producer` — 送り元。生産者は消費者からの催促を待ち、要求されたイベントに応じる。

- `:producer_consumer` — 送り元と受け手の両方。生産者-消費者は、生産者からの要求イベントと同じように、他の消費者からの催促に応じる。

- `:consumer` — 受け手。消費者は、生産者に要求してデータを受け取る。

生産者が催促を **待っている** ことに注意してください。GenStage では、消費者が上流の生産者に催促を送り、生産者から送られてくるデータを処理します。これによって、バックプレッシャーとして知られる仕組みが容易になります。バックプレッシャーというのは、消費者の処理がビジーの時に過剰なプレッシャーがかからないよう、生産者側にその責任を負わせる仕組みです。

さて、GenStage の内部の役割についておさらいしたところで、実際のアプリケーションを作ってみましょう。

## はじめに

この例では、数値を生成し、偶数をソートし、そして最後にそれらを出力する GenStage アプリケーションを構築します。

このアプリケーションでは、3 つの GenStage の役割を全て使います。「生産者」は、数を数えたり出力したりする責任があります。「生産者-消費者」は、偶数だけにフィルタリングし、下流から来る催促に応えます。最後に、数字を表示する「消費者」をつくります。

まずはスーパーバイザーツリーを持つ mix プロジェクトを作成しましょう。

```shell
$ mix new genstage_example --sup
$ cd genstage_example
```

続いて `mix.exs` ファイルを開いて依存ライブラリに `gen_stage` を加えます。

```elixir
defp deps do
  [
    {:gen_stage, "~> 0.11"}
  ]
end
```

この先に進む前に、依存ライブラリを取得してコンパイルを通しておきましょう。

```shell
$ mix do deps.get, compile
```

生産者を構築する準備が整いました！

## 生産者

GenStage アプリケーションの第一歩は、生産者の作成です。先ほど述べたように、一定の数のストリームを生成する生産者を作成する必要があります。という訳で早速、生産者のファイルを作成しましょう:

```shell
$ mkdir lib/genstage_example
$ touch lib/genstage_example/producer.ex
```

そして以下のコードを書き加えましょう:

```elixir
defmodule GenstageExample.Producer do
  use GenStage

  def start_link(initial \\ 0) do
    GenStage.start_link(__MODULE__, initial, name: __MODULE__)
  end

  def init(counter), do: {:producer, counter}

  def handle_demand(demand, state) do
    events = Enum.to_list(state..(state + demand - 1))
    {:noreply, events, state + demand}
  end
end
```

ここで着目すべき 2 つの重要な個所が `init/1` と `handle_demand/2` です。GenServer のところでやったように `init/1` で初期状態をセットしますが、もっと大切なのは自分自身に「生産者」とラベル付けすることです。GenStage は `init/1` 関数からの戻り値で自身のプロセスを分類します。

`handle_demand/2` 関数は生産者の大部分がある場所ですが、この関数は全ての GenStage 生産者が実装しなければなりません。ここで消費者によって催促された数のセットを返し、カウンターを増やします。消費者からの催促は、上記のコードでは `demand` となっていますが、これは処理可能なイベントの数に対応する整数として表されています（デフォルト値は 1000 です）。

## 生産者-消費者

数を生成する生産者ができたので、今度は生産者-消費者に移りましょう。生産者に数を催促し、数をフィルターして偶数のみに絞り、消費者からの催促に応えたいと思います。

```shell
$ touch lib/genstage_example/producer_consumer.ex
```

以下のサンプルコードのようにファイルを更新しましょう:

```elixir
defmodule GenstageExample.ProducerConsumer do
  use GenStage

  require Integer

  def start_link do
    GenStage.start_link(__MODULE__, :state_doesnt_matter, name: __MODULE__)
  end

  def init(state) do
    {:producer_consumer, state, subscribe_to: [GenstageExample.Producer]}
  end

  def handle_events(events, _from, state) do
    numbers =
      events
      |> Enum.filter(&Integer.is_even/1)

    {:noreply, numbers, state}
  end
end
```

生産者-消費者に `init/1` の中の新しいオプションと新しい関数 `handle_events/3` を導入したことに気づかれたかもしれません。ここでは `subscribe_to` オプションを用いて、ある特定の生産者と通信することを GenStage に指示しています。

`handle_events/3` 関数は、入ってくるイベントを受取って処理し変換した値の集合を返す場所です。ここからもわかるように、消費者はほとんど同じ方法で実装されますが、重要な違いは、 `handle_events/3` 関数の戻り値とその使用方法です。プロセスに producer_consumer とラベル付けすると、タプルの第 2 引数（ここでは `numbers`）が下流の消費者からの要求を満たすために使われます。消費者側ではこの値は破棄されます。

## 消費者

最後に消費者があります。でははじめましょう:

```shell
$ touch lib/genstage_example/consumer.ex
```

消費者と生産者-消費者はとてもよく似ているので、さほど変わりありません:

```elixir
defmodule GenstageExample.Consumer do
  use GenStage

  def start_link do
    GenStage.start_link(__MODULE__, :state_doesnt_matter)
  end

  def init(state) do
    {:consumer, state, subscribe_to: [GenstageExample.ProducerConsumer]}
  end

  def handle_events(events, _from, state) do
    for event <- events do
      IO.inspect({self(), event, state})
    end

    # 消費者としては二度とイベントを出力しない
    {:noreply, [], state}
  end
end
```

前のセクションで学習したように、消費者はイベントを出力しないので、タプルの 2 番目の値は破棄されます。

## 全部まとめる

さて、生産者、生産者-消費者、そして消費者が生まれた今、すべてを結びつける準備が整いました。

`lib/genstage_example/application.ex` ファイルを開いて、スーパーバイザーツリーにこれらの新しいプロセスを追加しましょう。

```elixir
def start(_type, _args) do
  import Supervisor.Spec, warn: false

  children = [
    worker(GenstageExample.Producer, [0]),
    worker(GenstageExample.ProducerConsumer, []),
    worker(GenstageExample.Consumer, [])
  ]

  opts = [strategy: :one_for_one, name: GenstageExample.Supervisor]
  Supervisor.start_link(children, opts)
end
```

もし間違いが一つもなければ、このプロジェクトを実行すると全てが正しく動くはずです。

```shell
$ mix run --no-halt
{#PID<0.109.0>, 2, :state_doesnt_matter}
{#PID<0.109.0>, 4, :state_doesnt_matter}
{#PID<0.109.0>, 6, :state_doesnt_matter}
...
{#PID<0.109.0>, 229062, :state_doesnt_matter}
{#PID<0.109.0>, 229064, :state_doesnt_matter}
{#PID<0.109.0>, 229066, :state_doesnt_matter}
```

できました！ アプリケーションが偶数だけを出力するのを期待していましたが、こんなにも **素早く** 出力しています。

ここに処理パイプラインがあります。生産者は数字を出力し、生産者-消費者は奇数を破棄して、そして消費者はそれらの数を表示するというフローを続けています。

## 複数の生産者もしくは消費者

イントロダクションで言及したように、1 つ以上の生産者、または、消費者を持つことができます。少し見てみましょう。

サンプルから `IO.inspect/1` の出力を調べると、各々のイベントは単一の PID によって処理されていることがわかります。 `lib/genstage_example/application.ex` を修正して複数のワーカーが動くように少し調整してみましょう。

```elixir
children = [
  worker(GenstageExample.Producer, [0]),
  worker(GenstageExample.ProducerConsumer, []),
  worker(GenstageExample.Consumer, [], id: 1),
  worker(GenstageExample.Consumer, [], id: 2)
]
```

ここで 2 つの消費者を設定してアプリケーションを実行するとどうなるか、見てみましょう。

```shell
$ mix run --no-halt
{#PID<0.120.0>, 2, :state_doesnt_matter}
{#PID<0.121.0>, 4, :state_doesnt_matter}
{#PID<0.120.0>, 6, :state_doesnt_matter}
{#PID<0.120.0>, 8, :state_doesnt_matter}
...
{#PID<0.120.0>, 86478, :state_doesnt_matter}
{#PID<0.121.0>, 87338, :state_doesnt_matter}
{#PID<0.120.0>, 86480, :state_doesnt_matter}
{#PID<0.120.0>, 86482, :state_doesnt_matter}
```

見ての通り、単に一行のコードを付け足し、消費者の ID を与えただけで、複数の PID ができました。

## ユースケース

さてこれまでに GenStage について学習し最初のサンプルアプリケーションを構築してきましたが、GenStage の **実際の** ユースケースはどのようなものがあるのでしょうか。

- データ変換パイプライン - 生産者は、単純な数値ジェネレータである必要はありません。データベースや、Apache Kafka のような別のデータソースからイベントを生成することもできます。生産者-消費者と消費者を組み合わせれば、それらが利用可能になった時に、メトリクスを処理したりソートしたりカタログ化したり保存したりすることができます。

- 処理キュー - イベントはどんなものでもありえますが、一連の消費者によって実行される、まとまった処理を生成することができます。

- イベント処理 - データパイプラインと同様に、ソースからリアルタイムで出力されたイベントに対し、受信し、処理し、並べ替えたり等の処理することができます。

これらは、GenStage で出来ることのほんの数例にすぎません。
