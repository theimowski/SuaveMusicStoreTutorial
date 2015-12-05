# 購入フォーム

まずは深呼吸して、アプリケーションの最後の山場にとりかかることにしましょう！

今回も標準的なフォームとして購入フォームを用意します：
Again, checkout will be based on a form:

```fsharp
type Checkout = {
    FirstName : string
    LastName : string
    Address : string
    PromoCode : string option
}

let checkout : Form<Checkout> = Form ([], [])
```

最初の3つは必須で、最後の「PromoCode」は省略可能(`string option`)になっていることに注意してください。

購入用のフォームは以下の通りです：

```fsharp
let checkout = [
    h2 "住所および支払情報"
    renderForm
        { Form = Form.checkout 
          Fieldsets = 
              [ { Legend = "お届け先情報"
                  Fields = 
                      [ { Label = "名前(名)"
                          Xml = input (fun f -> <@ f.FirstName @>) [] }
                        { Label = "名前(姓)"
                          Xml = input (fun f -> <@ f.LastName @>) [] }
                        { Label = "住所"
                          Xml = input (fun f -> <@ f.Address @>) [] } ] }
                
                { Legend = "支払情報"
                  Fields = 
                      [ { Label = "プロモーションコード"
                          Xml = input (fun f -> <@ f.PromoCode @>) [] } ] } ]
          SubmitText = "注文する"
        }
]
```

前回は1つでしたが、今回は2つのフィールドセットを使用しています。
フィールドセットを使用して、一般的なフィールドを1つのグループにまとめて表示しています。
