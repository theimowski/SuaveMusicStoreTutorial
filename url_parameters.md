## Url parameters

In addition to that static string path, we can specify route arguments.
Suave comes with a cool feature called "typed routes", which gives you statically typed control over arguments for your route. As an example, let's see how we can add `id` of an album to the details route:

<pre class="fssnip highlighted"><div lang="fsharp">        <span class="i">pathScan</span> <span class="s">&quot;/store/details/%d&quot;</span> (<span class="k">fun</span> <span onmouseout="hideTip(event, 'App.fs:11-11_fs1', 1)" onmouseover="showTip(event, 'App.fs:11-11_fs1', 1)" class="i">id</span> <span class="k">-&gt;</span> <span class="i">OK</span> (<span onmouseout="hideTip(event, 'App.fs:11-11_fs2', 2)" onmouseover="showTip(event, 'App.fs:11-11_fs2', 2)" class="i">sprintf</span> <span class="s">&quot;Details: %d&quot;</span> <span onmouseout="hideTip(event, 'App.fs:11-11_fs1', 3)" onmouseover="showTip(event, 'App.fs:11-11_fs1', 3)" class="i">id</span>))&#10;</div></pre>&#10;<div class="tip" id="App.fs:11-11_fs1">val id : x:&#39;T -&gt; &#39;T<br /><br />Full name: Microsoft.FSharp.Core.Operators.id</div>&#10;<div class="tip" id="App.fs:11-11_fs2">val sprintf : format:Printf.StringFormat&lt;&#39;T&gt; -&gt; &#39;T<br /><br />Full name: Microsoft.FSharp.Core.ExtraTopLevelOperators.sprintf</div>&#10;&#10;

This might look familiar to print formatting from C++, but it's more powerful.
What happens here is that the compiler checks the type for the `%d` argument and complains if you pass it a value which is not an integer.
The WebPart will apply for requests like `http://localhost:8083/store/details/28`
In the above example, there are a few important aspects:
- `sprintf "Details: %d" id` is statically typed string formatting function, expecting the id as an integer
- `(fun id -> OK ...)` is an anonymous function or lambda expression if you like, of type `int -> WebPart`
- the lambda expression is passed as the second parameter to `pathScan` function
- first argument of `pathScan` function also works as a statically typed format
- type inference mechanism built into F# glues everything together, so that you do not have to mark any type signatures

To clear things up, here is another example of how one could use typed routes in Suave:

<pre class="fssnip highlighted"><div lang="fsharp">        <span class="i">pathScan</span> <span class="s">&quot;/store/details/%s/%d&quot;</span> (<span class="k">fun</span> (<span class="i">a</span>, <span onmouseout="hideTip(event, 'App.fs:12-12_fs1', 1)" onmouseover="showTip(event, 'App.fs:12-12_fs1', 1)" class="i">id</span>) <span class="k">-&gt;</span> <span class="i">OK</span> (<span onmouseout="hideTip(event, 'App.fs:12-12_fs2', 2)" onmouseover="showTip(event, 'App.fs:12-12_fs2', 2)" class="i">sprintf</span> <span class="s">&quot;Artist: %s; Id: %d&quot;</span> <span class="i">a</span> <span onmouseout="hideTip(event, 'App.fs:12-12_fs1', 3)" onmouseover="showTip(event, 'App.fs:12-12_fs1', 3)" class="i">id</span>))&#10;</div></pre>&#10;<div class="tip" id="App.fs:12-12_fs1">val id : x:&#39;T -&gt; &#39;T<br /><br />Full name: Microsoft.FSharp.Core.Operators.id</div>&#10;<div class="tip" id="App.fs:12-12_fs2">val sprintf : format:Printf.StringFormat&lt;&#39;T&gt; -&gt; &#39;T<br /><br />Full name: Microsoft.FSharp.Core.ExtraTopLevelOperators.sprintf</div>&#10;&#10;

for request `http://localhost:8083/store/details/abba/1`

For more information on working with strings in a statically typed way, visit [this site](http://fsharpforfunandprofit.com/posts/printf/)


---

GitHub commit: [0b15ed76a8922d54934d3a75cbcaaa2b4749752a](https://github.com/theimowski/SuaveMusicStoreTutorial/commit/0b15ed76a8922d54934d3a75cbcaaa2b4749752a)

Files changed:

* App.fs (modified)
