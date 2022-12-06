---
title: implementing a judy array 2
date: 2022-12-06
---

## Introduction

In my [last](https://tr00.github.io/2022/12/05/implementing-a-judy-array.html) blog post I've implemented a trie which is the conceptual basis of a judy array.

```C
typedef uintptr_t JP;

typedef struct TRIE
{
    JP nodes[];
} judy_trie_t;
```

A trie node is very memory wasteful so judy introduces several new node types which are compressed in some form or only hold a limited number of child nodes.
In order to distinguish which pointer is pointing to which type of node we are using [tagged pointers](https://en.wikipedia.org/wiki/Tagged_pointer).

These tagged pointers are called jp's or judy pointers in this context. In the [official](https://judy.sourceforge.net/doc/shop_interm.pdf) implementation they take up the size of 2 pointers but I think i can get away with making jp's only 64bits large.

## tagged pointers

Tagged pointers, in case you never heard of them, are just normal pointers but they use some unused bits in the pointer to store additional information, in this case the type of node.

On 64bit machines allocations are usually required to be  aligned on a 8 byte boundary. I think most malloc implementations even align on a 16 byte boundary.
This means the 3 lowest order bits are always 0, which gives us the opportunity to store the type in those 3 bits as long as we zero them out before trying to derefence our pointer.

I'll only be using those 3 bits for now because 2^3 = 8 types should be enough. However currently only 48bits are used as virtual address space, so technically we could use the two highest order bytes as well.
And most linux machines split those 48bits in the middle into kernel address space and user address space.

So if we ever need to we can get up to 21 bits for meta data.

```C
#define JUDY_MASK_PTR 0xfffffffffffffff8ull
#define JUDY_MASK_TAG 0x0000000000000007ull

typedef uintptr_t JP;

#define typeof(ptr) ((ptr)&JUDY_MASK_TAG)
#define decode(ptr) ((ptr)&JUDY_MASK_PTR)

JP encode(void *node, uintptr_t tag)
{
    uintptr_t ptr = (uintptr_t)node;

    assert((ptr & JUDY_MASK_TAG) == 0);
    assert((tag & JUDY_MASK_PTR) == 0);

    return (JP)(ptr | tag);
}

typedef struct JUDY
{
    JP root;
} judy_t;
```

Up to 8 types should suffice for now, so lets port what we got from last time. 


## the trie node

The types we got from last time are just two which were stored implicitly last time, trie nodes and leaf nodes.

```C
enum
{
    LEAF,
    TRIE,
};
```

### lookup

All the operations can be implemented as a kind of state machine.
For the lookup operation not much has changed except we have to decode the node on each step.

```C
void *judy_lookup(judy_t *judy, const uchar *key)
{
    JP node = judy->root;

    while (true)
    {
        // current char
        uchar cc = *key;

        switch (typeof(node))
        {
        case LEAF:
            return NULL;
        case TRIE:
        {
            judy_trie_t *trie = (judy_trie_t *)decode(node);

            node = trie->nodes[cc];

            break;
        }
        }

        if (!cc)
            return (void *)decode(node);

        ++key;
    }
}
```

Although this code is quite a bit longer than the one in the previous blog post, it still accomplishes the very same thing.
The difference is that now we explicitly differentiate between TRIE & LEAF node.

### remove

Removing a node is also the same as before but with a bit more logic to decode the type of a node.

```C
void judy_remove(judy_t *judy, const uchar *key)
{
    JP *node = &judy->root;

    while (1)
    {
        uchar cc = *key;

        switch (typeof(*node))
        {
        case LEAF:
            return;
        case TRIE:
        {
            judy_trie_t *trie = (judy_trie_t *)decode(*node);

            node = &trie->nodes[cc];

            break;
        }
        }

        if (!cc)
            break;

        ++key;
    }

    *node = encode(NULL, LEAF);
}
```

### insert

Insertion gets quite more complicated.

```C
void judy_insert(judy_t *judy, const uchar *key, void *val)
{
    JP *node = &judy->root;
```
We first have to traverse the trie down until the key is no longer contained.

```C
    while (1)
    {
        uchar cc = *key;

        switch (typeof(*node))
        {
        case LEAF:
            goto EXPAND;
        case TRIE:
        {
            judy_trie_t *trie = (judy_trie_t *)decode(*node);
            node = &trie->nodes[cc];
            break;
        }
        }

        if (!cc)
            break;

        ++key;
    }
```
The second step is to allocate new nodes for the remaining key.
```C
EXPAND:

    while (1)
    {
        uchar cc = *key;

        judy_trie_t *trie = (judy_trie_t *)calloc(1, sizeof(judy_trie_t));
        *node = encode(trie, TRIE);
        node = &trie->nodes[cc];

        if (!cc)
            break;

        ++key;
    }
```
And the final step is to just store the value as a leaf.
```C
    *node = encode(val, LEAF);
}
```
