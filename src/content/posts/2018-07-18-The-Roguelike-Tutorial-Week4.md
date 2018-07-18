---
title: "The Roguelike Tutorial - Week 4"
date: 2018-07-18 12:00:00 -0700
tags:
    - C++
    - Roguelike
    - Barbarian!
---

The past week has been equal parts frustration and excitement with this project. I ran into a pretty common design problem: entity components need to know about the rest of the game. Well, since each entity is just a collection of components it shouldn't be difficult to pass a pointer to the game engine. Then, the components would have a pointer to the entity that "owns" it. Simple and clean... or so I thought. This didn't work, at all. Individual entities could access the engine just fine, but the components just could not access anything about their owner. Even trying to print the address of the owner caused the game to segfault and crash. GDB was no help, and I tried MANY things to get this to work.

I realized that I was bashing my head against this problem, and totally stumped, so I decided to work on something else. I changed  the Engine's entity list from a `std::vector<Entity>` to a `wsl::DLList<Entity>` (from the fancy list class I created last week). To my surprise and delight, it worked perfectly! What's nice about using the list I created is that it manages it's own memory - I don't have to worry about allocations/deallocations no matter how big the list gets!

This got me thinking about best practices and clean code again. In general, I've heard you should **always** use the Standard Template Library containers over a custom rolled solution. However, I know my code, I know how it works, and I can fix/change it with minimal fuss. Creating a custom wrapper for `std::priority_queue` or `std::map`? No way, I have no idea where to even begin! 

So, with this in mind I returned to my original problem. To hell with best practice, I'm not a professional and I don't work with other programmers. My code is written clearly and commented well, so it should be easy to figure out why I did things the way I did. I decided that a good, bare bones, no frills system has components that are nothing more than data containers, and entities that are collections of components. All the logic is then moved to the entity class, all the accessors are in the entity class, and the entity itself is responsible for maintaining it’s components. Yeah, this does mean that a good large chunk of the game is going to be in the entity class, and that the class may get a bit bloated - but who cares? It works, and my original problem went away completely. It will make serialization easier too, because I only need to serialize Entity - Entity can destruct/construct all of it’s components from raw data.

## Part 6
With all that nonsense out of the way, I set about implementing combat. The first step was collecting all the data parts of the Entity class that dealt only with actors and moved it to the new Actor component, and adding in the health points, defense, and power data. Eventually, I’ll want to implement a better combat system (I like the idea of a [percentile based system](https://www.reddit.com/r/roguelikedev/comments/46i4i7/comment/d05al46)), but for now I’ll stick with following the ideas in the tutorial. Combat was actually really surprisingly easy to add at this point!

The minor tweaks needed were actually minor, which is nice for a change - `GameMap::isBlocked()` for instance only returned true if the tile at the spot blocked movement, it now also checks the entity list for an entity at that spot that blocks movement. What’s nice is this inadvertently fixed a glitch with the pathfinding, monsters now see other monsters as obstacles in the way of their target. `Engine::update()` also checks if an entity is dead, to remove it from the entity list and change the tile it occupied to a corpse. Adding a GameState for player death was also easy, so the game actually “ends” now. I still need to make the `Engine::newGame()` function, but that shouldn’t be difficult. All the work put into this so far paid off, watching monsters discover, path to, and attack the player is very exciting - I can`t wait to continue tweaking combat and actually use the time management I wrote last week!

## Part 7
So I have an idea of how I want this game to look - eventually. However, I decided to just drop a basic UI in the game for Part 7. The first task here was to add a `Console::print()` function - ha, the last time I opened the console code was **way** back when the project was first started! Fortunately it’s all pretty simple and I tried my best to write it with modularity in mind so I can use it on later projects. A print function has one task, display a string at a coordinate position. Easy enough - but what if the string is longer than the console width? Well, thats another easy fix, just advance the y part of the coordinate pair and continue writing! Simple!

{{< highlight cpp >}}
void Console::print(int x, int y, std::string msg)
{
    size_t msgLength = msg.size();
    int widthAvailable = width_ - x;
    int dX = 0;
    int dY = y;
    for(size_t i = 0; i < msgLength; ++i)
    {
        if(dY != y)
        {
            dX = x + int(i) - widthAvailable;
        }
        else
        {
            dX = x + int(i);
        }
        if(dX >= width_)
        {
            dX = 0;
            dY += 1;
        }
        screen_[dX + (dY * width_)] = msg[i];
    }
}
{{< / highlight >}}

Beautiful! Simple! It works! But wait, do you see the problem? Yeah, this splits a string into characters instead of words! Dang it. Ah well, it’s now my next thing to fix.

The message log is yet another good fit for the list class I wrote last week. Engine now has a list it maintains of messages. When something needs to add a message, the engine examines the top message on the list (head), and combines it with the new message if the combined length is under a maximum size (the size of the area on top of the screen for log messages). If the combined string is over the maximum length, the message will be truncated and pushed to the list with a new message created for the remainder. A new game state will be added shortly, `GameState::MSG_WAIT` which will pause the game (“Press any key to continue”) when there is more than one message in the message list. I want to limit the amount of message noise the player sees, so hopefully this shouldn’t occur too often. I also want to add a 6x12 font file for the UI, since square fonts are sorta ugly to read. More stuff for the To-Do list!

Here’s a video with this week’s progress and how the game looks at the end of Part 7:

{{< video mp4="/assets/2018-07-18/RoughUI.mp4" >}}

Super cool, and I’m pretty stoked at this point. Next on the agenda (besides some minor fixes listed above) is tackling items and healing potions!
