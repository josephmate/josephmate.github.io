---
layout: post
title: "1,000,000 Concurrent Connections"
date: 2022-04-14 9:00
author: joseph
comments: true
categories: [Java, Server, Connections]
---

I hear the misconception that a server can only accept 65K connections
or a server consumes a port for each accepted connection all the time.
Here is a taste of some of them:

> [A TCP/IP address only supports 65,000 connections, so you would have to have to assign around 30,000 IP addresses to that server.](https://www.quora.com/Is-it-possible-to-handle-1-billion-connections-simultaneously-in-Java-with-a-single-server)

> [There are 65535 TCP port numbers, does that mean only 65535 clients can connect to a TCP server? One might think that this places a hard limit on the number of clients that a single computer/application can maintain.](https://networkengineering.stackexchange.com/questions/48283/is-a-tcp-server-limited-to-65535-clients)

> [If there is a limit on the number of ports one machine can have and a socket can only bind to an unused port number, how do servers experiencing extremely high amounts (more than the max port number) of requests handle this? Is it just done by making the system distributed, i.e., many servers on many machines?](https://serverfault.com/questions/533611/how-do-high-traffic-sites-service-more-than-65535-tcp-connections)

So I put together this article to dispel this myth from three directions:

1. WhatsApp, a chatting app you have probably used, and Phoenix, a web framework built on top of Elixir, have already demonstrated achieving millions of connections on a single server using a single port.
2. What is theoretically possible based on the TCP/IP protocol
3. A simple Java experiment anyone can run on their machine if they are still not convinced.

Jump to the summary at the end if you want to skip the details.

# Experiments

The Phoenix framework which [achieved 2,000,000 concurrent websocket connections](https://www.phoenixframework.org/blog/the-road-to-2-million-websocket-connections).
In the article they demonstrate a chat application where they simulate 2 million users, taking 1 second to broadcast to all users.
They also provide details on the technical challenges they hit with their framework to achieve that benchmark.
Some of the ideas they shared in their article, I used to write this article,
like assigning multiple IPs to overcome the 65k limit on client connections.

WhatsApp also [achieved 2,000,000](https://blog.whatsapp.com/1-million-is-so-2011).
Unfortunately, they are light on the details.
They only reveal the hardware and the OS configuration they used.

# Theoretical Max

Some think the limit 2<sup>16</sup>=65,536 because that's all the ports available in the TCP spec.
That is true for a single client making outgoing connections to a single IP, port pair.
For instance, my laptop will only be able to make 65,536 connections to
172.217.13.174 (google.com)
(probably a lot less because they will block me before I reach 65k connections).
So if you have a use case with intense intertwined communication between two machines using more than 65K concurrent connections,
the client will need to connect from a second IP address.

For a server listening on a port, each incoming connection *DOES NOT* consume a port on the server.
The server only consumes the one port that it is listening on.
Secondly connections will be coming from multiple IP addresses.
In the best case the server will be able to listen to all IP addresses, coming from all ports.

Each packet has:
1. 32 bit source IP (the IP address the connection is coming from)
2. 16 bit source port (the port on the source IP address the connection is coming from)
3. 32 bit destination IP (the IP address the connection is going to)
4. 16 bit destination port (the port on the destination IP address the connection is going to)

Then theoretical limit is 2<sup>48</sup> which is about 1 quadrillion because:

1. [number of source IP addresses]x[num of source ports]
2. because the server multiplexes the connections from the client using that.
3. 32 bits for the address and 16 bits for the port
4. Putting that all together: 2<sup>32</sup> * 2<sup>16</sup> =  2<sup>48</sup>.
5. Which is about a quadrillion (log(2<sup>48</sup> - 1)/log(10)=14.449)!

# Practical Limit
To understand the practical limit,
I put together some experiments trying to open as many TCP connections 
and have the server send and receive a message on each connection.
The workload is nowhere near as practical as
[Phoenix's](https://www.phoenixframework.org/blog/the-road-to-2-million-websocket-connections)
or
[WhatsApp's](https://blog.whatsapp.com/1-million-is-so-2011)
articles but simpler to run if you wanted to try for yourself.
You will need to overcome three battles to get the experiment to run:
the OS, the JVM, and the TCP/IP protocol.

## The experiment
If you're interested in the source code, take a look 
[here](https://github.com/josephmate/java-by-experiments/blob/main/max_connections/src/Main.java).

The pseudo code is:

```
Thread 1:
  open server socket
  for i from 1 to 1 000 000:
    accept incoming connection
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
The defaults severely limit file descriptors.
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

1. A buffer for sending
2. A buffer for receiving

The same goes for client connections.
As a result, running this experiment on a single machine will require:

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
as recommended by this [stackoverflow answer](https://superuser.com/a/1644788).

For Ubuntu 20.04, the quickest way is to:
```
sudo su
# 2^25 should be more than enough
sysctl -w fs.nr_open=33554432
fs.nr_open = 33554432
ulimit -Hn 33554432
ulimit -Sn 33554432
```


## Java File Descriptor Limits

Now that the operating system is complying,
the JVM doesn't like what you're doing with this experiment either.
When you run the experiment you still get the same or similar stack trace.

This [stackoverflow answer]( https://superuser.com/a/1398360) identifies a JVM flag as the solution:

> [`-XX:-MaxFDLimit` :     Disables the attempt to set the soft limit for the
> number of open file descriptors to the hard limit. By default, this option is
> enabled on all platforms, but is ignored on Windows. The only time that you
> may need to disable this is on Mac OS, where its use imposes a maximum of
> 10240, which is lower than the actual system maximum.](https://docs.oracle.com/en/java/javase/16/docs/specs/man/java.html)

```
java -XX:-MaxFDLimit Main 6000
```
As the above quote from the javadocs says, this is only required for the Mac.
I was able to get the experiment to run without the flag on Ubuntu.

## Source Ports

It still won't work.
You will encounter a stacktrace like:
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
Currently we have frozen the server address, server port, and client IP address.
That leaves us with only 16 bits of freedom.
As a result, we can only open 65k connections.

Our experiment goes way beyond that.
We cannot change the server IP nor the server port because that's the problem
we're exploring with this experiment.
That leaves us with changing the client IP, giving access to an additional 32 bits of freedom.
As a result, we work around this by conservatively assigning a client IP address for every 5,000 client connections.
This is the same technique they used in 
[Phoenix's experiment](https://www.phoenixframework.org/blog/the-road-to-2-million-websocket-connections).

On bigSur 11.4 you can add a bunch of fake loopback addresses with:
```
for i in `seq 0 200`; do sudo ifconfig lo0 alias 10.0.0.$i/8 up  ; done 
```

To test to make sure you IP addresses are working you can ping them:
```
for i in `seq 0 200`; do ping -c 1 10.0.0.$i  ; done 
```

To remove:
```
for i in `seq 0 200`; do sudo ifconfig lo0 alias 10.0.0.$i  ; done 
```

On Ubuntu 20.04 you need to use the `ip` tool instead:
```
for i in `seq 0 200`; do sudo ip addr add 10.0.0.$i/8 dev lo; done 
```

To remove:
```
for i in `seq 0 200`; do sudo ip addr del 10.0.0.$i/8 dev lo; done 
```


## Results

*On Mac*, I was able to reach 80,000.
However, mysteriously a few minutes after completing the experiment,
my poor Mac crashes everytime without any crash reports in `/Library/Logs/DiagnosticReports` so I'm not able to diagnose what happened.

The tcp send and receive buffers on my Mac are 131072 bytes:
```
sysctl net | grep tcp | grep -E '(recv)|(send)'
net.inet.tcp.sendspace: 131072
net.inet.tcp.recvspace: 131072
```

So it might be because I used `80000 connections*131072 bytes per buffer * 2 input and output buffer * 2 client and server connection` bytes which is about 39 GB virtual memory.
Or maybe Mac OS didn't like me using `80,000*2*2=320,000` file descriptors.
Unfortunately, I'm not familiar with debugging on Mac without a crash report so if anyone has any resources let me know.

*On Linux*, I was able to reach 840,000!
However, while the experiment ran,
my mouse movements would take a few seconds to register on my screen.
Anything beyond that and my Linux would freeze and become unresponsive.

I used [sysstat](https://github.com/sysstat/sysstat) to investigate what resource was in contention.
You can take a look at the graphs sysstat generated [here](https://raw.githubusercontent.com/josephmate/java-by-experiments/main/max_connections/data/out.840000.svg).

I used these commands to get sysstat to record hardware stats on everything and then generate graphs:
```
sar -o out.840000.sar -A 1 3600 2>&1 > /dev/null  &
sadf -g  out.840000.sar -- -w -r -u -n SOCK -n TCP -B -S -W > out.840000.svg
```

Some interesting facts:

* `MBmemfree` bottomed out at 96 MB
* `MBavail` was still 1587MB
* `MBmemused` was only 1602MB (19.6% of my total 8GB)
* `MBswpused` peeked at 1086MB (despite memory still available)
* 1,680,483 sockets (840k server sockets and 840k client connections plus whatever else was running on my desktop)
* OS decided to start using swap a few seconds into the experiment even though I had more memory

I suspect the buffers were requested, but since only 4 bytes were needed from each, only a fraction of the buffer was used.

On linux [to find the default size of your send and receive buffers you can use:](https://stackoverflow.com/questions/7865069/how-to-find-the-socket-buffer-size-of-linux)
```
# minimum, default and maximum memory size values (in bytes)
cat /proc/sys/net/ipv4/tcp_rmem
4096    131072  6291456
cat /proc/sys/net/ipv4/tcp_wmem
4096    16384   4194304

sysctl net.ipv4.tcp_rmem
net.ipv4.tcp_rmem = 4096        131072  6291456
sysctl net.ipv4.tcp_wmem
net.ipv4.tcp_wmem = 4096        16384   4194304
```

So to support all the connections, I would need 247 GB of virtual memory!
```
131072 bytes for receive 
16384 for write
(131072+16384)*2*840000
=247 GB virtual memory
```

But even if I only I load 1 page of memory because I only need to write 4 bytes
to write an integer to the buffer:
```
getconf PAGESIZE
4096

4096 bytes pagesize
(4096+4096)*2*840000
=13 GB
```
then I use 13GB from touching `2*840000` pages of memory.
I have no idea how this didn't crash!
I'm happy with 840,000 concurrent connections though.

You could improve upon my result if you have more memory,
or further optimize the OS settings like reducing the buffer size.
How many can you run? Let me know!

# Summary

1. Phoenix Framework achieved 2,000,000 connections
2. WhatsApp achieved 2,000,000 connections
3. Theoretical limit is ~1 quadrillion (1,000,000,000,000,000)
4. You will run out of source ports (only 2<sup>16</sup>)
5. You can fix this by add loopback client IP addresses
6. You will run out of file descriptors
7. You can fix this by overriding the file descriptor limits of your OS
8. Java will also limit the file descriptors
9. You can override this by adding the `-XX:MaxFDLimit` JVM argument
10. Practical limit on my 16GB Mac is 80,000 
11. Practical limit on my 8GB Linux is 840,000

