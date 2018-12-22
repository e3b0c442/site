+++
title = "Advent of Code 2015, Day 1: Not Quite Lisp"
date = 2018-12-21T09:42:00-06:00
tags = ["Advent of Code", "Advent of Code 2015"]
markup = "markdown"
+++

Advent of Code debuted on December 1, 2015, with this puzzle:

> Santa is trying to deliver presents in a large apartment building, but he can't find the right floor - the directions he got are a little confusing. He starts on the ground floor (floor `0`) and then follows the instructions one character at a time.
>
> An opening parenthesis, `(`, means he should go up one floor, and a closing parenthesis, `)`, means he should go down one floor.
>
> The apartment building is very tall, and the basement is very deep; he will never find the top or bottom floors.
>
> For example:
>
> - `(())` and `()()` both result in floor `0`.
> - `(((` and `(()(()(` both result in floor `3`.
> - `))(((((` also results in floor `3`.
> - `())` and `))(` both result in floor `-1` (the first basement level).
> - `)))` and `)())())` both result in floor `-3`.
>
> To __what floor__ do the instructions take Santa?

## Analysis

<!--more-->
As can be expected for a "warm-up" puzzle this one is fairly straightforward.
Each byte in the input must result in a calculation being performed, making the
complexity *O(n)*.

The easiest way to accomplish this is to read the input stream one byte at a
time and increment or decerement the counter.

### C
{{< highlight c >}}
FILE *input = fopen(inputFile);
int floor = 0;
char c;

while(fread(&c, 1, 1, input) > 0) {
    switch(c) {
    case '(':
        floor++;
        break;
    case ')':
        floor--;
        break;
    }
}
{{< /highlight >}}

### Optimization
In order to optimize, depending on the implementation it may be faster to read
the whole file first and then loop over the in-memory array. This does not
change the asymptotic complexity.

---

Upon submitting the correct value, a second problem appears:

> Now, given the same instructions, find the __position__ of the first character that causes him to enter the basement (floor `-1`). The first character in the instructions has position `1`, the second character has position `2`, and so on.
> 
> For example:
> 
> * `)` causes him to enter the basement at character position `1`.
> * `()())` causes him to enter the basement at character position `5`.
> 
> What is the __position__ of the character that causes Santa to first enter the basement?

## Analysis

Now, in addition to tracking which floor we are on, we need to track how many
steps we've taken. The original code can be modified to find both answers,
however for the sake of clarity I will treat both problems of a given day as
separate.

Again, the input must be looped, this time with the addition of incrementing a
counter. This time, however, we can break out of the loop once we achieve the 
desired floor, saving some time.

### C
{{< highlight c >}}
FILE *input = fopen(inputFile);
int floor = 0;
char c;
int pos;

while(fread(&c, 1, 1, input) > 0) {
    pos++;
    switch(c) {
    case '(':
        floor++;
        break;
    case ')':
        floor--;
        break;
    }
    if(floor < 0) {
        break;
    }
}
{{< /highlight >}}

### Optimization

Optimization may again be achieved by reading the whole file, although the 
picture here is somewhat murkier as this problem does not necessarily require
us to loop through the whole array. Regardless, the complexity remains at _O(n)_.

Next: [Advent of Code 2015, Day 2: I Was Told There Would Be No Math]({{< relref "advent-2015-2.md" >}})