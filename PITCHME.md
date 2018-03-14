## Akka Streamsを使ってOutOfMemoryErrorを回避する話

---

#### <span class="underline">自己紹介</span>

- インターネット広告代理店でウェブアプリケーションを開発しています
- 主に広告効果計測ツールを開発しています
- [t-mochizuki・GitHub](https://github.com/t-mochizuki)

---

#### <span class="underline">今日話すこと</span>

- 大きなサイズのSQL問い合わせ結果をExcelに書き込む方法について話します
- そして、そのことから分かったこと・気が付いたことについて話します

---

#### <span class="underline">今日話さないこと</span>

- 利用したライブラリの詳細については話しません

---

#### <span class="underline">目次</span>

- なぜ？
- どのようにした？
- 分かったこと・気付いたこと
  - 省メモリ場面でIteratorはすごく役に立つ！
  - なんでコンパイルエラーになるの？

---

#### <span class="underline">なぜ？</span>

- レポートを作成するバッチ処理でOutOfMemoryErrorが起こるようになったからです
- そのレポートのフォーマットは複数あり、Excelもありました
- そのため、大きなサイズのSQL問い合わせ結果をExcelに書き込む必要がありました
- こういうときにはストリーム化するのが良さそうだったので試してみました

---

#### <span class="underline">ストリームとは？</span>

![Stream](assets/Stream.svg)

- 終わりがない要素の配列のようなもので
- Subscriber（または、コンシューマー）がデータを要求して |
- Publisher（または、プロデューサー）がデータを供給します |

---

#### <span class="underline">Akka Streamsとは？</span>

- ストリームを取り扱うことができるライブラリです
- つまり、有限のバッファーで無限のデータを取り扱うことができます |

---

#### <span class="underline">どのようにした？</span>

![Example](assets/Example.svg)

- PublisherにScalikeJDBC Streams、SubscriberにPOI SXSSFを使うようにしました
- サンプル・ソースコードを割愛しますが、リポジトリへのリンクのみを紹介します
- [t-mochizuki/scalikejdbc-example at topic-excel+streams-3.2.0](https://github.com/t-mochizuki/scalikejdbc-example/tree/topic-excel%2Bstreams-3.2.0)
- ここでは少しだけ使ったライブラリを紹介します

---

#### <span class="underline">ScalikeJDBC Streamsとは？（1）</span>

> scalikejdbc-streamsは、全てのResultSetを読み込まずにDBがサポートするCURSORなどの仕組みを使ってストリーム処理を行えるよう設計されたDBアクセスのためのモジュールです。

- [ScalikeJDBC streamsモジュールの使い方解説 - yoskhdia's diary](http://yoskhdia.hatenablog.com/entry/2017/05/20/155847)

---

#### <span class="underline">ScalikeJDBC Streamsとは？（2）</span>

> Since ScalikejDBC 3.0, we support the Publisher of Reactive Streams to subscribe a stream from a database query.

- [Reactive Streams Support - ScalikeJDBC](http://scalikejdbc.org/documentation/reactivestreams-support.html)

---

#### <span class="underline">POI SXSSFとは？</span>

> Streaming version of XSSFWorkbook implementing the "BigGridDemo" strategy.

- [SXSSFWorkbook (POI API Documentation)](https://poi.apache.org/apidocs/org/apache/poi/xssf/streaming/SXSSFWorkbook.html)

---

#### <span class="underline">分かったこと・気が付いたこと</span>

- 省メモリ場面でIteratorはすごく役に立つ！
- なんでコンパイルエラーになるの？

---

#### <span class="underline">省メモリ場面でIteratorはすごく役に立つ！</span>

```
// JAVA_OPTS="-Xmx50M" sbt run
object Main extends App {
  val input = Source.fromFile("./input.txt")
  val xs: Iterator[String] = input.getLines

  val tempFile = File.createTempFile("output", ".txt")
  println(tempFile.getPath)
  val fileOutputStream = new FileOutputStream(tempFile)
  val outputStreamWriter = new OutputStreamWriter(fileOutputStream, "UTF-8")

  xs.foreach(x => outputStreamWriter.write(s"$x\n"))

  outputStreamWriter.close()
  fileOutputStream.close()
}
```
@[3](ファイルサイズが50MB以上のファイルを使います)
@[4](イテレーターなのでOutOfMemoryErrorになりません)
@[4](一方、toSeqでイテレータをシーケンスに変換するとOutOfMemoryErrorになります)
@[4](興味がある方はお試し下さい)

---

#### <span class="underline">なんでコンパイルエラーになるの？（1）</span>

![Example](assets/Example2_1.svg)

```
source
  .map(entity => s"${entity.id}, ${entity.name}, ${entity.createdAt}\n")
  .runWith(sink)
  .andThen {
    case _ => system.terminate
  }
```
@[3](ここでtype mismatchでコンパイルエラーになります)
@[3](検索してみたのですが、意外とSQL問い合わせ結果を使うケースが見つかりませんでした)
@[3](そこで困ってしまったのですが、いろいろ試すことで解決することができました)

---

#### <span class="underline">なんでコンパイルエラーになるの？（2）</span>

![Example](assets/Example2_2.svg)

```
source
  .map(entity => entity.id)
  .runWith(sink)
  .andThen {
    case _ => system.terminate
  }
```
@[3](ここでtype mismatchでコンパイルエラーになります)
@[3](しかし、少しエラーメッセージが変わります)
@[3](そこで、ここではByteStringに変換する必要があることに気が付きました)

---

#### <span class="underline">なんでコンパイルエラーになるの？（3）</span>

![Example](assets/Example2_3.svg)

```
source
  .map(entity => s"${entity.id}, ${entity.name}, ${entity.createdAt}\n")
  .map(ByteString(_))
  .runWith(sink)
  .andThen {
    case _ => system.terminate
  }
```
@[3](このようにするとコンパイルエラーになりません)
@[4](ここでのSinkは入って来るByteString要素を与えられたfile pathに書き込むのでコンパイルエラーになっていました)

- [t-mochizuki/scalikejdbc-example at topic-streams-3.2.0](https://github.com/t-mochizuki/scalikejdbc-example/tree/topic-streams-3.2.0)

---

#### <span class="underline">結論</span>

- 同じような課題のときはストリーム化が有効
- 省メモリ場面でIteratorはすごく役に立つ！
- いつものように型合わせがある！？

---

#### <span class="underline">宣伝</span>

- We are hiring!
- フロントエンド、サーバサイド、インフラなど幅広く求人しています!
- 興味がある方はお声掛け頂けると幸いです!

