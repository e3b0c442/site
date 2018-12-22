+++
title = "Advent of Code 2015, Day 4: The Ideal Stocking Stuffer"
date = 2018-12-22T07:57:00-06:00
tags = ["Advent of Code", "Advent of Code 2015"]
markup = "markdown"
+++

Santa spent so much time working on his efficiency, he's run out of time for
stocking-stuffers and has decided to start an initial coin offering instead.

> Santa needs help mining some AdventCoins (very similar to bitcoins) to use as gifts for all the economically forward-thinking little girls and boys.
> 
> To do this, he needs to find [MD5](https://en.wikipedia.org/wiki/MD5) hashes which, in [hexadecimal](https://en.wikipedia.org/wiki/Hexadecimal), start with at least __five zeroes__. The input to the MD5 hash is some secret key (your puzzle input, given below) followed by a number in decimal. To mine AdventCoins, you must find Santa the lowest positive number (no leading zeroes: `1`, `2`, `3`, ...) that produces such a hash.
> 
> For example:
> 
> * If your secret key is `abcdef`, the answer is `609043`, because the MD5 hash of `abcdef609043` starts with five zeroes (`000001dbbfa...`), and it is the lowest such number to do so.
> * If your secret key is `pqrstuv`, the lowest number it combines with to make an MD5 hash starting with five zeroes is `1048970`; that is, the MD5 hash of `pqrstuv1048970` looks like `000006136ef...`.

## Analysis

<!--more-->
This problem is conceptually simple but difficult to place a complexity on.

For each increment of the numeric suffix, there are three basic operations that
need to take place:

* Build a string from the prefix and incremented suffix
* Hash the combined string
* Check the hash for the correct number of zeroes.

To build the string, we can use a standard formatter such as `sprintf`.

Hashing the string can be done with your favorite crypto library; I chose
OpenSSL because of its ubiquity. Your language of choice (Go, Python, Rust all
fall into this category) may have an MD5 hashing function built into the
standard library.

Where you can have some fun is verifying the hash. The task is to verify that
the hexadecimal representation of the hash begins with five zeroes, but most
libraries only output a byte array. Our first instinct might be to format that
byte array into a string, and then check the first 5 characters of the string;
however, the conversion operation is relatively expensive. Since hexadecimal
format was specificed, we can verify the byte array itself! In a hexadecimal
formatted string, every two characters represent one byte. For five zeroes, this
means the first two and a half bytes (so, the first two bytes, and then the
higher-order bits on the third byte) need to be zero. So, we can check that
the first two bytes are zero, and that the third byte is less than 0x10. This
allows us to more efficiently verify the hash requirements.

When analyzing for complexity, our _n_ in this case is one suffix. For each
suffix, we are building a string _O(n)_, hashing it _O(n)_, and then verifying
the hash _O(n)_, making an asymptotic complexity of _O(n)_. What we do not know,
however, is how many suffixes we'll need, which makes it effectively impossible
to determine how long the program will run.

### C
{{< highlight c >}}
#include <openssl/md5.h>
FILE *input = fopen(inputFile);

char prefix[16];
fgets(prefix, 16, input);

char work[64];
int suffix = 0;
unsigned char hash[MD5_DIGEST_LENGTH];

while(1) {
    suffix++;
    memset(work, 0, 64);
    sprintf(work, "%s%d", prefix, suffix);
    MD5((const unsigned char *)work, strlen(work), hash);
    if(hash[0] == 0 && hash[1] == 0 && hash[2] < 0x10) {
        break;
    }
}
{{< /highlight >}}

### Optimization
There are a couple of potential optimization avenues here.

Firstly is the string building. Formatters may not be the most efficient option
here; it is possible that manually converting the suffix integer to a string or
making the suffix a native string and "incrementing" it manually may be more
efficient than using a formatter.

Finding the fastest, rather than the most common, MD5 implementation will also
likely speed things up. Depending on your implementation, it may be backed by
assembly code that makes good use of registers and low-level caching.

---

Apparently the AdventCoin mining difficulty has just been bumped up:

> Now find one that starts with six zeroes.

## Analysis

There is no change required in our algorithm, except for checking that the third
byte of the hash is 0 instead of < 0x10.

Next: _coming soon_