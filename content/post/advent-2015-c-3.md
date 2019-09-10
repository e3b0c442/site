---
date: 2019-09-10T00:00:00Z
title: "Advent of Code 2015 Day 3: Perfectly Spherical Houses in a Vacuum"
tags: ["Programming", "Advent of Code", "C"]
---

You've made it this far, so either nobody's reading this, you're all masochists,
or I'm doing OK. Let's take a look at day 3:

> Santa is delivering presents to an infinite two-dimensional grid of houses.

> He begins by delivering a present to the house at his starting location, and then an elf at the North Pole calls him via radio and tells him where to move next. Moves are always exactly one house to the north (`^`), south (`v`), east (`>`), or west (`<`). After each move, he delivers another present to the house at his new location.
> 
> However, the elf back at the north pole has had a little too much eggnog, and so his directions are a little off, and Santa ends up visiting some houses more than once. How many houses receive **at least one present**?
> 
> For example:
> 
> * `>` delivers presents to `2` houses: one at the starting location, and one to the east.
> * `^>v<` delivers presents to `4` houses in a square, including twice to the house at his starting/ending location.
> * `^v^v^v^v^v` delivers a bunch of presents to some very lucky children at only 2 houses.

This problem seems pretty simple on the face of it, but let's break it down. We
will need to:

* Read the input from the file
* Iterate over the input
* Track which locations have been visited
* Count the visited locations

In order to track the location, we'll need to use a slightly more advanced data
structure. Since we are working with quantities in 2D space, a 2D array makes
sense. Thankfully, in newer versions of the C standard, the array sizes do not
need to be known at compile time, so we can avoid some messy pointer arithmetic.

Let's get to work.

```c
int main(int argc, char const *argv[])
{
    int rval = 0;

    /* check that filename was provided */
    if (argc < 2)
    {
        fprintf(stderr, "Must provide filename of input file\n");
        return -1;
    }

    /* read the file into memory */
    char *input;
    int filesize = read_file_to_buffer(&input, argv[1]);
    if (filesize < 0)
    {
        fprintf(stderr, "File read failed: %s\n", strerror(errno));
        return -1;
    }
```

If this looks familiar, you're catching on. We check that the filename is
provided, read the input file to memory, and handle any errors.

Now we're ready to get started... but are we? We still have no idea how big
our grid array needs to be. We could guess, but we run the danger of either
making the arrays too small and going out-of-bounds, or making them too big and
unnecessarily tying up memory. The better plan would be to iterate over the
input and test the constraints.

```c
    printf("Day 3:\n");

    /* size the grid */
    int x = 0, y = 0, min_x = 0, min_y = 0, max_x = 0, max_y = 0;
    for (int i = 0; i < filesize; i++)
    {
        switch (input[i])
        {
        case '>':
            x++;
            break;
        case 'v':
            y--;
            break;
        case '<':
            x--;
            break;
        case '^':
            y++;
            break;
        default:;
        }

        if (x > max_x)
            max_x = x;
        if (y > max_y)
            max_y = y;
        if (x < min_x)
            min_x = x;
        if (y < min_y)
            min_y = y;
    }

    int w = (max_x - min_x) + 1;
    int h = (max_y - min_y) + 1;
```

After printing our preamble, we'll create tracking variables for the current x
and y positions, and the minimum and maximums of both. We'll initialize all of
them to zero, and then iterate over the input, adjusting the current value of
x and y and keeping the maximums and minimums updated as needed. Once the loop
is complete, we have enough information to compute the sizes of the arrays. Note
that we need to add 1 to the total; e.g. `1 - (-1) = 2`, but that covers the
values `-1, 0, 1`.

```c
    /* create the grid and travel again */
    bool grid[w][h];
    memset(grid, 0, w * h * sizeof(bool));

    x = 0 - min_x;
    y = 0 - min_y;
```

Now that we know the sizes, we can declare the two-dimensional array. Again, 
older versions of the C standard do not allow runtime determination of the array
sizes for stack declarations; if you are stuck using an old version, you will
need to allocate the array on the heap with `malloc` or the like. Since we only
need to track if a house has been visited or not, we'll use the `bool` data type
for our grid. The more astute reader might know that `bool` is still represented
with a full byte in C, but we can still use the type as a hint to those who
might read your code as to the function.

Once we've declared the array, we will need to initialize it; while we can
declare the array in this fashion now, we still cannot initialize it at
declaration. We'll use the `memset` function to zero out the allocated arrays.
This is safe with our `bool` data type as `false` is represented as 0.

Finally, we'll set our initial indices for x and y for the upcoming run. Since
we cannot have negative indices in an array, we'll need to offset them using the
minimum values previously found.

