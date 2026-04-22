---
title: "Amazon S3 Files for Stateful Containers"
seoTitle: "WordPress on ECS Fargate with Amazon S3 Files and AWS CDK"
seoDescription: "Learn how to run stateful WordPress on AWS ECS Fargate using Amazon S3 Files. A complete guide with AWS CDK for mounting S3 as a filesystem."
datePublished: 2026-04-19T10:55:13.879Z
cuid: cmo5ngerx000f2cp65srg0964
slug: amazon-s3-files-for-stateful-containers
cover: https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/70e948f7-16ac-4fad-b67b-f16378adc1df.png
tags: aws, wordpress, serverless, cloudcomputing, aws-cdk

---

Deploying stateless containers on AWS is a straightforward process, but deploying stateful applications on serverless containers has always been a bit of a "choose your own adventure" when it comes to storage. That's especially true for CMS applications like WordPress and Strapi, or any application that expects a writable shared filesystem. The database is only part of the story; user uploads, plugins, themes, and caches still need somewhere persistent to live.

Historically, that usually pushed you toward Amazon EFS, or toward application-specific workarounds like uploading media directly to S3 through a plugin. Both approaches can work, but they each come with tradeoffs.

Recently, AWS introduced **Amazon S3 Files**, a feature that allows you to mount S3 buckets as local file systems. In this post, we'll dive into a demo project that wires this up using the AWS CDK, using WordPress as an example workload.

> **Tip:** While this demo uses WordPress, the strategy applies to any application requiring persistent, shared storage, from content management systems to data processing pipelines.

## Why this Matters

Containers on Fargate are ephemeral by design. If a task is replaced, anything written to the container's local filesystem disappears. That's fine for APIs and workers that keep all state in external services. It's not fine for platforms that write important data to disk.

WordPress is notoriously stateful. While the database handles your posts and metadata, the filesystem handles:

*   **Media Uploads:** Images and videos stored in `wp-content/uploads`.
    
*   **Customizations:** Plugins and themes installed via the admin dashboard.
    

In a traditional Fargate deployment, these files are ephemeral. If your container restarts or scales out, your data disappears.

The exact same problem shows up in self-hosted headless CMS platforms like Strapi, internal admin tools, ML workflows that generate artifacts, and legacy applications that assume a shared writable directory exists.

## What Amazon S3 Files Changes

