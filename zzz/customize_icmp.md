# Customizing ICMP Payload in Ping Command


When use the term “ping,” most often we are just limiting its definition to checking whether a host can be connected to or not. In my opinion, while this purpose is correct, most of its technical details are often ignored. Many network administrators are unable to answer what ping is in details; or how it works. So, let’s unfold few details in this research-based article.

# What is PING?

Ping stands for **Packet InterNet Gopher**. Ping basically is the simplest tool to verify network connectivity. We can verify connectivity between any two devices within a private or public network.

In simple words, a simple network can be made up of two or more devices connected together using any media and properly configured as well. For example if we connect two computers using Ethernet cable and configured the IP settings properly then we can call it a network. Here ping is the first thing to do to verify the connectivity between two machines within this small network. If network is perfect then both machines A & B will be able to send and receive ping packets between each other.

The original version of the ping was written in **1983** by **Mike Muuss** at the University of California at Berkeley. Since then, several variants of the ping command have been created and implemented.

The name **Ping** comes from sonar terminology.  
In sonar, a ping is an audible sound wave sent out to find an object. If the sound hits the object, the sound waves will reflect, or echo, back to the source. The distance and location of the object can be determined by measuring the time and direction of the returning sound wave.

Some variants send only four packets and exit while others send consecutive packets until they are asked to stop. Most variants allow users to specify whether to send a request and wait for a reply or send a series of requests and wait. In the end, all variants display statistics about message loss or success and report the amount of time it takes for packets to return.

Usually, using the ping command without any option is sufficient to indicate that a problem exists, but it is not always sufficient to pinpoint problems. To help identify the exact cause of the problem, almost all variants support several options.

## Differences between Linux and Windows implementation of the ping command

In Windows, the ping command uses a 32 bytes long message and by default sends only four messages. The system that receives ping messages adds an 8-byte timestamp to messages and sends them back to the sender. The timestamp is used to calculate round-trip delays through the network.

In Linux, the ping command uses a 64 bytes long message and by default sends continuous messages until it is asked to stop. It assigns a sequence number to each message and reports when it receives the response of the message. Replies are not necessarily to be received in the same order the messages were sent out.

## ICMP PROTOCOL:

Ping uses ICMP protocol. ICMP stands for Internet Control Message Protocol. Ping basically sends ICMP echo request packet to the destination device and wait for ICMP echo reply from the destination. At the end of its output it shows a statistical summary which contains packets sent and received, packet loss and minimum, maximum and average round-trip time values. The time taken in between sending ICMP echo request and receiving ICMP echo reply is called round-trip time. Ping is not only used for troubleshooting to test connectivity but also it is used to check the response time which determines network latency. First thing ping does is to resolve the destination domain name into IP address which also test whether dns services are working properly or not.

The output of ping from windows machine which by default sends 4 packets and stop.

```
<span id="1e78" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ ping google.com</strong><br>Pinging google.com [216.58.192.14] with 32 bytes of dat<br>Reply from 216.58.192.14: bytes=32 time=306ms TTL=49<br>Reply from 216.58.192.14: bytes=32 time=303ms TTL=49<br>Reply from 216.58.192.14: bytes=32 time=302ms TTL=49<br>Reply from 216.58.192.14: bytes=32 time=304ms TTL=49</span><span id="b68b" class="nz mx fu ow b hj pe pb l pc pd" data-selectable-paragraph="">Ping statistics for 216.58.192.14:<br>    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)<br>Approximate round trip times in milli-seconds:<br>    Minimum = 302ms, Maximum = 306ms, Average = 303ms</span>
```

The output of ping from Linux machine which by default continue pinging until ctrl+c is pressed to cancel.

```
<span id="8f01" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ ping google.com</strong><br>PING google.com (216.58.220.14) 56(84) bytes of data.<br>64 bytes from bom05s05-in-f14.1e100.net (216.58.220.14): icmp_seq=1 ttl=52 time=26.0 ms<br>64 bytes from bom05s05-in-f14.1e100.net (216.58.220.14): icmp_seq=2 ttl=52 time=25.9 ms</span>
```

# FPING:

Manual page of utility fping explains it beautifully in simple words as fping is a program like ping which uses the Internet Control Message Protocol (ICMP) echo request to determine if a target host is responding. fping differs from ping in that you can specify any number of targets on the command line, or specify a file containing the lists of targets to ping. Instead of sending to one target until it times out or replies, fping will send out a ping packet and move on to the next target in a round-robin fashion. In the default mode, if a target replies, it is noted and removed from the list of targets to check; if a target does not respond within a certain time limit and/or retry limit it is designated as unreachable.

```
<span id="db00" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ fping google.com yahoo.com facebook.com<br></strong>google.com is alive<br>yahoo.com is alive<br>facebook.com is alive</span><span id="1671" class="nz mx fu ow b hj pe pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ fping google.com yahoo.com facebook.com nogoogle.com</strong><br>google.com is alive<br>yahoo.com is alive<br>facebook.com is alive<br>nogoogle.com is unreachable</span>
```

