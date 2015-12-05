# 登録用フォーム

登録用のフォームは標準的なものを元にするため、`Form`モジュールにコードを追加していきます：

```fsharp
open System.Net.Mail
```

```fsharp
type Register = {
    Username : string
    Email : MailAddress
    Password : Password
    ConfirmPassword : Password
}

let pattern = @"(\w){6,20}"

let passwordsMatch = 
    (fun f -> f.Password = f.ConfirmPassword), "同じパスワードを入力してください"

let register : Form<Register> = 
    Form ([ TextProp ((fun f -> <@ f.Username @>), [ maxLength 30 ] )
            PasswordProp ((fun f -> <@ f.Password @>), [ passwordRegex pattern ] )
            PasswordProp ((fun f -> <@ f.ConfirmPassword @>), [ passwordRegex pattern ] )
            ],[ passwordsMatch ])
```

上のコードでは：

- 型`MailAddress`を使用するために`System.Net.Mail` 名前空間をオープンしています
- フォームには4つのフィールドがあります：
    - `string`型の`Username`
    - `MailAddress`型の`Email` (フィールドに対して特定の検証を行います)
    - `Password`型の`Password` (特定のHTML入力フォームになります)
    - 同じく`Password`型の`ConfirmPassword`
- `pattern` はパスワードに対する正規表現です
- `passwordsMatch` はサーバーサイドだけで実行される検証用関数です
- `register` はフォームの定義で、以下の制約を課しています：
    - `Username` が最大30文字であること
    - `Password` が正規表現に一致すること
    - `ConfirmPassword` が正規表現に一致すること

`passwordMatch`のような、サーバーサイドだけで実行される検証関数は`('FormType -> bool) * string`型にします。
つまり判定用の関数と、エラー用の文字列とのタプルです。
こういった検証用関数を多数用意して、`Form`の定義に渡す事ができます。
これらの関数は1つ以上のフィールドを参照したり、複雑なロジックを必要とするような検証機能で使用することができます。
なおこのチュートリアルではパスワードが一致しているかをクライアントサイドでチェックする機能を実装しませんが、JavScriptコードを少し用意するだけで実装できます。

フォームの定義が用意出来たので、`View`の実装を続けていきましょう：

```fsharp
let register msg = [
    h2 "アカウントの新規作成"
    p [
        text "以下のフォームを使用してアカウントを作成してください。"
    ]
    
    divId "register-message" [
        text msg
    ]

    renderForm
        { Form = Form.register
          Fieldsets = 
              [ { Legend = "アカウントの新規作成"
                  Fields = 
                      [ { Label = "ユーザー名(30文字以内)"
                          Xml = input (fun f -> <@ f.Username @>) [] }
                        { Label = "メールアドレス"
                          Xml = input (fun f -> <@ f.Email @>) [] }
                        { Label = "パスワード(6文字以上、20文字以内)"
                          Xml = input (fun f -> <@ f.Password @>) [] }
                        { Label = "パスワードの再入力"
                          Xml = input (fun f -> <@ f.ConfirmPassword @>) [] } ] } ]
          SubmitText = "登録" }
]
```

見ての通り、`View.logon`と同じくエラーメッセージが表示出来るようにするために、`msg`引数を指定出来るようにしています。
残りの部分についてはコードの通りです。
