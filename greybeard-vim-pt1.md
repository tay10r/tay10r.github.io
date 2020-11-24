Greybeard Vim Part 1
====================

### Choosing a Starting Point

The first question with most refactoring projects is where to start.
As it currently stands, [vim](https://github.com/vim/vim) is currently
sitting at over 350,000 lines of source code (counted with `cloc`. While
that's not large in some regards, it is large for one person.

I've decided to start in an earlier point in Vim's history, when it was
not as large as it is now. Unfortunately, going back too far means losing
out on some of the features that have been implemented along the way. And
while I don't know if it's been re-written in the release history, there's
also the risk of starting before a major re-write.

The sweet spot is going far enough back to where the code base is still small
but not too far in that it doesn't even look like Vim. I chose to download
[vim-history](https://github.com/vim/vim-history) since the main Vim repository
does not contain the earliest version of the code. Looking at the commit history:

```
git log --reverse
```

Shows that the earliest commit is this:

```
commit e328c16bdeeaf2067d367d13a0cdd95cfe0d1a19
Author: Bram Moolenaar <Bram@vim.org>
Date:   Sat Nov 2 00:00:00 1991 +0000

    Vi IMitation v1.14
```

As of writing this (Nov 23, 2020) this is almost 30 years old!
Kudos to Bram Moolenaar to maintaining a project faithfully for that long.
This commit is sitting at 11,079 lines of code. That's not too bad. Let's
see how the code grows as the versions go on:

| Version | LoC    | Date         |
|---------|--------|--------------|
| v1.14   | 11,079 | Nov 02, 1991 |
| v1.17   | 13,115 | Apr 20, 1992 |
| v1.24   | 17,909 | Jan 05, 1993 |
| v1.27   | 18,697 | Apt 04, 1993 |

It gets pretty large, pretty fast. I chose to go with version v1.24 because
it was the first one to show some sort of support for UNIX. Thankfully, it
builds successfully! I made some very minor changes and added a CMake build system.

[See these changes on GitHub.](https://github.com/tay10r/greybeard-vim/tree/4c3ccb6e74956687713f8297db3e26c5e6ba6f07)

The only downside is that it crashes when it runs! In the [next part](greybeard-vim-pt2.md), we'll
go into the problem and solution for the crash.
