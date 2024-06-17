---
title: GSoC Week 3
tags:
  - GSoC
categories:
  - GSoC 2024
date: 2024-06-17 13:54:45
---

## What I Have Done

This week, I focus on handling the reviews from my mentors and Junio. My mentors have put some comments on the [PR](https://github.com/shejialuo/git/pull/2) before I send the patch to the mailing list.

### Reviews From Junio

There are many important reviews from Junio. And we have discussed some issues:

1. The dangling symref should not be reported by `git-fsck(1)`. There is a common situation where people would use dangling symref to create the pointee ref. Like Junio said:
  ```sh
  git symbolic-ref refs/heads/maint refs/heads/maint-2.46
  git branch maint v2.46.0
  ```
2. We should change `fsck_files_symref` to `fsck_symref_target`. Because we are checking the target not the symref.
3. The use of `unsigned int *trailing` is not proper which will lose a lot of information for the caller.
4. The current implementation should check the symbolic link "escape" situation.
5. Some error information will cause misleading.

### The Problems I Found

However, I have still found some problems in the current implementation:

1. When checking symref target, the `referent` buf could be end with `'\n'`. If we simply pass the `referent` to `check_refname_format` function, we will get incorrect behavior.
2. The current implementation totally ignores the absolute symbolic link situation.
3. The current tests lack testing for symbolic links.I wanna make sure that the symbolic link check will work especially in Windows. I am a little worry about the compatibility.

## Next Plan

So in order to enhance this patch, I will do the following things:

1. Change the defined fsck messages and drop the `danglingSymref`.
2. Enhance the checks for symbolic link and add the corresponding tests.

## Challenges

This week, I spent most of my time addressing reviews. The biggest challenge I encountered was how to reasonably explain my intentions and design ideas when facing others' reviews. Additionally, understanding the principles and motivations behind the reviews was also challenging. Rather than calling it a challenge, I would say it was a very interesting experience.
