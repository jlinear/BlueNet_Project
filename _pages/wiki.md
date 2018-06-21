---
layout: page
title:  "Wiki"
---

<p align="center">
<img src="../assets/img/wiki_logo.png" alt="bluenet" height="75px"/>
</p>
<!-- ![](../assets/img/wiki_logo.png) -->


[Introducing BlueNet](#intro)

[Architecture](#archi)

[Stack Protocol](#stack)
    - [Overview](#overview)

[Android/Java App Development](#andr)

[Reference](#ref)


## Introducing BlueNet <a name="intro"></a>

BlueNet is the stack protocol designed and implemented on top of Bluetooth&reg Low Energy (BLE) stack. While BLE was originally designed as a single-hop networking solution, its widespread use can help fuel adoption of BLE as basis for ad-hoc established networks. However, BLE is not readily able to form such networks and, therefore, BlueNet will provide the mechanisms and protocol enhancements needed to turn a BLE-equipped smartphone into an ad-hoc network capable device.

**Developing Environment**

The current version of BlueNet is developed using java, targeting for Android applications (API level 23 and higher). 

**Core Technology**

BlueNet solely relies on the BLE technology to provide infrastructure-less communications. Connections are established based on BLE Generic Attribute Profiles (GATT), and data is transmitted in the form of GATT characteristics encapsulated by GATT services. Please see [Bluetooth core specifications][2] for more details.



## Architecture <a name="archi"></a>

The architecture of the BlueNet stack is shown as below.

<p align="center">
<img src="../assets/img/archi.png" alt="archi" height="400px"/>
</p>


## Stack Protocol <a name="stack"></a>

**0.0 Overview** <a name="overview"></a>

This section describes the protocol built on top of BLE to construct a mesh network. The mesh network is meant to operate in an opportunistic fashion with highly mobile nodes. Moreover, this addition to the network stack will be implemented in Android in application space.

**1.0 Functions of the Network Stack**
 - Device discovery
 - Link maintenance
 - Addressing
 - Send Message
 - Receive Message
 - Message routing

**1.1 Message Types and Advertisement Structure**

The advertisement packet is structured thusly:
```
=============================================================
| src_id | dest_id | msg_id | TTL | HP  | U/A | len | <msg> |
-------------------------------------------------------------
| 4 B    | 4 B     | 1 B    | 3b  | 1b  | 4b  | 1B  | 0-9 B |
=============================================================
```

`src_id` and `dest_id` are 'unique' 4 byte alphanumeric strings called BlueNet IDs. The same type of ID is used for groups.

`msg_id` is a single byte that is incremented by a sender for each message sent (including all control messages).

`TTL` is a 3 bit number that represents the number of hops that the message can still be forwarded.

`HP` is the high priority bit used to designate more important messages.

`U/A` is a nybble of unassigned bits

`len` is the length of the <msg> field

`<msg>` contains the relevant information for the given message type, if anything

The message type will be reflected in the GATT service UUID field. The types are as follows:

 - `0x186A - small_msg`: The entire message fits within the 20 byte packet boundary packet so no connection is necessary
    - <msg> is up to 9 bytes of an application message
 - `0x1869 - regular_msg`: The entire message (including header) is longer than 20 bytes. The payload will either be in the <msg> field or in a separate characteristic to be read later
    - <msg> is the message or empty 
 - `0x1868 - location_update`: The device with `src_id` is advertising its location as two adjacent floats
    - <msg> is two floats for the coordinates and two bytes for the group table checksum (0 if not available or empty): | latitude (4 bytes) | longitude (4 bytes) | chksum (2 bytes)
 - `0x186B - group_update`: A device is providing its group table
    - <msg> is the group table (<group_id> <group_type> <group props> <group_id> ...)
 - `0x186C - group_query`:  Query a device for its group table (response is group update)
    - <msg> is the timestamp

**1.2 Device Discovery**

All devices scan and advertise at the same time; i.e., perform both the role of Central and Peripheral at the same. It has been shown in [1] that long scan intervals and short advertisement intervals can increase the discovery probability (advertisements and scan intervals coincide) and therefore decrease the discovery latency.

**1.2a Device Discovery--Server**

A GATT server on the devices advertises the BlueNet service through BLE advertisements without any payload. The service contains a characteristic for each message type as well as a pull characteristic. The message type specific characteristics are used to notify other devices about a new message that is available to be read from the pull characteristic of the type indicated by the characteristic. After a client connects they will need to register with the server for these notifications.

- **1.2a.1 Data Structures**

    - Table of registered devices

**1.2b Device Discovery--Client**

A GATT client on the devices scans for advertisements for the BlueNet service and connects to the GATT server if not already connected. The client will register for notifications for each of the message type characteristics.

- **1.2a.1 Data Structures**

    - Table of device connections: used for managing the GATT connection with each server
    - Table of Bluetooth and BlueNet addresses: helps with tracking topology information and directing communication

**1.3 Link Maintenance**

Since GATT connections will disconnect when the connection becomes stale (no data transferred between two devices for 20 seconds on Android), location updates are sent frequently enough to prevent disconnections due to timeouts.

**1.4 Send Message**

Except for Small Messages, all message types have their header bytes and payload bytes place into separate characteristics. The header goes into the characteristic corresponding to that message's type, and the payload goes into a Pull characteristic. If a particular one-hop neighbor is indicated by upper layers of the network stack, then only that neighbor receives the notification for the changed (header) characteristic; otherwise, all neighbors are notified of the change. Once a receiving device decides it wants the message, it will read the data out of the Pull characteristic. If _t_ milliseconds have elapsed without any read attempts, then a new message can be sent (and can replace the contents of the Pull characteristic). 

With Small Messages, the entire payload fits within the 20 bytes allotted to a BLE packet (without renegotiating the MTU), thus the payload is sent along with the header in the notification.

**1.5 Receive Message**

- **1.5.1 Data Structures**
    - Buffer of recent messages: Tracks (src_id, msg_id) pairs that can be used to identify repeated messages. Only need to track a finite number of entries.
    
- **1.5.2 Small Message/Group Query/Group Feedback -- DEPRECATED?**
    - Read from advertisement (or characteristic) and enqueue message in send queue if node_id does not match the dest_id. Hand the message in the appropriate module, drop it, or forward it. Add (msg_src_id, msg_id) to the ring buffer.

- **1.5.3 Other Messages**
    - PUSH: the message is received and handled by the appropriate module, dropped, or forwarded.
    - PULL: the header is evaluated to see if it is relevant to the current node. If it is, then the payload is read and we proceed much like in PUSH.
    - Add (msg_src_id, msg_id) to the buffer.

**Location Dissemination**
- There are a number of different approaches to disseminating location information throughout the network beyond flooding. Here are some of the following options inspired by current literature:

- Location updates are sent or forwarded based on the error estimate where the error is driven by:
    - the elapsed time since the last location update
    - the historical changes in the node's location

- Location updates are sent or forwarded based on the distance that a node has traveled from its previous location. This can be combined with the error estimation.

- Location updates are sent or forwarded based on the velocity of the node.

- The above can be combined with hop or distance thresholds to decrease the frequency at which more distant nodes get updated about a particular node's location.
- The implementation that we will be following is:
    - Establish four zones (me, near, medium, far) based on radius from the current node each with their own difference threshold (D).
        - me: current node (alt. 0 hops)
        - near: 15 m (1 hop)
        - medium: 50 m (2 hops)
        - far: > 100 m (>3 hops)
    - A node tracks the last five locations for each node it has heard from (including itself).
    - The decision on whether to (re)broadcast is based on the root mean squared (RMS) distance of the last five location readings (as a way to capture the tendency of GPS measurements to wobble near a particular location if standing still and what a deviation from this should look like). If no recent locations are available then the difference between the current location and the last location are used (with an initialized location of 0,0 which could be problematic if we are operating in the Gulf of Guinea)
    - The following thresholds will be used for determining when to (re)broadcast, but require further evaluation:
        - zone 'me' (the current node): D > 1m
        - zone 'near': D > 3m
        - zone 'medium': D > 7m
        - zone 'far': D > 13m

- Node of interest propagation:
Given that the location update frequency is adjusted based on distance (or hop count) then the following addition ought to be considered:
    - When a node serves as a forwarder for a message sent from node A to B then the location update frequency (in the direction of A?) for node B is temporarily increased so that A can find out B's location more precisely.
    - The assumption is that interest in communication implies interest in location
    - Additionally, if a location update message is ever sent with a destination ID other than the broadcast ID (A addresses B, specifically) then location update rate from B will be temporarily increased (by decreasing thresholds).


**1.6 Message Dissemination / Routing**
- **1.6.1 Data Structures**

    - Buffer of recent messages: Tracks (src_id, msg_id) pairs that can be used to identify repeated messages. Only need to track a finite number of entries. (same as one from 1.5.1)
- **1.6.2 Description**
    - In an ad-hoc network, one of the go-to methods of routing is on-demand routing in which a route request is broadcast and a route is unicast back. This has the unfortunate side effect of taking longer to send a message, and this method is not immune to disconnections in the network. Further, location information is an assumed aspect of BlueNet (modifications to this discussed later). The core of our routing protocol is greedy geographic routing, but since we know this is not perfect either, we propose the following adaptations based on the literature:

    - Geographic flooding (k-path geographic routing):
        - the k best paths toward the destination receive the message for forwarding
        - assumes push-based sending
        - Preferred method of routing for low traffic network. Scaled from 8 (max number of connections) down to 1 based on network conditions.

    - When traffic volume increases we will switch to pull-based communication and can consider additional node properties in addition to location to help with forwarding decisions:
        - forwarding vector (traffic direction (x,y) and volume z)
        - direction of movement
        - below require additional control messages such as sharing neighbor/GATT connection tables:
        - (node centrality)
        - (number of node connections)
        - (node recent connections)
        
    - Geographic Routing Failure:
It is possible for geographic routing to fail. This could be caused by erroneous (or very stale) location updates, a complete lack of localization (GPS turned off?), or the destination node cannot be reached. In this case we can fall back to a couple of methods:
        - On demand routing
        - constrained flooding (in all directions but no repeated broadcasts and TTL is considered)
        - store, carry, and forward: the node that is unable to forward a message anymore caches it to pass along later until the buffer is full or a timeout has expired.


**1.7 Addressing**
- **1.7.1 Description**
    - All nodes and groups have a unique 4 byte id
    - A group is a set of nodes (possibly empty), identified by a unique id that is either a Named Group or Geographic Group
        - A *Named Group* is simply a labelling mechanism to which a device can subscribe (a could be part of group 'Blue' for instance)
        - A *Geographic Group* is a group that is associated with a circle (specified in meters) around a set of coordinates.
    - A node can belong to an number of groups
    - Groups can be addressed just like a node
        - forwarding to a group only differs in that a message is forwarded as long as TTL > 0 rather than ending when an appropriate recipient is found.
    - Groups are created by anyone
    - Lists of available groups can be shared and be queried for.
    - Geographic groups are entered and exited automatically (as long as the group is known to the node, of course).
    - There is always a special Broadcast or All group that contains all nodes running BlueNet (ID = 0000)
    
    - Named groups allow nodes in a network to be associated and addressed concisely no matter their location. This does, however, create an issue for a routing mechanism based on geography. To address this we make use of the group feedback message type. This message is used to indicate to another node that a group member is in a particular direction. This message can be sent at two points in the communication process: 
        - When a node enters a named group it sends a message to all of its neighbors indicating that it is a member of the given group.
        - When a node receives a message destined for the group to which the current node subscribes and the sending node has not sent anything to the group since this node joined or for T seconds.

    - The feedback message only contains the group ID (see part XXX) and is used internally to generate a 2D vector in the direction of the node sending the feedback of length 1. When multiple feedback messages are received for the same group then vector addition is used to aggregate all the directions of the group.

    - The group location can then be recorded as the point on a circle 2R from the current node that coincides with the group direction vector. If the magnitude of the vector is not beyond a certain threshold then the message will be sent in all directions; otherwise, the location attained from the vector is treated as if it were the location of a single node and routing proceeds like normal.

**2.0 Further Investigation**

 - How much can the scanning and advertising intervals be tuned in Android? What is optimal?
    - Developers cannot control over the exact millisecond frequency, but can experiment with different settings, such as (ADVERTISE_MODE_LOW_LATENCY, or ADVERTISE_MODE_LOW_POWER). Refer to [here][1]
 - How many GATT connections can be maintained simultaneously?
    - 7 or 8 max.
 - Can GATT connections be maintained bidirectionally?
    - the establishment process can only be initiated by a scanner, but the data flow and the operation of ending connection can be done form both side.
 - when a new payload is loaded and broadcast, the BLE mac address is actually changed (out of security consideration by BLE designer), so the only identifier for a specific device is the userid. So if a device plays as an mediator and changed its payload, the source device may take longer time to find this mediator and connect the first hop.

## Android/Java App Development <a name="andr"></a>

[TO BE FILLED SOON!]

## Reference <a name="ref"></a>


    [1]: Cho, Keuchul, Woojin Park, Moonki Hong, Gisu Park, Wooseong Cho, Jihoon Seo, and Kijun Han. 2014. “Analysis of Latency Performance of Bluetooth Low Energy (BLE) Networks.” Sensors (Basel, Switzerland) 15 (1): 59–78. https://doi.org/10.3390/s150100059.
    [2]: https://stackoverflow.com/questions/37151579/schemes-for-streaming-data-with-ble-gatt-characteristics
    [3]: https://devzone.nordicsemi.com/f/nordic-q-a/47/writing-more-than-20-bytes-at-a-time-to-a-characteristic?ReplyFilter=Answers&ReplySortBy=Answers&ReplySortOrder=Descending
    [4]: https://stackoverflow.com/questions/24135682/android-sending-data-20-bytes-by-ble
    [5]: https://stackoverflow.com/questions/38640908/android-ble-how-is-large-characteristic-value-read-in-chunks-using-an-offset
    [6]: https://stackoverflow.com/questions/31030718/handling-indications-instead-of-notifications-in-android-ble






  [1]: https://stackoverflow.com/questions/34346589/android-ble-advertising-interval-change
  [2]: https://www.bluetooth.com/specifications/bluetooth-core-specification