---
title: GSoC Week 1
tags:
  - GSoC
categories:
  - GSoC 2024
date: 2024-06-02 16:51:10
---

## What I Have Done

This is the first week of writing code. I spent most of my time building the infrastructure for the ref consistency check and implementing some consistency checks for the files backend.

### Set Up the Infrastructure

I reuse the polymorphism provided by `ref_storage_be` and add a new interface called `fsck_fn` to facilitate consistency checks for refs as the [refs: setup ref consistency check infrastructure](https://lore.kernel.org/git/20240530122753.1114818-1-shejialuo@gmail.com/T/#m63c0d98020a814954f26c54c65bf04830bb0862a) shows. Additionally, I implement dummy methods for different backends to ensure compatibility and extensibility.

### Name and Content Check For Files Backend

After setting up the infrastructure, I decide to add some basic checks for the file backend. We may need to parse every file located in a specific directory and perform checks on its content or file name. Therefore, I design the following workflow:

```pseudocode
iter = dir_iterator_begin(dir, 0);
while (dir_iterator_advance(iter) == ITER_OK)
  if (IS_FILE(iter))
    for (fn = fns; fn; fn++)
      fn(iter)
```

And I have written the following code to achieve above pseudo code.

```c
typedef int (*files_fsck_refs_fn)(const char *refs_check_dir,
        struct dir_iterator *iter);

static int files_fsck_refs(struct ref_store *ref_store,
        const char* refs_check_dir,
        files_fsck_refs_fn *fsck_refs_fns)
{
  ...
  while ((iter_status = dir_iterator_advance(iter)) == ITER_OK) {
    error("test: %s", iter->relative_path);
    if (S_ISDIR(iter->st.st_mode)) {
      continue;
    } else if (S_ISREG(iter->st.st_mode)) {
      for (files_fsck_refs_fn *fsck_refs_fn = fsck_refs_fns;
          *fsck_refs_fn; fsck_refs_fn++) {
        ret |= (*fsck_refs_fn)(refs_check_dir, iter);
      }
    } else {
      error(_("unexpected file type for '%s'"), iter->basename);
      ret = -1;
    }
  }

  ...
}
```

For name check and content check, we should just following the prototype of `files_fsck_refs_fn`. Below shows the detail about these two functionalities.

+ Name check: use the existing API `check_refname_format` to check the reference name.
+ Content check: check the following two cases.
  + check whether the ref content length is valid and the last character is a newline.
  + check whether the content is be range of `[0-9a-f]`.

### Limitations

There are many limitations for the current implementation:

+ The semantic of `fsck_fn` should be presented either in code using comments or in commit message. As [Junio](https://github.com/gitster) said:
  > What is the extra blank line doing there?  It makes reader wonder why the .fsck member is somehow very special and different from others.  Is there a valid reason to single it out (and no, "yes this is special because I invented it" does not count as a valid reason)? The same comment applies to a few other places in this patch.
+ The current implementation totally ignores the symbolic ref such as `ref: refs/heads/main`.
  > In any case, the two checks are good ONLY for regular refs and not for symbolic refs.  The users are free to create symbolic refs next to their branches, e.g. here is a way to say, "among the maintenance tracks maint-2.30, maint-2.31, ... maint-2.44, maint-2.45, what I consider the primary maintenance track is currently maint-2.45".
+ The current implementation should consider the real symbolic file.
  > The caller also needs to be prepared to find a real symbolic link that is used as a symbolic ref.
+ The content check is not proper.
  > I do not think it is a good idea to suddenly redefine what a valid way to write object names in a loose ref file after ~20 years. it should be at most FSCK_WARN when it does not look like what _we_ wrote.
+ The current implementation does not interact with the fsck error levels at all.
  > How well does this interact with the fsck error levels (aka fsck_msg_type), by the way?  It should be made to work well if the current design does not.

## Next Plan

The next plan is very clear now.

+ Really understand what `git-fsck` does to know whether we should port some functions to ref part and also to find a way to interact with teh fsck error levels.
+ Enhance the current implementation to support regular file containing symbolic ref and symbolic link file.
+ Understand how `git-worktree` works and incorporate the worktree like `git-fsck`.
