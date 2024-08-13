---
title: GSoC Week 10-11
tags:
  - GSoC
categories:
  - GSoC 2024
date: 2024-08-13 21:41:26
---


## What I Have Done

In these two weeks, I continue to solve the reviews. After I have sent the [Patch v13](https://lore.kernel.org/git/ZqeXrPROpEg_pRS2@ArchLinux/). Patrick gives an opinion here, we want to reuse the `fsck_vreport` function here. We want to make it generic but we design the callback not generic. And Patrick gives a wonderful idea here:

> A better design would likely be to make `error_func()` receive a `void*` pointer such that `error_func()` and then have the respective subsystems provide a function that knows to format the message while receiving either a `struct fsck_object_report *` or a `struct fsck_ref_report *`.

However, Patrick thinks we may put unnecessary effort here to make things complicated again. But after discussion, I think we should use the below design for the following reasons:

```c
typedef int (*fsck_error)(struct fsck_options *o,
                          void *info,
                          enum fsck_msg_type msg_type, enum fsck_msg_id msg_id,
                          const char *message);
```

1. We only expose one interface called `fsck_reportf` which will make the code clear. Actually, there is no different between reporting refs and reporting objects.
2. We provide more extensibility here, because we will never change `fsck_reportf` and `fsck_error` prototype when we want to add more info for either refs or objects.

And Patrick advices me that I should drop the last patch "[PATCH v13 10/10] fsck: add ref content check for files backend", because we should speed up the review process due to the ddl of GSoC.

And Junio gives two advices:

1. The prototype of `files_fsck_refs_fn` should adapt to the Patrick's new change.
2. Unless the most common use of an array is to pass it around as a collection of items and operate on the collection, it is a better practice to name an array with a singular noun.  Name the array as `fsck_refs_fn[]` not `fsck_refs_fns[]`.

I solve these problems in [Patch v14](https://lore.kernel.org/git/ZqulmWVBaeyP4blf@ArchLinux/):

1. By following the advice from Patrick, we should make the callback function be generic by adding only one `void *fsck_report` parameter. Thus the commit sequence will be much more clearer. And it wll be much easier for reviewers to review. And I have split the commit into more commits in this version.
2. Enhance the commit messages to provide more context about why we should do this.
3. Patrick advices that we should initialize the `fsck_options` member when parsing the options. However, because the original `strict` and `verbose` field are defined as the bit field, we cannot take the address of them. So I simply remove the bit field.
4. As Patrick said, ".lock" should not be reported as error. At current, ignore files ending with ".lock".
5. Add a fsck msg type called "badRefFiletype" which indicates that a ref has a bad file type when scanning the directory.
6. Junio advices instead of using `fsck_refs_fns`, we should use the singular version `fsck_refs_fn`, fix this.
7. Drop the last patch because in this series, we mainly focus on the infra, I will add a series later to add ref content check.

And I only do some minor changes in [Patch v15](https://lore.kernel.org/git/ZrEBKjzbyxtMdCCx@ArchLinux/) and [Patch v16](https://lore.kernel.org/git/ZrSqMmD-quQ18a9F@ArchLinux.localdomain/).

The code is merged into [next](https://github.com/git/git/commit/3bde10da94b1424849233d19eeeab475c7a57152). What a long journey.

## Next Plan

My GSoC is going to end. But my contribution to Git will bot end. I will implement ref content check. And I will concentrate on writing the final report.
