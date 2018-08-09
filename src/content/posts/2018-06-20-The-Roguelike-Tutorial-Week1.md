---
title:  "The Roguelike Tutorial - Week 1"
date:   2018-06-20 07:00:00 -0700
tags:
  - C++
  - Roguelike
  - Barbarian!
---

Last year r/roguelikedev did a really cool thing with a weekly post where everyone followed along with a tutorial, shared ideas and problems, and motivated each other
to actually finish a project. Well [they're doing it again](https://old.reddit.com/r/roguelikedev/comments/8s5x5n/roguelikedev_does_the_complete_roguelike_tutorial/)
this year, and with the announcement last week I decided to take a look at the new and improved, [Python3 version of the tutorial](rogueliketutorials.com/libtcod1).

For those not familiar with this famous tutorial - each part covers a small chunk of building a roguelike using simple language and easy to follow code. In just 13
parts you have a working roguelike to build off of and branch out from. A really, really cool idea. I did the old Python2 version a couple years back, and it blew
my mind. After completing the last part I finally had made a game, and was pretty proud of the results. I didn't know Python when I started, and hadn't done any
coding before beginning, but was able to follow along and even add my own stuff along the way. Eventually, my lack of knowledge caught up with me and my bloated
single file Python program started to become difficult to modify and maintain. I took a step back, and decided to start relearning C and then ventured into C++.

I quickly found that I actually really enjoy more of the building of the game than the designing of the game. Which, is cool because the amount of things to build
and the resources available for building things from scratch are darn near infinite - especially with the knowledge of the internet at my fingertips. It is not, however,
conducive to finishing a project or making something playable.

As I read through the new and improved tutorial, I realized something pretty amazing: I could build this in C++. Reading each part I could think of ways (plural!)
that I could accomplish the same thing, but with my own code. So, I decided to play along with the tutorial, and force myself to follow the deadlines - a week for
each part.

I decided to start posting this to the blog **after** I had finished Parts 0 - 3, which covers weeks one and two - so I've written a brief summary
of what I've done in parts 1 and 2 of the tutorial so far.

## Part 1
So I started this project using SFML - which is a fantastic and easy to use library, with tons of features. The goal of Part 0 was to get a console screen up,
and an '@' moving around. I thought about the idea of using SFML sprites directly on a simulated grid - but decided that although this was the easiest way 
to accomplish the '@' moving around, it would be beneficial to try and emulate a console much like libtcod does.

A console (defined in [console.hpp](https://github.com/zwilder/Barbarian/blob/master/include/console.hpp)) is a square grid of defined width and height, where each x,y coordinate pair is the location of a glyph. A glyph is an 
integer representing a character and a foreground/background color. Color (for now) is just a simple class (defined in [color.hpp](https://github.com/zwilder/Barbarian/blob/master/include/color.hpp) with an integer representing
the r,g,b and alpha of a color.

Okay, with the console out of the way, it was time to make the 'engine' (defined in [engine.hpp](https://github.com/zwilder/Barbarian/blob/master/include/engine.hpp)). So engine does three things primarily - handle input, update the
game, and draw it to the window ([the game loop](http://gameprogrammingpatterns.com/game-loop.html)).

Handling input is easy... oh wait, the tutorial does some crazy magic with dictionaries. So, basically the idea of the dictionary is that it has a command 
and a value associated with that command. Sounds to me like a job for some bit flags, which would allow me to easily test if a command has a flag turned 
on and then check a value with it (The Action class - defined in [input_handlers.hpp](https://github.com/zwilder/Barbarian/blob/master/include/input_handlers.hpp)). Now, when checking keyboard input, I can just have a function 
return an Action, test the flags, and do whatever with the result. If needed, this allows multiple commands to be returned with one action - which is kinda cool.

Update! Well, update doesn't do anything yet. A good chunk of the game logic will eventually go here.

Draw is where some cool stuff happens. First, I have an array of rectangles, where each rectangle is a position on the font image of a character. What's
neat here is that the array contains 256 elements - the number of glyphs represented by a CP437 font image. This means that if an entity has a symbol represented
by an '@' symbol, I can reference the position on the font image by passing '@' as the index of the array! So the draw routine basically draws everything in the
game to the console, and then reads the console and translates that to the window using the window size and sprite size to position glyphs. 

With all that done, there is now an '@' symbol moving around the screen.

## Part 2
The big things in part are creating the Entity class and the GameMap class. The entity class is pretty simple so far - an entity has a glyph and a position. Nice and
easy. The GameMap is where things get fun - so I spent a bit more time on it. The tutorial uses simple booleans to check for tile values, again I decided that this
would be a good fit for bit flags - instead of checking if something blocksMovement and blocksLight and isDoor (or whatever) I could just do something simple like

{{< highlight cpp >}}
int flag = blocksMovement | blocksLight | isDoor;
if(tile->checkFlag(flag))
{
    // Do something cool here!
}
...
bool checkFlag(int flag)
{
    return ((tileMask_ & flag) == flag);
}
{{< / highlight >}}

GameMap (defined in [game_map.hpp](https://github.com/zwilder/Barbarian/blob/master/include/game_map.hpp)) is a grid of Tiles (defined in [tile.hpp](https://github.com/zwilder/Barbarian/blob/master/include/tile.hpp)). Easy enough. For part 2, the GameMap class generates a nice hard-coded wall for testing
Tiles and player movement. 

---

Next week I'll write up a bit about my struggles with the dungeon generator - the algorithm from the tutorial worked great, but of course this is the first 
of many opportunities to branch out from the tutorial and do your own thing. I'll also touch on why I ditched SFML for SDL, and some of the fun stuff I wrote
for that.

I've nicknamed this project 'Barbarian!', since the player character in this type of roguelike does very little that is like a rogue and a lot that is like a 
loin-cloth wearing, great sword wielding burly barbarian. I even have some fun ideas for flavoring the project with more brute force and shiny muscles!

Here is the source code for [Parts 0-2](https://github.com/zwilder/Barbarian/tree/Part_0-2)
