Greybeard Vim Part 2
====================

### Investigating the Crash

When running vim under valgrind, it's easy to see where the problem is:

```
==15680== Process terminating with default action of signal 11 (SIGSEGV)
==15680==  Bad permissions for mapped region at address 0x12DAAB
==15680==    at 0x118A21: expand_env (misccmds.c:691)
==15680==    by 0x111E8D: dosource (cmdline.c:2120)
==15680==    by 0x11656E: main (main.c:278)
```

What's happening here is that the function `expand_env` is writing to a string
literal that's stored in read-only memory. The function writes to it in order
to temporarily assign a null-terminator so that it can be used to query an environment
variable with `getenv`.

```c
void
expand_env(char *src, char *dst, size_t dstlen)    
{
    char    *tail;
    int     c;
    char    *var;

    if (*src == '$')
    {
        for (tail = src + 1; *tail; ++tail)
            if (*tail == PATHSEP)
                break;
        c = *tail;
        *tail = NUL; /* <- causes segfault */
        var = getenv(src + 1);
        *tail = c;
        if (*tail)
            ++tail;
        if (var && strlen(var) + strlen(tail) + 1 < dstlen)
        {
            strcpy(dst, var);
            strcat(dst, PATHSEPSTR);
            strcat(dst, tail);
            return;
        }
    }
    strncpy(dst, src, (size_t)dstlen);
}

```

While we could just re-write the input, called `SYSVIMRC_FILE`,
to be a writable `char[]`, we are refactoring right? This is the first opportunity to
refactor part of the project.

### First Patch

I'm choosing to write new code in C++, because I enjoy some of the type-safety idioms
and it has a more fully featured standard library. Rewriting this function properly took
roughly a hundred lines of source code to account for the fact that:

 - Input strings are sometimes read-only
 - Environment variable names are dimilited by more than just path separators and null terminators.

What was left is a new function prototype of `expand_env`:

```cxx
template <typename DerivedEnv>
std::string
expand_env(const std::string_view &input, const Environment<DerivedEnv> &env);
```

With this new function, we can use static polymorphism to create a custom environment
for either sandboxing or unit testing. Here's the base definition:

```cxx
template <typename DerivedEnv>
class Environment
{
public:
    std::string
    get_value(const std::string_view &) const;
};
```

I chose to dynamically allocate a string for the return value, because `std::getenv` returns
a non-reentrant pointer. By copying the string returned by `std::getenv`, we avoid the risk of
storing a copy of the return value and using it after a successive call to `std::getenv`. If this
was not the case, I would have chosen to return another `std::string_view`.

In this patch, I've also opted to make the following unit tests for the environment variable expansion:

 - `$FOO/$BAR ` expands to `foo/bar ` when `FOO` is `foo` and `BAR` is `bar`.
 - `$ $FOO` expands to `$ foo` (test case from bash)
 - `$MISSING$FOO` expands to `foo`

### Running Vim

Now that I've fixed the write to read-only memory, Vim runs like a charm!

[See these changes on GitHub](https://github.com/tay10r/greybeard-vim/tree/2801b4a72e0c205ceb8b2f5ac0e36507645fd8fb)

In the next part, I will go into setting up regression tests to make sure that
any changes I make will not break Vim.
