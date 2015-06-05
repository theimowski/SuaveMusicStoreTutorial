# Registration and checkout

There's already a bunch of nice features in our app, however we still lack of registration.
Currently it's only admin account without possibility to create new users.
It would be a pity if no one can register, because only registered users can buy albums in our store!







So let's take a deep breath and roll to the last significant feature in our application!

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

Now let's add a route for checkout (`Path.Cart`):

```fsharp
let checkout = "/cart/checkout"
```

a navigation button in `View.nonEmptyCart` :

```fsharp
let nonEmptyCart (carts : Db.CartDetails list) = [
    h2 "Review your cart:"
    pAttr ["class", "button"] [
            aHref Path.Cart.checkout (text "Checkout >>")
    ]

    ...
```

and GET handler for checkout in `App` module:

```fsharp
let checkout =
    session (function
    | NoSession | CartIdOnly _ -> never
    | UserLoggedOn {Username = username } ->
        choose [
            GET >>= (View.checkout |> html)
        ])
```

```fsharp
path Path.Cart.checkout >>= loggedOn checkout
```

Remarks:

- `checkout` can only be invoked if user is logged on. If there is `NoSession` or `CartIdOnly` we don't want to let anyone in.
- additionally we decorated the `checkout` WebPart with `loggedOn` in the routing code. This ensures to redirect user to logon form if necessary.

Now it's time to implement `placeOrder` function in the `Db` module:

```fsharp
let placeOrder (username : string) (ctx : DbContext) =
    let carts = getCartsDetails username ctx
    let total = carts |> List.sumBy (fun c -> (decimal) c.Count * c.Price)
    let order = ctx.``[dbo].[Orders]``.Create(DateTime.UtcNow, total)
    order.Username <- username
    ctx.SubmitUpdates()
    for cart in carts do
        let orderDetails = ctx.``[dbo].[OrderDetails]``.Create(cart.AlbumId, order.OrderId, cart.Count, cart.Price)
        getCart cart.CartId cart.AlbumId ctx
        |> Option.iter (fun cart -> cart.Delete())
    ctx.SubmitUpdates()
```

Explanation of the above snippet:

1. we retrieve all carts that belong to the given user
2. we count the total amount of cash to pay
3. a new order is created with current timestamp, total, and username
4. we invoke `SubmitUpdates` so that the `order` instance gets its `OrderId` property filled in
5. we iterate through all the carts and for each of them:
    1. we create `OrderDetails` with all properties
    2. we delete the corresponding `cart` from database. Note that we cannot delete the cart which is currently being iterated through, because it represents a row from database view `CartDetails`, not the table `Cart` itself.
6. finally we `SubmitUpdates` at the very end.

When user checks out the cart, we want to show him a following confirmation (`View` module):

```fsharp
let checkoutComplete = [
    h2 "Checkout Complete"
    p [
        text "Thanks for your order!"
    ]
    p [
        text "How about shopping for some more music in our "
        aHref Path.home (text "store")
        text "?"
    ]
]
```

With that in place, we can now implement the POST checkout handler:

```fsharp
POST >>= warbler (fun _ ->
    let ctx = Db.getContext()
    Db.placeOrder username ctx
    View.checkoutComplete |> html)
```

We need to invoke the `Db.placeOrder` in a warbler, to ensure that it's not called for the GET handler as well.

Phew, I think we're done for now!

Browse the code here: [Tag - registration_and_checkout](https://github.com/theimowski/SuaveMusicStore/tree/registration_and_checkout)