---
layout: page
title: Project 2
---

* toc
{:toc}

# Project 2: Simple TCP-like Transport Protocol over UDP

<!-- ## Revisions -->

## Overview

The purpose of this project is to learn the basics of TCP protocol, including connection establishment and congestion control.
You will need to implement a simple data transfer client and server applications, using UDP sockets as an unreliable transport.
Your client should receive data even though some packets are dropped or delayed.

## Task Description

The developed client should establish connection to the developed server, after which the server will need to transfer data to the client.
Individual packets sent by the client and the server may be lost or reordered.
Therefore, your task is to make reliable data transfer by implementing parts of TCP such as the sequence number, the acknowledgement, and the congestion control.

The project will be graded by testing transfer a single file over the unreliable link.
Your second project does not need to be extended from your first project (if you extend, you will get extra credit, see below).
Instead, you can simply implement a single threaded server that starts data transfer as soon as the connection is established.

## Basic Transmission over UDP

- In this project, you will be implementing a simple congestion window based reliable transport protocol similar to TCP.
 You must write one program implementing the client side, and another program implementing the server side. Only C/C++ are allowed to use in this project as you did in the first project.

- The client and the server will communicate using the User Datagram Protocol (UDP) socket, which does not guarantee data delivery.

- The initial and **minimum** congestion window size starts from 1024 bytes. **Make sure your congestion windows doesn't go below 1024**

- The fixed retransmission timeout value 500 milliseconds (adaptive RTO for the extra credit)

- The client should establish a connection to the server using a three-way handshake.  During the handshake, the server has to send its initial sequence number to the client.

- After establishing the connection, the server should send the specified in the command-line file, dividing it into multiple packets and adding necessary header information. The maximum packet size should be 1032 bytes including all the headers (max 1024 bytes of payload).

- Command-line specification for client and server program:

        ./client SERVER-HOST-OR-IP PORT-NUMBER

        ./server PORT-NUMBER FILE-NAME

- You should use the packet format defined later in the project description.

- Note that your programs will act as both a network application (file transfer program) as well as a reliable
transport layer protocol built over the unreliable UDP transport layer.


## Error Handling

Although using UDP does not ensure reliability of data transfer, the actual rate of packet loss or corruption in LAN
may be too low to test your program.  Therefore, we are going to emulate packet loss by using `tc` command in linux.

- The file transmission should be completed successfully, unless the packet loss rate is 100%.
- The timer on both the client and the server should work correctly to retransmit the lost packet.

### Examples to emulate packet loss

