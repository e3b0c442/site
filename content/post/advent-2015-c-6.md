---
date: 2019-09-17T00:00:00Z
title: "Advent of Code 2015 Day 6: Probably a Fire Hazard"
tags: ["Programming", "Advent of Code", "C", "Regular Expressions"]
---

Welcome back to my Advent of Code series. I hope you're enjoying exploring these
problems in depth in C. Let's take a look at the problem for day 6:

> Because your neighbors keep defeating you in the holiday house decorating contest year after year, you've decided to deploy one million lights in a 1000x1000 grid.
> 
> Furthermore, because you've been especially nice this year, Santa has mailed you instructions on how to display the ideal lighting configuration.
> 
> Lights in your grid are numbered from 0 to 999 in each direction; the lights at each corner are at `0,0`, `0,999`, `999,999`, and `999,0`. The instructions include whether to `turn on`, `turn off`, or `toggle` various inclusive ranges given as coordinate pairs. Each coordinate pair represents opposite corners of a rectangle, inclusive; a coordinate pair like `0,0 through 2,2` therefore refers to 9 lights in a 3x3 square. The lights all start turned off.
> 
> To defeat your neighbors this year, all you have to do is set up your lights by doing the instructions Santa sent you in order.
> 
> For example:
> 
> * `turn on 0,0 through 999,999` would turn on (or leave on) every light.
> * `toggle 0,0 through 999,0` would toggle the first line of 1000 lights, turning off the ones that were on, and turning on the ones that were off.
> * `turn off 499,499 through 500,500` would turn off (or leave off) the middle four lights.
> * After following the instructions, **how many lights are lit**?

Let's look at our constituent problems:

* Read the input file into memory
* Set up a grid
* Parse the input lines
* Follow the input instructions

We already know how to read our file into memory, so we'll move onto setting up
the grid. As with [Day 3]({{< relref "advent-2015-c-3.md" >}}), we will use a 
two-dimensional array to represent the grid. Unlike day 3, we know in advance
what the size of the grid is, so we don't need to do an extra loop to size it;
we can just declare `bool grid[1000][1000]`, initialize, and move on.

Likewise, parsing the input lines is also a situation we have previously 
encountered. In previous days, we have used [scanners]({{< relref 
"advent-2015-c-2.md" >}}) and [regular expressions]({{< relref 
"advent-2015-c-5.md" >}}). While using scanners is ideal for simple situations,
we have to remember the downsides: scanners are tokenized by any whitespace, and
scanners cannot safely be used to scan into strings. Both of these cases are 
disqualifiers for our input; we have a component of the input that might have a
space in it, and we also need to scan into a string. For this reason, we'll use
regular expressions to parse our input. Because we can only parse into strings
with regular expressions, we will also need to convert our number strings to
numerical values.

Once we have our values, we can use them to set up a nested loop to update the
values on the grid.

Let's look at some code:
```c
int main(int argc, char const *argv[])
{
    int rc = 0;
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

    printf("Day 6:\n");
```

This is our now familiar check for argument, read input, and print preamble,
included only for completeness at this point.

```c
    regex_t inst_r;
    int result = regcomp(&inst_r, "\\(turn on\\|turn off\\|toggle\\) \\([[:digit:]]\\{1,\\}\\),\\([[:digit:]]\\{1,\\}\\) through \\([[:digit:]]\\{1,\\}\\),\\([[:digit:]]\\{1,\\}\\)", 0);
    if (result != 0)
        goto err_cleanup;
```

Next, we compile the regular expression to match against. Without the escape
characters, this expression looks like: `(turn on|turn off|toggle) ([[:digit:]]{1,}),([[:digit:]]{1,}) through ([[:digit:]]{1,}),([[:digit:]]{1,})`. 
Reading this expression aloud, we get _Capture a group containing "turn on",
"turn off", or "toggle". Find a space. Capture a group of one or more digits,
then find a comma, then capture a group of one or more digits. Find the phrase
" through ". Capture a group of one or more digits, then find a comma, then 
capture a group of one or more digits._ At the end of it, we have captured five submatches; one for the command, and then one each for the low and high X and Y coordinates.

