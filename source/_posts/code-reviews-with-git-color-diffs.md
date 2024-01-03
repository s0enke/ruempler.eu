---
title: Easier Code reviews with Git Colored Move Diffs
date: 2023-12-31 12:00:00
---

Code reviews can be very time-consuming. Especially if you are reviewing a lot of code changes. Often, code changes are simple refactorings, such as extracting code into methods/functions or moving code around. These changes are more or less irrelevant for the review, since they usually don't change the behavior of the code. As a reviewer, one still wants to make sure that code hasn't been changed during the code move.

<!--more-->

This is where git colored move diffs come into play. They make it easier to detect and display code moves in a different color than other code changes. 

## How to use git colored move diffs on demand

Colored Move Diffs work with all git commends that display diffs, such as `git diff`, `git show`, `git log -p`, etc.

```bash
git diff --color-moved
git show --color-moved HEAD
git log --color-moved -p
```

You can also set different color schemes for moved code, I prefer the `dimmed-zebra` color profile since it greys out moved code:
```bash
git diff --color-moved=dimmed-zebra
```

Here's an example screenshot of how a code move looks like with git colored move diffs and `dimmed-zebra` setting. The moved code is grey, so you can easily spot it the real change, `% 5` changed to `% 8`, which might have been accidentally introduced during the code move (which I consider harmful since it mixes a refactoring and a behavioral change into one commit - but that's a different story):

![Example of how a code move looks like with git colored move diffs](/2023/12/31/code-reviews-with-git-color-diffs/example.png)

There is also an option to ignore whitespace changes in moved code:
```bash
git diff --color-moved=dimmed-zebra --color-moved-ws=ignore-all-space
```

## How to enable git colored move diffs

To globally configure git to use colored moves in git diffs, run the following commands:

```bash
git config --global diff.colorMoved dimmed-zebra # enables colored moved diffs with the dimmed-zebra color profile
git config --global diff.color-moved-ws ignore-all-space # ignores whitespace changes in moved code
```

And now, happy code reviewing!