基本的なルーティング
====================

WebPartを拡張して、複数のルートをサポートできるようにしましょう。
まずWebPartを展開して、1つの識別子に束縛(bind:バインド)するよう変更します。
つまり `let webPart = OK "Hello World"` とした後、
この識別子を使用して `startWebServer defaultConfig webPart` というコードで
`startWebServer` 関数を呼び出すようにします。
C#の場合は「webPartを1つの変数に割り当てて」というような言い方になるでしょうが、
関数型の世界においては変数という概念が存在しません。
その代わり、値を識別子に「束縛」することで値を再利用できます。
束縛された値を実行時に変更することは出来ません。
ではWebPartに制限を追加して、アプリケーションのルートパスに対してのみ
「Hello World」を返すようにしましょう。
(`localhost:8083/`はOKで`localhost:8083/anything`には返さないようにします。)
`let webPart = path "/" >>= OK "Hello World"` とします。
`path` は `Suave.Http.Applicatives` モジュールで定義されているため、
`App.fs` の先頭でこのモジュールをオープンする必要があります。
`Suave.Http` と `Suave.Types` も重要です。
併せてオープンしておきます。

`path` は `string -> WebPart` という型の関数です。
つまり文字列を指定するとWebPartが返されます。
実際のところは、受信したリクエストがパスに一致する場合には `Some` 、
そうでなければ `None` が返されます。
`>>=` 演算子もやはりSuaveのライブラリで定義されているものです。
この演算子は左右のWebPartを1つのWebPartとするもので、
左辺に指定されたWebPartが `Some` を返した場合にのみ、
引き続いて右辺のWebPartを適用します。

続けて、`choose` 関数を使用していくつかのルートを
アプリケーションに追加してみましょう。
この関数にWebPartのリストを渡すと、最初に適用された
(`Some`を返した)WebPartが選択されます。
適用されたWebPartが一つも無かった場合には、やはり `None` が返されます：

````fsharp
let webPart =
    choose [
        path "/" >>= (OK "Home")
        path "/store" >>= (OK "Store")
        path "/store/browse" >>= (OK "Browse")
        path "/store/details" >>= (OK "Details")
    ]
````