We can make file and add IPs or hostnames in the file to feed fping command.

```
<span id="ee38" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ cat ips.txt</strong><br>google.com<br>facebook.com<br>yahoo.com<br>nogoogle.com</span><span id="e1eb" class="nz mx fu ow b hj pe pb l pc pd" data-selectable-paragraph="">#fping -f ips.txt<br>google.com is alive<br>yahoo.com is alive<br>facebook.com is alive<br>nogoogle.com is unreachable</span>
```

With fping we use switch -f for reading file containing destination hosts. This option can only be used with root user. For standard user follow the command below.

```
<span id="d132" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ fping &lt; ips.txt</strong><br>google.com is alive<br>yahoo.com is alive<br>facebook.com is alive<br>nogoogle.com is unreachable</span>
```

We can also provide range of IP addresses like below.

```
<span id="554d" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ fping 192.168.10.{249,250,251,252}</strong><br>192.168.10.249 is alive<br>ICMP Host Unreachable from 192.168.10.249 for ICMP Echo sent to 192.168.10.250<br>ICMP Host Unreachable from 192.168.10.249 for ICMP Echo sent to 192.168.10.250<br>ICMP Host Unreachable from 192.168.10.249 for ICMP Echo sent to 192.168.10.251<br>ICMP Host Unreachable from 192.168.10.249 for ICMP Echo sent to 192.168.10.251<br>ICMP Host Unreachable from 192.168.10.249 for ICMP Echo sent to 192.168.10.252<br>ICMP Host Unreachable from 192.168.10.249 for ICMP Echo sent to 192.168.10.252<br>192.168.10.250 is unreachable<br>192.168.10.251 is unreachable<br>192.168.10.252 is unreachable</span>
```

# HPING:

hping is another sister command of ping to test the network. In some distributions it is known as hping3 and not installed by default. hping3 is a network tool able to send custom TCP/IP packets and to display target replies like ping program does with ICMP replies.

It also uses the same switches like ping for basic functionalities like -h for help and -c for count etc. Here are few basic examples for hping3.

```
<span id="95a7" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ hping3 localhost -c 3</strong><br>HPING localhost (lo 127.0.0.1): NO FLAGS are set, 40 headers + 0 data bytes<br>len=40 ip=127.0.0.1 ttl=64 DF id=10586 sport=0 flags=RA seq=0 win=0 rtt=3.5 ms<br>len=40 ip=127.0.0.1 ttl=64 DF id=10657 sport=0 flags=RA seq=1 win=0 rtt=2.3 ms<br>len=40 ip=127.0.0.1 ttl=64 DF id=10668 sport=0 flags=RA seq=2 win=0 rtt=2.4 ms</span><span id="bc61" class="nz mx fu ow b hj pe pb l pc pd" data-selectable-paragraph="">--- localhost hping statistic ---<br>3 packets transmitted, 3 packets received, 0% packet loss<br>round-trip min/avg/max = 2.3/2.7/3.5 ms</span>
```

For sending TCP packet to specific port like 443 -p switch is used. Below it will send 3 packets to port 443 of localhost.

```
<span id="145f" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ hping3 localhost -c 3 -p 443</strong><br>HPING localhost (lo 127.0.0.1): NO FLAGS are set, 40 headers + 0 data bytes<br>len=40 ip=127.0.0.1 ttl=64 DF id=10925 sport=443 flags=RA seq=0 win=0 rtt=1.2 ms<br>len=40 ip=127.0.0.1 ttl=64 DF id=11032 sport=443 flags=RA seq=1 win=0 rtt=1.2 ms<br>len=40 ip=127.0.0.1 ttl=64 DF id=11164 sport=443 flags=RA seq=2 win=0 rtt=1.2 ms</span><span id="c0a8" class="nz mx fu ow b hj pe pb l pc pd" data-selectable-paragraph="">--- localhost hping statistic ---<br>3 packets transmitted, 3 packets received, 0% packet loss<br>round-trip min/avg/max = 1.2/1.2/1.2 ms</span>
```

For more verbosity we can use -V option which shows more details to help analyzing network.

