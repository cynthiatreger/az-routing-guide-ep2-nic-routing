# Episode #2: ~~VM Routing~~ NIC Routing

In our Azure networking documentation you will find terms like *Route Table*, *Effective routes*, *UDRs* (for User Defined Routes) etc. that are often associated together and which can all sometimes be confusing.

None of these concepts actually translates to the good old routing table I was used to in traditional networking. 

This episode is to understand better the routes we have been looking at so far and the elementary routing components used in Azure Networking.

[2.1. Network Interface Card (NIC)](xxx)

# 2.1.	Network Interface Card (NIC)
Before exploring the Azure routing terminology, let’s put some context around the idea of NIC in Azure vs On-Prem.

A standard On-Prem L3 networking device (switch/router/firewall) comes with the idea of at least 1 inbound + 1 outbound interface and transit routing capabilities. In Azure, VMs too can provide transit routing or have multiple Network Interface Cards (NICs). However most of the VMs receive and forward traffic out of the same NIC, like in the scenarios in [Episode #1](https://github.com/cynthiatreger/az-routing-guide-ep1-vnet-peering-and-virtual-network-gateways).

Part of creating an Azure VM is configuring its NIC(s). A NIC is connected to a subnet in a VNET and gets allocated an IP address from that subnet. 

Unlike a physical L3 device, if more than 1 NIC is attached to the VM, there still wouldn’t be a per-VM routing table. 

For clarity the last environment of Episode 1 has been slightly adapted:
- Spoke1VM2 is removed from Spoke1/subnet2
- Spoke2 VNET, which had GW transit disabled, is not represented
- A 2nd NIC is attached to Spoke1VM and connected to Spoke1/subnet2, while the exisitng NIC is connected to Spoke1/subnet1. 

<img width="1028" alt="image" src="https://user-images.githubusercontent.com/110976272/215286468-3fd3c83d-3459-4df7-b1a2-007053ef8004.png">

:arrow_right: NIC1 contains the default effective routes that come with the VNET peering of Spoke1 and the Virtual Network GW in the Hub.

:arrow_right:	NIC2, because of the UDR attached to Spoke1/subnet2 configured to prevent the propagation of On-Prem route, only has the local VNET and peered VNET ranges in its *Effective routes*.

Because of this subtility, the term “*Effective routes* of a VM” used in this article and that you may often see in other documentation as well is not completely precise as it is a shortcut for “*Effective routes* of a NIC attached to a VM”.

