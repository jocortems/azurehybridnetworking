# Inspecting Internet Bound Traffic Through an NVA in Azure Virtual WAN

## Introduction

Currently Azure Virtual WAN doesn't allow a default route through an NVA to be configured in the virtual hub as documented [here](https://docs.microsoft.com/en-us/azure/virtual-wan/scenario-route-through-nvas-custom#workflow), which means that if you require to inspect outbound Internet traffic the only options are to deploy a [secured virtual hub](https://docs.microsoft.com/en-us/azure/firewall-manager/secured-virtual-hub) and use [Azure Firewall](https://docs.microsoft.com/en-us/azure/firewall/overview) in conjunction with [Azure Firewall Manager](https://docs.microsoft.com/en-us/azure/firewall-manager/overview), or leverage one of our [Security Partner Providers](https://docs.microsoft.com/en-us/azure/firewall-manager/trusted-security-partners). Depending on your security policies this might not be enough, but that doesn't mean you cannot use Azure Virtual WAN and still be able to inspect Internet bound traffic using your NVA of choice. In this article we will explore an option to inspect outbound Internet traffic through an NVA in Azure Virtual WAN.

## Architecture

Although it is not possible to configure a default route with next hop NVA directly in the virtual hub it is, however, possible to advertise a default route from a branch connected to the virtual hub either via ExpressRoute or Site to Site VPN. This is, in fact, the way our security partner providers -Zscaler, Check Point and iBoss, integrate with Azure Virtual WAN secured hub to provide Internet traffic filtering. In this case because we want to use an NVA hosted directly in Azure that we have full control over to inspect outbound Internet traffic we can create a dedicated VNET to deploy the NVA of our choice and connect it to Azure Virtual WAN like any other branch using Site to Site IPSec VPN tunnel directly between the NVA and the virtual hub and advertise a default route via BGP. Figure 1 below shows the proposed architecture:

![Figure 1 - Reference Architecture][Figure1]

[Figure1]: ../Images/internet-to-nva-vwan/NVA_VWAN.png "Reference Architecture"

### Considerations

Although the proposed architecture works it is important to keep in mind the following considerations when planning for your deployment:

- The 0.0.0.0/0 route is not propagated across virtual hubs. If you have a multi-region Virtual WAN deployment and require to inspect Internet bound traffic from all VNETs you need to either create an NVA in a dedicated VNET in each region or connect the NVA to each virtual hub via Site to Site IPSec tunnels

- By default the 0.0.0.0/0 route is programmed into every NIC of every VM in every VNET connected to the virtual hub the NVA VNET is connected to. You can selectively disable it by editing the Hub Virtual Network Connection object in the portal or through [REST](https://docs.microsoft.com/rest/api/virtualwan/hubvirtualnetworkconnections/createorupdate) and setting the value of `properties.enableInternetSecurity` to `false`

- By default the 0.0.0.0/0 route is not advertised to any branches. You can selectively advertise it by editing the VPN site configuration in the portal or through [REST](https://docs.microsoft.com/rest/api/virtualwan/vpnconnections/createorupdate) and setting the value of `properties.enableInternetSecurity` to `false`

- An IPSec tunnel can support up to 1.25Gbps of throughput. If you need additional throughput you must create additional tunnels. Note however that each tunnel must use a different public IP address and network interface, and a VM can support up to 8 network interfaces depending on its SKU. Additionally, a virtual hub only provides two public IP addresses to establish Site to Site tunnels with, if you need more than two tunnels you will need to use Front Door VRF and MP-BGP. I have written a detailed article on that [here](https://github.com/jocortems/azurehybridnetworking/tree/main/Multiple-VPN-Tunnels-From-CiscoCSR-to-AzureVNG)

- Although it is possible to advertise route 0.0.0.0/0 from multiple NVAs it is not recommended. The reason is Azure will load share traffic across up to 8 equal cost paths and there is no guarantee of flow symmetry, thus advertising route 0.0.0.0/0 from multiple NVAs can lead to traffic drops because of asymmetric routing

- Traffic between the virtual hub and the NVA will be billed according to our inter region bandwidth charges which you can consult [here](https://azure.microsoft.com/pricing/details/bandwidth/)

- You cannot advertise route 0.0.0.0/0 using Azure native VPN Gateway. If your NVA doesn't support BGP, or if you need more than 2.5Gbps of throughput (2 tunnels) and your NVA doesn't support Front Door VRF and MP-BGP, you can use a separate NVA -like Cisco CSR 1000v or Arista vEOS Router- to establish the IPSec tunnels and BGP routing to the virtual hub and send all traffic to your firewall NVA

## Conclusion

It is possible to use an NVA to inspect Internet bound traffic from Azure Virtual WAN by treating it like an additional branch and advertising a default route to a virtual hub. Just keep in mind the considerations listed in this article.