```
<span id="57c0" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ hping3 localhost -c 3 -p 443 -V</strong><br>using lo, addr: 127.0.0.1, MTU: 65536<br>HPING localhost (lo 127.0.0.1): NO FLAGS are set, 40 headers + 0 data bytes<br>len=40 ip=127.0.0.1 ttl=64 DF id=15202 tos=0 iplen=40<br>sport=443 flags=RA seq=0 win=0 rtt=1.1 ms<br>seq=0 ack=2084985995 sum=3ce urp=0</span><span id="1987" class="nz mx fu ow b hj pe pb l pc pd" data-selectable-paragraph="">len=40 ip=127.0.0.1 ttl=64 DF id=15246 tos=0 iplen=40<br>sport=443 flags=RA seq=1 win=0 rtt=1.3 ms<br>seq=0 ack=1926997853 sum=9b8c urp=0</span><span id="8e68" class="nz mx fu ow b hj pe pb l pc pd" data-selectable-paragraph="">len=40 ip=127.0.0.1 ttl=64 DF id=15314 tos=0 iplen=40<br>sport=443 flags=RA seq=2 win=0 rtt=4.6 ms<br>seq=0 ack=1033548582 sum=12b6 urp=0<br></span><span id="79c4" class="nz mx fu ow b hj pe pb l pc pd" data-selectable-paragraph="">--- localhost hping statistic ---<br>3 packets transmitted, 3 packets received, 0% packet loss<br>round-trip min/avg/max = 1.1/2.3/4.6 ms</span>
```

# Testing a loopback interface

TCP/IP protocol stack provides a loopback interface. To test whether the TCP/IP stack is properly implemented on the local system, you can send a ping request to the loopback interface. If the loopback interface replies to ping messages, the TCP/IP protocol stack is properly configured. The IP addresses of the loopback interface are **127.0.0.1** and **::1** in IPv4 and IPv6, respectively.

To test the IPv4 implementation, use the following command.

```
<span id="e0d8" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ ping 127.0.0.1</strong></span>
```

To test the IPv6 implementation, use the following command.

```
<span id="42e2" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ ping ::1</strong></span>
```

Instead of IP addresses, you can also use the hostname of the system.

```
<span id="caec" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ ping localhost</strong></span>
```

# Troubleshooting Ping command

## No reply for ping

You might notice that certain hosts do not reply to the ping request. It seems like the ping command has hanged because there is no response. The command just stays there, it doesn’t even times out.

## Destination host unreachable

This error can occurred because of one of the two reasons:

-   Either the local system has no route to the remote host
-   or the end point router has no route to the remote host

If you only see the ‘destination host unreachable’ error, this means your system couldn’t find a route to the remote host.

On the other hand, if you see the error in the “Reply from <IP>” part of the reply, it means that the packet was sent outside of your network but it couldn’t reach the destination.

Some times servers also block the ICMP traffic that could show this error.

## Request times out

This error means that the packets reached the remote server but the reply could not reach your system. The issue could be lost packets or routing error.

# What is OSI Model?

Open system interconnection (OSI) reference model consists of seven layers or seven steps which concludes the overall communication system.

It is important to understand this OSI model as each of the software applications works based on one of the layers in this model.

## Architecture Of The OSI Reference Model

