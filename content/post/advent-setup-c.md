---
date: 2019-09-08
title: Advent of Code in-depth
tags: ["programming", "advent of code", "c", "best practices", "DRY"]
---

In the first technical article for this iteration of the site, I'd like to turn
to [Advent of Code](https://adventofcode.com). These are programming problems
built around a fun Christmas-themed storyline each year. I especially appreciate
that the problems are language-agnostic, so I frequently turn to some of the
easier ones when I'm getting familiar with a new language.

It can be easy to throw together just enough code to make the puzzle work, but
I thought I'd revisit these problems from a different angle, looking not only
at how to solve the problem, but how to write high-quality code, including error
handling and memory management. In order to up the challenge, the solutions will
be written in C, and the code will pass all Valgrind tests for memory leaks and
other issues. All code discussed in these posts will be in my [Advent of Code
solutions repository](https://github.com/e3b0c442/advent).

To kick things off, we'll take a look at code reuse. When you're quickly
prototyping or problem solving, you're frequently going to have identical
snippets of code in multiple places. If we're writing high-quality code, we want
to avoid this wherever possible. One common theme throughout the Advent of Code
problems will be reading an input file into memory. In order to avoid writing
the same code over and over, we're going to start things off by writing a
function to read a file into a memory buffer:

```c
int read_file_to_buffer(char **buf, char *filename);
```

Already, just in the function signature, we are making design decisions. We
could have written the function to return the pointer to the memory buffer, and
in most languages this makes the most sense. However, in C, the length of an
array is not an integral part of the data type, so we need to pass that
information back as well. Therefore, I made the decision to pass a pointer to 
pointer to char (array) to the function, which will be modified in the function
with the actual pointer to the allocated buffer, and to return the size of the
buffer in the return value (or -1 if there is an error). We use this pattern 
rather than passing in the buffer itself because we do not know ahead of time 
how the buffer needs to be sized. The caller will be responsible for freeing the 
buffer when they are done using it.

Now, lets think about what actually needs to happen to read a file into memory,
and what can go wrong.

* Does the file exist? Is it a valid, readable file?
* How big is the file; consequently, how big does my buffer need to be?
* What happens if we don't succeed in reading the actual data?

As you can see, when reading a file, there are a lot of potential failure
points. Keeping that in mind, let's start writing our code.

```c
int read_file_to_buffer(char **buf, const char *filename)
{
    int rval = 0;
    FILE *f = NULL;
    *buf = NULL;

    /* check that the file exists and is valid to read */
    struct stat s;
    int rc = stat(filename, &s);
    if (rc == -1)
        goto err_cleanup; /* errno is set by stat() */
```

The first thing we do is initialize variables that will exist for the life of
the function (and be part of the cleanup). `rval` will hold our return value, 
`f` will hold our file pointer, and `*buf` is the pointer to the buffer that 
will hold the file contents, which is passed in as a parameter. Each is
initialized to a sane default, as variables are not automatically initialized
in C.

With that out of the way, we'll make our first check. Using the POSIX `stat`
function, we query the file for information. Right off the bat, this will tell
us if the file exists; if it does not, `stat` will return `-1` and set `errno`
accordingly. If we see this, we will jump to our error cleanup.

At this point, I'm going to take a sidebar and talk a bit about `goto`. For most
programmers, there's a knee-jerk reaction to avoid `goto` at all costs.
Generally, I agree with this with one exception: _error cleanup._ In this case,
`goto` properly used provides an avenue to short-circuit function execution and
provide for cleanup in the case of an error without reusing the same code.

If the file exists, the `struct stat` will be populated with information about
the file, including the type and size. Continuing on:

```c
    /* check that we are working with an actual file */
    if (!S_ISREG(s.st_mode))
    {
        errno = EINVAL;
        goto err_cleanup;
    }
```

Our next check is that we are working with a regular file, as opposed to a 
directory, block device, or other file type. (note: `stat` follows symlinks!).
We use the `S_ISREG` macro to accomplish this. Because this is a check and not
necessarily an error condition, if the provided file is not a regular file we 
will set `errno` ourselves to indicate an invalid parameter and then jump to our
error cleanup.

```c
    /* open the file */
    f = fopen(filename, "r");
    if (f == NULL) 
        goto err_cleanup; /* errno is set by fopen() */
```

Now that we know the file exists, and that it's a regular file, we will try to
open it for reading. If the file is not readable for any reason, `fopen` will
return `NULL` and set `errno` appropriately. We will check for the `NULL`
return value and short-circuit as needed.

```c
    /* allocate the buffer */
    *buf = malloc(s.st_size);
    if (*buf == NULL)
        goto err_cleanup; /* errno is set by malloc() */
```

Now that we've successfully opened the file, we'll allocate the buffer. Even
though we already had the size information from the `stat` call, I've chosen
not to allocate the buffer until we know the file is readable, to avoid
unnecessary allocations. For this problem it doesn't really matter, but it's
a good habit to be in. If there is an error allocating the buffer such as
running out of memory, `malloc` will return `NULL` and set `errno`; again we
will check for this and short-circuit the function as necessary. I'm using
`malloc` instead of `calloc` because we have designed the function such that the
buffer will be completely filled by the file, negating the need to initialize
it.

```c
    /* read the file into the buffer */
    size_t rd = fread(*buf, 1, s.st_size, f);
    if (rd < s.st_size)
        if (ferror(f))
            goto err_cleanup;
```

Now we perform the actual read. `fread` is a buffered read which calls the
low-level `read` syscall, and as such it will block until the requested size
is read unless there is an error, allowing us to avoid need to loop until the
expected data is read. Instead, we will check to make sure the expected size
of data is read (the file size) and check for an error if it is not. If there
is an error reading the file, we'll short-circuit to our error cleanup.

```c
    /* file is read into the buffer, return the number of bytes read */
    rval = rd;
    goto cleanup;
```

We finish up the logic of our function by setting the return value to the amount
of data read, and then jumping to our non-error cleanup.

```c
err_cleanup:
    rval = -1;
    if (*buf != NULL)
        free(*buf);
    *buf = NULL;
cleanup:
    if (f != NULL)
        fclose(f);
    return rval;
```

Now for our cleanup. There are two things that need to be cleaned up in this
function: first, the file needs to be closed if it was opened, to avoid file
handle leaks. Also, if there was an error, the data buffer needs to be freed 
if it was allocated and the pointer set to NULL. Finally, if we did error,
we set `rval` to -1 which indicates the error to the caller, who can then get
more information by inspecting `errno.`

The whole function:
```c
int read_file_to_buffer(char **buf, const char *filename)
{
    int rval = 0;
    FILE *f = NULL;
    *buf = NULL;

    /* check that the file exists and is valid to read */
    struct stat s;
    int rc = stat(filename, &s);
    if (rc == -1)
        goto err_cleanup; /* errno is set by stat() */

    /* check that we are working with an actual file */
    if (!S_ISREG(s.st_mode))
    {
        errno = EINVAL;
        goto err_cleanup;
    }

    /* open the file */
    f = fopen(filename, "r");
    if (f == NULL)
        goto err_cleanup; /* errno is set by fopen() */

    /* allocate the buffer */
    *buf = malloc(s.st_size);
    if (*buf == NULL)
        goto err_cleanup; /* errno is set by malloc() */

    /* read the file into the buffer */
    size_t rd = fread(*buf, 1, s.st_size, f);
    if (rd < s.st_size)
        if (ferror(f))
            goto err_cleanup;

    /* file is read into the buffer, return the number of bytes read */
    rval = rd;
    goto cleanup;

err_cleanup:
    rval = -1;
    if (*buf != NULL)
        free(*buf);
    *buf = NULL;
cleanup:
    if (f != NULL)
        fclose(f);
    return rval;
}
```

We've now written a function with appropriate error checking that can be reused
throughout our Advent of Code exercises!