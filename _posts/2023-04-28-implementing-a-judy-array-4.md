---
title: implementing a judy array 4
date: 2023-04-28
---

## adding the tiny node (again)

I spent the last 2 days debugging the code. Mainly insertion is causing troubles since it's not that trivial. I have to iterate over the key string and follow the trie down until
we either reach the end of string or a leaf node. However I still have to insert the '\0' because otherwise the trie could not contain e.g. "a" and "aa" at the same time. 
And all the nodes have multiple states they could be in, like;

- I already contain that key you want to insert right now...
- I don't contain the key but I got some space left...
- If you really want me to take this key then I need to be promoted to a bigger node type...


For the trie node all of this is simple but this leads to me thinking the api can be as simple but then I think of other nodes which would complicate the api again. So instead of inventing and complying to a superficial internal node interface, I will just write code that does what it's supposed to and if code duplication occurs I can still move that into it's own function.

Anyways, I've added the tiny node and here are the results:


| node types | # allocations | # bytes | improvement |
|------------|:-------------:|:-------:|:-----------:|
| trie       |           332 |  679936 |          1x |
| trie & tiny|           345 |   30016 |       22.7x |

Besides the trie only version being terrible memory-wise, 
this new node type makes the entire thing 22x smaller which is huge.
A little disclaimer, as always, the data used are just ~150 english words 
all starting with 'a'. 
The improvements would surely look different depending on the workload.
Well, we can't degrade memory usage by adding a tiny node which only fits under
some conditions but this could be a very good test case for the tiny node.
Accidental cherry-picking can happen.

We can also see that there are 13 additional allocations. There aren't more nodes in the judy array but those are the cases where tinys got promoted to trie nodes.

## explaining the internals of tinys

Conceptually speaking, the trie nodes associates chars (keys) to pointers (child nodes).
However, the trie is very wasteful because 
it allocates 256 pointers even with only a single entry being used.

So the simplest improvement is to add a linear node which stores N keys and N nodes.
And if we ever try to insert more than N elements, it promotes itself into a proper trie node.
So how do we choose N?

It would prolly make sense to create one small, one medium and one big linear node to cover most scenarios. 
Here is example code for a generic linear node.

```C
#define N /* ??? */

typedef struct
{
    int len;
    uint8_t keys[N];
    void * nodes[N];
} node_t;

void * node_lookup(node_t *node, uint8_t cc)
{
    for (int i = 0; i < len; ++i)
        if (node->keys[i] == cc)
            return node->nodes[i];

    return NULL;
}
```

I could collect some more benchmarks and just find out which Ns work best.
Or remind myself that I'm writing this for a specific platform and can optimize for that.


### Step 1: Cache Line Optimization

I decided to start with the smallest node because it's the polar opposite of trie nodes. Cache behaviour is absolutely critical today, so let's try and fit this into a single cache line which is 64B on my target platform.

We will have to make N < 8 because `8 * sizeof(void *)` is already 64 bytes just for the child nodes. Let's try N=7.

```C
typedef struct
{
    uint8_t keys[7];
    uint8_t len;
    void * nodes[7];
} tiny_t;
```

This fits perfectly into a single cache line and we can use a uint8 as length because we are storing at most 7 elements. In fact, we could just use 3 bits as length if we needed the other 5 for something else.


### Step 2: Loop Unrolling

Us knowing N ahead of time enables us to do some optimization like fully unrolling
the loop from earlier.

```C
void *tiny_lookup(tiny_t *tiny, uint8_t cc)
{
    switch (tiny->len)
    {
    case 7: if (tiny->keys[6] == cc) return tiny->nodes[6];
    case 6: if (tiny->keys[5] == cc) return tiny->nodes[5];
    case 5: if (tiny->keys[4] == cc) return tiny->nodes[4];
    case 4: if (tiny->keys[3] == cc) return tiny->nodes[3];
    case 3: if (tiny->keys[2] == cc) return tiny->nodes[2];
    case 2: if (tiny->keys[1] == cc) return tiny->nodes[1];
    case 1: if (tiny->keys[0] == cc) return tiny->nodes[0];
    }

    return NULL;
}
```

