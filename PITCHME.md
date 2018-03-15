## Akka Streamsを使ってOutOfMemoryErrorを回避する話

---

#### <span class="underline">自己紹介</span>

- ウェブサービスを開発しています
- 広告の効果を計測するツールを開発していてサーバサイドを担当することが多いです
- 趣味は散歩、読書、競技プログラミングです

---

#### <span class="underline">今日話すこと</span>

- Akka Streamsを使って、試してみたこと・困ったことについて話したいと思います

---

#### <span class="underline">試してみたこと</span>

![Example](assets/Example.svg)

- 背景としては担当しているプロダクトのバッチ処理でOutOfMemoryErrorが起こるようになり、それを回避するためにストリーム化を検討している感じです
- PublisherにScalikeJDBC Streams、Subscriberにストリーミング版のPOIを使ってみました
- レコード数50万件、最大ヒープサイズ300MBで確認してみたところ、期待した通りOutOfMemoryErrorを回避できました
- [t-mochizuki/scalikejdbc-example at topic-excel+streams-3.2.0](https://github.com/t-mochizuki/scalikejdbc-example/tree/topic-excel%2Bstreams-3.2.0)

---

#### <span class="underline">困ったこと</span>

- Akka Streamsを使ってファイルコピーを試したあと、PublisherをScalikeJDBC Streamsにしたところエラーになってしまいました
- ちゃんと公式ドキュメントを読んでいれば困ることはないかもしれませんが。。
- 同じように困っているひとも居るかもしれないので紹介したいと思います

---

#### <span class="underline">なんでコンパイルエラーになるの？（1）</span>

![Example](assets/Example2_1.svg)

```
source
  .map(employee => s"${employee.id}, ${employee.name}, ${employee.createdAt}\n")
  .runWith(sink)
  .andThen {
    case _ => system.terminate
  }
```
@[3](ここでtype mismatchでコンパイルエラーになります)
@[3](検索してみたのですが、同じようなケースを見つけることができませんでした)
@[3](それで困ってしまったのですが、いろいろ試すことで解決することができました)

---

#### <span class="underline">なんでコンパイルエラーになるの？（2）</span>

![Example](assets/Example2_2.svg)

```
source
  .map(employee => employee.id)
  .runWith(sink)
  .andThen {
    case _ => system.terminate
  }
```
@[2](StringではなくLongにしてみました)
@[3](それでもtype mismatchでコンパイルエラーになります)
@[3](しかし、少しエラーメッセージが変わります)
@[3](それで、ここでByteStringに変換する必要があるということに気が付くことができました)

---

#### <span class="underline">なんでコンパイルエラーになるの？（3）</span>

![Example](assets/Example2_3.svg)

```
source
  .map(employee => s"${employee.id}, ${employee.name}, ${employee.createdAt}\n")
  .map(ByteString(_))
  .runWith(sink)
  .andThen {
    case _ => system.terminate
  }
```
@[3](このようにByteStringに変換するとコンパイルエラーになりません)
@[4](コンパイルエラーの原因ですが、ここでのSinkは入って来るByteStringを与えられたfile pathに書き込むものでByteString以外が入って来ていたのでコンパイルエラーになっていました)

- [t-mochizuki/scalikejdbc-example at topic-streams-3.2.0](https://github.com/t-mochizuki/scalikejdbc-example/tree/topic-streams-3.2.0)

---

#### <span class="underline">まとめ</span>

- Akka Streamsを使ってストリーム化するとOutOfMemoryErrorを回避できる！
- いつものように型合わせがある！

---

#### <span class="underline">宣伝</span>

- We are hiring!
- フロントエンド、サーバサイド、インフラなど幅広く求人しています!
- 興味がある方はお声掛け頂けると幸いです!

