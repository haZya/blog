---
title: "Modern AWS Networking: Architecting with Transit Gateway, Cloud WAN, and VPC Lattice"
datePublished: 2026-06-29T18:26:11.502Z
cuid: cmqzjttw400000aj3cigqcsou
slug: modern-aws-networking-architecting-with-transit-gateway-cloud-wan-and-vpc-lattice
cover: https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/9870814b-d171-4922-bfec-665537f157f4.png
tags: microservices, aws, cloud-computing, devops, system-architecture, networking

---

In modern enterprise cloud adoption, managing network topology across multiple AWS accounts, geographical regions, and on-premises environments is a critical challenge. A simple hub-and-spoke model is rarely enough. Large-scale deployments require a highly available, secure, and scalable architecture that bridges on-premises data centers with global AWS footprints.

This article dissects a robust, production-grade hybrid-cloud network architecture utilizing **AWS Transit Gateway (TGW)**, **AWS Direct Connect (DX)** with transit virtual interfaces (VIFs), and redundant **IPsec Site-to-Site VPNs** for high availability (HA). We will explore the layout, analyze key design decisions, and address critical routing, DNS, and encryption gotchas you must navigate to run this in production.

## The Architecture at a Glance

The architecture employs a central hub-and-spoke pattern optimized for multi-account organizations. Here is a high-level representation of the architecture:

![AWS Transit Gateway Hybrid Architecture.](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/8dc49976-690f-4a31-8e2e-8e383b4e6d71.png align="center")

### Architectural Pillars

1.  **Core Networking Account (**`Account A`**)**:
    
    *   Hosts the central **Transit Gateway (TGW)** in the primary region (`us-east-1`).
        
    *   Acts as the central traffic hub, managing a unified **TGW Route Table** that processes both VPC-to-VPC traffic and on-premises routes learned dynamically via Border Gateway Protocol (BGP).
        
    *   Uses **AWS Resource Access Manager (RAM)** to share the central TGW across the AWS Organization to spoke accounts.
        
2.  **Spoke Accounts (**`Account 1` **&** `Account 2`**)**:
    
    *   Host standard workloads in dedicated VPCs (`VPC 1` & `VPC 2`).
        
    *   Connect to the central TGW via local TGW attachments. This isolates traffic at the account boundary while ensuring seamless transit routing.
        
3.  **Connectivity/Edge Account (**`Account 3`**)**:
    
    *   Isolates edge connectivity and public-facing ingress/egress.
        
    *   Houses the **Direct Connect Gateway (DX GW)** and the edge connectivity constructs used for Direct Connect and Site-to-Site VPN failover.
        
    *   Associates the DX GW with the central TGW and terminates VPN failover as a **Transit Gateway VPN attachment**.
        
4.  **On-Premises Infrastructure**:
    
    *   Connected via **AWS Direct Connect (DX)** over a Transit VIF as the primary data path (high speed, low latency).
        
    *   Backed up by a **BGP-based Redundant IPsec VPN** over the public internet to provide high availability (HA) and predictable route failover.
        
5.  **Cross-Region Extension**:
    
    *   Extends the mesh to secondary regions (e.g., `eu-west-1`) using **Inter-Region TGW Peering Attachments** between local Transit Gateways.
        

## Deep-Dive: Key Technical Details & Practical Workarounds

While AWS makes setting up a Transit Gateway straightforward, several real-world limitations require deliberate workarounds during implementation.

### 1\. Multi-Account Sharing via AWS RAM

AWS Transit Gateway is a regional resource, but its utility spans accounts. Rather than deploying TGWs in every account (which introduces routing loops and administrative overhead), you should share a single TGW using **AWS Resource Access Manager (RAM)**.

*   **How it works**: The Core Networking Account (`Account A`) shares the TGW resource with the AWS Organization or specific Organizational Units (OUs).
    
*   **The Benefit**: Once shared, administrators in `Account 1` and `Account 2` can create TGW attachments from their local VPCs to the shared TGW. The TGW owner (`Account A`) then accepts the attachments and controls the route tables.
    

### 2\. The Inter-Region Peering Routing Gotcha

A common pitfall when connecting multiple regions (e.g., `us-east-1` and `eu-west-1`) is assuming that routes propagate automatically across peering connections.

> \[!WARNING\] **TGW Peering attachments do NOT support automatic route propagation.** You must manually configure static routes in your Transit Gateway route tables for peered traffic.

#### Workaround: Manual Route Management

For every inter-region peering connection, you must manually add static routes pointing to the peering attachment ID in both TGW route tables.

**Example Scenario:**

*   `VPC 1` (in `us-east-1`) uses the CIDR block `10.0.0.0/16`.
    
