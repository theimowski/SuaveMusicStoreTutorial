# Registration and checkout

There's already a bunch of nice features in our app, however we still lack of registration.
Currently it's only admin account without possibility to create new users.
It would be a pity if no one can register, because only registered users can buy albums in our store!





We're now left with proper `Path.Account` entry:

```fsharp
let register = "/account/register"
```

GET handler for registration in `App`:

```fsharp
let register =
    choose [
        GET >>= (View.register "" |> html)
    ]
```

```fsharp
path Path.Account.register >>= register
```

and a direct link from the `View.logon` :

```fsharp
let logon msg = [
    h2 "Log On"
    p [
        text "Please enter your user name and password."
        aHref Path.Account.register (text " Register")
        text " if you don't have an account yet."
    ]

    ...
```

This allows us to navigate to the registration form.
Moving on to implementing the actual POST handler, let's first create necessary functions in `Db` module:

```fsharp
let getUser username (ctx : DbContext) : User option = 
    query {
        for user in ctx.``[dbo].[Users]`` do
        where (user.UserName = username)
        select user
    } |> firstOrNone
```

```fsharp
let newUser (username, password, email) (ctx : DbContext) =
    let user = ctx.``[dbo].[Users]``.Create(email, password, "user", username)
    ctx.SubmitUpdates()
    user
```

`getUser` will be crucial to check if a user with given `username` already exists in the database - we don't want two users with the same `username`.
`newUser` is a simple function that creates and returns the new user.
Note that we hardcode "user" for each new user's role. 
This way, they can be distinguished from admin's role.

After a successful registration, we'd like to authenticate user at once - in other words apply the same logic which happens after successful logon.
In a real application, you'd probably use a confirmation mail mechanism, but for the sake of simplicity we'll skip that.
In order to reuse the logic from logon POST handler, extract a separate function:

```fsharp
let authenticateUser (user : Db.User) =
    Auth.authenticated Cookie.CookieLife.Session false 
    >>= session (function
        | CartIdOnly cartId ->
            let ctx = Db.getContext()
            Db.upgradeCarts (cartId, user.UserName) ctx
            sessionStore (fun store -> store.set "cartid" "")
        | _ -> succeed)
    >>= sessionStore (fun store ->
        store.set "username" user.UserName
        >>= store.set "role" user.Role)
    >>= returnPathOrHome
```

after extraction, `logon` POST handler looks like this:

```fsharp
match Db.validateUser(form.Username, passHash password) ctx with
| Some user ->
    authenticateUser user
| _ ->
    View.logon "Username or password is invalid." |> html
```

Finally, the full register handler can be implemented following:

```fsharp
let register =
    choose [
        GET >>= (View.register "" |> html)
        POST >>= bindToForm Form.register (fun form ->
            let ctx = Db.getContext()
            match Db.getUser form.Username ctx with
            | Some existing -> 
                View.register "Sorry this username is already taken. Try another one." |> html
            | None ->
                let (Password password) = form.Password
                let email = form.Email.Address
                let user = Db.newUser (form.Username, passHash password, email) ctx
                authenticateUser user
        )
    ]
```

Comments for POST part:

- bind to `Form.register` to validate the request
- check if a user with given `username` already exists
- if that's the case then show the `View.register` form again with a proper error message
- otherwise read the form fields' values, create new user and invoke the `authenticateUser` function

This concludes register feature - we're now set for new customers to do the shopping.
Wait a second, they can put albums to the cart, but how do they checkout?
Ah yes, we haven't implemented that yet.

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