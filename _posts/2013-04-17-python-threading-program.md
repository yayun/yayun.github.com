---
layout: post
title: "Python Threading join()"
description: ""
category: Python
tags: [Python-threading]
---
{% include JB/setup %}

-----
####Basic syntax : join([timeout]):
[wait until the thread terminate](http://docs.python.org/2/library/threading.html#threading.Thread.join)

{% highlight python %}

import threading
import time
import logging
logging.basicConfig(level=logging.DEBUG,
                    format='[%(levelname)s] (%(threadName)-10s) %(message)s',
                    )
def non_daemon():
    time.sleep(4)
    logging.debug("Test non_daemon")
t=threading.Thread(name='non-daemon',target=non_daemon)
t.start()
logging.debug('Test one')
print t.isAlive()
t.join()
print t.isAlive()
logging.debug('Test two')
{% endhighlight %}


    [DEBUG] (MainThread) Test one
    True
    (Here wait 4 secends,than print:)
    [DEBUG] (non-daemon) Test non_demon
    False
    [DEBUG] (MainThread) Test two


 **Here we can comprehend it like this**:<!--more-->
 
 join() method is called by main-thread( When you create a program , itself is a process which includes a Thread that is main thread.),here the timeout is None (it has the same result if you set the timeout>4) so the main-thread can wait long enough so that the child thread (t) can finish it's task (ofen the thread has a timeout so that it could waste so many time).In the meantime main-thread is like "sleeping".But if you set the timeout`<`4 or we don't have `t.join()`,the main-thread can not wait so long (it can only wait `<4` secends and the thread use 4') and the print is :

        [DEBUG] (MainThread) Test one
        True
        True
        [DEBUG] (MainThread) Test two
        [DEBUG] (non-daemon) Test non_demon

From the above ,we can conclusion that no matter what the flow of the program our non-daemon thread can exiting finally.But if we flag the 'non-daemon' as a "daemon-thread" and also without `join()` it varies:here `[DEBUG] (non-daemon) Test non_demon` can't show up anymore.

 -----

####[Daemon](http://en.wikipedia.org/wiki/Daemon_%28computing%29)(wiki):
A thread can be flagged as a “daemon thread”. The significance of this flag is that the entire Python program exits when only daemon threads are left. The initial value is inherited from the creating thread. The flag can be set through the daemon property.
<!--Daemon threads are abruptly stopped at shutdown. Their resources (such as open files, database transactions, etc.) may not be released properly. If you want your threads to stop gracefully, make them non-daemonic and use a suitable signalling mechanism such as an Event.There is a “main thread” object; this corresponds to the initial thread of control in the Python program. It is not a daemon thread.There is the possibility that “dummy thread objects” are created. These are thread objects corresponding to “alien threads”, which are threads of control started outside the threading module, such as directly from C code. Dummy thread objects have limited functionality; they are always considered alive and daemonic, and cannot be join()ed. They are never deleted, since it is impossible to detect the termination of alien threads.-->

<!--<pre class="pre-color"></pre>-->


{% highlight python %}

import threading
import time
import logging

logging.basicConfig(level=logging.DEBUG,
                    format='(%(threadName)-10s) %(message)s',
                    )

def daemon():
    logging.debug('Starting')
    time.sleep(4)
    logging.debug('Exiting')

d = threading.Thread(name='daemon', target=daemon)
d.setDaemon(True)

def non_daemon():
    logging.debug('Starting')
    time.sleep(2)
    logging.debug('Exiting')

t = threading.Thread(name='non-daemon', target=non_daemon)
t.setDaemon(False)

d.start()
t.start()

d.join()
t.join()

{% endhighlight %}


    (daemon    ) starting
    (non-daemon) starting
    (non-daemon) Exiting
    (daemon    ) Exiting
Here if we don't have `d.join()`,it means that 'main-thread' will not wait the 'daemon-thread' also it is a 'daemon' thread ,it runs without blocking the main program from exiting.It means that it is still alive even though the 'main-thread'(our program) exits


{% highlight python %}
import threading
import time
import logging

logging.basicConfig(level=logging.DEBUG,
                    format='(%(threadName)-10s) %(message)s',
                    )

def daemon():
    logging.debug('Starting')
    time.sleep(4)
    logging.debug('Exiting')

d = threading.Thread(name='daemon', target=daemon)
d.setDaemon(True)

def non_daemon():
    logging.debug('Starting')
    time.sleep(2)
    logging.debug('Exiting')

t = threading.Thread(name='non-daemon', target=non_daemon)
t.setDaemon(False)

d.start()
t.start()
{% endhighlight %}


    (daemon    ) starting
    (non-daemon) starting
    (non-daemon) Exiting

---

Reference:

<a href="http://pymotw.com/2/threading/">threading – Manage concurrent threads</a>

<a href="http://stackoverflow.com/questions/15085348/what-is-the-use-of-join-in-python-threading">what is the use of join() in python threading</a>
