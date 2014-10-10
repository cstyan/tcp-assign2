tcp-assign2
===========

Repo. for COMP 7005 Assignment 2, which consists of TCP packet capture analysis and reading of Jacobson's paper.  

For now this repo. only contains a summary of Jacobson's paper to aid answering the questions listed at the start of the summary  

#Summary of Jacobson's paper.

##Introduction
This is a summary of Jacobson and Karels paper on the topic of congestion avoidance and control. The summary is meant as an aid to help better understand congestion avoidance control for someone who doesn't have the same level of knowledge as the original intended audience. The summary is meant to supplement, **not** replace, a thorough reading of the original paper. 

In order to do this I have shortened the descriptions in each section, in some cases the summary contains word for word copying from the original paper as well as paraphrasing, plus additional explanation and details from other sources.

In particular this summary was written for an assignment in the COMP 7005 at BCIT.  The assignment is an excercise in proper understanding of the TCP protocol and how improper use in a program will affect performance.

##Assignment Questions

Jacobson states 
>The obvious ways to implement a window-based transport protocol can result in exactly the wrong
>in response to network congestion.

What does he mean by this?

As a software developer provide a set of 3 guidelines that you would follow in implementing
an audio video streaming application. Use conclusions arrived at in the paper to justify your
guidelines.

---
##Summary
###Introduction
A TCP connection's flow should obey a 'conservation of packets' principle, congestion collapse should be exception rather than the rule.

Conservation of packets means that the connection should be in equilibrium (full window of data in transit) and no new packets are put into the network until an existing packet leaves.

If the system (TCP stack?) follows this then it is robust in terms of congestion control/avoidance.  Observation of the Internet suggests that it was not particularly robust.

---

###Potential problems
1. connection doesn't get to equilibrium
2. sender injects a new packet before there is space; no old packet has left
3. equilibrium can't be reached because of resource limits along the path

---

###Fixing these problems  
####1. Getting to equilibrium: slow start
Problem 1 has to be from a connection that is either starting or restarting after a packet loss. 
Another way to look at the conservation property is to say that the sender uses acks as a 'clock' that tells it when to put new packets into the network.

Since the receiver can't generate acks faster than data packets can get through network, the protocol is 'self clocking'. Self clocking automatically adjust to bandwidth and delay variations, and has a wide dynamic range.

The same thing that makes it stable is what makes it hard to start; to get data flowing there must be acks to clock out new packets but for those acks to be sent there has to be data flow.

**The slow start algorithm:**  
CWND starts at a very small size (1,2 or 10 segements for example) and each ack received by the sender causes an increase in the CWND by 1 for each ACK. This means that if the CWND starts as one the first ACK received will increase the CWND to two, causing two segments to be sent over the network and next the sender will receive two ACKs, so the CWND will be increased to 4.

This is known as the exponential growth phase, the CWND (window size) doubles for each successful round trip of the network.  This behaviour continues until the CWND reaches the size of the recievers advertised window (buffer size) or a loss occurs.

When a loss occurs the CWND is cut in half and saved as the Slow Start Threshold, the algorithm then starts again with CWND as 1(or 2 or 10, just the starting value).  Now when the CWND reaches SST, TCP goes into **congestion avoidance**, the linear growth phase of **slow start**.

With slow start the window size increases quickly enough to have a negligible effect on performance, even on links with a large bandwidth-delay product.

Slow start guarantees that a connection will source data at a rate at most 2x the max possbile on the path.  This happens when the previous CWND was the max possible transfer rate so the next CWND will be doubled, and then a loss occur causing a cut of the CWND size.

By contrast when 10mbps hosts talk over 56kbps Arpanet without slow start the initial transfer has a burst of 8 packets deliered at 200x the bandwidth of the 56k portion of the path
this burst puts the connection into a persistent failure mode of continuous retransmission.

*see figure 3 in paper for retransmission example  
see figure 4 in paper for slow start example*

Make sure to take note of the difference in the growth of packet sequence numbers  

---

###2. Conservation at equilibrium: round-trip timing
Once data is flowing reliably we need to address problems 2 and 3.

Assuming the protocols implementation is correct problem 2 represents a failure of the sender's retransmit timer. Round trip time estimation, the core of the retransmit timer, is the most important feature of a protocol implementation that expects to survive a heavy load.

One mistake is not estimation the variation of the round trip time.  Round trip time and the variation in round trip time increases quickly with load.  This causes unnecessary retransmission of packets that have only been delayed in transit and will eventually be delivered.

