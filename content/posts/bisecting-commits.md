---
title: "Bisecting Commits"
date: 2025-05-02T16:01:34+01:00
showtoc: true
---

Every once in a while I encounter a reason to use my favourite Git command, `git bisect`.

## Problem Space

Let's say we have a repository with a few commits in it, containing a simple `file.py` script with some functions in it:

```bash
~/c/showcase> git log
475a134 (HEAD -> master) fix: call the `main` function
bafdb17 feat: add some basic functionality
a1f3745 feat: define `divide` function
0d84ce9 feat: define `multiply` function
676689d chore: no-op refactoring
e94ec21 feat: define `subtract` function
de5d193 feat: define `add` function
0c0dd22 chore: initial commit
```

At some point, one of the commits broke the `subtract` function. We don't know which one it was, but we have a separate file that uses it called `harness.py`:

```python
#!/usr/bin/env python3

from file import subtract

print(subtract(1, 1))
```

With `master` checked out, we can see that it returns the wrong result:

```bash
~/c/showcase> ./harness.py
2
```

Our first option is a sanity check, let's ensure it worked at the start:

```bash
~/c/showcase> git checkout e94ec21
HEAD is now at e94ec21 feat: define `subtract` function

~/c/showcase> ./harness.py
0
```

Okay good. We now have 2 commits, one where the behaviour is as we would expect and one where it is not. Since there's a small number of commits we could start checking them one by one, but that wouldn't work on larger repositories.

## Beginning a Bisection

`git bisect` operates on the idea of binary search. We give it a good commit and a bad commit and it gives us a new commit to test in the middle. This reduces our search space significantly, since in each run we remove half of the remaining commits.

To get started, we run `git bisect start`:

```bash
~/c/showcase> git bisect start
status: waiting for both good and bad commits
```

This tells us that it needs a good and a bad commit before we can begin the search. Since we know these from before, we can provide them:

```bash
~/c/showcase> git bisect good e94ec21
status: waiting for bad commit, 1 good commit known

~/c/showcase> git bisect bad 475a134
Bisecting: 2 revisions left to test after this (roughly 1 step)
[0d84ce96e1f6c8903dd1ab3a3c96d972512beef1] feat: define `multiply` function
```

Once we have provided both, it checks out a commit in the middle. We then run our test:

```bash
~/c/showcase> ./harness.py
2
```

Since this produces the wrong result, we feed that information back into `git bisect`:

```bash
~/c/showcase> git bisect bad
Bisecting: 0 revisions left to test after this (roughly 0 steps)
[676689dbe7c216443317997992aed2ff0b024847] chore: no-op refactoring
```

Again, it picks another commit for us to test:

```bash
~/c/showcase> ./harness.py
2
```

This also produces the wrong result, so we run `git bisect bad` again:

```bash
~/c/showcase> git bisect bad
676689dbe7c216443317997992aed2ff0b024847 is the first bad commit
676689dbe7c216443317997992aed2ff0b024847 (HEAD) chore: no-op refactoring
 file.py | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

There are no more commits to check, so this returns and tells us that `676689d` (our supposed no-op refactoring) is the first bad commit.

As we can see from the changes, it's quite obvious in this case:

```bash
~/c/showcase> git diff HEAD~ | cat
diff --git a/file.py b/file.py
index 9a35fcf..2dd699d 100644
--- a/file.py
+++ b/file.py
@@ -22,4 +22,4 @@ def subtract(a, b):
     Returns:
     int: The result of a - b.
     """
-    return a - b
+    return a + b
```

In busier repositories however it may be less obvious what caused the problem, and there may be far more commits to work through. `git bisect` provides a way of quickly trimming these down.
