---
title: GSoC Week 7
tags:
  - GSoC
categories:
  - GSoC 2024
date: 2024-07-16 21:45:49
---


## What I Have Done

In this week, I still put my effort on handling reviews.

+ [Patch v9](https://lore.kernel.org/git/Zo0sQCBqyxX8dJ-f@ArchLinux/): In version 8, Karthik thinks that instead of renaming `skiplist` to `oid_skiplist`, we'd better rename `skiptlist` to `skip_oids`. So, I simply apply this idea. And I have made a mistake in version 8, I use the `static` for a string buf, but I forget to use `strbuf_reset` for each function call. So I add this statement here. However, Eric thinks that using `static` variable is a very bad idea, because `fsck_refs_error_function` should not be called frequently. The design will make the code harder to "libify". This is one thing I have learned. And Justin has provided some reviews about the commit sequence and commit messages, need to make it clearer.
+ [Patch v10](https://lore.kernel.org/git/Zo6eJi8BePrQxTQV@ArchLinux/): This patch solves the problem of version 9. But there are some new reviews coming from Junio. The first problem is that we should not name the report interface `vfsck_report` but "fsck_vreport", and Junio feels that the current commit sequence is not so clear, need to adjust.

However, In patch v10, the most important question Junio asks is about the functionality `fsck_refs_error_function`. The declaration of this function in patch v10 is shown below:

```c
int fsck_refs_error_function(struct fsck_options *options UNUSED,
                             const struct object_id *oid,
                             enum object_type object_type UNUSED,
                             const char *checked_ref_name,
                             enum fsck_msg_type msg_type,
                             enum fsck_msg_id msg_id UNUSED,
                             const char *message );
```

Junio thinks above declaration only handles the problem of the single ref such as `refs/heads/main`. What about the packed-refs?

> The error reporting function for refs consistency check was still about reporting a problem for a single ref.  I am wondering how consistency violations that are not about a single ref should be handled.  For example, if `refs/packed-backend.c:packed_fsck()` finds that the file is not sorted properly or has some unparsable garbage in it, it is not something you can report as "refs/heads/main is broken", but those who are interested in seeing the "reference database consistency" verified, it is very much what they want the tool to notice.

So I need to find a way to handle this. So I decide to look at the spec of the packed-refs and reftable refs. And I finally find a way to do this.

After some investigations, there are the following situations where we should handle when checking ref consistency.

1. When checking loose refs and reflofs, we only need the `checkee` information, because they are standalone files.
2. When checking packed-refs, we should check the packed-refs itself, for example whether it is sorted or there are some garbage trailing contents. However, we should also check each ref (`sub_checkee`) in the file.
3. When checking reftable refs, we need to check the binary file, we could still use the idea like 2 case.

By the above statements, I change the `fsck_refs_error_function` shown as below:

```c
int fsck_refs_error_function(struct fsck_options *options UNUSED,
                             const struct object_id *oid,
                             enum object_type object_type UNUSED,
                             const char *ref_checkee,
                             const char *sub_ref_checkee,
                             enum fsck_msg_type msg_type,
                             enum fsck_msg_id msg_id UNUSED,
                             const char *message)
{
    struct strbuf sb = STRBUF_INIT;
    int ret = 0;

    if (sub_ref_checkee)
        strbuf_addf(&sb, "%s.%s", ref_checkee, sub_ref_checkee);
    else
        strbuf_addstr(&sb, ref_checkee);

    if (oid)
        strbuf_addf(&sb, " -> (%s)", oid_to_hex(oid));

    if (msg_type == FSCK_WARN)
        warning("%s: %s", sb.buf, message);
    else
        ret = error("%s: %s", sb.buf, message);

    strbuf_release(&sb);
    return ret;
}
```

It could provide the following report messages:

1. "ref_checkee": "fsck error name": "user message".
2. "ref_checkee.sub_ref_checkee": "fsck error name": "user message".
3. "ref_checkee -> (oid hex)": "fsck error name": "user message".
4. "ref_checkee.sub_ref_checkee -> (oid hex)": "fsck error name": "user message".

This is what I done in [patch v11](https://lore.kernel.org/git/ZpPEdmUN1Z5tqbK3@ArchLinux/).

## Next Plan

It's important for me to set up the infra of consistency check for refs. Because everything will be built upon that. I think I will continue to handle the reviews.

## Challenges

The recent challenges I've encountered mainly stem from two aspects. Over the past two weeks, I've felt mentally exhausted because I haven't received much positive feedback. Since May 30th, my first patch is still under review. Sometimes, I can't help but feel the pressure from my peers. Seeing other GSoC participants successfully merge several patches does indeed make me feel pressured. Therefore, I realize that I must learn how to adjust my mindset during prolonged review periods.

Secondly, there's the issue raised by Junio regarding how to improve the "fsck_refs_error_function." Junio actually brought this up in the initial version, but I subconsciously ignored it. Although I am responsible for building the infrastructure, I have limited knowledge about packed-refs and reftable refs. I thought it would be very difficult for me to design a perfect function. However, in the end, I had to face this problem head-on. Sometimes, when I encounter difficult issues, I tend to avoid them subconsciously. Despite this, I eventually found a solution.
