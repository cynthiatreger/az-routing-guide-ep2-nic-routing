### [<< BACK TO THE MAIN MENU](https://github.com/cynthiatreger/az-routing-guide-intro)
##
# Episode #2: ~~VM Routing~~ NIC Routing & Azure routes

*Introduction note: This guide aims at providing a better understanding of the Azure routing mechanisms and how they translate from On-Prem networking. The focus will be on private routing in Hub & Spoke topologies. For clarity, network security and resiliency best practices as well as internet breakout considerations have been left out of this guide.*
##
[2.1. Network Interface Card (NIC) considerations](https://github.com/cynthiatreger/az-routing-guide-ep2-nic-routing/blob/main/README.md#21network-interface-card-nic)

&emsp;[2.1.1. VM vs physical device](https://github.com/cynthiatreger/az-routing-guide-ep2-nic-routing#211-vm-vs-physical-device)

&emsp;[2.1.2. VMs and NICs](https://github.com/cynthiatreger/az-routing-guide-ep2-nic-routing#212-vms-and-nics)

&emsp;[2.1.3. Concept representation](https://github.com/cynthiatreger/az-routing-guide-ep2-nic-routing#213-concept-representation)

[2.2. Azure routes etc](https://github.com/cynthiatreger/az-routing-guide-ep2-nic-routing/blob/main/README.md#22-azure-routes-etc)

&emsp;[2.2.1. *Effective routes*](https://github.com/cynthiatreger/az-routing-guide-ep2-nic-routing#221-effective-routes)

&emsp;[2.2.2. System routes](https://github.com/cynthiatreger/az-routing-guide-ep2-nic-routing#222-system-routes-default-routes--virutal-network-gateway-routes)

&emsp;[2.2.3. *Route table* and User Defined Routes (UDRs)](https://github.com/cynthiatreger/az-routing-guide-ep2-nic-routing#223-route-table-and-user-defined-routes-udrs)

&emsp;[2.2.4. Custom routes](https://github.com/cynthiatreger/az-routing-guide-ep2-nic-routing#224-custom-routes-are-either-udrs-or-routes-programmed-by-a-virtual-network-gw-or-route-server-ars)
##
In our Azure networking documentation you will find terms like *Route Table*, *Effective routes*, *UDRs* (for User Defined Routes) etc. that are often associated together and which can all sometimes be confusing.

The main purpose of this episode is to highlight that none of these concepts actually translates to the good old routing table used in traditional networking. 

It will also provide a description of the routes we have been looking at so far and the elementary routing components used in Azure Networking.

# 2.1.	Network Interface Card (NIC)
Before exploring the Azure routing terminology, let’s put some context around the idea of NIC in Azure vs On-Prem.

## 2.1.1. VMs vs physical devices
A standard On-Prem L3 networking device (switch/router/firewall) comes with the idea of at least 1 inbound + 1 outbound interface and transit routing capabilities. In Azure, VMs too can provide transit routing or have multiple Network Interface Cards (NICs). However most of the VMs receive and forward traffic out of the same NIC, like in the [Episode #1](https://github.com/cynthiatreger/az-routing-guide-ep1-vnet-peering-and-virtual-network-gateways) scenarios.

## 2.1.2. VMs and NICs
Part of creating an Azure VM is configuring its NIC(s). A NIC is connected to a subnet in a VNET and gets allocated an IP address from that subnet. 

:arrow_right: A VM can have one or multiple NICs. NICs can be in different subnets, but the VM and its NICs must all be in the same VNET.

Unlike a physical L3 device, if more than 1 NIC is attached to the VM, there still wouldn’t be a "per-VM routing table". 

## 2.1.3. Concept representation
For clarity the last environment of Episode #1 has been slightly adapted:
- Spoke1VM2 is removed from Spoke1/subnet2
- Spoke2 VNET, which had GW transit disabled, is not represented
- A 2nd NIC is attached to Spoke1VM and connected to Spoke1/subnet2 (with *GW route propagation* still disabled), while the exisitng NIC is connected to Spoke1/subnet1.

<img width="1028" alt="image" src="https://user-images.githubusercontent.com/110976272/215286468-3fd3c83d-3459-4df7-b1a2-007053ef8004.png">

:arrow_right: NIC1 contains the default *Effective routes* that come with the VNET peering of Spoke1 and the Virtual Network GW in the Hub.

:arrow_right: NIC2, because of the *Route table* associated to Spoke1/subnet2 configured to prevent the propagation of On-Prem route (*GW route route propagation* = disabled), only has the local VNET and peered VNET ranges in its *Effective routes*.

Because of this subtility, the term “*Effective routes* of a VM” used in this article and that you may often see in other documentation as well is not completely precise as it is a shortcut for “*Effective routes* of a NIC attached to a VM”.

# 2.2. Azure routes etc.

As explained by Jose in his article, [Azure NICs should actually be considered as mini routers](https://blog.cloudtrooper.net/2023/01/21/azure-networking-is-not-like-your-on-onprem-network/#:~:text=Azure%20NICs%20are%20actually%20routers).

In Azure, *Effective routes/System Routes/UDRs/Customs routes/Route Tables* **apply to NICs** (Network Interface Cards). 

:arrow_right: There is no overall routing table at the VM-level, that would aggregate the routing information received through its NICs.

### 2.2.1. *Effective routes*

The *Effective routes* of a NIC is the routing table of the NIC and lists the prefixes reachable from the NIC. There are the routes available on the subnet to which this VM NIC is connected.

:arrow_right: **The goal of the Effective routes is to determine where the traffic should be forwarded to, once sent out of the NIC, when reaching the wire of the Azure platform.**

There is **NO recursive routing** performed at the NIC-level: each prefix in the *Effective routes* has a Next-Hop that is directly reachable from the point of view of the Azure platform (for example an IP known through VNET peering or the IP of a Virtual Network Gateway).

The *Effective routes* include *System Routes* and *Custom Routes*.

### 2.2.2. System routes (=*Default* routes & *Virutal Network Gateway* routes)

The local VNET IP range and the routes created automatically by VNET peering are called [System routes](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview#system-routes) and tagged with Source = “*Default*” in the *Effective routes*.

System routes also include "[optional default routes](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview#:~:text=All-,Virtual%20network%20gateway,All,-Default)" like the routes received from On-Prem and programmed by a Virtual Network Gateway (Source = "*Virutal Network Gateway*").

### 2.2.3. *Route table* and User Defined Routes (UDRs)

A *Route table* is an Azure resource that contains static routes or UDRs ([User Defined Routes](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview#user-defined)) and that is associated to a subnet in a VNET to influence the *Effective Routes* of the NICs contained in that subnet (and subsequently the attached VMs).

Once associated to a subnet, a UDR will reconfigure (add and/or override) the *Effective routes* of the NICs connected to this subnet. UDRs appear with Source = "*User*" in the *Effective routes*.

:arrow_right: UDRs take precedence over *Default* routes of same prefix length (see [*How Azure selects a route*](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview#how-azure-selects-a-route) for more details.)

### 2.2.4. Custom routes are either UDRs or routes programmed by a Virtual Network GW or Route Server (ARS)

Finally, Custom routes refer to routes created either by UDRs (*User*) or received from On-Prem and programmed by a Virtual Network Gateway (*Virutal Network Gateway*) or programmed by a Route Server.
##
### [>> EPISODE #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals)
