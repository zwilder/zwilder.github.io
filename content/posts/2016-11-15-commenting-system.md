---
title:  "Commenting System with Staticman!"
date:   2016-11-15 19:18:06 -0700
---

The one bad thing about hosting a static blog is that implementing comments means you:

* Outsource, letting some other site handle it (like Disqus)
* Come up with some sort of crazy contraption to force a comment system into a static blog
* Spend hours scouring the internet to see what other people have done.

Obviously, outsourcing wasn't an option - Disqus looks nice and all, and I'm sure it would be fine for a lot of people, but it just wasn't what I wanted. I like having full control of my data. My next idea was to find a free
hosting site and write some crazy PHP and set up a SQL database to set up comments.

But this got me thinking - how did sites handle comments before all this dynamic content was commonplace? A careful google search led me to an amazing blog post by [Eduardo Boucas](https://eduardoboucas.com/blog/2016/08/10/staticman.html) on how he solved this exact problem. And holy crap does his solution work exactly like I wanted. In a quick nutshell: [Staticman](https://staticman.net) is a Github bot that pushes formatted yml (or json) files into a specified directory that can than be read as comments on the page when the site is built!

So how did I set this up in Jekyll? Read on!

Obviously, the first thing I did was read the documentation on the [Staticman](https://staticman.net/docs/) site, and followed the steps listed there. This is what was added to my `_config.yml` file for the site.

{{< highlight yml >}}
staticman:
  allowedFields: ['name', 'message']
  branch: master
  filename: 'entry{@timestamp}'
  format: yml
  generatedFields:
    date:
      type: date
      options:
        format: "timestamp-seconds"
  moderation: false
  path: "_data/comments/{options.slug}"
  requiredFields: ['name', 'message']
  url: https://api.staticman.net/v1/entry/zwilder/Hack_Blog/master
{{< / highlight >}}

Now, since the blog is accessible from GitHub, and the repository is public - I removed the email field. I don't think anyone would appreciate their email being so easily accessible (something something security).
The important part of this is the path variable -  this saves comments in their own folder, named after the post they were generated from.

I added the comments into their very own partial HTML page in the `_includes\` directory. 

At the top of the `comments.html` page:
<figure class="highlight">
<pre>
<code class="language-html" data-lang="html">
&#123;&#37; capture post_slug &#37;&#125;&#123;&#123; page.date | date: "&#37;Y-&#37;m-&#37;d" &#125;&#125;-&#123;&#123; page.title | slugify &#125;&#125;&#123;&#37; endcapture &#37;&#125;
</code>
</pre>
</figure>
This is what makes the name for the directory from the post name - the [Jekyll docs](https://jekyllrb.com/docs/templates/) do a much better job explaining this than I could. Really, it just shoves the 'slugified' name into a variable called 'post_slug'.

The forms were also not very complex - basic HTML forms, but the important bit is the `name="fields[]"` part. In the `_config.yml` file, notice that the fields declared are the same here. 
<figure class="highlight"><pre><code class="language-html" data-lang="html"><span class="nt">&lt;form</span> <span class="na">class=</span><span class="s">"form"</span> <span class="na">action=</span><span class="s">"&#123;&#123; site.staticman.url &#125;&#125;"</span> <span class="na">method=</span><span class="s">"POST"</span><span class="nt">&gt;</span>
    <span class="nt">&lt;input</span> <span class="na">type=</span><span class="s">"hidden"</span> <span class="na">name=</span><span class="s">"options[slug]"</span> <span class="na">value=</span><span class="s">"&#123;&#123; post_slug &#125;&#125;"</span><span class="nt">&gt;</span>
    <span class="nt">&lt;fieldset</span> <span class="na">class=</span><span class="s">"form-group form-success"</span><span class="nt">&gt;</span>
      <span class="nt">&lt;label</span> <span class="na">for=</span><span class="s">"username"</span><span class="nt">&gt;</span>USERNAME:<span class="nt">&lt;/label&gt;</span>
      <span class="nt">&lt;input</span> <span class="na">id=</span><span class="s">"username"</span> <span class="na">type=</span><span class="s">"text"</span> <span class="na">name=</span><span class="s">"fields[name]"</span> <span class="na">placeholder=</span><span class="s">"Anonymous"</span> <span class="na">class=</span><span class="s">"form-control"</span> <span class="na">maxlength=</span><span class="s">"15"</span><span class="nt">&gt;</span>
    <span class="nt">&lt;/fieldset&gt;</span>
    <span class="nt">&lt;fieldset</span> <span class="na">class=</span><span class="s">"form-group form-textarea form-success"</span><span class="nt">&gt;</span>
      <span class="nt">&lt;label</span> <span class="na">for=</span><span class="s">"message"</span><span class="nt">&gt;</span>MESSAGE:<span class="nt">&lt;/label&gt;</span>
      <span class="nt">&lt;textarea</span> <span class="na">id=</span><span class="s">"message"</span> <span class="na">rows=</span><span class="s">"5"</span> <span class="na">name=</span><span class="s">"fields[message]"</span> <span class="na">class=</span><span class="s">"form-control"</span><span class="nt">&gt;&lt;/textarea&gt;</span>
    <span class="nt">&lt;/fieldset&gt;</span>
    <span class="nt">&lt;div</span> <span class="na">class=</span><span class="s">"form-actions form-success"</span><span class="nt">&gt;</span>
      <span class="nt">&lt;button</span> <span class="na">type=</span><span class="s">"submit"</span> <span class="na">class=</span><span class="s">"btn btn-success btn-block"</span> <span class="na">value=</span><span class="s">"Send"</span><span class="nt">&gt;</span>Submit<span class="nt">&lt;/button&gt;</span>
    <span class="nt">&lt;/div&gt;</span>
<span class="nt">&lt;/form&gt;</span></code></pre></figure>

Finally, for the fun part: Reading the comments into the file.
<figure class="highlight"><pre><code class="language-html" data-lang="html">        
&#123;&#37; if site.data.comments[post_slug] &#37;&#125;
    &#123;&#37; assign comments = site.data.comments[post_slug] | sort &#37;&#125; 
    &#123;&#37; for comment in comments &#37;&#125;
        &#123;&#37; assign name = comment[1].name &#37;&#125;
        &#123;&#37; assign date = comment[1].date &#37;&#125;
        &#123;&#37; assign message = comment[1].message &#37;&#125;
        <span class="nt">&lt;div</span> <span class="na">class=</span><span class="s">"cell -3of12"</span><span class="nt">&gt;</span>
            <span class="nt">&lt;span</span> <span class="na">class=</span><span class="s">"pull-right"</span><span class="nt">&gt;</span>&#123;&#123; name &#125;&#125;@<span class="nt">&lt;span</span> <span class="na">style=</span><span class="s">"color: #4caf50"</span><span class="nt">&gt;</span>WSL_Guest<span class="nt">&lt;/span&gt;</span>%<span class="nt">&lt;/span&gt;&lt;br/&gt;</span>
        <span class="nt">&lt;/div&gt;</span>
         <span class="nt">&lt;div</span> <span class="na">class=</span><span class="s">"cell -9of12"</span><span class="nt">&gt;</span>
            <span class="nt">&lt;span</span> <span class="na">class=</span><span class="s">"pull-left"</span> <span class="na">style=</span><span class="s">"color: #FFF; margin-left: 5px; "</span><span class="nt">&gt;</span>[<span class="nt">&lt;span</span> <span class="na">style=</span><span class="s">"color: #4caf50"</span><span class="nt">&gt;</span>&#123;&#123; date | date: "&#37;d&#37;b&#37;y : &#37;T" &#125;&#125;<span class="nt">&lt;/span&gt;</span>] &#123;&#123; message &#125;&#125;<span class="nt">&lt;/span&gt;&lt;br</span> <span class="nt">/&gt;&lt;br</span> <span class="nt">/&gt;</span>
        <span class="nt">&lt;/div&gt;</span>
    &#123;&#37; endfor &#37;&#125;
&#123;&#37; endif &#37;&#125;
</code></pre></figure>

Broken down, the code basically checks to see if there is a folder in the comments directory (the path specified in `_config.yml`) - if there is, the comments are sorted and then looped through.
For each comment, a variable is assigned for the name, date and message, and then with some fancy styling and HTML it is put into the post! Thats it!

I honestly didn't believe it would be that easy - most of what took me time was formatting the forms and laying out the comments section in a pleasing (and hopefully easy to read!) manner. I took some inspiration from my terminal window as I was working - I think it actually looks pretty cool for comments. [Staticman](https://staticman.net) is the coolest thing, and I'm gonna have to go check out the source and maybe play around with it.

Now, for some fun obstacles - typing liquid code into a markdown page for Jekyll to process as highlighted code... yeah that doesn't work. Fortunately, you can still use HTML, but then the post becomes a bit
less human readable in it's raw form. To get the HTML code to highlight I let Jekyll build the site with the liquid code processed, opened up the generated HTML file, copy-pasta'd it back into the markdown file, and changed the liquid code back to what it should be before Jekyll processed it. Kinda a pain, but definitely easier than doing all the highlighting by hand!
