# NetDevice Up/Down Consistency and Event Chain

This is a summary of what was done as part of Google Summer of Code 2020 during June 2020 to August 2020 with the organization ns-3. 

I would like to thank Dr. Mohit P. Tahiliani for motivating me to apply for the program, my mentor Tomasso Pecorella for the guidance throughout the program. I would also like to thank Tom Henderson,  Rediet and all the others who helped me increase the quality of my work.


# Introduction

Network device or Net device is the representation of Network Interface Card (NIC). Any network transaction is made through a Network device, that is, a device that is able to exchange data with other hosts. Usually, a Network device is a hardware device, but it might also be a pure software device, like the loopback interface. A network device is in charge of sending and receiving data packets, driven by the network subsystem of the kernel, without knowing how individual transactions map to the actual packets being transmitted. Many network connections (especially those using TCP) are stream-oriented, but network devices are, usually, designed around the transmission and receipt of packets. It resides in L2 (Link layer) in the Linux network stack. NICs are of different types that enable communication with different types of networks. For example, if you want to connect your computer into a LAN, your computer needs an Ethernet card installed on it. Same goes for Wifi, LTE, etc.
 
Ns-3 offers a rich set of Network devices which helps simulate various types of network. Prominent Net devices include WifiNetDevice that enables simulation of wireless networks, PointToPointNetDevice that can be used to simulate point-to-point networks, CsmaNetDevice that can be used to simulate Ethernet topologes, LteNetDevice for LTE networks, etc.

# States of a Net device

Linux distinguishes between the administrative and operational state of an interface. The administrative state is the result of "ip link set dev <dev> up or down" and reflects whether the administrator wants to use the device for traffic.

However, an interface is not usable just because the admin enabled it - ethernet requires to be plugged into the switch and, depending on a site's networking policy and configuration, an 802.1X authentication to be performed before user data can be transferred. The operational state shows the ability of an interface to transmit this user data.

IP layer, DHCP, FIB, etc listens in to the state of a net device and whenever state changes they take appropriate measures. For example, When the administrative device of a Net device is set to DOWN, IP needs to be disabled for that device and routes associated with the device needs to be updated so that those routes are no longer used.  

# Problem Statement

ns-3 has facilities (i.e functions and variables) to store the link state of the device as well as propagate the state change to upper layers and services. But the usage of these facilities are not consistent among Net devices.

LteNetDevice does not use them at all. Therefore state changes in LTE are neither recorded nor made known to upper layers. On the other hand, WifiNetDevice uses these facilities. When STA gets connected or disconnected to a channel state change is triggered. In CsmaNetDevice state change happens only when the device is connected to a channel. Even though we have functions to detach and reattach CsmaNetDevice from and to a channel, it is not utilised properly and state changes are not triggered when CsmaNetDevice gets detached and reattached. In PointToPointNetDevice the state is set to UP as soon as the device is connected to a channel. But in real life, link is live only when both ends are attached. This does not happen in PointToPointNetDevice.

The state changes are directly related to the presence or absence of a channel. In other words the current state represents operational states of a NetDevice. Administrative state is not supported in ns-3.

Moreover, IP interfaces do not listen to the state changes happening in the Net device. Therefore IP is unable to react to state changes. 

# Project Overview 

The project is divided into three phases as follows:

* Phase 1: Define behavior of NetDevice API and correct P2PNetDevice.
* Phase 2: Correct CsmaNetDevice and WifiNetDevice. 
* Phase 3: Check and correct EventChains. Bonus: LteNetDevice.


 

## Phase 1: Define behavior of NetDevice API and correct P2PNetDevice.

In this phase, Linux kernel code was examined to get a glimpse of the working of net device states and on the basis of that an architecture was created. All throughout the coding period, the architecture went through several changes as suggested by mentors. As a result, the final proposal is modeled after RFC 2863: The Interfaces Group MIB. 

We created a class NetDeviceState that contains administrative and operational states of a Net device. It also has trace sources so that higher layer protocols can listen in on state changes. A Net device wanting to use this feature can aggregate the object of this class using ns-3’’s object aggregation system. This class can be aggregated directly. If changing administrative states requires device specific actions then this class can be extended and device specific actions can be specified there. Then the child class can be aggregated to the Net device. 

It was decided that instead of extending NetDeviceState class for PointToPointNetDevice, it would be better to extend it for CsmaNetDevice. This is because CsmaNetDevice can be attached, detached and reattached from the channel and it would be more suitable than PointToPointNetDevice to test the architecture. 


## Phase 2:  Correct CsmaNetDevice and WifiNetDevice. 

CsmaNetDevice is the first net device that we tested the new architecture on. The following changes were made in csma-module to properly change operational states:

When CsmaNetDevice is attached or detached or reattached to a Channel, the operational state is changed accordingly now. Whenever the device is connected to a channel, Operational state is set to IF_OPER_UP and whenever it is disconnected, Operational state is set to IF_OPER_DOWN. Also, Operational state changes when administrative state is set DOWN. 

CsmaNetDevice now checks administrative and operational state before sending a packet and before forwarding a received packet to higher layers.

When administratively bringing UP the device, it needs to be checked whether the device is connected to a channel and is active. If it is, then Operational state can be set to IF_OPER_UP. Otherwise if the device is disconnected, then there is no need to change the Operational state as it is already in IF_OPER_DOWN state.

Test suite and example program is created to verify and demonstrate the working of states in CsmaNetDevice.

Out of all the Net devices that are selected to correct as part of this project, WifiNetDevice is the best when it comes to state changes. Without the NetDeviceState architecture, Wifi uses the existing facilities in ns-3 to change link state when connected/ disconnected from a channel. Changes made in wifi-module is as follows:

Added functions in AP, STA, AdHoc MAC to turn on/off PHY and cancel/restart probing activities when WifiNetDevice changes its administrative state. 

Before sending and receiving packets, administrative and operational states are checked and will go ahead only when the device is operational.

## Phase 3







