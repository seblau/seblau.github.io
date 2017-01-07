---
layout: post
title: "Linux Anti Debugging"
description: "With the ptrace syscall it is quite easy to implement some simple linux anti debugging techniques. In this post however we'll cover a slightly advanced usage of the ptrace syscall in order to implement a more resistent anti debugging feature."
tags: [reversing, linux, anti-debugging]
---

[See github repo here!](https://github.com/seblau/linux-anti-debugging)

## Single PTRACE syscall

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

Next setup your environment:

{% highlight bash %}
bin/setup
{% endhighlight %}

## Development

Run Jekyll:

{% highlight bash %}
bundle exec jekyll serve
{% endhighlight %}

## Deploy to GitHub Pages

Run this in the root project folder in your console:

{% highlight bash %}
bin/deploy
{% endhighlight %}

You can find more info on how to use the gh-pages branch and a custom domain [here](https://help.github.com/articles/quick-start-setting-up-a-custom-domain/).

[View this](https://github.com/nielsenramon/kickster#automated-deployment-with-circle-ci) for more info about automated deployment with Circle CI.

_If you have any questions about using or configuring Chalk please create an issue <a href="" title="here" target="_blank">here</a>!_
