---
title: "ゼロから作るCatsライブラリ ~ 型クラス編 ~"
emoji: "🐱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Scala"]
published: false
---

# はじめに
はじめまして。やがです。普段はScalaを用いて保育園の業務支援SaaSの開発をしています。
Zennは初投稿になります💪

筆者は関数型プログラミングに入門したばかりであるため、なるべく誤りがないよう試行錯誤しながら記事を書いていますが、もし誤っている点等があればコメントで指摘していただけるととても嬉しいです。

## Catsライブラリの概要
CatsライブラリはScala言語で使用することができる関数型プログラミングライブラリです。名前の由来は圏(*category*)の省略系であると公式ドキュメントに書かれています。関数型プログラミングに大きな影響を与えている圏論との繋がりを感じますね。

Catsライブラリには関数型プログラミングのツールのようなものが多く用意されています。そしてこれらのツールの大部分は型クラスと呼ばれるプログラミングパターンで実装されています。

本記事ではその型クラスについてサンプルコードを交えて紹介をしていきます。

## 本記事の内容

本記事のゴールは以下のテストを通すことです。

```scala
...

class OptionInstancesTest extends AnyFunSuite {
  test("semigroup") {
    assert((Option(3)  |+| Option(4)) == Option(7))
    assert((Option(10) |+| Option(5)) == Option(15))
  }
}
```

以上のテストでは、`Option[Int]`型である2つの値を、`|+|`メソッドにより1つの値にまとめています。**まとめる**と書きましたが、具体的には2つの`Option[Int]型`の値が内包する`Int`型の値を加算した結果を、1つの`Option[Int]`型に包んで返すというシンプルな処理です。

`|+|`メソッドはScala標準ライブラリには実装されておらず、Scalaの標準ライブラリのみでは以上のテストは通りません。本記事では`|+|`メソッドを提供する`Additive`という名前の型クラスをゼロから実装していきます。

:::message
Catsライブラリにおいて`|+|`メソッドは、`Semigroup`や`Monoid`といった型クラスによって提供されますが、本記事ではそれらの詳しい説明は省略します。
:::

# 型クラスとは
## 型クラスの概要
型クラスとはHaskellに由来するプログラミングパターンです。型クラスは既存のプリミティブ型や標準ライブラリ等のソースコードに変更を加えることなく、新しい機能を拡張して追加することができます。

テストコードを例に挙げると、`Option[Int]`型のインスタンスに`|+|`メソッドを用意するためには、通常の`trait`を用いると以下のような実装イメージになるかと思います。（あくまでイメージです。）

```scala
trait Additive[A] {
  def |+|(rhs: A): A
}

class Option[T](value: T) extends Additive[Option[T]] {
  def |+|(rhs: Option[T]): Option[T] = Option(value + rhs.value)
}
```
しかし、`Option[T]`型はScalaの標準ライブラリで定義されている既存のクラスであるため、上記のコードのように後からクラス定義を書き換えることはできません。

以上のような問題を解決するのが型クラスです。

より専門的な言葉を用いると、**アドホック多相を実現するデザインパターン**といったような説明が正しいように学んでいます。

**アドホック**という単語は「臨時の・暫定的な・場当たり的な」といったような意味を持つようなので、既に定義されているクラスに対しても**場当たり的に**何らかの特性（型クラス）を追加する事ができるといった認識です。

## Scalaにおける型クラス
Scalaでは`implicit`の機能を用いることによって、型クラスを実装できます。

Scalaの型クラスは主に以下の4つのコンポーネントによって実装することができます。
1. `trait`による型クラス定義
2. `implicit value`による型クラスインスタンス定義
3. `implicit parameter`によるメソッド定義
4. `Enrich my library`パターン（`implicit conversion`）によるクラス拡張

`implicit`の機能を知らない方からすると知らない単語が並んでいると思うので、次に`implicit`が具体的にどのような機能を持っているかを見ていきます。

# implicitの機能
`implicit`は「暗黙的な」という意味を持つ単語です。以下、順に`implicit`の機能を紹介していきます。

## implicit parameter
`implicit parameter`は、メソッドやクラスの引数を暗黙的に渡す事ができる機能です。

`implicit parameter`は暗黙的に渡したい引数グループに`implicit`修飾子をつける事で使用します。

そして`implicit`修飾子による引数に値を渡すためには、メソッドやクラスを呼び出す処理のスコープ内に、引数の型に対応した暗黙的な値（以下、`implicit value`）がただ一つ定義されている必要があります。

### サンプルコード
`implicit`修飾子を用いた引数を持つメソッドは以下のように定義します。
```scala
scala> def showGreet(name: String)(implicit greet: String): String = s"$greet $name"
// def showGreet(name: String)(implicit greet: String): String
```
挨拶の文字列を表す`greet`引数は暗黙的に渡すため、`showGreet("hoge")`の形で、`name`引数のみでメソッドを呼び出すことができます。

実際にメソッドを呼んでみましょう。
```scala
scala> showGreet("yaga")
/*              ^
       error: could not find implicit value for parameter greet: String
 */
```
`greet`引数のための`implicit value`が見つからないというコンパイルエラーが出ました。

`implicit value`は通常の値定義に`implicit`修飾子をつける事で定義します。
```scala
scala> implicit val hello: String = "Hello"
// val hello: String = Hello
```
`implicit parameter`は引数の**型**に対応する`implicit value`を必要とするため、値の名前が異なっていても問題はありません。

再度、`showGreet`メソッドを呼んでみます。
```scala
scala> showGreet("yaga")
// val res1: String = Hello yaga
```
今度は正常にメソッドを呼び出す事ができました👏

以上が`implicit parameter`の機能の概要となります。

