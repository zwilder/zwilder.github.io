---
title: "The Roguelike Tutorial - Week 5a"
date: 2018-07-24 16:00:00 -0700
tags:
    - C++
    - Makefile
    - Roguelike
    - Barbarian!
---

Another very productive week for this project! Although I was unable to
finish last weeks goals of getting Parts 8 and 9 completed by today, I
still feel pretty darn good about what I did accomplish. Besides, Parts
10, 11, and 12 are all fairly simple - heck the code is already in
place, just needs to actually be used!

After finishing Part 7 last week I decided to try and fix/change a few
things that have been bothering me. The square, 12x12 font, looked great
for the map tiles but it was **really** ugly for text. I figured my
alternatives were to either use two fonts (like a 6x12 for text and a
12x12 for map), or just use a different font altogether. All my years
playing games in a terminal window pushed me to the second option - if
it’s good enough for the best roguelikes, it’s good enough for me! The
only difficulty with this option is that if I ever want to add graphical
tiles easily, I’d have to make the tiles in a strange size (ie 9x14
instead of 16x16). Adding graphical tiles in will be a chore anyways, so
I’ll just go ahead and not worry about it now. This is what the game
looks like now:

{{< figure src="/assets/2018-07-24/NewFont.png" >}}

Pretty slick! After changing to a nice 9x14 font, I decided to tackle
another minor improvement that makes a huge impact: fullscreen mode!
First, I had to set the game window to a standard resolution (800x600),
and then center the console in that window. Easy enough. Conveniently
SDL has a super cool built in function to turn on fullscreen! Oddly, it
doesn’t work at all. Maybe a fault with my code, maybe a fault with SDL.
Not sure, but google showed me that a bunch of people have the same
issue - and very few people had work-arounds. The solution I chose to go
with is to make the window a bit larger than the computer desktop
resolution, and then center it. So, it’s very much a fake fullscreen,
but it works and looks darn nice. Here’s my version of this solution:

{{< highlight cpp >}}
if(fullscreen_)
{
            windowWidth_ = 800;
            windowHeight_ = 600;
            SDL_SetWindowSize(window_, windowWidth_, windowHeight_);
            SDL_SetWindowPosition(window_, SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED);
            fullscreen_ = false;
        }
        else
        {
            SDL_DisplayMode dm;
            SDL_GetDesktopDisplayMode(0, &dm);
            windowWidth_ = dm.w;
            windowHeight_ = dm.h;
            SDL_SetWindowSize(window_, windowWidth_ + 10, windowHeight_ + 10);
            SDL_SetWindowPosition(window_, SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED);
            fullscreen_ = true;
        }
{{< / highlight >}}

I might have to revisit this when it comes time to get this running on
MacOS and Windows, I’m not sure how those systems will respond to this
hacky sort of fix! The cool thing is I think I can scale all the sprites
when drawn in `Engine::draw()`, and then resize the window with this
sort of code... another addition to the To-Do list! (I should also add
the To-Do list to the repository)

Another thing that has always bothered me is my makefile - it really
wasn’t as efficient as it could be. One thing that always bothered me
with the way it was written is that if I changed a header file, it would
not rebuild the source files that included that header. Make is
apparently capable of tracking these things, and GCC has options for
creating a dependency file for each source file, so I tinkered around
with the makefile and ended up with this:

{{< highlight make >}}
CXX = g++
# OPTS = -std=c++17 -O2
OPTS = -std=c++17 -Og -g -Wall

LIBS = -lSDL2 -lSDL2_image
OBJ_NAME = Barbarian
FILES := color texture sprite random console input_handlers entity tile game_map pathfinding fov engine main
OBJS := $(FILES:=.o)
SRC := $(OBJS:.o=.cpp)
DEP := $(FILES:=.d)

VPATH = ./src:./include

.PHONY: clean

all: $(OBJ_NAME)

$(OBJ_NAME): $(OBJS)
	$(CXX) $(OPTS) -o $(OBJ_NAME) $(OBJS) $(LIBS)

$(OBJS): %.o: %.cpp %.d
	$(CXX) $(OPTS) -c $< -o $@

%.d: %.cpp
	@set -e; rm -f $@; \
	$(CXX) -M $(OPTS) $< > $@.$$$$; \
	sed 's,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.$$$$ > $@; \
	rm -f $@.$$$$

-include $(DEP)

clean:
	rm *.o *.d
{{< / highlight >}}

