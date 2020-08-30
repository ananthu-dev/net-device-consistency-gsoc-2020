# NetDevice Up/Down Consistency and Event Chain

This is a summary of what was done as part of Google Summer of Code 2020 during June 2020 to August 2020 with the organization [ns-3](https://www.nsnam.org/).

Student: [Ananthakrishnan S](mailto:ananthakrishnan190@gmail.com) 

Mentor: [Tommaso Pecorella](mailto:tpecorella@mac.com)


I would like to thank Dr. Mohit P. Tahiliani for motivating me to apply for the program, my mentor Tommaso Pecorella for the guidance throughout the program. I would also like to thank Tom Henderson,  Rediet and all the others who helped me increase the quality of my work.

# Introduction

Network device or netdevice is the representation of Network Interface Card (NIC). Any network transaction is made through a Network device, that is, a device that is able to exchange data with other hosts. Usually, a Network device is a hardware device, but it might also be a pure software device, like the loopback interface. A network device is in charge of sending and receiving data packets, driven by the network subsystem of the kernel, without knowing how individual transactions map to the actual packets being transmitted. Many network connections (especially those using TCP) are stream-oriented, but network devices are, usually, designed around the transmission and receipt of packets. It resides in L2 (Link layer) in the Linux network stack. NICs are of different types that enable communication with different types of networks. For example, if you want to connect your computer into a LAN, your computer needs an Ethernet card installed on it. Same goes for Wifi, LTE, etc.
 
Ns-3 offers a rich set of Network devices which helps simulate various types of network. Some of the netdevices in ns-3 include `WifiNetDevice` that enables simulation of wireless networks, `PointToPointNetDevice` that can be used to simulate point-to-point networks, `CsmaNetDevice` that can be used to simulate Ethernet topologes, `LteNetDevice` for LTE networks, etc.

## States of a Net device

Linux distinguishes between the administrative and operational state of an interface (netdevice). Administrative state reflects whether the administrator wants to use the device for traffic. A device can be in UP or DOWN administrative state. The administrative state can be change using commands such as ifconfig or ip. For example, `ip link set dev eno1 up` sets netdevice eno1 to UP state and `ip link set dev eno1 down` sets netdevice eno1 to DOWN state. When a device is plugged in, it is automatically configured for use.
 
However, an interface is not usable just because the admin enabled it - ethernet requires to be plugged into the switch and, depending on a site's networking policy and configuration, an 802.1X authentication to be performed before user data can be transferred. The operational state shows the ability of an interface to transmit this user data.

[RFC 2863: The Interfaces Group MIB](https://tools.ietf.org/html/rfc2863#) defines operational states for a netdevice. It is clearly described in linux kernel [documentation](https://www.kernel.org/doc/Documentation/networking/operstates.txt) as follows:

* **IF_OPER_UNKNOWN (0)**:
 Interface is in unknown state, neither driver nor userspace has set
 operational state. Interface must be considered for user data as
 setting operational state has not been implemented in every driver.
* **IF_OPER_NOTPRESENT (1)**:
 Unused in current kernel (notpresent interfaces normally disappear),
 just a numerical placeholder.
* **IF_OPER_DOWN (2)**:
 Interface is unable to transfer data on L1, f.e. ethernet is not
 plugged or interface is ADMIN down.
* **IF_OPER_LOWERLAYERDOWN (3)**:
 Interfaces stacked on an interface that is IF_OPER_DOWN show this
 state (f.e. VLAN).
* **IF_OPER_TESTING (4)**:
 Unused in current kernel.
* **IF_OPER_DORMANT (5)**:
 Interface is L1 up, but waiting for an external event, f.e. for a
 protocol to establish. (802.1X)
* **IF_OPER_UP (6)**:
 Interface is operational up and can be used.

IP layer, DHCP, FIB, etc listen to any state changes happening in a netdevice and takes appropriate measures upon detection of state change. For example, When the administrative device of a Net device is set to DOWN, IP needs to be disabled for that device and routes associated with the device needs to be updated so that those routes are no longer used.  

# Problem Statement

* The state changes are directly related to the presence or absence of a channel. In other words the current state represents operational states of a NetDevice. Administrative state is not available in ns-3.

* ns-3 has facilities (i.e functions and variables) to store the link state of the device as well as propagate the state change to higher layer protocols. But the usage of these facilities are not consistent among netdevices. 
  * `LteNetDevice` does not use them at all. Therefore state changes in LTE are neither recorded nor made known to higher layers.
  * `WifiNetDevice` uses these facilities. When STA gets connected or disconnected to a channel, state change is recorded and a callback is triggered to announce the change.
  * `CsmaNetDevice` state change happens only when the device is initially connected to a channel. Even though `CsmaNetDevice` have functions like `bool Detach (Ptr<CsmaNetDevice> device);` to detach and `bool Reattach (Ptr<CsmaNetDevice> device)` to reattach `CsmaNetDevice` from and to a `CsmaChannel`, it is not utilised properly and state changes are not triggered when `CsmaNetDevice` gets detached and reattached. As a result, any protocol watching out for state changes happening in `CsmaNetDevice` won't receive such state changes. 
  * In `PointToPointNetDevice`, the state is set to UP as soon as the device is connected to a channel irrespective of the presence of a device on the other end. In reality, link is live only when both ends are attached.

* IP interfaces do not listen to the state changes happening in the Net device. Therefore IP is unable to react to state changes.

# Objectives

This project aims to do the following:

* Bring consistency in the behavior of netdevices regarding state changes. We aim to examine three netdevices `PointToPointNetDevice`, `WifiNetDevice` and `CsmaNetDevice`. These three netdevices are the most used in simulations. Codebase of `LteNetDevice` is complex. Therefore it is left out for now.

* Examine higher layer protocols such as IPv4, IPv6, DHCP, etc and make sure that they are reacting properly to state changes.

# Project Overview 

The project is divided into three phases as follows:

* Phase 1: Define behavior of NetDevice API and correct `PointToPointNetDevice`.
* Phase 2: Correct `CsmaNetDevice` and `WifiNetDevice`. 
* Phase 3: Check and correct EventChains.

## Phase 1: Define behavior of NetDevice API and correct PointToPointNetDevice.

In this phase, Linux kernel code was examined to get a glimpse of the working of netdevice states and on the basis of that an architecture was created. All throughout the coding period, the architecture went through several changes as suggested by mentors. As a result, the final proposal is modeled after [RFC 2863: The Interfaces Group MIB](https://tools.ietf.org/html/rfc2863#). 

A useful document describing the working of device states and notifications in Linux prepared as part of this project can be found [here](https://docs.google.com/document/d/1KCZevur9F5bes84OBoN5e96kNfRyfCP_hRep722xpw4/edit?usp=sharing). 

First iteration of NeDeviceState architecture can be found [here](https://docs.google.com/document/d/1t7MN63itRNjUoCqjFLM3voVk2_Iw5UAWOisUsc33KVI/edit?usp=sharing).

Differences between how states change in Linux and the proposal (not final) can be found [here](https://docs.google.com/document/d/1nmN2WuRCbbeWnwY4vJxw_-DocAYk1ynIgbX3Kc85bV4/edit).

The final proposal of NetDeviceState architecture that is agreed upon by everyone can be found [here](https://docs.google.com/document/d/1RA8OgcW8euTNzZHAHcR6VQa21IT0GGrwCq71yUb-Y1w/edit?usp=sharing).


A new class `NetDeviceState` is designed that contains administrative and operational states of a Net device. It also has trace sources that allow higher layer protocols to listen in on state changes. A netdevice wanting to use this feature can aggregate the object of this class using ns-3’s object aggregation system. These features are enclosed in a new class because it is decided that not all netdevices would want to use this feature. There are two ways to aggregate `NetDeviceState` object to a netdevice:
 * This class can be aggregated directly.
 * If changing administrative states requires device specific actions then this class can be extended and device specific actions can be specified there. Then the child class can be      aggregated to the Net device. For this purpose two virtual functions are created; `virtual void DoSetUp (void)` and `virtual void DoSetDown (void)` can be defined if needed in child class.

Some of the operational states described in RFC 2863 is not relevant to ns-3. They are excluded from the implementation. They are:

* **IF_OPER_UNKNOWN**:  Used for devices where RFC 2863 operational states are not
  implemented in their device drivers in Linux kernel. In ns-3, devices
  that does not use RFC 2863 operational states do not aggregate 
  NetDeviceState object with them. This state is therefore not needed.
  
* **IF_OPER_NOTPRESENT**: Can be used to denote removed netdevices. Not used
  in linux kernel. Removed devices disappear and there is no need to 
  maintain a state for them.

* **IF_OPER_TESTING**: Unused in Linux kernel. Testing mode; not relevant in ns-3 either.

 A [merge request](https://gitlab.com/nsnam/ns-3-dev/-/merge_requests/370) was generated for this work. At the time of writing this report, it needs to be merged.

It was decided that instead of extending NetDeviceState class for PointToPointNetDevice, it would be better to extend it for CsmaNetDevice. This is because there are functions in CsmaNetDevice that can be used to attach, detach and reattach the device from the channel and it would be more suitable than PointToPointNetDevice to test the architecture. 

## Phase 2:  Correct CsmaNetDevice and WifiNetDevice.

### CSMA Module

Csma module was examined and following problems were identified:

* `CsmaNetDevice` can be attached, detached and reattached from a `CsmaChannel` using `int32_t Attach ()`, `bool Detach ()` and `bool Reattach ()` functions present in `CsmaChannel`. These functions doesnot modify the state of the device. This means even when the device is detached, state of the device stays UP. An [issue #234](https://gitlab.com/nsnam/ns-3-dev/-/issues/234) was created on [ns-3-dev](https://gitlab.com/nsnam/ns-3-dev) repository in Gitlab describing this issue.

* State of the device is not taken into consideration when received packet is processed.

The following changes were made in csma-module to properly change operational states:

* When CsmaNetDevice is attached or detached or reattached to a Channel, the operational state is changed accordingly now. Whenever the device is connected to a channel, Operational state is set to IF_OPER_UP and whenever it is disconnected, Operational state is set to IF_OPER_DOWN. Also, Operational state changes when administrative state is set DOWN. 

* CsmaNetDevice now checks administrative and operational state before sending a packet and before forwarding a received packet to higher layers.

* A new class `CsmaNetDeviceState` is created by inheriting `NetDeviceState` to take care of specific task related to changing administrative state of `CsmaNetDevice`. 
  * When administratively bringing UP the device, it needs to be checked whether the device is connected to a channel and is active. If it is, then Operational state can be set to IF_OPER_UP. Otherwise if the device is found to be disconnected, then there is no need to change the Operational state as it is already in IF_OPER_DOWN state.
  * In case of bringing DOWN the device, device queue needs to be flushed.

Other additions include:

* Test suite and example program are created to verify and demonstrate the working of states in CsmaNetDevice.

* A fix was provided by my mentor Tommaso Pecorella [@tommypec](https://gitlab.com/tommypec) to fix issue #234 and it is intergrated into the csma-module.

A [merge request](https://gitlab.com/nsnam/ns-3-dev/-/merge_requests/353) is generated for this work which is under review.

### Wifi Module

Out of all the netdevices that are selected to correct as part of this project, `WifiNetDevice` is the best when it comes to state changes. Without the `NetDeviceState` architecture, Wifi uses the existing facilities in ns-3 to change link state when connected/ disconnected from a channel. Changes made in wifi-module is as follows:

* A new class `WifiNetDeviceState` is created to take care of device specific tasks during state change. 

* Added functions in AP (`class ApWifiMac`), STA (`class StaWifiMac`), AdHoc (`class AdHocWifiMac`) MAC to turn on/off PHY and cancel/restart probing activities when WifiNetDevice changes its administrative state. 

* Before sending and receiving packets, administrative and operational states are checked and will go ahead only when the device is operational.

## Phase 3

In Phase 3, it was planned to examine higher layer protocols such as IPV4, IPv6, DHCP, ARP etc to see whether they are correctly reacting to state changes happening in the Net device. Unfortunately I was not able to do that. IT was decided to work on `PointToPointNetDevice` that was postponed in Phase 1. 

`PointToPointNetDevice` always stays in UP state. This device cannot be detached from the channel; It has no functions present to detach and reattach itself to a channel. Some problems need to be solved before extending `NetDeviceState` architecture to `PointToPointNetDevice`.

* Currently packet transmission is not affected by state change. When device state changed to DOWN for some reason while a packet is being transmitted, the transmission is not aborted. Instead, transmission continues without interruption and succeeds. This is because ns-3 simulates packet transmission by scheduling a `TransmitComplete` event after a calculated transmission delay and scheduling a `Receive` event in the receiver netdevice to be executed after a delay of Transmission delay + Propagation delay. In order to properly simulate a state change, we need to cancel both events and drop the packet.

* There are no functions to detach a `PointToPointNetDevice` from `PointToPointChanne`l. The main advantage of implementing `NetDeviceState` architecture is to enable simulating more complex scenarios involving disconnected devices. This is not possible if the device lacks functions to connect and disconnect from a channel. 

* Another issue is that `PointToPointNetDevice` should be UP only when it is connected and another device connected and is UP on the other end of the channel. So if one end is disconnected, state change should happen in both devices. Currently, the presence of a device on the other end is not taken into consideration.

At the time of writing this report, last two issues have been addressed and is under testing.

# What is next?

Unfortunately, not all goals as described in the project are accomplished. The following goals are remaining and will be completed in coming days:

* Completion of work remaining in PointToPoint module.
* Examination of higher layer protocols to verify that they are properly reacting to state changes.












