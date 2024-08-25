---
title: GSoC Final Report
tags:
  - GSoC
categories:
  - GSoC 2024
date: 2024-08-25 22:32:06
---


Suddenly, I recalled myself writing the proposal back then, and I couldn't help but feel a bit wistful. Every time it ends, there's always an unintentional sense of sadness. After three months, my GSoC project has also come to an end.

## The Motivation of The Project

The `git-fsck(1)` command is mainly used to check the consistency of the object database which misses check for the ref database. Although `git-fsck(1)` implicitly checks some properties of the ref database when checking connectivity, these checks aren't sufficient to ensure that all refs are properly consistent like [report](https://lore.kernel.org/git/6cfee0e4-3285-4f18-91ff-d097da9de737@rd10.de/) shows.

The goal of this small GSoC project is to establish the infrastructure for the ref consistency check, enabling developers to easily add checks for both the files backend and the reftable backend. For more detailed information, please refer to the [proposal](https://docs.google.com/document/d/1pWnnyykGmJIN-wyosZ3PtueFfs_BRdvJq-cwroRorBI/).

## The Patches

I began my work by implementing the infrastructure. However, during the review process, the design underwent significant changes. My main focus was on the following tasks:

1. Refactoring the fsck error messages to make them generic, allowing both the object database and the ref database to use the same fsck error message interfaces to avoid repetition.
2. Utilizing the existing polymorphism provided by the `ref_storage_be` structure. For every backend, it needs to provide its own function pointer which would bring a lot of flexibility.
3. Designing extensible interfaces for checking ref consistency in the files backend.
4. Adding a ref name consistency check for the files backend.

The ultimate merged patch links are given below:

+ [\[GSoC\]\[PATCH v16 1/9\] fsck: rename "skiplist" to "skip_oids"](https://lore.kernel.org/git/ZrSq3Z8tYrGwBOqC@ArchLinux/).
+ [\[GSoC\]\[PATCH v16 2/9\] fsck: rename objects-related fsck error functions](https://lore.kernel.org/git/ZrSq6XV8hKRZMrnt@ArchLinux/)
+ [\[GSoC\]\[PATCH v16 3/9\] fsck: make "fsck_error" callback generic](https://lore.kernel.org/git/ZrSrdwOCWrXpMYIA@ArchLinux/)
+ [\[GSoC\]\[PATCH v16 4/9\] fsck: add a unified interface for reporting fsck messages](https://lore.kernel.org/git/ZrSrgRGwI_jldprn@ArchLinux/)
+ [\[GSoC\]\[PATCH v16 5/9\] fsck: add refs report function](https://lore.kernel.org/git/ZrSrjO2ltoJuppKA@ArchLinux/)
+ [\[GSoC\]\[PATCH v16 6/9\] refs: set up ref consistency check infrastructure](https://lore.kernel.org/git/ZrSrlU8TesYsTb2C@ArchLinux/)
+ [\[GSoC\]\[PATCH v16 7/9\] builtin/refs: add verify subcommand](https://lore.kernel.org/git/ZrSroE8vLlZCK2jp@ArchLinux/)
+ [\[GSoC\]\[PATCH v16 8/9\] files-backend: add unified interface for refs scanning](https://lore.kernel.org/git/ZrSsk27zqOqSXTpH@ArchLinux/)
+ [\[GSoC\]\[PATCH v16 9/9\] fsck: add ref name check for files backend](https://lore.kernel.org/git/ZrSsngBqfyTPDg7g@ArchLinux/)

This series has finally been merged into the [master](https://github.com/git/git/commit/b3d175409d9bfe005515ffe361e959fb9965111c) branch.

After establishing the infrastructure, I continue to implement additional checks for the files backend, as shown in the following patch links:

+ [\[PATCH v1 1/4\] fsck: introduce "FSCK_REF_REPORT_DEFAULT" macro](https://lore.kernel.org/git/ZsIM0L72bei9Fudt@ArchLinux/)
+ [\[PATCH v1 2/4\] ref: add regular ref content check for files backend](https://lore.kernel.org/git/ZsIM2DRDbJsvNjAM@ArchLinux/)
+ [\[PATCH v1 3/4\] ref: add symbolic ref content check for files backend](https://lore.kernel.org/git/ZsIM4OZWfylcP5Ix@ArchLinux/)
+ [\[PATCH v1 4/4\] ref: add symlink ref consistency check for files backend](https://lore.kernel.org/git/ZsIM6JZ7miA3j09j@ArchLinux/)

However, this series is still under review, and I plan to continue following up after GSoC.

## Challenges

### Design Challenge

Implementation is much more complex than refactoring. One of my intuitive feelings is that when designing something myself, I need to consider many factors. However, during the implementation process, I made many mistakes, mainly in two areas:

1. Over-engineering: I was always concerned about the extensibility of the implementation, worried that poor design would require future refactoring. However, I didn't realize that this could introduce noise into the code. We should focus solely on the features we need right now.
2. Inability to balance between building something that can be easily extended in the future and something that is specific to the task at hand.

Here are the challenges I encountered during the implementation process:

#### Whether We Need To Refactor "fsck_options"

There are many fields in fsck_options that are specific to the object database. My first mistake was trying to make `fsck_options` contain the general options and sub-structures, which is what I attempted in the previous patch.

> The git-fsck(1) focuses on object database consistency check. It relies on the "fsck_options" to interact with fsck error levels. However "fsck_options" aims at checking the object database which contains a lot of fields only related to object database.
>
> In order to add ref operations, create a new struct named "fsck_refs_options" and a new struct named "fsck_objs_options". Remove object-related fields from "fsck_options" to "fsck_objs_options". Change the "fsck_options" with three parts of members:
>
> 1. The "fsck_refs_options".
> 2. The "fsck_objs_options".
> 3. The common settings both for refs and objects. Because we leave common settings in "fsck_options". The setup process could be fully reused without any code changing.

While this approach may seem natural, it introduced a lot of complexity because there is a significant amount of existing code that accepts `struct fsck_options *` as a parameter. As a result, we had to deal with a lot of code that was not directly related to our main goal. In the final implementation, I simply added a `verbose` field to the `fsck_options`.

Junio gave me a wonderful suggestion:

> Just like premature optimization is bad, premature factoring and over-modularization is bad.

#### How to Reuse "report" Function

To adapt to the existing fsck error levels, my initial approach was to create a new `fsck_report_ref` function that was separate from the object report function `report`. However, Patrick, Karthik, and Junio disagreed with this design, suggesting that I should reuse the `report` function.

The `report` function is closely tied to object reporting. At the time, I considered adding parameters to the `report` function and its corresponding callback function `error_func` to make their prototypes consistent. However, I was not satisfied with this solution because we couldn't predict whether additional parameters would be needed for other ref checks in the future, which would result in poor extensibility.

But Patrick gave me a wonderful suggestion which eventually solved the problem:

> A better design would likely be to make `error_func()` receive a `void*` pointer such that `error_func()` and then have the respective subsystems provide a function that knows to format the message while receiving either a `struct fsck_object_report *` or a `struct fsck_ref_report *`.

### Long Review Duration

Building the ref consistency check infrastructure posed challenges not only in terms of design and coding but also due to the lengthy review process. This created a significant mental burden for me during the middle of GSoC. This is what I have recorded in my [GSoC Week 7](https://luolibrary.com/2024/07/16/GSoC-Week-7/) blog:

> The recent challenges I’ve encountered mainly stem from two aspects. Over the past two weeks, I’ve felt mentally exhausted because I haven’t received much positive feedback. Since May 30th, my first patch is still under review. Sometimes, I can’t help but feel the pressure from my peers. Seeing other GSoC participants successfully merge several patches does indeed make me feel pressured. Therefore, I realize that I must learn how to adjust my mindset during prolonged review periods.

## After GSoC

After GSoC, I decided to remain involved with the Git community to continue the work I need to complete. My tentative roadmap is as follows:

1. Implement the pack-refs consistency check for the files backend.
2. Implement the reflogs consistency check for the files backend.
3. Implement the consistency check for the reftable backend.
4. Separate the object database check logic from `git-fsck(1)`.
5. Enhance the `git-fsck(1)` command to allow users to easily disable subprocess checks, providing greater flexibility.

There is a lot to accomplish, which is exciting!

## Closing Remarks

During my GSoC journey, besides improving my coding skills, I believe the greatest gain was strengthening connections between people. I established a good relationship with my two mentors, Patrick and Karthik. Initially, I rarely conversed in English with others, and during my first video meeting with Patrick, I was quite nervous, worried about my limited English speaking skills.

Additionally, I feel that I have built a good relationship with the Git community. Through an open-source project, I experienced the charm of open source, as it connects people who might not have otherwise crossed paths in life.

I am especially grateful to my two mentors from GitLab, [Patrick](https://gitlab.com/pks-gitlab) and [Karthik](https://www.linkedin.com/in/karthik%2Dnayak/). They provided me with a lot of guidance, embodying the role of a teacher who imparts knowledge and resolves doubts. I also want to thank [Junio](https://github.com/gitster/), Eric, and [Justin](https://www.linkedin.com/in/justintobler/) for their meticulous and detailed work during the code review process, which made the final implementation exceptionally elegant.

I also feel quite proud of myself for being able to implement a new feature for foundational software used by so many people, something I once thought could only happen in my dreams. I am very grateful for this experience and hope that I can become a mentor in the future, igniting the flame in others' hearts.
