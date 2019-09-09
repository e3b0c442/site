---
date: 2019-09-09T02:00:00Z
title: "Advent of Code 2015 Day 2: I Was Told There Would Be No Math"
tags: ["Programming", "Advent of Code", "C", "String parsing"]
---

I'm going to go ahead and do a second puzzle today since the early ones are
relatively simple. Day 1's puzzles were mostly incrementing, decrementing and
tracking variables; Day 2's have a bit more math involved.

> The elves are running low on wrapping paper, and so they need to submit an order for more. They have a list of the dimensions (length l, width w, and height h) of each present, and only want to order exactly as much as they need.
> 
> Fortunately, every present is a box (a perfect right rectangular prism), which makes calculating the required wrapping paper for each gift a little easier: find the surface area of the box, which is `2*l*w + 2*w*h + 2*h*l`. The elves also need a little extra paper for each present: the area of the smallest side.
> 
> For example:
> 
> * A present with dimensions `2x3x4` requires `2*6 + 2*12 + 2*8 = 52` square feet of wrapping paper plus `6` square feet of slack, for a total of `58` square feet.
> * A present with dimensions `1x1x10` requires `2*1 + 2*10 + 2*10 = 42` square feet of wrapping paper plus `1` square foot of slack, for a total of `43` square feet.
> 
> All numbers in the elves' list are in feet. How many total **square feet of wrapping paper** should they order?

As with day 1, we're going to take a minute to break the problem down. As the
problem states, the input is formatted as XXxYYxZZ, so we will need to parse
the input into usable numbers. Since we need to operate on the smallest
dimensions for the slack, we'll need to be able to sort the inputted values.
Finally, we'll need to do the math and keep a running total.

Let's dig in:

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

The preamble to our function is identical to day 1, with the exception of 
initializing our return value early as we'll be using our `goto` short circuit
pattern to handle some later potential errors. We check that the user provided a
filename and read the file into the `input` buffer.

```c
    printf("Day 2:\n");

    int paper = 0;
```

We'll then print our daily header and initialize the running total for the
wrapping paper.

```c
    /* tokenize the data into lines */
    char *cursor = input;
    char *line = strsep(&cursor, "\n");
```

Now we start getting into some more interesting string handling. Note that the
input has one present per line, so we need to effectively split the string on
the lines. We will use the POSIX `strsep` function for this. There is a similar
`strtok` function in the C standard librory, but it has the downside of being
non-reentrant. If portability is a concern, you may want to stick use `strtok`,
however `strsep` is widely supported and should  be used instead of `strtok` 
wherever possible. 

In order to avoid changing the original value of the pointer to the input, we 
make a copy into `cursor` which becomes the moving pointer passed as the first 
argument of `strsep`. Finally, we are splitting on the newline character `\n`. 
We will not bother to make a copy of the input buffer as we only need to parse 
it once.

```c
    while (line != NULL)
    {
        if (strlen(line) == 0)
            goto tokenize;
```

One side effect of `strsep` that one needs to be aware of is that if multiple
delimiters are encountered in a row, `strsep` will return a pointer to an empty
string instead of consuming all of the delimiters at once. We check for the
empty string and short-circuit the loop if we encounter this.

```c
        /* scan each line's values */
        int dims[3] = {0};

        int scanned = sscanf(line, "%dx%dx%d", &dims[0], &dims[1], &dims[2]);
        if (scanned != 3)
        {
            fprintf(stderr, "Invalid input line \"%s\"\n", line);
            goto err_cleanup;
        }
```

We now initialize an array of three `int`s to hold the parsed values. We use an
array instead of separate variables because we will need to sort the result.
Once the array is initialized, we use the `sscanf` function to scan the 
dimensions into the array. As we are not trying to scan _into_ string variables,
`sscanf` is perfectly safe for our usage. As an error check, we will
short-circuit to the function cleanup if we have an invalid line which scans
less than three values.

```c
        qsort(dims, 3, sizeof(int), cmp_int_asc);
```

Once the values are scanned, they need to be sorted so that we can find the
slack using the smallest side. For this we use the standard library function
`qsort` with a simple comparator function:

```c
int cmp_int_asc(const void *a, const void *b)
{
    int l = *(const int *)a;
    int r = *(const int *)b;

    return l < r ? -1 : l > r ? 1 : 0;
}
```

