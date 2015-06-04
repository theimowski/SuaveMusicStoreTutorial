# Registration and checkout

There's already a bunch of nice features in our app, however we still lack of registration.
Currently it's only admin account without possibility to create new users.
It would be a pity if no one can register, because only registered users can buy albums in our store!

Register feature will be based on a standard form, so let's add one to the `Form` module:

```
open System.Net.Mail
```

```
type Register = {
    Username : string
    Email : MailAddress
    Password : Password
    ConfirmPassword : Password
}

let pattern = @"(\w){6,20}"

let passwordsMatch = 
    (fun f -> f.Password = f.ConfirmPassword), "Passwords must match"

let register : Form<Register> = 
    Form ([ TextProp ((fun f -> <@ f.Username @>), [ maxLength 30 ] )
            PasswordProp ((fun f -> <@ f.Password @>), [ passwordRegex pattern ] )
            PasswordProp ((fun f -> <@ f.ConfirmPassword @>), [ passwordRegex pattern ] )
            ],[ passwordsMatch ])
```

In the above snippet:

- we open the `System.Net.Mail` namespace to use the `MailAddress` type
- the form consists of 4 fields:
    - `Username` of `string`
    - `Email` of type `MailAddress` (ensures proper validation of the field)
    - `Password` of type `Password` (ensures proper HTML input type)
    - `ConfirmPassword` of the same type
- `pattern` is a regular expression pattern for password
- `passwordsMatch` is a server-side only validation function
- `register` is the definition of our form, with a few constraints:
    - `Username` must be at most 30 characters long
    - `Password` must match the regular expression
    - `ConfirmPassword` must match the regular expression

Server-side only validation, like `passwordMatch` are of type `('FormType -> bool) * string`.
So this is just a tuple of a predicate function and a string error.
We can create as many validations as we like, and pass them to the `Form` definition.
These can be used for validations that lookup more than one field, or require some complex logic.
We won't create client-side validation to check if the passwords match in the tutorial, but it could be achieved with some custom JavaScript code.

With the form definition in place, let's proceed to `View`:

```
let register msg = [
    h2 "Create a New Account"
    p [
        text "Use the form below to create a new account."
    ]
    
    divId "register-message" [
        text msg
    ]

    renderForm
        { Form = Form.register
          Fieldsets = 
              [ { Legend = "Create a New Account"
                  Fields = 
                      [ { Label = "User name (max 30 characters)"
                          Xml = input (fun f -> <@ f.Username @>) [] }
                        { Label = "Email address"
                          Xml = input (fun f -> <@ f.Email @>) [] }
                        { Label = "Password (between 6 and 20 characters)"
                          Xml = input (fun f -> <@ f.Password @>) [] }
                        { Label = "Confirm password"
                          Xml = input (fun f -> <@ f.ConfirmPassword @>) [] } ] } ]
          SubmitText = "Register" }
]
```

As you can see, we're using the `msg` parameter here similar to how it was done in `View.logon` to include possible error messages.
The rest of the snippet is rather self-explanatory.

We're now left with proper `Path.Account` entry:

```
let register = "/account/register"
```

GET handler for registration in `App`:

```
let register =
    choose [
        GET >>= (View.register "" |> html)
    ]
```

```
path Path.Account.register >>= register
```

and a direct link from the `View.logon` :

```
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

```
let getUser username (ctx : DbContext) : User option = 
    query {
        for user in ctx.``[dbo].[Users]`` do
        where (user.UserName = username)
        select user
    } |> firstOrNone
```

```
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

```
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

```
match Db.validateUser(form.Username, passHash password) ctx with
| Some user ->
    authenticateUser user
| _ ->
    View.logon "Username or password is invalid." |> html
```

Finally, the full register handler can be implemented following:

```
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

```
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

```
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

```
let checkout = "/cart/checkout"
```

a navigation button in `View.nonEmptyCart` :

```
let nonEmptyCart (carts : Db.CartDetails list) = [
    h2 "Review your cart:"
    pAttr ["class", "button"] [
            aHref Path.Cart.checkout (text "Checkout >>")
    ]

    ...
```

and GET handler for checkout in `App` module:

```
let checkout =
    session (function
    | NoSession | CartIdOnly _ -> never
    | UserLoggedOn {Username = username } ->
        choose [
            GET >>= (View.checkout |> html)
        ])
```

```
path Path.Cart.checkout >>= loggedOn checkout
```

Remarks:

- `checkout` can only be invoked if user is logged on. If there is `NoSession` or `CartIdOnly` we don't want to let anyone in.
- additionally we decorated the `checkout` WebPart with `loggedOn` in the routing code. This ensures to redirect user to logon form if necessary.

Now it's time to implement `placeOrder` function in the `Db` module:

```
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

```
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

```
POST >>= warbler (fun _ ->
    let ctx = Db.getContext()
    Db.placeOrder username ctx
    View.checkoutComplete |> html)
```

We need to invoke the `Db.placeOrder` in a warbler, to ensure that it's not called for the GET handler as well.

Phew, I think we're done for now!

Browse the code here: [Tag - registration_and_checkout](https://github.com/theimowski/SuaveMusicStore/tree/registration_and_checkout)