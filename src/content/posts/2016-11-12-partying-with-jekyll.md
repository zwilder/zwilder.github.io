---
title:  "Partying With Jekyll"
date:   2016-11-12 18:15:06 -0700
tags:
  - Jekyll
  - Web Programming
---

I've been wanting to fix this blog for a while now, and I decided in the process that I am **awful** at web design/programming. Haha, well, the first step to being kinda good at something is kinda sucking at something, right?

In my scouring of the internet I discovered that a lot of static blog sites used [Jekyll][jekyll] to generate their site. At first, I figured it was nothing more than another insta-blog program where you just dump
some content into a folder and it spits out a blog - but holy cow it is so much more than (and can be exactly) that. I've only scratched the surface of what it is capable of, since I don't have many skills in the web developement
category (yet).

I originally attempted to create my own [Jekyll][jekyll] theme - and successfully recreated my original blog format. Unfortunately, I remembered (after spending a good deal of time working on it) that I didn't really like the look/feel of my original blog - and trashed it. The next attempt was to find and use another, premade, theme. I wasn't really a fan of that route from the start, but I tried a few different ones - and tweaked them to my liking. The blog still didn't look/feel quite right, and didn't really seem to work well with posts that are largely text/code.

The last theme I tried was based on [hack.css][hackcss]. It was kinda neat looking, but still felt a little cluttered to me. After discovering what [hack.css][hackcss] originally was, I decided that it wouldn't be too difficult
to create my own "theme" based off the framework hack.css provided. The result is what you have in front of you.

Yeah, a few things still need tweaking - I'm not super happy with the way code blocks look for large sections of code... IE this looks fine:

{{< highlight cpp >}}
#include <iostream>
int main(int argc, char * argv[])
{
	std::cout << "Hello Jekyll!\n";
	return 0;
}
{{< / highlight >}}

However, looking at my previous [post]({{ site.baseurl }}/2016/07/15/fun-with-dijkstra-mapping.html)... the large chunks of code posted there look atrocious. Ah well, it's fun to tweak things and beat them into
looking like a real website.

[jekyll]: http://jekyllrb.com
[hackcss]: http://www.hackcss.com
