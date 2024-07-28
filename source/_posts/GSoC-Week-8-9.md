---
title: GSoC Week 8-9
tags:
  - GSoC
categories:
  - GSoC 2024
date: 2024-07-28 23:31:46
---


## What I Have Done

In these two weeks, I mainly focus on solving the reviews. After I have sent the [Patch v11](https://lore.kernel.org/git/ZpPEdmUN1Z5tqbK3@ArchLinux/) to the mailing list. Karthik suggests that instead of using `ref_checkee` and `sub_ref_checkee`, we may use the following data structure to provide more extensibility here.

```c
struct fsck_refs_info {
    char *refname;
    union {
        struct {
            ...
        } reftable;
        struct {
            ...
        } files;
       } u;
}
```

So in [patch v12](https://lore.kernel.org/git/ZpuCg1GL1YE_sJBP@ArchLinux/), I implement the above data structure.

However, Patrick has a different opinion about this. Patrick thinks this is too complicated. We should make thing simple here. Instead of using a union to represent the options, why we not just use a flat structure here. So I just use the following data structure:

```c
struct fsck_refs_info {
    const char *path;
}
```

It's simple and we reserve the extensibility here. The code is under review in the [github](https://github.com/shejialuo/git/pull/11). I have already solve the review from Karthik. However, due to the reason that Patrick has just come back to Office. I am waiting for the reviews from Patrick. When everything is OK, I will send the Patch v13 to the mailing list.

What a long journey!

## Next Plan

Continue to handle the reviews.
