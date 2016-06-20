---
layout: article
title: "first post"
categories: mysql
---

> this is main
> sites: <http://www.isjian.com/{{ page.url }}>

## sub1

## add2-test-git

this is sub1
`this is code`

### sub1-1
this is sub1-1

## sub2

this is sub2
`this is sub2 code`

## sub3

顺序执行的单线程(single_thread.py)

{% highlight shell %}
{% raw %}

#! /usr/bin/python

from threading import Thread
import time

def my_counter():
    i = 0
    for _ in range(100000000):
        i = i + 1
    return True

def main():
    thread_array = {}
    start_time = time.time()
    for tid in range(2):
        t = Thread(target=my_counter)
        t.start()
        t.join()
    end_time = time.time()
    print("Total time: {}".format(end_time - start_time))

if __name__ == '__main__':
    main()

{% endraw %}
{% endhighlight %}
