# Checkout form

Let's take a deep breath and roll to the last significant feature in our application!

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

Note that, while first three fields are mandatory, the last one "PromoCode" is optional (it's type is `string option`).

Following is the view for the checkout form:

```fsharp
let checkout = [
    h2 "Address And Payment"
    renderForm
        { Form = Form.checkout 
          Fieldsets = 
              [ { Legend = "Shipping Information"
                  Fields = 
                      [ { Label = "First Name"
                          Xml = input (fun f -> <@ f.FirstName @>) [] }
                        { Label = "Last Name"
                          Xml = input (fun f -> <@ f.LastName @>) [] }
                        { Label = "Address"
                          Xml = input (fun f -> <@ f.Address @>) [] } ] }
                
                { Legend = "Payment"
                  Fields = 
                      [ { Label = "Promo Code"
                          Xml = input (fun f -> <@ f.PromoCode @>) [] } ] } ]
          SubmitText = "Submit Order"
        }
]
```

Note that there are 2 fieldsets here, whereas before we used only 1.
Thanks to multiple fieldsets we can group fields that have something in common.