*   `VPC 3` (in `eu-west-1`) uses the CIDR block `10.1.0.0/16`.
    

To enable communication across the peered regions:

1.  In **Transit Gateway A's** (us-east-1) route table: Add a static route for `10.1.0.0/16` targeting the **Peering Attachment ID**.
    
2.  In **Transit Gateway B's** (eu-west-1) route table: Add a static route for `10.0.0.0/16` targeting the **Peering Attachment ID**.
    

### 3\. DNS Resolution Limitations Across TGW Peering

Another critical limitation of Transit Gateway peering is DNS resolution.

> **Transit Gateway peering does not support resolving public or private IPv4 DNS hostnames to private IPv4 addresses** across VPCs on either side of the peering connection using Amazon Route 53 Resolver in another Region.
> 
> Private hosted zones (PHZs) can be associated with VPCs in multiple Regions, but TGW peering alone does not provide that DNS plumbing. If a `eu-west-1` VPC is not associated with the PHZ or configured to forward queries to the right resolver endpoint, instances there will not resolve those records simply because network routes exist.

#### Workarounds for Cross-Region DNS:

*   **PHZ Association Where Appropriate**: If the records are hosted in Route 53 and the consuming VPCs can be associated with the PHZ, associate the PHZ directly with each VPC that needs those names. This is the cleanest option when the domain ownership and account model allow it.
    
*   **Route 53 Resolver Endpoints (Recommended)**: Deploy Route 53 Resolver Inbound Endpoints in the target VPC and Outbound Endpoints in the source VPC. Configure forwarding rules in the source region to direct queries for specific domains (e.g., `*.corp.local`) across the TGW peering connection to the inbound resolver endpoint.
    
*   **Centralized DNS Solution**: Deploy a dedicated pair of DNS servers (e.g., Bind9, Windows DNS, or Route 53 resolvers) in a Shared Services VPC. Configure all VPCs across all regions to forward local DNS requests to these central servers.
    
*   **Direct IP Usage**: Bypass DNS entirely by configuring application workloads to communicate via static IP addresses. However, this is rarely scalable and introduces certificate verification challenges for TLS/HTTPS.
    

### 4\. Securing Direct Connect (DX) Traffic

By default, **AWS Direct Connect connections are not encrypted in transit.** If your compliance framework requires encryption for data moving between your on-premises data center and AWS, you must apply additional layers.

#### Encryption Options:

1.  **MACsec (Layer 2 Encryption)**:
    
    *   If your physical routers support IEEE 802.1AE (MACsec) and you have a dedicated 10 Gbps or 100 Gbps Direct Connect port, you can enable MACsec encryption directly on the physical connection between your router and the AWS device.
        
2.  **IPsec VPN over Direct Connect (Layer 3 Encryption)**:
    
    *   For connections where MACsec is not supported (such as hosted connections under 10 Gbps), you can provision a **Site-to-Site VPN** that runs *over* a Transit VIF or public VPN endpoints over a Public VIF on your Direct Connect connection. This creates a secure, encrypted tunnel over the private physical link.
        

#### Performance Tip: LAG (Link Aggregation Groups)

To scale bandwidth and improve link-level resiliency, you can use **Link Aggregation Groups (LAGs)**. LAG allows you to group multiple physical Direct Connect connections into a single logical link using the Link Aggregation Control Protocol (LACP). This active-active configuration increases aggregate throughput and protects against an individual member link failure, but for a complete DX HA strategy for production, use redundant Direct Connect connections across separate devices and, ideally, separate DX locations, with VPN or secondary DX paths available for failover.

## Architectural Alternatives: AWS Cloud WAN

As networks scale to dozens of regions and hundreds of accounts, manually managing TGW route tables, peering attachments, and static routes can become administratively prohibitive. For large-scale, multi-region deployments, consider migrating to **AWS Cloud WAN** via **AWS Network Manager**.

Unlike Transit Gateway, where you peer individual regional hubs, Cloud WAN manages a unified global network based on a central **Core Network Policy**.

![AWS Cloud WAN Hybrid Architecture.](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/a5d6b370-5078-4938-ba89-2c87ba2b27d7.png align="center")

### Key Architectural Shifts with Cloud WAN:

1.  **Core Network Edges (CNEs)**:
    
    *   Instead of regional Transit Gateways, you define a single global network. AWS automatically deploys and manages **Core Network Edges (CNEs)** in your designated regions (`CNE 1` in `us-east-1` and `CNE 2` in `eu-west-1`).
        
2.  **Unified Edge Connectivity**:
    
    *   Spoke VPCs attach directly to the regional CNE using a standard **VPC Attachment**.
        
    *   The DX GW with Transit VIF can be associated with the Cloud WAN core network, while Site-to-Site VPNs connect through **Cloud WAN VPN attachments**.
        