:::message
`implicit`修飾子による引数の型に対応する`implicit value`が複数定義されている場合、以下のようなコンパイルエラーが出ます。
:::
```scala
scala> implicit val hello: String = "Hello"
// val hello: String = Hello

scala> implicit val goodMorning: String = "Good Morning"
// val goodMorning: String = Good Morning

scala> showGreet("yaga")
/*              ^
       error: ambiguous implicit values:
        both value hello of type String
        and value goodMorning of type String
        match expected type String
 */
```

### その他補足
`implicit value`は`val`だけではなく`def`を用いることも可能です。
```scala
implicit def foo: String = ...
```

`def`を用いると、型パラメータを使用できるようになります。
```scala
implicit def foo[T]: Option[T] = ...
```

`implicit value`が`implicit parameter`を持つことも可能です。
```scala
implicit def foo[T](implicit a: T): Option[T] = ...
```

## implicit conversion
`implicit conversion`は`implicit`修飾子を持つメソッドの引数の型から返り値の型へと暗黙的な型変換を行う機能です。

### サンプルコード
例として、`Int`型から`String`型への暗黙的な型変換を行うメソッドを定義します。
```scala
scala> implicit def intToString(a: Int): String = a.toString
// def intToString(a: Int): String
```

そして受け取った`String`型の値をそのまま返すメソッドがあるとします。
```scala
scala> def receiveString(a: String): String = a
// def receiveString(a: String): String
```

`receiveString`に`Int`型の値を渡してみます。通常は引数に与える型が`String`型ではないため,`type mismatch`エラーが発生しますが、`implicit conversion`によりコンパイルが通ります。
```scala
scala> receiveString(3)
// val res0: String = 3
```

`implicit conversion`は一見便利な機能に見えます。しかし、上記のようにコンパイルエラーを簡単にすり抜ける等の理由から、次に紹介する`Enrich my library`パターン以外での使用は、ずいぶん前から非推奨とされているようです。

## Enrich my libraryパターン
`Enrich my library`パターンとは`implicit conversion`の機能を用いて、既存クラスの拡張を行う実装パターンです。日本語では**拡張メソッド**という名前で呼ばれています。

### サンプルコード
例として、既存の`Int`型の値に`show`メソッドを追加する実装を行っていきます。まず最初に`show`メソッドを定義している`ShowIntOps`クラスを定義します。
```scala
scala> class ShowIntOps(a: Int) {
     |   def show: String = a.toString
     | }
// class ShowIntOps
```
次に`implciit conversion`を用いて、`Int`型の値を`ShowIntOps`クラスのインスタンスに型変換するメソッドを定義します。
```scala
scala> implicit def syntaxShowInt(a: Int): ShowIntOps = new ShowIntOps(a)
// def syntaxShowInt(a: Int): ShowIntOps
```

では`show`メソッドを呼んでみましょう。
```scala
scala> 3.show
// val res0: String = 3
```
以上のようにして、`Emrich my library`パターンによる既存クラスの拡張を行う事ができました。

### implicit class
`scala 2.10`以降ではクラス定義に`implicit`修飾子をつける事で、上記のサンプルコード実装をさらに簡潔に記述する事ができます。
```scala
scala> implicit class ShowIntOps(a: Int) {
     |   def show: String = a.toString
     | }
// class ShowIntOps

scala> 3.show
// val res0: String = 3
```

`Enrich my library`パターンを実装する場合は、基本的に`implicit class`で良いかとは思いますが、`implicit class`を用いていないライブラリもあるようです（Catsライブラリの`cats.syntax`パッケージ内は`implicit class`を使用していなさそう）。そのためどちらの実装方法も読めるようにしておくと良いのかなと思いました。

# Additive型クラスの実装
型クラスを実装する準備が整ったので、いよいよ`Additive`型クラスを実装していきます。

本記事のゴールであるテストコードを再度確認します。
```scala
...

class OptionInstancesTest extends AnyFunSuite {
  test("semigroup") {
    assert((Option(3)  |+| Option(4)) == Option(7))
    assert((Option(10) |+| Option(5)) == Option(15))
  }
}
```
正直なところ、`Option[Int]`型のインスタンスに`|+|`メソッドを追加するだけであれば、前節で説明した`Enrich my library`パターンのみを用いた簡単な実装ですぐにテストを通せます。

しかし型クラスは、**アドホック多相を実現するデザインパターン**という説明にあるように**多相性**を持ち合わせている事が重要です。

ここからはCatsライブラリの設計・実装を参考にしながら、`Additive`型クラスの実装を行っていきます。

## プロジェクト立ち上げ
今回の実装環境は以下の通りです。
```
sbt 1.4.1
scala 2.13.3
```

最初に`sbt new scala/scala-seed.g8`コマンドによって、シードプロジェクトを作成します。本記事ではproject名を猫繋がりで`tabby`（トラ猫）という名前にしています。

## 実装手順
冒頭でも説明した以下の4つのコンポーネントを順番に実装していきます。
1. `trait`による型クラス定義
2. `implicit value`による型クラスインスタンス定義
3. `implicit parameter`によるメソッド定義
4. `Enrich my library`パターン（`implicit conversion`）によるクラス拡張

## 1. traitによる型クラス定義
`tabby`パッケージ以下に、`Additive`トレイトを定義します。
```scala:src/main/scala/tabby/Additive.scala
package tabby

trait Additive[A] {
  def combine(x: A, y: A): A
}
```
`combine`メソッドは、`Additive`トレイトの型パラメータに与えられた型である2つの値を1つにまとめるメソッドです。最終的にテストコードで呼んでいる`|+|`メソッドは、`combine`メソッドのエイリアスになります。

## 2. implicit valueによる型クラスインスタンス定義
次に`Additive[Int]`型のインスタンスを返す`implicit value`を実装します。

