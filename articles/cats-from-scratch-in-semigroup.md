---
title: "ゼロから作るCatsライブラリ ~ implicit編 ~"
emoji: "🐱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Scala"]
published: true
---

# はじめに
はじめまして。やがです。普段はScala言語を用いて保育園の業務支援SaaSの開発をしています。
Zennは初投稿になります💪

筆者は関数型プログラミングに入門したばかりであるため、なるべく誤りがないよう試行錯誤しながら記事を書いていますが、もし誤っている点等があればコメントをいただけるととても嬉しいです。

## Catsライブラリの概要
CatsライブラリはScala言語で使用することができる関数型プログラミングライブラリです。

名前の由来は圏(category)の省略系であると公式ドキュメントに書かれています。関数型プログラミングに大きな影響を与えている圏論との繋がりを感じますね。

Catsライブラリには関数型プログラミングのツールのようなものが多く用意されています。そしてこれらのツールの大部分は型クラスと呼ばれるプログラミングパターンで実装されています。

:::message
本記事では型クラスに関する説明は省略します。
:::

## 本記事の内容
本記事は型クラスをScala言語で実装するための、`implicit`の機能と実装パターンについてサンプルコードを交えて紹介をしていきます。

`implicit`は「暗黙的な」というような意味を持つ単語になります。

本記事で紹介する、Scala言語の`implicit`の機能と実装パターンのは以下の3つです。
- `implicit parameter`
- `implicit conversion`
- `Enrich my library`パターン

以下、順に紹介していきます。

# implicit parameter
`implicit parameter`は、メソッドやクラスの引数を暗黙的に渡す事ができる機能です。

`implicit parameter`は暗黙的に渡したい引数グループに`implicit`修飾子をつける事で使用します。

そして`implicit`修飾子による引数に値を渡すためには、メソッドやクラスを呼び出す処理のスコープ内に、引数の型に対応した暗黙的な値（以下、`implicit value`）がただ一つ定義されている必要があります。

## サンプルコード
例として、`implicit`修飾子を用いた引数を持つメソッドは以下のように定義します。
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
// val res0: String = Hello yaga
```
今度は正常にメソッドを呼び出す事ができました。

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

## その他補足
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
その他`context bound`等、説明できていない部分もあるとは思いますが、本記事では省略します。

# implicit conversion
`implicit conversion`は`implicit`修飾子を持つメソッドの引数の型から返り値の型へと暗黙的な型変換を行う機能です。

## サンプルコード
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

`receiveString`メソッドに`Int`型の値を渡してみます。通常は引数に与える型が`String`型ではないため、`type mismatch`エラーが発生しますが、`implicit conversion`によりコンパイルが通ります。
```scala
scala> receiveString(3)
// val res0: String = 3
```

`implicit conversion`は一見便利な機能に見えます。しかし、上記のようにコンパイルエラーを簡単にすり抜ける等の理由から、次に紹介する`Enrich my library`パターン以外での使用は、ずいぶん前から非推奨とされているようです。

# Enrich my libraryパターン
`Enrich my library`パターンとは`implicit conversion`の機能を用いて、既存クラスの拡張を**場当たり的（アドホック）** に行う事ができる実装パターンです。

日本語では**拡張メソッド**という名前で呼ばれています。

## サンプルコード
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

## implicit class
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

# おわりに
今回はCatsライブラリの基礎となる型クラスを実装するための、Scala言語の`implicit`の機能についての説明をしました。

繰り返しになりますが、誤っている点等あればコメントをいただけると嬉しいです🙇‍♂️

モチベーション次第ですが、できれば今後もシリーズとして書いていきたいです。シリーズでやるのであれば次回は`Semigroup`,`Monoid`型クラスについての内容にしようかなと考えています。

最後まで読んでいただきありがとうございました！

# 参考資料
参考資料を以下にまとめます。
https://github.com/typelevel/cats
https://www.scalawithcats.com/dist/scala-with-cats.html
https://www2.slideshare.net/AoiroAoino/scala-79575940
https://scala-text.github.io/scala_text/implicit.html
https://gakuzzzz.github.io/slides/implicit_reintroduction/#1
https://kmizu.hatenablog.com/entry/2017/05/19/074149
https://www2.slideshare.net/MakotoFukuhara/scala-56310825
http://chopl.in/post/2012/11/06/introduction-to-typeclass-with-scala/
http://blog.takeda-soft.jp/blog/show/396.html