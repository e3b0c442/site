---
date: 2019-09-09T00:00:00Z
title: "Advent of Code 2015 Day 1: Not Quite Lisp"
tags: ["Programming", "Advent of Code", "C"]
---

Yesterday, I talked in general about what I am trying to accomplish with this
series, and wrote a reusable function to read the input files into memory, which
will be used frequently throughout the series. Now it's time to dig in.

> Santa was hoping for a white Christmas, but his weather machine's "snow" function is powered by stars, and he's fresh out! To save Christmas, he needs you to collect fifty stars by December 25th.
> 
> Collect stars by helping Santa solve puzzles. Two puzzles will be made available on each day in the advent calendar; the second puzzle is unlocked when you complete the first. Each puzzle grants one star. Good luck!
> 
> Here's an easy puzzle to warm you up.
> 
> Santa is trying to deliver presents in a large apartment building, but he can't find the right floor - the directions he got are a little confusing. He starts on the ground floor (floor 0) and then follows the instructions one character at a time.
> 
> An opening parenthesis, `(`, means he should go up one floor, and a closing parenthesis, `)`, means he should go down one floor.
> 
> The apartment building is very tall, and the basement is very deep; he will never find the top or bottom  floors.
> 
> For example:
> 
> * `(())` and `()()` both result in floor `0`.
> * `(((` and `(()(()(` both result in floor `3`.
> * `))(((((` also results in floor `3`.
> * `())` and `))(` both result in floor `-1` (the first basement level).
> * `)))` and `)())())` both result in floor `-3`.
> 
> To **what floor** do the instructions take Santa?

Before we start writing code, let's think about what needs to happen. We will
need to input a file, read through it, determine which characters are important
(`(` and `)`) and adjust the floor number accordingly. Then we'll need to output
our solution somehow.

Generally speaking, each day's puzzle will be its own executable and the input
filename will be a command-line argument.

Now that we have a plan of attack, let's dig in.

```c
int main(int argc, char const *argv[])
{
    /* check that filename was provided */
    if (argc < 2)
    {
        fprintf(stderr, "Must provide filename of input file\n");
        return -1;
    }
```

First things first, let's check and make sure the user actually provided a
filename. Don't forget that `argv[0]` is always the name of the invoked command,
so `argc` must be at least `2` if the user has provided any arguments at all. If
the argument is not present, we'll print a useful error message to `stderr` and
return from `main` with a non-zero code.

```c
    /* read the file into memory */
    char *input;
    int filesize = read_file_to_buffer(&input, argv[1]);
    if (filesize < 0)
    {
        fprintf(stderr, "File read failed: %s\n", strerror(errno));
        return -1;
    }

    printf("Day 1:\n");
```

Remember that input file function we wrote yesterday? Now we get to use it. We 
declare a pointer to the buffer and pass it into the function, along with the
filename which should be in `argv[1]`. If the file doesn't exist, isn't
readable, or any other problem that we handled in the function, it will return
`-1` and `errno` will be set. In that case, we print the error message to
`stderr` and return non-zero. If everything worked, the contents of the file
are now in the location in memory pointed to by our `input` pointer. Once we've
successfully read the file, we'll print our header line for the output.


Now it's time to solve the problem. The most straightforward way is going to be
to iterate over the contents of the file in memory byte-by-byte and adjust the
floor accordingly.

```c
    /* solve the first puzzle */
    int floor = 0;
    for (int i = 0; i < filesize; i++)
    {
        switch (input[i])
        {
        case '(':
            floor++;
            break;
        case ')':
            floor--;
            break;
        default:;
        }
    }

    printf("\tSolution 1: %d\n", floor);
```
We initialize our floor variable to zero as noted in the problem description
(`He starts on the ground floor (floor 0)...`), and then construct a `for` loop
with the index starting at `0` (remember, C arrays are zero-indexed) and
incrementing by one until we reach the file size which was returned by our file
input function. During each iteration of the loop, we check the byte at that
index and either increment or decrement the floor number depending on the
character. In this case, I have chosen to ignore any spurious characters that
may be in the input.

The object of the problem is to determine which floor we arrive at (`To what 
floor do the instructions take Santa?`), so the value of the `floor` variable
is our solution.

Once we input our solution, a second problem appears!

> Now, given the same instructions, find the **position** of the first character that causes him to enter the basement (floor -1). The first character in the instructions has position 1, the second character has position 2, and so on.
> 
> For example:
>
> * `)` causes him to enter the basement at character position `1`.
> * `()())` causes him to enter the basement at character position `5`.
> 
> What is the **position** of the character that causes Santa to first enter the basement?

In the interest of saving compute time and code, we will attempt to integrate
the finding of the second solution into the original code. In this case, it's
a matter of finding the index where we first hit floor `-1`. We can do this by
setting a variable to a known bad value, and checking whether it has been reset
to a known good value and if not, setting it. So, we'll update our problem logic
as such:

```c
    /* solve the puzzles */
    int floor = 0;
    int basement = -1;
    for (int i = 0; i < filesize; i++)
    {
        switch (input[i])
        {
        case '(':
            floor++;
            break;
        case ')':
            floor--;
            break;
        default:;
        }
        if (floor == -1 && basement == -1)
        {
            basement = i + 1;
        }
    }

    printf("\tSolution 1: %d\n", floor);
    printf("\tSolution 2: %d\n", basement);
```

In this updated logic, we set `basement` to a sentinel value of `-1`; since we
are counting up, the value cannot possibly be negative. Then, in each iteration
of the loop, we check for the floor to be `-1` (since we can only move one
floor at a time) and for the basement value to be the sentinel. For those of
you counting instructions, notice that the floor check will short-circuit; i.e.
the basement check will not happen unless the floor is `-1`. The net effect is
that on average we are only adding one instruction per loop. Also notice that
we set the value of basement to `i + 1` as the positions are one-indexed per
the problem parameters (`The first character in the instructions has position 
1...`).

```c

    free(input);
    return 0;
}
```

Now that we have solved the problem, we'll clean up and return `0`. Note that it
is not strictly necessary to free `input`, but it is a good habit to be in.

Here's the whole program for day 1:
```c
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "common.h"

int main(int argc, char const *argv[])
{
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

    printf("Day 1:\n");

    /* solve the puzzles */
    int floor = 0;
    int basement = 0;
    for (int i = 0; i < filesize; i++)
    {
        switch (input[i])
        {
        case '(':
            floor++;
            break;
        case ')':
            floor--;
            break;
        default:;
        }
        if (floor == -1 && basement == 0)
        {
            basement = i + 1;
        }
    }

    printf("\tSolution 1: %d\n", floor);
    printf("\tSolution 2: %d\n", basement);

    free(input);
    return 0;
}
```

This program makes proper error checks and manages memory properly, and will
pass all Valgrind checks in a variety of test scenarios.

If I've done something horribly wrong or you have questions, please comment!