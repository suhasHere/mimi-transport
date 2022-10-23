---
title: Cached and Async meSsage Transport (CAST)	 
abbrev: mimi-cast
docname: draft-jennnings-mimi-transport-latest
category: info

ipr: trust200902
area: App
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -  ins: C. Jennings
    name: Cullen Jennings
    organization: Cisco
    email: fluffy@iii.ca
 -  ins: S. Nandakumar
    name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com

--- abstract

This specification defines message transport based on publish/subscribe semantics 
for interoperable inter-server messaging that is rooted in modern, cloud friendly 
and scalable architectural underpinnings.

--- middle

# Motivation

When designing federation systems for message exchange, one common way to implement 
a messaging transport would follow architectural patterns like 
“Send and Forget Architecture”.

In this type of architecture, the requirement of the message transport is to 
attempt to send the message from the source domain to the target domain with 
no additional protections against losses in transit or causes that render messages 
not being delivered.

As depicted in the diagram below, the happy path flow results in the message 
being from alice@abc.com to be delivered to bob@xyz.com. 


~~~~~
              ┌────────────┐ (2)Post:hi bob  ┌────────────┐
              │            │                 │            │
          ┌──▶│  abc.com   │──┬────────────▶ │  xyz.com   │───┐
          │   │            │  │              │            │   │
          │     ───────────┘  │              └────────────┘   │
