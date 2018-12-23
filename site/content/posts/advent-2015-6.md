+++
title = "Advent of Code 2015, Day 6: Probably a Fire Hazard"
date = 2018-12-23T13:07:00-06:00
tags = ["Advent of Code", "Advent of Code 2015"]
markup = "markdown"
+++

Maybe those naughty and nice strings were encrypted... anyway, Santa discovered
YouTube and finally understands what you're asking for for Christmas.

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
> - `turn on 0,0 through 999,999` would turn on (or leave on) every light.
> - `toggle 0,0 through 999,0` would toggle the first line of 1000 lights, turning off the ones that were on, and turning on the ones that were off.
> - `turn off 499,499 through 500,500` would turn off (or leave off) the middle four lights.
>
> After following the instructions, **how many lights are lit**?

## Analysis

<!--more-->

This is a relatively simple problem where we are given a list of
human-formatted instructions that need to be parsed and executed. Our general
flow for this problem will be:

- Read an instruction
- Parse the instruction
- Execute the parsed instruction

Parsing the instruction is the most complex part of this problem, and like the
last one, we can choose to either manually parse the commands, or use regular
expressions. The complexity of manual parsing is increased by the fact that
some commands have a space in them, which requires extra logic around
tokenization. Given that and the fact that we just warmed ourselves up with
regular expressions, we will continue to use them in this case to parse the
command.

Before we start reading the commands, we need to set ourselves up by
initializing the 2D array to store the lights, and by compiling our regular
expression. Compared to day 5, our regular expression is relatively simple.

We need to capture the command, and each of the dimensions, which means we need
five matching groups. The commands can be `turn on`, `turn off`, or `toggle`,
which we can construct into one matching group as `(turn on|turn off|toggle)`.
Each of the dimensions is 0-999, so we know there will be 1-3 digits, which we
can build into a matching group as `(\d{1,3})`. From there, we just need to
build the pattern up to match the command format, ending up with a pattern of
`(turn on|turn off|toggle) (\d{1,3}),(\d{1,3}) through (\d{1,3}),(\d{1,3})`.

We can now begin reading the instructions. When we apply the pattern to the
command, we should get six matches; one for the whole command, one for the
command invocation, and then four corresponding to the starting x, starting y,
ending x, and ending y coordinates. The coordinates can be converted to numbers,
at which point we can set up a loop to do the command action on each cell.

When the command loop is finished, loop over the light grid and count the
lit cells to arrive at the solution.

Because we do not have nested loops on the input, our solution has an asymptotic
complexity of _O(n)_.

### C

{{< highlight c >}}
#include <pcre.h>
FILE *input = fopen(input_file, "r");

const char *pcre_err;
int pcre_erroffset;

char *pattern = "^(turn off|turn on|toggle) (\\d{1,3}),(\\d{1,3}) through (\\d{1,3}),(\\d{1,3})$";
pcre *re = pcre_compile(pattern, 0, &pcre_err, &pcre_erroffset, NULL);

char lights[1000][1000];
memset(lights, 0, 1000 * 1000);

char buf[64];
int ovector[18];
while (fgets(buf, 64, input) != NULL)
{
    int rc = pcre_exec(re, NULL, buf, strlen(buf), 0, 0, ovector, 18);

    const char *command, *x_min_s, *x_max_s, *y_min_s, *y_max_s;
    pcre_get_substring(buf, ovector, rc, 1, &command);
    pcre_get_substring(buf, ovector, rc, 2, &x_min_s);
    pcre_get_substring(buf, ovector, rc, 3, &y_min_s);
    pcre_get_substring(buf, ovector, rc, 4, &x_max_s);
    pcre_get_substring(buf, ovector, rc, 5, &y_max_s);

    int x_min = atoi(x_min_s);
    int y_min = atoi(y_min_s);
    int x_max = atoi(x_max_s);
    int y_max = atoi(y_max_s);

    int cmd;
    if (!strcmp(command, "turn on"))
        cmd = 0;
    if (!strcmp(command, "turn off"))
        cmd = 1;
    if (!strcmp(command, "toggle"))
        cmd = 2;

    int i, j;
    for (i = x_min; i <= x_max; i++)
    {
        for (j = y_min; j <= y_max; j++)
        {
            switch (cmd)
            {
            case 0:
                lights[i][j] = 1;
                break;
            case 1:
                lights[i][j] = 0;
                break;
            case 2:
                lights[i][j] = !lights[i][j];
                break;
            }
        }
    }
}

int i, j;
int count = 0;
for (i = 0; i < 1000; i++)
{
    for (j = 0; j < 1000; j++)
    {
        count += lights[i][j];
    }
}
{{< /highlight >}}

### Optimization

Given the relative simplicity of the text vs day 5, manually parsing may be
faster than using regular expressions. We want to make sure that we are not
doing any more string comparisons than we need to either, so we want to make
sure we map the string values to an integral value for fast comparisons when we
loop through the cells.

---

You're wondering why Santa's instructions don't touch on brightness... then 
you read again.

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
> What is the total brightness of all lights combined after following Santa's instructions?
> 
> For example:
> 
> * `turn on 0,0 through 0,0` would increase the total brightness by 1.
> * `toggle 0,0 through 999,999` would increase the total brightness by 2000000.

## Analysis

The only thing that needs to change in this program is the action taken for a
given command invocation. In addition, if we had chosen to use a boolean grid
for part one, that would need to be changed to something that can hold more
values. 

Next: _coming soon_