The comparator dereferences and casts the inputs and then returns `-1` if `a < 
b`, `1` if `a > b`, or `0` if they are equal. I use the ternary operator here
for conciseness, but I would refrain from using it for anything more complex.

```c
        paper += 2 * dims[0] * dims[1] + 2 * dims[1] * dims[2] + 2 * dims[2] * dims[0] + dims[0] * dims[1];
    tokenize:
        line = strsep(&cursor, "\n");
    }
```

Now that we have our values and we have them sorted by smallest to largest, we
just need to do the math and update our running total. As noted in the problem,
for each present we need to add the area of each side plus the area of the
smallest side as slack: `2xy + 2yz + 2xz + xy`.

```c
    printf("\tSolution 1: %d\n", paper);

    goto cleanup;

err_cleanup:
    rval = 1;
cleanup:
    free(input);
    return rval;
}
```

Once we have broken out of the input parsing loop, `paper` should have the total
required wrapping paper. We can print the solution and then proceed with
cleanup. As noted in day 1, freeing `input` is not strictly necessary in this
case but is good from a habit-forming perspective.

Once the solution has been input, the second problem is presented:

> The elves are also running low on ribbon. Ribbon is all the same width, so they only have to worry about the length they need to order, which they would again like to be exact.
> 
> The ribbon required to wrap a present is the shortest distance around its sides, or the smallest perimeter of any one face. Each present also requires a bow made out of ribbon as well; the feet of ribbon required for the perfect bow is equal to the cubic feet of volume of the present. Don't ask how they tie the bow, though; they'll never tell.
> 
> For example:
> 
> * A present with dimensions `2x3x4` requires `2+2+3+3 = 10` feet of ribbon to wrap the present plus `2*3*4 = 24` feet of ribbon for the bow, for a total of `34` feet.
> * A present with dimensions `1x1x10` requires `1+1+1+1 = 4` feet of ribbon to wrap the present plus `1*1*10 = 10` feet of ribbon for the bow, for a total of `14` feet.
> 
> How many total **feet of ribbon** should they order?

As with day 1, we already have all of the infrastructure present to handle this
new problem. We'll add a new variable to track the running total of ribbon:

```c
    int paper = 0;
    int ribbon = 0;
```

Then update the total ribbon on each loop, following the instructions to add
`2x+2y+xyz` to the total:

```c
        paper += 2 * dims[0] * dims[1] + 2 * dims[1] * dims[2] + 2 * dims[2] * dims[0] + dims[0] * dims[1];
        ribbon += 2 * dims[0] + 2 * dims[1] + dims[0] * dims[1] * dims[2];
```

Finally, we'll print the second solution:

```c

    printf("\tSolution 1: %d\n", paper);
    printf("\tSolution 2: %d\n", ribbon);
```

When it's all put together, we have:

```c
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
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

    printf("Day 2:\n");

    int paper = 0;
    int ribbon = 0;

    /* tokenize the data into lines */
    char *cursor = input;
    char *line = strsep(&cursor, "\n");
    while (line != NULL)
    {
        if (strlen(line) == 0)
            goto tokenize;

        /* scan each line's values */
        int dims[3] = {0};

        int scanned = sscanf(line, "%dx%dx%d", &dims[0], &dims[1], &dims[2]);
        if (scanned != 3)
        {
            fprintf(stderr, "Invalid input line \"%s\"\n", line);
            goto err_cleanup;
        }

        qsort(dims, 3, sizeof(int), cmp_int_asc);

        paper += 2 * dims[0] * dims[1] + 2 * dims[1] * dims[2] + 2 * dims[2] * dims[0] + dims[0] * dims[1];
        ribbon += 2 * dims[0] + 2 * dims[1] + dims[0] * dims[1] * dims[2];
    tokenize:
        line = strsep(&cursor, "\n");
    }

    printf("\tSolution 1: %d\n", paper);
    printf("\tSolution 2: %d\n", ribbon);

    goto cleanup;

err_cleanup:
    rval = 1;
cleanup:
    free(input);
    return rval;
}
```

As before, this program will pass all Valgrind checks and does not leak memory.

Suggestions, questions, and criticism welcome! Comments can be added below.