```c
    char *cursor = input;
    char *line = strsep(&cursor, "\n");
    regmatch_t matches[6] = {0};
    bool grid[1000][1000];
    memset(grid, 0, 1000 * 1000 * sizeof(bool));
```

We now prepare to start our loop. We create the cursor variable and get the
first line, declare an array to hold the regular expression match coordinates
and initialize it, and declare and initialize the grid array. Note that we use
`memset` for the grid array because the `{0}` initialization shorthand cannot be
used with multidimensional arrays. Also note that we declared a six member array
for the regular expression matches; the first match is always the full matched
string, so we need room for this plus the five submatches that we actually care
about.

```c
    while (line != NULL)
    {
        if (strlen(line) == 0)
            goto tokenize;

        memset(matches, 0, 6 * sizeof(regmatch_t));
        result = regexec(&inst_r, line, 6, matches, 0);
        if (result != 0)
            goto err_cleanup;

        char instr[9] = {0};
        char oxs[4] = {0};
        char oys[4] = {0};
        char dxs[4] = {0};
        char dys[4] = {0};

        strncpy(instr, line + matches[1].rm_so, matches[1].rm_eo - matches[1].rm_so);
        strncpy(oxs, line + matches[2].rm_so, matches[2].rm_eo - matches[2].rm_so);
        strncpy(oys, line + matches[3].rm_so, matches[3].rm_eo - matches[3].rm_so);
        strncpy(dxs, line + matches[4].rm_so, matches[4].rm_eo - matches[4].rm_so);
        strncpy(dys, line + matches[5].rm_so, matches[5].rm_eo - matches[5].rm_so);

        unsigned long ox = strtoul(oxs, NULL, 10);
        if (ox == 0 && errno == EINVAL)
            goto err_cleanup;
        unsigned long oy = strtoul(oys, NULL, 10);
        if (oy == 0 && errno == EINVAL)
            goto err_cleanup;
        unsigned long dx = strtoul(dxs, NULL, 10);
        if (dx == 0 && errno == EINVAL)
            goto err_cleanup;
        unsigned long dy = strtoul(dys, NULL, 10);
        if (dy == 0 && errno == EINVAL)
            goto err_cleanup;

        if (strcmp("turn on", instr) == 0)
        {
            for (int x = ox; x <= dx; x++)
                for (int y = oy; y <= dy; y++)
                    grid[x][y] = true;
        }
        else if (strcmp("turn off", instr) == 0)
        {
            for (int x = ox; x <= dx; x++)
                for (int y = oy; y <= dy; y++)
                    grid[x][y] = false;
        }
        else if (strcmp("toggle", instr) == 0)
        {
            for (int x = ox; x <= dx; x++)
                for (int y = oy; y <= dy; y++)
                    grid[x][y] = !grid[x][y];
        }
        else
        {
            goto err_cleanup;
        }

    tokenize:
        line = strsep(&cursor, "\n");
    }
```

Now, we perform our loop on each of the input lines, parsing and following the
directions. We'll zero out the match array, then execute the compiled regular
expression against the input line. We then initialize destination variables for
each of the five submatch values, and use the match coordinates to copy the 
matched substrings into the destination variables.

Once the destination variables are populated, we'll use `strtoul` to parse the
four numerical values into integral equivalents, giving us the final set of
parsed values to handle our input.

At this point, we use strcmp to compare the instruction string against the known
instructions, and then set up a nested loop using the X and Y values. Finally,
the interior of the loop sets the value of that coordinate per the instruction.
You'll notice that we parsed the instruction first, requiring the loop to be
specified multiple times, seemingly breaking the _don't-repeat-yourself_ rule.
In this case, however, this is justified, as if we reversed things, the string
comparison would need to be done for each position, which would be prohibitvely
computationally expensive. Once the loop is complete, the values of the grid
are set.

