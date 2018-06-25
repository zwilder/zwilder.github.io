---
title: "The Roguelike Tutorial - Week 2"
date: 2018-06-25T08:00:00-07:00
tags:
  - C++
  - Roguelike
  - Barbarian!

---

So this week's goal is to finish [part 2](http://rogueliketutorials.com/libtcod/2) and [part 3](http://rogueliketutorials.com/libtcod/3) of the tutorial - I
worked ahead because I was excited to get this project going, which worked out well since shortly after finishing part 2 I switched to SDL from SFML and it gave
me some time to work on this blog. 

## The Switch to SDL
So the game logic is completely divorced from the rendering system - I don't want to be tied into using a set framework if I want to change later, and I know for a
fact from previous projects this will make save files **much** easier to generate. I briefly described last week that the game has a virtual console that is written
to, and is read/translated into graphics to be
displayed by SFML. Well, the console is a grid of say, 80x50 (960px x 600px). Each cell in the grid has a glyph that needs to be translated and rendered. During 
engine initialization, it loads one texture (image) as the spritesheet - in this case it's a .png image of the CP437 font.

{{< figure src="/assets/2018-06-25/cp437_12x12.png" >}} 

Now what Engine::Draw() did originally was make a new SFML sprite for each cell, with the position set to the font size multiplied by the cell x,y coordinates.
Seemed simple enough, and it worked - but that's 4000 sprites created and destroyed EVERY time Draw() is called. The next iteration created two arrays of sprites
in Engine::Init() that were persistent and lived throughout the life of the program - one sprite for each cell in an array, and one for each glyph. Then, in draw,
each cell's sprite was just changed to be the same value as the sprite representing the glyph on the console. Yep, that worked too, but it still seemed there had
to be a cleaner, better way of doing this.

I realized something shortly after finishing that, I don't need all those sprites. I could just create an array of rectangles representing the sprite location on 
the texture and then change each cell's sprite location in that array! Further, doing this I didn't really need 4000 sprites (one for each cell) - couldn't I just
use one sprite, updating it's position and rectangle before drawing it, since I only clear the render surface before drawing the first time? Well, SFML just did 
not like this - but that was okay, since things were working for now.

I finished part 3 of the tutorial, which created a fairly simple dungeon generator, which looked like this:

{{< figure src="/assets/2018-06-25/Screenshot1.png" >}}