[Amazon S3 Files](https://aws.amazon.com/s3/features/files) provides a middle ground between the massive scale of S3 and the POSIX-compliant interface required by apps like WordPress.

The "secret sauce" here is that **S3 Files is built using Amazon EFS**. It acts as an intelligent cache layer that loads your active working set onto high-performance EFS storage for low latency, while the authoritative source of truth remains your S3 bucket.

When you read a file, it's lazily loaded from S3 into this cache. When you write, the data is saved to the high-performance layer and then synchronized back to S3. This means you get the performance of a real filesystem with the durability and ecosystem of S3.

## The Architecture in this Demo

Instead of redesigning the application to speak directly to S3, we mount storage into the container and keep the application model unchanged. The Fargate tasks mount S3 Files at `/bitnami/wordpress`, which is where the Bitnami WordPress image expects its data.

The demo project provisions a robust infrastructure through five CDK stacks:

*   `VPC`: A VPC with public, private, and isolated subnets. It uses **VPC Endpoints** (S3, ECR, Secrets Manager) instead of NAT Gateways to keep service-to-service traffic entirely within the AWS network.
    
*   `Database`: A MariaDB instance managed by RDS, tucked away in isolated subnets with credentials in Secrets Manager.
    
*   `S3Files`: An S3 bucket backed by an S3 Files file system, access point, and mount targets.
    
*   `ECR`: A private ECR repository seeded with the [Bitnami WordPress Image](https://gallery.ecr.aws/bitnami/wordpress).
    
*   `App`: An `ApplicationLoadBalancedFargateService` running two WordPress tasks mounted to `S3Files`.
    

Here is how the stacks and resources connect to each other:

![AWS ECS Fargate Architecture with Amazon S3 Files.](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/66d69f2f-527d-497f-bc14-bba6803d56b4.png align="center")

At the stack dependency level, `App` depends on `VPC`, `Database`, `S3Files`, and `ECR`. At runtime, the WordPress tasks pull the container image from ECR, read credentials from Secrets Manager, connect to MariaDB, and mount the S3 Files volume through the access point and mount targets.

The important part is the storage path:

1.  WordPress runs on ECS Fargate.
    
2.  The task mounts an `S3 Files` volume.
    
3.  That volume maps to an S3-backed file system.
    
4.  The application writes to `/bitnami/wordpress` as if it were local persistent storage.
    

This means uploads, plugins, themes, and similar filesystem state survive task replacement and can be shared across tasks.

> **Complete Source Code:** You can find the full implementation, including all the CDK stacks, in the [GitHub Repository](https://github.com/haZya/s3-files-demo-with-wp).

## Implementing with CDK

Since S3 Files is a relatively new feature, the AWS CDK currently supports it primarily through **L1 constructs** (the low-level `Cfn` resources auto-generated from the CloudFormation schemas).

In our `S3FilesStack`, we define the `CfnFileSystem`, `CfnAccessPoint`, and `CfnMountTarget`. These are then wired into the ECS Task Definition in the `AppStack` using a bit of "escape hatch" patching to configure the `s3FilesVolumeConfiguration`.

The implementation breaks down into four practical steps.

### 1\. Create the Backing Bucket and S3 Files Resources

In `lib/s3-files-stack.ts`, the stack creates a versioned S3 bucket and then wires that bucket into `aws-s3files` resources. We define the File System, the Access Point (defining how users/groups interact with the files):

```ts
// From lib/s3-files-stack.ts
this.s3FilesBucket = new Bucket(this, 'AppFilesBucket', {
  versioned: true,
  blockPublicAccess: BlockPublicAccess.BLOCK_ALL,
  removalPolicy: RemovalPolicy.DESTROY,
  autoDeleteObjects: true,
});

this.s3FilesFileSystem = new CfnS3FileSystem(this, 'S3FileSystem', {
  bucket: this.s3FilesBucket.bucketArn,
  roleArn: s3FilesBucketRole.roleArn,
  acceptBucketWarning: true,
});

this.s3FilesAccessPoint = new CfnS3AccessPoint(this, 'S3FilesAccessPoint', {
  fileSystemId: this.s3FilesFileSystem.attrFileSystemId,
  posixUser: { gid: '1001', uid: '1001' },
  rootDirectory: {
    path: '/wordpress',
    creationPermissions: {
      ownerGid: '1001',
      ownerUid: '1001',
      permissions: '755',
    },
  },
});
```

This is the heart of the pattern:

*   The bucket is the durable storage layer.
    
*   `CfnS3FileSystem` exposes that storage as an S3 Files file system.
    
*   `CfnS3AccessPoint` defines how the application will enter that file tree.
    

The access point is configured with UID/GID `1001` and a root directory of `/wordpress`, which lines up with the filesystem expectations of the Bitnami WordPress container.

### 2\. Create Mount Targets Inside the VPC

The file system still needs network reachability from the ECS tasks. This creates mount targets in the `PRIVATE_WITH_EGRESS` subnets and opens NFS traffic on port `2049` from within the VPC:

```ts
// From lib/s3-files-stack.ts
const mountTargetSecurityGroup = new SecurityGroup(this, 'S3FilesMountTargetSg', {
  description: 'S3 Files mount target SG',
  vpc,
});

mountTargetSecurityGroup.addIngressRule(
  Peer.ipv4(vpc.vpcCidrBlock),
  Port.tcp(2049),
  'Allow NFS from VPC',
);

const mountTargetSubnets = vpc
  .selectSubnets({ subnetType: SubnetType.PRIVATE_WITH_EGRESS, onePerAz: true })
  .subnets;

this.s3FilesMountTargets = mountTargetSubnets.map((subnet, index) => {
  return new CfnS3MountTarget(this, `S3FilesMountTarget${index + 1}`, {
    fileSystemId: this.s3FilesFileSystem.attrFileSystemId,
    subnetId: subnet.subnetId,
    securityGroups: [mountTargetSecurityGroup.securityGroupId],
  });
});
```

That is an important detail because this is not just an S3 bucket reference in ECS. The workload needs the network path to the mounted file system, so the mount target configuration matters.

### 3\. Give the Service and Tasks the Right Permissions (IAM)

There are two sides to the permission model here:

First, `S3 Files` itself needs a specific service-linked role that lets the service synchronize data between the file system and the S3 bucket. In `lib/s3-files-stack.ts`, the role is assumed by `elasticfilesystem.amazonaws.com` and gets bucket, object, and EventBridge permissions:

```typescript
// From lib/s3-files-stack.ts
const s3FilesBucketRole = new Role(this, 'S3FilesBucketRole', {
  assumedBy: new ServicePrincipal('elasticfilesystem.amazonaws.com', {
    conditions: {
      StringEquals: { 'aws:SourceAccount': this.account },
      ArnLike: { 'aws:SourceArn': `arn:aws:s3files:${this.region}:${this.account}:file-system/*` },
    },
  }),
});

s3FilesBucketRole.addToPolicy(new PolicyStatement({
  actions: ['s3:ListBucket', 's3:GetObject*', 's3:PutObject*', 's3:DeleteObject*'],
  resources: [bucket.bucketArn, `${bucket.bucketArn}/*`],
}));
```

Second, the ECS task role needs permission to actually use the mounted file system. In `lib/app-stack.ts`, the task gets the AWS managed policy `AmazonS3FilesClientReadWriteAccess` plus explicit S3 read/list access to the bucket:

```ts
// From lib/app-stack.ts
fargateService.taskDefinition.taskRole.addManagedPolicy(
  ManagedPolicy.fromAwsManagedPolicyName('AmazonS3FilesClientReadWriteAccess'),
);

fargateService.taskDefinition.taskRole.addToPrincipalPolicy(new PolicyStatement({
  sid: 'S3ObjectReadAccess',
  actions: ['s3:GetObject', 's3:GetObjectVersion'],
  resources: [`${bucketArn}/*`],
}));

fargateService.taskDefinition.taskRole.addToPrincipalPolicy(new PolicyStatement({
  sid: 'S3BucketListAccess',
  actions: ['s3:ListBucket'],
  resources: [bucketArn],
}));
```

This split is easy to miss when you first look at the feature. You need permissions for the service-side bucket integration and for the task-side runtime access.

### 4\. Patch the ECS Task Definition with `s3FilesVolumeConfiguration`

The ECS service itself is created with the familiar `ApplicationLoadBalancedFargateService` construct:

```ts
// From lib/app-stack.ts
const fargateService = new ApplicationLoadBalancedFargateService(this, 'AppService', {
  serviceName: 'WordpressDemoService',
  cluster,
  cpu: 256,
  memoryLimitMiB: 512,
  desiredCount: 2,
  publicLoadBalancer: true,
  taskImageOptions: {
    containerName: 'WordpressDemoContainer',
    family: 'WordpressDemoTask',
    image: ContainerImage.fromEcrRepository(ecrRepo, 'latest'),
    containerPort: 8080,
  },
  minHealthyPercent: 100,
});
```

Then the default container gets a mount point:

```ts
// From lib/app-stack.ts
const volumeName = 's3files';
const mountPath = '/bitnami/wordpress';  // App's data path

fargateService.taskDefinition.defaultContainer?.addMountPoints({
  containerPath: mountPath,
  sourceVolume: volumeName,
  readOnly: false,
});
```

And finally, since high-level L2 construct support is still evolving for this feature, we use a CDK **Escape Hatch**. This allows us to "drop down" to the underlying CloudFormation resource (`CfnTaskDefinition`) and manually configure the `s3FilesVolumeConfiguration` property:

```ts
// From lib/app-stack.ts
const cfnTaskDefinition = fargateService.taskDefinition.node.defaultChild as CfnTaskDefinition;
const existingVolumes = Array.isArray(cfnTaskDefinition.volumes)
  ? cfnTaskDefinition.volumes
  : [];

cfnTaskDefinition.volumes = [
  ...existingVolumes,
  {
    name: volumeName,
    s3FilesVolumeConfiguration: {
      accessPointArn: s3FilesAccessPoint.attrAccessPointArn,
      fileSystemArn: s3FilesFileSystem.attrFileSystemArn,
      rootDirectory: '/',
    },
  },
];
```

> **Note:** Using Escape Hatches is a standard practice in CDK when you need to use a new AWS feature before the high-level constructs have been updated to support it.

## End-to-End Request Flow

Once deployed, the runtime model is straightforward:

![End-to-end request flow for WordPress App on AWS ECS Fargate with Amazon S3 Files.](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/532a58b8-70d6-47c3-b235-4170113aefd5.png align="center")

1.  A request hits the Application Load Balancer.
    
2.  One of the Fargate WordPress tasks handles it.
    
3.  WordPress reads or writes persistent content under `/bitnami/wordpress`.
    
4.  That path is backed by the S3 Files volume.
    
5.  The durable backing store for that content is the S3 bucket from the `S3Files` stack.
    

That means if a task is replaced or if the service scales horizontally, the content directory is still shared and persistent.

## Why the Mount Path Matters

One subtle but important detail in this demo is that the S3 Files volume is mounted at `/bitnami/wordpress`, not at some arbitrary side directory.

That is what keeps the application model simple. The container image already expects its writable application data there, so the infrastructure is adapting to the app rather than forcing the app to adapt to the infrastructure.

That is a big part of why this same pattern can apply to other stateful applications. For example, in the case of Strapi, the mount path would be `/public/uploads`. If you can identify the directory that the application treats as its durable shared state, you can often mount `S3 Files` there and avoid more invasive application changes.

## Why this is Attractive for Stateful Applications

What I like about this approach is that it keeps the application architecture simple.

You do not need to teach WordPress how to store media in S3. You do not need to redesign the app around object storage APIs. And you do not need to split "database state" from "filesystem state" in an application-specific way just to make containers viable.

Instead, you can preserve the existing runtime assumptions:

*   The app writes files.
    
*   Multiple tasks can share those files.
    
*   The data survives task churn.
    
*   The durable backing store is S3.
    

That last point is especially appealing because S3 has operational advantages people already know how to use: lifecycle policies, replication strategies, inventory, access controls, and long-term storage economics.

> **Practical Note:** Because S3 Files uses an EFS-backed cache, writes are available to other tasks almost instantly. However, the background synchronization to the S3 bucket itself can take 30 to 60 seconds. If you're checking the S3 console for your files immediately after an upload, don't panic if they don't show up right away!

## How Does It Compare?

When deciding how to handle WordPress storage, you generally have three paths:

### 1\. Amazon S3 Files (The New Way)

*   **Pros:** Near-infinite storage of S3, **up to 90% lower cost compared to EFS**, and easier data management (it's just a bucket!).
    
*   **Cons:** Currently requires lower-level configuration (L1 constructs) in CDK.
    

### 2\. Amazon EFS (The Traditional Way)

*   **Pros:** Mature, fully POSIX-compliant, and has excellent L2 construct support in CDK.
    
*   **Cons:** More expensive than S3 for large amounts of "cold" media files, and managing EFS lifecycle policies can be more complex than S3.
    

### 3\. "Offload Media" Plugins

*   **Pros:** No complex infrastructure needed; plugins like *WP Offload Media* sync your `uploads` folder directly to S3 and rewrite URLs.
    
*   **Cons:** Usually only handles media. Plugins and themes still need a persistent home, and these plugins often have a premium cost or require extra configuration within WordPress itself.
    

### The "S3 Files" Advantage

S3 Files is particularly powerful because it treats your bucket as the source of truth. You can use standard S3 features like **Lifecycle Policies**, **Replication**, and **Inventory** while your application thinks it's just writing to a local disk.

This makes it an ideal fit for:

*   **Legacy Migrations:** Apps that expect a filesystem, but you want to store data in S3.
    
*   **Shared Assets:** Multiple containers needing access to a common set of images, logs, or configurations.
    
*   **Cost Management:** Leveraging S3's low-cost storage classes for large datasets.
    

## The Future

Using L1 constructs today feels a bit like "coding close to the metal," but it gives us access to this powerful feature right now. As the feature matures, we can expect the AWS CDK team to release **L2 constructs** that will make this integration much simpler.

## Taking it to Production

While this demo gives you a solid foundation, there are a few "Day 2" enhancements you'll want to consider before going live:

*   **CloudFront & WAF:** Place a CloudFront distribution in front of your ALB to cache static assets (like images from S3) at edge locations. This reduces the load on your Fargate tasks and saves you money on data transfer. Don't forget to attach **AWS WAF** to block SQL injection and cross-site scripting (XSS) attacks.
    
*   **Database High Availability:** In this demo, we use a single MariaDB instance. For production, you should enable **Multi-AZ** for RDS to ensure your database can failover automatically if an Availability Zone goes down.
    
*   **Backup & Recovery:** While S3 is highly durable, you should still use **AWS Backup** or S3 Versioning to protect against accidental deletions or application-level corruption.
    
*   **Auto-scaling:** Configure your Fargate service to scale the number of tasks automatically based on CPU or memory usage. This ensures your site stays responsive during traffic spikes without over-provisioning.
    
*   **Monitoring:** Set up **CloudWatch Alarms** for your ALB's 5XX errors and RDS CPU utilization, so you're the first to know if something goes wrong.
    

## Conclusion

Amazon S3 Files represents a significant step forward in simplifying stateful container architecture. By combining the infinite scale of S3 with the accessibility of a file system, it provides a flexible, cost-effective solution for any stateful workload on AWS, including Lambda, EC2, ECS, EKS, Fargate, and Batch.

If you are working with a CMS, an internal platform, or any application that expects shared writable files, this is one of the most promising new AWS features to experiment with.

Check out the full source code for this demo project [here](https://github.com/haZya/s3-files-demo-with-wp) and start modernizing your stateful apps today!