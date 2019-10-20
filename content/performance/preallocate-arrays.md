---
title: "Preallocate Arrays"
section: "Performance"
menu:
  toc:
    parent: "performance"
    weight: 20
---

## Problem

Your code is performance sensitive and suffers from poor performance due to arrays being resized while in use.

## Solution

It's common for new Pony programmers to create arrays that use the default allocation:

```pony
// allocate space for no entries
let small: Array[U8] = Array[U8]
```

Instead, preallocate more space than you expect to use:

```pony
// preallocate space for at least 2048 entries
let not_so_small: Array[U8] = Array[U8](2048)
```

## Discussion

Arrays are an incredibly flexible data structure. Programmers can deploy them in situations where the amount of data needed will vary over time. If the array is too small and more space is needed to hold additional items, the array will be expanded so that it can hold more items.

This expansion of an array is often called "reallocating." In general, you want to avoid reallocating because:

- a new chunk of memory is allocated
- the contents of the old array are copied over into the newly allocated space

By preallocating more space than you need immediately, you are limiting future allocations and copying.

If you are interested in the particulars  `Array`'s resizing algorithm, you can check out [the source](https://github.com/ponylang/ponyc/blob/master/packages/builtin/array.pony).
