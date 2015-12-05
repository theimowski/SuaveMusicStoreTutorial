# Session型

ここまでの変更により、ユーザー「admin」パスワード「admin」で認証出来るようになりました。
しかしユーザーが認証されていることを必須とするようなハンドラーがまだ無いので、あまり便利になった感じがしません。

そこでまず、ユーザーの状態を表す型を`App`モジュールに追加します：

```fsharp
type UserLoggedOnSession = {
    Username : string
    Role : string
}

type Session = 
    | NoSession
    | UserLoggedOn of UserLoggedOnSession
```

型`Session` はいわゆるF#の「判別共用体(Discriminated Union)」です。
つまり`Session`型のインスタンスは`NoSession`か`UserLoggedOn`のいずれかになり、その他の値にはならないということです。
`of UserLoggedOnSession`の部分は`UserLoggedOnSession`型に関連するデータがあるということを表します。
判別共用体の詳細については[こちら](http://fsharpforfunandprofit.com/posts/discriminated-unions/)を参照してください。

一方で、`UserLoggedOnSession`は「レコード(Record)型」です。
レコード型はPOCO(Plain-Old-Class Object: 昔ながらの素朴なクラスのオブジェクト)、あるいはDTO、あるいはそういった何かとみなすことができます。
しかしF#にあらかじめ組み込まれた多数の機能のおかげで、レコード型は大変役立つものになっています。具体的には以下の通りです：

- デフォルトで不変性(immutability)を持ちます
- デフォルトで構造的同値性(structural equality)を持ちます
- パターンマッチに対応します
- コピーして変更(copy-and-update)するための式があります

繰り返しますが、レコード型の詳細を学びたい場合には是非[こちら](http://fsharpforfunandprofit.com/posts/records/)の記事を参照してください。

これら2つの型を使用することによって、ユーザーがアプリケーションにログインしているのか、していないのか区別することができるようになります。

既に宣言しておいたように、`session`関数に引数を1つ追加します：

```fsharp
let session f = 
    statefulForSession
    >>= context (fun x -> 
        match x |> HttpContext.state with
        | None -> f NoSession
        | Some state ->
            match state.get "username", state.get "role" with
            | Some username, Some role -> f (UserLoggedOn {Username = username; Role = role})
            | _ -> f NoSession)
```

`f`の型は`Session -> WebPart`です。
ご想像の通り、ここではユーザーのセッション状態に応じて異なるレスポンスを返す、あるいは異なる動作が行えるようにしています。
ユーザーがログイン済みかどうかチェックするためには、セッションステートストアに「username」と「role」の両方の値が含まれている必要があります。

> 注釈：上のコードではタプルに対するパターンマッチを行っています。`match`に対してカンマで区切った2つの値を指定していて、両方の値に対してパターンマッチを実行しています。`Some username, Some role`だと両方の値が存在することを表します。続くパターン(`_`)はその他すべてのケースに一致します。

`session`の用途は今のところ`logon`のPOSTハンドラーだけです。新しいバージョンに書き換えましょう：

```fsharp
...
Auth.authenticated Cookie.CookieLife.Session false 
>>= session (fun _ -> succeed)
>>= sessionStore (fun store ->
...
```

わかっています。`session`関数には手の込んだ何かを指定するという話だったはずですが、今は我慢してください。後でちゃんとやります。
今のところ`logon`の`session`には特別な処理が必要ありませんが、ユーザーの状態を「初期化(initialize)」するために、この関数を呼び出す必要があります。
