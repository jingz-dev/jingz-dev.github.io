---
layout: post
title:  "Explaining why your friend goes offline the moment you reach out, from a systems design perspective"
date:   2024-03-08 00:00:00 -0800
categories: systems design infrastructure interview
---
Have you ever noticed a friend go offline just as you message them<sup>[1](#1)</sup>? Let's explore this from a systems design perspective. 

The chance that they don't want to talk to you is non-zero, but I think a better explaination lies in how these online status systems are architected. Having run into these scenarios frequent enough, I like my interpretation because it also makes me feel better.

Here's a theorycraft of how the online status feature may be designed.

Consider a server architecture, where users A, B and C can communicate with each other through the server, and they are friends with each other. When A, B and C connect to the server, the server tells everyone who has A as their friend that A is online. We ignore the mechanisms of push notification scaling for this article, and assume our server has a client handle in its client session, that when the handle is invoked, the server can send a message to the client. Here's the diagram if the server were to actively notify A's status.
![image](/assets/images/status_consistency/consistent_status.jpg)  

Here comes a bunch of assumptions:  
If we have a chat application across the globe with 1 million daily active users, and an average person has 50 online friends<sup>[2](#2)</sup>. 
* A user goes online at least once per day. 
* For every person that comes online, the effort the server must take is to query people who has A as friends, and broadcast it to the 50 friends. We ignore the querying part here.
* We assume that user logins are spread out evenly through out the day. That's 1 million *users/day*  × 50 *push messages/user* ÷ (24 *h/day* × 3600 *s/hour*) = 579 *push messages/second*. 
* In reality, due to peak times, the number of push messages per second would be much higher.
Looking at the number, 579 messages per second, that's a lot of work to handle for just a online status change. If we want the user status to be updated when a user goes offline, then the client sends another message to the server, and this would effectively double the 579 *push messages/second* estimate. 

But what if someone loses their internet connection, or terminates the chat program abnormally? If the server does not receive an "I'm going offline" update from the client, should it indefinitely keep the user state as online? Probably not. 

One way to work around this is to record a timestamp of 'last seen on time', and the server has to periodically receive a heartbeat<sup>[3](#3)</sup> from the client. This again, would further increase that 579 *push messages/second* number, and the status is still not accurate. It depends on how often we check if the client is still alive. If we check once per 10 minutes, then worse case, we'll still see the client online for 10 minutes after the client lost their network connection. If we increase that check frequency, we can get better fidelity. If we want the online status check to be real-time, then we need to stream heartbeat messages. This is due to the server cannot reliably depend on the client's offline notification for accurate status. The only way to determine if a client is offline with certainty is by monitoring for regular keep-alive signals or heartbeats from the client. If the server fails to receive these periodic signals, it can infer that the client is offline.

Now given up on the idea of constant streaming between client and server just for online status, we still want the service to be able to proactively keep the online statuses up-to-date. Such system probably would look something like this:
![image](/assets/images/status_consistency/consistent_status-Page-2.jpg)  
<sub>This is a form of reconciliation<sup>[4](#4)</sup></sub>
<br/>
<br/>
So far, we've tried to solve the consistency problem and tried our best to match the user's online state with their server-side status, and found out that real-time status updates is not an easy problem, the more accurate we try to make it, the more complex the system will be (Tradeoffs).

If I were to pay for the infrastructure of such service, I wouldn't want the infrastructure of real-time status updates to be a lot. It adds complexity, and it's not a core feature (we haven't even touched the chat feature yet, right?). We must<sup>[5](#5)</sup> relax the requirements for such "real-time" status, and just say we want to know the online status of the person __if needed__. Then the service can be "lazy"<sup>[6](#6)</sup>, and not actively track user statuses unless it's convient to do so <sup>see [passive updates](#PU)</sup>.
How can we tell when would user A need to know the online status of user B? 
* User A needs to know the status of user B, at all times (A may not even care. Also, if B is at the bottom of A's list, B wouldn't even appear in A's list of friends view unless A scrolls down)
* User A needs to know the status of user B, when A is looking at the friends list (how do we know A is "looking" at the friends list? We could assume A is looking when the chat program is in focus)
* User A needs to know the status of user B, __when user A wants to talk to user B__ (Absolutely)
<br/>

I think you can tell where I'm trying to go. Client A may or may not have received the latest online status updates of B. When A opens up B's chat window, we can use this action to determine A wants to talk to B, and A needs to know B's most up-to-date online status. A's chat client will send the service something like a "refresh" command to fetch the most up-to-date online status of B. The service will lookup B's last known status, and if B's status is stale, can reach out to B to see if they're still online or not. 

What I've just described is a potential design where online state is evaluated lazily, and I used the action of A opening up B's chat window to determine the necessity of an online status refresh. 
![image](/assets/images/status_consistency/consistent_status-Page-3.jpg)  

There are other user signals we could use to tell A needs to know B's most up-to-date status, such as looking up B's profile, or mouse hovering over B's avatar, and so on. This lazy evaluation of online status is not exclusive with some passive or active mechanisms, such as, <a name="PU"></a>if B recently sent a message to someone else, we can conviently tell the chat message processing logic to update B's "last seen" time (passive update), or, have the B's client ping the server once every 10 minutes. Key point being, we can use lazy evaluation of a user's online status as a baseline, and use various techniques to improve the freshness of the online status, with the tradeoff of building more complex mechanisms.

So there we have an explanation. The reason your friend goes offline the moment you start typing, or right after you typed something, is more likely a slow systems reaction to the online status, instead of your friend trying to actively avoid you -- your friend went offline a while ago. I do want to note, if your friend was *really* trying to avoid you, they're actually making it hard for themselves too, since going offline would mean to avoid you, they'd also miss other people's messages.

----
### References
<a name="1"></a> [1] [Similar Experience, Quora, https://www.quora.com/Why-do-my-friends-go-offline-whenever-I-message-them-It-s-like-they-see-I-am-typing-and-they-go-offline-Do-they-think-I-am-annoying](https://www.quora.com/Why-do-my-friends-go-offline-whenever-I-message-them-It-s-like-they-see-I-am-typing-and-they-go-offline-Do-they-think-I-am-annoying)  
<a name="2"></a> [2] Assuming the friend relationship is symmetrical, where if A is friend of B, then B is also a friend of A.  
<a name="3"></a> [3] [Heartbeat Mechanism, Wikipedia, https://en.wikipedia.org/wiki/Heartbeat_(computing)](https://en.wikipedia.org/wiki/Heartbeat_(computing))  
<a name="4"></a> [4] [Reconciliation and Eventual Consistency, Wikipedia, https://en.wikipedia.org/wiki/Eventual_consistency#Conflict_resolution](https://en.wikipedia.org/wiki/Eventual_consistency#Conflict_resolution)  
<a name="5"></a> [5] Due to economical and practical reasons  
<a name="6"></a> [6] [Lazy Evaluation, Wikipedia, https://en.wikipedia.org/wiki/Lazy_evaluation](https://en.wikipedia.org/wiki/Lazy_evaluation)  
