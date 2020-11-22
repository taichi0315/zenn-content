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

筆者は関数型プログラミングに入門したばかりであるため、なるべく誤りがないよう試行錯誤しながら記事を書いていますが、もし誤っている点等があればコメントをいただけるととても嬉しいです。

## Catsライブラリの概要
CatsライブラリはScala言語で使用することができる関数型プログラミングライブラリです。

名前の由来は圏(*category*)の省略系であると公式ドキュメントに書かれています。関数型プログラミングに大きな影響を与えている圏論との繋がりを感じますね。

Catsライブラリには関数型プログラミングのツールのようなものが多く用意されています。そしてこれらのツールの大部分は型クラスと呼ばれるプログラミングパターンで実装されています。

本記事ではその型クラスについてサンプルコードを交えて紹介をしていきます。

## 本記事の内容

本記事のゴールは以下のテストを通すことです。

```scala
...

class OptionInstancesTest extends AnyFunSuite {
  test("additive") {
    assert((Option(3)  |+| Option(4)) == Option(7))
    assert((Option(10) |+| Option(5)) == Option(15))
  }
}
```

以上のテストでは、`Option[Int]`型である2つの値を、`|+|`メソッドにより1つの値にまとめています。**まとめる**と書きましたが、具体的には2つの`Option[Int]型`の値が内包する`Int`型の値を加算した結果を、1つの`Option[Int]`型に包んで返すというシンプルな処理です。

`|+|`メソッドはScala標準ライブラリには実装されておらず、Scalaの標準ライブラリのみでは以上のテストは通りません。

本記事では`|+|`メソッドを提供する`Additive`という名前の型クラスをゼロから実装していきます。

:::message
Catsライブラリにおいて`|+|`メソッドは、`Semigroup`や`Monoid`といった型クラスによって提供されますが、本記事ではそれらの詳しい説明は省略します。
:::

# 型クラスとは
## 型クラスの概要
型クラスとはHaskellに由来するプログラミングパターンです。

型クラスは既存のプリミティブ型や標準ライブラリ等のソースコードに変更を加えることなく、新しい機能を拡張して追加することができます。

テストコードを例に挙げると、`Option[T]`型のインスタンスに`|+|`メソッドを用意するためには、通常の`trait`を用いると以下のような実装イメージになるかと思います。（あくまでイメージです。）

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

また**多相**という単語は、**多態**や**ポリモーフィズム**という言葉でも言い換えられるかと思います。**多相性**に関しては関数型プログラミングに留まらない概念であるため、本記事での説明は省略します。

## Scalaにおける型クラス
Scalaでは`implicit`の機能を用いることによって、型クラスを実装できます。

Scalaにおける型クラスは主に以下の4つのコンポーネントによって実装することができます。
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
その他`implicit object`等、説明できていない部分もあるとは思いますが、本記事では省略します。

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

`receiveString`メソッドに`Int`型の値を渡してみます。通常は引数に与える型が`String`型ではないため,`type mismatch`エラーが発生しますが、`implicit conversion`によりコンパイルが通ります。
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

`Enrich my library`パターンを実装する場合は、基本的に`implicit class`で良いかとは思いますが、`implicit class`を用いていないライブラリもあるようです（Catsライブラリの`cats.syntax`パッケージ内は`implicit class`を使用していなさそう）。

そのためどちらの実装方法も読めるようにしておくと良いのかなと思いました。

# Additive型クラスの実装
型クラスを実装する準備が整ったので、いよいよ`Additive`型クラスを実装していきます。

本記事のゴールであるテストコードを再度確認します。
```scala
...

class OptionInstancesTest extends AnyFunSuite {
  test("additive") {
    assert((Option(3)  |+| Option(4)) == Option(7))
    assert((Option(10) |+| Option(5)) == Option(15))
  }
}
```
正直なところ、`Option[Int]`型のインスタンスに`|+|`メソッドを追加するだけであれば、前節で説明した`Enrich my library`パターンのみを用いた簡単な実装ですぐにテストを通せます。

では型クラスによって場当たり的に、かつ多相性を持つ`|+|`メソッドを追加できるようにするには何が違うのか。Catsライブラリを参考にしながら、実際に`Addtivie`型クラスを実装していきます。

## プロジェクト立ち上げ
今回の実装環境は以下の通りです。
```
sbt 1.4.1
scala 2.13.3
```

最初に`sbt new scala/scala-seed.g8`コマンドによって、シードプロジェクトを作成します。本記事ではproject名を猫繋がりで`tabby`（トラ猫）という名前にしています。

