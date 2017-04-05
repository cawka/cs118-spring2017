---
layout: page
title: "Project 3: Build Your Own Router"
group: "Project 3"
---

* toc
{:toc}

## 1. Overview

In this project, you will be writing a simple router with a static routing table. Your router will receive raw Ethernet frames. It will process the packets just like a real router, then forward them to the correct outgoing interface. We'll make sure you receive the Ethernet frames; your job is to create the forwarding logic so packets go to the correct interface.

Your router will route real packets from a emulated host (client) to two emulated application servers (http server 1/2) sitting behind your router. The application servers are each running an HTTP server. When you have finished the forwarding path of your router, you should be able to access these servers using regular client software. In addition, you should be able to ping and traceroute to and through a functioning Internet router. The topology is shown below:

<p align="center">
  <img src="./figures/topo-sample.png" width="240">
</p>

If the router is functioning correctly, all of the following operations should work:

- Pinging from the client to any of the router's interfaces (192.168.2.1, 172.64.3.1, 10.0.1.1).
- Tracerouting from the client to any of the router's interfaces
- Pinging from the client to any of the app servers (192.168.2.2, 172.64.3.10)
- Tracerouting from the client to any of the app servers
- Downloading a file using HTTP from one of the app servers
 
This assignment runs on top of Mininet which was built at
Stanford. Mininet allows you to emulate a topology on a single
machine. It provides the needed isolation between the emulated nodes
so that your router node can process and forward real Ethernet frames
between the hosts like a real router. You don't have to know how
Mininet works to complete this assignment, but more information about
Mininet (if you're curious) is available [here](http://mininet.org/).

You are allowed to use some high-level abstractions, including C++11 extensions, for parts that are not directly related to networking, such as string parsing, multi-threading.
We will also accept implementations written in C, however use of C++ is preferred.

You are required to use `git` to track the progress of your work. **The project can receive a full grade only if the submission includes git history no shorter than 3 commits.**

You are encouraged to host your code in private repositories on [GitHub](https://github.com/), [GitLab](https://gitlab.com), or other places.  At the same time, you are PROHIBITED to make your code for the class project public during the class or any time after the class.  If you do so, you will be violating academic honestly policy that you have signed, as well as the student code of conduct and be subject to serious sanctions.
{: class="alert alert-danger"}

## 2. Task Description

There are three main parts in this assignment: Handling ARP,
Forwarding IP, and handling ICMP.
 
### 2.1 Protocols to Understand

- Ethernet 
 
  You are given a raw Ethernet frame and have to send raw [Ethernet
  frames](https://en.wikipedia.org/wiki/Ethernet_frame). You should understand source and destination MAC addresses
  and the idea that we forward a packet one hop by changing the
  destination MAC address of the forwarded packet to the MAC address
  of the next hop's incoming interface.


- [Address Resolution Protocol](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) ([RFC826](https://tools.ietf.org/html/rfc826))

  ARP is needed to determine the next-hop MAC address that corresponds
  to the next-hop IP address stored in the routing table. Without the
  ability to generate an ARP request and process ARP replies, your
  router would not be able to fill out the destination MAC address
  field of the raw Ethernet frame you are sending over the outgoing
  interface. Analogously, without the ability to process ARP requests
  and generate ARP replies, no other router could send your router
  Ethernet frames. Therefore, your router must generate and process
  ARP requests and replies. To lessen the number of ARP requests sent
  out, you are required to cache ARP replies. Cache entries should
  time out after 15 seconds to minimize staleness. The provided ARP
  cache class already times the entries out for you. When forwarding a
  packet to a next-hop IP address, the router should first check the
  ARP cache for the corresponding MAC address before sending an ARP
  request. In the case of a cache miss, an ARP request should be sent
  to a target IP address about once every second until a reply comes
  in. (*If the ARP request is sent five times with no reply, an ICMP
  destination host unreachable is sent back to the source IP. However,
  in this project, you are not required to implement this logic.*) The
  provided ARP request queue will help you manage the request
  queue. In the case of an ARP request, you should only send an ARP
  reply if the target IP address is one of your router's IP
  addresses. In the case of an ARP reply, you should only cache the
  entry if the target IP address is one of your router's IP
  addresses. Note that ARP requests are sent to the broadcast MAC
  address (ff-ff-ff-ff-ff-ff). ARP replies are sent directly to the
  requester's MAC address.

- [Internet Protocol](https://en.wikipedia.org/wiki/IPv4) ([RFC791](https://tools.ietf.org/html/rfc791))

  Before operating on an IP packet, you should verify its checksum and make sure it meets the minimum length of an IP packet. You should understand how to find the longest prefix match of a destination IP address in the routing table. If you determine that a datagram should be forwarded, you should correctly decrement the TTL field of the header and recompute the checksum over the changed header before forwarding it to the next hop.

- [Internet Control Message Protocol](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol) ([RFC792](https://tools.ietf.org/html/rfc792))

  ICMP is a simple protocol that can send control information to a host. In this assignment, your router will use ICMP to send messages back to a sending host. You will need to properly generate the following ICMP messages (including the ICMP header checksum) in response to the sending host under the following conditions:

  - Echo reply (type 0): Sent in response to an echo request (ping) to one of the router's interfaces. (This is only for echo requests to any of the router's IPs. An echo request sent elsewhere should be forwarded to the next hop address as usual.)
  - Port unreachable (type 3, code 3): Sent if an IP packet containing a UDP or TCP payload is sent to one of the router's interfaces. This is needed for traceroute to work.
  - Time exceeded (type 11, code 0): Sent if an IP packet is discarded during processing because the TTL field is 0. This is also needed for traceroute to work. The source address of an ICMP message can be the source address of any of the incoming interfaces, as specified in RFC 792. As mentioned above, the only incoming ICMP message destined towards the router's IPs that you have to explicitly process are ICMP echo requests. You may want to create additional structs for ICMP messages for convenience, but make sure to use the packed attribute so that the compiler doesn't try to align the fields in the struct to word boundaries.

### 2.2 Details of IP Forwarding 
Given a raw Ethernet frame, if the frame contains an IP packet that is not destined towards one of our interfaces:

- Sanity-check the packet (meets minimum length and has correct checksum).
- Decrement the TTL by 1, and recompute the packet checksum over the modified header.
- Find out which entry in the routing table has the longest prefix match with the destination IP address.
- Check the ARP cache for the next-hop MAC address corresponding to the next-hop IP. If it's there, send it. Otherwise, send an ARP request for the next-hop IP (if one hasn't been sent within the last second), and add the packet to the queue of packets waiting on this ARP request. Obviously, this is a very simplified version of the forwarding process, and the low-level details follow. For example, if an error occurs in any of the above steps, you will have to send an ICMP message back to the sender notifying them of an error. You may also get an ARP request or reply, which has to interact with the ARP cache correctly.
 
If an incoming IP packet is destined towards one of your router's IP addresses, you should take the following actions, consistent with the section on protocols above:

- If the packet is an ICMP echo request and its checksum is valid, send an ICMP echo reply to the sending host.
- If the packet contains a TCP or UDP payload, send an ICMP port unreachable to the sending host. Otherwise, ignore the packet. Packets destined elsewhere should be forwarded using your normal forwarding logic.

## 3. Environment Setup

Clone project template

```bash
git clone https://github.com/cs118/spring17-project3 ~/cs118-proj3
cd ~/cs118-proj3
```

Initialize VM
```bash
vagrant up
```

To establish an SSH session to the created VM, run

```bash
vagrant ssh
```

        
**NOTE: do not update the VM's Ubuntu OS, as this project will ONLY work on the given Ubuntu OS version**

## 4. Code Overview

### 4.1 The starter source code
You can build and run the starter code as follows:

        $ cd ~/project3/router/
        $ make
        $./sr

### 4.2 Functions You Need to Implement

Your router receives a raw Ethernet frame and sends raw Ethernet
frames when sending a reply to the sending host or forwarding the
frame to the next hop. The basic functions to handle these functions are:

```bash
void sr_handlepacket(struct sr_instance* sr, uint8_t * packet, unsigned int len, char* interface)
```

This method, located in sr_router.c, is called by the router each time
a packet is received. The "packet" argument points to the packet
buffer which contains the full packet including the ethernet
header. The name of the receiving interface is passed into the method
as well.

```bash
int sr_send_packet(struct sr_instance* sr, uint8_t* buf,  unsigned int len, const char* iface)
```
		
This method, located in sr\_vns\_comm.c, will send an arbitrary packet of length, len, to the network out of the interface specified by iface.

You should not free the buffer given to you in sr\_handlepacket (this is why the buffer is labeled as being "lent" to you in the comments). You are responsible for doing correct memory management on the buffers that sr\_send\_packet borrows from you (that is, sr\_send\_packet will not call free on the buffers that you pass it).

```bash
void sr_arpcache_sweepreqs(struct sr_instance *sr)
```

The assignment requires you to send an ARP request about once a second until a reply comes back or we have sent five requests. This function is defined in sr_arpcache.c and called every second, and you should add code that iterates through the ARP request queue and re-sends any outstanding ARP requests that haven't been sent in the past second. If an ARP request has been sent 5 times with no response, a destination host unreachable should go back to all the sender of packets that were waiting on a reply to this ARP request.

### 4.3 Data Structures
 
- The Router (sr\_router.h):
The full context of the router is housed in the struct sr_instance (sr\_router.h). sr\_instance contains information about topology the router is routing for as well as the routing table and the list of interfaces.

- Interfaces (sr\_if.c/h):
After connecting, the server will send the client the hardware
information for that host. The stub code uses this to create a linked list of interfaces in the router instance at member if\_list. Utility methods for handling the interface list can be found at sr_if.c/h.

- The Routing Table (sr\_rt.c/h):
The routing table in the stub code is read on from a file (default filename "rtable", can be set with command line option -r ) and stored in a linked list of routing entries in the current routing instance (member routing_table).

- The ARP Cache and ARP Request Queue (sr\_arpcache.c/h):
You will need to add ARP requests and packets waiting on responses to those ARP requests to the ARP request queue. When an ARP response arrives, you will have to remove the ARP request from the queue and place it onto the ARP cache, forwarding any packets that were waiting on that ARP request. Pseudocode for these operations is provided in sr\_arpcache.h. The base code already creates a thread that times out ARP cache entries 15 seconds after they are added for you. You must fill out the sr\_arpcache\_sweepreqs function in sr\_arpcache.c that gets called every second to iterate through the ARP request queue and re-send ARP requests if necessary. Psuedocode for this is provided in sr_arpcache.h.

- Protocol Headers (sr\_protocol.h)
Within the router framework you will be dealing directly with raw Ethernet packets. The stub code itself provides some data structures in sr\_protocols.h which you may use to manipulate headers easily.

### 4.4 Debugging Functions
We have provided you with some basic debugging functions in sr\_utils.h, sr\_utils.c. Feel free to use them to print out network header information from your packets. Below are some functions you may find useful:

- print\_hdrs(uint8\_t \*buf, uint32_t length) - Prints out all possible headers starting from the Ethernet header in the packet
- print_addr\_ip\_int(uint32\_t ip) - Prints out a formatted IP
  address from a uint32_t. Make sure you are passing the IP address in
  the correct byte ordering.

### 4.5 Logging Packets
You can log the packets receives and generated by your SR program by
using the "-l" parameter with your SR program. The file will be in
pcap format, i.e., you can use wireshark or tcpdump to read it.

```bash
$ ./sr -l logname.pcap
```

Besides SR, you can also use mininet to monitor the traffic goes in
and out of the emulated nodes, i.e., router, server1 and
server2. Mininet provides direct access to each emulated node. Take
server1 as an example, to see the packets in and out of it, go to
mininet CLI:

```bash
mininet> server1 sudo tcpdump -n -i server1-eth0
```

or you can bring up a terminal inside server1 by

```bash
mininet> xterm server1
```

then inside the newly popped xterm,

```bash
$ sudo tcpdump -n -i server1-eth0
```

### 4.6 Length of Assignment
This assignment is considerably harder than the first 2 labs, so get started early.

In our reference solution, we added 520 lines of C, including
whitespace and comments. Of course, your solution may need fewer or more lines of code, but this gives you a rough idea of the size of the assignment to a first approximation.

To help you debug your topologies and understand the required behavior we provide a reference binary and you can find it at ~/project3/sr\_solution in your directory.

## 5. Grading 
### 5.1 Grading Criteria
 
- The router must successfully route packets between the Internet and the application servers.
- The router must correctly handle ARP requests and replies.
- The router must correctly handle traceroutes through it (where it is not the end host) and to it (where it is the end host).
- The router must respond correctly to ICMP echo requests.
- The router must handle TCP/UDP packets sent to one of its interfaces. In this case the router should respond with an ICMP port unreachable.
- The router must maintain an ARP cache whose entries are invalidated after a timeout period (timeouts should be on the order of 15 seconds).
- The router must queue all packets waiting for outstanding ARP replies. If a host does not respond to 5 ARP requests, the queued packet is dropped and an ICMP host unreachable message is sent back to the source of the queued packet.
- The router must not needlessly drop packets (for example when waiting for an ARP reply)
- The router must enforce guarantees on timeouts--that is, if an ARP
  request is not responded to within a fixed period of time, the ICMP
  host unreachable message is generated even if no more packets arrive
  at the router. (Note: You can guarantee this by implementing the
  sr\_arpcache\_sweepreqs function in sr\_arpcache.c correctly.)

### 5.1 Deductions

The submission archive contains temporary or other non-source code
file, except `README.md`, `Vagrantfile`, files under `.git` subfolder
(and any files from extra credit part).

### 5.2 Extra Credit

## 6. Acknowledgement
 
This project is largely based on the CS144 class project by Professor
Philip Levis and Professor Nick McKeown, Stanford University.
 
 
