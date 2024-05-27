---
title: GSoC-Community Bonding Period
tags:
  - GSoC
categories:
  - GSoC 2024
date: 2024-05-27 15:37:51
---


During the community bonding period, I engaged in various activities to familiarize myself with my mentors, the community, and the git codebase. I had video conferences with each of my two mentors, where we discussed not only how to quickly set up a basic infrastructure for reference consistency checks but also talked about our respective careers and technical exchanges. Additionally, we scheduled bi-weekly video meetings for ongoing synchronization.

At the same time, all mentors and contributors agreed to an online meeting on June 7th, allowing us to further familiarize ourselves with the community.

## Setup the Mutt to Improve Productivity

To enhance the efficiency of handling emails in the community later on, I plan to implement the use of mutt on ArchLinux before the official coding begins, and I have written a blog post called [Use Mutt in ArchLinux](https://luolibrary.com/2024/05/12/Use-Mutt-in-ArchLinux/) documenting the configuration process.

## Infrastructure Implementation

Before the official GSoC coding period begins, I was ready to set up the architecture for reference consistency checks as envisioned in the proposal. The core idea is to add a callback to `ref_storage_be`, then add the functionality to check if the ref name is correct for the file backend, and add code in the `fsck.c` file for testing.

```c
struct ref_storage_be {
  ...
  check_refs_fn *check_refs;
  ...
};
```

My mentors, Patrick and Karthik, have already commented on the corresponding [commit](https://github.com/shejialuo/git/commit/d4c33868502f7a74066cc8bce9c78941b68f58cb), with the main issues focusing on the following aspects:

1. The current commit should be split; the first commit should focus on the implementation of the infrastructure, and then the second commit should add the name check for the file backend.
2. Do not use refs code to enumerate through the refs here. The reason is that this function would hide broken references from us. Use `dir-iterator.h` to check `refs/`.
3. Suggestions for some naming changes.
4. The need to also check the refs of the worktree.

## Next Plan

I have now essentially completed the coding for the infrastructure. The subsequent work mainly involves addressing the comments from my two mentors on the current commit, then cleaning up the code, and reviewing the code through PRs in my own [repository](https://github.com/shejialuo/git/).

Secondly, I will maintain an email in the community to organize all updated blogs, making it convenient for the community to promptly follow my updates and provide me with feedback.