## 実装手順
再掲になりますが、型クラスは以下の4つのコンポーネントによって実装ができます。
1. `trait`による型クラス定義
2. `implicit value`による型クラスインスタンス定義
3. `implicit parameter`によるメソッド定義
4. `Enrich my library`パターン（`implicit conversion`）によるクラス拡張

本記事では、最初に`Additive[Int]`を実装した後に、`Additive[Option[A]]`を実装していきます。

## Additive[Int]
### traitによる型クラス定義
`tabby`パッケージ以下に、`Additive`トレイトを定義します。
```scala:src/main/scala/tabby/Additive.scala
package tabby

trait Additive[A] {
  def combine(x: A, y: A): A
}
```
`combine`メソッドは、`Additive`トレイトの型パラメータに与えられた型である2つの値を1つにまとめるメソッドです。最終的にテストコードで呼んでいる`|+|`メソッドは、`combine`メソッドのエイリアスになります。

### implicit valueによる型クラスインスタンス定義
次に`Additive[Int]`型のインスタンスを返す`implicit value`を実装します。本記事では`Int`型である2つの値を1つにまとめる処理は加算であるとします。
```scala:src/main/scala/tabby/instances/IntInstances.scala
package tabby
package instances

trait IntInstances {
  implicit val intAdditive: Additive[Int] = new Additive[Int] {
    def combine(x: Int, y: Int): Int = x + y
  }
}
```
Catsライブラリでは`cats.instances`パッケージ下に、既存の型別に型クラスインスタンスを返す`implicit value`が定義されています。

`IntInstances`トレイトで定義した`implicit value`を取り込みしやすくするために、以下のような`implicits`オブジェクトを作成します。
```scala:src/main/scala/tabby/implicits.scala
package tabby

object implicits
  extends instances.IntInstances
```
これにより、
```scala
import tabby.implicits._
```
を1行追加するだけで、`implicit value`をスコープ内に取り込む事ができるようになります。

Catsライブラリを使用しているとおまじないのように追加する
```scala
import cats.implicits._
```
と同じ役割です。

以上の`implicit value`により、`implicit parameter`に`Additive[Int]`型の値を渡す準備ができました。

### implicit parameterによるメソッド定義
これまでの実装を用いて、console内で`implicit parameter`を用いたメソッドを動かしてみます。

まず最初に`tabby.instances.IntInstances`トレイト内に定義されている`Additive[Int]`型のインスタンスを返す`implicit value`を取り込みます。
```scala
scala> import tabby.implicits._
// import tabby.implicits._
```
次に`Additive[A]`型の`implicit value`を受け取る`implicit parameter`を用いた`combine`メソッドを定義します。
```scala
scala> import tabby.Additive
// import tabby.Additive

scala> def combine[A](x: A, y: A)(implicit aa: Additive[A]): A = {
     |   aa.combine(x, y)
     | }
// def combine[A](x: A, y: A)(implicit aa: tabby.Additive[A]): A
```
実際に`combine`メソッドを呼んでみましょう。
```scala
scala> combine[Int](3, 4)
// val res0: Int = 7
```
`implicit parameter`を用いて、`Additive[Int]`型の`implicit value`が持つ`combine`メソッドを呼ぶ事ができました。

しかしこのままでは`combine`メソッドに毎回型パラメータを与える等、少々使いにくい部分があるので、`Enrich my library`パターンを用いて`Int`型のインスタンスに`combine`メソッドを追加します。

### Enrich my libraryパターン（implicit conversion）によるクラス拡張
Catsライブラリでは、既存の型のインスタンスを拡張する`Enrich my library`パターンは、`cats.syntax`パッケージ下に実装されています。

`Int`型に`combine`メソッドを追加するのは以下にような実装になります。
```scala:src/main/scala/tabby/syntax/additive.scala
package tabby
package syntax

trait AdditiveSyntax {
  implicit final class AdditiveOps[A](lhs: A) {
    def |+|(rhs: A)(implicit aa: Additive[A]): A = combine(rhs)
    def combine(rhs: A)(implicit aa: Additive[A]): A = aa.combine(lhs, rhs)
  }
}
```
Catsライブラリでは、`implciit class`を用いずに`Enrich my library`パターンが実装されているのですが、本記事では簡単のため`implicit class`を用いています。

また`combine`メソッドのエイリアスである`|+|`メソッドも追加しています。

