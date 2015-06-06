# View amends

First we'll create a left-sided navigation menu with all possible genres in our Store. 
This will allow users to find their favorite tracks faster.

Add following `partGenres` to the `View` module:

```fsharp
let partGenres (genres : Db.Genre list) =
    ulAttr ["id", "categories"] [
        for genre in genres -> 
            li (aHref 
                    (Path.Store.browse |> Path.withParam (Path.Store.browseKey, genre.Name)) 
                    (text genre.Name))
    ]
```

`partGenres` creates an unordered list with direct links to each genre.

To include it in the main index view, let's pass it as a new parameter, and render just before the `container`:

```fsharp
let index partNav partUser partGenres container = 
    html [

    ...
    
        partGenres
    
        divId "main" container
        
        ...
```

This forces us to extend the invocation in nested `result` function in the `html` WebPart:

```fsharp
let result cartItems user =
        OK (View.index 
                (View.partNav cartItems) 
                (View.partUser user) 
                (View.partGenres (Db.getGenres ctx))
                container)
        >>= Writers.setMimeType "text/html; charset=utf-8"
```