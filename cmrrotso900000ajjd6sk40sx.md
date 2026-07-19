---
title: "Amazon EKS Multi-Account Strategy: Architecting Centralized vs. Decentralized Cluster Topologies"
datePublished: 2026-07-19T11:03:40.932Z
cuid: cmrrotso900000ajjd6sk40sx
slug: amazon-eks-multi-account-strategy-architecting-centralized-vs-decentralized-cluster-topologies
cover: https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/a6d5c822-39fd-4392-99c9-5719f5b591e5.png
tags: aws, kubernetes, devops, eks, platform-engineering

---

Implementing a robust multi-account strategy on AWS is one of the most critical foundational steps for building secure, scalable, and manageable cloud platforms. According to the [AWS Multi-Account Strategy Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html), distributing workloads across multiple AWS accounts provides strong logical boundaries, isolates operational issues, simplifies billing, and prevents API rate limiting.

When running containerized workloads on Amazon Elastic Kubernetes Service (EKS), you must align your Kubernetes tenant structures with your AWS account topography. This article explores the two primary architectural patterns outlined in the [AWS EKS Multi-Account Best Practices Guide](https://docs.aws.amazon.com/eks/latest/best-practices/multi-account-strategy.html): **Centralized EKS Clusters** and **Decentralized EKS Clusters**.

## Strategy 1: Centralized EKS Clusters (Shared Cluster Model)

In the **Centralized EKS Cluster** pattern, multiple workloads or application teams share a single large EKS cluster. This does not have to mean one cluster for the entire organization. AWS recommends using multiple cluster accounts when needed to align with software development lifecycle boundaries, limit blast radius, or provide additional Kubernetes and AWS service quota capacity. A common model is to separate production and non-production platforms.

### Core Concepts: Cluster vs. Workload Accounts

In a multi-account EKS topology, AWS resources are divided into two categories:

1.  **Cluster Account**: A dedicated AWS account that hosts the Amazon EKS cluster control plane, node groups (EC2 or AWS Fargate compute resources), the primary VPC, and essential cluster-wide add-ons (such as Karpenter, AWS Load Balancer Controller, and observability agents).
    
2.  **Workload Account**: Individual AWS accounts associated with specific business units, applications, or tenants. These accounts contain workload-specific resources like Amazon RDS instances, DynamoDB tables, S3 buckets, SQS Queues, ElastiCache clusters, and KMS keys, etc.
    

This separation mimics Kubernetes namespaces but at the AWS infrastructure level.

![Centralized EKS Cluster](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/1b91f6a6-63f9-4249-af7c-cda681e3d3e4.png align="center")

### Soft Isolation

A shared cluster uses soft multi-tenancy. Namespaces, RBAC, network policies, Pod Security Standards, quotas, and admission policies can provide strong logical separation, while cross-account access mechanisms are used to consume AWS resources in separate workload accounts.

> **Important:** The cluster remains the primary Kubernetes security boundary. A cluster administrator or a node-level compromise can affect multiple tenants. Workloads that are mutually untrusted or require a stronger boundary should therefore consider dedicated clusters.

#### A. Shared Networking with AWS Resource Access Manager (RAM)

AWS Resource Access Manager (AWS RAM) can share subnets from the cluster account with workload accounts in the same AWS Organization. This allows workload resources (e.g., RDS, ElastiCache) to be launched directly into the cluster's subnet space in their respective workload accounts. This uses the VPC's implicit routing and can avoid creating separate VPCs, peering connections, or transit gateway attachments for those resources.

The VPC owner retains control of VPC-level resources, routing, and network ACLs. Participant accounts own the resources and security groups they create in the shared subnets. This pattern is most appropriate when the accounts are within the same trust boundary, and AWS Organizations resource sharing is enabled.

This CDK snippet utilizes `CfnResourceShare` to securely share data subnets with workload accounts:

```typescript
const principals = [...new Set(config.tenants.map((tenant) => tenant.workloadAccountId))];

new ram.CfnResourceShare(this, 'DataSubnetShare', {
  name: `${config.clusterName}-data-subnets`,
  allowExternalPrincipals: false,
  principals,
  resourceArns: this.dataSubnets.map((subnet) => this.formatArn({
    service: 'ec2',
    resource: 'subnet',
    resourceName: subnet.subnetId,
  })),
});
```

#### B. Cross-Account Credentials via EKS Pod Identity or IRSA

When pods running in the central cluster account need to write to S3 buckets or query databases in a workload account, they require AWS credentials.

Both EKS Pod Identity and IAM roles for service accounts (IRSA) deliver temporary AWS credentials to pods. The choice matters in a multi-account cluster:

*   **EKS Pod Identity** associates a Kubernetes service account with a role in the cluster account. For cross-account access, EKS can use that role to assume a target role in the workload account. AWS recommends Pod Identity for new workloads on supported node types. It requires Linux Amazon EC2 nodes and is not supported by pods on Fargate or Windows nodes. EKS Auto Mode includes the Pod Identity Agent.
    
*   **IRSA** can directly federate to a role in a workload account after the cluster's IAM OIDC provider is configured there. IRSA remains fully supported and is a good fit for existing IRSA deployments and environments that Pod Identity does not support, including Fargate.
    

AWS EKS Pod Identity supports cross-account credential delivery using **role chaining**. In the cluster account, it creates a local pod role and includes both that role and the target workload-account role in the Pod Identity association.

1.  **In the Cluster Account**: The platform stack creates a local `podRole` and maps it via `CfnPodIdentityAssociation`, specifying the target cross-account role ARN:
    
    ```typescript
    const podRole = new iam.Role(blueprint, `${tenant.namespace}-pod-role`, {
      roleName: `${tenant.namespace}-pod`,
      assumedBy: new iam.ServicePrincipal('pods.eks.amazonaws.com').withSessionTags(),
    });
    
    const targetRoleArn = `arn:${blueprint.partition}:iam::${tenant.workloadAccountId}:role/${tenant.namespace}-target`;
    podRole.addToPolicy(new iam.PolicyStatement({
      actions: ['sts:AssumeRole', 'sts:TagSession'],
      resources: [targetRoleArn],
    }));
    
    new eks.CfnPodIdentityAssociation(blueprint, `${tenant.namespace}-pod-association`, {
      clusterName: cluster.clusterName,
      namespace: tenant.namespace,
      serviceAccount: tenant.serviceName,
      roleArn: podRole.roleArn,
      targetRoleArn, // Triggers automatic cross-account role assumption in Pod Identity
    });
    ```
    
2.  **In the Workload Account**: The workload stack creates the `PodTargetRole` that trust-binds back to the platform's pod role, locked down using session tags:
    
    ```typescript
    const platformRoleArn = `arn:${this.partition}:iam::${props.platformAccountId}:role/${namespace}-pod`;
    const targetRole = new iam.Role(this, 'PodTargetRole', {
      roleName: `${namespace}-target`,
      assumedBy: new iam.ArnPrincipal(platformRoleArn).withSessionTags().withConditions({
        StringEquals: {
          'aws:RequestTag/kubernetes-namespace': namespace,
          'aws:RequestTag/kubernetes-service-account': 'microservice-a',
        },
      }),
    });
    ```
    

This configuration is highly secure because it ensures only the specific service account in the designated namespace can assume the target workload account role. Crucially, the application code doesn't need to manually call `AssumeRole` in the AWS SDK; the EKS Pod Identity agent intercepts credential requests and presents the assumed target role credentials transparently.

#### C. In-Cluster Tenant Isolation

Because multiple tenants share the same cluster, in-cluster security is paramount. The `TenantBaseline` construct sets up critical guardrails:

*   **Namespaces** with the `restricted` Pod Security Standard set to enforce, audit, and warn.
    
*   **ResourceQuotas** restricting CPU/Memory request footprint to prevent noisy neighbor scenarios.
    
*   **LimitRanges** providing default CPU and memory requests and limits for containers.
    
*   **NetworkPolicies** blocking cross-namespace traffic by default, only allowing egress to DNS and observability endpoints.
    

## Strategy 2: Decentralized EKS Clusters (Dedicated Cluster Model)

In the **Decentralized EKS Cluster** pattern, an EKS cluster is deployed in each workload account alongside that workload's AWS resources. Each business unit or application team can operate its cluster independently and use a reusable platform blueprint appropriate to its needs, such as general-purpose, batch, or AI/ML.

![Decentralized EKS Cluster](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/e9a00bb6-620c-4569-996b-1c89293e732f.png align="center")

### Hard Isolation

This strategy offers the strongest logical isolation. If an EKS cluster is compromised, misconfigured, or experiences a resource exhaustion issue, the impact is strictly confined to that specific workload account.

Because the cluster and AWS resources normally reside in the same workload account, Pod Identity and IRSA can deliver same-account roles directly to pods without cross-account role chaining.

### Key Operational Challenges & GitOps Fleet Management

While decentralized EKS provides maximum security, managing dozens of individual clusters creates severe operational complexity. Every cluster must be provisioned, upgraded, observed, secured, and kept aligned with platform standards. Automation is therefore essential and requires a centralized management model.

To solve this, organizations often deploy a GitOps tool such as **Argo CD** or **Flux** in a central management account. This central hub uses ApplicationSets and directory generators to automatically register and deploy standard add-ons and application manifests to the fleet of workload clusters.

### Optional Centralized Networking

Decentralized clusters do not require every workload account to own a VPC. A central networking account can share subnets through AWS RAM, and workload accounts can deploy their EKS clusters and other resources into those shared subnets. This preserves decentralized cluster ownership while centralizing VPC administration and simplifying connectivity. It also introduces a shared networking dependency that must be reflected in the failure-domain and governance model.

![Centralized Networking for Decentralized EKS Clusters](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/b99eab23-04c0-450a-9d6f-94c9f4a4fe7c.png align="center")

## Cross-VPC Service Connectivity with VPC Lattice

A critical challenge in any multi-account architectural pattern is establishing secure, private connectivity between microservices running inside Kubernetes and services in other AWS accounts or VPCs. VPC Lattice is not required by the EKS multi-account strategy, but it can complement either model when applications need private service-to-service (east-west) connectivity across VPCs and accounts. It provides managed service networks, service discovery, routing, and optional IAM authorization without requiring consumers and producers to manage direct VPC peering routes.

Transit Gateway, VPC peering, and AWS PrivateLink remain valid alternatives that operate at Layer 3/4, but they introduce significant complexity. AWS RAM subnet sharing is ideal for deploying workload databases directly into shared subnets, but it doesn't address service-to-service HTTP/gRPC routing between distinct VPCs.

![Cross-VPC Service Connectivity with VPC Lattice](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/f4d2e3e2-2432-4020-8014-1814320c162b.png align="center")

#### Cross-Account Service Mesh in Decentralized Topologies

In a decentralized strategy where EKS clusters are scattered across multiple workload accounts, connecting service `A` in Cluster `1` to service `B` in Cluster `2` is a primary headache. VPC Lattice acts as an application-level service mesh:

*   You define a **Service Network** (a logical boundary for routing).
    
*   You associate the VPCs of both EKS clusters with that Service Network.
    
*   Services in both clusters are registered to the Service Network with custom DNS domain names.
    
*   Pods in Cluster `1` can invoke services in Cluster `2` by making a standard HTTP request to the DNS name. VPC Lattice handles cross-VPC translation and transit automatically.
    

#### Centralized Ingress Gateway exposing EKS to Workload Accounts

In a centralized cluster topology, you might want to expose internal Kubernetes APIs or ingress gateways to applications running in external workload VPCs (e.g., standard EC2 instances, ECS tasks, or legacy servers in other accounts) without exposing them to the internet.

A `LatticeStack` class can register the EKS ingress (Gateway ALB) to a VPC Lattice Service, allowing other workload accounts to securely call it via IAM authentication:

```typescript
const targetGroup = new vpclattice.CfnTargetGroup(this, 'GatewayAlbTargetGroup', {
  type: 'ALB',
  config: {
    vpcIdentifier: props.vpc.vpcId,
    protocol: 'HTTP',
    protocolVersion: 'HTTP1',
    port: 80,
  },
  targets: [{ id: props.config.gatewayAlbArn, port: 80 }],
});

const service = new vpclattice.CfnService(this, 'GatewayService', {
  name: `${props.config.clusterName}-gateway`,
  authType: 'AWS_IAM', // Enforce IAM Auth
});

// Resource Policy authorizing only designated tenant workload accounts
const allowedAccounts = [...new Set([this.account, ...props.config.tenants.map((tenant) => tenant.workloadAccountId)])];
new vpclattice.CfnResourcePolicy(this, 'GatewayServicePolicy', {
  resourceArn: service.attrArn,
  policy: {
    Version: '2012-10-17',
    Statement: [{
      Effect: 'Allow',
      Principal: { AWS: allowedAccounts.map((account) => `arn:${this.partition}:iam::${account}:root`) },
      Action: 'vpc-lattice-svcs:Invoke',
      Resource: service.attrArn,
    }],
  },
});
```

#### IAM-based Auth Policies (Zero-Trust)

By setting `authType: 'AWS_IAM'` on the Lattice Service and defining a `CfnResourcePolicy`, we achieve a zero-trust topology. When a client pod in a workload account attempts to call the EKS cluster ingress, the VPC Lattice endpoint enforces that the caller must sign their HTTP requests with AWS SigV4 using credentials assumed via EKS Pod Identity / IRSA. Security is verified at the application layer, bypassing coarse IP-based network rule controls.

## Centralized vs. Decentralized: A Comparative Analysis

Choosing between these strategies involves balancing security, operational cost, and development velocity.

| Dimension | Centralized EKS Clusters | Decentralized EKS Clusters |
| --- | --- | --- |
| **Security & Blast Radius** | **Soft Isolation:** Rely on namespaces, NetworkPolicies, and IAM policies. A cluster administrator compromise affects all tenants. | **Hard Isolation:** Complete control plane and infrastructure boundary. Blast radius is restricted to a single account. |
| **Operational Overhead** | **Low:** Fewer clusters to provision, upgrade, and monitor. Add-ons (e.g., ingress controllers, monitoring) are managed once. | **High:** Managing upgrades, certificate renewals, and configuration drift across a fleet of clusters. |
| **AWS API Quota Allocation** | **Shared:** All workloads share the API limits of the Cluster Account, which can lead to throttling at scale. | **Isolated:** Each workload gets its own dedicated account API quotas. |
| **Cost Allocation & Billing** | **Complex:** Demands in-cluster cost-allocation tools (like Kubecost) to map shared node CPU/Memory usage back to tenant cost centers. | **Simple:** All costs (Control Plane fee, EC2 nodes, NAT gateway traffic) are directly billed to the workload account. |
| **Resource Efficiency** | **High:** Compute scaling is consolidated, reducing resource fragmentation and idle capacity waste. | **Low:** Each cluster requires minimum baseline replicas for system pods (CoreDNS, Metrics Server), increasing baseline costs. |
| **Network Connectivity** | **Shared VPC (RAM):** Workloads launch databases into shared VPC subnets. Outbound cross-VPC communication is minimized. | **VPC Lattice / TGW:** Cross-cluster communications utilize VPC Lattice networks or Transit Gateways to manage separate VPC spaces. |

## Decision Framework: Choosing the Right Strategy

### Choose a centralized model when:

1.  Workloads are operated within a common trust and governance boundary.
    
2.  Shared compute and add-ons provide meaningful cost or utilization benefits.
    
3.  The platform team wants to minimize the number of cluster upgrades and operational surfaces.
    
4.  Closely coupled services benefit from simple, low-latency in-cluster connectivity.
    

### Choose a decentralized model when:

1.  Workloads require stronger control-plane and compute isolation.
    
2.  Teams need independent cluster lifecycles, Kubernetes versions, add-ons, or cluster-administrator access.
    
3.  A workload needs dedicated Kubernetes capacity or independent AWS account quotas.
    
4.  Risk or compliance requirements favor a dedicated cluster boundary. A dedicated cluster supports isolation goals but does not by itself establish regulatory compliance.
    

## Conclusion

Centralized and decentralized EKS clusters optimize for different outcomes. Centralized clusters favor utilization and operational simplicity, while decentralized clusters favor stronger isolation, independent scaling, and team autonomy. Many organizations use a hybrid model: shared clusters for workloads with similar trust and lifecycle requirements, and dedicated clusters for high-risk, high-scale, or independently operated workloads.

By leveraging AWS Resource Access Manager (RAM) for cross-account networking, AWS VPC Lattice for application-layer service-to-service routing, and EKS Pod Identity role chaining for cross-account credentials, platforms can seamlessly connect Kubernetes workloads to their isolated AWS environments, achieving a balance of security and developer velocity.

## References

*   [Amazon EKS Best Practices Guide: Multi-Account Strategy](https://docs.aws.amazon.com/eks/latest/best-practices/multi-account-strategy.html)
    
*   [Amazon EKS Best Practices Guide: Tenant Isolation](https://docs.aws.amazon.com/eks/latest/best-practices/tenant-isolation.html)
    
*   [Amazon EKS User Guide: EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
    
*   [Amazon VPC User Guide: Share VPC Subnets](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-sharing.html)
    
*   [Amazon VPC Lattice User Guide: Share VPC Lattice Entities](https://docs.aws.amazon.com/vpc-lattice/latest/ug/sharing.html)
    
*   [Amazon VPC Lattice User Guide: Auth Policies](https://docs.aws.amazon.com/vpc-lattice/latest/ug/auth-policies.html)