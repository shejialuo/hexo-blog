---
title: GSoC Week 6
tags:
  - GSoC
categories:
  - GSoC 2024
date: 2024-07-08 21:38:41
---

## What I Have Done

In this week, I mainly focus on solving the reviews.

+ [Patch v6](https://lore.kernel.org/git/ZoLHtmOKTfxMSxvw@ArchLinux/): In this version, I decide to reduce the complexity. However, I still misunderstood what Junio said, I still explicitly extracts two options `fsck_refs_options` and `fsck_objects_options` here. In my view, I suppose we could make the code more clean. However, Junio has commented as the following:
  > Just like premature optimization is bad, premature factoring and over-modularization is bad.
+ [Patch v7](https://lore.kernel.org/git/ZoVX6sn2C9VIeZ38@ArchLinux/): In this version, I just add a new ref-related field `verbose_refs` into the `fsck_options` without any over-modularization.
+ [Patch v8](https://lore.kernel.org/git/ZovqY4vQnQBAs7PH@ArchLinux/): Some commit messages in Patch v7 are not clear which makes the reviewer confused. Enhance the commit messages in this version.

## Next Plan

I guess this series should be OK. After setting up the infrastructure. I decide to add packed-refs consistency checks with the following steps:

1. Get a list of checks that packed-refs should checks, discussing with my mentors and the community.
2. Implement the code.

## Challenges

Since becoming a full-time employee, time has become a bit tight. I will manage my time more efficiently.
