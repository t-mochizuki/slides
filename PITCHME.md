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
- コンシューマーがストリームにデータを要求して |
- プロデューサーがストリームにデータを供給する |

---

#### <span class="underline">Akka Streamsとは</span>

- ストリームを取り扱うことができるライブラリ
- つまり、有限のバッファーで無限のデータを取り扱うことができる |

---

#### <span class="underline">ケース①テキストファイルのコピー</span>

- Iterator

---

#### <span class="underline">ケース②SQLによる問い合わせ結果をテキストファイルに書き込む</span>

- 非ストリームな問い合わせ結果
- ストリームな問い合わせ結果

---

#### <span class="underline">ケース③SQLによる問い合わせ結果をExcelファイルに書き込む</span>

- POI-XSSF
- POI-SXXF

---

#### <span class="underline">結論</span>

- SQLによる問い合わせ結果（大きなサイズ）をExcelファイルに書き込める
