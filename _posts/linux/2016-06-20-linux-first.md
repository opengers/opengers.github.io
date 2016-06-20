---
layout: article
title: "linux-first1"
categories: linux
image:
    teaser: /images/linux/bio-photo.jpg
---


# first1
test

test

## second2
second2`code1`
NUMA中，虽然内存直接attach在CPU上，但是由于内存被平均分配在了各个die上。
![numa](/images/linux/bio-photo.jpg)


``` python
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
```


{% highlight python %}
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