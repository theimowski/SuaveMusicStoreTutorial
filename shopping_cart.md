# Shopping cart

What's a shop without cart feature?
We would like to let the user add albums to a cart while shopping at our Store.
To fill in the gap, let's start by declaring new routes in `Path` module:

```fsharp
module Cart =
    let overview = "/cart"
    let addAlbum : IntPath = "/cart/add/%d"
    let removeAlbum : IntPath = "/cart/remove/%d"
```

Before we move to the `View`, add new type annotation for yet another database view `CartDetails`.
`CartDetails` is a view that joins `Cart` with its corresponding `Album` in order to contain album's title and its price.

```fsharp
type CartDetails = DbContext.``[dbo].[CartDetails]Entity``
```







We still have to recognize the `CartIdOnly` case - coming back to the `session` function, the pattern matching handling `CartIdOnly` looks like this:

```fsharp
match state.get "cartid", state.get "username", state.get "role" with
| Some cartId, None, None -> f (CartIdOnly cartId)
| _, Some username, Some role -> f (UserLoggedOn {Username = username; Role = role})
| _ -> f NoSession
```

This means that if the session store contains `cartid` key, but no `username` or `role` then we invoke the `f` parameter with `CartIdOnly`.

To fetch the actual `CartDetails` for a given `cartId`, let's define appropriate function in `Db` module:

```fsharp
let getCartsDetails cartId (ctx : DbContext) : CartDetails list =
    query {
        for cart in ctx.``[dbo].[CartDetails]`` do
            where (cart.CartId = cartId)
            select cart
    } |> Seq.toList
```

This allows us to implement `cart` handler correctly:

```fsharp
let cart = 
    session (function
    | NoSession -> View.emptyCart |> html
    | UserLoggedOn { Username = cartId } | CartIdOnly cartId ->
        let ctx = Db.getContext()
        Db.getCartsDetails cartId ctx |> View.cart |> html)
```

Again, we use two different patterns for the same behavior here - `CartIdOnly` and `UserLoggedOn` states will query the database for cart details.

Remember our navigation menu with `Cart` tab? 
Why don't we add a number of total albums in our cart there?
To do that, let's parametrize the `partNav` view:

```fsharp
let partNav cartItems = 
    ulAttr ["id", "navlist"] [ 
        li (aHref Path.home (text "Home"))
        li (aHref Path.Store.overview (text "Store"))
        li (aHref Path.Cart.overview (text (sprintf "Cart (%d)" cartItems)))
        li (aHref Path.Admin.manage (text "Admin"))
    ]
```

as well as add the `partNav` parameter to the main `index` view:

```fsharp
let index partNav partUser container = 
```

In order to create the navigation menu, we now need to pass `cartItems` parameter.
It has to be resolved in the `html` function from `App` module, which can now look like following:

```fsharp
let html container =
    let ctx = Db.getContext()
    let result cartItems user =
        OK (View.index (View.partNav cartItems) (View.partUser user) container)
        >>= Writers.setMimeType "text/html; charset=utf-8"

    session (function
    | UserLoggedOn { Username = username } -> 
        let items = Db.getCartsDetails username ctx |> List.sumBy (fun c -> c.Count)
        result items (Some username)
    | CartIdOnly cartId ->
        let items = Db.getCartsDetails cartId ctx |> List.sumBy (fun c -> c.Count)
        result items None
    | NoSession ->
        result 0 None)
```

The change is that the nested `result` function in `html` now takes two arguments: `cartItems` and `user`. 
Additionally, we handle all cases inside the `session` invocation.
Note how the `result` function is passed different set of parameters in different cases.

Now that we can add albums to the cart, and see the total number both in `cart` overview and navigation menu, it would be nice to be able to remove albums from the cart as well.
This is a great occasion to employ AJAX in our application.
We'll write a simple script in JS that makes use of jQuery to remove selected album from the cart and update the cart view.

