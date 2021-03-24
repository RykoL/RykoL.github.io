---
layout: post
title: About Docker, nginx alpine and proxy timeouts on arm
date: 2021-03-21
---

So there I was developing a pet fullstack project. After I finished
the first usable version of the project I was about to deploy it on my
Raspberry Pi (3 Model B Plus Rev 1.3 ARMv7). As I wanted to check how
good docker worked on arm I quickly put together a docker-compose file
to ramp up the infrastructure, consisting of nginx as a reverse proxy,
the backend application and last postgresql as dbms.

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

As you might've noticed, I decided to go for mostly alpine
images. This is due to a slow SD-card mounted in the
raspberry. Because I didn't want to wait hours until I can test out my
app, I took some shortcuts such as prebuilding and mounting the
frontend package and the fat Jar into their respective containers.

## The problem begins

After those optimizations I wanted to check out my freshly deployed
app, but was quickly presented with **502 Bad Gateway** errors in the
frontend. Looking at the logs of the backend application I was able
to see that the backend container has stopped, because it couldn't
find the host of the database, which is kinda weird on it's own.

{% highlight sh %}
backend_1        | Caused by: java.net.UnknownHostException: db
backend_1        | 	at java.base/java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:220) ~[na:na]
backend_1        | 	at java.base/java.net.SocksSocketImpl.connect(SocksSocketImpl.java:403) ~[na:na]
backend_1        | 	at java.base/java.net.Socket.connect(Socket.java:609) ~[na:na]
{% endhighlight %}


Usually docker compose would create a private bridge network that
allows communication between all of the containers. Additionally it
will create a network alias for each container that is equal to it's
service name. By inspecting the network and the container using ```docker <resource> inspect```,
I couldn't see a problem network wise though.

Since the problem might not lie in the network, let's take a step back
and look at the db container instead. The logs we're quite clear about
it: As postgres just **segfaulted** itself out of existence, the
container runtime removed it from the network and hence the
Exception. But wait there is something else about it. The log was
written on the **27th of April 1970** ???

{% highlight sh %}
db_1             | 1970-04-27 05:30:16.009 GMT [1] LOG:  startup process (PID 22) was terminated by signal 11: Segmentation fault
db_1             | 1970-04-27 05:30:16.009 GMT [1] LOG:  aborting startup due to startup process failure
db_1             | 1970-04-27 05:30:16.009 GMT [1] LOG:  database system is shut down
{% endhighlight %}

There was something clearly off at this point and after a hour of
researching, I resigned and changed the image to the non alphine
one, which apparently fixed the segfault. 

## Now its nginx

Back to business I restarted the db and backend services and
everything went up as expected. Happy to finally see my app deployed
on the raspi I decided to try it out in the browser and *everything worked* ...

Not quite. The app was mostly working but every fifth request would
suddenly **time out**. At this point I started regretting my career
again and thought about how relaxing it would've been as a gardener
not having to deal with sudden http timeouts. 

{% highlight sh %}
reverse-proxy    | 2037/05/22 05:50:48 [error] 25#25: *5 upstream timed out (110: Operation timed out) while reading response header from upstream
{% endhighlight %}

Again looking at the timestamp we see that nginx is using a time
machine just like postgres. At the time of debugging I didn't notice
the timestamp and was blindly playing around with the timeout settings
in nginx until falling back to the **non alpine based docker
image**. However due to nginx having a weird system time it only makes
sense that the default 60 second treshold is reached and nginx
terminates the connection to the upstream.

## Alpine and it's time machine

The common denominator for this problem seems to be the alpine linux
docker image that was used, so I did another hour of research until I
finally found what I was looking for. Outgoing from this
**[wiki](https://wiki.alpinelinux.org/wiki/Release_Notes_for_Alpine_3.13.0#time64_requirements)**
entry, it turns out that raspbian 32bit was not suitable for running
alpine in docker all along. 
Now we need to dig a little bit deeper into what this means.

First of all it is worth mentioning that Alpine Linux uses `musl` as
it's libc implementation, contrary to what most linux distros use,
which is `glibc`. Musl up from version 1.2 started to support
time64-compatible syscalls, which changes `time_t` and other variants
of this struct to be 64bit on all architectures.

However as mentioned in the wiki page there was a
[bug](https://github.com/opencontainers/runc/issues/2151) tracked in
runc that would prevent using fallback solutions for new system calls.

### Seccomp

In order to understand whats going on there we first need to
understand what `seccomp` is and how docker uses it. Seccomp short for
*Secure computing mode* is a linux kernel facility that allows for
restricting which `syscalls` are allowed for a process. For example we
could prevent a process from opening a socket by using seccomp filters.

When it comes to docker, the `moby` project maintains a default [allow
list](https://github.com/moby/moby/blob/master/profiles/seccomp/default.json)
of all allowed `syscalls`.

But how does that play into the weird time behaviour we observed
before ? As previously stated the problem has to do with the *"newly"*
introduced `time64` syscalls and a bug in `runc` which had to do with
`seccomp`. To make it short: 
1. Libseccomp had no up to date list of syscalls
2. Runc which internally uses libseccomp would always return EPERM
   instead of ENOSYS
3. Musl's fallback mechanisms didn't apply because it would only do so on ENOSYS

## TL;DR
 
Alpine linux(3.13) **[doesn't properly work](https://wiki.alpinelinux.org/wiki/Release_Notes_for_Alpine_3.13.0#time64_requirements)** on Armv7 x86 systems. 

To make it work:
- Upgrade docker to version 19.03.9
- Upgrade your hosts libseccomp to atleast version 2.4.2
- **This is not advised** Override the default seccomp profile
