---
title: "The Roguelike Tutorial - Week 5b/6"
date: 2018-07-31 13:00:00 -0700
tags:
    - C++
    - Roguelike
    - Barbarian!
---

Oh man, this has been a super exciting and productive week - this
project is starting to feel like a real game! Started off last week a
bit behind schedule, having just finished part 8. After my rant last
week about my source code files getting too long, I decided to spend a
bit of time going back over my core classes and cleaning up the code. I
also split `Engine` into multiple files - which ended up making a lot of
things a lot easier. Heck, recompiling a huge source file each time I
make a minor change was tedious, and the time savings alone was worth
splitting the files. Each of the Engine main functions now has it’s own
file, and then each of those main functions calls separate routines
depending on what `GameState` the game is in. Each of those separate,
state specific functions has its own file. Code that’s reused between
routines lives in the main function file.

I think it’s pretty cool - makes finding and changing things easier. I
also noticed that this project is now more than twice the length (lines
of code) of any project I’ve done so far. This weeks post was originally
intended to cover just part 9, but I threw 10 and 11 in as well since
they were relatively short.

## Part 9 - Targeting
I wasn’t sure when I started this part how I would go about doing it.
Style-wise, I like the way ADOM and Angband handling targeting with the
auto select target and keyboard interface. Using a mouse feels weird to
me when playing roguelikes, so I knew I didn’t want to stick to the
ideas in the Python tutorial. The game engine is already set up to
handle different game states, but targeting isn’t really a state - more
of a miniature loop with it’s own events and draw cycle.

So, I decided to make targeting exactly that - a game state that isn’t
really a state, but a miniature replica of the main game loop. When a
function (any function) calls `Engine::target()` the game saves the
current state, and starts it’s own loop. When the loop finishes, the
game returns the state to where it was, and whatever called the
targeting state can go about doing whatever it was doing. This allows
any function in the game to have the player move and set the cursor to a
desired location, and then the function can do something at that
position. Neat!

{{< highlight cpp >}}
void Engine::target()
{
    // This function is called by other functions to have the player move the cursor to a desired location,
    // select that location with [enter], and then the other function has access to the cursorPos via Engine::cursor();
    targetSelected_ = false;
    targeting_ = true;
    std::string prevMsg = currentMsg_;
    GameState prevState = gameState_;
    advanceMsg_();

    // Target is a mini-main loop - so this limits it's speed a bit
    using namespace std::chrono;
    const milliseconds MS_PER_FRAME = std::chrono::milliseconds(16); // 16ms = ~60fps, 33ms = ~30fps

    while(targeting_)
    {
        milliseconds start = duration_cast<milliseconds>(system_clock::now().time_since_epoch());
        handleEvents(); // moving the cursor, waiting for [enter] to set targetSelected_ to true
        draw(); // Drawing the path from the player to the cursor
        std::this_thread::sleep_for(milliseconds(start + MS_PER_FRAME - duration_cast<milliseconds>(system_clock::now().time_since_epoch())));
    } 
    changeState(prevState);
    addMessage(prevMsg);
    advanceMsg_();
}
{{< / highlight >}}

The code looks a bit different in the actual game - just because I
wanted to reuse the targeting code for a look function. Here’s a short
video of the result!

{{< video mp4="/assets/2018-07-30/Part9.mp4" >}}

One of the cool things here is the code to draw the line from the
character to the cursor when in "target" mode. I spent a good deal of
time trying to figure out a way to easily draw a line... and then
realized I already wrote code to do exactly that with the FOV code! It's
always neat to use code you've already written for things you didn't
originally plan for. 

## Part 10 - Saving and loading
From the beginning of this project I planned to use
[Cereal](https://github.com/USCiLab/cereal) for serialization of the
game data. This header only library is phenomenally easy to use, and my
previous projects had great success incorporating it. The library only
requires that you provide a template for archiving your classes in each
class that will be archived, basically a set of instructions on what to
save and what to load back.

{{< highlight cpp >}}
template <class Archive>
void serialize(Archive & ar)
{
    ar(myData_);
}
{{< / highlight >}}

That’s it! Except... crap. I wrote a ton of custom containers and
classes, so I had to figure out how to tell Cereal how to handle
serializing them! `wsl::Vector2<T>` and `wsl::Rectangle<T>` were easy
enough, but what about my glorious lists? What about all the classes
using them?

Well, I couldn’t figure out a nice way of making cereal happy, so my
solution was a little hacky. I created a couple new transition classes
(basically replacing my list with a STL vector), since cereal can handle
those just fine. Whenever the game calls save game, it makes those
transition classes and saves those instead. Loading is exactly the
reverse, the game loads the transition classes and recreates the
original. Eh, it’s not pretty, but it works and works very well. The
only classes requiring a transition class are my priority queue, double
linked list, and entity class - so thats not too shabby. What’s nice is
that the core classes are already in place - adding more to the game
shouldn’t affect save/load functionality all that much!

## Part 11 - Advancement

Adding stairs was a pretty trivial matter - looking at the core classes,
stairs should be a tile and not an entity. The player does nothing with
the stairs except use them to get to the next level, so there is no
reason to make them an entity. In addition, the stairs will NEVER have
any of the components in the entity class, whoever heard of stairs with
an inventory or health? Doors, when they’re added, will be an entity
because they’ll have an interactable component (same with treasure
chests, traps, levers, etc). Making the player go down the stairs was
likewise easy, and just for fun a quick check was added to see if there
are any monsters near the player when they move down - if there are,
they follow the player. It may be fun to add a random chance the player
is unable to go down the stairs when monsters are nearby.

Leveling up and character advancement is something that will change
dramatically when the percentile combat system is in place. So, for now,
the system is identical to what’s used in the Python tutorial. 

---

With all that done, I figured I would play around in REXpaint and try
making a title screen for the game:

{{< figure src="/assets/2018-07-30/BarbarianLogo.png" >}}

Super happy with how this turned out - I’m not much of an ASCII artist,
but I think this looks darn good. One of the things high up on my
priority list is to implement an animation system (fireballs need to
explode, arrows need to fly, blood needs to splatter, etc) - how cool
would this logo look if the blood on the right dripped down into the
puddle at the bottom?!

