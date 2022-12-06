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

typedef struct JUDY_TINY_NODE
{
    JP nodes[256];
} judy_trie_t;
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

## What have we achieved?

Well, this refactoring didn't fix the memory bottleneck. 
It added a couple of bit tricks to encode & decode nodes and their types but no decrease in memory consumption.

## the tiny node

Let's change that by introducing a new node type. 
We can observe that for trie nodes with a low population, that is lots of NULL's and almost no child nodes, the memory waste is largest.

A trie node which contains only a single key has allocated 2KiB to store an 8B pointer and a bunch of NULL's. 
If we create a node type which takes only up to N keys and searches them with linear search, we can allocate this smaller node for small populations and swap it for a full trie node as soon as its capacity is exceeded.

Lets just choose N=8 because that seems resonable. A new node could look like this:

```C
#define N 8
typedef struct JUDY_TINY_NODE
{
    size_t size;
    uchar keys[N];
    JP nodes[N];
} judy_tiny_t;
```

The upsides of this approach is that we don't even need to linearly search the keys we can search them in parallel by using SIMD/SWAR tricks later and the memory consumption is now significantly smaller for low populations.
However the size of this node is 80. Which is not ideal for a mainly 2 reasons:

1. 80B leads to bad memory fragmentation (compared to e.g. 64B)
2. 2 cache misses if we access the end of the nodes array

But we can squish this struct into 64B (one cache line) if we decrease N to 7 and since the size can't exceed 7 we can use one byte to represent it.

```C
typedef struct JUDY_TINY_NODE
{
    uchar keys[7];
    uint8_t size;
    JP nodes[7];
} judy_tiny_t;
```
Now this node fits into 1 cache line, can still be searched in parallel but
the size got smaller by a single key. Well, the upsides heavily outweigh the downsides.

### lookup

We can now add a new node type to the state machine of the lookup function.

```C
switch (typeof(node))
{
...
case TINY:
{
    judy_tiny_t *tiny = (judy_tiny_t *)decode(node);

    int i = 0;

    // linear search
    for (; i < tiny->size; ++i)
    {
        if (tiny->keys[i] == cc)
        {
            node = tiny->nodes[i];
            break;
        }
    }

    // key not found
    if (i == 7)
        return NULL;

    break;
}
...
}
```
This is the point where we put on our low level wizard hat and start optimizing this node type. So far we have already optimized the structure to fit in a single cache line and have allocator friendly alignment. Now we can tackle this little algorithm here.

### Trick 1: loop unrolling

We know that the size is at most 7 which allows us to fully unroll this loop.
I am currently completely ignoring the size which will come clear later.

```C
    judy_tiny_t *tiny = (judy_tiny_t *)decode(node);

    if (tiny->keys[0] == cc)
    {
        node = tiny->nodes[0];
    }
    else if (tiny->keys[1] == cc)
    {
        node = tiny->nodes[1];
    }
    // ...
    else if (tiny->keys[6] == cc)
    {
        node = tiny->nodes[6];
    }
    else // key not found
    {
        return NULL;
    }

    break;
```

This may not look like superb software engineering but I would argue that this exactly is what software _engineering_ is supposed to be. Anyways, let's take a look at what we got: lots of branches.

### Trick 2: branchless programming

These branches exist because we currently assume that we have to compare the current char to the keys in order, but we know that a char is at most once inside the keys array.
Thus only one of those branches is going to turn out as true and the order is unimportant.
Let's refactor again keeping that in mind:

```C
    judy_tiny_t *tiny = (judy_tiny_t *)decode(node);

    int cmp0, cmp1, /* ... */ cmp6;

    cmp0 = tiny->keys[0] == cc;
    cmp1 = tiny->keys[1] == cc;
    // ...
    cmp6 = tiny->keys[6] == cc;

    int idx = 0;

    idx += cmp0 * 0;
    idx += cmp1 * 1;
    // ...
    idx += cmp6 * 6;

    int cmp = 0;

    cmp |= cmp0;
    cmp |= cmp0;
    // ...
    cmp |= cmp6;

    if (cmp)
    {
        node = tiny->nodes[idx - 1];
    }
    else
    {
        // key not found
        return NULL;
    }

    break;
```

This looks much better. We have decreased the number of branches from 7 to 1 but we are still missing the last piece of the puzzle...

### Trick 3: instruction level parallelism

Right now we are computing the same instruction with different data 7 times in suggession. 
This is known as instruction level parallelism.
We can make use of it by using a single wider instruction to compute the same thing but in parallel.
I will rely on some mmx instructions which limits portability but I'm sure there are equivalent instructions for arm neon and otherwise the previous version has to do the job.

Okay but instead of just dropping the code here I will explain it step by step because it becomes increasingly tricky.

When we encounter a tiny node (<=7 keys) we first fetch it's data with a single cache line fill.
We move the 7 keys (56 bits) into a single 64bit xmm register. The last 8 bits of are the size but we will treat it as an 'dont-care' value and mask it out in the end.

```C
    judy_tiny_t *tiny = (judy_tiny_t *)decode(node);

    //
    // | k0 | k1 | k2 | k3 | k4 | k5 | k6 | sz |
    //
    __m64 vec = _m_from_int64(*(uint64_t *)&tiny->keys);
```

We also fill another xmm register with 8 repetitions of the currently
decoded char.

```C
    //
    // | cc | cc | cc | cc | cc | cc | cc | cc |
    //
    __m64 key = _mm_set1_pi8(cc);
```

Next we compare those two vectors in parallel.
There are only two outcomes, either a single byte is set or none at all.

```C
    //
    // | 00 | 00 | ff | 00 | 00 | 00 | 00 | -- |
    //
    __m64 cmp = _mm_cmpeq_pi8(key, vec);
```

Previously I've ignored how we deal with potentially reading beyond the size,
but now we can simply mask off the keys which are not actually there.

```C
    // <          size          >
    // | ff | ff | ff | ff | ff | 00 | 00 | 00 |
    //
    uint64_t msk = (uint64_t)-1 << ((8 - tiny->size) << 3);

    //
    // | 00 | 00 | ff | 00 | 00 | 00 | 00 | 00 |
    //
    uint64_t res = _m_to_int64(cmp) & msk;
```

And now we can compute the index by counting the zeros infront of the set byte.

```C
    int idx = __builtin_clzll(res) >> 3;
```

Let's put it all together:

```C
    judy_tiny_t *tiny = (judy_tiny_t *)decode(node);

    __m64 vec = _m_from_int64(*(uint64_t *)&tiny->keys);
    __m64 key = _mm_set1_pi8(cc);
    __m64 cmp = _mm_cmpeq_pi8(vec, key);

    uint64_t msk = (uint64_t)-1 << ((8 - tiny->size) >> 3);

    uint64_t res = _m_to_int64(cmp) & msk;

    if (!res)
        return NULL;

    node = tiny->nodes[__builtin_clzll(res) >> 3];

    break;
```

All of this effort may seem like a lot for comparing 7 bytes and returning either NULL or another node but these 10 over-engineered lines of code are in the hot path of our data structure.
This code is likely to get executed on every lookup quite possibly multiple times so each branch we removed will pay off.

The last branch is necessary but inherently unpredictable or at least very data dependent. We can't do much about it, but let's take a step back and look what we have achieved.

Our goal was to create a new node which should drastically reduce memory waste for nodes with a tiny population.

We have kind of reached that goal, but only for populations of less than 8.
This leaves us with a huge spike in memory waste as soon as we pass that threshold.
This can be solved by other nodes in the future...