---
title: "The Roguelike Tutorial - Week 3a"
date: 2018-07-02T08:00:00-07:00
tags:
  - C++
  - Roguelike
  - Barbarian!
---

I decided to split the third week into two separate posts so that I wouldn't ramble on as long as last week. Over the past week I continued fighting the BSP dungeon
generator I had attempted to write the week prior, and gave up again. I'll probably revisit the dungeon generator later, and will definitely write something about
whatever I end up doing. The artificial deadlines imposed by following along with the rest of [r/roguelikedev](https://reddit.com/r/roguelikedev/) really helps
motivate me to call something "good enough" and move on... Heck, I think I could be satisfied just tinkering with dungeon generators and never moving on to actually
making this a playable game!

I did some [reading on writing clean C++ by google](https://sites.google.com/a/chromium.org/dev/developers/coding-style/cpp-dos-and-donts), which prompted me to take
 another look at the code I've already written. The big thing I gleaned from the Google article was to avoid "in-lining" things in headers, and put most of the
code in the implementation. This is something I **thought** I did already, but apparently using non-trivial types inline in headers is bad for performance. This means
stuff like this is OK:

{{< highlight cpp >}}
class Foo
{
    public:
        ...
        const int & count() { return count_; }
        ...
    private:
        int count_;
};
{{< / highlight >}}

However, doing this is not:
{{< highlight cpp >}}
class Bar
{
    public:
        ...
        Foo badFunc() { return data_; }
        ...
    private:
        Foo data_;
};

{{< / highlight >}}

Along the same lines, Google recommends not to inline constructors/destructors with non-trivial types. This is something I did in almost every header file in the
game! Fortunately, since this program is still relatively small, the changes to my code were easy enough to implement.

So the first challenge for this week was to implement a "field of view" for our dungeon delving barbarian. Two options I considered for this were following along with
the original Rogue or using Bresenham's line drawing algorithm. The original Rogue showed the entirety of whichever room the player was in, and kept track of explored
areas. Field of view using line drawing basically illuminates a radius around the player, while intelligently stopping sight from passing through walls and other
obstructions. I decided to go with the second option for [part 4](http://rogueliketutorials.com/libtcod/4).

## Bresenham's Line Drawing
The [original algorithm](https://en.wikipedia.org/wiki/Bresenham%27s_line_algorithm) was written in 1962, as a punch card program, to draw lines. Since the game
space is a Cartesian grid, we can "draw" lines using this algorithm from the player to represent how far the player can see. The end point of the line is the player's
Field of View radius. The only modification to the original algorithm is that we need to terminate the line drawing early if an object is in the way and blocks sight.
My adaptation is straightforward, and probably a bit verbose - I think that any extremely minor speed possibly lost is well worth it for the readability. 

The line drawing function takes a pointer to a vector of coordinates to store the results, and a pointer to the map to check if the tiles block light. The original 
map and results vector are owned by `Engine` and stored there. Also, I apologize for the abstract variable names - `xO,yO` are the x,y coordinates of the origin,
`xF,yF` are the final coordinates, `i,j` are the test coordinates for each point.

{{< highlight cpp >}}
void bhline(int xO, int yO, int xF, int yF, std::vector<wsl::Vector2i> * results, GameMap * map)
{
    int dX = abs(xF - xO);
    int dY = abs(yF - yO);

    if(dX >= dY)
    {
        int e = dY - dX; // [e]rror
        int j = yO;

        if(xO < xF)
        {
            // Octants 1,2
            for(int i = xO; i <= xF; ++i)
            {
                add(results, wsl::Vector2i(i,j));
                if(map->tiles[map->index(i,j)].blocksLight())
                {
                    break;
                }
                if((e >= 0) && (yF >= yO))
                {
                    // 1
                    j += 1;
                    e -= dX;
                }
                else if((e >= 0) && (yF < yO))
                {
                    // 2
                    j -= 1;
                    e -= dX;
                }
                e += dY;
            }
        }
        else if(xO > xF)
        {
            // Octants 5,6
            for(int i = xO; i >= xF; --i)
            {
                add(results, wsl::Vector2i(i, j));
                if(map->tiles[map->index(i,j)].blocksLight())
                {
                    break;
                }
                if((e >= 0) && (yF >= yO))
                {
                    // 6
                    j += 1;
                    e -= dX;
                }
                else if((e >= 0) && (yF < yO))
                {
                    // 5
                    j -= 1;
                    e -= dX;
                }
                e += dY;
            }
        }
    }
    else if (dX < dY)
    {
        int e = dX - dY; // [e]rror
        int i = xO;
        if(yO < yF)
        {
            // Octants 7,8
            for(int j = yO; j <= yF; ++j)
            {
                add(results, wsl::Vector2i(i,j));
                if(map->tiles[map->index(i,j)].blocksLight())
                {
                    break;
                }
                if((e >= 0) && (xF >= xO))
                {
                    // 8
                    i += 1;
                    e -= dY;
                }
                else if((e >= 0) && (xF < xO))
                {
                    // 7
                    i -= 1;
                    e -= dY;
                }
                e += dX;
            }
        }
        else if(yO > yF)
        {
            // Octants 3,4
            for(int j = yO; j >= yF; --j)
            {
                add(results, wsl::Vector2i(i,j));
                if(map->tiles[map->index(i,j)].blocksLight())
                {
                    break;
                }
                if((e >= 0) && (xF >= xO))
                {
                    // 3
                    i += 1;
                    e -= dY;
                }
                else if((e >= 0) && (xF < xO))
                {
                    // 4
                    i -= 1;
                    e -= dY;
                }
                e += dX;
            }
        }
    }
}
{{< / highlight >}}

One of the things that tripped me up in this adaptation was if `dX == dY` what happens? Well, originally the program just ignored it, which left
some interesting diagonal gaps in the corners of our hero's vision. Not really ideal when danger could be lurking around any corner! I'm not entirely sure that
I fixed this, but setting `dX >= dY` in the first conditional appeared to fix it without making anything implode. The other pitfall was figuring out where to
terminate the loop when the line encounters a tile that blocks light. Breaking the loop before adding the coordinates to the results causes walls to NEVER be visible.
Which, could work as a style choice if that's what you're going for, I guess. Breaking the loop as the very last thing after incrementing the error appeared fine at
first, but then the loop terminates too early in some cases, resulting in weird gaps (such as in corners of rooms). Through trial and error and brute force it seems
that the correct answer is to terminate right after adding the tile to the results. 

After all that - here's a couple of videos!

The first video highlights the visible cells a nice light yellow color, and then shows the explored cells.
{{< video mp4="/assets/2018-07-02/FoV_highlight.mp4" >}}

The next video shows the results without doing anything fancy, just showing explored tiles. I think this actually looks and reads better than highlighting the 
"visible" cells.
{{< video mp4="/assets/2018-07-02/FoV.mp4" >}}

---

The tutorial dungeon actually ends up being pretty nice to explore, and looks a bit less chaotic when not everything is visible. I'll probably continue tinkering with
the results of the FoV algorithm, and possibly come up with some sort of 'fog' to differentiate what is visible to the player and what isn't. As is, the player might
see enemies 'randomly appear' in large rooms when the vision range of the player doesn't extend all the way to the walls. Speaking of enemies, this dungeon is getting
a little lonely! Time to start working on some adversaries...

---

