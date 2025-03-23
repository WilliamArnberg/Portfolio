---
date: '2025-03-01T09:53:42+02:00' # date in which the content is created - defaults to "today"
title: 'Networking, Physics And Data-Quantization (WIP)'
draft: false # set to "true" if you want to hide the content 
summary: "Networking a physics simulation with custom client & server and compression techniques"
    
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

### Quaternion

##### Compression
The largest component of the state we are sending is the Quaternion.
It stores four four byte values representing x,y,z,w.

A Quaternion has a magnitude of 1 derived from $ x^2 + y^2 + z^2 + w^2 = 1 $.

If we drop 1 component we can derive it from the largest three through substitution.

Assuming w is the largest component.

$w = \sqrt{1-x^2 + y^2 + z^2}$
Notice how we are taking the square root of the components to calculate $w$?
This means that $w$ may never be negative, however a 3D rotation in quaternion space may contain a negative rotation.

$-x,-y,-z,-w$ in a quaternion represents the same rotation as $x,y,z,w$.
This means that if the largest component that we want to drop is negative we can negate the entire quaternion and still have a valid representation of the rotation.


![image](images/network/largestValueNegate.png)

With the largest value found, it is now possible to extract the smallest three components and begin quantizing them.

![image](images/network/smallestThreeQuantize.png)

For the quantization we need to take the floating point value and get it into a format that we can serialize to a uint32_t which consists of 4 bytes or 32 bits.

The data that have to be stored is the index of the largest component (2 bits), that leaves 9 bits for each remaining component.
9 bits gives us $2^9 = 511$.   
Therefor the integer range we can encode each component to is $[0,511]$.  
Another aspect of the quaternion is that since we are subsituting the largest value the remaining values will be within a range of $[-1/\sqrt{2}) , 1/\sqrt{2}]$.
Because the magnitude is 1 we can derive this from the fact that if the largest value a component can take is $v = 1$. The second largest is if two components have the same absolute value $v^2 + v^2 = 1$ or $2v^2 = 1$.  
There for with a little substitution we can see that $v = 1 / \sqrt{2}$.  
This is useful because we may now have a larger precision since we are not encoding in $[-1,1]$.

![image](images/network/quantizeDequantize.png)

All that is left of the Compression step now is to shift the values into a ```uint32_t``` by bitmasking and left shifting.

![image](images/network/shift.png)

##### Decompression

On the recieving end we will now decompress the ```uint32_t compressed.data``` back into the original value with a slight precision loss.

Starting with shifting out the data and dequantizing it, the data is now in a state which where we can begin to reconstruct the missing component.

![image](images/network/outShift.png)

All that is left now is to take the combined sum of the largest components and reconstruct the quaternion, assuming $Y$ was the largest component leaves $Y = \sqrt{1 - X^2 + Z^2 + W^2}$.

![image](images/network/reconstruct.png)



















