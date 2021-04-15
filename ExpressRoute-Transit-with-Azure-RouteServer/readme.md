# Enable Transit Between ExpressRoute Circuits without Using Global Reach

[ExpressRoute Global Reach](https://docs.microsoft.com/azure/expressroute/expressroute-global-reach) allows branches connected to Azure via ExpressRoute circuits to communicate with each other using [Microsoft global network](https://docs.microsoft.com/azure/networking/microsoft-global-network); however there are some limitations with it:

- It is not available in all countries due to regulatory compliance reasons
- Circuits must be connected to different [ExpressRoute locations](https://docs.microsoft.com/azure/expressroute/expressroute-locations-providers#locations)
- It is point to point and non-transitive
- If the circuits are in different geopolitical regions both of them must be [Premium SKU](https://docs.microsoft.com/azure/expressroute/expressroute-introduction#global-connectivity-with-expressroute-premium)

These limitations can be problematic when, for example, there is a requirement to connect to services like [Azure VMWare Solution (AVS)](https://docs.microsoft.com/azure/azure-vmware/introduction), [SAP HANA on Azure Large Instances (HLI)](https://docs.microsoft.com/azure/virtual-machines/workloads/sap/hana-overview-architecture) or [Oracle Cloud Infrastructure (OCI)](https://docs.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/connectivity-to-other-providers) from on-premises using ExpressRoute.

This article offers a set of reference architectures to enable transit between ExpressRoute circuits leveraging [Azure Route Server](https://docs.microsoft.com/azure/route-server/overview) and [Azure Virtual WAN](https://docs.microsoft.com/azure/virtual-wan/). In order to keep it succinct no configuration examples are provided, but rather the advantages and disadvantages of each design are discussed.

> Although these patterns have been personally validated and tested there are no guarantees Microsoft will support them.

## 1. Single Region – IPSec VPN over ExpressRoute Private Peering using Azure VPN Gateway

![Single Region – IPSec over ExpressRoute Private Peering Route Server Reference Architecture][diagram1]

[diagram1]: ../Images/expressroute-transit/AVS-TRANSIT-AZURE-VNG.png

> Microsoft might disable VPN to ExpressRoute transit for regions where Global Reach is not available due to regulatory compliance reasons. Do not rely on this pattern in those regions.

Azure Route Server enables [transit between VPN and ExpressRoute](https://docs.microsoft.com/azure/route-server/quickstart-configure-route-server-cli#configure-route-exchange) in the same VNET; additionally it is now possible to establish [Site-to-Site VPN over ExpressRoute private peering](https://docs.microsoft.com/azure/vpn-gateway/site-to-site-vpn-private-peering) when using zone redundant VPN gateways, making this pattern a simple and effective option when all ExpressRoute circuits are connected to the same Azure VNET regardless of ExpressRoute location and circuit SKU.

It is possible to achieve transit across multiple circuits up to the [maximum of 4](https://docs.microsoft.com/azure/azure-resource-manager/management/azure-subscription-service-limits#expressroute-limits); however only one circuit can be left without configuring Site-to-Site VPN on top of it, limiting this pattern to allow transit from on-premises to only one of AVS, HLI or OCI.

**Advantages:**

- No need to maintain and configure third party NVAs

- All components deployed in Azure are first party managed PaaS services

- Offers high availability by default

- Simple and easy to deploy

- Static route IPSec tunnels can be used. BGP is not needed but is recommended

- [ExpressRoute FastPath](https://docs.microsoft.com/azure/expressroute/about-fastpath) can be enabled for improved performance

**Disadvantages:**  

- Only supported in [Azure regions with availability zones](https://docs.microsoft.com/azure/availability-zones/az-region#azure-regions-with-availability-zones)

- Route Server, ExpressRoute Gateway and VPN Gateway must be deployed in the same VNET

- Only one ExpressRoute circuit can be left without configuring Site-to-Site VPN on top of it

- Transit from on-premises to only one of AVS, HLI or OCI is supported

- Traffic between on-premises branches cannot be filtered nor inspected in Azure

- Increased latency because of IPSec overhead

- A single [IPSec tunnel throughput is capped at 1 Gbps](https://docs.microsoft.com/azure/vpn-gateway/vpn-gateway-about-vpn-gateway-settings#benchmark) depending on the encryption and integrity algorithms used and when using BGP [configuring multiple tunnels with Azure VPN gateway](../Multiple-VPN-Tunnels-From-CiscoCSR-to-AzureVNG/readme.md) can be challenging.

## 2. IPSec VPN over ExpressRoute Private Peering using Azure virtual WAN

![IPSec over ExpressRoute Private Peering vWAN Reference Architecture][diagram2]

[diagram2]: ../Images/expressroute-transit/AVS-TRANSIT-vWAN.png

> Microsoft has disabled VPN to ExpressRoute transit in Azure virtual WAN for regions where Global Reach is not available due to regulatory compliance reasons.

Azure virtual WAN natively provides transit between on-premises branches, be it Site-to-Site, Point-to-Site or ExpressRoute, either within a region or across regions; however transit between ExpressRoute circuits still requires Global Reach to be enabled. By configuring Site-to-Site VPN over ExpressRoute private peering it is possible to achieve transit across ExpressRoute circuits; the added benefit of this pattern is it can be extended to support multi-region designs by deploying virtual hubs in different regions. Finally, Site-to-Site VPN over ExpressRoute private peering in virtual WAN is not limited to Azure regions that support availability zones.

**Advantages:**

- No need to maintain and configure third party NVAs

- All components deployed in Azure are first party managed PaaS services

- Offers high availability by default

- Simple and easy to deploy

- Supports multi-region deployments

- In multi-region deployments, ExpressRoute circuits in different Azure and geopolitical regions can be [Local SKU](https://docs.microsoft.com/azure/expressroute/expressroute-faqs#what-is-expressroute-local)

- Static route IPSec tunnels can be used. BGP is not needed but is recommended

**Disadvantages:**  

- Only supported in regions where Global Reach is available

- Only one ExpressRoute circuit can be left without configuring Site-to-Site VPN on top of it across all virtual hubs

- Transit from on-premises to only one of AVS, HLI or OCI is supported across all virtual hubs

- Traffic between on-premises branches cannot be filtered nor inspected in Azure

- Increased latency because of IPSec overhead

- A single [IPSec tunnel throughput is capped at 1 Gbps](https://docs.microsoft.com/azure/vpn-gateway/vpn-gateway-about-vpn-gateway-settings#benchmark) depending on the encryption and integrity algorithms used and when using BGP [configuring multiple tunnels with Azure VPN gateway](../Multiple-VPN-Tunnels-From-CiscoCSR-to-AzureVNG/readme.md) can be challenging.

## 3. Single Region – IPSec VPN over ExpressRoute Private Peering using NVA

![IPSec over ExpressRoute Private Peering NVA Reference Architecture][diagram3]

[diagram3]: ../Images/expressroute-transit/AVS-TRANSIT-SINGLE-REGION-NVA.png

By using an NVA to terminate IPSec VPN tunnels instead of Azure native solutions some of the limitations in the previous two patterns can be overcome; however it introduces the complexity of having to maintain and configure the NVA as well as any licensing costs associated with it.

This pattern requires BGP over IPSec configured between the NVA in Azure and the VPN appliance on-premises and [route exchange](https://docs.microsoft.com/azure/route-server/quickstart-configure-route-server-cli#configure-route-exchange) enabled in Azure Route Server. It can also be implemented using VxLAN in lieu of IPSec on top of ExpressRoute

It is possible to achieve transit across multiple circuits up to the [maximum of 4](https://docs.microsoft.com/azure/azure-resource-manager/management/azure-subscription-service-limits#expressroute-limits); however only one circuit can be left without configuring IPSec VPN or VxLAN on top of it, limiting this pattern to allow transit from on-premises to only one of AVS, HLI or OCI.

> Azure fabric load shares traffic across up to eight equal-cost paths. If multiple NVAs are deployed and it is required to filter and inspect traffic between on-premises branches, implement an Active/Passive deployment leveraging BGP AS-PATH prepending to prevent traffic from being dropped because of asymmetrical routing.

**Advantages:**

- Supported across all Azure regions

- Traffic between on-premises branches can be filtered and inspected at the NVA

- [ExpressRoute FastPath](https://docs.microsoft.com/azure/expressroute/about-fastpath) can be enabled for improved performance

**Disadvantages:**  

- The NVAs must be managed by the user

- Only one ExpressRoute circuit can be left without configuring IPSec VPN or VxLAN on top of it

- Transit from on-premises to only one of AVS, HLI or OCI is supported

- Increased latency because of the tunnel overhead

- A single tunnel throughput, whether it is IPSec or VxLAN, is capped at ~1.5 Gbps. This is a limitation of tunnels in virtualized environments

- Multiple NVAs are needed for high availability

## 4. Multi-Region – Multi-NIC NVAs with Route Tables

![Multi-NIC NVA Reference Architecture][diagram4]

[diagram4]: ../Images/expressroute-transit/AVS-TRANSIT-MULTI-NIC-NVA-UDR.png

This pattern allows on-premises branches in any Azure region, regardless of the ExpressRoute circuit SKU, to communicate without the need to configure tunnels; however because there are route tables involved it doesn't scale well beyond two VNETs. Following are the key considerations with this pattern:

- NVAs must remove or replace all BGP ASNs used by Azure from the route advertisements towards Azure Route Server to guarantee route propagation between branches. These ASNs are:

  - 65515 - Used by Azure Route Server and Azure ExpressRoute gateway

  - 12076 - Used by ExpressRoute private peering

  - 398656 - Used by AVS and HLI

- BGP sessions are established using IP addresses associated with a NIC. It is not possible to use loopback interfaces or APIPA addresses

- Each NVA establishes a BGP peering with its local Azure Route Server and one remote NVA per VNET

- "Allow Gateway Transit" and "Use Remote Gateway" options are disabled for the VNET peerings

- When deploying multiple NVAs for high availability the NICs used to establish BGP sessions between NVAs must reside in different subnets and each subnet is associated with a route table that has routes towards the remote on-premises branch address prefixes pointing to its NVA BGP neighbor IP address as next-hop

- If only two VNETs are required and only addresses from [RFC1918](https://tools.ietf.org/html/rfc1918) or Carrier-Grade NAT (100.64/10) address spaces are used across Azure and on-premises the route tables can be simplified to include only routes towards the supernets. Furthermore, if the NVAs are not used for Internet breakout a single route towards 0.0.0.0/0 can be used across all route tables

- Gateway routes propagation is disabled across all route tables

- This pattern can be used in conjunction with pattern 3. Single Region – IPSec VPN over ExpressRoute Private Peering using NVA to connect up to four ExpressRoute circuits to each VNET

> Azure fabric load shares traffic across up to eight equal-cost paths. If multiple NVAs are deployed in the same VNET and it is required to filter and inspect traffic between on-premises branches, implement an Active/Passive deployment leveraging BGP AS-PATH prepending for route advertisements towards Azure Route Server to prevent traffic from being dropped because of asymmetrical routing.

**Advantages:**

- Supported across all Azure regions

- Traffic between on-premises branches can be filtered and inspected at the NVAs

- Traffic between on-premises branches can be filtered using [network security groups](https://docs.microsoft.com/azure/virtual-network/network-security-groups-overview) at the NVA NICs subnets facing Azure Route Server if no IPSec or VxLAN tunnels are used

- ExpressRoute circuits can be [Local SKU](https://docs.microsoft.com/azure/expressroute/expressroute-faqs#what-is-expressroute-local)

- Throughput is not constrained by tunnels

- [ExpressRoute FastPath](https://docs.microsoft.com/azure/expressroute/about-fastpath) can be enabled for improved performance

**Disadvantages:**  

- The NVAs must be managed by the user

- TCO is high. It requires duplication of artifacts across VNETs

- It is not scalable. As more VNETs are introduced route table management complexity increases exponentially

- Complex to configure, operate and troubleshoot

## 5. Multi-Region – Single-NIC NVAs with VxLAN

![Single-NIC NVA Reference Architecture][diagram5]

[diagram5]: ../Images/expressroute-transit/AVS-TRANSIT-SINGLE-NIC-NVA-VxLAN.png

This pattern allows on-premises branches in any Azure region to communicate between them regardless of the ExpressRoute circuit SKU. It can be used with both single-NIC and multi-NIC NVAs. Because there are no route tables involved it is better suited for large deployments with multiple on-premises branches across multiple regions. IPSec can be used in lieu of VxLAN. Following are the key considerations with this pattern:

- NVAs must remove or replace all BGP ASNs used by Azure from the route advertisements towards Azure Route Server to guarantee route propagation between branches. These ASNs are:

  - 65515 - Used by Azure Route Server and Azure ExpressRoute gateway

  - 12076 - Used by ExpressRoute private peering

  - 398656 - Used by AVS and HLI

- NVAs establish VxLAN tunnels with all remote NVAs

- A dedicated address space must be reserved for the VxLAN interfaces. It must not be used anywhere else across Azure and on-premises. It is however possible to use APIPA addresses for the VxLAN interfaces

- BGP sessions between NVAs are established using IP addresses associated with the VxLAN interfaces

- BGP sessions between NVAs and Azure Route Server are established using IP addresses associated with a NIC. It is not possible to use loopback interfaces or APIPA addresses

- NVAs establish BGP peerings with their local Azure Route Server and all remote NVAs

- NVAs must not advertise VxLAN interface addresses to Azure Route Server if APIPA addresses are used

- Each VxLAN tunnel throughput is capped at ~1.5 Gbps. If multiple tunnels between a pair of NVAs are needed the NICs must be assigned multiple IP configurations

- If multiple NVAs are deployed in the same VNET ECMP must be enabled in BGP to leverage all VxLAN tunnels and prevent traffic from being dropped because of asymmetrical routing in Azure SDN

- This pattern can be used in conjunction with pattern 3. Single Region – IPSec VPN over ExpressRoute Private Peering using NVA to connect up to four ExpressRoute circuits to each VNET

> Azure fabric load shares traffic across up to eight equal-cost paths. If multiple NVAs are deployed in the same VNET and it is required to filter and inspect traffic between on-premises branches, implement an Active/Passive deployment leveraging BGP AS-PATH prepending for route advertisements towards **Azure Route Server and NVAs** to prevent traffic from being dropped because of asymmetrical routing.

**Advantages:**

- Supported across all Azure regions

- Traffic between on-premises branches can be filtered and inspected at the NVAs

- ExpressRoute circuits can be [Local SKU](https://docs.microsoft.com/azure/expressroute/expressroute-faqs#what-is-expressroute-local)

- Route tables are not required

- It can scale to support multiple VNETs

- Traffic between on-premises branches can be filtered using [network security groups (NSGs)](https://docs.microsoft.com/azure/virtual-network/network-security-groups-overview) at the NVA NICs subnets facing Azure Route Server if no IPSec or VxLAN tunnels are used for multi-NIC NVAs. For single-NIC NVAs the NSG can be applied in the NVA NIC subnet if no IPSec or VxLAN tunnels are used for multi-NIC NVAs

- [ExpressRoute FastPath](https://docs.microsoft.com/azure/expressroute/about-fastpath) can be enabled for improved performance

**Disadvantages:**  

- The NVAs must be managed by the user

- TCO is high. It requires duplication of artifacts across VNETs

- A single tunnel throughput, whether it is IPSec or VxLAN, is capped at ~1.5 Gbps. This is a limitation of tunnels in virtualized environments

- Increased latency because of the tunnel overhead

- Complex to configure, operate and troubleshoot
