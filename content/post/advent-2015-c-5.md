---
date: 2019-09-14T00:00:00Z
title: "Advent of Code 2015 Day 5: Doesn't He Have Intern-Elves For This?"
tags: ["Programming", "Advent of Code", "C", "Regular Expressions"]
---

Welcome back! I feel refreshed after a few days, hopefully you do too. Today, we
are going to look at day 5:

> Santa needs help figuring out which strings in his text file are naughty or nice.
>
> A **nice string** is one with all of the following properties:
>
> - It contains at least three vowels (`aeiou` only), like `aei`, `xazegov`, or `aeiouaeiouaeiou`.
> - It contains at least one letter that appears twice in a row, like `xx`, `abcdde` (`dd`), or `aabbccdd` (`aa`, `bb`, `cc`, or `dd`).
> - It does **not** contain the strings `ab`, `cd`, `pq`, or `xy`, even if they are part of one of the other requirements.
>
> For example:
>
> - `ugknbfddgicrmopn` is nice because it has at least three vowels (`u...i...o...`), a double letter (`...dd...`), and none of the disallowed substrings.
> - `aaa` is nice because it has at least three vowels and a double letter, even though the letters used by different rules overlap.
> - `jchzalrnumimnmhp` is naughty because it has no double letter.
> - `haegwjzuvuyypxyu` is naughty because it contains the string `xy`.
> - `dvszwmarrgswjxmb` is naughty because it contains only one vowel.
>
> How many strings are nice?

Let's begin by breaking down the problem. We need to:

- Read the input file into memory
- Split the input into its components
- Determine whether the string meets the naughty or nice requirements

The first two parts of this problem are familiar, and generally speaking,
identical. We'll use the variable `good` to track the good strings.

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

    printf("Day 5:\n");
```

The last part, however, is the most difficult problem that has yet been
presented. How are we going to check the strings?

While it might be tempting to loop over the strings and implement tracking
variables to check these conditions (and, depending on the implementation, might
be faster), the more appropriate tool in this situation is regular expressions.

```c
    regex_t rule1, rule2, rule3;
    if (regcomp(&rule1, "[aeiou].*[aeiou].*[aeiou]", 0))
        goto err_cleanup;

    if (regcomp(&rule2, "\\(.\\)\\1", 0))
        goto err_cleanup;

    if (regcomp(&rule3, "\\(ab\\|cd\\|pq\\|xy\\)", 0))
        goto err_cleanup;

```

Each of the three rules can be checked using a valid POSIX basic regular
expression (note that in the code snippet above, the appropriate escape
characters have been added):

- at least 3 vowels: `[aeiou].*[aeiou].*[aeiou]`
  - This regular expression says: _find a letter a, e, i, o, or u. Then find
    zero or more of any character, then a letter a,e,i,o,u, then zero or more of
    any character, then a, e, i, o, or u._
- at least one letter that occurs twice in a row: `(.)\1`
  - This regular expression uses a backreference to refer to a prior character.
    It says: _find a group containing any one character, then find the first
    group_. In this case, the first group is the matched one character.
- does not contain ab, cd, pq, xy: `(ab|cd|pq|xy)`
  - We'll actually be looking for a negative result on this one because it is
    simpler than trying to do the negation in the expression itself. This
    expression says, _find the strings ab or cd or pq or xy_.

We compile the expressions once to be used to quickly match the strings as we
loop through the input:

```c
    int good = 0;

    char *cursor = input;
    char *line = strsep(&cursor, "\n");
    while (line != NULL)
    {
        if (strlen(line) == 0)
            goto tokenize;

        if (regexec(&rule3, line, 0, NULL, 0) && !regexec(&rule1, line, 0, NULL, 0) && !regexec(&rule2, line, 0, NULL, 0))
            good++;

    tokenize:
        line = strsep(&cursor, "\n");
    }

    printf("\tSolution 1: %d\n", good);

    goto cleanup;
```

If rule 3 does not match (we put this one in front to short-circuit, since if
it matches the string is automatically invalid) and rules 1 and 2 match, then
the string is a good string. Note that this appears backwards due to `regexec`
returning 0 on match and 1 on no match. We output the counted good strings for
our solution.

Then, we just need to clean up:

```c
err_cleanup:
    rc = -1;
    printf("badman\n");
