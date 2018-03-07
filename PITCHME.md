# akka-streamsを使ってOut of memory errorを回避する話

---

# SlideAaa

---

#### SlideBbb

```
scalaVersion := "2.13.4"

libraryDependencies ++= Seq(
  "org.apache.poi"    %  "poi"                     % "3.16",
  "org.apache.poi"    %  "poi-ooxml"               % "3.16",
  "com.monitorjbl"    %  "xlsx-streamer"           % "1.2.0",
  "com.typesafe.akka" %% "akka-stream"             % "2.5.6",
  "org.scalikejdbc"   %% "scalikejdbc"             % "3.2.0",
  "org.scalikejdbc"   %% "scalikejdbc-config"      % "3.2.0",
  "org.scalikejdbc"   %% "scalikejdbc-streams"     % "3.2.0",
  "com.h2database"    %  "h2"                      % "1.4.196",
  "ch.qos.logback"    %  "logback-classic"         % "1.2.3"
)
```
@[1](scala versionを指定する)

---

## SlideCcc

---

### SlideCcc

---

#### SlideCcc

---

##### SlideCcc
