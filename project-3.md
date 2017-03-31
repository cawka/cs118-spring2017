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

Within the `sr` framework you will be dealing directly with raw Ethernet packets, which includes Ethernet header and IP header. There are a number of online resources which describe the protocol headers in detail. For example, find IP, ARP, and Ethernet on [www.networksorcery.com](http://www.networksorcery.com). The stub code itself provides data structures in sr_protocols.h for IP, ARP, and Ethernet headers, which you can use. You can also choose to use standard system header files found in /usr/include/net and /usr/include/netinet as well.

With a pointer to a packet (uint8_t *), you can cast it to an Ethernet header pointer (struct sr_ethernet_hdr *) and access the header fields. Then move the pointer pass the Ethernet header and cast it to a pointer to ARP header or IP header, and so on. This is how you access different protocol headers.


## Grading

The project will be graded on a different topology. Don’t hardcode anything about your topology in the source code. 

You can work by yourself or in a group of two students. No group can have more than two students.

Your code must be written in C or C++ and use the stub code.

Your router will be graded using vagrant VMs only.

Grading is based on functionality (i.e., what works and what doesn’t), not the source code (i.e., what has been written). For example, when a required functionality doesn’t work, its credit will be deducted, regardless of whether it’s caused by a trivial oversight in the code vs. a serious design flaw.

## Submission

Only one submission per group.

1. Name your working directory “topXX”, where XX is the topology ID in your assignment.

2. Make sure this directory has all the source files and the Makefile. Include a README.txt file listing the names and emails of group members, and anything you want us to know about your router. Especially when it only works partially, it helps to list what works and what not.

3. Create a tarball 

   cd topXX
   make clean
   cd ..
   tar -zcf topXX.tgz topXX

4. Upload topXX.tgz onto D2L. 



