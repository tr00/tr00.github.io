---
title: implementing a judy array 3
date: 2023-04-25
---

## Coninuing with judy arrays

It's been over 4 months since I last touched judy arrays or anything else related to programming really.
Lot's has happened in my life, mostly sad things, but that's not point of this blog post.
Today I want to continue with my implementation of a judy array by writing a simple custom allocator for it.

## changes to the API

I've decided to ditch the `remove` operation because I don't need it 
for my intended use case and this allows me to make better tradeoffs.
Inserting NULL should be a good enough alternative for now because it effectively removes
any value but does not deallocate memory.

## why write a custom allocator?

A reasonable question with several answers:

- to have written a custom allocator
- to have better spacial locality 
- to gather profiling information

With the last answer to be the most important one. 
Judy is trying to be a space-efficient associative data structure 
and the only way to confirm that is by instrumenting the allocator.

For now I will be only measuring the total number of allocations and
the total number of bytes allocated. 
But maybe in the future it is important to be able to look further under the hood.

## allocation strategy

There are quite a bunch of allocations strategies to choose from so I should first figure out
what I want from my allocator. What are my criteria, what are my constraints?

Well, I don't know... so far I am allocating 2048B large nodes which live as long as the
entire data structure and since I removed the remove operation they never get deallocated.
But since there are going to be other types of nodes with different sizes... 
I don't know what my criteria are yet.

So I choose the simplest allocator, a stack allocator, backed by mmap. 
I will (hopefully) revisit the allocator in the future when I got a better picture of
what kind of allocator is suitable for judy's demands.


Ok, here a brief but complete description of how it works.

1. get a page from the operating system with mmap
2. advance page pointer by size after each allocation
3. when end of page is reached get a new page

All pages are stored in a linked list and the first 16 bytes of each page are
used to store the linked list node and the offset/remaining size.
We deallocate all at once by traversing the list and unmapping page after page.

I don't want to dump the code here because nothing really fancy is happening but
here is a snippet of how to get a page from the operating system.

```C
#include <sys/mman.h>

// ...

static void *get_page_from_os()
{
    int size = 4096;
    int prot = PROT_READ | PROT_WRITE;
    int flag = MAP_PRIVATE | MAP_ANON;

    void *page = mmap(NULL, size, prot, flag, -1, 0);

    if (page == MAP_FAILED)
    {
        fprintf(stderr, "[error]: failed to mmap page from os!\n");
        exit(1);
    }

    return page;
}
```

The next step is to make judy use this allocator and write a nice benchmark to
find out how much memory is actually wasted and how much we safe with the tiny node.

## benchmarking trie's memory usage

There are probably still a few bugs hiding somewhere because I've not tested anything yet but
I wrote little script which reads line by line from a file and inserts whatever it finds into
a judy array and when it's done it prints out how many allocations occured and how many bytes
were allocated.

I found a [list of english words](https://github.com/dwyl/english-words) and downloaded it for
my test data. The file is 4MB large and thus to big for unoptimized tries so I just took the first ~150 lines.
This test data is far from representable for any actual use case but will do for now.

| node types | # allocations | # bytes |
|------------|:-------------:|:-------:|
| trie       |           332 |  679936 |

Half a megabyte is quite a lot for storing merely 150 words but now I have the foundation ready to create new nodes and compare them.