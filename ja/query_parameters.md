クエリパラメーター
==================

ルート自身の引数としてパラメーターを指定するだけではなく、
`localhost:8083/store/browse?genre=Disco` というように、
URLのクエリ部として引数を指定することもできます。
そのために、別のWebPartを作成します：

````fsharp
let browse =
    request (fun r ->
        match r.queryParam "genre" with
        | Choice1Of2 genre -> OK (sprintf "ジャンル: %s" genre)
        | Choice2Of2 msg -> BAD_REQUEST msg)
````

`request` は `HttpRequest -> WebPart` という型の引数を1つとる関数です。
別の関数を引数にとる関数は「高階関数」とも呼ばれます。
ラムダ内の `r` は `HttpRequest` を表します。
この型には `string -> Choice<string,string>` 型のメンバー関数 `queryParam` が
定義されています。
`Choice` は2つの型のいずれかをとる型を表します。
通常 `Choice` の1つめの型はハッピーパス(正常系)、2つめの型は異常系とします。
今回の場合、1つめの文字列はクエリパラメーターを表す値で、
2つめの文字列はエラーメッセージを表す値です
(クエリ中に特定のキーを持ったパラメーターが指定されなかったことを意味します)。
2つの値のいずれかになったのかを区別するには、パターンマッチを使用します。
パターンマッチも非常に強力な機能で、最近のプログラミング言語の
ほとんどで実装されています。
今のところはswitch式と、値を識別子に束縛する機能が同時に行えるもの、
という程度にとらえておいてください。
また、F#コンパイラは取り得る条件(今回の場合は `Choice1Of2` と `Choice2Of2`)が
網羅されていなかった場合に警告を出します。
後ほど説明しますが、パターンマッチにはさらなる機能もあります。
`BAD_REQUEST` はSuaveライブラリの関数で、応答本文に特定の文字列を含んだ
400 Bad Requestステータスを返すWebPartです
(`Suave.Http.RequestErrors` をオープンする必要があります)。
`browse` WebPartの動作は、要約すると次の通りです：
もしURLクエリに「genre」パラメーターがある場合は「genre」の値とともに
200 OKを返します。
そうでなければエラーメッセージとともに400 Bad Requestを返します。

この`browse` WebPartは次のようにして既存のWebPartへ組み込む事ができます：

````fsharp
path "/store/browse" >=> browse
````

これまでの変更をまとめると [Tag - basic_routing][tag_basicrouting] のようになります。

[tag_basicrouting]: https://github.com/theimowski/SuaveMusicStore/tree/basic_routing
([Suave 0.28.1](https://github.com/SuaveIO/suave/tree/v0.28.1))
