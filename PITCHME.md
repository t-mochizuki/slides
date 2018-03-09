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

- ストリームとは？
- Akka Streamsとは？
- ケース①テキストファイルのコピー
- ケース②SQLによる問い合わせ結果をテキストファイルに書き込む
- ケース③SQLによる問い合わせ結果をExcelファイルに書き込む

---

#### <span class="underline">ストリームとは？</span>

- 終わりがない要素の配列のようなもの
- コンシューマーがデータをストリームに要求して |
- プロデューサーがデータをストリームに供給する |

---

#### <span class="underline">Akka Streamsとは？</span>

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
@[43-49](SeqなのでOutOfMemoryErrorになります)

+++

#### <span class="underline">ストリームな問い合わせ結果</span>

OutOfMemoryErrorになりません！

```
package example

import akka.NotUsed
import akka.actor.ActorSystem
import akka.stream.{ActorMaterializer, IOResult}
import akka.stream.scaladsl.{FileIO, Sink, Source}
import akka.util.ByteString
import java.io.File
import java.nio.file.StandardOpenOption._

import scalikejdbc._
import scalikejdbc.config._
import scalikejdbc.streams._

import scala.concurrent.Future

// JAVA_OPTS="-Xmx80M" sbt run
object Main extends App {

  println("Main start")

  DBs.setup()

  implicit val system = ActorSystem("system")
  implicit val materializer = ActorMaterializer()
  implicit val dispatcher = system.dispatcher

  // source
  val emp = Emp.syntax("emp")
  val databasePublisher: DatabasePublisher[Emp] = DB readOnlyStream {
    withSQL {
      selectFrom(Emp as emp)
    }.map { rs =>
      Emp(rs.get(emp.resultName.id), rs.get(emp.resultName.name), rs.get(emp.resultName.createdAt))
    }.iterator()
  }
  val source: Source[Emp, NotUsed] = Source.fromPublisher(databasePublisher)

  // sink
  val output = new File("taro.csv")
  val sink: Sink[ByteString, Future[IOResult]] = FileIO.toPath(output.toPath, options = Set(CREATE, WRITE, APPEND))

  // run
  source
    .map(entity => s"${entity.id}, ${entity.name}, ${entity.createdAt}\n")
    .map(ByteString(_))
    .runWith(sink)
    .andThen {
      case _ => system.terminate
    }

  println("Main end")
}
```
@[1](entity objectとextractorは同じパッケージにあります)
@[28-37](Sourceを用意します)
@[39-41](Sinkを用意します)
@[43-50](SourceとSinkの間にFlowを並べます)
@[43-50](そして実行します)
@[43-50](ストリームなのでOutOfMemoryErrorになりません)

---

#### <span class="underline">ケース③SQLによる問い合わせ結果をExcelファイルに書き込む</span>

- ストリームな問い合わせ結果＋POI-SXSSF

+++

#### <span class="underline">ストリームな問い合わせ結果＋POI-SXSSF</span>

OutOfMemoryErrorになりません！

```
package example

import akka.NotUsed
import akka.actor.ActorSystem
import akka.stream.ActorMaterializer
import akka.stream.scaladsl.Source
import java.io.{File, FileOutputStream}

import org.apache.poi.xssf.streaming.SXSSFWorkbook
import scalikejdbc._
import scalikejdbc.config._
import scalikejdbc.streams._

// JAVA_OPTS="-Xmx125M" sbt run
object Main extends App {

  println("Main start")

  DBs.setup()

  implicit val system = ActorSystem("system")
  implicit val materializer = ActorMaterializer()
  implicit val dispatcher = system.dispatcher

  // source
  val databasePublisher: DatabasePublisher[Emp] = DB readOnlyStream {
    val emp = Emp.syntax("emp")
    withSQL {
      selectFrom(Emp as emp)
    }.map { rs =>
      Emp(rs.get(emp.resultName.id), rs.get(emp.resultName.name), rs.get(emp.resultName.createdAt))
    }.iterator()
  }
  val source: Source[Emp, NotUsed] = Source.fromPublisher(databasePublisher)

  // POI-SXSSF
  val workbook = new SXSSFWorkbook
  val sheet = workbook.createSheet("test1")

  source
    .zipWithIndex
    .runForeach {
      case (entity, rowIndex) =>
        val row = sheet.createRow(rowIndex.toInt)
        row.createCell(0).setCellValue(entity.id)
        row.createCell(1).setCellValue(entity.name)
        row.createCell(2).setCellValue(entity.createdAt.toString)
    }
    .andThen {
      case _ => {
        val output = File.createTempFile("taro", ".xlsx")
        println(s"File path is ${output.getPath}")
        val stream = new FileOutputStream(output)
        workbook.write(stream)

        stream.close
        system.terminate
      }
    }

  println("Main end")
}
```
@[25-34](Sourceを用意します)
@[36-38](Sinkのようなものを用意します)
@[40](ストリームなのでOutOfMemoryErrorになりません)
@[41](行を指定するために各要素に添字を追加します)
@[42-48](書き込む値とその値の書き込み先を指定します)
@[42-48](SXSSFなのでOutOfMemoryErrorになりません)

---

#### <span class="underline">結論</span>

- SQLによる問い合わせ結果（大きなサイズ）をExcelファイルに書き込める
