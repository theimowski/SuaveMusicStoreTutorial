## CSS

It's high time we added some CSS styles to our HTML markup.
We'll not deep-dive into the details about the styles itself, as this is not a tutorial on Web Design.
The stylesheet can be downloaded [from here](https://raw.githubusercontent.com/theimowski/SuaveMusicStore/master/Site.css) in its final shape.
Add the `Site.css` stylesheet to the project, and don't forget to set the `Copy To Output Directory` property to `Copy If Newer`.

In order to include the stylesheet in our HTML markup, let's add the following to our `View`:

<pre class="fssnip highlighted"><div lang="fsharp"><span class="k">let</span> <span onmouseout="hideTip(event, 'View.fs:8-15_fs1', 1)" onmouseover="showTip(event, 'View.fs:8-15_fs1', 1)" class="f">cssLink</span> <span onmouseout="hideTip(event, 'View.fs:8-15_fs2', 2)" onmouseover="showTip(event, 'View.fs:8-15_fs2', 2)" class="i">href</span> <span class="o">=</span> <span class="i">linkAttr</span> [ <span class="s">&quot;href&quot;</span>, <span onmouseout="hideTip(event, 'View.fs:8-15_fs2', 3)" onmouseover="showTip(event, 'View.fs:8-15_fs2', 3)" class="i">href</span>; <span class="s">&quot; rel&quot;</span>, <span class="s">&quot;stylesheet&quot;</span>; <span class="s">&quot; type&quot;</span>, <span class="s">&quot;text/css&quot;</span> ]&#10;&#10;<span class="k">let</span> <span onmouseout="hideTip(event, 'View.fs:8-15_fs3', 4)" onmouseover="showTip(event, 'View.fs:8-15_fs3', 4)" class="i">index</span> <span class="o">=</span> &#10;    <span class="i">html</span> [&#10;        <span class="i">head</span> [&#10;            <span class="i">title</span> <span class="s">&quot;Suave Music Store&quot;</span>&#10;            <span onmouseout="hideTip(event, 'View.fs:8-15_fs1', 5)" onmouseover="showTip(event, 'View.fs:8-15_fs1', 5)" class="i">cssLink</span> <span class="s">&quot;/Site.css&quot;</span>&#10;        ]&#10;</div></pre>&#10;<div class="tip" id="View.fs:8-15_fs1">val cssLink : href:&#39;a -&gt; &#39;b<br /><br />Full name: CDocument.cssLink</div>&#10;<div class="tip" id="View.fs:8-15_fs2">val href : &#39;a</div>&#10;<div class="tip" id="View.fs:8-15_fs3">val index : obj<br /><br />Full name: CDocument.index</div>&#10;&#10;

This enables us to output the link HTML element with `href` attribute pointing to the CSS stylesheet.

There's two more things before we can see the styles applied on our site.

A browser, when asked to include a CSS file, sends back a request to the server with the given url.
If we have a look at our main `WebPart` we'll notice that there's really no handler capable of serving this file.
That's why we need to add another alternative to our `choose` `WebPart`:

```fsharp
    pathRegex "(.*)\.css" >=> Files.browseHome
```

The `pathRegex` `WebPart` returns `Some` if an incoming request concerns path that matches the regular expression pattern.
If that's the case, the `Files.browseHome` WebPart will be applied.
`(.*)\.css` pattern matches every file with `.css` extension.
`Files.browseHome` is a `WebPart` from Suave that serves static files from the root application directory.

The CSS depends on `logo.png` asset, which can be downloaded from [here](https://raw.githubusercontent.com/theimowski/SuaveMusicStore/master/logo.png).

Add `logo.png` to the project, and again don't forget to select `Copy If Newer` for `Copy To Output Directory` property for the asset.
Again, when the browser wants to render an image asset, it needs to GET it from the server, so we need to extend our regular expression to allow browsing of `.png` files as well:

<pre class="fssnip highlighted"><div lang="fsharp">        <span class="i">pathRegex</span> <span class="s">&quot;(.*)\.(css|png)&quot;</span> <span class="o">&gt;</span><span class="o">=&gt;</span> <span class="i">Files</span><span class="o">.</span><span class="i">browseHome</span>&#10;</div></pre>&#10;&#10;

Now you should be able to see the styles applied to our HTML markup.


---

GitHub commit: [d40731ce9a0b0d7328de480815aa90fdf2d77a88](https://github.com/theimowski/SuaveMusicStoreTutorial/commit/d40731ce9a0b0d7328de480815aa90fdf2d77a88)

Files changed:

* App.fs (modified)
* Site.css (added)
* SuaveMusicStore.fsproj (modified)
* View.fs (modified)
* logo.png (added)
