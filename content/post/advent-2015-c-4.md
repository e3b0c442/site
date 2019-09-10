---
date: 2019-09-10T03:00:00Z
title: "Advent of Code 2015 Day 4: The Ideal Stocking Stuffer"
tags: ["Programming", "Advent of Code", "C", "Cryptography", "Hashing"]
---

I'm back for a second time today since this puzzle is fairly straightforward:

> Santa needs help mining some AdventCoins (very similar to bitcoins) to use as gifts for all the economically forward-thinking little girls and boys.
> 
> To do this, he needs to find MD5 hashes which, in hexadecimal, start with at least **five zeroes**. The input to the MD5 hash is some secret key (your puzzle input, given below) followed by a number in decimal. To mine AdventCoins, you must find Santa the lowest positive number (no leading zeroes: `1`, `2`, `3`, ...) that produces such a hash.
> 
> For example:
> 
> * If your secret key is `abcdef`, the answer is `609043`, because the MD5 hash of `abcdef609043` starts with five zeroes `(000001dbbfa...`), and it is the lowest such number to do so.
> * If your secret key is `pqrstuv`, the lowest number it combines with to make an MD5 hash starting with five zeroes is `1048970`; that is, the MD5 hash of `pqrstuv1048970` looks like `000006136ef...`.

As always, we'll start with breaking down the problem into discrete parts:
* Read the input
* Convert a number to a string
* Combine the input and the number string
* Get the MD5 digest of the combined string
* Check that the first five digits of the digest are `0`

Rather than write our own MD5 hashing implementation, we will use the well-known
and commonly-installed OpenSSL library's implementation. You'll need to make
sure that your linker pulls in the OpenSSL library.

Let's write some code:

```c
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
```

For the sake of completion, this is our standard prelude of verifying the
command-line argument and reading the input file.

```c
    printf("Day 4:\n");

    /* initialize variables */
    int i = 1;
    char buf[strlen(input) + 11];
    memset(buf, 0, strlen(input) + 11);
    strcpy(buf, input);
    unsigned char digest[16];
```

We print our preamble, then initialize some variables. Note that I'm doing this
outside of a loop so that we do not incur this cost in the loop. Our code will
be written in such a way that the variables do not need to be reinitialized on
every pass. Step by step we have:

* The declaration of the iteration index `i`, which is the number that will be 
added to the input string on each pass
* The declaration of the buffer which stores the whole string to be hashed. We
size from the input and then add 11 -- 10 for the maximum number of digits in a 
32-bit integer, and 1 for the null terminator.
* The initialization of the string buffer. Ensuring all bytes are set to `0` now
will help us not need to re-initialize the variable on every loop
* Copying the input into the buffer
* Creating a buffer for the MD5 digest. MD5 digests are 128 bits, so we need a
16-byte buffer. Note that we do _not_ have a null terminator here because this
is not a string.

```c
    while (1)
    {
        sprintf(buf + strlen(input), "%d", i);
        MD5((const unsigned char *)buf, strlen(buf), digest);
```

Now we begin the loop. W se formatted print to append the string representation 
of `i` to the buffer. Notice that we are using pointer arithmetic to point to 
the first index in the buffer after the original input string. Because the
numbers will only ever increase, we will overwrite this on each loop and not
need to reinitialize the variable. We then get the MD5 digest of the constructed
string.

At this point, one might be tempted to use formatted print to output the 
hexadecimal representation of the digest. _Don't!_ String formatting operations
are computationally expensive, and we have the information we need to solve
the problem without doing it.

```c
        if (digest[0] == 0 && digest[1] == 0 && digest[2] < 16)
            break;
        i++;
    }
```

Note that the ratio of bytes to hex characters is 1:2; that is, each byte of the
digest would translate to two hex characters. With this in mind, remember that
this digest is just one big number; so, rather than convert the number back to
a string, let's just check the number.

We are searching for a digest where the first five digits of the hexadecimal 
representation of the digest are `0`. For bytes 0 and 1, this is easy: the value
of the byte must be 0. For byte 2, it requires a little bit more thought, since
only the first digit must be zero. In this case, it's just a matter of places
like in decimal math. There are 16 possible hexadecimal digits, so any value
of the byte that is less than 16 will have the first hex digit as 0.

By checking this way and avoiding the string conversion, we cut our compute
usage down by 10-100x, which is important in this exercise as hashing is not a
fast operation, relatively speaking.

```c
    printf("\tSolution 1: %d\n", i);
```

We now have our solution, which is the current value of `i`. Once this is
validated, our second puzzle appears:

> Now find one that starts with six zeroes.

Well, that's succinct, isn't it? Literally all we need to do at this point is
resume the loop right where we left off, making one change: we can check that
byte 2 of the digest is equal to zero, rather than less than 16:

```c
    while (1)
    {
        sprintf(buf + strlen(input), "%d", i);
        MD5((const unsigned char *)buf, strlen(buf), digest);

        if (digest[0] == 0 && digest[1] == 0 && digest[2] == 0)
            break;
        i++;
    }
```

We don't even need to reset `i` here, since by definition a digest can't start
with six zeroes without starting with five. In the circumstance that the first
digest that begins with five zeroes also begins with six zeroes, the first 
iteration of our loop will catch it because we broke out of the previous loop 
before incrementing the index, and we don't increment until the end of the loop.

We now have our solution, and can print it and clean up:

```c
    printf("\tSolution 2: %d\n", i);

    free(input);
    return 0;
}
```

When we put it all together, we have:

'''c
#include <errno.h>
#include <openssl/md5.h>
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

    printf("Day 4:\n");

    /* initialize variables */
    int i = 1;
    char buf[strlen(input) + 11];
    memset(buf, 0, strlen(input) + 11);
    strcpy(buf, input);
    unsigned char digest[16];

    /* find the first matching digest */
    while (1)
    {
        sprintf(buf + strlen(input), "%d", i);
        MD5((const unsigned char *)buf, strlen(buf), digest);

        if (digest[0] == 0 && digest[1] == 0 && digest[2] < 16)
            break;
        i++;
    }

    printf("\tSolution 1: %d\n", i);

    /* find the first matching digest */
    while (1)
    {
        sprintf(buf + strlen(input), "%d", i);
        MD5((const unsigned char *)buf, strlen(buf), digest);

        if (digest[0] == 0 && digest[1] == 0 && digest[2] == 0)
            break;
        i++;
    }

    printf("\tSolution 2: %d\n", i);

    free(input);
    return 0;
}
```