```c
    grid[x][y] = true;
    for (int i = 0; i < filesize; i++)
    {
        switch (input[i])
        {
        case '>':
            x++;
            break;
        case 'v':
            y--;
            break;
        case '<':
            x--;
            break;
        case '^':
            y++;
            break;
        default:;
        }
        grid[x][y] = true;
    }
```

We're finally ready to gather our data. We'll iterate through the input again,
flipping the value on the grid to true as we visit each space. Notice that we
set the origin space to `true` as well per the problem parameters (`He begins by delivering a present to the house at his starting location...`).

```c
    /* count the houses touched */
    int houses = 0;
    for (int i = 0; i < w; i++)
        for (int j = 0; j < h; j++)
            houses += grid[i][j] ? 1 : 0;
    printf("\tSolution 1: %d\n", houses);
```

Once we've completed the loop, we can count the houses by iterating through the
grid and incrementing a counter for each set of coordinates that is true. The
ternary operator is used as a shortcut here.

Once we submit our solution, the second problem appears:

> The next year, to speed up the process, Santa creates a robot version of himself, **Robo-Santa**, to deliver presents with him.
> 
> Santa and Robo-Santa start at the same location (delivering two presents to the same starting house), then take turns moving based on instructions from the elf, who is eggnoggedly reading from the same script as the previous year.
> 
> This year, how many houses receive **at least one present**?
>
> For example:
> 
> * `^v` delivers presents to `3` houses, because Santa goes north, and then Robo-Santa goes south.
> * `^>v<` now delivers presents to `3` houses, and Santa and Robo-Santa end up back where they started.
> * `^v^v^v^v^v` now delivers presents to `11` houses, with Santa going one direction and Robo-Santa going the other.

As we read, the difference here is that we have a second "Santa", and they
alternate directions in the input, rather than one taking all the directions.
This means we will need to implement a toggle, as well as keep two sets of
coordinates on the same grid. As with previous answers, I chose to integrate
this solution into the original problem rather than writing a completely new
set of code to solve problem 2.

```c
    printf("Day 3:\n");

    /* size the grid */
    int x = 0, y = 0, min_x = 0, min_y = 0, max_x = 0, max_y = 0;
    int x2[2] = {0}, y2[2] = {0}, min_x2 = 0, min_y2 = 0, max_x2 = 0, max_y2 = 0;
    int s = 0;
    for (int i = 0; i < filesize; i++)
    {
        switch (input[i])
        {
        case '>':
            x++;
            x2[s]++;
            break;
        case 'v':
            y--;
            y2[s]--;
            break;
        case '<':
            x--;
            x2[s]--;
            break;
        case '^':
            y++;
            y2[s]++;
            break;
        default:;
        }

        if (x > max_x)
            max_x = x;
        if (y > max_y)
            max_y = y;
        if (x < min_x)
            min_x = x;
        if (y < min_y)
            min_y = y;

        if (x2[s] > max_x2)
            max_x2 = x2[s];
        if (y2[s] > max_y2)
            max_y2 = y2[s];
        if (x2[s] < min_x2)
            min_x2 = x2[s];
        if (y2[s] < min_y2)
            min_y2 = y2[s];

        s ^= 1;
    }

    int w = (max_x - min_x) + 1;
    int h = (max_y - min_y) + 1;
    int w2 = (max_x2 - min_x2) + 1;
    int h2 = (max_y2 - min_y2) + 1;
```
In addition to the single `x` and `y` value for the first problem, I declare
a two-element array to represent the coordinates of both santas for the second
problem. I alse declare and initialize another set of min/max variables. 
Finally, a variable is declared to act as a toggle between `0` and `1`.

Now to find our coordinates, we run through the loop. We increment the x and y
of the appropriate Santa, and check that against the x and y minimums and
maximums for the second grid (remember, both Santas share the grid, so there
is only one set of mins and maxes). We'll update the values for the second grid
based on the active Santa, and then once all values have been updated, we use
the bitwise XOR assignment operator to flip the toggle value. Once the loop is
complete, we calculate the width and height of the second grid.

```c

    /* create the grids and travel again */
    bool grid[w][h];
    bool grid2[w2][h2];
    memset(grid, 0, w * h * sizeof(bool));
    memset(grid2, 0, w2 * h2 * sizeof(bool));

    x = 0 - min_x;
    y = 0 - min_y;
    x2[0] = 0 - min_x2;
    x2[1] = 0 - min_x2;
    y2[0] = 0 - min_y2;
    y2[1] = 0 - min_y2;
```

The grid creation for the second problem is identical to the first problem. The
only difference comes in assigning the initial coordinates; this needs to be 
done for both Santas for the second problem.

