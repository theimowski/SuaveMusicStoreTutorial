Pathモジュール
==============

アプリケーションの他のビューを定義する前に、もう1つ `Path.fs` というファイルを
`View.fs` の**前に**追加しましょう：

````fsharp
module SuaveMusicStore.Path

type IntPath = PrintfFormat<(int -> string), unit, string, string, int>

let home = "/"

module Store =
    let overview = "/store"
    let browse = "/store/browse"
    let details : IntPath = "/store/details/%d"
````

このモジュールにはアプリケーションで有効なすべてのルートが含まれることになります。
ルートを一カ所で定義するようにすることで、`App`と`View`の両方のモジュールで
再利用出来るようにしておきます。
そうすることにより、`View`モジュールでタイプミスをするリスクを最小限に抑えられます。
ここでは1つの機能に関連するルートをグループ化するために、
`Store`というサブモジュールを定義しています。
チュートリアルの後半ではさらにサブモジュールを追加しますが、
それぞれアプリケーションの特定の機能セットを反映することになります。

型エイリアスとして定義されている `IntPath` は、静的に型付けされたSuaveのルート
(`App`モジュール内の`pathScan`)と、ルートの定義とをつなぐためのものです。
この型のシグネチャを完全に理解する必要はありません。
今のところは整数値を引数にとるようなルートを表す型だと思ってください。
実際、`details`にはこの型を明記しているため、
コンパイラがこの値を*特別扱い*するようになります。
`details`が型付けされているという利点を生かしつつ、
`App`や`View`モジュールで使用する方法については後ほど紹介します。

では`App`内で`Path`モジュールを使用してみましょう：

````fsharp
let webPart =
    choose [
        path "/" >=> (OK View.index)
        path "/store" >=> (OK "Store")
        path "/store/browse" >=> browse
        pathScan Path.Store.details (fun id -> OK (sprintf "詳細 %d" id))
    ]
````

同じく`View`の`aHref`でも`home`を指定するようにします：

````fsharp
divId "header" [
    h1 (aHref Path.home (text "F# Suave Music Store"))
]
````

`App`モジュールでは以前と同じく、Suaveの機能である
静的に型付けされたルーティング機能を活用できています。
つまり引数`id`はコンパイラによって整数型だと推論されます。
型推論のメカニズムに詳しくないのであれば、
[こちらのリンク][typeinference]
を参照されるとよいでしょう。

[typeinference]: http://fsharpforfunandprofit.com/posts/type-inference/