The roundtrip timeout is set to ßR, where ß accounts for the round trip time variation and R is the average round trip time estimate.  Previously ß was set to a fixed value, suggested as 2 by someone probably did a lot of math. 

Having a fixed value meant large variations in round trip time wouldn't be handled properly.  Using a fixed value meant that once the connection reached a certain load percentage, and the connection began to experience delay in transmission times, it responded by starting premature retransmission of packets that are only delayed and not lost.

Estimating ß eliminates these unnecessary retransmissions, and performance over the connection is improved in both low and high load situations, especially over high delay paths such as satelite links.

Another timer mistake is in the backoff after a retransmit.  It seems exponential backoff is the only method that works. Read the wikipedia page on exponential backoff for more info, but this is essentially means that the time between each retransmit of the same packet increases exponentially.  The first retransmit would be between 0-1, the second between 0-3, etc.

Linear system theory says that if a system is stable, the stability is exponential.  This suggests that an unstable system, a network subject to random to random load shocks and prone to congestive collapse, can be stabilized by adding some exponential damping to its primary excitation.  In other words, implemet exponential timer backoff on a networks traffic sources(hosts sending data).

He doesn't seem to adress solving problem 3 in this section.

---

###3. Adapting to the path: congestion avoidance
If the the timers are working properly, we can safely assume that a timeoout indicates a lost packet and not just a delayed packet.  This is where we can adress problem 3.  Packets get lost either because they were damaged or the network is congested and at some point on the path the packet was dropped.

On most network paths loss due to damage accounts for less than 1% of packet loss, so we can safely assume a lost packet was a result of congestion in the network.

Implementation of congestion avoidance consists of two components.  First, the network must be able to signal to the transport enpoints that congestion is ocurring or about to occur.  Second, the endpoints must have a policy that causes them to decrease their network utilization if this signal is received and increase their utilization if it is not received.

Assuming packet loss is always due to congestion and a timeout is always due to a lost packet, we have a good candidate for the network congestion signal.  It sounds like at the time this paper was written this signal was already in use by existing networks, and implementation of the endnode action was the only modification required for Jacobson's version of TCP.

A congested network will only stabilize if the traffic sources throttle back their window size at the same rate the receivers queues are growing.  Jacobson's endpoint policy results in a exponential decrease over time if the congestion persists.  The queue lengths are increasing exponentially as a result of the equation listed in this section; which accounts for average rate of new traffic, and the fraction of traffic leftover from the last time interval.

I don't entirely understand the theory and math behind this section, but the main points are clear:  
 - Congestion avoidance is achieved by both a signal to traffic sources of network congestion, or impending network congestion, and a policy for traffic sources to throttle their sending upon receiving this signal.
 - The implementation of these is done for us as part of the protocols (TCP for sure, Ethernet as well?) on network hardware(routres, switches, etc.) and the TCP stack within our own machines(the traffic sources).

It is important to note that although this and slow start seem similar because they are both triggered by a timeout and both manipulate the CWND, they are completely independant and solve seperate problems.  In practice both slow start and congestion avoidance are implemented, and can even be combined into one algoritm.

---

###4. Future work: the gateway side of congestion control
Note that while algorithms at the transport enpoints can ensure the newtork capacity isn't exceeded, they cannot ensure fair sharing of that capacity between traffic sources.  Only in gateways, at the convergence of flows, is there enough information to control sharing and fair allocation.

The goal here is to send send the congestion avoidance signal to endnodes as early as possible but not so early that the gateway becomes starved for traffic.  Again here we use packet loss as a trigger for the congestion signal, gateway 'self-protection' from a mis-behaving host should fall-out for free.  *This means the gateway simply deals with a host overusing the network capacity by dropping it's packets before sending them along the path.*  As a result this gateway algorithm should reduce congestion even if none of the endpoints are using congestion avoidance.  Endpoints that are using congestion avoidance will get both their fair share of bandwidth and a minimum number of packet drops.

Detecting congestion early is important since congestion grows exponentially.  If detected early, small adjustments to the senders' windows will cure it, otherwise large adjustments are needed in order to give the network enough spare capacity to deal with the backlog.

Because of the bursty nature of traffic, reliable detection of potential congestion is a non-trivial problem.  Jacobson planned to use round-trip time/queue length prediction as the basis of detection.
The initial results suggested that this approach worked well at high load, was immune to second-order effects in traffic, and was computationally cheap enough to not slow down kilopacket-per-second gateways.

There is no concrete description of the implementation, and based on the title of this section we can assume congestion control was implemented in TCP after the writing of this paper.

**End of Summary**

