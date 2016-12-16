# Basic routing

It's time to extend our WebPart to support multiple routes.
First, let's extract the WebPart and bind it to an identifier.
We can do that by typing:
`let webPart = OK "Hello World"`
and using the `webPart` identifier in our call to function `startWebServer`:
`startWebServer defaultConfig webPart`.
In C#, one would call it "assign webPart to a variable", but in functional world there's really no concept of a variable. Instead, we can "bind" a value to an identifier, which we can reuse later.
Value, once bound, can't be mutated during runtime.
Now, let's restrict our WebPart, so that the "Hello World" response is sent only at the root path of our application (`localhost:8083/` but not `localhost:8083/anything`):
`let webPart = path "/" >=> OK "Hello World"`
`path` function is defined in `Suave.Filters` module, thus we need to open it at the beggining of `App.fs`. `Suave.Operators` and `Suave.Successful` modules will also be crucial - let's open them as well.

<pre class="fssnip highlighted"><div lang="fsharp"><span class="k">open</span> <span class="i">Suave</span><span class="o">.</span><span class="i">Filters</span>&#10;<span class="k">open</span> <span class="i">Suave</span><span class="o">.</span><span onmouseout="hideTip(event, 'App.fs:1-4_fs1', 1)" onmouseover="showTip(event, 'App.fs:1-4_fs1', 1)" class="i">Operators</span>&#10;<span class="k">open</span> <span class="i">Suave</span><span class="o">.</span><span class="i">Successful</span>&#10;</div></pre>&#10;<div class="tip" id="App.fs:1-4_fs1">module Operators<br /><br />from Microsoft.FSharp.Core</div>&#10;&#10;

`path` is a function of type:
`string -> WebPart`
It means that if we give it a string it will return WebPart.
Under the hood, the function looks at the incoming request and returns `Some` if the path matches, and `None` otherwise.
The `>=>` operator comes also from Suave library. It composes two WebParts into one by first evaluating the WebPart on the left, and applying the WebPart on the right only if the first one returned `Some`.

Let's move on to configuring a few routes in our application.
To achieve that, we can use the `choose` function, which takes a list of WebParts, and chooses the first one that applies (returns `Some`), or if none WebPart applies, then choose will also return `None`:

<pre class="fssnip highlighted"><div lang="fsharp"><span class="k">let</span> <span onmouseout="hideTip(event, 'App.fs:6-14_fs1', 1)" onmouseover="showTip(event, 'App.fs:6-14_fs1', 1)" class="i">webPart</span> <span class="o">=</span> &#10;    <span class="i">choose</span> [&#10;        <span class="i">path</span> <span class="s">&quot;/&quot;</span> <span class="o">&gt;</span><span class="o">=&gt;</span> (<span class="i">OK</span> <span class="s">&quot;Home&quot;</span>)&#10;        <span class="i">path</span> <span class="s">&quot;/store&quot;</span> <span class="o">&gt;</span><span class="o">=&gt;</span> (<span class="i">OK</span> <span class="s">&quot;Store&quot;</span>)&#10;        <span class="i">path</span> <span class="s">&quot;/store/browse&quot;</span> <span class="o">&gt;</span><span class="o">=&gt;</span> (<span class="i">OK</span> <span class="s">&quot;Store&quot;</span>)&#10;        <span class="i">path</span> <span class="s">&quot;/store/details&quot;</span> <span class="o">&gt;</span><span class="o">=&gt;</span> (<span class="i">OK</span> <span class="s">&quot;Details&quot;</span>)&#10;    ]&#10;&#10;<span class="i">startWebServer</span> <span class="i">defaultConfig</span> <span onmouseout="hideTip(event, 'App.fs:6-14_fs1', 2)" onmouseover="showTip(event, 'App.fs:6-14_fs1', 2)" class="i">webPart</span>&#10;</div></pre>&#10;<div class="tip" id="App.fs:6-14_fs1">val webPart : obj<br /><br />Full name: CDocument.webPart</div>&#10;&#10;


---

GitHub commit: [4105c21e3f1e1aa432de98e0388f71f89b693234](https://github.com/theimowski/SuaveMusicStoreTutorial/commit/4105c21e3f1e1aa432de98e0388f71f89b693234)

Files changed:

* App.fs (modified)
