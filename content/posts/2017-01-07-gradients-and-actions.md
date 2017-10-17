---
title:  "Gradients for Colors, and Actions for Actors"
date:   2017-01-07 11:45:00 -0700
---

Over the past couple of weeks I've been working on simplifying the code for my roguelike project - heavily inspired by one of my favorite books on my shelf,
[Game Programming Patterns](http://gameprogrammingpatterns.com).

I've seriously read this book cover to cover, multiple times - and every time find a new use for one of the 'patterns' in it. I just discovered that the
author's [blog](http://journal.stuffwithstuff.com/) has a bunch of good reads on it as well. One of the biggest problems I've had with the roguelike project was a simple way to issue commands 
(which should have been a clue) from the player to the character, and from the computer AI to the enemies.

The first chapter in the book, and [this](http://journal.stuffwithstuff.com/2014/07/15/a-turn-based-game-loop/) blog post had the answer - the 'Command' pattern. I have no idea how I didn't think of this sooner - basically,
all the actions that an actor (player or enemy) can take are issued as 'commands'. A command is just a subclass of an action, and each actor when updating 
gets to call it's most current action's `perform()` routine. What makes this super cool and useful is that the perform routine returns an `ActionResult`, which
has a flag stating whether or not the current action was successful, and a suggested alternate action (for use if the action was unsuccessful). 

{{< highlight cpp  >}}
class Action 
{
public:
    virtual ActionResult perform(Actor & actor) = 0;
protected:
    Action() { }
};

class ActionResult
{
public:
    ActionResult() { status_ = true; }
    bool success() { return status_; }
    void setFalse() { status_ = false; }
    void setTrue() { status_ = true; }
    Action * getAlternate() { return alternative_; }
    void setAlternate(Action * alternate) { alternative_ = alternate; }
private:
    Action * alternative_;
    bool status_;
};
{{< / highlight >}}

Now, if the player moves into a square - instead of the keypress moving the player, the keypress just sets the player's next action to attack in the direction of that tile.
During the players update, the `perform()` for the attack action checks to see if there is another actor there - and if there isn't returns a move action as an alternate. If
the move action fails (because the tile is blocked or whatever), an open door action is returned. If there isn't a door at the tile (or the door is locked and the player doesn't have the key),
the action returns null and the player doesn't 'lose' their turn.

This makes logic for AI so much easier, and the game easier to play. If I press 'o' (open door), but there's only open doors around - I probably wanted to close that door. AI when pathfinding can just ignore things like doors,
since moving into them will just use their turn to open the door - if they are capable.

Another fun thing I wrote recently is some code to dynamically create color gradients. It turns out, you can just do a simple line (y=mx+b) for component (r,g,b) of a color: `m` being equal to the change in color (finish - start) divided by the number
of steps, and `b` as the start color.

{{< highlight cpp  >}}
int getDeltaColor(int sC, int fC, int steps)
{
    // This is the slope equation for 'm' : (finishColor - startColor) / NumberOfSteps
	if(steps == 0)
	{
		return 0;
	} 
	return int((fC - sC) / steps);
}

std::vector<WColor> getColorGradient(WColor startColor, WColor finishColor, int steps)
{
	std::vector<WColor> result;

    // First, we get the slopes for each color component
	int redDelta = getDeltaColor(startColor.r, finishColor.r, steps);
	int greenDelta = getDeltaColor(startColor.g, finishColor.g, steps);
	int blueDelta = getDeltaColor(startColor.b, finishColor.b, steps);
	int alphaDelta = getDeltaColor(startColor.a, finishColor.a, steps);

	result.push_back(startColor);
	for(int i = 0; i < steps; ++i)
	{
		WColor stepColor;
        //Then at each step: stepColor = slope * stepNumber + startColor
		stepColor.r = (redDelta * i) + startColor.r;
		stepColor.g = (greenDelta * i) + startColor.g;
		stepColor.b = (blueDelta * i) + startColor.b;
		stepColor.a = (alphaDelta * i) + startColor.a;
		result.push_back(stepColor);
	}
	result.push_back(finishColor);

	return result;
}
{{< / highlight >}}

Finally, another cool thing I've been working on is playing around with Perlin noise. Inspired by [Red Blob Games's post](http://www.redblobgames.com/maps/terrain-from-noise/) on using Perlin noise to generate 2D maps I decided that I had to do something similar for my roguelike.
The code for the noise generating function is just a straight translation of Ken Perlin's [improved noise function](http://mrl.nyu.edu/~perlin/noise) which was written in Java. I have no idea the math behind most of this - but 
man is it cool.

Here's what I ended up with, starting with just a picture of the noise (check out the sweet dynamically created gradient from white to red!) and then a picture of the resulting map after the terrain is applied:

![Heatmap](/assets/2017-01-07-gradients-and-actions/heatmap.png)

![Terrain map](/assets/2017-01-07-gradients-and-actions/terrain.png)

Neat right? I'm not entirely happy with it yet, but it's a heck of a start! Applying the terrain to the heatmap was the tricky part. The perlin noise function returns numbers from -1 to 1, which I normalized (by dividing by two
and adding 0.5) to numbers from 0 to 1. I then took those numbers, multiplied them by 255, and forced the numbers to an integer. I then took the maximum number (highest 'elevation') and stored that, then multiplied the max elevation by 0.6 to find the waterline (so 60% of the map is underwater).
The difference between the max elevation and the waterline is divided by the number of land 'biomes' I have (six) and the terrain is divided by elevation that way. It's not great, but it works for now. I need to look at how
Dwarf Fortress's maps look again - but I can't open that game without being sucked into it for days at a time. I'd like to add rivers to the map, which I could do by just finding a mountain tile and 'rolling downhill' to get to
a water tile (dijkstra mapping!). I also need to work on placing locations, like dungeons and a town. 

I also tried to use characters to represent the increase in elevation, but couldn't settle on ones I liked - and ended up just using the ones from Dwarf Fortress. Even worse than not being able to decide on characters was
deciding on colors. My first attempt (like most of my attempts with colors) failed miserably - I just used whatever approximations I could think of for what colors things should be (green trees and grass,
 grey mountains, blue water - etc). I decided that I need to find a 'color palette' and use that exclusively in my roguelike. For lack of being able to come up with a better idea, I chose the ever popular [DawnBringer 16 color
palette](http://pixeljoint.com/forum/forum_posts.asp?TID=12795).

So, my basic idea for this game project is as follows: The player finds themselves in a small settlement on the coast of an island and is tasked with investigating various threats and ruins around the island. Every tile on the world
map will be able to be visited (ADOM style) and locations dynamically generated. Dungeons will also be dynamically generated, and each will have a 'theme'. The theme will determine what types of enemies the player encounters at a
location, along with the types of treasure and dungeon layout. The difficulty of the dungeons will be based on how far from the settlement town they are (dijkstra mapping to the rescue, again!). I'm not sure how quests will
fit into all this - I want the game to have a light sort of feel to it, where the story doesn't matter too much... just good ol' fashion dungeon crawling. Maybe the player will be given objectives like
vanquishing a Goblin Commander in a goblin themed cave, or finding a certain relic at the bottom of some ancient ruins populated by the undead.

