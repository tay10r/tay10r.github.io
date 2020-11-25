Retro Vim Part 3
================

### main.c

The first thing I wanted to do with this file is to have it compile with a C++ parser.
I gave it a shot and, upon the many compiler errors, I discovered that all function prototypes
are in `.proto` files in the `proto/` directory with a macro that wraps around the parameter list.
So changing all of these files to headers is the first task.

I decided to use a script:

```bash
#!/bin/bash

for f in *.pro; do
        mv -- "$f" "${f%.pro}.h"
done

for f in *.h; do
        sed -i 's/__PARMS(//g' $f
        sed -i 's/));/);/g' $f
        echo -e "#pragma once\n\n$(cat $f)" >$f
done
```

Once that was done, a few variables had to be renamed because they were C++ keywords and that was it!
Now `main.cpp` can be written entirely in C++.

### TODO

This part is still a work in progress.