Basically, a new rule to create dependency `%.d` files from the source
files was added. All of that magic is straight from the [GNU Make
manual](https://www.gnu.org/software/make/manual/html_node/Automatic-Prerequisites.html#Automatic-Prerequisites).
Then, the source files depend on the dependency files - so if any header
listed in the dependency file has changed, the object file for that
source is rebuilt. Make is pretty much magic, it will be fun figuring
out how to modify it to build for different systems!

## Part 8

Okay! With all that ~~super important work~~ procrastination done (no
wonder I didn’t finish Part 9), I began work on part 8. This part is all
about items and the inventory screen - possibly one of the most
important parts of any roguelike game is items and item management.
Additionally, this is yet another hurdle I’ve never effectively made it
past. Previous attempts at defining items and inventory were messy and
ineffective, especially when trying to allow monster’s to pick up and
use items.

First, I had to define what an item is in the context of my code. The
core classes besides the engine so far are `GameMap`, `Tile` and
`Entity`. An item can’t be a tile, since that would be weird. It makes
sense to make an item a type of entity, and that was what I wanted to do
in the first place - following ECS ideas and defining an entity by it’s
composition of components. An item component would then track all the
relevant data of the item (does it stack, is it carried, how many are
there, etc) and then Entity would manage the logic. The one thing that
makes an item more than just it’s data is the same problem that makes an
AI unique - what type of item is it and what does it do?

To solve this, I decided that the item component would have a simple
enumeration for it’s use function that would be set on making that
entity an item. Then when the actor uses the item, a switch could be set
up to call a specific (private) use function in entity.

`GameMap` needed some changes now - `GameMap::entityAt()` needed to be
expanded to `GameMap::itemAt()` and `GameMap::actorAt()`. Each of these
functions is identical, but the item and actor functions add a separate
conditional to check the entity to see if it is an actor or an item.
Also added was a function to place items - right now place items and
place actors are two separate functions. When the next level code is
implemented, this will likely be changed to account for dungeon depth.

Picking up items was less difficult than I thought it would be, the item
just needed to be removed from the game’s entity list and added to the
player’s inventory. The inventory is a good fit for another component,
and was originally added as a `std::vector<Entity>`. I quickly realized
that this would not work - you can’t rearrange, sort, or randomly remove
parts of a vector (all things necessary for a clean inventory). However,
this is a trivial matter for the doubly linked list -
`wsl::DLList<Entity>`! That darn class has been probably the single most
useful thing I’ve learned how to do with C++ so far - I’ll probably
write a more detailed blog post on it eventually.

Displaying the inventory took some time - I made a few mockups of what I
wanted the game UI to look like in REXpaint, but thing’s didn’t really
flow as well as I wanted. I really like the detailed screens in ADOM,
and will probably move the UI to look more inspired by that later. For
now, the inventory screen is super simple, super basic, and super clean.

{{< figure src="/assets/2018-07-24/Inventory.png" >}}

I actually really like how it looks right now, its not obtrusive and
it’s functional. Well, the UI overhaul is already on the To-Do list, so
I’ll revisit it then.

Dropping items is the opposite of picking them up - remove the item from
the player inventory list and add it to the game inventory list. Oh,
yeah, and change it’s current position. Otherwise dropping items
magically places them back to where they were picked up.

The engine already had an enumeration to track the current game state
and code to switch it, so the only real changes were to
`Engine::draw()`. The draw function checks which state the game is in
and draws the appropriate screen now. The `Input` class needed a bit of
an overhaul too, so a bunch of case statements were added to the switch
for keyboard input for each alphabet key on the keyboard. The class also
now has a simple char tracking what letter was pressed, and that is
easily checked against which item in the inventory it corresponds to:

{{< highlight cpp >}}
else if (gameState_ == GameState::INVENTORY)
    {
        if(input.alpha() >= 'a' && input.alpha() <= 'z')
        {
            int index = int(input.alpha() - 97);
            wsl::DLNode<Entity> * itemNode = player_->inventory()->at(index);
            if(itemNode)
            {
                player_->use(index);
                addMessage("You use the " + itemNode->data.name() + "!");
                changeState(GameState::ENEMY_TURN);
            }
        }
    }
{{< / highlight >}}

This basically checks if the player has an item in their inventory at
the index, and if it does calls `Entity::use()` and change states to the
enemy turn - which also effectively closes the inventory screen. The
code for dropping an item is similar, except it calls `Entity::drop()`
instead.

Here’s a super cool gameplay video showing all the neat things added in
the past week!

{{< video mp4="/assets/2018-07-24/Part8.mp4" >}}

---

My goal for the next week is to finish parts 9-11. I think I will tackle
targeting the same way the classic roguelikes did - when a target needs
to be selected, switch to a different state where player input with the
move keys moves a cursor instead of the player, and then whatever is
under the cursor is selected by pressing enter. I might also add a
"look" function in at the same time, since it should be relatively easy
to add and would be a good way to test the targeting code.  The saving
code is going to be fun - I've been trying to think about save files the
entire time I've been working on this project, asking "how will I
write/load this to a file" every time I add something to the core
classes. It'll be neat to see if my planning works out. Part 11 is also
going to be interesting, since most of the "next level" code is already
in place and just needs to be used.

All this coding has made me break one of my own self imposed coding
style rules - I’ve always tried to keep source files small (under 200
lines of code, if possible), and I’ve far surpassed that in the source
files for Engine and Entity. I expected to, and thought it wouldn’t
bother me - but it does. Rebuilding times are slowing down and scrolling
through tons of code is already irritating when I know where I’m going.
I have never split implementation files before, but I think that might
be something I’ll have to do shortly. `engine.cpp` could be split into
each of its functions: `engine_handleEvents.cpp`,`engine_update.cpp`,
and `engine_draw.cpp` for example, leaving the utility functions in the
core cpp file. Heck, draw could be split further into draw functions for
each game state, which might make handling the UI easier to deal with.
Really not sure if this will work, but it’s been a constant nagging in
the back of my brain every time I open a super long code file (engine is
677 lines long, and entity is already 290 lines). Maybe I’ll pull the
trigger and try this during the next week - if not it is definitely
going on the top of the To-Do ~~list~~ pile!


