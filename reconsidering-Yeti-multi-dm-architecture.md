---
layout: post
title:  Reconsidering Yeti's Multi-DM Architecture after PINZ Experiment
author: Davey Song
categories: Yeti blog
---

# Reconsidering Yeti's Multi-DM Architecture after PINZ Experiment

## Introduction

The Internet's Domain Name System(DNS) is a vital component of Internet infrastructure mapping human-readable names into Internet Protocol (IP) addresses. DNS is designed and built on a single root, known as the Root Server System including root zone generation and root zone distribution. As part of IANA function, DNS Root zone management has a long history([SAC067](https://www.icann.org/en/system/files/files/sac-067-en.pdf)) and experienced technical changes (IPv6, DNSSEC, anycast etc) as well as policy changes (IANA transition, new gTLD). None of previous changes violates the assumption that a centralized entity takes charge in one particular function role in the Root zone management. Take [DNSSEC at root level](http://www.root-dnssec.org/wp-content/uploads/2010/06/draft-icann-dnssec-arch-v1dot4.pdf) for example, VeriSign is the only Root Zone Maintainer(RZM) who maintain the ZSK, signs the zone taken from ICANN and distribute the zone to 13 root servers. 

In order to introduce redundancy and avoid single point of failure, Yeti DNS testbed introduce Multi-DM model which uses 3 internal master servers doing DNSSEC signing (the VeriSign role) and root zone distribution. Each of them acts autonomously and equivalently to its siblings. They are referred to as Distribution Masters (DMs) in the context of root server system. 

Multi-DM practice in Yeti testbed received expected benefit due to its property of redundancy. However, root operation based on Multi-DM model exposed some issues as well. One issue is reported on the observation of delay of root zone distribution which is introduced in [section 5.2 of draft-song-yeti-testbed-experience](https://tools.ietf.org/html/draft-song-yeti-testbed-experience-10#section-5.2). Another issue is recently observed during PINZ experiment which will be introduced in this post.

In this post, the Yeti's Multi-DM Architecture is revisited with recently issues exposed during PINZ experiment. It is inferred that a fault-tolerant property is desired for future Yeti DM architecture. Most specifically, a  consensus should be achieve before any signed zone be published. It is proposed in conclusion that a Distributed Distribtion Master Architecuture (DDMA) is worth of doing as an experiment and important property in Yeti's future agenda.

## Brief report of PINZ and the problem statement
[PINZ experiment](http://yeti-dns.org/yeti/blog/2018/05/01/Experiment-plan-for-PINZ.html) was performed in the May 2018. The goal of PINZ is to minimize the difference between Yeti root zone and the zone retrieved from IANA. It is done by preserving IANA NSEC Chain and ZSK RRSIGs in Yeti root zone. It is firstly proposed as a experiment in Yeti mailing list and passed the test in lab environment. As expected, when it was performed in Yeti testbed, resolvers can validate the signed RR with signatures from IANA without any failure reports or so. 

As we did during KSK rollover in Yeti testbed, we deployed Atlas probes to observed the size of response for DNSKEY query. It was increased upto 1693 octets when IANA's ZSK was added into Yeti root zone overlapping with Yeti's ZSK rollover. Unsurprisingly, the failure rate for one probe under the large response increase to nearly 7% , then same as we reported previously in KSK rollover. In addition If KSK rollover is performed in the same time when PINZ is adopted, it will definitely produce larger response than we observed in KSK rollover. So adding more ZSKs into root zone is not a good direction which is one reason lead me to revisit the Multi-DM model. 

Another reason is that we experienced operational event during PINZ experiment due to the power failure in one of DMs which produced a zone without any signature!! It is not the flaw in design of PINZ but the inherit nature of Multi-DM model. In current Multi-DM model, each DM implements the loop of zone retrieval, DNSSEC signing and zone distribution independently. They only synchronize some meta data in advance. The approach adding more DMs in to the system do add more redundancy but also add possibility of failure points. By simple calculation, if the failure probability of one DM is *p*, then the failure probability of the whole system with *n* DM is *1-(1-p)^n* which is larger than *p* (if *p=0.1%* and *n=3*, then *1-(1-p)^n=0.2997%*). It is inferred that the redundancy property of Multi-DM is at the risk of the increase of failure probability, especially when *n* is large. So adding more DM into root system is not a good choice in practice.

In addition, the role of DM as root zone master server is a sort of special. From the Internet governance point of view, it is a authority on the root zone , the top-level name space of Internet! Adding another 2 DMs may be a little better than 1 but do not change the game with centralized role. People may still ask the same question why 3? How should take the role of DM? How we trust the 3 DM if we are not one of them? So it returns to the beginning. 

## Conception of Distributed Distribution Master Architecture (DDMA)

To answer and resolve the questions proposed above, we may need to think about a fault-tolerant property in design for Yeti DM Architecture. More concepts and technologies of modern fault-tolerant distributed system are worth of learning from. We may think about the DNSSEC of root zone in the background of classic BFT(Byzantine fault tolerance) context in which we can borrow tools from distributed ledge (So called Blockchain). 

Those thought and consideration will lead us to propose a Distributed Distribution Master Architecture (DDMA) for Yeti root testbed. The intuition is simple that a group of DM operators can work together to form a BFT-like system, in which each DM dependently generate the zone, signing the zone with their own key and publish to the BFT-like system just as we did in Yeti multi-DM. However, before the root zone published into DNS, a certain consensus is required every time among all DMs to confirm the To-Be-Published root zone for each SOA serial. 

The properties of this conception of DDMA are as follows:

* Most notably, the system is fault-tolerant both on operation failures or malicious attack given no trust is required for each node in BFT-like system as long as the majority of nodes agree on each transaction (generating a new root zone in Yeti DM case).
* The number of DM operator is not limited. We may require permission to join to simplify the scalability issue (similar with permission chain).
* Only one publish key (ZSK or KSK) is to be added to root zone if some kind of threshold signature scheme is used to support DNSSEC.
* Diversity of root zone data. DM can independently retrieved root zone from different sources, like ICANN, root servers, TLD authoritative server, or some real time DNSDB.
* Merit on Internet Governance. Root zone management (without registration) will be come a truly self-organized system with centralized entity. 

To achieve the goal, DDMA client should have functions listed below:

* Each DM client can generate root zone according to a specified and to-be-defined routine or guidance which guarantee the data consistency.
* Each DM client can sign the data and share the data with others which is basic building block to achieve consensus on the to-be-published root zone.
* A consensus algorithm should be run among each DM client so that majority of them can confirm a to-be-published root zone (may be added to a distributed ledge).
* Most vital functions of DDMA is a sort of group signing scheme to fit DNSSEC context. It is required to provide a same interface for DNSSEC validating resolver. [Threshold signature scheme](https://en.wikipedia.org/wiki/Threshold_cryptosystem) provides basis primitives for that purpose producing  three tuple <one Public key, {secret shares}, one signature>. In proposed DDMA, each DM client is expected to sign the zone with its secret share and broadcast to other clients. *m* of *n* signatures can be transformed to the to-be-publish signature for each resource record according to the basic primitive of Threshold signature scheme. 

More detailed design is needed such as initial trust setup, key generation, the choice of crypoto curve, KSK rollover in DDMA etc. But again it is worth of time and effort to introduce more distributed property into Yeti Root DM Architecture.
