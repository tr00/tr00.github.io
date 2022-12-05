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
