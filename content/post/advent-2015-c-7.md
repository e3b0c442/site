---
date: 2019-10-15T00:00:00Z
title: "Advent of Code 2015 Day 7: Some Assembly Required"
tags:
  [
    "Programming",
    "Advent of Code",
    "C",
    "Data Structures",
    "Regular Expressions",
  ]
---

It's been a few weeks, but I'm glad to be able to pick up the Advent of Code series again. In the last post, I took a bit of a digression to discuss hash tables. In today's post, we'll see why. Let's see what we have in store for day 7:

> This year, Santa brought little Bobby Tables a set of wires and [bitwise logic gates](https://en.wikipedia.org/wiki/Bitwise_operation)! Unfortunately, little Bobby is a little under the recommended age range, and he needs help assembling the circuit.
>
> Each wire has an identifier (some lowercase letters) and can carry a [16-bit](https://en.wikipedia.org/wiki/16-bit) signal (a number from `0` to `65535`). A signal is provided to each wire by a gate, another wire, or some specific value. Each wire can only get a signal from one source, but can provide its signal to multiple destinations. A gate provides no signal until all of its inputs have a signal.
>
> The included instructions booklet describes how to connect the parts together: `x AND y -> z` means to connect wires `x` and `y` to an AND gate, and then connect its output to wire `z`.
>
> For example:
>
> - `123 -> x` means that the signal `123` is provided to wire `x`.
> - `x AND y -> z` means that the [bitwise AND](https://en.wikipedia.org/wiki/Bitwise_operation#AND) of wire `x` and wire `y` is provided to wire `z`.
> - `p LSHIFT 2 -> q` means that the value from wire `p` is [left-shifted](https://en.wikipedia.org/wiki/Logical_shift) by `2` and then provided to wire `q`.
> - `NOT e -> f` means that the [bitwise complement](https://en.wikipedia.org/wiki/Bitwise_operation#NOT) of the value from wire `e` is provided to wire `f`.
>
> Other possible gates include `OR` ([bitwise OR](https://en.wikipedia.org/wiki/Bitwise_operation#OR)) and `RSHIFT` ([right-shift](https://en.wikipedia.org/wiki/Logical_shift)). If, for some reason, you'd like to **emulate** the circuit instead, almost all programming languages (for example, [C](https://en.wikipedia.org/wiki/Bitwise_operations_in_C), [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Bitwise_Operators), or [Python](https://wiki.python.org/moin/BitwiseOperators)) provide operators for these gates.
>
> For example, here is a simple circuit:
>
> ```
> 123 -> x
> 456 -> y
> x AND y -> d
> x OR y -> e
> x LSHIFT 2 -> f
> y RSHIFT 2 -> g
> NOT x -> h
> NOT y -> i
> ```

```
>
> After it is run, these are the signals on the wires:
>
> ```
d: 72
e: 507
f: 492
g: 114
h: 65412
i: 65079
x: 123
y: 456
```

> In little Bobby's kit's instructions booklet (provided as your puzzle input), what signal is ultimately provided to **wire `a`**?

This is a fairly complex problem, and in contrast to previous days' entries, there is more than one way to go about solving it successfully and efficiently. I chose to solve this problem by creating _hash tables_ with the gate statements, and then recursively calling an operator function to find the gate value. Hash tables are approprate for this task for two reasons:

1. Accessing the gate instructions will effectively be random-access since there is no linearity to how the instructions are arranged. Hash tables on average have _O(1)_ time complexity for searches.
2. The gates are named by strings, allowing us to translate them directly into hash table keys.

We will utilize the [esht]({{< ref "esht.md" >}}) implementation discussed in the prior article. As noted in that article, while there is a POSIX hash table implementation, it is non-reentrant, and a re-entrant version included as part of the GNU C library is not present in many UNIX flavors, including macOS. In this case, reentrancy is important as we will soon see that we need to maintain two tables.

As we examine this problem, what might immediately jump out is that each of these gates is a function that takes one or two 16-bit input values and outputs another value. Indeed, it would be possible, and some might consider elegant, to parse the inputs into a linked tree of function pointers using function pointers. However, I feel like this adds some unnecessary complexity and does not avoid the need for an additional caching mechanism, which is necessary due to the fact that we cannot guarantee that the gates are a true tree.

Given this, it is much simpler and cleaner to parse the gate definitions into a data structure and make recursive calls to a single executor function to accomplish the goal of the task.

With that lengthy discussion out of the way, let's break the problem down a bit as we have done in previous weeks.

- Read the input file into memory
- Parse the individual gates into a hash table
- Execute the gates

As gates can also be inputs to another gate, we will need to call the gate executor recursively, so we will make this a separate function. We'll start our coding here.

```c
uint16_t do_op(esht *instrs, esht *cache, char *key)
```

Our gate execution function takes three arguments: the hash table with the gate instructions keyed by gate name, a cache of already-resolved values also keyed by the gate name, and the gate key itself. The return value is `uint16_t`, which is an unsigned 16-bit integer.

```c
{
    uint16_t rval;
    uint16_t *cached = (uint16_t *)esht_get(cache, key, NULL);
    if (cached != NULL)
    {
        rval = *cached;
        free(cached);
        return rval;
    }
```

The first thing we are going to do is check for a cached value. We use the provided key and cache table pointer to call `esht_get()`; we cast the return value to a pointer to `uint16_t` as the values are stored in the hash table as void pointers. If a result is found in the cache, we copy the value, clean up the allocated memory (remember that with `esht`, the caller is responsible for cleanup of returned values), and return the retrieved value.

```c
    char *instr = (char *)esht_get(instrs, key, NULL);
    if (instr == NULL)
    {
        errno = ENOKEY;
        goto err_cleanup;
    }
    regmatch_t matches[4] = {0};
    int res = regexec(&instr_r, instr, 4, matches, 0);
    if (res == REG_NOMATCH)
    {
        errno = EINVAL;
        goto err_cleanup;
    }
```

If there is no value in the cache for the key, it is time to start solving the gate value. We'll first pull the gate definition from the hash table where we've already stored it. We then declare an array of four matches; the instruction parsing expression has three submatches in addition to the whole string match, which we will discuss later. For now, it is sufficient to know that we are capturing the three components of the gate expression. Finally, we execute the regular expression to populate the match variables. If either the instruction lookup fails or the regular expression execution returns no match, `errno` is set appropriately and execution jumps to the error cleanup handler.

```c
    char ls[6] = {0};
    char op[7] = {0};
    char rs[6] = {0};

    char *check;
    unsigned long l, r = 0;
```

Now we start working on the gate values. First, we set up temporary buffers for the string representations of each of the components of the gate expression: the left operand, the operation, and the right operand. The buffer sizes are chosen specifically; since l and r can be no larger than 65536, we know that 5 characters is the max storage; likewise, the longest gate operation is 6 characters. All buffers have the extra byte for the null terminator, and are initialized appropriately. We then additionally define a `char` pointer which is used as part of the `strtoul` error check, and finally integral values for the left and right operands of the gate.

```c
    if (matches[1].rm_so != -1)
    {
        strncpy(ls, instr + matches[1].rm_so, matches[1].rm_eo - matches[1].rm_so);
        l = strtoul(ls, &check, 10);
        if (strlen(ls) > 0)
            if (l == 0 && check == ls)
            {
                l = do_op(instrs, cache, ls);
                if (l == UINT16_MAX && errno != 0)
                    goto err_cleanup;
            }
    }
```

With our variable initalization complete, we'll start checking the values. For the left operand, the first thing we will check is that it is actually there, since some gates only have a single (right) operand. One useful properties of regular expressions in this case is that even if a submatch is conditional, its position in the results is still
preserved, so we know that the right operand will always have match index 3, regardless of whether the left operand and operator exist at all. If the submatch does not actually exist, member `rm_so` of the `regmatch_t` returned by `regexec` will be -1.

Once we've verified the submatch exists, we will copy just the submatch value to our temporary buffer by using the offests returned in the `regmatch_t`. We will then attempt to convert the buffer to an unsigned long using `strtoul`. We pass `check` into the function; if the value of `check` is the same as the string pointer once the operation is done and the return value is 0, we know that no number was actually found in the string and we can assume that rather than a value, it is the key to another gate and we can consequently make the recursive call. If the call returns the sentinel value and `errno` is set, we'll propogate that up.

```c
    if (matches[3].rm_so != -1)
    {
        strncpy(rs, instr + matches[3].rm_so, matches[3].rm_eo - matches[3].rm_so);
        r = strtoul(rs, &check, 10);
        if (strlen(rs) > 0)
        {
            if (r == 0 && check == rs)
            {
                r = do_op(instrs, cache, rs);
                if (r == UINT16_MAX && errno != 0)
                    goto err_cleanup;
            }
        }
        else
        {
            errno = EINVAL;
            goto err_cleanup;
        }
    }
```
Now we do the same for the right operand; the only difference being that we always expect the right operand to be there; if it is absent, that is considered an error and we will return appropriately.

```c
    if (matches[2].rm_so != -1)
    {
        strncpy(op, instr + matches[2].rm_so, matches[2].rm_eo - matches[2].rm_so);

        if (!strcmp(op, "LSHIFT"))
            rval = l << r;
        else if (!strcmp(op, "RSHIFT"))
            rval = l >> r;
        else if (!strcmp(op, "AND"))
            rval = l & r;
        else if (!strcmp(op, "OR"))
            rval = l | r;
        else if (!strcmp(op, "NOT"))
            rval = ~r;
        else
        {
            errno = EINVAL;
            goto err_cleanup;
        }
    }
    else
        rval = r;
```

Then we will parse the operator. We know what the expected values are here; if the operator does not exist, we know that we are just returning the value provided by the right operand; else, we find one of the known bitwise operations and perform it, setting the return value accordingly. If the operator exists but is not an expected value, this is an error condition and we jump appropriately.

```c
    esht_update(cache, key, &rval, sizeof(uint16_t));
    goto cleanup;

err_cleanup:
    rval = UINT16_MAX;
cleanup:
    if (instr != NULL)
        free(instr);
    return rval;
}
```

If we have gotten this far, we now have a final value for this gate that can be cached. We will add the value to the cache table with the gate's key so that future calls to this gate can just return the cached value. Then we will go to the cleanup, where we make sure to free the instruction we retrieved from the hash table and free our iretrieved instruction.

Now that we have our operation function implemented, we can look at the containing function for the problem.

```c
int main(int argc, char const *argv[])
{
    int rc, res = 0;

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

    printf("Day 7:\n");
```

Identically to previous days, we parse the arguments, open the file, and copy it into memory.

```c
    /* create the regular expression for parsing into the table */
    regex_t gate_r;
    int result = regcomp(&gate_r, "(.*) -> ([a-z]+)", 0);
    if (result != 0)
        goto err_cleanup;
```

We then compile the regular expression that splits the gate operation from its key. If this returns an unexpected result, we will jump to the error cleanup. This is a fairly simple regex, that captures two submatches: anything to the left of ` -> `, and then a gate key to the right of ` -> `, which is only letters.

```c
    char *cursor = input;
    char *line = strsep(&cursor, "\n");
    regmatch_t matches[3] = {0};
    esht *inst_table = esht_create();
    if (inst_table == NULL)
        goto err_cleanup;
```

Now we will set up some variables in preparation for looping over the input, including our line cursor and an array of `regmatch_t` for capturing the gate operations and keys, which we initialize to all zeroes on declaration. Then we create the table that will hold all of the gates.

```c
    /* parse the gates into the hash table */
    while (line != NULL)
    {
        if (strlen(line) == 0)
            goto tokenize;

        memset(matches, 0, 3 * sizeof(regmatch_t));
        result = regexec(&gate_r, line, 3, matches, 0);
        if (result != 0)
            goto err_cleanup;

        char key[4] = {0};
        char val[64] = {0};

        strncpy(val, line + matches[1].rm_so, matches[1].rm_eo - matches[1].rm_so);
        strncpy(key, line + matches[2].rm_so, matches[2].rm_eo - matches[2].rm_so);

        int res;
        if ((res = esht_update(inst_table, key, val, strlen(val) + 1)) != 0)
            goto err_cleanup;

    tokenize:
        line = strsep(&cursor, "\n");
    }
```

Now we loop over the input, parsing the input lines into the hash table. We first check that there is actually length ta the line; if so, then we zero out the match objects (since we aren't declaring them inside the loop) and execute the regular expression against the line. If there is no match, we'll break the loop and error out.

Assuming all goes well, we will then declare temporary buffors for the key and value; we chose a sane length for both to avoid the need to allocate on the heap. We then use the offsets in the `regmatch_t` to copy the key and gate operation into the buffers. Finally, we add the gate to the hash table, erroring out if there is an unexpected return from the update operation.

```c
    result = regcomp(&instr_r, "^(?:([a-z0-9]+) )?(?:(AND|OR|RSHIFT|LSHIFT|NOT|) )?([a-z0-9]+)$", 0);
    if (result)
        goto err_cleanup;
```

We then create the expression for parsing the gate operations, which is stored in a file-local manner as mentioned before. This expression searches for an optional alphanumeric value, followed by an optional operator from the list provided in the problem specification, followed by a _required_ alphanumeric expression. If there is a problem compiling the regular expression, we'll jump to the error handler.

```c
    /* create a cache and perform the recursive operation to get the gate value */
    esht *cache = esht_create();
    if (cache == NULL)
        goto err_cleanup;
    uint16_t a = do_op(inst_table, cache, "a");
    printf("\tSolution 1: %d\n", a);

```

Then we allocate another table to cache values. This is required due to the fact that the connections between the gates form more of a web than a tree, which can cause, if not complete short-circuits, unnecessary multiple executions of the gate to find the value. We call the operation function with the key "a" as defined in the problem statement, which will recursively call itself with the appropriate gates to calculate the final value at wire "a", which we will then print.

Now that we have solved the first problem, let's see what the second problem will require:

> Now, take the signal you got on wire `a`, override wire `b` to that signal, and reset the other wires (including wire `a`). What new signal is ultimately provided to wire `a`?

It's a good thing we made that cache table! Since we didn't alter the original instructions, all we need to do is destroy the old cache, then set value "b" in the new cache as directed in the instructions:

```c

    /* clear cache and set b to value of a */
    esht_destroy(cache);
    cache = esht_create();
    if (cache == NULL)
        goto err_cleanup;
    if ((res = esht_update(cache, "b", &a, sizeof(uint16_t))) != 0)
        goto err_cleanup;

    /* get solution 2 */
    a = do_op(inst_table, cache, "a");
    printf("\tSolution 2: %d\n", a);
    goto cleanup;

```

Once Solution 2 is printed, we can clean up:

```c

err_cleanup:
    rc = -1;
cleanup:
    if (cache != NULL)
        esht_destroy(cache);
    regfree(&instr_r);
    if (inst_table != NULL)
        esht_destroy(inst_table);
    regfree(&gate_r);
    free(input);
    return rc;
}
```

Putting the whole thing together, we have:

```c
#include <errno.h>
#include <pcreposix.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "common.h"
#include "esht.h"

static regex_t instr_r;

uint16_t do_op(esht *instrs, esht *cache, char *key)
{
    uint16_t rval;
    uint16_t *cached = (uint16_t *)esht_get(cache, key, NULL);
    if (cached != NULL)
    {
        rval = *cached;
        free(cached);
        return rval;
    }

    char *instr = (char *)esht_get(instrs, key, NULL);
    if (instr == NULL)
    {
        errno = ENOKEY;
        goto err_cleanup;
    }
    regmatch_t matches[4] = {0};
    int res = regexec(&instr_r, instr, 4, matches, 0);
    if (res == REG_NOMATCH)
    {
        errno = EINVAL;
        goto err_cleanup;
    }

    char ls[6] = {0};
    char op[7] = {0};
    char rs[6] = {0};

    char *check;
    unsigned long l, r = 0;

    if (matches[1].rm_so != -1)
    {
        strncpy(ls, instr + matches[1].rm_so, matches[1].rm_eo - matches[1].rm_so);
        l = strtoul(ls, &check, 10);
        if (strlen(ls) > 0)
            if (l == 0 && check == ls)
            {
                l = do_op(instrs, cache, ls);
                if (l == UINT16_MAX && errno != 0)
                    goto err_cleanup;
            }
    }

    if (matches[3].rm_so != -1)
    {
        strncpy(rs, instr + matches[3].rm_so, matches[3].rm_eo - matches[3].rm_so);
        r = strtoul(rs, &check, 10);
        if (strlen(rs) > 0)
        {
            if (r == 0 && check == rs)
            {
                r = do_op(instrs, cache, rs);
                if (r == UINT16_MAX && errno != 0)
                    goto err_cleanup;
            }
        }
        else
        {
            errno = EINVAL;
            goto err_cleanup;
        }
    }

    if (matches[2].rm_so != -1)
    {
        strncpy(op, instr + matches[2].rm_so, matches[2].rm_eo - matches[2].rm_so);

        if (!strcmp(op, "LSHIFT"))
            rval = l << r;
        else if (!strcmp(op, "RSHIFT"))
            rval = l >> r;
        else if (!strcmp(op, "AND"))
            rval = l & r;
        else if (!strcmp(op, "OR"))
            rval = l | r;
        else if (!strcmp(op, "NOT"))
            rval = ~r;
        else
        {
            errno = EINVAL;
            goto err_cleanup;
        }
    }
    else
        rval = r;

    esht_update(cache, key, &rval, sizeof(uint16_t));
    goto cleanup;

err_cleanup:
    rval = UINT16_MAX;
cleanup:
    if (instr != NULL)
        free(instr);
    return rval;
}

int main(int argc, char const *argv[])
{
    int rc, res = 0;

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

    printf("Day 7:\n");

    /* create the regular expression for parsing into the table */
    regex_t gate_r;
    int result = regcomp(&gate_r, "(.*) -> ([a-z]+)", 0);
    if (result != 0)
        goto err_cleanup;

    char *cursor = input;
    char *line = strsep(&cursor, "\n");
    regmatch_t matches[3] = {0};
    esht *inst_table = esht_create();
    if (inst_table == NULL)
        goto err_cleanup;

    /* parse the gates into the hash table */
    while (line != NULL)
    {
        if (strlen(line) == 0)
            goto tokenize;

        memset(matches, 0, 3 * sizeof(regmatch_t));
        result = regexec(&gate_r, line, 3, matches, 0);
        if (result != 0)
            goto err_cleanup;

        char key[4] = {0};
        char val[64] = {0};

        strncpy(val, line + matches[1].rm_so, matches[1].rm_eo - matches[1].rm_so);
        strncpy(key, line + matches[2].rm_so, matches[2].rm_eo - matches[2].rm_so);

        if ((res = esht_update(inst_table, key, val, strlen(val) + 1)) != 0)
            goto err_cleanup;

    tokenize:
        line = strsep(&cursor, "\n");
    }

    result = regcomp(&instr_r, "^(?:([a-z0-9]+) )?(?:(AND|OR|RSHIFT|LSHIFT|NOT|) )?([a-z0-9]+)$", 0);
    if (result)
        goto err_cleanup;

    /* create a cache and perform the recursive operation to get the gate value */
    esht *cache = esht_create();
    if (cache == NULL)
        goto err_cleanup;
    uint16_t a = do_op(inst_table, cache, "a");
    printf("\tSolution 1: %d\n", a);

    /* clear cache and set b to value of a */
    esht_destroy(cache);
    cache = esht_create();
    if (cache == NULL)
        goto err_cleanup;
    if ((res = esht_update(cache, "b", &a, sizeof(uint16_t))) != 0)
        goto err_cleanup;

    /* get solution 2 */
    a = do_op(inst_table, cache, "a");
    printf("\tSolution 2: %d\n", a);
    goto cleanup;

err_cleanup:
    rc = -1;
cleanup:
    if (cache != NULL)
        esht_destroy(cache);
    regfree(&instr_r);
    if (inst_table != NULL)
        esht_destroy(inst_table);
    regfree(&gate_r);
    free(input);
    return rc;
}
```

This is by far the most complex problem we have dealt with, requiring both a new data structure and the use of recursion to solve. 

As always, feel free to leave a comment if you have constructive feedback. We'll be back for day 8 shortly!