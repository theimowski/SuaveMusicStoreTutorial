# Cart view

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