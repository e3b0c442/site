---
date: 2019-09-23
title: In which Nick creates a hash table
tags: ["Data Structures"]
---

_The code discussed here can be found [on Github](https://github.com/e3b0c442/esht)_

Before we tackle Day 7, we need to talk a bit about data structures. Choosing the proper data structures for your program's data is extremely important in making your application both performant and easily understood. 

As we will see in Day 7's discussion, we will need to be able to access an item by a provided string key. Storing these items in a linked list or array would work, but would be unnecessarily slow as the array would need to be iterated each time we try to find the item we are searching for. While there are ways to optimize this, such as by using binary search on a sorted array, these still only achieve an average time complexity of O(log(n)).

The solution in this case is to use a hash table. Hash tables allow for a quick access given a piece of data (generally a string) as a key. This is accomplished by using a hashing algorithm to convert the key into a numerical value which can then be used as an array index. Given a well designed hash table, average search time can be constant time.

Unfortunately, the C standard library comes with neither a hash table implementation nor a hash algorithm. POSIX defines a hash table, but it is defined and stored globally, making it effectively _non-reentrant_. While this is of only a minor concern for us at this time, it also means that only one such table can be defined at a given time, which will likely be an issue in all but the most simple programs. GNU provides an extension which contains reentrant versions of the POSIX algorithm, but it is not available on even all UNIX platforms, most notably macOS.

For these reasons, we will take some time and thought in this post to design and construct a hash table that gets around these limitations. We will have two primary design motivations:

* The table will be universal, so it must compile on standard C. I'm going to take this a step further and make it valid ANSI C.
* The table will be simple for end users. We will make a clear demarcation of responsibility for memory; each time the user adds something to the table, the key and value are copied, and the hash table owns those copies. When an item is retrieved, it is copied out. Therefore the end user is responsible for freeing all items provided to or retrieved from the table, but _not_ the copies in the table.

Regardless of platform, there are two important design decisions that need to be made when designing a hash table:

* **Hashing algorithm**. The choice of hashing algorithm is very important. If a slow algorithm is used, the speed benefits of the hash table are reduced as the time to calculate hashes is high. If an algorithm with poor distributions are used, there will be additional collisions and therefore additional need to iterate over the hash buckets to find the requested item. Realistically, we need a hash algorithm which will produce a hash that fits in a standard numerical data type.
* **Collision handling**. Even with a proper hash algorithm, unless you are using the full value of a modern cryptographic hash (Hint: don't do this, they are slow and unnecessary, and you can't use the full value anyway!), you will inevitably have collisions -- two identical keys which produce the same hash digest. There are two common ways to handle this: using _chaining_, i.e. each hash key points to an array or linked list instead of a single item; or by using _open addressing_, in which we make the primary array larger and, if an item already occupies the hash slot, iterating along the array until an open position is found.

For our hash table, we will use Daniel J. Bernstein's DJB2a algorithm. This algorithm is simple and extremely fast, yet has reasonably good distribution. To handle collisions, we will use chaining with a singly-linked list. This reduces the complexity of adding and removing items from the chain with little to no performance penalty over an array iteration.

In order to implement our hash table, we need to handle five different operations:

* Allocating and initializing the table
* Inserting items into the table (or updating items already in the table)
* Searching for items in the table
* Deleting items from the table
* Destroying the table when we are done using it.

Depending on one's use case, there may be a desire to separate the _adding_ and _updating_ functions. As we are trying to keep things simple though, we will keep them as one function under the assumption that if an item exists for the key, it will be updated.

To start things off, we will need to create structures for the data in the table. In addition to the data itself, we need to track the key used to access the data and, since we will not be assuming a data type, the size of the data. In addition, since we are using linked-list chaining to handle collisions, we will need to have a pointer to the next item in the chain.

```c
typedef struct esht_entry
{
    char *k;                /* key */
    void *v;                /* value */ 
    size_t l;               /* size */
    struct esht_entry *n;   /* pointer to next item in chain */
} esht_entry;
```

With this structure defined, we can then define the structure for the table. This structures tracks both the current number of entries in the table, and the current size of the array backing the table, as well as the array itself. The size and capacity are used to calculate the _load factor_. Generally speaking, the load factor of a hash table should be kept below 80% to ensure minimal collisions, thus keeping performance high.

```c
struct esht
{
    size_t len;             /* current size */
    size_t cap;             /* current size of backing array */
    esht_entry **entries;   /* array of pointers to entries */
};
```

With these structs defined, we now have a place for all of the information that needs to be stored as part of our hash table. Now, we want to define our hash function. As mentioned before, this is Daniel J. Bernstein's DJB2a algorithm:

```c
unsigned long esht_hash(char *str)
{
    unsigned long hash = 5381;
    int c;

    while ((c = *str++))
        hash = ((hash << 5) + hash) ^ c;

    return hash;
}
```

Now we can get down to the business of allocating and initializing the hash table. For simplicity's sake, we are going to design our table with an initial capacity of 1. We will allocate the memory for the table struct, set the capacity and length, and allocate the memory for the backing array. We will check for errors after each allocation and return NULL on error, or return a pointer to the table struct if everything goes as planned. Note that we will use an array of pointers to item structs, rather than making an array of item structs themselves; this will simplify the addition and removal of items later on. We also use `malloc` to allocate the table, but `calloc` to allocate the array. This is because we will explicitly initialize the table members, but the array will just have `NULL` pointers to start since there are no items. (The value of a `NULL` pointer is `0`).

```c
esht *esht_create()
{
    esht *table;

    table = malloc(sizeof(esht)); 
    if (table == NULL)
        return NULL;

    table->cap = 1;
    table->len = 0;
    table->entries = calloc(1, sizeof(esht_entry *));
    if (table->entries == NULL)
    {
        free(table);
        return NULL;
    }

    return table;
}
```

Now that we have a table, we can start doing operations on it. However, there is one common piece of code that can be factored out, and that is the _resize_ operation, which will resize the backing array and redistribute the elements.

A bit of explanation: In order to not consume an inordinate amount of memory (e.g. 2**32 * sizeof(esht_entry *)), we don't actually use the full hash value, but the modulus of the hash value against the capacity of the array. The size of the array then becomes the second operand of the modulus. The side effect of this is that the array lookup index will change any time the array is resized, so we need to recompute the lookup index by re-hashing and modulizing the key.

What this means is that you want to be judicious with resizing, as there will be a performance hit while the entirety of the table's keys are hashed again. For our table, we are setting a maximum load factor of 0.75 and a minimum load factor of 0.25, and then either doubling or cutting in half the size once those limits are breached. This allows for a bit of hysteresis, so that if there are several insert or delete operatinos close to either limit, the table wouldn't be resized on every insert/delete. If we wanted to simplify further, we could remove the minimum load factor and only allow the table to grow, but this potentially causes a memory leak.

```c
int esht_resize(esht *table, size_t new_cap)
{
    /* declare variables at the beginning to conform with ANSI C */
    int i, ii;
    esht_entry *cur, *next, **old;

    /* allocate and zero out a new backing array */
    esht_entry **new_entries = calloc(new_cap, sizeof(esht_entry *));
    if (new_entries == NULL)
        return 1;

    /* iterate over the old array... */
    for (i = 0; i < table->cap; i++)
    {
        /* ...and then over the chains, recompute the hash index for each item, 
           and insert the item into the new array */
        next = table->entries[i];
        while (next != NULL)
        {
            cur = next;
            next = cur->n;

            ii = esht_hash(cur->k) % new_cap;
            cur->n = new_entries[ii];
            new_entries[ii] = cur;
        }
    }

    /* replace the old array with the new array, and free the old array */
    old = table->entries;
    table->entries = new_entries;
    table->cap = new_cap;
    free(old);

    return 0;
}
```

One additional factored-out operation is finding the value for a key. This is used both by our searching and adding (read: updating) operations, and returns a pointer to the table entry for either updating or copying out a value to return.

```c
esht_entry *esht_get_entry(esht *table, char *key)
{
    /* declare variables for ANSI C conformance */
    unsigned long i;
    esht_entry *e;

    /* hash the key and perform a modulus against the table capacity to get the
       lookup index */
    i = esht_hash(key) % table->cap;

    /* iterate the chain until a matching key is found and return the entry */
    e = table->entries[i];
    while (e != NULL)
    {
        if (!strcmp(key, e->k))
            return e;
        e = e->n;
    }

    /* if no entry is found, return NULL */
    return NULL;
}
```

Now we can get down to the meat and potatoes: adding (updating), finding, and removing items from the table. First, we will write our insertion function. This function searches for an existing item with the same key and updates it; absent an existing item for the key, it allocates a new item and inserts it into the table. Finally, if the insertion causes the load factor to be exceeded, the table is resized.

```c
int esht_update(esht *table, char *key, void *value, size_t len)
{
    /* declare variables for ANSI C conformance */
    unsigned long i;
    esht_entry *e;
    void *v, *k, *old;

    /* find an existing entry for the key... */
    e = esht_get_entry(table, key);
    if (e != NULL)
    {
        /* allocate memory for and copy the value */
        v = malloc(len);
        if (v == NULL)
            return 1;
        memcpy(v, value, len);

        /* replace the value, update the value size, and free the old value */
        old = e->v;
        e->l = len;
        e->v = v;
        free(old);

        return 0;
    }

    /* ...or allocate memory for a new entry */
    e = malloc(sizeof(esht_entry));
    if (e == NULL)
        return 1;

    /* allocate for and copy the value */
    v = malloc(len);
    if (v == NULL)
    {
        free(e);
        return 1;
    }
    memcpy(v, value, len);

    /* allocate for and copy the key */
    k = malloc(strlen(key) + 1);
    if (k == NULL)
    {
        free(e);
        free(v);
        return 1;
    }
    strcpy(k, key);

    /* set the entry variables */
    e->k = k;
    e->v = v;
    e->l = len;

    /* hash the key and perform a modulus against the table capacity to get the
       lookup index */
    i = esht_hash(key) % table->cap;

    /* insert the item into the table and update the size */
    e->n = table->entries[i];
    table->entries[i] = e;
    table->len++;

    /* if the load factor has been exceeded, resize the backing array */
    if ((float)table->len / (float)table->cap > ESHT_MAX_FACTOR)
        if (esht_resize(table, table->cap * 2))
            return 1;

    return 0;
}
```

We've already written most of the logic for finding an item; we'll use that to find the item, then copy the value out to the end user. For end-user friendliness, we return a pointer to the value and not an entry struct. If the caller needs to know the size of the data (i.e. it is not a `NULL`-terminated string or known data size), they can pass a pointer to a `size_t` value, in which we can place the size of the data.

```c
void *esht_get(esht *table, char *key, size_t *len)
{
    /* declare variables for ANSI C conformance */
    esht_entry *e;
    void *r;

    /* find an existing entry. If there is no entry matching the key, set the 
       value of the length pointer to 0 and return NULL */
    e = esht_get_entry(table, key);
    if (e == NULL)
    {
        if (len != NULL)
            *len = 0;
        return NULL;
    }

    /* allocate memory for and copy the value of the table entry */
    r = malloc(e->l);
    if (r == NULL)
    {
        if (len != NULL)
            *len = 0;
        return NULL;
    }
    memcpy(r, e->v, e->l);

    /* if a valid pointer was provided, set the length of the return data, then
       return a pointer to the copied value */
    if (len != NULL)
        *len = e->l;
    return r;
}
```

Now, we will provide a way for end users to remove items from the table that are no longer needed. This function takes the key as an argument and returns `1` if no item was found, or `0` if the item was found and successfully removed. Then the table is resized if the load factor exceeds the minimum value.

```c
int esht_remove(esht *table, char *key)
{
    /* declare variables for ANSI C conformance */
    unsigned long i;
    esht_entry *e;

    /* hash and modulus the key. Note: we are not reusing the esht_get_entry
       function here because we need to know the previous item in the chain */
    i = esht_hash(key) % table->cap;
    e = table->entries[i];

    /* if the initial lookup is NULL, the item does not exist */
    if (e == NULL)
        return 1;

    /* otherwise, check the first key, and if the key matches, adjust the chain */
    if (!strcmp(e->k, key))
    {
        table->entries[i] = e->n;
        goto cleanup;
    }

    /* If the first key did not match, iterate the table until it is found and
       adjust the chain */
    while (e->n != NULL)
    {
        if (!strcmp(e->n->k, key))
        {
            e->n = e->n->n;
            goto cleanup;
        }
    }

    return 1;

    /* This jump is only taken if a valid entry is found */
cleanup:
    /* Free the key, value, and entry struct */
    free(e->k);
    free(e->v);
    free(e);

    /* adjust the table size and resize if needed */
    table->len--;
    if ((float)table->len / (float)table->cap < ESHT_MIN_FACTOR)
        if (esht_resize(table, table->cap / 2))
            return 1;

    return 0;
}
```

We now have a table to which we can add, find, and remove items. Now, we need to be able to clean the table up. The destroy function iterates the backing array and chains, frees the key and value of every entry and then the entry itself, frees the backing array, then finally frees the table struct.

```c
void esht_destroy(esht *table)
{
    /* declare variables for ANSI C conformance */
    int i;
    esht_entry *e, *n;

    /* iterate the backing array... */
    for (i = 0; i < table->cap; i++)
    {
        /* and then the chains, freeing all keys, values, and entries */
        e = table->entries[i];
        while (e != NULL)
        {
            n = e->n;
            free(e->k);
            free(e->v);
            free(e);
            e = n;
        }
    }

    /* free the backing array and then the table itself */
    free(table->entries);
    free(table);
}
```

We now have an ANSI-C compliant, end-user-friendly hash table. The end user is  responsible for managing items provided to or retrieved from the table, and is insulated from accidentally changing internal table values. The end user only needs to know about the table struct itself; the entry struct is internal-only and the find operation returns a pointer to a copy of the value with an optional length.

We will use this table to complete Day 7 in the next post. Until then, your
comments, criticisms, and inputs are appreciated in the comments.