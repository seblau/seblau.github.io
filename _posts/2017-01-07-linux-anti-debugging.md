---
layout: post
title: "Linux Anti Debugging"
description: "With the ptrace syscall it is quite easy to implement some simple linux anti debugging techniques. In this post however we'll cover a slightly advanced usage of the ptrace syscall in order to implement a more resistent anti debugging feature."
tags: [reversing, linux, anti-debugging]
---

[See github repo here!](https://github.com/seblau/linux-anti-debugging)

The point here is that debuggers like [gdb](https://www.sourceware.org/gdb/), [edb](https://github.com/eteran/edb-debugger) or `strace(1)` for example utilize the `ptrace(2)` function to attach to a process at runtime. But there is only one process allowed to do this at a time and therefore having a call to `ptrace(2)` in your code can be used to detect debuggers.

First I am going to quickly introduce this anti debugging technique, that uses one call to the `ptrace(2)` syscall. Afterwards I am going to introduce a slightly more advanced version to this method, that is resistent against the most common countermeasures.

## Calling `ptrace(2)` once

*traceme1.c:*
{% highlight c %}
#include <stdio.h>
#include <sys/ptrace.h>

int main()
{
    if (ptrace(PTRACE_TRACEME, 0, 1, 0) == -1) 
    {
        printf("don't trace me !!\n");
        return 1;
    }
    
    printf("normal execution\n");
    return 0;
}
{% endhighlight %}

The *traceme1.c* snippet successfully detects any debuggers. For example:
{% highlight bash %}
> strace ./traceme1.out
...
ptrace(PTRACE_TRACEME)                  = -1 EPERM (Operation not permitted)
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 3), ...}) = 0
brk(NULL)                               = 0x55ccbe9aa000
brk(0x55ccbe9cb000)                     = 0x55ccbe9cb000
write(1, "don't trace me !!\n", 18don't trace me !!
)     = 18
exit_group(1)                           = ?
+++ exited with 1 +++
{% endhighlight %}

To bypass this `ptrace(2)` anti debugging technique you can do one of the following:
* patch the call to the `ptrace(2)` syscall with NOP's
* overwrite the `ptrace(2)` function by preloading a custom ELF shared library with `LD_PRELOAD`, for example:

*cptrace.c:*
{% highlight c %}
long ptrace(int request, int pid, int addr, int data)
{
    return 0;
} 
{% endhighlight %}

{% highlight bash %}
> make cptrace.so
> export LD_PRELOAD="./cptrace.so"

> strace ./traceme1.out
> normal execution
> +++ exited with 0 +++
{% endhighlight %}

In order to be resistent against those bypasses, lets look at a slightly advanced version.

## Calling `ptrace(2)` *TWICE*

Let us look at the following snippet now.

*traceme2.c:*
{% highlight c %}
#include <stdio.h>
#include <sys/ptrace.h>

int main()
{
    int offset = 0;

    if (ptrace(PTRACE_TRACEME, 0, 1, 0) == 0)
    {
        offset = 2;
    }

    if (ptrace(PTRACE_TRACEME, 0, 1, 0) == -1)
    {
        offset = offset * 3;
    }

    if (offset == 2 * 3)
    {
        printf("normal execution\n");
    }
    else
    {
        printf("don't trace me !!\n");
    }

    return 0;
}
{% endhighlight %}

If we execute the binary without any debugger attached we get the expected behaviour:

{% highlight bash %}
> make traceme2.out
> ./traceme2.out
> normal execution
{% endhighlight %}

But if we try to execute the binary with `strace(1)` for example, we detect the debugger. This time also the `LD_PRELOAD` trick does not help us out here:

{% highlight bash %}
> make cptrace.so
> export LD_PRELOAD="./cptrace.so"

> strace ./traceme2.out
> don't trace me !!
> +++ exited with 0 +++
{% endhighlight %}

So we would need to add some state to our shared library and return different results based on this state. But that would require some static analysis in the first place, because those `ptrace(2)` calls could be chained arbitrarily.

Furthermore patching the code with NOP's will not work out of the box either, because the offset calculation must not be destroyed in order to guarantee normal execution.

A sample that I analyzed which used the same anti debugging technique, calculated some offset based on the results off various calls to `ptrace(2)`. This offset was then used as an initialization value for an unpacking function. So neither the `LD_PRELOAD` trick nor patching the code with NOP's helped me in the first place.