Download jQuery from [here](https://jquery.com/download/) (I used the compressed / minified version) and add it to the project.
Don't forget to set the "Copy to Output Directory" property.

Now add new JS file to the project `script.js`, and fill in its contents:

```fsharp
$('.removeFromCart').click(function () {
    var albumId = $(this).attr("data-id");
    var albumTitle = $(this).closest('tr').find('td:first-child > a').html();
    var $cartNav = $('#navlist').find('a[href="/cart"]');
    var count = parseInt($cartNav.html().match(/\d+/));

    $.post("/cart/remove/" + albumId, function (data) {
        $('#container').html(data);
        $('#update-message').html(albumTitle + ' has been removed from your shopping cart.');
        $cartNav.html('Cart (' + (count - 1) + ')');
    });
});
```

We won't go into much details about the code itself, however it's important to know that the script:

- subscribes to click event on each `removeFromCart` element
- parses information such as:
    - album id
    - album title
    - count of this album in cart
- sends a POST request to "/cart/remove" endpoint 
- upon successful POST response it updates:
    - html of the container element
    - message, to indicate which album has been removed
    - navigation menu to decrement count of albums

The `update-message` div should be added to the `nonEmptyCart` view, before the table:

```fsharp
    divId "update-message" [text " "]
```

We explicitly have to pass in non-empty text, because we cannot have an empty div element in HTML markup.
With jQuery and our `script.cs` files, we can now attach them to the end of `nonEmptyCart` view, just after the table:

```fsharp
scriptAttr [ "type", "text/javascript"; " src", "/jquery-1.11.3.min.js" ] [ text "" ]
scriptAttr [ "type", "text/javascript"; " src", "/script.js" ] [ text "" ]
```

We also need to allow browsing for files with "js" extension in our handler:

```fsharp
pathRegex "(.*)\.(css|png|gif|js)" >>= Files.browseHome
```

The script tries to reach route that is not mapped to any handler yet.
Let's change that by first adding `removeFromCart` to `Db` module:

```fsharp
let removeFromCart (cart : Cart) albumId (ctx : DbContext) = 
    cart.Count <- cart.Count - 1
    if cart.Count = 0 then cart.Delete()
    ctx.SubmitUpdates()
```

then adding the `removeFromCart` handler in `App` module:

```fsharp
let removeFromCart albumId =
    session (function
    | NoSession -> never
    | UserLoggedOn { Username = cartId } | CartIdOnly cartId ->
        let ctx = Db.getContext()
        match Db.getCart cartId albumId ctx with
        | Some cart -> 
            Db.removeFromCart cart albumId ctx
            Db.getCartsDetails cartId ctx |> View.cart |> Html.flatten |> Html.xmlToString |> OK
        | None -> 
            never)
```

and finally mapping the route to the handler in main `choose` WebPart:

```fsharp
pathScan Path.Cart.removeAlbum removeFromCart
```

A few comments to the `removeFromCart` WebPart:

- this handler should not be invoked with `NoSession`, `never` prevents from unwanted requests
- the same happens, when someone tries to invoke `removeFromCart` for `albumId` not present in his cart (Db.getCart returns `None`)
- if proper cart has been found, `Db.removeFromCart` is invoked, and
- an inline portion of HTML is returned. Note that we don't go through our `html` helper function here like before, but instead return just the a part that `script.js` will inject into the "container" div on our page with AJAX.

This almost concludes the cart feature.
One more thing before we finish this section: 
What should happen if user first adds some albums to his cart and later decides to log on?
If user logs on, we have his user name - so we can "upgrade" all his carts from GUID to the actual user's name.
Add necessary functions to the `Db` module:

```fsharp
let getCarts cartId (ctx : DbContext) : Cart list =
    query {
        for cart in ctx.``[dbo].[Carts]`` do
            where (cart.CartId = cartId)
            select cart
    } |> Seq.toList
```

```fsharp
let upgradeCarts (cartId : string, username :string) (ctx : DbContext) =
    for cart in getCarts cartId ctx do
        match getCart username cart.AlbumId ctx with
        | Some existing ->
            existing.Count <- existing.Count +  cart.Count
            cart.Delete()
        | None ->
            cart.CartId <- username
    ctx.SubmitUpdates()
```

and update the `logon` handler in `App` module:

```fsharp
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

Remarks:

- `Db.getCarts` returns just a plain list of all carts for a given `cartId`
- `Db.upgradeCarts` takes `cartId` and `username` in order to iterate through all carts (returned by `Db.getCarts`), and for each of them:
    - if there's already a cart in the database for this `username` and `albumId`, it sums up the counts and deletes the GUID cart. This can happen if logged on user adds an album to cart, then logs off and adds the same album to cart, and then back again logs on - album's count should be 2
    - if there's no cart for such `username` and `albumId`, simply "upgrade" the cart by changing the `CartId` property to `username`
- `logon` handler now recognizes `CartIdOnly` case, for which it has to invoke `Db.upgradeCarts`. In addition it wipes out the `cartId` key from session store, as from now on `username` will be used as a cart id.

Whoa, we now have the cart functionality in our Music Store! 
See the following link to browse the code: [Tag - cart](https://github.com/theimowski/SuaveMusicStore/tree/cart)