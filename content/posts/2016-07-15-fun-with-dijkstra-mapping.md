---
title:  "Fun With Dijkstra Mapping"
date:   2016-07-15 08:29:06 -0700
---

Continuing in my never-ending project to make a "simple" roguelike in C++, I decided to tackle something I have never before tried: artificial intelligence. My first rough idea was to implement some sort of component-type system, and build the AI up like I built up entities. This turned out to be far too complicated, and even getting a monster to move randomly was a chore. Obviously, this wasn't going to work.

After browsing [/r/roguelikedev](http://www.reddit.com/r/roguelikedev) and [RogueBasin](http://www.roguebasin.com/) I came across a few articles on "need driven AI" - an absurdly cool concept, and so (seemingly) simple to implement I wasn't sure how I didn't stumble across this earlier. Basically, creatures in the game assign a value to things they know about (treasure, the player, exits, other monsters they are friendly with, other monsters they are terrified of, etc. etc.) and make an 'informed' decision about where to move next. A monster could desire killing the player over treasure, and would "decide" to move towards the player instead of a closer pile of gold. Maybe that monster is a pack hunter, and also strongly desires to be near it's pack - so when singled out it moves away to regroup with others. Cats could hunt mice, the undead could wander aimlessly attacking anything, golems could guard exits, archers could stay a safe distance away, and 

...

Clearly, this idea has so many uses that I absolutely **had** to attempt an implementation in my game. The best [article](http://www.roguebasin.com/index.php?title=The_Incredible_Power_of_Dijkstra_Maps) I found describing what I hope to accomplish was written by the creator of the awesome Brogue, and talked about something called "Dijkstra mapping." A number on a Dijkstra map is the shortest distance from that tile to the goal. Moving from any numbered tile to any lower numbered tile adjacent will take you on the shortest path to the goal. Using this technique I could also accomplish intelligent pathfinding (another thing I've never been quite able to do on my own). So,	the first step was to write up something to generate these maps.

After much confusion (I'm still not much of a C++ programmer), I ended up with a fairly simple and straightforward set of code to generate a vector of integers representing a Dijkstra map from the vector of integers representing a normal dungeon map.

_As an aside, I previously determined that a vector of integers is my favorite way of representing a 2 dimensional map. The x,y coordinates can be mathematically used to determine the index of the vector if the width or height of the map is known. A simple enumeration in my map generator class assigns each number to a tile type, making it easy to say "There is a wall at (32,16), and a floor at (5,5)."_

The result (displayed on a Cartesian grid with the goal marked in red) looked like this:

{{< highlight cpp >}}
                                                                                                                                                                                                                                                
                                                                              28 27 26 25    23 22 21 20 19 18 19 20 21 22                                                                                                                      
                                                                              27 26 25 24    22 21 20 19 18 17 18 19 20 21             38 37 36 35 34 33 32 31          33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48
                                                                              26 25 24 23 22 21 20 19 18 17 16 17 18 19 20                                  30          32 33 34 35 36 37 38 39 40 41             48
                                                                              27 26 25 24    20 19 18 17 16 15 16 17 18 19             36 35 34 33 32 31 30 29 28 29 30 31 32 33 34 35 36 37 38 39 40       51 50 49 50
                                                      42 41 40 39 38          28             19 18 17 16 15 14 15 16 17 18                                  28 27 28    32 33 34 35 36 37 38 39 40 41       52 51 50 51
                                                      41 40 39 38 37    31 30 29 30                         13                                              27 26 27    33 34 35 36 37 38 39 40 41 42       53 52 51 52
                                                      40 39 38 37 36    32 31 30 31 32 33                13 12 11 10 11 12                                  26 25 26
                                                      39 38 37 36 35 34 33 32 31 32    34    16 15 14 13 12 11 10  9 10 11                                  25 24 25          29 30 31 32    34 35 36 37 38 39 40
                                                      40 39 38 37 36                   35                11 10  9  8  9 10                                     23             28 29 30 31    33 34 35 36 37 38 39 
                                                      41 40 39 38 37                   36                10  9  8  7  8  9 10 11 12 13 14       17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38
                                                         41                                                        6                            16 17 18 19 20 21 22 23 24    28 29 30 31    33 34 35 36 37 38 39
                                                45 44 43 42 43 44             63                 5  4  3  2  3  4  5  6     8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23    29 30 31 32 
                                                46 45 44 43 44 45             62                 4  3  2  1  2  3  4  5     7  8  9 10 11 12    16 17 18 19 20 21 22 23 24 
                                                47 46 45 44 45 46             61                 3  2  1 _@_ 1  2  3  4  5  6  7  8  9 10 11             20
               59 58 57 56 55 54 53 52 51 50 49 48 47 46 45 46 47             60                 4  3  2  1  2  3  4  5     7  8  9 10 11 12          22 21 22
                  59                            49 48 47 46 47 48             59                 5  4  3  2  3  4  5  6                               23 22 23
               61 60 61                                     48                58                 6                                                    24 23 24
               62 61 62                56 55 54 53 52 51 50 49 50 51 52 53    57    11 10  9  8  7  8  9 10                                           25 24 25
               63 62 63                57    55 54 53 52 51 50 51 52 53 54 55 56    12 11 10  9  8  9 10 11                                           26 25 26    30 31 32 33 34 35 36 37 38 39 
               64 63 64                58    56 55 54 53 52 51 52 53 54 55          13 12 11 10  9 10 11 12                                     29 28 27 26 27 28 29 30 31 32 33 34 35 36 37 38
               65 64 65                59                                           14 13 12 11 10 11 12 13                                     30    28 27 28    30 31 32 33 34 35 36 37 38 39 
                     66                60                                           15 14 13 12 11 12 13 14                                     31 
            70 69 68 67 68 69 70       61                                           16 15 14 13 12 13 14 15
            71 70 69 68 69 70 71       62                                           17 16 15 14 13 14 15 16
            72 71 70 69 70 71 72       63                                           18 17 16 15 14 15 16 17
                                       64                                           19 18 17 16 15 16 17 18

{{< / highlight >}}

Neat right?? Here's the code.

{{< highlight cpp >}}
#pragma once
#ifndef WSL_DIJKSTRA_HPP
#define WSL_DIJKSTRA_HPP

#include <vector>
#include <algorithm> // std::min_element
#include <iterator>  // std::begin, std::end
#include "wsl_wvec.hpp"

namespace wsl
{
struct DijkstraMap
{
	std::vector<int> map;
	int mapWidth;
	int mapHeight;
	DijkstraMap() { }
};

DijkstraMap generateDijkstra(std::vector<int> inputMap, int w, int h, std::vector< WVec2<int> > goals);
DijkstraMap generateDijkstra(std::vector<int> inputMap, int w, int h, WVec2<int> goal);

DijkstraMap generateDijkstraSafety(std::vector<int> inputMap, int w, int h, std::vector< WVec2<int> > goals);
DijkstraMap generateDijkstraSafety(std::vector<int> inputMap, int w, int h, WVec2<int> goal);

void scanDijkstra(DijkstraMap & inputMap);

int getDijkstraIndex(int w, int x, int y);
int getDijkstraIndex(DijkstraMap & inputMap, int x, int y);
int getDijkstraIndex(DijkstraMap & inputMap, WVec2<int> pos);
int getDijkstraIndex(WVec2<int> dim, int x, int y);
int getDijkstraIndex(WVec2<int> dim, WVec2<int> pos);

} // namespace wsl

#endif // WSL_DIJKSTRA
{{< / highlight >}}

{{< highlight cpp >}}
#include "wsl_dijkstra.hpp"
#include <iostream>

namespace wsl
{

/*
 * These two functions take an input, generally generated by one of the map functions (IE: wsl::RLDungeon) with passable tiles set to a high number,
 * and not passable tiles (IE: walls, pits, etc) set to exactly 999. The width and height of the map are also passed, and another function could be
 * added to pass in a specific dungeon type... but I felt it was unnecessary. The goal/goals are passed as 2 dimensional vectors (WVec2).
 *
 */
DijkstraMap generateDijkstra(std::vector<int> inputMap, int w, int h, std::vector< WVec2<int> > goals)
{
	DijkstraMap resultMap;
	resultMap.map = inputMap;
	resultMap.mapWidth = w;
	resultMap.mapHeight = h;
	for(int i = 0; i < goals.size(); ++i)
	{
		resultMap.map[getDijkstraIndex(resultMap, goals[i])] = 0;
	}
	scanDijkstra(resultMap);
	return resultMap;
}

DijkstraMap generateDijkstra(std::vector<int> inputMap, int w, int h, WVec2<int> goal)
{
	std::vector< WVec2<int> > goals;
	goals.push_back(goal);
	return generateDijkstra(inputMap, w, h, goals);
}

/*
 * Safety maps are a neat way to invert the Dijkstra maps, so that moving to lower numbers on the map will take you AWAY from the goal.
 */

DijkstraMap generateDijkstraSafety(std::vector<int> inputMap, int w, int h, std::vector< WVec2<int> > goals)
{
	DijkstraMap resultMap = generateDijkstra(inputMap, w, h, goals);
	for(int i = 0; i < resultMap.map.size(); ++i)
	{
		if(resultMap.map[i] == 999)
		{
			continue;
		}
		resultMap.map[i] *= -7;
		resultMap.map[i] /= 5;
	}
	scanDijkstra(resultMap);
	return resultMap;
}

DijkstraMap generateDijkstraSafety(std::vector<int> inputMap, int w, int h, WVec2<int> goal)
{
	std::vector< WVec2<int> > goals;
	goals.push_back(goal);
	return generateDijkstraSafety(inputMap, w, h, goals);
}

/*
 * The scanning function, the iteration through the map tiles could most likely be improved/sped up, but this was the simplest way to do it.
 * The function takes a reference to a DijkstraMap, and modifies it directly.
 */
void scanDijkstra(DijkstraMap & inputMap)
{
	int impassable = 0;
	int passable = 0;
	int changes = 0;
	std::vector< wsl::WVec2<int> > passableCoords;
	for (int y = 0; y < inputMap.mapHeight; ++y)
	{
		for (int x = 0; x < inputMap.mapWidth; ++x)
		{
			int index = getDijkstraIndex(inputMap, x, y);
			if(inputMap.map[index] == 999)
			{
				impassable++;
				continue;
			}
			else
			{
				passable++;
				passableCoords.push_back(wsl::WVec2<int>(x, y));
			}
		}
	}
	std::vector<bool> visited;
	int target = passableCoords.size();
	visited.assign(passableCoords.size(), false);
	bool madeChange = true;
	while(madeChange)
	{
		madeChange = false;
		for(int j = 0; j < passableCoords.size(); ++j)
		{
			if(visited[j])
			{
				continue;
			}
			int x = passableCoords[j].x;
			int y = passableCoords[j].y;
			std::vector<int> minMaxVector;

			if(getDijkstraIndex(inputMap, x, y-1) > 0)
			{
				minMaxVector.push_back(inputMap.map[getDijkstraIndex(inputMap, x, y-1)]);
			}
			if(getDijkstraIndex(inputMap, x, y+1) > 0)
			{
				minMaxVector.push_back(inputMap.map[getDijkstraIndex(inputMap, x, y+1)]);
			}
			if(getDijkstraIndex(inputMap, x-1, y) > 0);
			{
				minMaxVector.push_back(inputMap.map[getDijkstraIndex(inputMap, x-1, y)]);
			}
			if(getDijkstraIndex(inputMap, x+1, y) > 0)
			{
				minMaxVector.push_back(inputMap.map[getDijkstraIndex(inputMap, x+1, y)]);
			} 
			int minimum = *(std::min_element(std::begin(minMaxVector), std::end(minMaxVector)));
			int index = getDijkstraIndex(inputMap, x, y);
			if((inputMap.map[index]) > (2 * minimum))
    		{

    				inputMap.map[index] = 1 + minimum;
    				changes++;
    				madeChange = true;
    				visited[j] = true;
    		}
		}
	}
	//if(changes != passableCoords.size())
	//{
	//	std::cout << "Changes do not equal passableCoords.size(): " << changes << " vs " << passableCoords.size() << std::endl;
	//}

}

/*
 * The getIndex functions will tell you the index of the DijkstraMap.map vector given an x,y coordinate.
 */
 
int getDijkstraIndex(int w, int x, int y)
{
	int index = (x + (w * y));
    if (index < 0)
    {
        return -1; //Error, no tile found (necessary otherwise it could point to an item in the vector from the end)
    }
    return index;
}

int getDijkstraIndex(DijkstraMap & inputMap, int x, int y)
{
	return getDijkstraIndex(inputMap.mapWidth, x, y);
}

int getDijkstraIndex(DijkstraMap & inputMap, WVec2<int> pos)
{
	return getDijkstraIndex(inputMap.mapWidth, pos.x, pos.y);
}


int getDijkstraIndex(WVec2<int> dim, int x, int y)
{
	return getDijkstraIndex(dim.x, x, y);
}

int getDijkstraIndex(WVec2<int> dim, WVec2<int> pos)
{
	return getDijkstraIndex(dim.x, pos.x, pos.y);
}

} // namespace wsl
{{< / highlight >}}

So it isn't perfect - I'm sure the iteration through the map cells could be optimized to run differently, and I'm not sure about making DijkstraMap a struct instead of a class. I originally tried this with a class, but the class seemed bulky and awkward to shove in to my game logic. Really, all the map should be is a vector of integers, and the only thing it needs to know about itself is the width/height of the map. Keeping the Dijkstra functions open and accessible means that they can be used anywhere, for really any reason.

From here, I decided to see just how 'easily' the maps could be used to implement things like auto-exploration of the map - turns out, it was INCREDIBLY easy. Instead of setting just one goal before generating the map, setting the goal to any unexplored area will cause the player to take the shortest possible route across the map to explore every single tile. Once all the tiles are explored, the goal is changed to the downward staircase, and the player paths quickly there.

And here's a video of the auto-explore in action (I accidently forgot about the cursor when I started recording the screen...):
<div style="margin: auto; text-align: center; position: relative; width: 100%; font-family: 'Roboto Mono', monospace; overflow: auto;">
<video style="margin: auto; padding: 1px;" controls src="{{ site.baseurl }}/assets/AutoExplore.mp4" type="video/mp4">Your browser does not support embedded videos!</video>
</div>

The next project will be actually using this in the game AI, and adding a 'desire' component for each monster. Of course, this might also be a good time to implement a time/turn tracking system... Or a health system...
