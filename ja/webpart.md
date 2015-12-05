WebPart
=======

先ほどのコードが実際には何をしているのか、気になる方もいることでしょう。
`startWebServer` はSuaveライブラリで定義されている関数で、
`SuaveConfig -> WebPart -> unit` という型になっています。
簡単に説明すると、1番目の引数に `SuaveConfig`、2番目の引数に `WebPart` をとり、
`unit` を返すということです。

`defaultConfig` は同じくSuaveライブラリ内で定義された `SuaveConfig` 型の値で、
その名前からも分かるとおり、サーバーのデフォルト設定を表します。
今の時点では、これがHTTPバインディングを構成すること、
すなわちループバックアドレス 127.0.0.1のポート8083番でリッスンするよう
サーバーが構成されるという点だけ把握しておいてください。

F#の `unit` は、唯一 `()` という値だけが対応する型です。
これはC#の `void` に相当しますが、
C#の場合は `void` を値として返すことは出来ないという違いがあります。
`unit` 型、ならびに `void` との違いの詳細を知りたい方は
[こちらのリンク][difffromvoid] を参照してください。

一番興味深いのは `WebPart` です。
`WebPart` は `HttpContext -> Async<HttpContext option>` という型のエイリアスです。
つまり `WebPart` は実際には関数で、1番目の引数に `HttpContext` をとり、
`HttpContext option` 型の「非同期ワークフロー(async workflow)」を返すものです。

`HttpContext` には受信したリクエストに関する情報や、
送信するレスポンスに関する情報、およびその他のデータが含まれます。
`Async` は非同期ワークフローとも呼ばれますが、
これは別の言語では「promise(プロミス)」あるいは「future(フューチャー)」とも
呼ばれるものです。
C# 5ではAsync/Awaitの機能が導入されましたが、F#のAsyncはある意味これと似たものです。
F#における非同期ワークフローや一般的な非同期プログラミングについては
[こちら][fsharpasync] が参考になるでしょう。
F#ではnullが許容されませんが、これはF#がC#より優れているところの1つだといえます。
F#コンパイラはnullを引数として渡すことを許可しません。
したがって、かの悪名高き `NullReferenceException` には対処する必要がないのです。
しかし値があったり無かったりするようなものを表すにはどうしたらよいのでしょうか？
その場合には `Option` 型を使用します。
`Option` 型のオブジェクトは `None` または `Some x` のいずれかになります。
`None` は値が無いことを、 `Some x` は `x` という値があり、
`x` がnullではないことを表します。

まとめると `WebPart` は以下のように説明できます：

> httpコンテキストを元にして、結果となるhttpコンテキストを(非同期的に)返す、
> あるいは返さないという約束をする。なお結果コンテキストには
> WebPart自身のロジックから得られる一連の応答が含まれることになる。

今回の場合、`(OK "Hello World!")`という、`WebPart` が返し得る
最も単純なケースが該当することになります。
HttpContextに何が含まれていようとも、常に`200 OK`というHTTP結果コードと、
「Hello World!」という応答本文を含んだ、非同期の`Some` `HttpContext`
が返されます。

ある意味では、`WebPart` を ASP.NET MVCアプリケーションにおける `Filter`
と見なす事ができますが、 `WebPart` は `Filter` よりも多機能です。

もし上記の説明でも `WebPart` があまりしっくりこない、
あるいはなぜ上のようなシグネチャになっているのかよく分からないとしたら、
それは筆者の責任です。
次の節では、関数プログラミングのパラダイムの1つである
**合成(composition)** と組み合わせると、この型が非常に便利なものであることを
説明しようと思います。

[difffromvoid]: https://msdn.microsoft.com/en-us/library/dd483472.aspx
[fsharpasync]: http://fsharpforfunandprofit.com/posts/concurrency-async-and-parallel/
