# Views

We've seen how to define basic routing in a Suave application. 
In this section we'll see how we can deal with returning good looking HTML markup in a HTTP response.
Templating HTML views is quite a big topic itself, that we don't want to go into much details about.
Keep in mind that the concept can be approached in many different ways, and the way presented here is not the only proper way of rendering HTML views.
Having said that, I hope you'll still find the following implementation concise and easy to understand.
In this application we'll use server-side HTML templating with the help of a separate Suave package called `Suave.Experimental`.

> Note: As of the time of writing, `Suave.Experimental` is a separate package. It's likely that next releases of the package will include breaking changes. It's also possible that the modules we're going to use from within the package will be extracted to the core Suave package.

To use the package, we need to take a dependency on the following NuGet:
```install-package Suave.Experimental -version 0.28.1```

Before we start defining views, let's organize our `App.fs` source file by adding following line at the beginning of the file:

```fsharp
module SuaveMusicStore.App

open Suave
```

The line means that whatever we define in the file will be placed in `SuaveMusicStore.App` module.
Read [here](http://fsharpforfunandprofit.com/posts/recipe-part3/) for more info about organizing and structuring F# code.
Now let's add a new file `View.fs` to the project just before the `App.fs` file and place the following module definition at the very top:

```fsharp
module SuaveMusicStore.View
```

We'll follow this convention throughout the tutorial to have a clear understanding of the project structure.

> Note: It's very important that the `View.fs` file comes before `App.fs`. F# compiler requires the referenced items to be defined before their usage. At first glance, that might seem like a big drawback, however after a while you start realizing that you can have much better control of your dependencies. Read the [following](http://fsharpforfunandprofit.com/posts/cyclic-dependencies/) for further benefits of lack of cyclic dependencies in F# project.