次に`tabby.syntax`以下の`implicit`定義も取り込みやすくなるよう、`tabby.implicits`オブジェクトを修正します。
```scala:src/main/scala/tabby/implicits.scala
package tabby

object implicits
  extends syntax.AdditiveSyntax
  with instances.IntInstances
```

以上で、`Int`型のインスタンスに`|+|`メソッドを追加する事ができました。consoleで確かめてみましょう。
```scala
scala> import tabby.implicits._
// import tabby.implicits._

scala> 3 combine 4
// val res0: Int = 7

scala> 3 |+| 4
// val res1: Int = 7
```

意図した通りに動いているようです。

## Additive[Option[A]]
本題である、`|+|`メソッドを`Option[A]`型に追加する実装を行っていきます。

型クラスを一度作成してしまえば、その他の型を型クラスに含めるのはとても簡単です。

具体的には4つのコンポーネントのうち、`Additive[Option[Int]]`型のインスタンスを返す`implicit value`を新たに追加するだけです。
1. ~~`trait`による型クラス定義~~
2. `implicit value`による型クラスインスタンス定義
3. ~~`implicit parameter`によるメソッド定義~~
4. ~~`Enrich my library`パターン（`implicit conversion`）によるクラス拡張~~

### implicit valueによる型クラスインスタンス定義
では、`Additive[Option[A]]`型のインスタンスを返す`implicit value`を定義します。
```scala:src/main/scala/tabby/instances/OptionInstances.scala
package tabby
package instances

trait OptionInstances {
  implicit def optionAdditive[A](implicit aa: Additive[A]): Additive[Option[A]] = new Additive[Option[A]] {
    def combine(x: Option[A], y: Option[A]): Option[A] =
      x match {
        case None    => y
        case Some(a) =>
          y match {
            case None    => x
            case Some(b) => Some(aa.combine(a, b))
          }
      }
  }
}
```
型パラメータとして受け取っている型`A`が`Additive`型クラスに含まれている必要がある実装になっています。

言い換えると「`A`が`Additive`型クラスに含まれていれば`Option[A]`も`Additive`型クラスに含まれる」といったような宣言です。

:::message
実際のCatsライブラリで上記のような`implicit parameter`を持つ`implicit value`の実装をみると、`context bound`と呼ばれる糖衣構文によって書かれている事が多いです。本記事では説明を省略しますが、興味のある方は調べてみてください。
:::

最後に、`OptionInstances`を`tabby.implicits`オブジェクトに継承します。
```scala:src/main/scala/tabby/implicits.scala
package tabby

object implicits
  extends syntax.SemigroupSyntax
  with instances.IntInstances
  with instances.OptionInstances
```
これでテストを通す全ての準備ができました。

## テスト
では最後にテストを通していきます。
```scala:src/test/scala/tabby/instances/OptionInstancesTest.scala
package tabby.instances

import org.scalatest.funsuite.AnyFunSuite
import tabby.implicits._

class OptionInstancesTest extends AnyFunSuite {
  test("additive") {
    assert((Option(3)  |+| Option(4)) == Option(7))
    assert((Option(10) |+| Option(5)) == Option(15))
  }
}
```
`import tabby.implicits._`を忘れないように気をつけます。
```scala
sbt:tabby> test
...
[info] All tests passed.
[success] Total time: 1 s, completed 2020/11/15 16:36:29
```
テストを通す事ができました🙌

# おわりに
今回はCatsライブラリの基礎となる型クラスについての説明を行っていきました。

先人の方々の説明資料に比べると少々冗長になってしまったかなと思いもありつつ、勉強した内容を自分なりに丁寧にアウトプットできたかなと思っています。

繰り返しになりますが、誤っている点等あればコメントをいただけると嬉しいです🙇‍♂️

モチベーション次第ですが、できれば今後もシリーズとして書いていきたいです。シリーズでやるのであれば次回は`Monoid`型クラスについてですね。

最後まで読んでいただきありがとうございました！

# 参考資料
かなり多いですが、どの資料もとても参考になったので以下にまとめます。

https://github.com/typelevel/cats
https://www.scalawithcats.com/dist/scala-with-cats.html
https://www2.slideshare.net/AoiroAoino/scala-79575940
https://scala-text.github.io/scala_text/implicit.html
https://gakuzzzz.github.io/slides/implicit_reintroduction/#1
https://kmizu.hatenablog.com/entry/2017/05/19/074149
https://www2.slideshare.net/MakotoFukuhara/scala-56310825
http://chopl.in/post/2012/11/06/introduction-to-typeclass-with-scala/
http://blog.takeda-soft.jp/blog/show/396.html
