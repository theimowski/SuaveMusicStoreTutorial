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

"As a user I want to see that my cart is empty when my cart is empty so that I can make my cart not empty"
With such a serious business requirement, we'd better distinguish case when user has anything in his cart from case when the cart is empty.
To do that, add separate `emptyCart` in `View` module: 

```fsharp
let emptyCart = [
    h2 "Your cart is empty"
    text "Find some great music in our "
    aHref Path.home (text "store")
    text "!"
]
```

In the latter case, we're going to display a table with all albums in the cart:

```fsharp
let aHrefAttr href attr = tag "a" (("href", href) :: attr)

...

let nonEmptyCart (carts : Db.CartDetails list) = [
    h2 "Review your cart:"
    table [
        yield tr [
            for h in ["Album Name"; "Price (each)"; "Quantity"; ""] ->
            th [text h]
        ]
        for cart in carts ->
            tr [
                td [
                    aHref (sprintf Path.Store.details cart.AlbumId) (text cart.AlbumTitle)
                ]
                td [
                    text (formatDec cart.Price)
                ]
                td [
                    text (cart.Count.ToString())
                ]
                td [
                    aHrefAttr "#" ["class", "removeFromCart"; "data-id", cart.AlbumId.ToString()] (text "Remove from cart") 
                ]
            ]
        yield tr [
            for d in ["Total"; ""; ""; carts |> List.sumBy (fun c -> c.Price * (decimal c.Count)) |> formatDec] ->
            td [text d]
        ]
    ]
]
```

With these two separate views we can declare a more general one, for when we're not sure whether the cart is empty or not:

```fsharp
let cart = function
    | [] -> emptyCart
    | list -> nonEmptyCart list
```

`cart` makes use of the short `function` pattern matching syntax. `[]` case holds only for empty lists, so the second case is valid for non-empty lists.

A few remarks regarding the `nonEmptyCart` function:

- first comes the column headings row ("Name, Price, Quantity")
- then for each `CartDetail` from the list there is a row containing:
    - link to album details with album title caption
    - single album price
    - count of this very album in cart
    - link to remove the item from cart (we'll soon apply AJAX updates to this one)
- finally there's a summary row that displays the total price for albums in the cart

We can't really test the view for now, as we haven't yet implemented fetching cart items from database. 
We can however see how the `emptyCart` view looks like if we add proper handler in `App` module:

```fsharp
let cart = View.cart [] |> html

...

path Path.Cart.overview >>= cart
```

A navigation menu item can also appear handy (`View.partNav`):

```fsharp
li (aHref Path.Cart.overview (text "Cart"))
```

It's time to revisit the `Session` type from `App` module.
Business requirement is that in order to checkout, a user must be logged on but he doesn't have to when adding albums to the cart.
That seems reasonable, as we don't want to stop him from his shopping spree with boring logon forms.
For this purpose we'll add another case `CartIdOnly` to the `Session` type.
This state will be valid for users who are not yet logged on to the store, but have already some albums in their cart:

```fsharp
type Session = 
    | NoSession
    | CartIdOnly of string
    | UserLoggedOn of UserLoggedOnSession
```

`CartIdOnly` contains string with a GUID generated upon adding first item to the cart.

Switching back to `Db` module, let's create a type alias for `Carts` table:

```fsharp
type Cart = DbContext.``[dbo].[Carts]Entity``
```

`Cart` has following properties:

- CartId - a GUID if user is not logged on, otherwise username
- AlbumId
- Count

To implement `App` handler, we need the following `Db` module functions:

```fsharp
let getCart cartId albumId (ctx : DbContext) : Cart option =
    query {
        for cart in ctx.``[dbo].[Carts]`` do
            where (cart.CartId = cartId && cart.AlbumId = albumId)
            select cart
    } |> firstOrNone
```

```fsharp
let addToCart cartId albumId (ctx : DbContext)  =
    match getCart cartId albumId ctx with
    | Some cart ->
        cart.Count <- cart.Count + 1
    | None ->
        ctx.``[dbo].[Carts]``.Create(albumId, cartId, 1, DateTime.UtcNow) |> ignore
    ctx.SubmitUpdates()
```

`addToCart` takes `cartId` and `albumId`. 
If there's already such cart entry in the database, we do increment the `Count` column, otherwise we create a new row.
To check if a cart entry exists in database, we use `getCart` - it does a standard lookup on the cartId and albumId.

Now open up the `View` module and find the `details` function to append a new button "Add to cart", at the very bottom of "album-details" div:

```fsharp
yield pAttr ["class", "button"] [
    aHref (sprintf Path.Cart.addAlbum album.AlbumId) (text "Add to cart")
]
```

With above in place, we're ready to define the handler in `App` module:

```fsharp
let addToCart albumId =
    let ctx = Db.getContext()
    session (function
            | NoSession -> 
                let cartId = Guid.NewGuid().ToString("N")
                Db.addToCart cartId albumId ctx
                sessionStore (fun store ->
                    store.set "cartid" cartId)
            | UserLoggedOn { Username = cartId } | CartIdOnly cartId ->
                Db.addToCart cartId albumId ctx
                succeed)
        >>= Redirection.FOUND Path.Cart.overview
```

```fsharp
pathScan Path.Cart.addAlbum addToCart
```

`addToCart` invokes our `session` function with two flavors:

- if `NoSession` then create a new GUID, save the record in database and update the session store with "cartid" key
- if `UserLoggedOn` or `CartIdOnly` then only add the album to the user's cart. Note that we could bind the `cartId` string value here to both cases - as described earlier `cartId` equals GUID if user is not logged on, otherwise it's the user's name.

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