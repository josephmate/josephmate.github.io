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


# Experiments

The Pheonix framework which (achieved 2,000,000 concurrent websocket connections)[https://www.phoenixframework.org/blog/the-road-to-2-million-websocket-connections]
(WhatsApp also achieved 2,000,000)[https://blog.whatsapp.com/1-million-is-so-2011].
Both used

Jump to the summary at the end if you want to skip the details.

# Theoretical Max

Most think it's 2^16=65536 because that's all the port available.
That's true for a client making a connection to an IP, port pair.
For instance, my laptop will only be able to make 65000 connections to Google (probably a lot less due to NAT).

However, for a server listening on a port, connections will be coming from multiple IP addresses.
So a server listening to one point, on one IP address in the best case be able
to listen to all IP address, coming from all ports on the IP addresses.

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

On my Mac, I was able to reach 80,000.
However, mysterously a few minutes after running the experiment,
the Mac crashes without any diagnostics so I'm not able to diagnose what happened.

On my Linux, I was able to reach 800,000.
However, while the experiment ran,
my mouse movements would take a few seconds to register on my screen.
Anything beyond that and my Linux would freeze and become unreponsive.

I used sysstat
https://raw.githubusercontent.com/josephmate/java-by-experiments/main/max_connections/data/out.800000.svg

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
11. Practical limit on my 8GB Linux is 640,000

