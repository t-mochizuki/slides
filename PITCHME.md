## Akka Streamsを使ってOutOfMemoryErrorを回避する話

---

#### <span class="underline">自己紹介</span>

- インターネット広告代理店でウェブアプリケーションを開発しています
- We are hiring! |

---

#### <span class="underline">今日話すこと（要約）</span>

- SQLによる問い合わせ結果（大きなサイズ）をExcelファイルに書き込めるか？

---

#### <span class="underline">今日話さない（話せない）こと</span>

- Akka Streamsの仕組み（マテリアライズなど）
- Akka StreamsのAPI

---

#### <span class="underline">目次</span>

- ストリームとは
- Akka Streamsとは
- ケース①テキストファイルのコピー
- ケース②SQLによる問い合わせ結果をテキストファイルに書き込む
- ケース③SQLによる問い合わせ結果をExcelファイルに書き込む

---

#### <span class="underline">ストリームとは</span>

- 終わりがない要素の配列のようなもの
- コンシューマーがデータをストリームに要求して |
- プロデューサーがデータをストリームに供給する |

---

#### <span class="underline">Akka Streamsとは</span>

- ストリームを取り扱うことができるライブラリ
- つまり、有限のバッファーで無限のデータを取り扱うことができる |

---

#### <span class="underline">ケース①テキストファイルのコピー</span>

- Iterator

+++

#### <span class="underline">Iterator</span>

OutOfMemoryErrorになりません！

```
import java.io.{File, FileOutputStream, OutputStreamWriter}
import scala.io.Source

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
@[6](ファイルサイズが50MB以上のファイルを使います)
@[7](イテレーターなのでOutOfMemoryErrorになりません)

---

#### <span class="underline">ケース②SQLによる問い合わせ結果をテキストファイルに書き込む</span>

- 非ストリームな問い合わせ結果
- ストリームな問い合わせ結果

+++

#### <span class="underline">非ストリームな問い合わせ結果</span>

OutOfMemoryErrorになります！

```
import scalikejdbc._
import scalikejdbc.config._

case class Emp(
  id: Long,
  name: String,
  createdAt: java.time.ZonedDateTime
)

object Emp extends SQLSyntaxSupport[Emp]

// JAVA_OPTS="-Xmx80M" sbt run
object Main extends App {

  println("Main start")

  DBs.setup()

  DB localTx { implicit session =>
    sql"""
    create table emp (
      id serial not null primary key,
      name varchar(64) not null,
      created_at timestamp not null
    )
    """.execute.apply()
  }

  val column = Emp.column
  1 to 500000 foreach { id =>
    DB localTx { implicit s =>
      withSQL {
        insert.into(Emp).namedValues(
          column.id -> id,
          column.name -> s"taro$id",
          column.createdAt -> sqls.currentTimestamp)
      }.update.apply()
    }
  }


  val emp = Emp.syntax("emp")
  val result = DB localTx { implicit s =>
    withSQL {
      selectFrom(Emp as emp)
    }.map { rs =>
      Emp(rs.get(emp.resultName.id), rs.get(emp.resultName.name), rs.get(emp.resultName.createdAt))
    }.list.apply()
  }

  println("Main end")
}
```
@[4-8](entity objectを定義する)
@[10](extractorを定義する)
@[17](データベース接続をセットアップする)
@[19-27](テーブルを作成する)
@[29-39](50万件のレコードを登録する)
@[43-49](50万件のレコードを取得する)
@[43-49](OutOfMemoryErrorになります)

+++

#### <span class="underline">ストリームな問い合わせ結果</span>

```
```

---

#### <span class="underline">ケース③SQLによる問い合わせ結果をExcelファイルに書き込む</span>

- POI-XSSF
- POI-SXSSF

+++

#### <span class="underline">POI-XSSF</span>

```
```

+++

#### <span class="underline">POI-SXSSF</span>

```
```

---

#### <span class="underline">結論</span>

- SQLによる問い合わせ結果（大きなサイズ）をExcelファイルに書き込める
