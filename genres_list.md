## Genres list

For more convenient instantiation of `DbContext`, let's introduce a small helper function in `Db` module:

<pre class="fssnip highlighted"><div lang="fsharp"><span class="k">let</span> <span onmouseout="hideTip(event, 'Db.fs:15-15_fs1', 1)" onmouseover="showTip(event, 'Db.fs:15-15_fs1', 1)" class="f">getContext</span>() <span class="o">=</span> <span class="i">Sql</span><span class="o">.</span><span class="i">GetDataContext</span>()&#10;</div></pre>&#10;<div class="tip" id="Db.fs:15-15_fs1">val getContext : unit -&gt; &#39;a<br /><br />Full name: CDocument.getContext</div>&#10;&#10;

Now we're ready to finally read real data in the `App` module:

<pre class="fssnip highlighted"><div lang="fsharp"><span class="k">let</span> <span onmouseout="hideTip(event, 'App.fs:18-23_fs1', 1)" onmouseover="showTip(event, 'App.fs:18-23_fs1', 1)" class="i">overview</span> <span class="o">=</span> <span class="i">warbler</span> (<span class="k">fun</span> _ <span class="k">-&gt;</span>&#10;    <span class="i">Db</span><span class="o">.</span><span class="i">getContext</span>() &#10;    <span class="o">|&gt;</span> <span class="i">Db</span><span class="o">.</span><span class="i">getGenres</span> &#10;    <span class="o">|&gt;</span> <span onmouseout="hideTip(event, 'App.fs:18-23_fs2', 2)" onmouseover="showTip(event, 'App.fs:18-23_fs2', 2)" class="i">List</span><span class="o">.</span><span onmouseout="hideTip(event, 'App.fs:18-23_fs3', 3)" onmouseover="showTip(event, 'App.fs:18-23_fs3', 3)" class="i">map</span> (<span class="k">fun</span> <span class="i">g</span> <span class="k">-&gt;</span> <span class="i">g</span><span class="o">.</span><span class="i">Name</span>) &#10;    <span class="o">|&gt;</span> <span class="i">View</span><span class="o">.</span><span class="i">store</span> &#10;    <span class="o">|&gt;</span> <span class="i">html</span>)&#10;</div></pre>&#10;<div class="tip" id="App.fs:18-23_fs1">val overview : obj<br /><br />Full name: CDocument.overview</div>&#10;<div class="tip" id="App.fs:18-23_fs2">Multiple items<br />module List<br /><br />from Microsoft.FSharp.Collections<br /><br />--------------------<br />type List&lt;&#39;T&gt; =<br />&#160;&#160;| ( [] )<br />&#160;&#160;| ( :: ) of Head: &#39;T * Tail: &#39;T list<br />&#160;&#160;interface IEnumerable<br />&#160;&#160;interface IEnumerable&lt;&#39;T&gt;<br />&#160;&#160;member GetSlice : startIndex:int option * endIndex:int option -&gt; &#39;T list<br />&#160;&#160;member Head : &#39;T<br />&#160;&#160;member IsEmpty : bool<br />&#160;&#160;member Item : index:int -&gt; &#39;T with get<br />&#160;&#160;member Length : int<br />&#160;&#160;member Tail : &#39;T list<br />&#160;&#160;static member Cons : head:&#39;T * tail:&#39;T list -&gt; &#39;T list<br />&#160;&#160;static member Empty : &#39;T list<br /><br />Full name: Microsoft.FSharp.Collections.List&lt;_&gt;</div>&#10;<div class="tip" id="App.fs:18-23_fs3">val map : mapping:(&#39;T -&gt; &#39;U) -&gt; list:&#39;T list -&gt; &#39;U list<br /><br />Full name: Microsoft.FSharp.Collections.List.map</div>&#10;&#10;

<pre class="fssnip highlighted"><div lang="fsharp">        <span class="i">path</span> <span class="i">Path</span><span class="o">.</span><span class="i">Store</span><span class="o">.</span><span class="i">overview</span> <span class="o">&gt;</span><span class="o">=&gt;</span> <span class="i">overview</span>&#10;</div></pre>&#10;&#10;

`overview` is a WebPart that...
Hold on, do I really need to explain it?
The usage of pipe operator here makes the flow rather obvious - each line defines each step.
The return value is passed from one function to another, starting with DbContext and ending with the WebPart.
This is just a single example of how composition in functional programming makes functions look like building blocks "glued" together.

We also need to wrap the `overview` WebPart in a `warbler`.
That's because our `overview` WebPart is in some sense static - there is no parameter for it that could influence the outcome.
`warbler` ensures that genres will be fetched from the database whenever a new request comes.
Otherwise, without the `warbler` in place, the genres would be fetched only at the start of the application - resulting in stale genres in case the list changes.
How about the rest of WebParts?

- `browse` is parametrized with the genre name - each request will result in a database query.
- `details` is parametrized with the id - the same as above applies.
- `home` is just fine - for the moment it's completely static and doesn't need to touch the database.


---

GitHub commit: [ddad611ccc2555d13a9f22d4340ed1fac8a0aacc](https://github.com/theimowski/SuaveMusicStoreTutorial/commit/ddad611ccc2555d13a9f22d4340ed1fac8a0aacc)

Files changed:

* App.fs (modified)
* Db.fs (modified)