```c
    int on = 0;
    for (int x = 0; x < 1000; x++)
        for (int y = 0; y < 1000; y++)
            if (grid[x][y])
                on++;

    printf("\tSolution 1: %d\n", on);
```

In order to get our solution, we set up a counter variable and then another
nested loop over the entire two-dimensional surface. Each time we encounter
a `true` value, we increment the counter. We then print our solution.

After submitting the solution, the second problem is presented:

> You just finish implementing your winning light pattern when you realize you mistranslated Santa's message from Ancient Nordic Elvish.
> 
> The light grid you bought actually has individual brightness controls; each light can have a brightness of zero or more. The lights all start at zero.
> 
> The phrase `turn on` actually means that you should increase the brightness of those lights by `1`.
> 
> The phrase `turn off` actually means that you should decrease the brightness of those lights by `1`, to a minimum of zero.
> 
> The phrase `toggle` actually means that you should increase the brightness of those lights by `2`.
> 
> What is the **total brightness** of all lights combined after following Santa's instructions?
> 
> For example:
> 
> * `turn on 0,0 through 0,0` would increase the total brightness by 1.
> * `toggle 0,0 through 999,999` would increase the total brightness by 2000000.

As with previous problems, this one is simalar, and we can implement it in the
existing loop to avoid the need to loop the input a second time.

```c
    char *cursor = input;
    char *line = strsep(&cursor, "\n");
    regmatch_t matches[6] = {0};
    bool grid[1000][1000];
    int grid2[1000][1000];
    memset(grid, 0, 1000 * 1000 * sizeof(bool));
    memset(grid2, 0, 1000 * 1000 * sizeof(int));
```

We will declare a second grid variable, this time of type `int[][]`, since we
need to track more than just on or off. 

```c
        if (strcmp("turn on", instr) == 0)
        {
            for (int x = ox; x <= dx; x++)
                for (int y = oy; y <= dy; y++)
                {
                    grid[x][y] = true;
                    grid2[x][y]++;
                }
        }
        else if (strcmp("turn off", instr) == 0)
        {
            for (int x = ox; x <= dx; x++)
                for (int y = oy; y <= dy; y++)
                {
                    grid[x][y] = false;
                    grid2[x][y] = grid2[x][y] > 0 ? grid2[x][y] - 1 : 0;
                }
        }
        else if (strcmp("toggle", instr) == 0)
        {
            for (int x = ox; x <= dx; x++)
                for (int y = oy; y <= dy; y++)
                {
                    grid[x][y] = !grid[x][y];
                    grid2[x][y] += 2;
                }
        }
        else
        {
            goto err_cleanup;
        }
```

The instructions are equivalent for both loops; we just need to add the
application of the instructions to the second grid following the second
transation's directions.

```c
    int on = 0;
    int bright = 0;
    for (int x = 0; x < 1000; x++)
        for (int y = 0; y < 1000; y++)
        {
            if (grid[x][y])
                on++;
            bright += grid2[x][y];
        }

    printf("\tSolution 1: %d\n", on);
    printf("\tSolution 2: %d\n", bright);
```

We'll add a second counter variable and perform the addition during the same
nested loop as the first solution, and then print both solutions.

```c

    goto cleanup;

err_cleanup:
    rc = -1;
cleanup:
    regfree(&inst_r);
    free(input);
    return rc;
}
```

Finally, we will clean up. As before, we need to call `regfree` to free the 
compiled regular expression in addition to freeing the input.

Our problems are gradually getting more difficult, but we're almost a week
through! Day seven's solution requires implementing a new data structure, so our
next article will take a break to discuss this in detail.

As always, comments, criticism, and banter are welcome, just leave a comment!