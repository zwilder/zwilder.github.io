---
title:  "Randomly Generated Caves with Cellular Automata"
date:   2016-12-22 19:18:06 -0700
---

So, every time I get the urge to write a blog post about whatever random bit of code I'm writing - I look at the blog and think: "Hm, this is kinda ugly." Then, I have to rewrite it until I'm mostly satisfied with
how it looks.

Recently, I've been working on writing a nice little interface for handling menus and other parts of the UI for my roguelike project. Unfortunately, like the layout of this blog, I can't seem to find a satisfactory
'look and feel' - I think I've got a good start though.

While procrastinating working on that, I've spent a lot of time thinking about the map, and my ECS system. At the risk of prematurely optimizing code, it seems like an awful waste of space to make every tile an entity. Most of a 
tiles properties are simple boolean flags anyways, so why not make tiles just a bitflag and glyph to represent them? After running some halfassed tests on this idea, the size of an entire map (100x100, or 10,000 tiles) worth of tiles was a mere 24 bytes. Yeah, this idea will work just fine.

Basically, how it works now is a tile is just:
{{< highlight cpp >}}
struct Tile 
{
    int bitMask;
    char glyph;
}
{{< / highlight >}}

And the flags:
{{< highlight cpp >}}
enum TileFlags : int
{
    NONE = 0,
    BLOCKS_MOVEMENT = 0x001,
    BLOCKS_LIGHT = 0x002,
    UP_STAIR = 0x004,
    DOWN_STAIR = 0x008,
    DOOR = 0x010,
    FLOOR = 0x020
};
{{< / highlight >}}

I'm a pretty huge fan of bitflags, mostly because it allows me to do all sorts of cool stuff. Say for instance we added a `LIQUID` flag to `TileFlags` - liquid blocks movement (our fearless adventurer never took swim lessons, apparently) but doesn't block light, so the adventurer's field of vision would see over the murky pool of whatever liquid.

But wait! It gets better, what if our adventurer decides that, while standing next to a murky pool of water, to read a Scroll of Explosion - The game logic can now easily test to see if any of the affected tiles are also liquid, and make steam that the player CAN'T see through! Or, even ~~better~~ worse, maybe it's a pool of acid... Acid steam clouds sound pretty deadly...

And.... this got me thinking about dungeon generating. The dungeon generator I have been using was some [code I pulled off RogueBasin](http://www.roguebasin.com/index.php?title=C%2B%2B_Example_of_Dungeon-Building_Algorithm) and modified to fit my uses. The modifications I made kinda ruined the modularity of
the generator, mostly because of the tile specifications in the generator. Also, the generator was making a dungeon and then the game was taking the dungeon that it made and making the tiles from the tiles the dungeon generated. If that sounds confusing - it was. Definitely not clean and simple, so I decided I needed to fix that. 

But of course, I couldnt just hack up the main generator for dungeon fun for the game - without testing the idea first on something completely different. I decided to rewrite something I wrote for the Libtcod Python Roguelike tutorial - Caves with Cellular Automata.

The whole idea is based on [Conway's Game of Life](https://en.wikipedia.org/wiki/Conway's_Game_of_Life) - just modified to make some cool, natural looking caves. The algorithm goes something like this:

* A map is created filled with floors and then each cell (x,y coordinate) is looped through, with a 45% chance of becoming a wall.
* A blank strip of floor tiles, 3 tiles wide is created going from left to right across the map and another from top to bottom. This step helps insure continuity of the caves, and reduces the chances of inaccessible spaces.
* A border around the map is added, 1 tile wide. The first time the border is added is just so that the caves can 'grow' from it.
* Next we loop through each tile twice, counting the number of wall tiles adjacent to it. If there are 5 or more wall tiles *or less than two wall tiles* in the 3x3 grid surrounding the tile, the tile becomes a wall tile. Otherwise, it becomes a floor tile. The less than two part of this run adds a little bit of random spots in the map.
* The final run through loops through the tiles ten times. If there are 5 or more wall tiles in the 3x3 grid surrounding the tile, the tile becomes a wall tile. Otherwise, it becomes a floor tile. These loops smooth out the rough edges, and make the caves look nice and organic.
* A border is readded to the map - don't want our adventurer wandering off screen!
* The up-stairs are randomly placed on a floor tile.
* A dijkstra map is created, with the upstairs as the goal, and the downstairs are placed on the last tile with the highest value (farthest distance) from the upstairs.

This generates some pretty neat looking caves:

<pre style="color: #f8f8f2; text-align: center;">
<code>

################################################################################
#.............####.............................................................#
#.............#####.......................................................##...#
##............######...............................####....<span style="color: #AB266D;"><</span>.............####..#
###............#####.............#####............######.................####..#
####............###.............#########.........######..................##..##
#####..........................###########........######......................##
######.........................#############.......####.......................##
#######........................##############................##..............###
#######........................################.............####.............###
########.......................###################........######..............##
########.......................####################......########.............##
########.....###................####################.....########......##....###
#########...#####...............####################.....#########....##########
#################................##################......##########..###########
##################................################......########################
###################.........................###.........########################
####################.........##.........................########################
#####################.......####.........................#######################
#########################..######................##.......######################
##################################..............####.......#####################
###################################............######.......####################
####################################..........########.......###################
#########<span style="color: #AB266D;">></span>.##########################.........#########...........##########..##
########....#########################..........#########...........######......#
########.....###############..######............########............####.......#
#########.........#######..........................####.......................##
##########...........###.....................................................###
##########...................................................................###
################################################################################

</code>
</pre>

This cave has some nice large open spaces for combat, and it's not claustrophobic enough to make exploration a chore! The idea, like a lot of my ideas, was inspired by a [post on cellular automata on RogueBasin](http://www.roguebasin.com/index.php?title=Cellular_Automata_Method_for_Generating_Random_Cave-Like_Levels).

The most current version of all my RL files is on [Github](https://github.com/zwilder/Weasel_Engine).
