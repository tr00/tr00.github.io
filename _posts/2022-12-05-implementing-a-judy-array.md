---
title: implementing a judy array
date: 2022-12-05
---

## Introduction

A while ago I was browsing some wikipedia articles when I came across the article about [judy arrays](https://en.wikipedia.org/wiki/Judy_array).
The page was quite empty and had little information about how a judy array actually works. 
But it stated it's a high performance and low memory dictionary-like data structure which claims to be faster than AVL-trees, B-trees and SkipLists.
However according to wikipedia it is also very complicated, not portable, requires thousands of lines of code and all in all not worth the extra effort
implementing such a monstrocity.

Naturally, I was intrigued. I've read through the [10min technical description](https://judy.sourceforge.net/doc/10minutes.htm) 
which defined some terminology and claimed to outperform other more common data structures. 
This document wasn't as judgy about the complexity as the wikipedia article but after reading it, I still barely understood how judy works.

Well, how can I spent my time any better than reading through the [3h technical description](https://judy.sourceforge.net/doc/shop_interm.pdf) of judy arrays.
And this is where my journey begins.

My goal is to implement my own version of a judy array and see how far I can over-engineer it...

## API

Well, let's get started with the API.

```C
/**
 * finds the value associated with the '\0'-terminated string key.
 * returns NULL if it can't be found.
 */
void *judy_lookup(void *judy, const uchar *key);

/**
 * inserts the value into the judy array. 
 * the value must be a valid pointer and can't be NULL.
 */
void judy_insert(void *judy, const uchar *key, void *val);

/**
 * removes a previously insert value from judy.
 * if the key can't be found nothing happens.
 */
void judy_remove(void *judy, const uchar *key);
```

This API isn't final and there is some functionality still missing (e.g. initialization, finalization, iteration?) but I will figure it out along the way.

## A note about portability

This implementation assumes x86-64 linux with 64B cache lines, 4096B pages, SSE compatibility and 48bit virtual address space.
Or in other words, there is no portability which is a downside but... only if you care about portability.

## Tries

Tries also known as digital trees (and many more names) are basically 256-ary trees which encode a strings from root to leaf by decoding one char at each level.
A key is inside the trie if there exists a leaf after decoding the entire string.
There is a better explaination of [tries](https://en.wikipedia.org/wiki/Trie) on Wikipedia.
The concept behind tries is quite rudimentary. Judy arrays are fundamentally tries with lots of optimization tricks.
So in order to implement a judy array, let's implement tries first.

```C
typedef struct TRIE
{
    trie_t *children[256];
} trie_t;
```

A node in a trie contains 256 pointers to either child nodes or null.
At least in my implementation the terminating '\0' is also stored in the trie and the associated values are stored in the pointers after the '\0'.

However every node being 2048 bytes large is a HUGE memory bottleneck.
Especially if we have only a few keys inside a trie node, there will be
a lot of unused but allocated space.

### lookup

```C
void *lookup(trie_t *root, const uchar *key)
{
    trie_t *node = root;

    while (node != NULL)
    {
        node = node->children[*key];

        if (*key == '\0')
            return (void *)node;

        ++key;
    }

    return NULL;
}
```

### insert

When inserting a new key-value pair it consists of 3 steps:

1. follow the nodes down until the key is no longer contained
2. allocate new nodes as you descend further
3. and when the end is reached save the value as leaf node

```C
void insert(trie_t *root, const uchar *key, void *val)
{
    trie_t **node = &root;

    while (true)
    {
        if (*node == NULL)
        {
            *node = calloc(1, sizeof(trie_t));
        }

        node = &node->children[*key];

        if (*key == '\0')
            break;

        ++key;
    }

    *node = val;
}
```

### remove

Removing a value is very much similar to looking it up but in the end 
instead of returning the value it gets overwritten with NULL.
If the key is not contained, meaning a NULL node is encoutered on the way down,
we just happily return.

```C
void remove(trie_t *root, const uchar *key)
{
    trie_t **node = &root;

    while (*node != NULL)
    {
        node = &node->children[*key];

        if (*key == '\0')
            *node = NULL;

        ++key;
    }
}
```

## Conclusion

Tries are very memory wasteful but they can be usefull as a baseline for future benchmarks.
They are also helpful to understand how judy arrays are supposed to work if we ever get bogged down too deep in optimization wizardry.

That's it for my first blog post. I hope you enjoyed this introduction to judy arrays & tries. In my [next]() blog post I'll introduce the first steps to avoid this big memory overconsumption.