---
title: "Explosions in the Dungeon"
date: 2018-09-18 14:00:00 -0700
tags:
    - C++
    - Roguelike
    - Barbarian!
---

One of the big goals for Barbarian and one of the coolest things on my
to do list was to add animations into the game. How exciting is it to
see an arrow flying toward your character, a firebolt explode in the
face of an undead horde, blood splatter flying from the wounds inflicted
by the mighty barbarian decorating the walls of the dungeon?

Obviously, the excitement added by all these visual effects is well
worth the time spent adding them in. I started thinking about how I
wanted to incorporate animations during the Tutorial Tuesday Challenge,
and started jotting implementation ideas down in my notebook almost two
months ago. For the life of me I could not find a good writeup on
creating procedural explosions on a 2d grid, and could not wrap my head
around how to make them look decent. I shared my struggles on the
[r/roguelikedev sharing Saturday post, and another dev (u/darkgnostic)
gave me some fantastic ideas about reusing my line-of-sight
code](https://www.reddit.com/r/roguelikedev/comments/9e007b/sharing_saturday_223/e5m8y55/).
So, armed with this knowledge I sat down last week and decided to
finally try some of my implementation ideas.

The idea was simple, for each frame expand outward from the origin of
the explosion to a set radius.

## The Framework

In order to test this out, I needed to actually get the animations
working. Like most bits of code that I only have a vague idea of how to
accomplish I started at the top: `Engine::update()` and `Engine::draw()`
should loop through a list of animations, updating and drawing them.
Easy enough to draw them, just need an `Animation::draw()` routine.
Update would be similar, but would need to know how much time had passed
since the last update. To get this, I needed to visit the main loop code
- something that hasn't been looked at in quite a while.

Cracked open `main.cpp`, dusted it off and examined my loop. Previously
the game was limited to 60fps by using a call to sleep the thread -
definitely gross and in need of some work. Where had I seen a beautiful,
simple game loop before? Ah - [Game Programming
Patterns](http://gameprogrammingpatterns.com/game-loop.html)! Probably
my favorite, most well worn book on my shelf of programming books. Easy
enough to modify the example given in that chapter to modern C++ using
the `std::chrono` library.

{{< highlight cpp >}}
using namespace std::chrono;
milliseconds previous = duration_cast<milliseconds>(system_clock::now().time_since_epoch());
milliseconds lag = std::chrono::milliseconds(0);
while(engine->running())
{
    milliseconds current = duration_cast<milliseconds>(system_clock::now().time_since_epoch());
    milliseconds elapsed = current - previous;
    previous = current;
    lag += elapsed;

    engine->handleEvents();
    engine->update(int(elapsed.count()));
    engine->draw();
    while(lag >= MS_PER_FRAME)
    {
        // Note that in the Game Programming Patterns example,
        // update is placed here, and draw is placed after this
        // loop. This forces draw to update less frequently since
        // it is technially the most expensive function in terms
        // of processing power. Not a problem in a turn based,
        // non-graphics intensive game like this.
        //
        // Instead, this loop just holds the framerate constant at
        // 60fps (MS_PER_FRAME = 17)
        lag -= MS_PER_FRAME;
    }
}
{{< / highlight >}}

Once all that was straightened out I knew that I needed a draw and
update function in the animation class. Also, from my doodles in my
notebook I knew that the way I wanted to approach animations was to make
each animation a collection of frames, and each frame a collection of
tiles consisting of a glyph (character, foreground/background color) and
a position. Also, I wanted to make sure that the animations would not
block input or advancement of the games update loop. 

Now, for some reason these collections just would not work with my fancy
list class. I'm not sure why, and spent an insane amount of time trying
to track down what was causing horrible segfaults. I then switched over
to using a `std::vector` and realized it was because I was trying to
access parts of the list out of bounds - I literally forgot an equals
sign in a single `currentFrame > frames.size()` conditional. With
everything working I didn't feel like changing code back to using my
list class - if it isn't broken why fix it?

## Making an Explosion
I started the explosion with the basic idea given to me: expand outwards
from an origin.

{{<highlight txt>}}
// as a circle with r=2
Frame   0   1    2

                 2
            1   212
         0 101 21012
            1   212
                 2

// as a square
Frame   0   1    2

               22222
           111 21112
         0 101 21012
           111 21112
               22222

{{< / highlight>}}

Each frame is simply one expansion outward. Setting the glyph of each
animation tile to a simple red `^` made a pleasing simple looking
explosion and reminded me of the explosions in the old great roguelikes.
[Cogmind](https://www.gridsagegames.com/cogmind/) has the most beautiful
animations and tasty explosions I have seen in just about any game, and
the developer has a [fantastic blog
post](https://www.gridsagegames.com/blog/2014/04/making-particles/)
about the explosions.  Playing Cogmind and seeing these explosions in
action inspired most of my desire to get animations working, and my goal
is to try and get something similar in place for Barbarian.

Watching my simple explosion moving outwards I realized that a simple
way to make the explosion procedural was to just add in a random chance
that the cell would get "lit up" by the blast. I decided to set the
chance at 75% initially each time a cell is visited to light it up, and
change just the background color of the cell to randomly
orange/yellow/red. This worked, and looked **awesome**. 

{{< video mp4="/assets/Various/Explosions.mp4" >}}

I realized after playing around with making things explode that there
should be a set of frames moving inward towards the origin as well, but
with a smaller chance of lighting up. Playing around with the numbers I
found 65%/25% to look pretty darn good. Shortened the duration by a bit
too to speed things up.

{{< video mp4="/assets/Various/Explosions3.mp4" >}}

I also played with changing the glyph in the explosion tile to a random
character, which ended up looking sorta cool... but I'm not sure I like
it. It definitely made the explosion more chunky, and gave it some
texture.

{{< video mp4="/assets/Various/Explosions2.mp4" >}}

The "final" code for the explosion animation looks like this:

{{< highlight cpp >}}
Animation explosion(wsl::Vector2i origin, int radius)
{
    uint32_t seed = uint32_t(std::chrono::duration_cast< std::chrono::milliseconds >(std::chrono::system_clock::now().time_since_epoch()).count());
    wsl::RNGState rng(123987445, seed); // Random numbers
    const int ANIMATION_DURATION = 250;
    Animation result;
    int frameDuration = ANIMATION_DURATION / (radius * 2); // Outward explosion/Inward Explosion
    //Outward explosion
    for(int i = 1; i <= radius; ++i)
    {
        AnimationFrame frame;
        for(int x = origin.x - i; x <= origin.x + i; ++x)
        {
            for(int y = origin.y - i; y <= origin.y + i; ++y)
            {
                // Without adding in an if sqrt(x^2 + y^2) <= r this just makes a square instead of a circle
                // if(sqrt(pow(x, 2) + pow(y, 2)) <= radius) continue;
                if(wsl::randomInt(1,100,&rng) >= 65)
                {
                    continue;
                }
                wsl::Color bg = wsl::Color::Red;
                switch(wsl::randomInt(1,3,&rng))
                {
                    case 1: bg = wsl::Color::Yellow; break;
                    case 2: bg = wsl::Color::Orange; break;
                    case 3: bg = wsl::Color::Red; break;
                    default: break;
                }
                frame.tiles.push_back(AnimationTile(wsl::Glyph('.',wsl::Color::Yellow,bg), wsl::Vector2i(x,y)));
            }
        }
        frame.duration = frameDuration;
        result.frames.push_back(frame);
    }
    //Inward explosion
    for(int i = radius; i >= 1; --i)
    {
        AnimationFrame frame;
        for(int x = origin.x - i; x <= origin.x + i; ++x)
        {
            for(int y = origin.y - i; y <= origin.y + i; ++y)
            {
                if(wsl::randomInt(1,100,&rng) >= 25)
                {
                    continue;
                }
                wsl::Color bg = wsl::Color::Red;
                switch(wsl::randomInt(1,3,&rng))
                {
                    case 1: bg = wsl::Color::Yellow; break;
                    case 2: bg = wsl::Color::Orange; break;
                    case 3: bg = wsl::Color::Red; break;
                    default: break;
                }
                frame.tiles.push_back(AnimationTile(wsl::Glyph('.',wsl::Color::Yellow,bg), wsl::Vector2i(x,y)));
            }
        }
        frame.duration = frameDuration;
        result.frames.push_back(frame);
    }
    result.engage(Animation::Flags::APPLY_BG | Animation::Flags::APPLY_BG | Animation::Flags::APPLY_GLYPH);
    return result;
}
{{< / highlight >}}

Now, if you look closely at the above videos you may notice the
projectile launching from the player to the origin of the explosion -
those frames had to be added before the others (obviously) into the same
animation. If it was just another animation added to the list it would
be launching and exploding at the same time - not ideal.

To accomplish this I have a couple of functions, one for a projectile
animation, one for a firebolt projectile, one for the explosion, and one
for the fireball. The normal projectile animation creates a projectile
that travels from point A to B with a specialized glyph. This can now be
used for weapon throwing, arrow shooting, fireball launching, etc. The
firebolt projectile code starts off by creating a projectile, but then
modifies the glyph in each frame. I want to add some smoke trails to
this as well eventually, and this gives me a good spot to put that code.
The explosion animation is described above, and the fireball code
combines the explosion and firebolt projectile animations into one.
Neat!

---

With the way animations are made and represented in the game, I should
be able to make animations by hand with just as much ease as the
procedural animations... which means the next big animation is going to
have to be the main menu!
