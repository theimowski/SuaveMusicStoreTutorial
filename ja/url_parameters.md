URLパラメーター
===============

パス文字列を静的に指定するだけではなく、
ルートに引数を指定することも出来ます。
Suaveには「型付きルート」という素晴らしい機能が備わっており、
静的に型付けされた状態でルートに対する引数を制御できます。
たとえばアルバムの`id`をdetailsルートに指定するコードは以下のようになります：

````fsharp
pathScan "/store/details/%d" (fun id -> OK (sprintf "詳細: %d" id))
````

このコードはC++の書式付きprint構文に似ていますが、それよりも強力です。
ここでは、コンパイラが引数 `%d` に対応する型をチェックして、
整数ではない値が渡されていないかどうか確認するという処理が行われます。
上のWebPartは `http://localhost:8083/store/details/28` というような
リクエストに対応します。
このコードの要点は以下の通りです：

* `sprintf "詳細: %d" id` は静的に型付けされた文字列フォーマット関数で、
  idには整数が期待されます
* `(fun id -> OK ...)` は匿名関数あるいはラムダ式で、 `int -> WebPart` という型です
* `pathScan` 関数の2番目の引数としてラムダ式が指定されています
* `pathScan` の1番目の引数もやはり静的型付けされます
* これらはF#に組み込まれた型推論メカニズムによって連携されているため、
  型のシグネチャを指定する必要がありません

もう少し分かりやすくなるように、Suaveの型付きルートの例をもう1つ紹介しましょう：

````fsharp
pathScan "/store/details/%s/%d" (fun (a, id) -> OK (sprintf "アーティスト: %s, Id: %d" a id))
````

たとえば `http://localhost:8083/store/details/abba/1` としてアクセスできます。

静的に型付けされた文字列を活用する方法については
[こちらのサイト][stringsinastaticallytypedway] を参照してください。

[stringsinastaticallytypedway]: http://fsharpforfunandprofit.com/posts/printf/
