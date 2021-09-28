---
layout: post
title:  "Python Enums and weak typing"
tag: Python
category: miscellaneous
---

I ran in an annoying little quirk of [Python's Enum class](https://docs.python.org/3/library/enum.html) today. Consider
the following:

```
/
+- foo/
|  +- __init__.py
+- bar/
|  +- __init__.py
+- quirk.py
```

and:

```python
# foo/__init__.py:
import enum

class T(enum.Enum):
    FOO = 1
    BAR = 2

def f():
    return T.FOO

# bar/__init__.py:
import enum

class T(enum.Enum):
    FOO = 1
    BAR = 2

def g():
    return T.FOO

# quirk.py
from foo import f
from bar import g


if __name__ == "__main__":
    x = f()
    y = g()
    assert x == y
```

As you may have imagined, the assertion fails, even though the definitions of the enums are the same. The reason for
that is that the `Enum` class does not have an equility test defined, so Python resorts to comparing what `id()`
returns. Since the enumerations are defined in different modules, they have different IDs, so they are not equal.

This may look like a convoluted example, but consider what I was working on today. The code base looked somewhat like
this:

```
myservice/
+- interface/
|  +- myservice_interface/
|  |  +- datatypes.py
|  |  +- controller.py
|  |  +- ...
|  +- (setup.cfg, etc.)
+- implementation/
+- tests/
```

There are advantages and disadvantages to this approach, but that's not the point of this post.

The `myservice_interface` package is distributed to users of the service, and the users would do something like:

```python
from myservice_interface import datatypes

myvalue = datatypes.SomeEnum.FOO
rpc_call(myvalue)
```

On the implementation side, code would simply do:

```python
from interface.myservice_interface import datatypes

uservalue = rcp_receive()
assert uservalue == datatypes.SomeEnum.FOO
```

This was nice, because then I don't need to reinstall the interface package all the time while I'm developing the
service. However, `myvalue` was pickled, so that approach quickly failed in testing and we resorted to do install the
interface package in the development environment. All the imports were changed accordingly - well, almost all of them.
Some small utility module still used the `from interface.myservice_interface import ...` statement, and I ended up with
unit tests failing with the less than helpful:

```
AssertionError: '<T.FOO: 1> != <T.FOO: 1>`
```

Eventually I stumbled upon [this StackOverflow post](https://stackoverflow.com/a/28125951/2564080) which explains it
all very nicely and how using `IntEnum` could have helped me here.

But let's look at the problem again: why can I define the same enumeration with the same fields and values twice? And why can I compare them?

Let's try the same in C:

```c
// foo.h
typedef enum {
    FOO = 1,
    BAR = 2
} T;

T f(void) {
    T x = FOO;
    return x;
}

// bar.h
typedef enum {
    FOO = 1,
    BAR = 2
} T;

T g(void) {
    T x = FOO;
    return x;
}

// quirk.c
#include "foo.h"
#include "bar.h"

int main(void) {
    T x = f();
    T y = g();
    if (x == y) {
        return 0;
    } else {
        return -1;
    }
}
```

This will not compile


```
% gcc quirk.c
In file included from quirk.c:2:
./bar.h:2:5: error: redefinition of enumerator 'FOO'
    FOO = 1,
    ^
./foo.h:2:5: note: previous definition is here
    FOO = 1,
    ^
In file included from quirk.c:2:
./bar.h:3:5: error: redefinition of enumerator 'BAR'
    BAR = 2
    ^
./foo.h:3:5: note: previous definition is here
    BAR = 2
    ^
In file included from quirk.c:2:
./bar.h:4:3: error: typedef redefinition with different types ('enum T' vs 'enum T')
} T;
  ^
./foo.h:4:3: note: previous definition is here
} T;
  ^
3 errors generated.
```

There are two different errors:
 - `FOO` and `BAR` are defined twice in an enumeration.
 - `enum T` is defined twice.

So let's try again and change `bar.h` to the following:

```c
typedef enum {
    BAZ = 1,
    QUU = 2
} S;

S g(void) {
    S x = BAZ;
    return x;
}
```

Also change `T y = g()` in `quirk.c` to `S y = g()` and try again:

```
% gcc quirk.c
quirk.c:7:11: warning: comparison of different enumeration types ('T' and 'S') [-Wenum-compare]
    if (x == y) {
        ~ ^  ~
1 warning generated.
```

It compiles, but with a very clear warning that we might be doing something unintentional here. The comparison
evaluates, because the integer values agree (just as Python's `IntEnum`).

 ---

I really like Python for its quick development cycles and extensive standard library, but its weak typing ("duck"
typing, dynamic typing, or whatever you want to call it), keeps biting me.

The code base I work on professionally is now almost entirely type annotated and running [mypy](http://mypy-lang.org)
catches a lot of errors, but evidentally not all of them. And having to type-annotate everything makes me wonder why
not using a strongly-typed language in the first place.
