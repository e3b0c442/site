+++
title = "Advent of Code 2015, Day 3: Perfectly Spherical Houses in a Vacuum"
date = 2018-12-21T20:12:00-06:00
tags = ["Advent of Code", "Advent of Code 2015"]
markup = "markdown"
+++

The elves have everything wrapped up, and it's time for Santa to do some more deliveries.

> Santa is delivering presents to an infinite two-dimensional grid of houses.
> 
> He begins by delivering a present to the house at his starting location, and then an elf at the North Pole calls him via radio and tells him where to move next. Moves are always exactly one house to the north (`^`), south (`v`), east (`>`), or west (`<`). After each move, he delivers another present to the house at his new location.
> 
> However, the elf back at the north pole has had a little too much eggnog, and so his directions are a little off, and Santa ends up visiting some houses more than once. How many houses receive __at least one present__?
> 
> For example:
> 
> * `>` delivers presents to `2` houses: one at the starting location, and one to the east.
> * `^>v<` delivers presents to `4` houses in a square, including twice to the house at his starting/ending location.
> * `^v^v^v^v^v` delivers a bunch of presents to some very lucky children at only `2` houses.


## Analysis

<!--more-->
This problem elicits echos of Day 1, except it involves more dimensions.

We are given the setup of an infinite two-dimensional grid of houses. Unfortunately for us, allocating an infinite amount of memory is impossible. Further muddying the water is the fact that arrays cannot have negative indices.

There are two ways we could handle this:

1. Guess, allocate an array much larger than we think we will need, and set the origin point right in the middle.
2. Make an extra pass on the input to determine exactly what the needed dimensions are.

Option 1 sounds like a fine idea if we are trying to save execution time at the expense of memory... until we find the input that we didn't expect, segfault, and have to spend a bunch of time debugging. In that context, taking the extra pass through the input to determine our actual needs really looks like the only option.

So, we will make one pass through the input, moving our cursor appropriately and measuring the actual space we need by tracking the maximum and minimum values of `x` and `y`. We can then use this information to allocate a grid large enough to contain all of the data (`[max_x - min_x + 1][max_y - min_y + 1]`) as well as a starting offset (`[abs(min_x), abs(min_y)]`) to ensure that we do not overflow the grid or have a negative index.

Once we have allocated the grid, we make a second pass through, incrementing the value at the cursor. Finally, we loop over every element of the grid, incrementing a count for every cell that has had at least one present delivered to find our solution.

To analyze our complexity, lets break down the operations;

* One loop through the input data to determine our size: _O(n)_
* Another loop through the input data to populate our grid: _O(n)_
* A loop through the grid to build our count. Our function here is not known and not necessarily directly related to _n_; however, given a single loop through a set of data we can reasonably call this _O(n)_.

This gives us a complexity of _O(3n)_ which reduces asymptotically to _O(n)_.

### C
{{< highlight c >}}
FILE *input = fopen(inputFile);

char c;
int x = 0, y = 0, x_min = 0, x_max = 0, y_min = 0, y_max = 0;
while(fread(&c, 1, 1, input)) {
    switch(c) {
    case '^':
        y++;
        break;
    case '>':
        x++;
        break;
    case 'v':
        y--;
        break;
    case '<':
        x--;
        break;
    }

    if(x > x_max)
        x_max = x;
    if(x < x_min)
        x_min = x;
    if(y > y_max)
        y_max = y;
    if(y < y_min)
        y_min = y;
}

int w = x_max - x_min + 1;
int h = y_max - y_min + 1;
char grid[w][h];
memset(grid, 0, width * height);

x = abs(x_min);
y = abs(y_min);

rewind(input);
grid[x][y] = 1;
while(fread(&c, 1, 1, input)) {
    switch(c) {
    case '^':
        y++;
        break;
    case '>':
        x++;
        break;
    case 'v':
        y--;
        break;
    case '<':
        x--;
        break;
    }
    grid[x][y]++;
}
{{< /highlight >}}

### Optimization
The problem is simple enough that optimization opportunities are few. One possible optimization, depending on implementation, would be to read the whole input file into memory and iterate over the memory buffer rather than reading byte-by-byte from a file stream.

---

Then we discover that apparently one Santa is not enough:

> The next year, to speed up the process, Santa creates a robot version of himself, __Robo-Santa__, to deliver presents with him.
> 
> Santa and Robo-Santa start at the same location (delivering two presents to the same starting house), then take turns moving based on instructions from the elf, who is eggnoggedly reading from the same script as the previous year.
> 
> This year, how many houses receive at __least one present__?
> 
> For example:
> 
> * `^v` delivers presents to `3` houses, because Santa goes north, and then Robo-Santa goes south.
> * `^>v<` now delivers presents to `3` houses, and Santa and Robo-Santa end up back where they started.
> * `^v^v^v^v^v` now delivers presents to `11` houses, with Santa going one direction and Robo-Santa going the other.

## Analysis

The overall problem is similar, the difference being that we now need to track two cursors instead of just one, and alternate/toggle between them. (There is still only a single grid, therefore a single min/max for each dimension when we are dimensioning the grid.) This does not increase the complexity or difficulty of the problem appreciably.

### C
{{< highlight c >}}
FILE *input = fopen(inputFile);

char c;
int s = 0;
int xs[2] = {0};
int ys[2] = {0};
int x_min = 0, x_max = 0, y_min = 0, y_max = 0;
while(fread(&c, 1, 1, input)) {
    switch(c) {
    case '^':
        ys[s]++;
        break;
    case '>':
        xs[s]++;
        break;
    case 'v':
        ys[s]--;
        break;
    case '<':
        xs[s]--;
        break;
    }

    if(xs[s] > x_max)
        x_max = xs[s];
    if(xs[s] < x_min)
        x_min = xs[s];
    if(ys[s] > y_max)
        y_max = ys[s];
    if(ys[s] < y_min)
        y_min = ys[s];

    s = !s
}

int w = x_max - x_min + 1;
int h = y_max - y_min + 1;
char grid[w][h];
memset(grid, 0, width * height);

x = abs(x_min);
y = abs(y_min);

rewind(input);
grid[x][y] = 2;
s = 0;
while(fread(&c, 1, 1, input)) {
    switch(c) {
    case '^':
        ys[s]++;
        break;
    case '>':
        xs[s]++;
        break;
    case 'v':
        ys[s]--;
        break;
    case '<':
        xs[s]--;
        break;
    }
    grid[xs[s]][ys[s]]++;
    s = !s
}
{{< /highlight >}}

Next: [Advent of Code 2015, Day 4: The Ideal Stocking Stuffer]({{< relref "advent-2015-4.md" >}})