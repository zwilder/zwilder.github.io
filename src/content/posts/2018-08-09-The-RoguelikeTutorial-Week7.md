---
title: "The Roguelike Tutorial - Week 7"
date: 2018-08-09 13:00:00 -0700
tags:
    - C++
    - Roguelike
    - Barbarian!
---

I didn't do a lot of coding over the last week, but I did finish the
challenge! Had a bit of a scare after finishing part 12, I found a
horrible lurking bug in the program that caused a LOT of little errors
and I wasn't sure (at first) how to track it down. Basically, shortly
after finishing adding in some cool randomization functions I found that
some entities weren't being copied correctly, or saved correctly. Turns
out, the [C++ Rule of
3/5/0](https://en.cppreference.com/w/cpp/language/rule_of_three) is
actually a rule and not just a suggestion/guideline. 

This project was the first time I attempted to use smart pointers and
write more "correct" C++ code, and apparently even when using smart
pointers to allocate memory you should write at least a copy constructor
and copy assignment function. I'm still not sure what I would put in a
custom destructor if all the memory that's allocated is done via smart
pointers, and that might be something I need to clean up now the
challenge is over.

When the `Entity` class was created with the default copy constructor
(or copy assignment), it wasn't allocating new memory for the new
entity. So, when one entity was destroyed another entity would suddenly
lose it's components! This never was a problem before part 12, because I
never tried to duplicate entities. One of the main goals in part 12 was
to intelligently place monsters, and I implemented that with a weighted
list that contained a prototype entity which was duplicated each time
one of those enemies was placed. All of a sudden, enemies suddenly
couldn't die after the first one was killed. Pathfinding spontaneously
broke. Entity names started being displayed as gibberish. I seriously
thought I was going to have to scrap the project at this point, because
I had no idea where to begin solving this problem. The answer eventually
came to me at work, and getting everything smoothed out was pretty
trivial.

## Part 12

So the theme of part 12 was to start implementing more randomization
functions. Up until this week I was using the built in STL version of
mersenne twister, and I don't think the way I was using it was the most
effective or how it was designed to be used. Searching the web for
answers, I stumbled upon a **ton** of information about pseudo random
number generators.  I never really though much about it, but computers
can not generate actual random numbers. The random numbers we see
generated are the result of a bunch of math that is way (way) over my
head. There is a lot of number generators out there, and some of them
seemed like they would be pretty simple to implement. So, I decided to
write a quick simple implementation of an xorshift generator:

{{< highlight cpp >}}
class RNGState
{
    public:
        // These are random numbers - Literally could use anything here. I've been just seeding the first number with the system time
        RNGState(uint32_t a = 123456789, uint32_t b = 362436069, uint32_t c = 521288629, uint32_t d = 88675123) : x(a), y(b), z(c), w(d) { }
        uint32_t x;
        uint32_t y;
        uint32_t z;
        uint32_t w;
};

uint32_t xor128(RNGState * rng)
{
    uint32_t s = rng->w;
    uint32_t t = rng->w;
    t ^= t << 11;
    t ^= t >> 8;
    rng->w = rng->z;
    rng->z = rng->y;
    rng->y = rng->x;
    s = rng->x;
    t ^= s;
    t ^= s >> 19;
    rng->x = t;
    return t;
}

int randomInt(int max, RNGState * rng)
{
    int result = xor128(rng) % max + 1;
    return result;
}

int randomInt(int min, int max, RNGState * rng)
{
    if(min > max)
    {
        int tmp = min;
        min = max;
        max = tmp;
    }
    if(min == max)
    {
        return min;
    }

    // Finding the random number this way is what allows the xorshift function to return '0'
    int range = max - min;
    int result = randomInt(range + 1, rng) - 1;
    return (min + result);
}
{{< / highlight >}}

The `xor128()` function uses some bitwise math and shifts the numbers
around, returning a 'new' 32 bit unsigned integer. The `randomInt()`
function takes a random number generator, calls the xorshift function,
does some modulo math, and returns the result. **This is not a very
computationally effective way to do this**, but for my purposes it's
actually super fast and works good enough. Plus, it's fun having some
idea of how the random numbers are generated. The cool thing about using
this generator is that the `RNGState` class is pretty trivial to
archive, and I can have the `GameMap` class and `Engine` class use their
own generators - which allows maps to be replayed!

A long existing glitch with my pathfinding code was that if no path
could be found - it would just enter an endless loop and freeze the
game. Thanks to be able to set the seed and play the same scenario over
and over, I was able to narrow down and squash that bug a lot easier
than I could have before.

The weighted list functions are pretty nifty too, I think. Basically
it's just a template wrapper around the `DLList` class I've been using
everywhere.

{{< highlight cpp >}}
template <typename T>
class WeightedList
{
    public:
        void add(T obj, int wt);
        T pick(RNGState * rng);

    private:
        DLList<T> list_;
};

template <typename T>
void WeightedList<T>::add(T obj, int wt)
{
    if(wt <=0)
    {
        return;
    }
    for(int i = 0; i < wt; ++i)
    {
        list_.push(obj);
    }
}

template <typename T>
T WeightedList<T>::pick(RNGState * rng)
{
    int index = randomInt(0, list_.size() - 1, rng);
    return list_.at(index)->data;
}
{{< / highlight >}}

Eventually, when the enemies are loaded from an external file, a
weighted list will be dynamically created with all enemies that can be
found at or below a certain dungeon level. Items will be generated the
same way. I moved the item/monster/player definitions to another source
file, to begin planning for this eventual move.

## Part 13

Part 13 was another big milestone and something I haven't ever
successfully done - add equippable items. Fortunately, all the
preparation and careful building of the framework so far made this part
incredibly easy. I added another component to `Entity`, `Equipment`.
This component has the bonuses for each actor statistic (health,
defense, power) and a bitmask containing which slot the item occupies
and if the item is equipped. This makes it easy for an item to occupy
multiple slots - like our hero's two handed greatsword! An entity with
the equipment component is created by default with the item component.
The last thing needed for this to be successful was to make sure the
`Actor` component kept track of what slots it has currently occupied by
equipment. Sometime in the future when I work on the UI, I'll allow the
player to select what slot they want to equip items to - so they can use
one handed weapons in each hand like a true barbarian. For now, small
weapons like the dagger are automatically equipped to the off hand.

{{< figure src="/assets/2018-08-09/EquippedItems.png" >}}

I also need to sort the inventory, displaying equipped items at the top.
Actually, equipped items probably won't show up in the inventory at all,
and just show up in the equipment screen of the future UI. Here's a
quick gameplay video of what I ended up with after part 13:

{{< video mp4="/assets/Various/Part13.mp4" >}}

---

So what's next? Well, I'm stoked to finally have a project make it this
far, and be solid enough that I can build off. I've got quite the list
of "to-do" items to start tackling, and lots of ideas for fun future
implementations for this project. Here's a truncated list, stripped of
general housekeeping things, in no particular order.

* Move monster/item definitions to external text files, and write a
  parser to change those to game data.

* Clean up SDL texture and sprite classes - these are pretty much
  straight from the [LazyFoo SDL
tutorials](http://lazyfoo.net/tutorials/SDL/), and I think I can spruce
them up a bit/modernize them. Maybe.

* Better dungeon generators, with external text file definitions.

* Finish A-star/Dijkstra pathfinding functions - BFS works just fine,
  but I still want to try and implement these.

* Fix combat - maybe do a fancy percentile based system or something!

* **Animations!** I’ve got a really good idea on how I want to do these,
  so it’ll probably be one of the first things I tackle next.

* Move all "flavor" text to an external file - would make it much easier
  to add things. 

* Improve the UI - I’ve got some mockups made, but I’m kinda enjoying
  the basic no frills look this has right now.

* Character creation/naming screen.

* Get the project to build on other operating systems! This is also high
  on my priorities, along with creating prebuilt binaries for people who
don’t want to fuss with compiling.

* Re-add the timing/scheduling system.

* Properly comment and organize the core classes code. My comments are
  pretty sparse, and it would probably be a good idea to do this before
the project gets any bigger. Plus, it might help explain my logic and
why I did something if anyone ever looked at my code. 

* Split game display into multiple virtual consoles (log console, game
  console, UI console, etc). It's all one console right now, which isn’t
bad but sometimes the log or UI can obscure the map.

Here is the source code for [Part 13](https://github.com/zwilder/Barbarian/tree/Part_13) and [Part 12](https://github.com/zwilder/Barbarian/tree/Part_12)!
