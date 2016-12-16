## Query parameters

Apart from passing arguments in the route itself, we can use the query part of url:
`localhost:8083/store/browse?genre=Disco`
To do this, let's create a separate WebPart:

<pre class="fssnip highlighted"><div lang="fsharp"><span class="k">let</span> <span onmouseout="hideTip(event, 'App.fs:7-11_fs1', 1)" onmouseover="showTip(event, 'App.fs:7-11_fs1', 1)" class="i">browse</span> <span class="o">=</span>&#10;    <span class="i">request</span> (<span class="k">fun</span> <span class="i">r</span> <span class="k">-&gt;</span>&#10;        <span class="k">match</span> <span class="i">r</span><span class="o">.</span><span class="i">queryParam</span> <span class="s">&quot;genre&quot;</span> <span class="k">with</span>&#10;        | <span onmouseout="hideTip(event, 'App.fs:7-11_fs2', 2)" onmouseover="showTip(event, 'App.fs:7-11_fs2', 2)" class="i">Choice1Of2</span> <span class="i">genre</span> <span class="k">-&gt;</span> <span class="i">OK</span> (<span onmouseout="hideTip(event, 'App.fs:7-11_fs3', 3)" onmouseover="showTip(event, 'App.fs:7-11_fs3', 3)" class="i">sprintf</span> <span class="s">&quot;Genre: %s&quot;</span> <span class="i">genre</span>)&#10;        | <span onmouseout="hideTip(event, 'App.fs:7-11_fs4', 4)" onmouseover="showTip(event, 'App.fs:7-11_fs4', 4)" class="i">Choice2Of2</span> <span class="i">msg</span> <span class="k">-&gt;</span> <span class="i">BAD_REQUEST</span> <span class="i">msg</span>)&#10;</div></pre>&#10;<div class="tip" id="App.fs:7-11_fs1">val browse : obj<br /><br />Full name: CDocument.browse</div>&#10;<div class="tip" id="App.fs:7-11_fs2">union case Choice.Choice1Of2: &#39;T1 -&gt; Choice&lt;&#39;T1,&#39;T2&gt;</div>&#10;<div class="tip" id="App.fs:7-11_fs3">val sprintf : format:Printf.StringFormat&lt;&#39;T&gt; -&gt; &#39;T<br /><br />Full name: Microsoft.FSharp.Core.ExtraTopLevelOperators.sprintf</div>&#10;<div class="tip" id="App.fs:7-11_fs4">union case Choice.Choice2Of2: &#39;T2 -&gt; Choice&lt;&#39;T1,&#39;T2&gt;</div>&#10;&#10;

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

<pre class="fssnip highlighted"><div lang="fsharp"><span class="k">let</span> <span onmouseout="hideTip(event, 'App.fs:13-19_fs1', 1)" onmouseover="showTip(event, 'App.fs:13-19_fs1', 1)" class="i">webPart</span> <span class="o">=</span> &#10;    <span class="i">choose</span> [&#10;        <span class="i">path</span> <span class="s">&quot;/&quot;</span> <span class="o">&gt;</span><span class="o">=&gt;</span> (<span class="i">OK</span> <span class="s">&quot;Home&quot;</span>)&#10;        <span class="i">path</span> <span class="s">&quot;/store&quot;</span> <span class="o">&gt;</span><span class="o">=&gt;</span> (<span class="i">OK</span> <span class="s">&quot;Store&quot;</span>)&#10;        <span class="i">path</span> <span class="s">&quot;/store/browse&quot;</span> <span class="o">&gt;</span><span class="o">=&gt;</span> <span class="i">browse</span>&#10;        <span class="i">pathScan</span> <span class="s">&quot;/store/details/%d&quot;</span> (<span class="k">fun</span> <span onmouseout="hideTip(event, 'App.fs:13-19_fs2', 2)" onmouseover="showTip(event, 'App.fs:13-19_fs2', 2)" class="i">id</span> <span class="k">-&gt;</span> <span class="i">OK</span> (<span onmouseout="hideTip(event, 'App.fs:13-19_fs3', 3)" onmouseover="showTip(event, 'App.fs:13-19_fs3', 3)" class="i">sprintf</span> <span class="s">&quot;Details: %d&quot;</span> <span onmouseout="hideTip(event, 'App.fs:13-19_fs2', 4)" onmouseover="showTip(event, 'App.fs:13-19_fs2', 4)" class="i">id</span>))&#10;    ]&#10;</div></pre>&#10;<div class="tip" id="App.fs:13-19_fs1">val webPart : obj<br /><br />Full name: CDocument.webPart</div>&#10;<div class="tip" id="App.fs:13-19_fs2">val id : x:&#39;T -&gt; &#39;T<br /><br />Full name: Microsoft.FSharp.Core.Operators.id</div>&#10;<div class="tip" id="App.fs:13-19_fs3">val sprintf : format:Printf.StringFormat&lt;&#39;T&gt; -&gt; &#39;T<br /><br />Full name: Microsoft.FSharp.Core.ExtraTopLevelOperators.sprintf</div>&#10;&#10;


---

GitHub commit: [9b393b0a961c888ebc310b9fbf706ac666224f23](https://github.com/theimowski/SuaveMusicStoreTutorial/commit/9b393b0a961c888ebc310b9fbf706ac666224f23)

Files changed:

* App.fs (modified)