```c
    s = 0;
    grid[x][y] = true;
    grid[x2[0]][y2[0]] = true;
    for (int i = 0; i < filesize; i++)
    {
        switch (input[i])
        {
        case '>':
            x++;
            x2[s]++;
            break;
        case 'v':
            y--;
            y2[s]--;
            break;
        case '<':
            x--;
            x2[s]--;
            break;
        case '^':
            y++;
            y2[s]++;
            break;
        default:;
        }
        grid[x][y] = true;
        grid2[x2[s]][y2[s]] = true;
        s ^= 1;
    }
```

During the second iteration of the input, we'll toggle the Santa as we did
during the range finding iteration. 

```c
    /* count the houses touched */
    int houses = 0;
    for (int i = 0; i < w; i++)
        for (int j = 0; j < h; j++)
            houses += grid[i][j] ? 1 : 0;
    printf("\tSolution 1: %d\n", houses);

    houses = 0;
    for (int i = 0; i < w2; i++)
        for (int j = 0; j < h2; j++)
            houses += grid2[i][j] ? 1 : 0;
    printf("\tSolution 2: %d\n", houses);
```

We'll count the number of visited houses for the second grid, exactly
as we did the first grid.

Finally, we'll free our allocated input array for completeness and return
success:

```c
    free(input);
    return rval;
}
```

All together now:
```c
#include <errno.h>
#include <stdbool.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include "common.h"

int main(int argc, char const *argv[])
{
    int rval = 0;

    /* check that filename was provided */
    if (argc < 2)
    {
        fprintf(stderr, "Must provide filename of input file\n");
        return -1;
    }

    /* read the file into memory */
    char *input;
    int filesize = read_file_to_buffer(&input, argv[1]);
    if (filesize < 0)
    {
        fprintf(stderr, "File read failed: %s\n", strerror(errno));
        return -1;
    }

    printf("Day 3:\n");

    /* size the grid */
    int x = 0, y = 0, min_x = 0, min_y = 0, max_x = 0, max_y = 0;
    int x2[2] = {0}, y2[2] = {0}, min_x2 = 0, min_y2 = 0, max_x2 = 0, max_y2 = 0;
    int s = 0;
    for (int i = 0; i < filesize; i++)
    {
        switch (input[i])
        {
        case '>':
            x++;
            x2[s]++;
            break;
        case 'v':
            y--;
            y2[s]--;
            break;
        case '<':
            x--;
            x2[s]--;
            break;
        case '^':
            y++;
            y2[s]++;
            break;
        default:;
        }

        if (x > max_x)
            max_x = x;
        if (y > max_y)
            max_y = y;
        if (x < min_x)
            min_x = x;
        if (y < min_y)
            min_y = y;

        if (x2[s] > max_x2)
            max_x2 = x2[s];
        if (y2[s] > max_y2)
            max_y2 = y2[s];
        if (x2[s] < min_x2)
            min_x2 = x2[s];
        if (y2[s] < min_y2)
            min_y2 = y2[s];
        s ^= 1;
    }

    int w = (max_x - min_x) + 1;
    int h = (max_y - min_y) + 1;
    int w2 = (max_x2 - min_x2) + 1;
    int h2 = (max_y2 - min_y2) + 1;

    /* create the grids and travel again */
    bool grid[w][h];
    bool grid2[w2][h2];
    memset(grid, 0, w * h * sizeof(bool));
    memset(grid2, 0, w2 * h2 * sizeof(bool));

    x = 0 - min_x;
    y = 0 - min_y;
    x2[0] = 0 - min_x2;
    x2[1] = 0 - min_x2;
    y2[0] = 0 - min_y2;
    y2[1] = 0 - min_y2;

    s = 0;
    grid[x][y] = true;
    grid[x2[0]][y2[0]] = true;
    for (int i = 0; i < filesize; i++)
    {
        switch (input[i])
        {
        case '>':
            x++;
            x2[s]++;
            break;
        case 'v':
            y--;
            y2[s]--;
            break;
        case '<':
            x--;
            x2[s]--;
            break;
        case '^':
            y++;
            y2[s]++;
            break;
        default:;
        }
        grid[x][y] = true;
        grid2[x2[s]][y2[s]] = true;
        s ^= 1;
    }

    /* count the houses touched */
    int houses = 0;
    for (int i = 0; i < w; i++)
        for (int j = 0; j < h; j++)
            houses += grid[i][j] ? 1 : 0;
    printf("\tSolution 1: %d\n", houses);

    houses = 0;
    for (int i = 0; i < w2; i++)
        for (int j = 0; j < h2; j++)
            houses += grid2[i][j] ? 1 : 0;
    printf("\tSolution 2: %d\n", houses);

    free(input);
    return rval;
}
```