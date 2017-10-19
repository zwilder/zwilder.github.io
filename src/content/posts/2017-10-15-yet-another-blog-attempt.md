---
title: "Yet Another Blog Attempt"
date: 2017-10-16T20:55:23-07:00
tags:
  - C++
  - Web Programming
  - Roguelike
  - Hugo
---

Man, I really don't do a lot of work on this blog - maybe because I am **not** a web programmer but I can see when this blog does not look very
good. 

Well, I officially gave up on web design. Yep! Turns out, I should probably leave that to the *professionals*. Oh, and I also decided to heck with 
Jekyll. It was cool, and worked well - but searching for a premade Jekyll theme was a chore. Not to mention getting all the silly ruby nonsense to work.
Even tried playing around with Ruby and seeing what all the fuss was about. No thanks.

Stumbled upon Hu(go) - another static blog generator. Well, Hu(go) is blazingly fast, and has a ton of cool features that you don't have to even
try hard to get to work. Not to mention the themes are easily searched through, easily switched, easily customized/expanded, and look darn good.
So, for now, this is what the blog will look like - A theme called [Hemmingway](https://github.com/tanksuzuki/hemingway) powered by [Hugo](https://gohugo.io).
I also caved and signed up for Disqus, to handle comments. It really is so much easier using premade tools, haha.

--

In other news, I continued the process of learning C++ by programming in C++ and made a bit of progress on my rougelike attempt. I had a revelation 
while working one day about how the "engine" should be structured: Everything manages its own shit and nothing else. No exceptions. But, you ask, how
will my keypresses to move the player work? Wouldn't the game engine have to interpret those key presses and then tell the game state to move the player,
and then the player respond with...

No. Doing that causes all sorts of crazy dependencies and connections that do not have to exist.

My solution?

* Engine manages a stack of states. It doesn't really know what a state is, just that it has them, and that it can add states to itself and remove states from
itself. The engine is responsible for interacting with the computer and reading user input.

* Each state manages a stack of windows. Again, the state doesn't know or care what a window contains, but is responsible for drawing the window in the correct
spot on the screen and adding/removing windows as needed.

* Each window contains a grid of cells that can contain a glyph (character, with a foreground/background color). Windows are responsible for interpreting user
input. A game window might contain a world (that has locations which has actors which have actions).

Brilliant! Extraordinary! Very minimal multitasking! But wait - from the description above each component has to interact with the others. How that is accomplished
is another fun thing: messages.

Imagine a mailcart. Like, a literal cart that has a bunch of mail. Now, each component recieves the mailcart, looks for letters addressed to it, and then passes
it on to the next component. The `update` routine now does exactly that.

So the major difference is that instead of each component directly influencing another, they now interact only through messages. Now, the engine could care less
what pressing `>` does, it just generates a message saying "Hey, this key was pressed" and sticks it in the mailcart. The state doesn't care about that message,
so whatever, sticks it back in the mailcart. The active window pulls that out of the mailcart and decides that pressing that means the player needs to move down
a flight of stairs. The window generates a message saying "Hey player, move downstairs" and puts that message in the cart. When the cart gets to the player,
the player checks to see if it is standing on stairs and if so moves down them.

 Previously this would have been handled by something like

{{< highlight cpp >}}
states_[activeState]->window_[activeWindow]->world_->locations[world_->currentLocationID]->player->moveDownstairs();
{{< / highlight >}}

Yuck. Lets replace that with:

{{< highlight cpp >}}
Engine::HandleEvents()
{
...
    mailbox_->push_back(Message::toState(Command::Keypress, '>');
...
}

GameWindow::ProcessMessage()
{
...
    mailbox_->push_back(Message::toPlayer(Command::MoveDownStairs));
...
}
{{< / highlight >}}

Much, much more readable and less prone to accidentally breaking. So each component during the update routine recieves the mail cart, looks through each message,
processes each one that is addressed to it, and passes the mail cart to the next component.

A cool side effect of doing things this way is that you don't have to worry about the timing of events at every step! Each component determines the order and
priority of each message. To hell with micromanaging! 

The best part of all this was it made working with the UI so much simpler. An inventory window could read a keypress as a selection of a certain item, the game window could read the same keypress as an action! Opening and closing windows becomes a simple manner of sending a message to the State! Its simple and quick - especially
because the only game entity that will interpret messages is the player. Everything else can generate messages, but has no need for intrepreting them. 

So, I made yet another little demo to test all this out, and it worked. Flawlessly. Now, I'm beginning work rewriting the engine to use SFML (again) instead of SDL.
I learned a lot from SDL, but also feel like I have no idea what I'm doing and that there has to be a more efficient way. Conviently, since everything is decoupled
so nicely this isn't really a big deal to switch frameworks. I've also been polishing and cleaning up some of the code, and thinking about the next project: managing
locations and travel? AI? Better dungeons?
