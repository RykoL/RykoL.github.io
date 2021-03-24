---
layout: post
title: About Docker, nginx alpine and time machines on arm
date: 2021-03-21
---

So there I was developing a pet fullstack project. After I finished
the first usable version of the project I was about to deploy it on my
Raspberry Pi (3 Model B Plus Rev 1.3 ARMv7). As I wanted to check how good docker
support is on arm I quickly put together a docker-compose file to ramp
up the infrastructure:

{% highlight yaml %}
version: "3.7"

services:
  backend:
    build: ./service
    environment:
      - DB_NAME=someapp
      - DB_PASSWORD=passwd123
      - DB_HOST=db

  db:
    image: postgres:alpine
    environment:
      - POSTGRES_DB=someuser
      - POSTGRES_PASSWORD=passwd123

  reverse-proxy:
    image: nginx:alpine
    ports:
      - 80:80
    volumes:
      - ${PWD}/nginx.conf:/etc/nginx/nginx.conf
      - ${PWD}/frontend/build:/usr/share/nginx/html

{% endhighlight %}

As you might've noticed, I decided to go for mostly alpine images due
to a slow SD-card mounted in the raspberry and my impatience. So I
wanted to check out my freshly deployed app, but i was quickly
presented with **502 Bad Gateway**. Damn the backend has crashed lets
consult the logs !

{% highlight sh %}
backend_1        | Caused by: java.net.UnknownHostException: db
backend_1        | 	at java.base/java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:220) ~[na:na]
backend_1        | 	at java.base/java.net.SocksSocketImpl.connect(SocksSocketImpl.java:403) ~[na:na]
backend_1        | 	at java.base/java.net.Socket.connect(Socket.java:609) ~[na:na]
{% endhighlight %}

Ah well, the backend couldn't connect to the database, probably a typo
in the compose file. That was my first thought, but after a few
minutes of trying around I couldn't figure why it couldn't find the
database in the docker network. Looking at the postgres container i
finally found out that it **segfaulted** right after the backend tried to connect 

{% highlight sh %}
db_1             | 1970-04-27 05:30:16.009 GMT [1] LOG:  startup process (PID 22) was terminated by signal 11: Segmentation fault
db_1             | 1970-04-27 05:30:16.009 GMT [1] LOG:  aborting startup due to startup process failure
db_1             | 1970-04-27 05:30:16.009 GMT [1] LOG:  database system is shut down
{% endhighlight %}

But wait what, the log entry was made on the **27th april 1970** ?
There must be something clearly off there. After researching around
for hours I finally found a thread in the postgres mailing list,
however there was no clear solution posted, so I decided to go with
plain old postgres image and guess what, it worked !

## Now its nginx

Back to business I restarted the db and backend services and
everything went up as expected. Happy to finally see my app deployed
on the raspi I decided to try it out in the browser and *everything worked* ...

Not quite. The app was mostly working but every fifth request would
suddenly **time out**. At this point I started regretting my career
again and thought about how relaxing it would've been as a
gardener. Anyway looking at the nginx I found a well covered problem

{% highlight sh %}
2037/05/22 05:50:48 [error] 25#25: *5 upstream timed out (110: Operation timed out) while reading response header from upstream,
{% endhighlight %}

Again looking at the timestamp we see that nginx is using a time machine. At the
time of debugging I didn't notice the timestamp and was blindly playing around
with the timeout settings in nginx until falling back to the **non alpine based
docker image**. However due to nginx having a weird system time it only makes
sense that the default 60 second treshold is reached and nginx terminates the
connection to the upstream.

## Alpine and it's time machine

The common denominator for this problem seems to be the alpine linux docker
image that was used, so I did another hour of research until I finally found
what I was looking for. Outgoing from this
**[wiki](https://wiki.alpinelinux.org/wiki/Release_Notes_for_Alpine_3.13.0#time64_requirements)**
entry, it turns out that raspbian 32bit was not suitable for running alpine in
docker all along due to `musl` using the `time64` syscalls and raspbian having
an out of date `libseccomp`.


## TL;DR

Alpine linux(3.13) doesn't properly work on Armv7 x86 systems. Consult this
**[wiki](https://wiki.alpinelinux.org/wiki/Release_Notes_for_Alpine_3.13.0#time64_requirements)**
page for possible solutions.

