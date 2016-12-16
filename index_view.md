## Index view

With the `View.fs` file in place, let's add our first view:

<pre class="fssnip highlighted"><div lang="fsharp"><span class="k">open</span> <span class="i">Suave</span><span class="o">.</span><span class="i">Html</span>&#10;&#10;<span class="k">let</span> <span class="i">divId</span> <span onmouseout="hideTip(event, 'View.fs_fs1', 1)" onmouseover="showTip(event, 'View.fs_fs1', 1)" class="i">id</span> <span class="o">=</span> <span class="i">divAttr</span> [<span class="s">&quot;id&quot;</span>, <span onmouseout="hideTip(event, 'View.fs_fs1', 2)" onmouseover="showTip(event, 'View.fs_fs1', 2)" class="i">id</span>]&#10;<span class="k">let</span> <span class="i">h1</span> <span class="i">xml</span> <span class="o">=</span> <span class="i">tag</span> <span class="s">&quot;h1&quot;</span> [] <span class="i">xml</span>&#10;<span class="k">let</span> <span class="i">aHref</span> <span class="i">href</span> <span class="o">=</span> <span class="i">tag</span> <span class="s">&quot;a&quot;</span> [<span class="s">&quot;href&quot;</span>, <span class="i">href</span>]&#10;&#10;<span class="k">let</span> <span class="i">index</span> <span class="o">=</span> &#10;    <span class="i">html</span> [&#10;        <span class="i">head</span> [&#10;            <span class="i">title</span> <span class="s">&quot;Suave Music Store&quot;</span>&#10;        ]&#10;&#10;        <span class="i">body</span> [&#10;            <span class="i">divId</span> <span class="s">&quot;header&quot;</span> [&#10;                <span class="i">h1</span> (<span class="i">aHref</span> <span class="s">&quot;/&quot;</span> (<span class="i">text</span> <span class="s">&quot;F# Suave Music Store&quot;</span>))&#10;            ]&#10;&#10;            <span class="i">divId</span> <span class="s">&quot;footer&quot;</span> [&#10;                <span class="i">text</span> <span class="s">&quot;built with &quot;</span>&#10;                <span class="i">aHref</span> <span class="s">&quot;http://fsharp.org&quot;</span> (<span class="i">text</span> <span class="s">&quot;F#&quot;</span>)&#10;                <span class="i">text</span> <span class="s">&quot; and &quot;</span>&#10;                <span class="i">aHref</span> <span class="s">&quot;http://suave.io&quot;</span> (<span class="i">text</span> <span class="s">&quot;Suave.IO&quot;</span>)&#10;            ]&#10;        ]&#10;    ]&#10;    <span class="o">|&gt;</span> <span class="i">xmlToString</span>&#10;</div></pre>&#10;<div class="tip" id="View.fs_fs1">val id : x:&#39;T -&gt; &#39;T<br /><br />Full name: Microsoft.FSharp.Core.Operators.id</div>&#10;&#10;

This will serve as a common layout in our application.
A few remarks about the above snippet:

- open `Suave.Html` module, for functions to generate HTML markup.
- 3 helper functions come next:
    - `divId` which appends "div" element with a string attribute `id`
    - `h1` which takes inner markup to generate HTML header level 1.
    - `aHref` which takes string attribute `href` and inner HTML markup to output "a" element.
- `tag` function comes from Suave. It's of type `string -> Attribute list -> Xml -> Xml`. First arg is name of the HTML element, second - a list of attributes, and third - inner markup
- `Xml` is an internal Suave type holding object model for the HTML markup
- `index` is our representation of HTML markup.
- `html` is a function that takes a list of other tags as its argument. So do `head` and `body`.
- `text` serves outputting plain text into an HTML element.
- `xmlToString` transforms the object model into the resulting raw HTML string.

> Note: `tag` function from Suave takes 3 arguments ().
> We've defined the `aHref` function by invoking `tag` with only 2 arguments, and the compiler is perfectly happy with that - Why?
> This concept is called "partial application", and allows us to invoke a function by passing only a subset of arguments.
> When we invoke a function with only a subset of arguments, the function will return another function that will expect the rest of arguments.
> In our case this means `aHref` is of type `string -> Xml -> Xml`, so the second "hidden" argument to `aHref` is of type `Xml`.
> Read [here](http://fsharpforfunandprofit.com/posts/partial-application/) for more info about partial application.

We can see usage of the "pipe" operator `|>` in the above code.
The operator might look familiar if you have some UNIX background.
In F#, the `|>` operator basically means: take the value on the left side and apply it to the function on the right side of the operator.
In this very case it simply means: invoke the `xmlToString` function on the HTML object model.

Let's test the `index` view in our `App.fs`:

<pre class="fssnip highlighted"><div lang="fsharp">        <span class="i">path</span> <span class="s">&quot;/&quot;</span> <span class="o">&gt;</span><span class="o">=&gt;</span> (<span class="i">OK</span> <span class="i">View</span><span class="o">.</span><span class="i">index</span>)&#10;</div></pre>&#10;&#10;

If you navigate to the root url of the application, you should see that proper HTML has been returned.


---

GitHub commit: [750cac6586d20a790c0b672d0ec0eac0e8ebaba8](https://github.com/theimowski/SuaveMusicStoreTutorial/commit/750cac6586d20a790c0b672d0ec0eac0e8ebaba8)

Files changed:

* App.fs (modified)
* View.fs (modified)
