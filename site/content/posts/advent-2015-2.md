+++
title = "Advent of Code 2015, Day 2: I Was Told There Would Be No Math"
date = 2018-12-21T09:42:00-06:00
tags = ["Advent of Code", "Advent of Code 2015"]
markup = "markdown"
+++

Santa was able to deliver his presents to the large apartment building. Now, his
elves need wrapping paper and ribbon.

> The elves are running low on wrapping paper, and so they need to submit an order for more. They have a list of the dimensions (length `l`, width `w`, and height `h`) of each present, and only want to order exactly as much as they need.
>  
> Fortunately, every present is a box (a perfect right rectangular prism), which makes calculating the required wrapping paper for each gift a little easier: find the surface area of the box, which is `2*l*w + 2*w*h + 2*h*l`. The elves also need a little extra paper for each present: the area of the smallest side.
> 
> For example:
> 
> * A present with dimensions `2x3x4` requires `2*6 + 2*12 + 2*8 = 52` square feet of wrapping paper plus `6` square feet of slack, for a total of `58` square feet.
> * A present with dimensions `1x1x10` requires `2*1 + 2*10 + 2*10 = 42` square feet of wrapping paper plus `1` square foot of slack, for a total of `43` square feet.
> 
> All numbers in the elves' list are in feet. How many total __square feet of wrapping paper__ should they order?

## Analysis

There are several steps required in order to achieve the requirements of this
program:

* Read the input file
* Parse the input lines
* Perform the arithmetic

Reading the input lines will require one read operation per line. 

The input lines are formatted in the form `#x#x#`, where the three numerical
dimensions are delimited by the character `x`. If a string split function is
available, an array of strings with the three dimensions (in string format) can
easily be had by splitting on the `x` delimiter; for lower-level languages such 
as C, the string may need to be (semi-)manually parsed. For my C implementation,
I used `strtok`, which--despite its bad reputation--is the best option for this
use case.

Once the input lines are split, each element will need to be converted
from a string to an integer. `atoi` or similar can be used here, with the end
result being an array of integers representing the three dimensions.

The "gotcha" is that the instructions require us to know which two dimensions
are the smallest. We can either sort the dimension array ahead of time or
manually determine which of the three dimensions is the largest and exclude
that dimension. Given the improvement in ergonomics, I prefer sorting ahead, so
that we can access elements `0` and `1` and know that they are the two smallest
elements.

Intuitively, despite multiple steps per operation, there is only one loop on
the original data set so it seems that the complexity of this task is _O(n)_.
As we break the operations down:

* Read the input file: _O(n)_
* Split the lines: _3 * O(n)_
* Convert the dimensions: _3 * O(n)_
* Sort the dimensions: This is where things can get a bit unclear. `qsort` in
the C standard library has a worst-case complexity of _O(n^2)_, however, _n_ in
this case refers to the length of the dimensions array, which is 3 in each case.
We are running this once per input line, so really we are adding _O(n)_ to the
complexity.
* Perform the math: _O(n)_

We end up with a final complexity of _O(9n)_, which asymptotically lands at
_O(n)_ matching our conclusion that we are only performing one loop on the
original data.


### C
{{< highlight c >}}
FILE *input = fopen(inputFile);

int area = 0;
char buf[256];
while(fgets(buf, 256, input) != NULL) {
    char *dims_s[3];
    char dims[3];

    int i = 0;
    char *tmp = buf;
    for(i = 0; i < 3; i++) {
        dims_s[i] = strtok(tmp, "x");
        tmp = NULL;
    }
    for(i = 0; i < 3; i++) {
        dims[i] = atoi(dims_s[i]);
    }
    qsort(dims, 3, sizeof(int), compi);
    area += 2 * dims[0] * dims[1] + 2 * dims[1] * dims[2] + 2 * dims[2] * dims[0] + dims[0] * dims[1];
}
{{< /highlight >}}

### Optimization
There are no immediately obvious areas for optimization, aside from potentially
improving the sort operation based on the knowledge that we will always have
three positive integers in the array.

---

The followup problem appears:

> The elves are also running low on ribbon. Ribbon is all the same width, so they only have to worry about the length they need to order, which they would again like to be exact.
> 
> The ribbon required to wrap a present is the shortest distance around its sides, or the smallest perimeter of any one face. Each present also requires a bow made out of ribbon as well; the feet of ribbon required for the perfect bow is equal to the cubic feet of volume of the present. Don't ask how they tie the bow, though; they'll never tell.
> 
> For example:
> 
> * A present with dimensions `2x3x4` requires `2+2+3+3 = 10` feet of ribbon to wrap the present plus `2*3*4 = 24` feet of ribbon for the bow, for a total of `34` feet.
> * A present with dimensions `1x1x10` requires `1+1+1+1 = 4` feet of ribbon to wrap the present plus `1*1*10 = 10` feet of ribbon for the bow, for a total of `14` feet.
> 
> How many total __feet of ribbon__ should they order?

## Analysis

The only change in operation for the second problem is an update of the
arithmetic. Sorting is still required as we still need to find the shortest
dimensions.
