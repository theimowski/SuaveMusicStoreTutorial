# Navigation partial

As a warm-up, let's add navigation menu at the very top of the view.
We'll call it `partNav` and keep in separate function:

```fsharp
let partNav = 
    ulAttr ["id", "navlist"] [ 
        li (aHref Path.home (text "Home"))
        li (aHref Path.Store.overview (text "Store"))
        li (aHref Path.Admin.manage (text "Admin"))
    ]
```

`partNav` consists of 3 main tabs: "Home", "Store" and "Admin". `ulAttr` can be defined like following:

```fsharp
let ulAttr attr xml = tag "ul" attr (flatten xml)
```

We want to specify the `id` attribute here so that our CSS can make the menu nice and shiny.
Add the `partNav` to main index view, in the "header" `div`:

```fsharp
divId "header" [
    h1 (aHref Path.home (text "F# Suave Music Store"))
    partNav
]
```

This gives a possibility to navigate through main features of our Music Store.