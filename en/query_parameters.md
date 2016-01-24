# Query parameters

Apart from passing arguments in the route itself, we can use the query part of url:
`localhost:8083/store/browse?genre=Disco`
To do this, let's create a separate WebPart:

```fsharp
let browse =
    request (fun r ->
        match r.queryParam "genre" with
        | Choice1Of2 genre -> OK (sprintf "Genre: %s" genre)
        | Choice2Of2 msg -> BAD_REQUEST msg)
```

`request` is a function that takes as parameter a function of type `HttpRequest -> WebPart`.
A function which takes as an argument another function is often called "Higher order function".
`r` in our lambda represents the `HttpRequest`. It has a `queryParam` member function of type 
`string -> Choice<string,string>`. `Choice` is a type that represents a choice between two types.
Usually you'll find that the first type of `Choice` is for happy paths, while second means something went wrong.
In our case the first string stands for a value of the query parameter, and the second string stands for error message (parameter with given key was not found in query).
We can make use of pattern matching to distinguish between two possible choices.
Pattern matching is yet another really powerful feature, implemented in variety of modern programming languages. 
For now we can think of it as a switch statement with binding value to an identifier in one go.
In addition to that, F# compiler will issue an warning in case we don't provide all possible cases (`Choice1Of2 x` and `Choice2Of2 x` here).
There's actually much more for pattern matching than that, as we'll discover later.
`BAD_REQUEST` is a function from Suave library (you may need to open `Suave.RequestErrors`), and it returns WebPart with 400 Bad Request status code response with given message in its body.
We can summarize the `browse` WebPart as following:
If there is a "genre" parameter in the url query, return 200 OK with the value of the "genre", otherwise return 400 Bad Request with error message.

Now we can compose the `browse` WebPart with routing WebPart like this:

```fsharp
path "/store/browse" >=> browse
```

Eventually we should end up with something similar to: [Tag - basic_routing](https://github.com/theimowski/SuaveMusicStore/tree/basic_routing)
