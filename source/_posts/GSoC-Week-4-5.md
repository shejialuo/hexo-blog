---
title: GSoC Week 4-5
tags:
  - GSoC
categories:
  - GSoC 2024
date: 2024-06-30 23:50:51
---


## Personal Life

Due to the need to help prepare for my master's graduation work and to move to Shanghai to start my job at NVIDIA on July 1st, I have divided the GSoC records into two weeks.

## What I Have Done

In these two weeks, I mainly work on three tasks:

+ [Patch v4](https://lore.kernel.org/git/ZnKKy52QFO2UhqM6@ArchLinux): In this version, I correct the previous implementation that attempted to use pointer casting and adopt a composition strategy to design the data structure.
+ [Patch v5](https://lore.kernel.org/git/Zn4xeoqhCeBCSNvg@ArchLinux): In this version, Junio suggest that I should reuse the existing report code. However, I overcomplicate the issue by setting up two parallel data structures, which increases the complexity of the problem.
+ Completed the GSoC midterm evaluation. My mentors have given me some really useful advice, especially.
  > Find the balance between building something that can be easily extended in the future vs. something that is specific to the task at hand.

## Next Plan

The next task is still based on the composition structure of the data. Although Junio provided a data structure using a union, I still find it too complex. I have always wanted my implementation to be extensible, which has slightly deviated from the task I was supposed to do.

## Challenges

I feel that the biggest challenge over these two weeks has been understanding the reviews. Every time I submit changes based on a review, I tend to veer off course a bit. I feel that the reviewers' comments are aimed at addressing specific issues, while I always strive for a perfect implementation, which has led to a lot of unnecessary work and made the code more complex.

How to balance this is something I have decided to think about carefully.
