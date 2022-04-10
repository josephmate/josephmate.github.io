---
layout: post
title: "1,000,000 Concurrent Connections"
date: 2022-04-07 11:59
author: joseph
comments: true
categories: [Java, Server, Connections]
---

I hear this misconception all the time:

> (A TCP/IP address only supports 65,000 connections, so you would have to have to assign around 30,000 IP addresses to that server.)[https://www.quora.com/Is-it-possible-to-handle-1-billion-connections-simultaneously-in-Java-with-a-single-server]

So I put together this article arguing from three directions:

1. WhatsApp and Pheonix have already demonstrated this.
2. What's theorically possible based on the TCP/IP protocol
3. A simple Java experiment anyone can run on their machine if there are still
   not convinced.

Jump to the summary at the end if you want to skip the details.

# Experiments

The Pheonix framework which (achieved 2,000,000 concurrent websocket connections)[https://www.phoenixframework.org/blog/the-road-to-2-million-websocket-connections]
(WhatsApp also achieved 2,000,000)[https://blog.whatsapp.com/1-million-is-so-2011].
Interestingly, both are built on Erlang.

# Theoretical Max

Some think it's 2^16=65,536 because that's all the port available.
That's true for a client making a connection to an IP, port pair.
For instance, my laptop will only be able to make 65000 connections to Google (probably a lot less due to NAT).

However, for a server listening on a port, connections will be coming from multiple IP addresses.
So a server listening to one port,
on one IP address in the best case will be able to listen to all IP address,
coming from all ports on the IP addresses.

To understand the theortical max,
you need to understand a little bit of TCP over IP.

Each packet has:
1. 32bit source IP (the IP address the packet is coming from)
2. 16bit source port (the port on the source IP address the packet is coming from)
3. 32bit destination IP (the IP address the packet is going to)
4. 16bit destination port (the port on the destination IP address the packet is
   going to)

Then theoretical limit is 2^48 which is about 1 quadrillion because:

1. [number of source IP addresses]x[num of source ports]
2. You could have multiple IP addresses and ports logically implemented by the
   same server, but source IP and source port are more than enough.
2. because the server multiplexes the connections from the client using that.
3. 32 bits for the address and 16 bits for the port
4. In total is 2^48.
5. Which is about a billion connections (log(2^48)/log(10)=14.449)!

# Practical Limit
For understanding the practical limit, I put together some experiments try to
open as many TCP connections have have the server send and receive a message on
each connection.

## The experiment
If you're interested in the source code, take a look 
(here)[https://github.com/josephmate/java-by-experiments/blob/main/max_connections/src/Main.java].

The psuedo code is:

```
Thread 1:
  for i from 1 to 1 000 000:
    open server socket
  for i from 1 to 1 000 000
    send number i on socket i
  for i from 1 to 1 000 000
    receive number j on socket i
    assert i == j

Thread 2:
  for i from 1 to 1 000 000:
    open client socket to server
  for i from 1 to 1 000 000:
    receive number j on socket i
    assert i == j
  for i from 1 to 1 000 000
    send number i on socket i
```

## The Machines
For my machines I tried on my Mac:
```
2.5 GHz Quad-Core Intel Core i7
16 GB 1600 MHz DDR3
```

and on my Linux desktop:
```
AMD FX(tm)-6300 Six-Core Processor
8GiB 1600 MHz
```

## File Descriptors

First battle you'll encounter is with the operating system.
The defaults prevent so many file descriptors.
You'll see an error like:
```
Exception in thread "main" java.lang.ExceptionInInitializerError
  at java.base/sun.nio.ch.SocketDispatcher.close(SocketDispatcher.java:70)
  at java.base/sun.nio.ch.NioSocketImpl.lambda$closerFor$0(NioSocketImpl.java:1203)
  at java.base/jdk.internal.ref.CleanerImpl$PhantomCleanableRef.performCleanup(CleanerImpl.java:178)
  at java.base/jdk.internal.ref.PhantomCleanable.clean(PhantomCleanable.java:133)
  at java.base/sun.nio.ch.NioSocketImpl.tryClose(NioSocketImpl.java:854)
  at java.base/sun.nio.ch.NioSocketImpl.close(NioSocketImpl.java:906)
  at java.base/java.net.SocksSocketImpl.close(SocksSocketImpl.java:562)
  at java.base/java.net.Socket.close(Socket.java:1585)
  at Main.main(Main.java:123)
Caused by: java.io.IOException: Too many open files
  at java.base/sun.nio.ch.FileDispatcherImpl.init(Native Method)
  at java.base/sun.nio.ch.FileDispatcherImpl.<clinit>(FileDispatcherImpl.java:38)
  ... 9 more
```


Each server socket needs two file descriptors:

1. A buffer for send
2. A buffer for receiving

The same goes for client connections.
As a result, running this experiment on a single machine whill require:

* 1,000,000 connection for the client
* 1,000,000 connection for the server
* 2 file descriptors per connection
* = 4,000,000 file descriptors

For a bigSur 11.4 Mac, you can increase the file descriptor limit with:
```
sudo sysctl kern.maxfiles=2000000 kern.maxfilesperproc=2000000
kern.maxfiles: 49152 -> 2000000
kern.maxfilesperproc: 24576 -> 2000000
sysctl -a | grep maxfiles
kern.maxfiles: 2000000
kern.maxfilesperproc: 1000000

ulimit -Hn 2000000
ulimit -Sn 2000000
```
as recommend by this [stackoverflow answer](https://superuser.com/a/1644788).

For Ubuntu 20.04, I the quickest way is to:
```
sudo su
# 2^25 should be more than enough
sysctl -w fs.nr_open=33554432
fs.nr_open = 33554432
ulimit -Hn 33554432
ulimit -Sn 33554432
```


## Java File Descriptor Limits

Now that operating system is complying,
the JVM doesn't like what you're doing with this experiment either.
When you run the experiment you still get the same or similar stack trace.

This [stackoverflow answer]( https://superuser.com/a/1398360) identifies a JVM
flag as the solution:

> [`-XX:-MaxFDLimit` :     Disables the attempt to set the soft limit for the
> number of open file descriptors to the hard limit. By default, this option is
> enabled on all platforms, but is ignored on Windows. The only time that you
> may need to disable this is on Mac OS, where its use imposes a maximum of
> 10240, which is lower than the actual system maximum.](https://docs.oracle.com/en/java/javase/16/docs/specs/man/java.html)

```
java -XX:-MaxFDLimit -cp out/production/max_connections Main 6000
```
This only seems to be required for the Mac.
I was able to get the experiment to run without the flag on Ubuntu.

## Source Ports

It still won't work.
You encounter a stacktrace like:
```
Exception in thread "main" java.net.BindException: Can't assign requested
address
        at java.base/sun.nio.ch.Net.bind0(Native Method)
        at java.base/sun.nio.ch.Net.bind(Net.java:555)
        at java.base/sun.nio.ch.Net.bind(Net.java:544)
        at java.base/sun.nio.ch.NioSocketImpl.bind(NioSocketImpl.java:643)
        at
java.base/java.net.DelegatingSocketImpl.bind(DelegatingSocketImpl.java:94)
        at java.base/java.net.Socket.bind(Socket.java:682)
        at java.base/java.net.Socket.<init>(Socket.java:506)
        at java.base/java.net.Socket.<init>(Socket.java:403)
        at Main.main(Main.java:137)
```

The final battle is the TCP/IP specification.
You laptop is limited to about 65,000 client ports.
Our experiment well exceeds this limit.
As a result, we work around this by conservatively assigning an IP address for every 5,000 client connections.

On bigSur 11.4 you can add with:
```
for i in `seq 0 200`; do sudo ifconfig lo0 alias 10.0.0.$i/8 up  ; done 
```

To test:
```
for i in `seq 0 200`; do ping -c 1 10.0.0.$i  ; done 
```

To remove:
```
for i in `seq 0 200`; do sudo ifconfig lo0 alias 10.0.0.$i  ; done 
```

On Ubuntu 20.04 you can:
```
for i in `seq 0 200`; do sudo ip addr add 10.0.0.$i/8 dev lo; done 
```

To remove:
```
for i in `seq 0 200`; do sudo ip addr del 10.0.0.$i/8 dev lo; done 
```


## Results

*On Mac*, I was able to reach 80,000.
However, mysterously a few minutes after running the experiment,
the Mac crashes without any crash reports in `/Library/Logs/DiagnosticReports` so I'm not able to diagnose what happened.
```
sysctl net | grep tcp | grep -E '(recv)|(send)'
net.inet.tcp.sendspace: 131072
net.inet.tcp.recvspace: 131072

pagesize
4096
```

80000*131072*2*2 bytes is about 39GB.
Could that be why my Mac crashed?
Was it unable handle that much virtual memory?
I'm not familiar with debugging on Mac without a crash report so if anyone has any resources let me know.

*On Linux*, I was able to reach 800,000.
However, while the experiment ran,
my mouse movements would take a few seconds to register on my screen.
Anything beyond that and my Linux would freeze and become unreponsive.

I used sysstat to investigate what resource was in contention:
https://raw.githubusercontent.com/josephmate/java-by-experiments/main/max_connections/data/out.800000.svg

TODO: clean up these commands:
```
sar -o out.$NUM_CONNECTIONS.sar -A 1 3600 2>&1 > /dev/null  &
sadf -g  out.1.sar -- -w -r -u -n SOCK -n TCP -B -S -W > out.$NUM_CONNECTIONS.svg
```

Some interesting facts:

* MBmemfree bottomed out at 117MB
* MBavail was still 1500MB
* MBmemused was only 1500MB (18% of my total 8GB)
* 1,600,454 sockets (840k server sockets and 840k client connections plus what ever else was running on my desktop)

I suspect the buffers were requested, but since only 4 bytes were needed from each, only a fraction of the buffer was used.

On linux:
https://stackoverflow.com/questions/7865069/how-to-find-the-socket-buffer-size-of-linux
```
# minimum, default and maximum memory size values (in byte)
cat /proc/sys/net/ipv4/tcp_rmem
4096    131072  6291456
cat /proc/sys/net/ipv4/tcp_wmem
4096    16384   4194304

sysctl net.ipv4.tcp_rmem
net.ipv4.tcp_rmem = 4096        131072  6291456
sysctl net.ipv4.tcp_wmem
net.ipv4.tcp_wmem = 4096        16384   4194304
```

Memory page size
```
getconf PAGESIZE
4096
```

TODO: rerun with 900,000

TODO: flame graphs
https://github.com/jvm-profiling-tools/async-profiler
```
sudo apt install openjdk-8-dbg
gdb /usr/lib/jvm/java-8-openjdk-amd64/jre/lib/amd64/server/libjvm.so -ex 'info address UseG1GC'
.
.
.
Symbol "UseG1GC" is at 0xdd9042 in a file compiled without debugging.

sudo sysctl kernel.perf_event_paranoid=1
kernel.perf_event_paranoid = 1
sudo sysctl kernel.kptr_restrict=0
kernel.kptr_restrict = 0

-agentpath:/home/joseph/lib/async-profiler-2.7-linux-x64/build/libasyncProfiler.so=start,collapsed,event=wall,file=collapsed.wall.txt

https://raw.githubusercontent.com/brendangregg/FlameGraph/master/flamegraph.pl

flamegraph.pl collapsed.wall.txt > flamegraph.svg

```

# Summary

1. Pheonix Framework achieved 2,000,000 connections
2. WhatsApp achieved 2,000,000 connections
3. Theortical limit is ~1 quadrillion (1,000,000,000,000,000)
4. You will run out of source ports (only 2^16)
5. You can fix this by creating 
6. You will run out of file descriptors
7. You can fix this by overriding the file descriptor limits of your OS
8. Java will also limit the file descriptors
9. You can override this by adding the `-XX:MaxFDLimit` JVM argument
10. Practical limit on my 16GB Mac is 80,000 
11. Practical limit on my 8GB Linux is 840,000

