---
title: GSoC Week 2
tags:
  - GSoC
categories:
  - GSoC 2024
date: 2024-06-10 14:01:39
---

## What I Have Done

This is the second week of writing code. I spend most of my time reading the `git-fsck(1)` source code and enhance my first-version patch.

### Understand Fsck Message

As Junior has commented:

> How well does this interact with the fsck error levels (aka fsck_msg_type), by the way?  It should be made to work well if the current design does not.

So the first thing I do in this week is to understand "fsck message". It takes me a lot of time here.

### Implement the Code

After understand how fsck messages works, I continue to start coding. I mainly do the following things like this [PR](https://github.com/shejialuo/git/pull/2) shows:

+ add refs check interfaces to interface with fsck error levels.
+ set up ref consistency check infrastructure.
+ add verify subcommand for builtin `git-refs`.
+ add git-refs verify child process for `git-fsck(1)`.
+ add unified interface for refs scanning for files backend.
+ add ref name check for files backend.
+ add ref content check for files backend.

However, due to the `git-refs` has not been merged into the `next`. I'll send the patches to the mailing list until everything is OK.

## Next Plan

I need to understand the `git-worktree`. The current implementation totally ignores worktree.

## Challenges

This week, I have encountered numerous challenges. The most difficult part for me has been designing the code.

In the process of integrating the fsck message into the reference consistency check, I encounter numerous design issues. Should I add two new fields directly to `fsck_options`, or should I create a new structure and reuse existing functions through polymorphism? I ponder this for a long time and ultimately choose the latter to ensure semantic clarity.

Additionally, I need to name the new fsck messages, which also takes considerable thought. Since I have to implement a new feature myself, coding is not the most complex part; rather, it is the design. How to design it in a way that makes both the semantics and the code clearer takes up a significant amount of my time.