This chunk of code may look nice but is every branch predictors nightmare. 
These branches are not unpredictable but we can maybe get rid of them entirely.
From a theoretical efficiency perspective this code is still optimal because it does minimal work.

### Step 3: Branchless Programming

I have to admit this code is getting trickier and trickier. 

```C
void *tiny_lookup(tiny_t *tiny, uint8_t cc)
{
    int index = -1;

    // assuming all keys are unique
    index += 1 * (tiny->keys[0] == cc);
    index += 2 * (tiny->keys[1] == cc);
    index += 3 * (tiny->keys[2] == cc);
    index += 4 * (tiny->keys[3] == cc);
    index += 5 * (tiny->keys[4] == cc);
    index += 6 * (tiny->keys[5] == cc);
    index += 7 * (tiny->keys[6] == cc);

    return ((uint8_t)index < tiny->len) ? tiny->nodes[index] : NULL;
}
```

Anyways, now we are down to a single branch to check bounds. 
Unfortunately, this code is comparing every entry and no longer returning early.


### Step 4: Instruction Level Parallelism

We can overcome this flaw by just computing all the comparisons simultaneously.
Okay, let me explain what's going on:

1. create a 8-wide vector containing the keys of tiny
2. create another vector containing cc repeated 8 times
3. compare all the entries and store the results in a third vector
4. convert the last vector into an 8bit bitmask
5. return NULL if none of the comparisons were true
6. else count trailing zeros (ctz) of the mask which gives us the index

```C
#include <xmmintrin.h>

void *tiny_lookup(tiny_t *tiny, uint8_t cc)
{
    __m64 v0, v1, v2;

    v0 = _m_from_int64(*(uint64_t *)&tiny->keys);
    v1 = _mm_set1_pi8(cc);
    v2 = _mm_cmpeq_pi8(v0, v1);

    uint64_t res = _mm_movemask_pi8(v2);

    if (!res) return NULL;

    return tiny->nodes[__builtin_ctz(res)];
}
```

There are still some technicalities, namely, we are ignoring the length and comparing with a 8 element vector when we only have 7 keys.

### Step 5: More Bit Hacks

We could add another branch and check for length 
but a better approach is to create a bitmask from the length and mask off all the false positive comparisons.
Something like `_mm_movemask_pi8(v2) & (0xff >> (8 - len));` could do the job I think.

However, we can even save those last couple of bit operations by not storing the length but instead storing this mask.

When inserting a key we now have to do `tiny->mask |= 0x01 << idx` but that's all it takes to keep up this invariant.

The actual code of lookup looks like this:

```C
void *judy_lookup(judy_t judy, const uchar *key)
{
    judyptr_t node = judy->root;

    while (1)
    {
        switch (TAG(node))
        {
        case LEAF: return (void *)node;
        /* ... */
        case TINY:
        {
            tiny_t *tiny = (tiny_t *)PTR(node);

            __m64 v0 = _m_from_int64(*(uint64_t *)&tiny->keys);
            __m64 v1 = _mm_set1_pi8(*key);
            __m64 v2 = _mm_cmpeq_pi8(v0, v1);

            uint64_t res = _mm_movemask_pi8(v2) & tiny->mask;

            if (!res) return NULL;

            node = tiny->nodes[__builtin_ctz(res)];

            ++key;

            break;
        }
        /* ... */
        }
    }

    __builtin_unreachable();
}
```

## Conclusion

That's pretty much it for the tiny node. 
The initial version of the tiny node would have sufficed to save 22x memory but this improves performance over the naive version.
I'm leaving out insertion because it's the same tricks as lookup but more ugly code.
right now there is a big spike in memory wasted when tiny nodes get promoted.
So maybe i'll create linear node which takes up 2 cache lines next or something similar.