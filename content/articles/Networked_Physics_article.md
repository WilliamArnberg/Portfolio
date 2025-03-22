---
date: '2025-03-01T09:53:42+02:00' # date in which the content is created - defaults to "today"
title: 'Networked Physics And Data-Quantization (WIP)'
draft: false # set to "true" if you want to hide the content 
summary: "Networked physics simulation with custom client & server"
    
---
Custom Client and Server architecture for physics simulation

{{< video src="/videos/net.mp4" autoplay="false" loop="true" width="800" height="450" >}}  


 # Messages, packets and buffers
To be able to send messages across networks we need to be able to serialize and deserialize data.
For that we need a base message class
 
 ![image](images/network/NetMessage.jpg)


 Each message consists of a message header that explicitly states what type the message is and the messageLength.
 The message class overloads the out and in stream operators so we can write data and read data from the message.

 We are going to be constructing multiple messages per update, and each frame we will send one state update so we need to be able to pack each individual message into a packet.
 
 ![image](images/network/packet.png)
  

Each packet gets filled with as many messages as we can write each frame.
The packet header tells us which sequence number the packet is part of.
Sequence number here doubles as the tick count.

The header also specifies a packet ack from the sender side, so that when a reciever recieves a packet it can see the most recent acked packet from the ack, and from that ack read the the ackField which is a bitfield consisiting of (ack - 32) last acks. e.g ack 100 means ackfield would consist of acks for packet 68 - 100.

This redundancy makes it so that we dont constantly have to worry about acks gettings lost.

When sending and recieving packages they get placed into sequence buffers 
The sequence buffer is a circular buffer giving us a fast and efficient way of storing data.

 ![image](images/network/sequence_buffer.png)


Each frame before sending a packet the sequence buffer constructs the package ackField by iterating over the N - 32 last packets and checking if they have been acked.

 ![image](images/network/generate_acks.png)

 If the packets have not been acked when constructing packages before sending, the sender will resend that package if the roundtrip-time have been crossed as that package is assumed to have been lost. 
 
# Inserting and Applying state

When state is recieved it is placed into a jitter buffer, and it will wait for 5 frames before applying the state to the simulation to avoid jitter.
The amount of time to wait would ideally be a calculation made with respect to latency.


# Data Quantization

As we are sending one state update per frame, the amount of data we are sending may quickly become a bottleneck as each update we would like to send 

The uncompressed data that are being sent each frame is roughly 64 bytes.

Depending on the games target bandwidth employing quantization techniques could make or break your game idea.

A typical networked game has a Target bandwidth of 50 KBps - 1MBps depending on the scale of the game.

![image](images/network/NonQuantizedState.png)

Assuming we would be sending a lot of state over the network this could quickly congest the network.

The largest component of the state we are sending is the Quaternion.
It stores four four byte values representing x,y,z,w.

A Quaternion has a magnitude of 1 derived from (x^2 + y^2 + z^2 + w^2) = 1.
If we drop 1 component we can derive it from the largest three through substitution.

w = sqrt(1-x^2 + y^2 + z^2)
Notice how we are taking the square root of the components to calculate w?




