Pretty neat! However, one of the main design considerations for this game is **completely** divorcing from the rendering - and in order to do so during this part
I created two new classes [wsl::Rect](https://github.com/zwilder/Barbarian/blob/master/include/rect.hpp) and [wsl::Vector](https://github.com/zwilder/Barbarian/blob/master/include/vector.hpp). Rect is a rectangle, defined by passing an x,y coordinate and a width/height, a pretty simple class.
Vector, on the other hand, I went a tad overboard with. I like multi-use classes, because I can use them in other projects - so for Vector I brushed up on my C++
Template skills and made vector a fancy template class. Honestly, it is a **lot** of code for a simple class that's basically a glorified holder for an x,y
coordinate pair! For ease of use I liked how SFML defined Vector2<int> as Vector2i, Vector2<float> as Vector2f etc - so I 
borrowed that. I'll probably make Rect into a fancy template class too - because it's fun and useful.

I realized at this point that SFML is just a little too full featured for what I have planned for this game. I had previously gone through the [LazyFoo SDL2](/)
tutorials and really liked playing with SDL2. I decided to look at some of my older SDL2 code and see if I could modify and adapt the LazyFoo texture handling
tutorial. This resulted in two more classes, [wsl::Texture](https://github.com/zwilder/Barbarian/blob/master/include/texture.hpp) and [wsl::Sprite](https://github.com/zwilder/Barbarian/blob/master/include/sprite.hpp) - I won't go into detail on those, since they are pretty straightforward
adaptations of the LazyFoo tutorials.

This brought me back to my problem with rendering, and wanting to just use one sprite as a sort of cursor to translate the console to the screen - heck, if I'm
writing the code then I'll make it work. Turns out, I didn't even need to do anything special! Everything renders quickly, cleanly, and I have the satisfaction of
having written some new code.

## BSP Dungeon Algorithm

Using binary space partitions to generate a dungeon is a pretty well documented process, but the amount of understandable or legible C++ code out there is pretty
sparse. I found a few tutorials on making a BSP dungeon in a few other languages - and put this together, following along as usual with a [roguebasin article](http://www.roguebasin.com/index.php?title=Basic_BSP_Dungeon_generation).
Basically, a binary tree is a collection of nodes, and each node can have exactly zero or two daughters. We then work our way down, splitting nodes into daughter
nodes, until a minimum node size (width/height of a rectangle) is reached. The very first node is the 'root' of the tree, and the very last nodes at the bottom are
the 'leaves' of the tree. To make this, I needed two classes, [Node and Tree](https://github.com/zwilder/Barbarian/blob/master/include/BSP.hpp).
Each Node has a few things to keep track of: it's parent, it's children, and it's siblings. The Tree needs to keep track of it's root, it's leaves, and the
rooms/corridors. The Node class contains the logic for splitting itself into children, and the Tree contains the logic for populating itself. 

Now, granted, this is all pretty abstract - so let's break down the code for splitting the Node first.

{{< highlight cpp >}}
Node::Node(wsl::Rect rect, Node * parent)
{{< / highlight >}}

A Node is created by passing it a rectangle representing it's area, and a reference to it's parent. Parent can be `NULL` - since the root has no parent.

{{< highlight cpp >}}
bool Node::split()
{
    bool success = false;
    if(!hasSplit())
    {
        bool horizontalSplit = wsl::randomBool();
        int width = nodeRect.w;
        int height = nodeRect.h;

        if((width > height) && (100 * (width / height) > 125))
        {
            horizontalSplit = false;
        }
        else if((height > width) && (100 * (height / width) > 125))
        {
            horizontalSplit = true;
        }

{{< / highlight >}}

First step is seeing if the node has already split, because if it has we don't want to split it again. Now, theoretically this will NEVER happen, but it never
hurts to have a bit of backup in your code just to make sure. Then, we have to decide whether the Node will be split horizontally or vertically. Simple enough to
select a random boolean - but what if splitting it horizontally makes the children nodes weirdly small? The next few lines cover that by making sure that if
the width is greater than the height and the ratio of width to height is greater than 1.25 then horizontalSplit will always be false. Note that since width
and height are integers, I normalized both sides by multiplying by 100.
 
{{< highlight cpp >}}
        int max = (horizontalSplit ? height : width) - MIN_NODE_SIZE;
        if(max <= MIN_NODE_SIZE) 
        {
            // Too small to split further
            success = false;
        }
        else // max > MIN_NODE_SIZE
        {
            int split = wsl::randomInt(MIN_NODE_SIZE, max);

            if(horizontalSplit)
            {
                splitPos_.y = split;
                splitPos_.x = nodeRect.x1;
                leftChild_ = new Node(wsl::Rect(nodeRect.x1, nodeRect.y1, width, split), this);
                rightChild_ = new Node(wsl::Rect(nodeRect.x1, nodeRect.y1 + split, width, height - split), this);
            }
            else
            {
                splitPos_.x = split;
                splitPos_.y = nodeRect.y1;
                leftChild_ = new Node(wsl::Rect(nodeRect.x1, nodeRect.y1, split, height), this); 
                rightChild_ = new Node(wsl::Rect(nodeRect.x1 + split, nodeRect.y1, width - split, height), this);
            }
            leftChild_->sibling_ = rightChild_;
            rightChild_->sibling_ = leftChild_;
            horizontal_ = horizontalSplit;
            success = true;
            split_ = true;
        }
       
    }

    return success;
}

{{< / highlight >}}

So most of the rest of the split function is self explanatory. The first check is to make sure that the large side of the node is greater than a constant 
`MINIMUM_NODE_SIZE`. `splitPos_` is a `wsl::Vector2i` that keeps track of the position of the split (which will come in handy when I add the corridors).
`leftChild_` and `rightChild_` are named for position on the tree, not so much position in the context of the game - so if we split
horizontally the left/right child are on the left and right of the node. If it's split vertically, the left/right child become the top/bottom child. 

All that is left now is to populate the tree! 

{{< highlight cpp >}}
void Tree::populate(wsl::Rect rootRect)
{
    root_ = Node(rootRect);

    std::vector<Node *> growth;
    growth.push_back(&root_);
    
    while(!growth.empty())
    {
        Node * currentNode = growth.back();
        if(!currentNode->hasSplit())
        {
            // Check if node can split, if so split it and add children to growth
            if(currentNode->split())
            {
                growth.push_back(currentNode->leftChild());
                growth.push_back(currentNode->rightChild());
            }
            else // if it can't split, add it to leaves
            {
                leaves_.push_back(currentNode);
                growth.pop_back();
            }
        }
        else // currentNode.hasSplit() == true
        {
            growth.pop_back();
        }
    }
}
{{< / highlight >}}
 
To populate the tree is pretty simple after having the split function taken care of. We start by creating a root node of a defined size (passed into the function).
We need to check every node as we create it, so a vector containing the current nodes `growth` is created and root is added to it. Then, while the growth vector has 
Nodes in it, we check a Node, split it if possible, and if not add it to the leaves and remove it from the growth vector. This leaves us with a vector of leaves
ready to put some rooms in! Here's a picture of what a result can look like with the minimum node size set to '8' on an 80x50 console.

{{< figure src="/assets/2018-06-25/Screenshot2.png" >}}

In each of those rectangles a smaller rectangle is then randomly placed, which gives something like this:

{{< figure src="/assets/2018-06-25/Screenshot3.png" >}}

Unfortunately... After all this fun work, if each room is connected to the room in the sibling node working our way up the tree, it ends up with a single
path connecting the dungeon! If instead the rooms are connected Rogue style (choosing a random x,y point on each room to connect) or tutorial style (connecting
the center of the rooms) this ends up looking darn near identical to the tutorial dungeon algorithm! Shoot. 

<!-- <iframe src="https://giphy.com/embed/hwIhyQDzVhn56" width="480" height="265" frameBorder="0"></iframe> -->

Well, they can't all be winners. Fortunately, roguebasin has some other awesome dungeon generation articles - I've already modified one of them for use in 
another project, and will most likely modify that same one again for this project.
