---
title: "The Roguelike Tutorial - Week 3b"
date: 2018-07-10T22:31:00-07:00
tags:
  - C++
  - Roguelike
  - Barbarian!
---

I originally planned for this week's blog post to cover my adaptation of part 5 of the Python RL tutorial - but after finishing part 5 in a matter of minutes
I realized there wouldn't be much to write here! Part 5 is basically making new entities and getting the collision logic in place before combat is added in
part 6. However, before adding combat I thought it would be beneficial to start thinking about game time - it's far more fun when some enemies are a lot quicker than
the player, forcing the player to devise strategies for survival. 

Naturally, there's a few [articles](http://www.roguebasin.com/index.php?title=Articles#Time_management) on roguebasin - [one](http://www.roguebasin.com/index.php?title=An_elegant_time-management_system_for_roguelikes)
of which I've read **many** times over the years, and never understood a word of it. Pascal is a goofy, blunt looking language and really makes you appreciate
how nice programming languages read these days. Or, maybe I'm just dense and don't get it. I love that article though - not only because it references my favorite
roguelike [ADOM](https://www.adom.de/home/index.html)(which, despite playing for over a decade have never beat), but because of the word "elegant." It just sounds
so, appealing. 

I sat down, notebook ready, to try and break this article in pieces and figure out what it's saying. The very first thing it says is:

 > Readers should be familiar with a linked list (next) and doubly-linked list (prev and next). A circular list simply means that the last object's "next" points to the first object; and in a doubly-linked circular list the first object's "prev" points to the last object. The advantage of a circular list over a regular list is that a circular list need not cater for null-references during insertion or deletion of objects.

Well, my severe lack of formal computer science education meant that I had no idea what any of those things are, really, beyond the fact that they exist and there
is likely an STL implementation of those I could find. Searching google for "linked list C++" led me to [this fantastic site](https://www.geeksforgeeks.org/data-structures/linked-list/).

I decided to take a step back from this project at this point and learn all about lists.

## Super Cool Lists
A list is apparently nothing more than a bunch of linked nodes, with each node containing a piece of data and a pointer to the next element in the list. Simple
enough, right? Well after reading and writing a few simple little examples toying with creating lists, I had the realization that this was exactly what I was 
missing in my understanding of pathfinding algorithms and could easily be used for a sort of scheduling system. If a list contained nodes whose data was just a
pointer to an entity, when that node is "popped" off the top of the list it could be called for updating in the engine. This means that if a list was kept as 
a "schedule", the entities who can act could be added to the schedule and popped off in sequence for updating. Cool right?

So the update routine is now split into two parts: First if the schedule is not empty, it goes through and updates each actor in the schedule by popping them
off the top. Ideally, I would be doing this in a `while(!schedule_->isEmpty())` loop, and will probably do that once I finish the actor update routines. A more
graceful way of handling entity updates would be to just return if the entity hasn't decided on an action, and set the entity actions in their AI update (for the
enemies) or in the keyboard input handling (for the player). For now,
since we have to let the player have a chance to use their turn, the conditional just returns after removing one actor from the schedule which allows the engine to
continue monitoring input.

{{< highlight cpp >}}
void Engine::update()
{
    if(!schedule_->isEmpty())
    {
        if(curActor_ == NULL)
        {
            curActor_ = schedule_->popFront();
        }
        /*
         * The next few lines will be replaced with
         * if(curActor_->nextAction == Action::NONE) return;
         * else curActor_->update();
         * once that system is in place.
         */
        if(gameState_ == GameState::PLAYERS_TURN)
        {
            return;
        }
        else
        {
            curActor_->update();
        }
        curActor_->energy() -= ACTION_COST;
        curActor_ = schedule_->popFront();
        return;
    }
{{< / highlight >}} 

Next, when the schedule is empty the game advances a "tick" and grants each actor a set amount of energy. Currently, the actors have a base speed that is added
to their energy pool every tick. When their energy level is greater than the cost of an action (a constant, currently set to 100), they get added to the schedule.

{{< highlight cpp >}}
    else
    {
        for(int i = 0; i < entityList_->size(); ++i)
        {
            entityList_->at(i).grantEnergy();
            if(entityList_->at(i).energy() >= ACTION_COST)
            {
                schedule_->push(&entityList_->at(i));
            }
        }
        player_->grantEnergy();
        if(player_->energy() >= ACTION_COST)
        {
            schedule_->push(player_.get());
        }
        gameState_ = GameState::PLAYERS_TURN;
    }
} 
{{< / highlight >}} 

Simple, right? What makes this neat is 

 * If the `ACTION_COST` is set to 100
 * Players speed is 50
 * Slug speed is 25
 * Bat speed is 100

Then for every 4 turns the player takes, the bat gets 8 and the slug gets 1. Now if the Bat is able to intelligently use hit and run tactics...

The other cool thing about lists is that making a simple priority queue is an easy leap from a linked list - just attach a priority to each node and arrange them
when the nodes are inserted! This made pathfinding much, much easier, and I can finally understand and interpret the [Red Blob Pathfinding Guide](https://www.redblobgames.com/pathfinding/a-star/introduction.html)!
Right now I have a basic greedy best first search up and running, and it's **quick**. Now, it's not very optimal for long paths, but the way I figure it is the
undead denizens of the ruins our burly hero is delving will only start tracking the player once they see/sense the player - which means they won't start hunting all
the way across the map right off. I'm still tinkering with trying to get A-Star pathfinding working, but I'm anxious to get the enemy actors to actually start doing
something! Plus, I'm a tad bit behind the current challenge schedule. I'm going to add this to my list of things to continue working on after the challenge is over.

---

Up next in part 6 is combat, which should be a lot more fun now that enemies can take turns at different rates and can intelligently move towards the player.
Hopefully, if all goes well, I'll be able to show off a nice video with the next update that will also feature a scrolling message log and an inventory screen
(part 7)! In the meantime, the [repository](https://github.com/zwilder/barbarian) is here - CLOC reports 2561 lines of code written, and hopefully it's legible
and easy enough to follow.

Here is the source code for [Part 5](https://github.com/zwilder/Barbarian/tree/Part_5)!