cleanup:
    regfree(&rule1);
    regfree(&rule2);
    regfree(&rule3);
    free(input);
    return rc;
}
```

With our solution submitted, we then get our second problem:

> Realizing the error of his ways, Santa has switched to a better model of determining whether a string is naughty or nice. None of the old rules apply, as they are all clearly ridiculous.
>
> Now, a nice string is one with all of the following properties:
>
> - It contains a pair of any two letters that appears at least twice in the string without overlapping, like `xyxy` (`xy`) or `aabcdefgaa` (`aa`), but not like `aaa` (`aa`, but it overlaps).
> - It contains at least one letter which repeats with exactly one letter between them, like `xyx`, `abcdefeghi` (`efe`), or even `aaa`.
>
> For example:
>
> - `qjhvhtzxzqqjkmpb` is nice because is has a pair that appears twice (`qj`) and a letter that repeats with exactly one letter between them (`zxz`).
> - `xxyxx` is nice because it has a pair that appears twice and a letter that repeats with one between, even though the letters used by each rule overlap.
> - `uurcxstgmygtbstg` is naughty because it has a pair (`tg`) but no repeat with a single letter between them.
> - `ieodomkazucvgmuy` is naughty because it has a repeating letter with one between (`odo`), but no pair that appears twice.
>
> How many strings are nice under these new rules?

As expected, the problem is very similar, if slightly more complicated. The good
news is that these rules can also be checked with valid POSIX basic regular
expressions, so we can easily add the new rules to the existing loop.

```c
    regex_t rule1, rule2, rule3, rule4, rule5;
    if (regcomp(&rule1, "[aeiou].*[aeiou].*[aeiou]", 0))
        goto err_cleanup;

    if (regcomp(&rule2, "\\(.\\)\\1", 0))
        goto err_cleanup;

    if (regcomp(&rule3, "\\(ab\\|cd\\|pq\\|xy\\)", 0))
        goto err_cleanup;

    if (regcomp(&rule4, "\\(..\\).*\\1", 0))
        goto err_cleanup;

    if (regcomp(&rule5, "\\(.\\).\\1", 0))
        goto err_cleanup;
```

The two new rules are:

- A pair of two letters that appear at least twice in the string without
  overlapping: `(..).*\1`
  - This regular expression says: _find a group with two of any character, then
    find zero or more of any character, then find the first group_.
- one letter which repeats with exactly one letter between: `(.).\1`
  - This regular expression says: _find a group with one of any character, then
    find one of any character, then find the first group_.

These are again compiled.

```c
    int good = 0;
    int great = 0;

    char *cursor = input;
    char *line = strsep(&cursor, "\n");
    while (line != NULL)
    {
        if (strlen(line) == 0)
            goto tokenize;

        if (regexec(&rule3, line, 0, NULL, 0) && !regexec(&rule1, line, 0, NULL, 0) && !regexec(&rule2, line, 0, NULL, 0))
            good++;

        if (!regexec(&rule4, line, 0, NULL, 0) && !regexec(&rule5, line, 0, NULL, 0))
            great++;

    tokenize:
        line = strsep(&cursor, "\n");
    }

    printf("\tSolution 1: %d\n", good);
    printf("\tSolution 2: %d\n", great);
```

We add a second check for the two new rules, with a separate counter, then print
the second result after the first.

```c
err_cleanup:
    rc = -1;
    printf("badman\n");
cleanup:
    regfree(&rule1);
    regfree(&rule2);
    regfree(&rule3);
    regfree(&rule4);
    regfree(&rule5);
    free(input);
    return rc;
}
```

Finally, we clean up the new compiled regexes in addition to the previous ones.

All together, we have:

```c
#include <errno.h>
#include <regex.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "common.h"

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

    printf("Day 5:\n");

    regex_t rule1, rule2, rule3, rule4, rule5;
    if (regcomp(&rule1, "[aeiou].*[aeiou].*[aeiou]", 0))
        goto err_cleanup;

    if (regcomp(&rule2, "\\(.\\)\\1", 0))
        goto err_cleanup;

    if (regcomp(&rule3, "\\(ab\\|cd\\|pq\\|xy\\)", 0))
        goto err_cleanup;

    if (regcomp(&rule4, "\\(..\\).*\\1", 0))
        goto err_cleanup;

    if (regcomp(&rule5, "\\(.\\).\\1", 0))
        goto err_cleanup;

    int good = 0;
    int great = 0;

    char *cursor = input;
    char *line = strsep(&cursor, "\n");
    while (line != NULL)
    {
        if (strlen(line) == 0)
            goto tokenize;

        if (regexec(&rule3, line, 0, NULL, 0) && !regexec(&rule1, line, 0, NULL, 0) && !regexec(&rule2, line, 0, NULL, 0))
            good++;

        if (!regexec(&rule4, line, 0, NULL, 0) && !regexec(&rule5, line, 0, NULL, 0))
            great++;

    tokenize:
        line = strsep(&cursor, "\n");
    }

    printf("\tSolution 1: %d\n", good);
    printf("\tSolution 2: %d\n", great);

    goto cleanup;

err_cleanup:
    rc = -1;
    printf("badman\n");
cleanup:
    regfree(&rule1);
    regfree(&rule2);
    regfree(&rule3);
    regfree(&rule4);
    regfree(&rule5);
    free(input);
    return rc;
}
```

This is just one instance where a problem that is potentially complex can be
simplified with regular expressions (note: this goes both ways!)

As always, if you have questions, comments, or suggestions, feel free to leave
a comment!