![](https://miro.medium.com/v2/resize:fit:888/0*OvZE3DA94tAodrgT.png)

## Relationship Between Each Layer

Let’s see how each layer in the OSI reference model communicates with one another with the help of the below diagram.

![](https://miro.medium.com/v2/resize:fit:700/0*p_Z3v4oU73BAf-ZH.png)

**Enlisted below is the expansion of each Protocol unit exchanged between the layers:**

-   **APDU**– Application protocol data unit.
-   **PPDU**– Presentation protocol data unit.
-   **SPDU**– Session protocol data unit.
-   **TPDU**– Transport protocol data unit (Segment).
-   **Packet**– Network layer host-router protocol.
-   **Frame**– Data-link layer host-router protocol.
-   **Bits**– Physical layer host-router protocol.
-   Layer 1, 2 and layer 3 combined are responsible for the effective transmission of ICMP packets.

## Ethernet, IP and ICMP Protocols

While “pinging” a system, layers 1, 2 and 3 are in action. Each layer defines its own “_format of protocol options_” known as a **header.** It precedes the actual data to be sent and has information such as the source IP, destination IP, checksum, type of protocol etc. And when it is attached with the actual data to be sent, it forms a protocol data unit which is renamed as per the layer.

Default protocol data units (PDU) in these layers are:

-   Layer 1: bits
-   Layer 2: frame
-   Layer 3: packet
-   Layer 4: datagram

We’ll be using these terms quite often and it is important to be clear what these terms mean. While the data is travelling through these layers, it is converted virtually into its own PDU.

## Ethernet

IEEE 802.3 is a set of standards and protocols that define Ethernet-based networks. Ethernet technologies are primarily used in LANs. There are a number of versions in IEEE 802.3 such as the 802.3a, 802.3i etc. When we ping, there is some data sent to the destination host and in a specific format.

## Ethernet header:

![](https://miro.medium.com/v2/resize:fit:800/1*PfEuXEdbIIVOaEVtParh2w.png)

## IP (Internet Protocol)

It is the principal communications protocol for relaying datagrams across network boundaries. IP has the task of delivering packets from the source to the host solely based on the IP address in the packet headers. IP defines packet structures that encapsulate the data and headers.

## ICMP (Internet Control Message Protocol)

Since IP does not have an inbuilt mechanism for error control and sending control messages, ICMP is here to provide that functionality. It is a supporting protocol used by network devices like routers for sending error messages.

# Options of ping command:

![](https://miro.medium.com/v2/resize:fit:2000/1*xYpYkLZDGtt_acJnm1dxzw.png)

![](https://miro.medium.com/v2/resize:fit:2000/1*NckHpOQB4rnE6CycW_0Row.png)

Ping sends out IP packets with ICMP\_Echo\_Request to the destination and waits for its reply (IP packet with ICMP\_Echo\_Reply).

The ICMP packet is encapsulated in an IPv4 packet. The general composition of the **IP datagram** is as follows:

![](https://miro.medium.com/v2/resize:fit:1116/0*-tBKcyb_u0s691u-)

When a ping command is sent to the host, the datagram is encapsulating Ethernet header, IP header, ICMP header and payload too. The minimum size of an IPv4 header is **20 bytes** and the maximum size is **60 bytes.**

**The default size of payload data in ping in windows is 32 bytes.** Let’s add 20 bytes of IP header in it and 8 bytes of ICMP header. 32+20+8, it comes out to be 60 bytes.

But, when we analyse ping in Wireshark, the size of the frame written in the log is 74 bytes. This is because while pinging, we would need the destination and source MAC address as well, which is available in the **Ethernet header.**

So, 14 + 20 + 8 + 32 = 74 bytes.

So, **ping** sends an encapsulated IP packet which is a combination of:

Ethernet header + IP header + ICMP header + ICMP payload

# Wireshark and Basic Filters

**Wireshark** is the world’s foremost and widely-used network protocol analyzer. It lets you see what’s happening on your network at a microscopic level. It’s a freely available network monitoring tool that is able to log all the traffic incoming and outgoing from a single NIC card. Please use [this](https://www.wireshark.org/docs/wsug_html_chunked/ChapterIntroduction.html) tutorial for setting up and start logging using Wireshark.

![](https://miro.medium.com/v2/resize:fit:1400/1*E2fQlSdptj7tuqHPIdmuXg.png)

![](https://miro.medium.com/v2/resize:fit:1400/1*_pcJdsn7DmwqYjD53VnVGA.png)

Let’s go step by step and see what just happened.

1.  I’ve pinged IP address 192.168.217.128 from my source IP 192.168.238.1 and captured just one packet to see clearly the action.
2.  On the top, there is a Wireshark filter applied for the IP address that goes like **addr == 192.168.217.128**
3.  Some other useful Wireshark filters are:  
    **http or dns**
4.  Next, you can see the total length of the datagram sent, that is, 74 bytes, as already explained above.
5.  Then you can see, in yellow, Ethernet frame. Here, you’ll find the Ethernet header.
6.  After that is the IPv4 packet. IP is used to deliver packets from source to destination based on the IP addresses.
7.  Finally, we see the ICMP header and payload datap.
8.  All these layers are involved in a simple ping over IEEE 802.3.

# Let us first understand why do we use hexadecimal?

For memory readouts, values are also often in hexadecimal. Even braille is coded in hexadecimal.

There are a couple obvious reasons why hexadecimal is preferable to the standard binary that computers store at the low level.

1.  Readability. Hexadecimal uses digits that more closely resemble our usual base-10 counting system and it’s therefore easier to decide at a glance how big a number like e7 is as opposed to 11100111.
2.  Higher information density. With 2 hexadecimal digits, we can express any number from 0 to 255. To do the same in binary, we need 8 digits. As we get bigger and bigger numbers, we start needing more and more digits and it becomes harder to deal with.

## _Why don’t we use decimal?_

The main problem with decimal can be illustrated by the following graph:

![](https://miro.medium.com/v2/resize:fit:1304/0*FMq1-gGNNWxJFATB.png)

Each purple tick is when a new digit is added when representing numbers

Notice that binary and decimal never line up, whereas hexadecimal and binary do, in fact, they do it every 4 binary digits. In practice, this means that one digit of hexadecimal can always be represented with 4 digits of binary. Decimal doesn’t work this way. The key property that allows hexadecimal to work this way is that it has a base which is a multiple of 2. (2⁴ for hexadecimal). For this reason, any number system we choose to compress binary data should have a base that is a multiple of 2.

## _Why don’t we use higher bases like base 128 or 256?_

Base 128 and base 256 both follow the rule we outlined above (2⁷ and 2⁸ respectively), why don’t we use them?

This comes down entirely to readability. Hexadecimal uses the Arabic digits 0–9 and the English letters a-f. To represent base 128 we would need to add new characters like / or (. Imagine trying to decide which number was bigger, $#@/ or $\*(). Because we have no inherent ordering in our minds for whether @ or ) are bigger, it is a lot harder to reason about numbers that use characters outside of the standard alphabet.

Base 64 encoding solves this problem by using capital letters for an extra 26, then also using + and / to fill in the last 2 spots.

![](https://miro.medium.com/v2/resize:fit:762/0*315z6hrX8Di_xkYK.png)

Great! So we should just be using base 64 then, right? It’s easy to understand and compresses information well!

Well, not quite. This is where things get interesting. Our bases actually tell us something else that is important to consider, especially when programming. One base 32 (2⁵) digit maps to 5 binary digits. One base 64 (2⁶) digit maps to 6 binary digits.

```
<span id="dbe1" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph="">Base Binary digits per character<br>2    1<br>4    2<br>8    3<br>16   4<br>32   5<br>64   6<br>128  7<br>256  8</span>
```

# The Byte

Bytes are units of information that consist of 8 bits. Almost all computers are byte-addressed, meaning all memory is referenced by byte, instead of by bit. This fact means that bytes come up all the time in programming. Using a counting system that can easily convert into bytes is another important requirement for our binary representation. Base 256 is ideal for this, since it’s 2⁸, meaning one digit is exactly one byte. Unfortunately, for reasons we went over above, 256 would have other problems for readability and be very unwieldy to use.

And now we’re getting to why we use hexadecimal. Hexadecimal, base 16, or base 2⁴, represents exactly half a byte. Each byte can be fully represented by two hexadecimal digits, no more, no less. Hexadecimal also fits all of our other specifications:

1.  It successfully compresses data. one hex digit can represent 0–15, much better than the 0–1 that binary offers.
2.  It is easy to read.
3.  It easily converts to bytes. Two hex digits = 1 byte.

![](https://miro.medium.com/v2/resize:fit:682/0*-r7Z_TVwfUtEXlS5.png)

Criteria for a good byte representation scheme

# How is a datagram sent to the destination?

When a datagram is sent from source to destination, in our case, when we ping, the format of the datagram is as follows:

![](https://miro.medium.com/v2/resize:fit:1208/0*aIMgd8klr29S6LDE)

We’ll ping a destination with default options:

```
<span id="9bfb" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ ping 192.128.217.128</strong></span>
```

Let’s start analyzing the traffic in Wireshark now.

I just captured a single datagram in Wireshark when I pinged a destination host and analyzed it layer by layer.

Talking about Layer 2 here, we can see that Ethernet type 2 is being used.

![](https://miro.medium.com/v2/resize:fit:1400/0*u40HFrAzdWXKB8Au)

As we talked in point 2, Ethernet frame sent with ping includes the source and destination MAC along with the type of protocol being used.

![](https://miro.medium.com/v2/resize:fit:1128/1*dNP144DGP24hRath7SmOHw.png)

The most important field here is the EtherType field as on analyzing these bytes we can find the protocol that was used in the communication. This is really essential for forensics point of view and some of the most important hex values are:

![](https://miro.medium.com/v2/resize:fit:1400/1*WJ5lPsHPGN3MX3mW5fLMfw.png)

Moving on to layer 3, we see an IP packet, since obviously ping is using IP protocol to check if a host is alive or not. Here is the observation of the hex dump of IP packet:

![](https://miro.medium.com/v2/resize:fit:1400/0*QWMk2A-3Wtm8PtQd)

![](https://miro.medium.com/v2/resize:fit:1400/1*ZO-f1jyRTV5buHkYt5eQsA.png)

From the analysis of this IP packet, we observe that IP header is then added to the Ethernet frame to make a packet that has the above-mentioned options specified. By analyzing this data we can observe a lot of different things. Some of which are:

![](https://miro.medium.com/v2/resize:fit:1400/1*YBEMXNQOVFQaDbKhghybpA.png)

One really interesting thing to note is the Header Length. We see that header length is written as 45 in hex which comes out to be 69 which is not possible. On further breaking down this byte, we see that the first nibble (half byte) tells the version of the protocol.

4 = hex value = 4 (version of protocol), that is IPv4

5 = hex value = IHL (Internet Header Length) = next for bits

And so on (total length = 00 3c = 60 bytes)

A datagram is completed and transported when ICMP header is added in Layer 3. We know that IP lacks error control mechanism, so ping uses ICMP to serve that purpose. ICMP protocol is specified in IP header as we saw above and hence, it is important to analyse ICMP header to better understand the underlying mechanism of ping.

![](https://miro.medium.com/v2/resize:fit:1400/0*UPCav8J-zkbn6auw)

![](https://miro.medium.com/v2/resize:fit:1308/1*ZHn8R2MtfzD52Tn5UbIPhg.png)

**Type:** It is the type of request of ICMP request sent. According to IANA, there are 45 assigned requests.

![](https://miro.medium.com/v2/resize:fit:1400/1*kgAlsii_60JkNzrSlFyuhw.png)

We can see, 08 as the Type of request which symbolizes Echo request. When one system pings another system, it sends a Type 8 request and if the host is alive, the host sends back Type 0 (Echo Reply) request.

**Code:** It is simply the hex value of the type of ICMP request message. It ranges from 0 to 15 for each of the types. This reference is taken from Wikipedia:

![](https://miro.medium.com/v2/resize:fit:1400/0*cwsvQ9YRf1OEbMCU)

**Checksum:** “The checksum is the 16-bit one’s complement of the one’s complement sum of the ICMP message starting with the ICMP Type. For computing the checksum, the checksum field should be zero. If the total length is odd, the received data is padded with one octet of zeros for computing the checksum. This checksum may be replaced in the future.” — RFC 792

So, to calculate the checksum, we have to split the ICMP header and payload data into multiple pairs of 2 bytes each, calculate the one’s complement of first two bytes, add with the next one and repeat it till all the bytes are exhausted.

Here, checksum status is shown as good and correct, so we need not look any further.

**Data:** Contains other essential data like timestamp, sequence number, magic number etc.

# Data size in ping

According to RFC 791, an IP can handle a datagram of maximum size 65,535 bytes. However, by default, the data in ping is **32 bytes** and the whole datagram is by default **60 bytes** in case of windows. Adding 14 extra bytes of Ethernet header, it comes out to be **74 bytes** which we see in Wireshark. Focus on the colour codes here:

Red= ICMP payload data size

Green= Total length of the IPv4 packet

Maroon= Total length of the datagram

Let’s talk about Linux now. The default IP packet size in Linux’s case is **56 bytes.** Adding extra 28 bytes of IP and ICMP header makes it **84 bytes**. Adding more 14 Ethernet frame header bytes makes it **98 bytes.**

However, we can add data as per our wish, as long as it remains under 65,535 bytes using the ping –l option in Windows. This also has some limitations.

# MTU in IEEE 802.3i

MTU stands for Maximum Transmission Unit and it is the size of the largest Protocol Data Unit that can be communicated in a single network-layer operation. IEEE 802.3 standards restrict the minimum to 48 bytes and the maximum to 1500 bytes. So, if I am to transfer more than 1500 bytes of data in a single datagram, it would be fragmented into multiple packets.

# Fragmentation in ping

IP fragmentation or ping fragmentation is a process in which a packet exceeding the size of MTU is broken into multiple pieces and transacted to the destination host. RFC 791 has the procedure for IP fragmentation, transmission and reassembly of packets.

-   Let’s increase the size of the data payload to 2000 bytes and let’s see what happens.

```
<span id="503e" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ ping -l 2000 192.128.217.128</strong></span>
```

Now, we can see that fragmentation is done and the datagram is split into two fragments of size 1480 and 520 bytes. Why?

![](https://miro.medium.com/v2/resize:fit:1400/1*VrT_H705X-gY1_gYmScZtQ.png)

The first fragment is 1500 bytes in total length but the data is only 1480 because 20 bytes are always in reserve for IP protocol header, and the second fragment is 520 bytes in payload data size.

**First fragment= 1480 (data size) + 20 (IP header)**

**Second fragment= 520 (data size) + 20 (IP header) + 8 (ICMP header)**

It is important to note that ICMP header is not added in the first fragment and it will only be added when the whole datagram is complete. It is because ICMP is responsible for error control and it calculates the checksum. For checksum to be calculated, whole data should be transmitted first and hence, ICMP header is not added in the initial fragments but only in the last fragment.

![](https://miro.medium.com/v2/resize:fit:1400/1*s6Xb4lYR1DqkSYPv17smuw.png)

-   Now we push these limits to the maximum size.

```
<span id="9c4d" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ ping -l 65500 192.128.217.128</strong></span>
```

Here, the datagram is split into multiple 44 fragments of 1480 bytes each and the last packet is 408 (minus 28= 380 bytes) bytes in size.

You must not get confused here. 65500 is the maximum size of a payload that we can specify in ping.

However, each of the 44 frames is of size 1500 bytes and the last frame is 388+20+8 bytes, which comes out to be 66416 bytes. Hence, the cumulative size of a packet datagram can be at max **66416 bytes.**

Adding extra 14 bytes of Ethernet header, it comes out to be: **67,046** bytes.

## Anomaly in Ping

Let’s focus on a case when the payload size is specified as 0. Let’s see what the protocols do with 0 bytes of data.

```
<span id="f6b5" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph="">$ ping -l 0 192.128.217.128</span>
```

As expected, the request goes with 42 bytes (20 + 8 + 14) with 0 bytes of data but wait, why does the reply come back as 60 bytes?

On analyzing the hex dump, we see that a Linux system is replying with 18 null bytes of data.

![](https://miro.medium.com/v2/resize:fit:1400/0*tVbXPvgwx8M5UVt1)

If we specify **ping –s 0** in Linux machine, maybe we get some answer.

![](https://miro.medium.com/v2/resize:fit:1192/0*GsJXHq9JJoIW8Eyt)

Okay, so in the terminal, we see “pinging with 0 (28) bytes of data” which means 28 bytes of IP plus ICMP header and 0-byte data. BUT, Wireshark shows us something size of 60 bytes and has again added an extra 18 bytes.

![](https://miro.medium.com/v2/resize:fit:1400/0*phldoQSYpABWemqi)

Turns out that these 18 bytes are Ethernet padding bytes and that’s why this anomaly has arisen. IEEE 802.3 adds extra bytes if data is less than 18 bytes in case of Linux known as padding bytes or pad bytes.

Let’s specify a data in the range: 1<x<18 for example, 10 bytes. Let’s see what happens.

![](https://miro.medium.com/v2/resize:fit:1216/0*622xigWqtn2R92Yy)

In Wireshark, we see the size is still 60 bytes. And 10 bytes have been written by the hex values from 0 to 9, while rest of them are still null bytes.

![](https://miro.medium.com/v2/resize:fit:1400/0*c34uapA_kSSNvFQU)

It is safe to say the same anomaly will exist when we specify data in the multiples of 1480 since the last fragment will have 0 bytes of data to be sent and padding would be added.

This was an interesting anomaly and made us understood some details of the IEEE 802.3 standards difference in Windows and Linux.

# Now, since we are done with exploring the details, let us manipulate the ICMP payload data.

-   There are multiple methods for customizing.

> **We will be using Wireshark (as shown above) to check the payload.**

## 1\. Modifying the ping command

## a. What about `ping -p <pattern>`?

Keep in mind that not all of versions of `ping` support `-p` option.

You may specify up to 16 ‘’pad’’ bytes to fill out the packet you send. This is useful for diagnosing data-dependent problems in a network.

For example, `ping -p ff` will cause the sent packet to be filled with all ones.

E.x: `ping -p 486920686572652e www.example.com`, _where 486920686572652e = Hi here._

I have created a python program which will help convert the string into hex and will send it using the `-p` option of ping command

> GitHub URL: [gursimarh/custompayload-ping (github.com)](https://github.com/gursimarh/custompayload-ping)

## b. Improving ping -p pattern:

Alternatively, we can use `ping -s32 -p$(echo 'abcdefghijklmnop' | hexdump -e '"%02X"') 127.0.0.1` . This will get us close.

Here, `-s` specifies the size; `-p` specifies a pattern (in hex). The standard Linux `ping -p` will only accept 16 bytes for its repeating pattern (`a-p`), though, whereas the Windows `ping` uses a 24-byte repeating pattern (`a-w`), so the Linux one is effectively sending `abcdefghijklmnopabcdefghijklmnop`. `nping` would probably be more-suitable, as it allows you to specify a full payload.

## 2\. HPING3

Hping3’s implementation makes the actual construction and transmission of a crafted packet transparent to the user.  
The tool easily assembles and sends custom ICMP/UDP/TCP packets, and displays target replies in the same way ping does with ICMP replies.

**Lets craft some files to be sent over network using hping**

Boot up the redhat machine and install the hping command

```
<span id="8a8f" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph="">$ yum install hping3</span>
```

Create a file to be sent over the payload

```
<span id="7db3" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph="">$ cat &lt;&lt;EOF &gt;$(pwd)/hello.txt<br>Hey world this hping3.<br>Data is being sent over the network using ICMP.<br>EOF</span>
```

Before sending the data we must ensure that both the system should be connected over the bridge / Host only network.  
It should be able to ping each other.  
IP of both the VM.

```
<span id="7b0f" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ ifconfig</strong><br><strong class="ow fv">$ ip addr show</strong> # If Ifconfig does not work<br><strong class="ow fv">$ ip a s</strong> # Also an alternative to check the IP</span>
```

Now send the file over the network using hping3 and receive the packets using tcpdump

```
<span id="a4d3" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""># On RedHat Machine <br><strong class="ow fv">$ hping3 -1 -E ./hello.txt -u -d 1500 192.168.217.128</strong><br># -1 is used to let hping3 now we are going to send ICMP request.<br># -E is used to tell hping3 the file we are going to sent<br># -u Please indicate the user when the transfer is complete <br># -d It is to indicate the size of the packet.<br># When its done you will see the EOF reached in the sender system<br>ctrl+c<br># To stop it sending the data further.</span><span id="79e0" class="nz mx fu ow b hj pe pb l pc pd" data-selectable-paragraph=""># On ubuntu machine <br><strong class="ow fv">$ sudo tcpdump -v -l enp0s3 'icmp and 192.168.217.128' -w file</strong><br># When its transmitted through the sender system  <br>ctrl+c # Press to stop</span>
```

We can analyze the data using Wireshark as discussed earlier.

## 3\. NPING

There are so many ping variations out there: ping, fping, hping3, psping, hrping. I am sure I will discover a couple more in the following years. It’s very well maintained and updated utility, while some of the ones listed above are unmaintained and haven’t aged well over the years.

Nping can do so much more than simple ICMP pinging. It can manipulate pretty much any parameter and field of TCP, UDP, ICMP packets, and it can be used for network stack stress testing, ARP poisoning, Denial of Service attacks, route tracing, etc.

## Installation

Nping is part of the nmap package and it’s installed by default if you install nmap on your Linux system with:

```
<span id="ec69" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">$ yum install nmap</strong></span>
```

Nmap and nping are supported on Linux, Windows, Mac OS X, so if you learn to use it on one platform your knowledge is transferable to the rest. That’s not the case for other utilities, such as hping, fping, and psping.

On Windows specifically you’d need to install the npcap driver for nping to work properly. If you run the nmap or the nping installer (or you have Wireshark installed), the drivers will be installed automatically.

```
<span id="07e2" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph=""><strong class="ow fv">Example: A representative Nping execution</strong><br><br># <strong class="ow fv">nping -c 1 --tcp -p 80,433 scanme.nmap.org google.com</strong><br><br>           Starting Nping ( https://nmap.org/nping )<br>           SENT (0.0120s) TCP 96.16.226.135:50091 &gt; 64.13.134.52:80 S ttl=64 id=52072 iplen=40  seq=1077657388 win=1480<br>           RCVD (0.1810s) TCP 64.13.134.52:80 &gt; 96.16.226.135:50091 SA ttl=53 id=0 iplen=44  seq=4158134847 win=5840 &lt;mss 1460&gt;<br>           SENT (1.0140s) TCP 96.16.226.135:50091 &gt; 74.125.45.100:80 S ttl=64 id=13932 iplen=40  seq=1077657388 win=1480<br>           RCVD (1.1370s) TCP 74.125.45.100:80 &gt; 96.16.226.135:50091 SA ttl=52 id=52913 iplen=44  seq=2650443864 win=5720 &lt;mss 1430&gt;<br>           SENT (2.0140s) TCP 96.16.226.135:50091 &gt; 64.13.134.52:433 S ttl=64 id=8373 iplen=40  seq=1077657388 win=1480<br>           SENT (3.0140s) TCP 96.16.226.135:50091 &gt; 74.125.45.100:433 S ttl=64 id=23624 iplen=40  seq=1077657388 win=1480<br><br>           Statistics for host scanme.nmap.org (64.13.134.52):<br>            |  Probes Sent: 2 | Rcvd: 1 | Lost: 1  (50.00%)<br>            |_ Max rtt: 169.720ms | Min rtt: 169.720ms | Avg rtt: 169.720ms<br>           Statistics for host google.com (74.125.45.100):<br>            |  Probes Sent: 2 | Rcvd: 1 | Lost: 1  (50.00%)<br>            |_ Max rtt: 122.686ms | Min rtt: 122.686ms | Avg rtt: 122.686ms<br>           Raw packets sent: 4 (160B) | Rcvd: 2 (92B) | Lost: 2 (50.00%)<br>           Tx time: 3.00296s | Tx bytes/s: 53.28 | Tx pkts/s: 1.33<br>           Rx time: 3.00296s | Rx bytes/s: 30.64 | Rx pkts/s: 0.67<br>           Nping done: 2 IP addresses pinged in 4.01 seconds</span>
```

## Usage

The simplest thing you can do with nping is to test if a port of any kind (TCP, UDP, ICMP) is open.

**ICMP Mode**

There are several ICMP utilities, and most of them do the basics of sending an echo request packet, waiting for an echo reply and messing the latency. Nping includes that functionality as shown below (you might need to run this with elevated privileges).

```
<span id="941f" class="nz mx fu ow b hj pa pb l pc pd" data-selectable-paragraph="">$ <strong class="ow fv">nping — icmp -c 2 google.com</strong></span>
```

The -c option just tells nping to send 2 packets instead of the default 5.

## Payload Options

`--data _<hex string>_` (Append custom binary data to sent packets)

This option lets you include binary data as payload in sent packets. `_<hex string>_` may be specified in any of the following formats: `0xAABBCCDDEEFF_<...>_`, `AABBCCDDEEFF_<...>_` or `\xAA\xBB\xCC\xDD\xEE\xFF_<...>_`. Examples of use are `--data 0xdeadbeef` and `--data \xCA\xFE\x09`. Note that if you specify a number like `0x00ff` no byte-order conversion is performed. Make sure you specify the information in the byte order expected by the receiver.

`--data-string _<string>_` (Append custom string to sent packets)

This option lets you include a regular string as payload in sent packets. `_<string>_` can contain any string. However, note that some characters may depend on your system's locale and the receiver may not see the same information. Also, make sure you enclose the string in double quotes and escape any special characters from the shell. Example: `--data-string "Jimmy Jazz..."`.

`--data-length _<len>_` (Append random data to sent packets)

This option lets you include `_<len>_` random bytes of data as payload in sent packets. `_<len>_` must be an integer in the range \[0–65400\]. However, values higher than 1400 are not recommended because it may not be possible to transmit packets due to network MTU limitations.

## We have seen a lot of details about Ping command and few methods of customizing the ICMP Payload in Ping Command.

Source:
https://gursimarsm.medium.com/customizing-icmp-payload-in-ping-command-7c4486f4a1be