(1)Post:hi│bob       ▲        │                             (4│Post:hi bob
          │          │        │                               │
          │          └────────┘                               │
          │                                                   ▼
    .─────────.     (3)Delete:hi bob                    .─────────.
    ╱           ╲                                       ╱           ╲
  (alice@abc.com)                                     ( bob@xyz.com )
    `.         ,'                                       `.         ,'
      `───────'                                           `───────'
~~~~~
{: title="Send and Forget Pattern"}

One of the shortcomings of this architecture is the federation message transport 
doesn’t provide protections or options to recover from reasons that might result 
in messages being undelivered. As shown in the below depiction, when one/more 
resources at the target domain becomes unavailable (due to server(s) failure, 
server down due to maintenance), the message from alice@abc.com misses its delivery 
to bob@xyz.com.  Also to note, mechanisms to retry may be unsuccessful under these 
circumstances. In such cases, the onerous task is on the message sender to realize 
and retry/resend the messages. This is either done out of band with human intervention 
typically.

~~~~~
                                                   ▮      ▮
                                                   ▮▮   ▮▮
                 ┌────────────┐ (2)Post:hi bob  ┌───▮▮─▮▮────┐
                 │            │                 │     ▮▮     │
             ┌──▶│  abc.com   │──┬────────────▶ │  xy▮▮▮▮m   │
             │   │            │  │              │   ▮   ▮▮▮  │
             │     ───────────┘  │              └──▮──────▮▮─┘
   (1)Post:hi│bob       ▲        │               ▮▮        ▮▮
             │          │        │
             │          └────────┘
             │
        .─────────.     (3)Delete:hi bob                    .─────────.
       ╱           ╲                                       ╱           ╲
      (alice@abc.com)                                     ( bob@xyz.com )
       `.         ,'                                       `.         ,'
         `───────'                                           `───────'

~~~~~
{: title="Send and Forget Pattern, Failure Case"}

SMTP and XMPP based federation transports are typical examples where 
“Send and Forget” architecture is employed. 

This specification proposes a message interop transport suited for modern 
messaging and cloud friendly architectures which intends to address 
shortcoming with existing transports.


# Modern Messaging Federation Transport

Any considerations for building interop messaging protocol for modern messaging 
workloads needs to meet certain functional requirements as listed below:

* Leverage modern cloud native architectures

* Scale to large number of messages and consumers

* Be less onerous on the message senders to ensure the delivery

* Easy to build high reliability cloud design

* Easy to build horizontal scalability cloud design

* Better aligned with internal architecture of some existing solutions

* Easy to build gateways to existing APIs


CAST is publish/subscribe based interoperable messaging transport for 
inter-server message delivery. Such a transport can be used for delivering 
messages between servers within the same domain of operation or for cross 
domain message delivery.

An example set of CAST messages exchanged for delivering messages for the 
flow depicted are given below

~~~~~

                 (1)Add Subscription
                 abc.com/messages/* -> xyz.com

                            ▮ ▮ ▮
                            ▮     ▮
                            ▮     ▮       (0)Sub:abc.com/messages/*
                 ┌──────────▾─┐   ▮                       ┌────────────┐
                 │            │◀──▮───────────────────────│            │
           ┌────▶│  abc.com   │                           │  xyz.com   │─────┐
           │     │            │───▮──────────────────────▶│            │     │
           │     └───────────▲┘   ▮                           ─────────┘     │
           │                 ▮    ▮            (4)Publish                    │
           │                 ▮    ▮            Name:...                      │
           │                   ▮▮▮                                           │
           │                                                                 │
           │       (3)Store:
           │       Publish message                             (5)Publish
           │                                                   Name:abc.com/messages/alice/..
                                                               Msg:{body:"hi bob",
    (2)Publish                                                      to:bob@xyz.com}
    Name:abc.com/messages/alice/..
    Msg:{body:"hi bob",                                                      │
         to:bob@xyz.com}                                                     │
                                                                             ▼
           │                                                            .─────────.
      .─────────.                                                      ╱           ╲
     ╱           ╲                                                    ( bob@xyz.com )
    (alice@abc.com)                                                    `.         ,'
     `.         ,'                                                       `───────'
       `───────'
~~~~~
{: title="CAST Message Flow"}

In the example, the target domain, xyz.com, is interested in having messages 
from the domain abc.com to be delivered. Such a setup may be possible due to 
business relationships between the domains or any entity within the target domain 
expresses such an interest. 

1. The CAST endpoint in the domain xyz.com sends “Subscribe” message to CAST 
endpoint in the “abc.com” indicating its interest to receive any message that 
matches the name “abc.com/message/*”

2. On receiving the “Subscribe” message, the CAST entity at the domain “abc.com” 
creates an active subscription entry for “abc.com/message/*” against the domain 
“xyz.com”. Thus created subscriptions remain active until they expire or are canceled. 
Subscriptions can be renewed periodically to keep them active as well.

3. When Alice from abc.com publishes message to Bob at xyz.com, by sending 
“Publish” message, the CAST entity (within abc.com) performs the following 
steps on receiving the message:
  - Store the message against the name (“abc.com/messages/alice/id1”) at least for 24 hours.
  - Look up for any active subscriptions that matches the name
  - Forward the message to the CAST endpoint that matches the name from the previous step
  - In this example, since there exists an active subscription for the pattern 
    “abc.com/messages/*”, Alice’s message will be delivered to the CAST entity 
    at “xyz.com” based on the lookup result

4, On receiving the CAST Publish message, the CAST entity at the domain “xyz.com” 
will have sufficient information in the message to forward it to the right target 
within its domain, in this case, to Bob

Messages within CAST are cached for at least 24 hours by default, regardless of 
the status of message delivery. This allows, for example, “xyz.com” to  ask for 
the message again if the transaction to store it is unsuccessful.

In scenarios where subscriber CAST entity is unavailable at the time of the 
message delivery, the CAST entity resyncs its state by reissuing a “Subscribe” 
message, when it's back in operation and thus retrieve any cached messages as 
well as stream new messages that get published in the future.

To horizontally scale to a huge volume of messages, one or more servers 
can ask for messages and only one of them ends up getting a copy of each 
message. One way to achieve this would be to perform an equivalent of 
“x=y” modulo arithmetic on the name in the subscriptions. For example, 
on subscription, one can take the hash of the UUID and then compute 
module 7 and if that equals 5, that server is chosen to receive the 
messages matching that name, thus allowing the load to be distributed 
amongst the incoming servers with subscriptions.
       

# Security Considerations

Within the CAST architecture, the interacting domains are trusted to deliver 
each other messages for their users and are bound by business agreements that 
further constrain the rules related to use of messages exchanged, dealing with 
spam and any other policies that govern the successful federation. This is 
meant for major services to connect to other major services and not designed 
to deal with the issue of a random small domain with no business relationship 
to another domain connected to it.


--- back

# Acknowledgements

TODO
