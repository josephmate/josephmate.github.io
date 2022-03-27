---
layout: post
title: "1,000,000 Concurrent Connections in Java"
date: 2022-03-31 11:59
author: joseph
comments: true
categories: [Java, Server, Connections]
---

I ran across the Pheonix framework which (achieved 2,000,000 concurrent websocket connections)[https://www.phoenixframework.org/blog/the-road-to-2-million-websocket-connections].

That got me thinking about two problems:
1. What's the theoretical max num of connections a server and client can concurrently keep open on a port?
2. What can I pratically achieve on my laptop with Java?

Jump to the summary at the end if you want to skip the details.

# Theoretical

To understand the theortical max,
you need to understand a little bit of background of TCP over IP.

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

## Java File Descriptor Limits

## Source Ports

## Results

On my Mac, I was able to reach 80,000. However, mysterously a few minutes after
running the experiment, the Mac crashes without any diagnostics so I'm not able
to diagnose what happened.

On my Linux, I was able to reach 640,000. However, while the experiment ran, my
mouse movements would take a few seconds to register on my screen. Anything
beyond that and my Linux would freeze and become unreponsive.

# Summary
1. theortical limit is ~1 quadrillion (1,000,000,000,000,000)
2. you will run out of source ports (only 2^16)
3. you can fix this by creating 
4. you will run out of file descriptors
5. you can fix this by overriding the file descriptor limits of your OS
6. Java will also limit the file descriptors
7. You can override this by adding the `-XX:MaxFDLimit` JVM argument
8. practical limit on my 16GB mac is 80,000 
9. practical limit on my 8GB Linux is 640,000