3.  **Global Segments & Policies**:
    
    *   The Cloud WAN configuration is driven by a single **Core Network Policy** where you declare routing rules and logical boundaries (e.g., `Segment: Prod`).
        
    *   The system handles the dynamic propagation of routes (BGP-learned and VPC CIDRs) globally, removing the peering-static-route burden described above for normal attachment reachability.
        

### Why choose Cloud WAN?

*   **Policy-Driven Configuration**: Instead of manually configuring attachments and route tables, you define a global network policy document. AWS automatically provisions the underlying infrastructure.
    
*   **Dynamic Routing**: Cloud WAN manages routing dynamically across regions, eliminating the need for manual static route additions on peered connections.
    
*   **Built-in Segmentation**: Easily segment environments (e.g., Production vs. Development) globally using tag-based policies.
    

## Service-Centric Networking: AWS VPC Lattice

While Transit Gateway and Cloud WAN solve network-level routing challenges, modern microservice architectures often require service-to-service connectivity that operates independently of the underlying IP routing topology. This is where **AWS VPC Lattice** fits in.

### What is VPC Lattice?

VPC Lattice is a fully managed networking service that lets you connect, secure, and monitor services and resources across different VPCs and AWS accounts. For application services, it provides HTTP/HTTPS/gRPC routing with listeners, rules, and service-level authorization. For private resources, it can also expose TCP endpoints through resource configurations and resource gateways.

Unlike Transit Gateway or Cloud WAN, which connect networks and route packets based on IP addresses, VPC Lattice exposes service and resource endpoints through DNS and policy. For application services, it routes requests based on service names, listeners, and HTTP-style rules; for TCP resources, it provides private connectivity without requiring broad CIDR-based network peering.

![AWS VPC Lattice Hybrid Architecture.](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/5299b809-3626-4248-b79a-b6f07da1a618.png align="center")

### Key Differences: Network Peering vs. Service Peering

| Feature | AWS Transit Gateway / Cloud WAN | AWS VPC Lattice |
| --- | --- | --- |
| **OSI Layer** | Layer 3 & Layer 4 (Network/Transport) | Primarily Layer 7 for services; TCP for resource configurations |
| **Primary Identifier** | IP Addresses, CIDR ranges, and BGP routes | DNS hostnames, service/resource endpoints, and listener rules |
| **Overlapping CIDRs** | Requires complex NAT configurations | Supported natively (IPs do not conflict) |
| **Security Mechanism** | Security Groups & Network ACLs | IAM/auth policies, service policies, and association-level security groups |
| **Integration** | EC2, VPCs, VPNs, Direct Connect | EC2, ECS, EKS, AWS Lambda, ALB, IP/TCP resources, and RDS resource configurations |

### Where does VPC Lattice fit in?

*   **Use VPC Lattice** if you want developers to connect services and private resources across accounts without coordinating CIDR block allocations or managing massive route tables, and if you want built-in service discovery, L7 traffic management, and IAM authorization at the service level.
    
*   **Use Transit Gateway / Cloud WAN** if you need broad IP connectivity, UDP or protocols outside the Lattice service/resource model, centralized network inspection, transitive VPC routing, or large-scale routing back to on-premises data centers. VPC Lattice service-network and resource endpoints can be accessed from on-premises over Direct Connect or VPN when DNS and routing are configured, but they are not a replacement for the hybrid backbone itself.
    
*   **Hybrid Approach (Best of Both Worlds)**: Enterprises often deploy **Transit Gateway** or **Cloud WAN** for baseline north-south hybrid connectivity (on-premises to cloud edge), egress filtering, and shared network services, while using **VPC Lattice** for internal, east-west service-to-service and resource access between developers' Kubernetes clusters, ECS workloads, and serverless applications.
    

## Summary & Key Takeaways

Building a hybrid cloud network requires balancing scalability, performance, and complexity. The TGW-hub-and-spoke model outlined here is a battle-tested architecture that provides:

1.  **Separation of Concerns**: Core networking, spoke applications, and edge/transit links are isolated across accounts.
    
2.  **High Availability**: Direct Connect Transit VIF functions as the primary pipe, backed up dynamically by a BGP-based IPsec VPN.
    
3.  **Structured Scale**: Organization-wide resource sharing via AWS RAM reduces billing overhead and architecture sprawl.
    

By planning for the static routing requirements of TGW peering, setting up PHZ associations or Route 53 Resolver endpoints for cross-region DNS, and implementing Layer 2/3 encryption on Direct Connect, you can build a resilient, secure hybrid cloud network ready for enterprise production workloads.