If are using the [Vagrantfile provided in project-2 skeleton](https://github.com/cawka/spring17-cs118-project2), you can automatically instantiate two virtual machines that are connected to each other using a private network (`eth1` interface on each) with emulated 10% loss and delay of 20ms in each direction.

You can use the following commands to adjust parameters of the emulation:

- To check the current parameters for the private network (`eth1`)

        tc qdisc show dev eth1

- To change current parameters to loss rate of 20% and delay 100ms:

        tc qdisc change dev eth1 root netem loss 20% delay 100ms

- To delete the network emulation:

        tc qdisc del dev eth1 root

- If network emulation hasn't yet setup or you have deleted it, you can add it to, e.g., 10% loss without delay emulation:

        tc qdisc add dev eth1 root netem loss 10%


- You can also change network emulation to re-order packets.  The command below makes 4 out of every 5 packets (1-4, 6-9, ...) to be delayed by 100ms, while every 5th packet (, 10, 15, ...) will be sent immediately:

        tc qdisc change dev eth1 root netem gap 5 delay 100ms

        # or if you're just adding the rule
        # tc qdisc add dev eth1 root netem gap 5 delay 100ms

- More examples can be found in [Network Emulation tutorial](http://www.linuxfoundation.org/collaborate/workgroups/networking/netem)

If you are not using the provided Vagrant, you should adjust network interface to one that matches your experiments.

## Congestion Control

Your server has to adjust the congestion window size depending on the link status.  **Note that for the full credit you need implement only TCP Tahoe congestion window adjustment logic**:

- *Slow start*: the congestion window size will be doubled until it meets the slow start threshold value (`ssthresh`).
- *Congestion Avoidance*: the congestion window size added by one if the congestion window size is larger than `ssthresh`.
- If a packet is lost as detected by the retransmission timeout

  * The server sets `ssthresh` to max(1/2 of current congestion window, 1024) and
  * set congestion window to 1024 bytes

In other words, after timeout, the sender adjusts the value of `sshthresh` and moves back to *Slow Start* stage.  As soon as the value of `cwnd` reaches `sshthresh`, it switches to *Congestion Avoidance*.  As soon as packet lost, the server again adjusts `sshthresh`, set `cwnd` to 1, and restart from *Congestion Avoidance* stage.

When you done with the basic project, you can continue with implementing TCP Reno and/or TCP NewReno congestion window adjustment logic for extra credit.

## Hints

The best way to approach this project is in incremental steps.  Do not try to implement all of the functionality at once.

- First, assume there is no packet loss. Just have the server send a packet, the receiver respond with an ACK,
and so on.

- Second, introduce a large file transmission. This means you must divide the file into multiple packets and transmit the packets based on the current congestion window size.

- Third, introduce packet loss.  Now you have to add a timer at the first sent and unacked packet. There should be one timeout whenever data segments are sent out. Also congestion control features should be implemented for the successful file transmission.

The credit of your project is distributed among the required functions. If you only finish part of the requirements, we still
give you partial credit. So please do the project incrementally.



## Requirements

- You must use an UDP socket for both sender and receiver.

- You should print messages to the screen when the server or the client is sending or receiving packets. There
are four types of output messages and should follow the formats below.

    * Server: Sending packets

            "Sending packet" [Sequence number] [CWND] [SSThresh] ("Retransmission") ("SYN") ("FIN")

        Example:

	        Sending packet 5095 1024 1024 SYN
	        Sending packet 5096 16384 15360
	        Sending packet 6000 16384 15360
	        Sending packet 6020 16384 15360
	        Sending packet 5096 1024 15360 Retransmission
	        Sending packet 6040 16384 15360 FIN

    * Server: Receiving packets

	        "Receiving packet" [ACK number]

        Example:

            Receiving packet 5096
            Receiving packet 6020

    * Client: Sending packets

	        "Sending packet" [ACK number] ("Retransmission") ("SYN") ("FIN")

        Example:

            Sending packet 5095 SYN
            Sending packet 5096
            Sending packet 6000
            Sending packet 6020
            Sending packet 6020 Retransmission
            Sending packet 6050 FIN

    * Client: Receiving packets

            "Receiving packet" [Sequence number]

        Example:

            Receiving packet 5096
            Receiving packet 6020

- The maximum packet size is 1032 bytes including a header (1024 bytes for the payload)

- The receiver and congestion window sizes must be in **the unit of bytes**, not in the unit of packet count. For example, if the minimum of the congestion and receiver windows is 5000 bytes, then four packets with payload of 1024 bytes each can be transmitted simultaneously (including length, each packet will be 1032 bytes) or five packets with payload of 1000 bytes (i.e., if you want, you can send payload less than maximum).

- The sequence number is given in **the unit of bytes** as well. The maximum sequence number should correspond to **30 Kbytes (30720 bytes)**. You have to reset back the sequence number when it reaches the maximum value.

- Packet retransmission should be triggered when he timer times out.

- Here are the default values for some variables.

    * Maximum packet length (including all your headers): **1032 bytes**
    * Maximum sequence number: **30720 bytes**
    * Initial and minimum congestion window size: **1024 byte**
    * Initial slow start threshold: **<del>30720</del> 15360 bytes**
    * Retransmission time out value: **500 ms**
    * The basic client's receiver window can be always **<del>30720</del> 15360 bytes**, but the server should be able to properly handle cases when the window is reduced.

- Simple TCP header format

    * Your simple TCP header should follow the below format which is 8 bytes fixed size.

             0                   1                   2                   3
             0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |        Sequence Number        |     Acknowledgment Number     |
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
            |             Window            |          Not Used       |A|S|F|
            +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

    * `Sequence Number` (16 bits): The sequence number of the first data octet in this packet (except when SYN is present). If SYN is present the sequence number is the initial sequence number (ISN) and the first data octet is ISN+1.

    * `Acknowledgement Number` (16 bits): If the ACK control bit is set this field contains the value of the next sequence number the sender of the segment is expecting to receive.  Once a connection is established this is always sent.

    * `Window` (16 bits): The number of data octets the sender of this packet is willing to accept.

    * `Not Used` (13 bits): Must be zero.

    * `A` (ACK, 1 bit): Indicates that there the value of `Acknowledgment Number` field is valid

    * `S` (SYN, 1 bit): Synchronize sequence numbers (TCP connection establishment)

    * `F` (FIN, 1 bit): No more data from sender (TCP connection termination)

- The client should save file to `received.data` in the current working directory.

- After server finishes transmission, it should terminate the connection using FIN/FIN-ACK procedure.  The client should implement TIME-WAIT mechanism, e.g., using 2*RTO as a waiting time.

## Environment Setup

The best way to guarantee full credit for the project is to do project development using a Ubuntu 14.04-based virtual machine.

You can easily create an image in your favourite virtualization engine (VirtualBox, VMware, Docker) using the Vagrant platform and steps outlined below.

### Set up Vagrant and create VM instance

**Note that all example commands are executed on the host machine (your laptop), e.g., in Terminal.app (or iTerm2.app) on OS X, cmd in Windows, and console or xterm on Linux.  After the last step (`vagrant ssh`) you will get inside the virtual machine and can compile your code there.**

- Download and install your favourite virtualization engine, e.g., [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

- Download and install [Vagrant tools](https://www.vagrantup.com/downloads.html) for your platform

- Set up project and VM instances

  * Clone [project template](https://github.com/cawka/spring17-cs118-project2)

        git clone https://github.com/cawka/spring17-cs118-project2 ~/cs118-proj2
        cd ~/cs118-proj2

  * Initialize VMs, one to run the client app and the other to run the server app

        vagrant up

        # Or you can set up them individually
        # vagrant up client
        # vagrant up server

  * To establish an SSH session to the created VM, run

    To ssh to the client VM

        vagrant ssh
        # or vagrant ssh client

    To ssh to the server VM

        vagrant ssh server

  If you are using Putty on Windows platform, `vagrant ssh` (`vagrant ssh server`) will return information regarding the IP address and the port to connect to your virtual machine.

- Work on your project

  All files in `~/cs118-proj2` folder on the host machine will be automatically synchronized with `/vagrant` folder on both virtual machines.  For example, to compile your code, you can run the following commands:

        vagrant ssh
        cd /vagrant
        make

### Notes

* If you want to open another SSH session, just open another terminal and run `vagrant ssh` (or create a new Putty session).

* The client and server VMs are connected using the private network `10.0.0.0/24` (`eth1`)

  - client's IP address: `10.0.0.2`
  - server's IP address: `10.0.0.1`

  Note that these addresses do not mean that your server and client must support only this environment.  The server and client should be able to work a linux host with any other IP addresses as well.

* If you are using Windows, read [this article](http://www.sitepoint.com/getting-started-vagrant-windows/) to help yourself set up the environment.

* The code base contains the basic `Makefile` and two empty files `server.cpp` and `client.cpp`.

        $ vagrant ssh
        vagrant@vagrant-ubuntu-xenial-64:~$ cd /vagrant
        vagrant@vagrant-ubuntu-xenial-64:/vagrant$ ls
        Makefile  README.md  Vagrantfile  client.cpp  server.cpp

* You are now free to add more files and modify the Makefile to make the `server` and `client` full-fledged implementation.


## Submission

To submit your project, you need to prepare:

1. All your source code and Makefile (no binaries) as `.tar.gz` archive.

2. A PDF project report that describes
   * the high level design of your server and client;
   * the problems your ran into and how you solved the problems;
   * additional instructions to build your project (if your project uses some other libraries);
   * how you tested your code and why.
   * the contribution of each team member (up to 3 members in one team) and their UID

Put all these above into a package and submit to CCLE.

Please make sure:

1. your code can compile
2. no unnecessary files in the package.

Otherwise, your will not get any credit.

## Grading

Your code will be first checked by a software plagiarism detecting tool. If we find any plagiarism, you will not get any credit.

Your code will then be automatically tested in some testing scenarios. If your code can pass all our automated test cases, you will get the full credit.

###  Bonus points

You can have:

* 1 extra point if you implement RTT estimation and adaptive RTO, as defined in TCP specification
* 1 extra point if you implement TCP Reno or TCP NewReno fast retransmission and recovery
* (**Very complicated. Proceed with caution**) 4 extra points if you replace TCP transport in your project-1 with the transport you have implemented in this project, which should involve:
  - creating a proper socket-like abstraction for your transport
  - bidirectional data flow (client will need to send request and server will need to send reply)
  - support for multiple parallel connections
