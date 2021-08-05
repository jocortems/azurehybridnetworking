# Inspecting Traffic across ExpressRoute Circuits in Azure

[In a previous article](../ExpressRoute-Transit-with-Azure-RouteServer/readme.md) we explored a series of patterns that facilitate communication across ExpressRoute circuits without the need for Global Reach, amongst other using Azure Route Server and Network Virtual Appliances. Those patterns are reliable and scalable when simple communication between circuits is all that is needed, however they fail to provide scalability when there is a need to secure the communication across circuits through a firewall appliance deployed in Azure; this is because the proposed patterns cannot guarantee flow symmetry across the NVAs if they are deployed in Active/Active configuration, forcing us to use an Active/Standby design which, depending on the volume of traffic, can quickly exhaust the [available VM flows](https://docs.microsoft.com/azure/virtual-network/virtual-machine-network-throughput#network-flow-limits).

This article explores 3 patterns that enable traffic inspection across ExpressRoute circuits while allowing the NVAs to be deployed in Active/Active configuration, providing scalability and reliability. These patterns make use of Azure Route Server to facilitate route exchange between Azure fabric and the NVAs, however User Defined Routes are still required because Azure Route Server doesn't currently support specifying a third party next hop through BGP, like the IP address of an Azure Internal Load Balancer configured with [HA ports](https://docs.microsoft.com/azure/load-balancer/load-balancer-ha-ports-overview) to guarantee flow symmetry. However, these patterns keep UDR management to a minimum by leveraging the BGP well known community "NO-ADVERTISE" (65535:65282) between the NVAs, so that only summary routes are advertised to Azure Route Server.

These patterns work best when private IP addressing is used across Azure and on premises, and there are only two ExpressRoute branches that need to communicate, but they can be extended to support public IP addressing and multiple ExpressRoute branches by carefully planning and configuring route summarization.

The scalability of the NVAs can be further enhanced by leveraging virtual machine scale sets with an autoscale policy based on Inbound flows, Outbound flows, CPU utilization and network throughput along with Azure Automation to bootstrap the VMSS instances configuration, create and remove the new BGP peerings in Azure Route Server as the VMSS scales in/out. My colleague and friend, Jose Moreno, has written an excellent blog post on how to do this which you can read [here](https://blog.cloudtrooper.net/2021/05/31/azure-route-server-and-nvas-running-on-scale-sets/).

> [!NOTE]
> I tested these patterns using open source [BIRD](https://bird.network.cz/) on blanket Ubuntu 18.04 virtual machines from Azure gallery for the router NVAs. I used another excellent blog post from Jose Moreno as a reference for how to configure BIRD which you can read [here](https://blog.cloudtrooper.net/2021/03/08/connecting-your-nvas-to-expressroute-with-azure-route-server/).
I used open source [OPNSENSE](https://opnsense.org/) for the firewall NVAs. My colleague and friend Daniel Mauser has put together an ARM template to automate the deployment of OPNSENSE to Azure which you can find [here](https://github.com/dmauser/opnazure)

> [!IMPORTANT]
> Although these patterns have been personally validated and tested there are no guarantees Microsoft will support them.

## 1. Single DMZ

In this pattern firewall NVAs are only deployed in one of the regions. The other region is considered as untrusted, so only stateless router NVAs are required there:

![Single DMZ Reference Architecture][diagram1]

[diagram1]: ../Images/Inspect-Traffic-Between-ExpressRoute-Circuits/ER2ER_SingleDMZ_NVA.png

This pattern requires the least configuration overhead to implement and a single set of firewall NVAs. The router NVAs in the untrusted region can be implemented using open source software on blanket Linux machines like [BIRD](https://bird.network.cz/) or [Quagga](https://www.quagga.net/), making it the most cost-effective pattern; however because there is a single set of firewalls, all workloads in Azure that require inspection through the firewall must be deployed in either the same VNet where the firewall is deployed or a directly peered VNet which, depending on the location of the ExpressRoute branches, might add excessive latency.

Following are some observations about this design.

### Single DMZ - BGP Configuration Requirements

The following points describe some important highlights of the BGP design:

- eBGP sessions are established between router and firewall NVAs for better routing domain separation
- eBGP sessions are established between the NVAs and Azure Route Server in their local VNet, since the Azure Route Server does not support iBGP adjacencies
- Azure Route Server is configured to [allow B2B transit](https://docs.microsoft.com/azure/route-server/quickstart-configure-route-server-cli#configure-route-exchange), so that it establishes iBGP adjacencies to the ExpressRoute Virtual Network Gateway instances
- To speed up convergence in the case of failure, it is recommended to configure BGP keepalive and hold timers in the NVAs to a lower value than the default 30/180 seconds, for example 20/60 seconds. Azure Route Server will honor these timers when the BGP sessions are negotiated. Although it is entirely possible to configure timers smaller than 20/60 seconds, doing so might cause BGP instability and route flaps
- NVAs tag routes received from the peer NVAs with BGP community "NO-ADVERTISE" (65535:65282), which instructs the NVAs to install those received routes (for the on premises networks and remote VNets) in their routing tables, but not to propagate them to Azure Route Server. As a consequence, on premises and the Azure fabric will not receive these specific routes for the remote branch and VNets; this facilitates the use of summary routes in the UDRs reducing management overhead and providing scalability as new network segments are onboarded in Azure and on premises (otherwise the more specific prefixes would win)
- NVAs advertise only summary routes to the Azure Route Server, in the example above this is RFC1918 supernets 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16.
- Additionally, if Internet breakout for on premises is desired through the firewall deployed in Azure the NVAs must advertise a default route to Azure Route Server. This enables the branches to send private traffic to Azure for which there isn't a more specific route on premises. Note that due to the nature of route summarization, there is an inherent risk of traffic black holing, hence careful consideration must be given to the summary routes advertised from the NVAs. If the NVAs don't have more specific routes for the remote branch or VNETs traffic will be black-holed at the local NVA, sparing VNet peering and ExpressRoute traffic charges

### Single DMZ - UDR Configuration Requirements

The main goal of UDRs is to overwrite the routes injected by the Azure Route Server in the Azure fabric (pointing to the individual IP addresses of the NVAs), and replace them with routes with the next hop of the respective Load Balancer:

- A route table per subscription per region is used for all subnets in the spoke VNets, DMZ-Spoke1 and DMZ-Spoke2 in the above example, deployed in the same region and same subscription connected to DMZ-VNET. Gateway route propagation is disabled and two routes are configured, the default route (0.0.0.0/0) and a route for DMZ-VNET address space (10.0.0.0/16), pointing to the IP address of FW-ILB
- A single route table is used for all subnets in DMZ-VNET other than GatewaySubnet, RouteServerSubnet and the subnets hosting the firewall NVAs. Gateway route propagation is disabled (which affects to routes coming from both the Virtual Network Gateways and the Route Server), and routes for the following networks are configured pointing to the IP address of FW-ILB:
  - Default route (0.0.0.0/0), to overwrite the system route
  - Local VNet (10.0.0.0/16), to overwrite the system route
  - All directly peered VNets (10.1.0.0/16, 10.2.0.0/16, 172.16.0.0/16), to overwrite the system routes
  - Summary networks being advertised from the peer router NVAs to Azure Route Server in UNTRUSTED-VNET. In the example above this is RFC1918 supernets 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16. In principle this route would not be required, since the default 0.0.0.0/0 would catch this traffic, but it is recommended for maintainability
- Another route table is configured for the GatewaySubnet. Gateway route propagation is enabled (otherwise that would break correct forwarding) and the following routes are configured:
  - Summary networks being advertised from the peer router NVAs to Azure Route Server in UNTRUSTED-VNET pointing to the IP address of FW-ILB. In the example above this is RFC1918 supernets 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16
  - Local VNET (10.0.0.0/16) pointing to the IP address of FW-ILB, to overwrite the system route
  - All directly peered VNETs (10.1.0.0/16, 10.2.0.0/16, 172.16.0.0/16) pointing to the IP address of FW-ILB, to overwrite the system routes
  - GatewaySubnet (10.0.2.64/27) and RouteServerSubnet (10.0.2.32/27) with a next-hop type of "Virtual network". These routes are not required but strongly recommended to prevent BGP traffic among ExpressRoute Gateway, VPN Gateway (if deployed) and Azure Route Server to be steered through the firewall, which can break route exchange and disrupt traffic flows

> [!NOTE]
> Azure doesn't allow to associate a route table with a default route to the GatewaySubnet; however it is not needed for this scenario because the firewall NVAs will be advertising a default route to Azure Route Server via BPG and traffic flow symmetry will be maintained by virtue of SNAT for Internet bound traffic. In addition to this, disabling Gateway route propagation for a UDR associated with the GatewaySubnet will break traffic flows to on premises.

- A route table is configured for the firewall's trusted/internal subnet. Gateway route propagation is enabled and routes for the summary networks being advertised from the router NVA to Azure Route Server pointing to the IP address of ROUTER-ILB are configured. In the example above this is RFC1918 supernets 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16. This route table is needed to prevent traffic loops because Azure fabric will have routes to these summary networks pointing back to the firewall's trusted/internal IP address by virtue of BGP route exchange with Azure Route Server
- A route table is configured for the firewall's untrusted/external subnet. Gateway route propagation is disabled and no routes are configured for this route table. This route table is not needed if the firewall NVA is not advertising a default route to Azure Route Server; it is used to prevent traffic loops because Azure fabric will have a default route pointing back to the firewall's trusted/internal IP address by virtue of BGP route exchange with Azure Route Server
- A route table is configured for the router's subnet in UNTRUSTED-VNET. Gateway route propagation is enabled and routes for the following networks are configured:
  - Default route (0.0.0.0/0) pointing to the IP address of FW-ILB
  - Summary networks being advertised from the firewall NVA to Azure Route Server in DMZ-VNET pointing to the IP address of FW-ILB. In the example above this is RFC1918 supernets 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16
  - DMZ-VNET (10.0.0.0/16) pointing to the IP address of FW-ILB
  - The IP addresses of DMZ-VNET firewall's NICs (/32 routes) in the trusted/internal subnet pointing to the IP addresses themselves. This is to prevent BGP traffic to be sent through FW-ILB, otherwise BGP sessions might fail to establish
- There is no need to configure a route table for GatewaySubnet in UNTRUSTED-VNET. Traffic will be distributed between both router NVAs by virtue of ECMP and asymmetric routing is expected; however because routers do not keep flow state like firewalls do traffic forwarding won't be impacted

### Single DMZ - Load Balancer Configuration Requirements

The main goal of Azure Load Balancers are deployed to provide traffic symmetry, as opposed to letting routing and ECMP decide which firewall NVA will handle the traffic:

- Two internal load balancers are required to provide redundancy for the NVAs and avoid traffic black holing. One for the firewall NVAs and another for the router NVAs
- The load balancers are configured with High Availability ports to guarantee flow symmetry and allow all protocol's traffic
- The load balancer in front of the router NVAs is only used as next hop of an UDR in the firewall NVA subnet. UDRs are required because otherwise the routes injected by the Azure Route Server in that subnet would cause a routing loop, and by using the load balancer as next hop the setup provides resiliency against router NVA failures
- It is recommended to configure a TCP probe for BGP port (TCP 179), this is to ensure that traffic will only be sent to the NVA if the BGP process is running and avoid traffic black holing in case the NVA is running but BGP is not

### Single DMZ - Limitations

The main limitation of this design is that traffic among Untrusted Branch, UNTRUSTED-VNET and untrusted spokes cannot be steered to the firewall NVA for inspection. If it is required to inspect all traffic between on premises and workloads deployed in Azure then those workloads must be deployed in spoke VNETs connected to DMZ-VNET or in DMZ-VNET itself

### Single DMZ - Extensibility

- If multiple ExpressRoute branches need to securely communicate among them through a firewall deployed in Azure, this pattern can be extended to a hub and spoke architecture. DMZ-VNET will be the hub VNet and a new VNet for each additional ExpressRoute branch with its own set of router NVAs can be peered to DMZ-VNET. This requires extensive IP addressing planning to facilitate route summarization and imposes additional UDR management overhead to guarantee traffic is forwarded to the appropriate set of router NVAs and prevent routing loops and traffic blackholing
- If the current IP addressing scheme makes it impossible to use route summarization, the specific prefixes can be advertised to Azure Route Server by not adding BGP community "NO-ADVERTISE" to the prefixes received from the peer NVAs. This imposes additional UDR management overhead as it requires one route per prefix to be configured in the route tables
- If required, a dedicated firewall NVA can be used for Internet breakout. The firewall dedicated for Internet breakout advertises the default route via BGP to Azure Route Server and the firewall for east-west traffic advertises the summary networks. The default route in the route tables points to the ILB fronting the Internet firewall while other routes are configured as described above

## 2. Dual DMZ

In this design firewall NVAs are deployed in both regions:

![Dual DMZ Reference Architecture][diagram2]

[diagram2]: ../Images/Inspect-Traffic-Between-ExpressRoute-Circuits/ER2ER_DualDMZ_NVA_only.png

By replacing the routers with firewalls it is now possible to inspect all traffic between on premises and Azure regardless of where the workloads in Azure are deployed. This however creates problems with asymmetric routing if traffic is sent directly to the firewalls rather than through the ILB fronting them, increasing the number of route tables and user defined routes required to configure and maintain and overall TCO.

Following are the requirements to implement this pattern.

### Dual DMZ - BGP Configuration Requirements

The BGP configuration requirements remain the same as for [Single DMZ pattern](#Single-DMZ---BGP-Configuration-Requirements)

### Dual DMZ - UDR Configuration Requirements

In addition to the UDR requirements for [Single DMZ pattern](#Single-DMZ---UDR-Configuration-Requirements), the following route tables and routes are required for this pattern:

- A route table per subscription per region is used for all spoke VNETs' subnets, Untrusted-Spoke1 and Untrusted-Spoke2 in the above example, deployed in the same region and same subscription connected to UNTRUSTED-VNET. Gateway route propagation is disabled and two routes are configured, the default route (0.0.0.0/0) and a route for UNTRUSTED-VNET address space (172.16.0.0/16), pointing to the IP address of UNTRUSTED-ILB
- A single route table is used for all subnets in UNTRUSTED-VNET other than GatewaySubnet, RouteServerSubnet and the subnet hosting the firewall NVAs. Gateway route propagation is disabled and routes for the following networks are configured pointing to the IP address of UNTRUSTED-ILB:
  - Default route (0.0.0.0/0), to overwrite the system route
  - Local VNET (172.16.0.0/16), to overwrite the system route
  - All directly peered VNETs (172.17.0/16, 172.18.0.0/16, 10.0.0.0/16), to overwrite the system routes
  - Summary networks being advertised from the peer firewall NVAs to Azure Route Server in DMZ-VNET. In the example above this is RFC1918 supernets 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16
- A route table is configured for the GatewaySubnet. Gateway route propagation is enabled (otherwise traffic to on-premises is broken) and the following routes are configured:
  - Summary networks being advertised from the peer firewall NVAs to Azure Route Server in DMZ-VNET pointing to the IP address of UNTRUSTED-ILB. In the example above this is RFC1918 supernets 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16  
  - Local VNET (172.16.0.0/16) pointing to the IP address of UNTRUSTED-ILB, to overwrite the system route
  - All directly peered VNETs (10.1.0.0/16, 10.2.0.0/16, 172.16.0.0/16) pointing to the IP address of UNTRUSTED-ILB, to overwrite the system route
  - GatewaySubnet (172.16.138.64/27) and RouteServerSubnet (172.16.138.0/27) with a next-hop type of "Virtual network". These routes are not required but strongly recommended to prevent BGP traffic among ExpressRoute Gateway, VPN Gateway (if deployed) and Azure Route Server to be steered through the firewall, which can break route exchange and disrupt traffic flows
  - If it is required to provide Internet breakout for the branches through a single set of firewalls in Azure, routes to 0.0.0.0/1 and 128.0.0.0/1 pointing to the IP address of UNTRUSTED-ILB are required

> [!IMPORTANT]
> Azure doesn't allow to associate a route table configured with a default route to the GatewaySubnet. It is possible to partition the default route in two routes as described above; however doing this will break management plane for the Gateways because, being a managed PaaS service, they have dependencies on Microsoft public services that are not covered by service tags. This is unsupported by Microsoft and should not be used in production environments. A better approach is to provide Internet breakout on each firewall or have a dedicated set of firewalls per VNET to provide Internet breakout. Another option, if a single set of firewalls is required to provide Internet breakout, is to deploy a set of router NVAs to advertise a default route to Azure Route Server in UNTRUSTED-VNET and configure a route table with the default route pointing to the IP address of UNTRUSTED-ILB associated to the router NVAs subnet. This is similar to pattern [Single DMZ with Azure Firewall](#3.-Single-DMZ-with-Azure-Firewall).

- A route table is configured for the firewall's subnet in UNTRUSTED-VNET. Gateway route propagation is enabled and the following routes are configured:
  - Summary networks being advertised from the peer firewall NVAs to Azure Route Server in DMZ-VNET pointing to the IP address of DMZ-ILB. In the example above this is RFC1918 supernets 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16. These routes are needed to prevent traffic loops because Azure fabric will have routes to these summary networks pointing back to the firewall's IP address by virtue of BGP route exchange with Azure Route Server
  - Default route (0.0.0.0/0) pointing to the IP address of DMZ-ILB, to overwrite the system route
  - DMZ-VNET (10.0.0.0/16) pointing to the IP address of DMZ-ILB, to overwrite the system route
  - The IP addresses of DMZ-VNET firewall's NICs (/32 routes) in the trusted/internal subnet pointing to the IP addresses themselves, to prevent BGP traffic to be sent through DMZ-ILB, otherwise BGP sessions might fail to establish
- A route table is configured for DMZ-VNET firewall's trusted/internal subnet. Gateway route propagation is enabled and the following routes are configured:
  - Summary networks being advertised from the peer firewall NVAs to Azure Route Server in UNTRUSTED-VNET pointing to the IP address of UNTRUSTED-ILB. In the example above this is RFC1918 supernets 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16. These routes are needed to prevent traffic loops because Azure fabric will have routes to these summary networks pointing back to the firewall's IP address by virtue of BGP route exchange with Azure Route Server
  - UNTRUSTED-VNET (172.16.0.0/16) pointing to the IP address of UNTRUSTED-ILB
  - The IP addresses of UNTRUSTED-VNET firewall's NICs (/32 routes) pointing to the IP addresses themselves. This is to prevent BGP traffic to be sent through DMZ-ILB, otherwise BGP sessions won't establish

### Dual DMZ - Load Balancer Configuration Requirements

The load balancer configuration requirements remain the same as for Single DMZ pattern described above

### Dual DMZ - Limitations

The main limitation of the Dual DMZ design is that Internet breakout for on premises must be provided by the firewall deployed in the local VNET

### Extensibility

- If multiple ExpressRoute branches need to securely communicate among them through firewalls deployed in Azure this pattern can be extended to a hub and spoke, partial mesh or full mesh architecture. A new VNET for each additional ExpressRoute branch with its own set of firewall NVAs can be peered to any VNET connecting a branch. This requires extensive IP addressing planning to facilitate route summarization and imposes additional UDR management overhead to guarantee traffic is forwarded to the appropriate set of NVAs and prevent routing loops and traffic blackholing
- If the current IP addressing scheme makes it impossible to use route summarization, the specific prefixes can be advertised to Azure Route Server by not adding BGP community "NO_ADVERTISE" to the prefixes received from the peer NVAs. This imposes additional UDR management overhead as it requires one route per prefix to be configured in the route tables

## 3. Single DMZ with Azure Firewall

Another variation of the single DMZ design is where the firewall NVAs in the first region are replaced by an Azure Firewall:

![Single DMZ with Azure Firewall Reference Architecture][diagram3]

[diagram3]: ../Images/Inspect-Traffic-Between-ExpressRoute-Circuits/ER2ER_SingleDMZ_AzFW.png

> [!NOTE]
> [Azure virtual WAN](https://docs.microsoft.com/azure/virtual-wan/virtual-wan-about) is offering a managed preview that provides traffic inspection between ExpressRoute branches using Azure Firewall with [secured virtual hub](https://docs.microsoft.com/azure/firewall-manager/secured-virtual-hub). If you are planning on using Azure Firewall to inspect your traffic the recommendation is to use Azure virtual WAN. Contact your Microsoft representative to get additional information about this managed preview

This pattern is very similar to pattern [Single DMZ](#1.-Single-DMZ), however because Azure Firewall doesn't integrate with Azure Route Server and doesn't support BGP, an additional set of router NVAs is required in the VNET where the Azure Firewall is deployed. These routers facilitate route injection into ExpressRoute, and route tables are used to send the traffic to Azure Firewall rather than through the routers. Furthermore, if Internet breakout for on premises through Azure Firewall is not required, neither application nor user data traffic will flow through these routers, making them a purely control plane component.

Following are the requirements to implement this pattern.

### Single DMZ with Azure Firewall - BGP Configuration Requirements

The BGP configuration requirements remain the same as for pattern [Single DMZ](#Single-DMZ---BGP-Configuration-Requirements)

### Single DMZ with Azure Firewall - UDR Configuration Requirements

The route tables and routes configured for this pattern are, for the most part, the same as for pattern [Single DMZ](#Single-DMZ---UDR-Configuration-Requirements) except that the routes now point to the VNet IP address of Azure Firewall instead of the load balancer. Additionally, the following route tables and routes are required:

- A route table is configured for Azure Firewall subnet. Gateway route propagation is enabled and the following routes are configured:
  - Summary networks being advertised from the peer router NVAs to Azure Route Server in UNTRUSTED-VNET pointing to the IP address of UNTRUSTED-ILB. In the example above this is RFC1918 supernets 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16. This route effectively bypasses the routers in DMZ-VNET for application and user data traffic
  - A default route with next-hop type of Internet. This route is only needed if providing Internet breakout through Azure Firewall for on premises
- A route table is configured for the router subnet in DMZ-VNET. Gateway route propagation is enabled and a default route is configured pointing to the VNET IP address of Azure Firewall. This route is only needed if providing Internet breakout through Azure Firewall for on premises

### Single DMZ with Azure Firewall - Load Balancer Configuration Requirements

A single ILB is required for the router NVAs in UNTRUSTED-VNET. All other requirements are the same as for pattern [Single DMZ](#Single-DMZ---Load-Balancer-Configuration-Requirements)

### Single DMZ with Azure Firewall - Limitations

The same limitations to pattern [Single DMZ](#Single-DMZ---Limitations) apply

### Single DMZ with Azure Firewall - Extensibility

This pattern can be extended similar to pattern [Single DMZ](#Single-DMZ---Extensibility) with one exception. If a dedicated Azure Firewall for Internet breakout is required it needs to be deployed in a VNET peered to DMZ-VNET because a single Azure Firewall per VNET is supported. Additionally an ILB for the routers in DMZ-VNET is required and the default route configured in route tables associated with the VNETs peered with DMZ-VNET must now point to this ILB. The default route configured in route tables associated with the subnets in DMZ-VNET will point to the IP address of the Azure Firewall dedicated for Internet breakout.
