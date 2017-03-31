---
layout: page
title: "Project 3: Build Your Own Router"
group: "Project 3"
---

* toc
{:toc}

## Before You Start
You are allowed to use some high-level abstractions, including C++11 extensions, for parts that are not directly related to networking, such as string parsing, multi-threading.
We will also accept implementations written in C, however use of C++ is preferred.

You are required to use `git` to track the progress of your work. **The project can receive a full grade only if the submission includes git history no shorter than 3 commits.**

You are encouraged to host your code in private repositories on [GitHub](https://github.com/), [GitLab](https://gitlab.com), or other places.  At the same time, you are PROHIBITED to make your code for the class project public during the class or any time after the class.  If you do so, you will be violating academic honestly policy that you have signed, as well as the student code of conduct and be subject to serious sanctions.
{: class="alert alert-danger"}

## Overview
In this project, you will implement a functional IP Simple Router (`sr`) that is able to route real traffic. 

You will be given an incomplete router to start with. What you need to do is to implement the Address Resolution Protocol (ARP), basic IP forwarding, and ICMP ping and traceroute. A correctly implemented `sr` should be able to forward traffic for any IP applications, including downloading files between a web browser and an Apache server via your router. We provide you with the skeleton code for the `sr`, which doesn’t implement any packet processing or forwarding, and will be filled in by you.

draw a topology figure here.

This is the network topology, and `vrhost` is the router that you will work on. The router connects a client to two subnets, each of which has two web servers in it. The goal of this project is to implement essential functionality in the router so that you can use a regular web browser on the client to access any of the four web servers, and also be able to ping and traceroute the servers. 

## Setting Up Environment 

I need some time to figure out how to write the script to set up the environment.

## Getting Started

Start all VMs using vagrant, log onto the router. Download the `sr` code (sr.tar) from CCLE. To run the code, untar the code package `tar xvf sr.tar`, and compile the code by `make`. 

Useful commands:

- `./sr -t`: start the router.

- `./sr -r routing_table_file`: allows you to specify the routing table to load. By default, it loads the routing table from file `rtable` (included in configuration files).

- `./sr -h`: print the list of acceptable command-line options.

The routing table is read from the file `rtable`. Each line of the file has four fields: 

        Destination Network, gateway(i.e., nexthop), mask, interface

A valid rtable file may look as follows: (Note: You should not assume any particular order of the routing table entries. E.g., the default route may be the first, second, or the third entry.)

        0.0.0.0 172.24.74.17 0.0.0.0  eth0
        172.24.74.64 0.0.0.0 255.255.255.248  eth1
        172.24.74.80 0.0.0.0 255.255.255.248  eth2

0.0.0.0 as the destination means that this is the default route; 0.0.0.0 as the gateway means that it is the same as the destination address of the incoming packet.

On connection the interface information will be printed and looks like the following: 

        Router interfaces:
        eth0    HWaddr c6:31:9f:bb:4b:6e
            inet addr 172.29.0.9
        eth1    HWadd rcb:6c:4f:12:a5:2d
            inet addr 172.29.0.10
        eth2    HWaddr 85:e4:4d:99:e1:2c
            inet addr 172.29.0.12

To test if the router is actually receiving packets, try access one of the web servers by running the following command on the client:

        wget http://ServerIP:16280

where ServerIP is the IP of one of the servers. `sr` should print out that it has received a packet. However, `sr` will not do anything with the packet, so you will not see any reply and wget will time out.

## Hints for Developing the Router

### Important Data Structures
The Router (`sr_router.h`): The full context of the router is housed in the struct `sr_instance` (`sr_router.h`). `sr_instance` contains information about topology the router is routing for as well as the routing table and the list of interfaces.

Interfaces (`sr_if.c`, `sr_if.h`): The `sr` code creates a linked-list of interfaces, `if_lis`t, in the router instance. Utility methods for handling the interface list can be found at `sr_if.h`, `sr_if.c`. Note that IP addresses are stored in network order, so you shouldn't apply htonl() when copying an address from the interface list to a packet header.

The Routing Table (`sr_rt.c`, `sr_rt.h`): The routing table in the stub code is read from a file (default filename 'rtable', can be set with command line option -r) and stored in a linked-list, `struct sr_rt * routing_table`, as a member of the router instance.

### The First Methods to Get Acquainted With

The two most important methods for developers to get familiar with are:

in `sr_router.c`

        void sr_handlepacket(struct sr_instance* sr, 
                uint8_t * packet /* lent */,
                unsigned int len,
                char* interface /* lent */)
                
This method is invoked each time a packet is received. The *packet points to the packet buffer which contains the full packet including the Ethernet header (but without Ethernet preamble and CRC). The length of the packet and the name of the receiving interface are also passed into the method as well.

in `sr_vns_comm.c`

        int sr_send_packet(struct sr_instance* sr /* borrowed */, 
                uint8_t* buf /* borrowed */ ,
                unsigned int len, 
                const char* iface /* borrowed */)

This method allows you to send out an Ethernet packet of certain length ('len'), via the outgoing interface 'iface'. Remember that the packet buffer needs to start with an Ethernet header.

**Thus the `sr` code already implemented receiving and sending packets. What you need to do is to fill in `sr_handlepacket( )` with packet processing that implements ARP and IP forwarding.**

### Downloading Files from Web Servers
Once you've correctly implemented the router, you can visit the web page located at `http://ServerIP:16280/` by using GUI browser, text-based browser like lynx, or command-line tools such as curl and wget, from the client. 'ServerIP' is the IP address of one of your servers. The application servers serve some files via HTTP, FTP, and also host a simple UDP service. You will see how to access them when you get to the front web page.

### Dealing with Protocol Headers

Within the `sr` framework you will be dealing directly with raw Ethernet packets, which includes Ethernet header and IP header. There are a number of online resources which describe the protocol headers in detail. For example, find IP, ARP, and Ethernet on [www.networksorcery.com](http://www.networksorcery.com). The stub code itself provides data structures in sr_protocols.h for IP, ARP, and Ethernet headers, which you can use. You can also choose to use standard system header files found in `/usr/include/net and /usr/include/netinet` as well.

With a pointer to a packet (`uint8_t *`), you can cast it to an Ethernet header pointer (`struct sr_ethernet_hdr *`) and access the header fields. Then move the pointer pass the Ethernet header and cast it to a pointer to ARP header or IP header, and so on. This is how you access different protocol headers.

### Inspecting Packets with `tcpdump`

`tcpdump` can server as an important debugging tool. As you work with the `sr` router, you will want to take a look at the packets that the router is sending and receiving on the wire. The easiest way to do this is by logging packets to a file and then displaying them using a program called `tcpdump`.

First, tell your router to log packets to a file in the `tcpdump` format:
        
        ./sr -t -l logfile

As the router runs, it will record all the packets that it receives and sends (including headers) into the file named `logfile`. After stopping the router, you can use `tcpdump` to display the contents of the `logfile`:

        tcpdump -r logfile -e -vvv –xx

The `-r` switch tells `tcpdump` to read logfile, `-e` tells `tcpdump` to print the headers of the packets, not just the payload, `-vvv` makes the output very verbose, and `-xx` displays the content in hex, including the link-level (Ethernet) header.

**Learn to read the hexadecimal output from `tcpdump`. It shows you the packet content including the Ethernet header. You can see how a correctly formatted ARP request (coming from the gateway) looks like, and check where your packet might have problem.**

### Troubleshooting of the Topology

You can view the status of your topology nodes
        
        ./vnltopo.sh gateway status
        ./vnltopo.sh vrhost status
        ./vnltopo.sh server1 status
        ./vnltopo.sh server2 status

If your topology does not work correctly, you can attempt to reset it, or notify the TA.

        ./vnltopo.sh gateway run
        ./vnltopo.sh server1 run
        ./vnltopo.sh server2 run

## Task Description

### Milestone 1: Answering ARP Requests

When the router is running and you initiate web access to one of the servers, the very first packet that the router receives will be an ARP request, sent by the gateway node to the router asking the Ethernet address of the router.

In the first milestone, you need to

- Get familiar with the `sr` code
- Process the ARP request
- Send back a correct ARP reply
- Receive the next packet, which should be a TCP SYN packet going to one of the web servers.

The correct behavior can be verified by logging the packets and viewing them with `tcpdump`.

### Milestone 2: Implementing a working router

In the 2nd milestone, you'll need to implement the rest of ARP and the basic IP forwarding. More specifically the required functionalities are listed as follows:

1. The router correctly handles ARP requests and replies. When it receives an ARP request, it can send back a correctly formatted ARP reply. When it wants to send an IP to the nexthop and doesn't know the nexthop's Ethernet address, it can send an ARP request and parse the returned ARP reply to get the Ethernet address.
2. The router maintains an ARP cache whose entries should be refreshed every time a matching packet passes, and should be removed after 15s of no activity
3. The router can successfully forward packets between the gateway and the application servers. When an IP packet arrives, the router should do the following:

    - Check if the destination address is itself. If yes, discard the packet.
    - Decrement the TTL by 1. If the result is 0, discard the packet. Otherwise, update the checksum field. Refer the textbook for the IP Checksum algorithm.
    - Look up the routing table to find out the IP address of the nexthop. 
    - Check ARP cache for the Ethernet address of the nexthop. If needed, it should send an ARP request to get the Ethernet address. 
    - While the router is waiting for ARP reply, it should store incoming packets that use the same nexthop. 
    - After the ARP reply is received, save the information in ARP cache, and send out all the packets that are waiting for the ARP reply.
4. Using the router, you can download files from the servers.
5. The IP checksum algorithm is covered in this course. A great way to test if your checksum function is working is to run it on an arriving packet. If you get the same checksum that is already contained in the packet, then your function is working. Remember to zero out the checksum field when you feed the packet to your checksum calculation. If the checksum is wrong, `tcpdump` will complain.

## Grading

The project will be graded on a different topology. Don’t hardcode anything about your topology in the source code. 

You can work by yourself or in a group of two students. No group can have more than two students.

Your code must be written in C or C++ and use the stub code.


We will test your code by two ways:

1. Access the web server from the client. E.g., 

       wget http://ServerIP:16280 (this retrieves the web front page from the server.)
       wget http://ServerIP:16280/64MB.bin (this retrieves a 64MB file.)

2. Log packets and analyze the router’s behavior. E.g., Retrieve a web page, then wait for 20s, then retrieve the page again. From the traffic log, we should see ARP request/reply at the beginning of each retrieval, but not during the retrieval because of ARP cache. We will also use this way to verify TTL decrement and checksum.

Grading is based on functionality (i.e., what works and what doesn’t), not the source code (i.e., what has been written). For example, when a required functionality doesn’t work, its credit will be deducted, regardless of whether it’s caused by a trivial oversight in the code vs. a serious design flaw.

## Submission

Only one submission per group.

1. Name your working directory "sr".

2. Make sure this directory has all the source files and the Makefile. Include a README file, listing the names and emails of group members, and anything you want us to know about your router. Especially when it only works partially, it helps to list what works and what not.

3. Create a tarball 

   cd sr
   make clean
   cd ..
   tar -zcf sr.tgz sr

4. Upload sr.tgz to CCLE. 



