# Inspect Azure Application Gateway Internet Bound Traffic Using an NVA

## Preface

The distributed nature of the cloud coupled with the evolution of the threat landscape in recent years calls for a paradigm shift in how we think about security in general and network security in particular. Gone are the days when placing a firewall at the edge of our network was enough to keep the bad guys at bay, and although firewalls still offer tremendous value for securing our network they cannot be the centerpiece of our network security strategy when it comes to cloud; instead we need to take a distributed approach to network security, leveraging cloud native tools such as Cloud Security Posture Management ([CSPM](https://www.gartner.com/documents/3899373/innovation-insight-for-cloud-security-posture-management)) like [Azure Security Center](https://docs.microsoft.com/azure/security-center/security-center-introduction) and  Cloud Workload Protection Platforms ([CWPP](https://www.gartner.com/reviews/market/cloud-workload-protection-platforms)) like [Azure Defender](https://docs.microsoft.com/azure/security-center/azure-defender) (formerly known as Azure Security Center Standard Tier) , cloud native SIEM and Security Operations Automation and Response systems ([SOAR](https://www.gartner.com/reviews/market/security-orchestration-automation-and-response-solutions)) like [Azure Sentinel](https://techcommunity.microsoft.com/t5/azure-sentinel/bg-p/AzureSentinelBlog), host based IDS and IPS and agents across our infrastructure components.

This distributed approach to security is even more relevant when working with PaaS services that can be deployed natively in the customer's virtual network which, due to their managed nature, need to communicate with the cloud provider's control plane that sits on the Internet, and the very thought of this makes a lot of people nervous. One of those services, and the focus of this article is [Azure Application Gateway](https://docs.microsoft.com/azure/application-gateway/overview). Often customers want to inspect traffic sourced from Azure Application Gateway through a Firewall, and although concerns about data exfiltration, command and control botnets and other attacks are very valid and need to be taken seriously, this architecture, from my perspective, adds little to no value at all because of the following reasons:

- Azure Application Gateway is a reverse proxy for HTTP, HTTPS, HTTP/2 and WebSockets traffic, as such it doesn't initiate outbound Internet traffic, rather it relays responses coming from the backend servers back to the clients that originated the request. If Application Gateway receives traffic from a backend server that doesn't correspond to a previously received request it will be dropped; thus if a backend server is compromised through lateral movement or any other attack vector it cannot use Azure Application Gateway as a proxy towards the Internet

- [Azure Web Application Firewall](https://docs.microsoft.com/azure/web-application-firewall/ag/ag-overview) can be associated with Azure Application Gateway, this protects the backend servers from malicious requests. It can also leverage [Microsoft Threat Intelligence feed](https://docs.microsoft.com/azure/web-application-firewall/ag/bot-protection-overview) to block requests from known malicious IP addresses

- In order to compromise Azure Application Gateway an entity would require to gain access into it. Although Azure Application Gateway requires that certain ports remain open in order to allow management of the service, [these ports are protected by Azure Certificates](https://docs.microsoft.com/azure/application-gateway/configuration-infrastructure#network-security-groups) to only allow connections coming from Azure GatewayManager service

- [Network Security Groups can be configured in the Application Gateway Subnet](https://docs.microsoft.com/azure/application-gateway/configuration-infrastructure#network-security-groups), further restricting the sources that can connect to it. Additionally the service tag for GatewayManager service can be leveraged, restricting Internet clients to only web traffic destined to the ports configured on the listeners

- Most regulatory compliance standards require traffic to be inspected by a Firewall, however these standards consider Azure NSGs Firewalls, which are supported with Azure Application Gateway, and although [NSG flow logs are not supported on NSGs associated to Application Gateway v2 subnet](https://docs.microsoft.com/azure/application-gateway/application-gateway-faq#are-nsg-flow-logs-supported-on-nsgs-associated-to-application-gateway-v2-subnet), Azure Application Gateway offers [comprehensive logging](https://docs.microsoft.com/azure/application-gateway/application-gateway-diagnostics#diagnostic-logging) that can be used for compliance reporting as well as to analyze and investigate traffic patterns

- [Azure Web Application Firewall integrates with Azure Sentinel](https://techcommunity.microsoft.com/t5/azure-network-security/integrating-azure-web-application-firewall-with-azure-sentinel/ba-p/1720306), making it easier for security teams to visualize patterns, detect potential malicious activities and respond to threats faster, as well as [detect and defend against data disclosure and exfiltration](https://techcommunity.microsoft.com/t5/azure-network-security/part-4-data-disclosure-and-exfiltration-playbook-azure-waf/ba-p/2031269) using the [Azure Monitor Workbook for WAF](https://github.com/Azure/Azure-Network-Security/tree/master/Azure%20WAF/Azure%20Monitor%20Workbook)

- Azure Defender can be used to detect compromised servers and data exfiltration directly at the servers through the Log Analytics Agent for Windows and Linux servers, not only in Azure but also on premises and any other cloud, rather than needing to rely on a centralized control point like a firewall. You can review the [set of alerts](https://docs.microsoft.com/azure/security-center/alerts-reference) that Azure Defender surfaces for IaaS and PaaS services. Even if you decide to inspect Application Gateway traffic through a firewall, I urge you to consider implementing Azure Defender in your security strategy as it provides invaluable information to keep your cloud environment secure

- Inspecting Azure Application Gateway traffic through a firewall can block the communication with the control plane due to misconfiguration, which can result in a self inflicted DoS attack rendering your service unavailable

If after reading this section you still need to inspect all traffic from Application Gateway through a firewall keep on reading to learn how you can accomplish this.

## Overview

Azure Application Gateway v2 doesn't currently support associating a route table containing route 0.0.0.0/0 and a next hop of Virtual Appliance ([ref](https://docs.microsoft.com/azure/application-gateway/configuration-infrastructure#supported-user-defined-routes)), this is because Gateway Manager - the control plane responsible for the management and provisioning of Gateways in Azure - which is hosted in a Microsoft owned tenant can only communicate with the public IP address assigned to Application Gateway, thus routing outbound Internet traffic from Application Gateway through a Virtual Appliance would create an asymmetric routing situation where Gateway Manager receives the response from the public IP address associated with the NVA, breaking communication and rendering the Application Gateway unusable. Additionally, lets keep in mind that certificate authentication is used between Application Gateway and Gateway Manager, so even if Gateway Manager accepted traffic coming from the NVA public IP address TLS inspection would break communication because Application Gateway wouldn't trust the root certificate used by the NVA to sign the certificate generated to decrypt TLS traffic.

This article will expand on how to solve for the above challenge - bypass inspection for traffic destined to Gateway Manager while still inspecting all other Internet bound traffic from Application Gateway.

>[!NOTE]
Even though Gateway Manager has a public IP address and is considered to be on the Internet, this IP address belongs to Microsoft's Global Network, thus communication between Application Gateway and Gateway Manager never really traverses the Internet and stays within Microsoft's low latency, high throughput secure network. You can read more about Microsoft's Global Network [here](https://docs.microsoft.com/en-us/azure/networking/microsoft-global-network)

## Configuration

>[!NOTE]
The configuration presented in this article uses a feature which is currently in Private Preview. Although there is no need to whitelist your subscription to be able to use it and is available in all Azure commercial regions it is still governed by the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/en-us/support/legal/preview-supplemental-terms/)

[Service Tags](https://docs.microsoft.com/en-us/azure/virtual-network/service-tags-overview) have been around for quite a while in Azure to facilitate configuration of NSGs and more recently Azure Firewall; however UDRs haven't supported service tags historically and that has been a major challenge when working with PaaS services. Fortunately that has changed and support for service tags in UDRs is now available through command line; this allows to create a UDR containing a route for GatewayManager service tag and next hop of Internet to bypass the NVA as follows:

```bash
az network route-table create -g APPGW-UDR -n APPGW-ROUTE-TABLE
az network route-table route create -g APPGW-UDR --route-table-name APPGW-ROUTE-TABLE -n GWMRoute --address-prefix GatewayManager.SouthCentralUS --next-hop-type Internet

```

We now need to create another route to send all traffic through the NVA, but if we add route 0.0.0.0/0 then we won't be able to associate it with the Application Gateway Subnet; however we can partition this route into two routes as follows:

```bash
az network route-table route create -g APPGW-UDR --route-table-name APPGW-ROUTE-TABLE -n UPPER-HALF --address-prefix 0.0.0.0/1 --next-hop-type VirtualAppliance --next-hop-ip-address 10.255.250.4
az network route-table route create -g APPGW-UDR --route-table-name APPGW-ROUTE-TABLE -n LOWER-HALF --address-prefix 128.0.0.0/1 --next-hop-type VirtualAppliance --next-hop-ip-address 10.255.250.4

```

Our final route table will look like this:

Name | Address Prefix | Next hop type
--- | --- | ---
APPGW-ROUTE-TABLE | GatewayManager.SouthCentralUS | Internet
UPPER-HALF | 0.0.0.0/1 | 10.255.250.4
LOWER-HALF | 128.0.0.0/1 | 10.255.250.4

This route table can now be associated with Application Gateway Subnet, forcing inspection of all user traffic through our firewall while allowing control plane traffic to flow directly to the Internet without breaking Application Gateway.

>[!NOTE]
In order to avoid breaking communication due to asymmetric routing it is important that Internet clients connect to the public IP address of the NVA rather than to the Application Gateway directly and configure Destination NAT on the NVA to the private IP address of Azure Application Gateway. This architecture is discussed in detail [here](https://docs.microsoft.com/azure/architecture/example-scenario/gateway/firewall-application-gateway#application-gateway-after-firewall)

>[!NOTE]
GatewayManager service tag is regional, you only need to allow the tag for the region where your Application Gateway is deployed. You can consult the full list of service tags available [here](https://www.microsoft.com/download/details.aspx?id=56519). You can also use CLI command [az network list-service-tags](https://docs.microsoft.com/cli/azure/network?view=azure-cli-latest#az_network_list_service_tags) or PowerShell command [Get-AzNetworkServiceTag](https://docs.microsoft.com/powershell/module/az.network/get-aznetworkservicetag?view=azps-5.4.0) to filter for specific service tags

## Conclusion

With the support of service tags for UDRs it is now possible to inspect Application Gateway Outbound Internet traffic through an NVA; however there is little to no value in doing this