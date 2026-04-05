---
title: "Deep Dive: Building a Secure, Event-Driven File Processing Pipeline with AWS CDK"
seoDescription: "Learn how to build a secure, event-driven file processing pipeline with AWS CDK using S3 staging buckets, GuardDuty malware scanning & Step Functions"
datePublished: 2026-04-03T14:10:59.282Z
cuid: cmnizeix9011f1qouhh4ghqa7
slug: deep-dive-building-a-secure-event-driven-file-processing-pipeline-with-aws-cdk
cover: https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/752f7f7b-7006-446e-8457-c0e571ac9c21.png
tags: microservices, aws-cdk, cloud-architecture, stepfunction, event-driven-architecture

---

When people say "file upload," they often mean a simple `PUT` to S3 and a database row. In a hobby project, that's fine. In production, it’s a liability.

Production-grade file processing requires answering a much harder set of questions:

*   How do I isolate untrusted files from my clean assets?
    
*   Where does malware scanning fit without blocking the user?
    
*   How do I validate file types beyond the easily spoofed browser MIME type?
    
*   How do I fan out status updates to users and other services?
    
*   How do I replace old files safely without leaking orphaned objects?
    
*   How do I keep the whole system event-driven instead of building a tightly coupled "upload monolith"?
    

In this article, we’ll walk through a self-contained file-processing microservice built with **AWS CDK and TypeScript**. We leverage S3, EventBridge, GuardDuty Malware Protection, Step Functions, and API Gateway WebSockets to build a pipeline that handles everything from ingestion to real-time status delivery.

> **Complete Source Code:** You can find the full implementation, including all Lambda handlers and CDK stacks, in the [GitHub Repository](https://github.com/haZya/upload-file-processing-pipeline).

## What We Are Building

At a high level, the service follows a strict security-first workflow:

1.  **Ingestion:** A client requests a presigned POST and uploads a file to a **Staging Bucket**.
    
2.  **Registration:** An S3 event triggers a Lambda to record the upload as `PENDING_SCAN` in DynamoDB.
    
3.  **Security Gate:** **AWS GuardDuty Malware Protection** scans the object asynchronously.
    
4.  **Orchestration:** A GuardDuty scan result event triggers an **AWS Step Functions Express Workflow**.
    
5.  **Processing:** The workflow validates the file signature, moves it to the **Clean Bucket**, transforms images (using `sharp`), updates metadata, and cleans up old files.
    
6.  **Real-time Notify:** Status changes are fanned out via **EventBridge** to a **WebSocket API** for immediate client feedback.
    

This design ensures a clean separation between ingestion, security, orchestration, and real-time communication.

![Step Functions Graph.](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/a3dfdac1-f4ec-42d2-8e5b-6467a335fd96.png align="center")

## Architecture Overview

The repository is organized into five modular CDK stacks, keeping domain concerns isolated:

*   `lib/storage-stack.ts` - Private S3 buckets + CloudFront distribution for serving assets.
    
*   `lib/database-stack.ts` - DynamoDB tables for uploads and relation tracking.
    
*   `lib/guard-duty-stack.ts` - GuardDuty Malware Protection for the staging bucket.
    
*   `lib/upload-processing-stack.ts` - The orchestration core: EventBridge, Step Functions, and Lambdas.
    
*   `lib/websocket-stack.ts` - API Gateway WebSocket API and connection tracking.
    

The app wiring in `bin/app.ts` is intentionally simple, demonstrating the power of stack composition:

```ts
const storageStack = new StorageStack(app, "Storage");
const databaseStack = new DatabaseStack(app, "Database");
const guardDutyStack = new GuardDutyStack(app, "GuardDuty", {
  stagingUploadBucket: storageStack.stagingUploadBucket,
});
const webSocketStack = new WebSocketStack(app, "WebSocket", {});

const uploadProcessingStack = new UploadProcessingStack(app, "UploadProcessing", {
  stagingUploadBucket: storageStack.stagingUploadBucket,
  uploadBucket: storageStack.uploadBucket,
  uploadsTable: databaseStack.uploadsTable,
  uploadRelationsTable: databaseStack.uploadRelationsTable,
  webSocket: webSocketStack,
});
```

## Why the Dual-bucket Strategy Matters

The first important design choice is the storage layer. Instead of uploading directly into your final bucket, this service uses two buckets: a **staging bucket** for untrusted files and an **upload bucket** for processed, clean assets. This "DMZ" approach prevents your final storage from ever becoming a dumping ground for unscanned content.

Here is the essence of `lib/storage-stack.ts`:

```ts
this.stagingUploadBucket = new Bucket(this, "StagingUploadBucket", {
  blockPublicAccess: BlockPublicAccess.BLOCK_ALL,
  enforceSSL: true,
  minimumTLSVersion: 1.2,
  eventBridgeEnabled: true,
  removalPolicy: RemovalPolicy.DESTROY,
  autoDeleteObjects: true,
  lifecycleRules: [
    {
      abortIncompleteMultipartUploadAfter: Duration.days(1),
      expiration: Duration.days(7),
    },
  ],
});

this.uploadBucket = new Bucket(this, "UploadBucket", {
  blockPublicAccess: BlockPublicAccess.BLOCK_ALL,
  enforceSSL: true,
  minimumTLSVersion: 1.2,
});

const cachePolicy = new CachePolicy(this, "UploadBucketCachePolicy", {
    defaultTtl: Duration.days(7),
    minTtl: Duration.seconds(0),
    maxTtl: Duration.days(30),
});

new Distribution(this, "UploadBucketDistribution", {
  defaultBehavior: {
    origin: S3BucketOrigin.withOriginAccessControl(this.uploadBucket),
    viewerProtocolPolicy: ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    cachePolicy,
  },
  minimumProtocolVersion: SecurityPolicyProtocol.TLS_V1_3_2025,
});
```

There are a few good ideas packed into this:

*   The staging bucket is private and event-enabled.
    
*   Incomplete multipart uploads are cleaned up automatically.
    
*   The clean upload bucket is also private.
    
*   CloudFront uses Origin Access Control, so S3 stays off the public internet path.
    

> **Pro Tip:** While this setup enforces TLS and private access, you should consider adding `BucketEncryption.KMS` for defense-in-depth and granular auditing in production environments.

A production-oriented version with a KMS key could look more like this:

```ts
const key = new kms.Key(this, "UploadsKey", {
  enableKeyRotation: true,
});

const uploadBucket = new Bucket(this, "UploadBucket", {
  encryption: BucketEncryption.KMS,
  encryptionKey: key,
  bucketKeyEnabled: true,
  // ...Other options
});
```

## Modeling Uploads in DynamoDB

The database layer splits responsibilities across two tables to handle different access patterns:

*   `UploadsTable` stores every upload event (the historical record).
    
*   `UploadRelationsTable` tracks the current file for a specific business entity (e.g., "user:123:avatar").
    

`lib/database-stack.ts` defines two useful GSIs on `UploadsTable`:

*   `ByRelation` for querying uploads by logical owner or relation.
    
*   `ByStagingKey` for resolving the latest record tied to a staging object key.
    

```ts
this.uploadsTable = new TableV2(this, "UploadsTable", {
  partitionKey: { name: "uploadId", type: AttributeType.STRING },
  pointInTimeRecoverySpecification: {
    pointInTimeRecoveryEnabled: true,
  },
  globalSecondaryIndexes: [
    {
      indexName: "ByRelation",
      partitionKey: { name: "relationKey", type: AttributeType.STRING },
      sortKey: { name: "createdAt", type: AttributeType.STRING },
    },
    {
      indexName: "ByStagingKey",
      partitionKey: { name: "stagingKey", type: AttributeType.STRING },
      sortKey: { name: "createdAt", type: AttributeType.STRING },
    },
  ],
});

this.uploadRelationsTable = new TableV2(this, "UploadRelationsTable", {
    partitionKey: { name: "relationKey", type: AttributeType.STRING },
    removalPolicy: RemovalPolicy.DESTROY,
    pointInTimeRecoverySpecification: {
        pointInTimeRecoveryEnabled: true,
    },
});
```

This separation allows you to query "what happened to upload X" while simultaneously allowing the UI to instantly find the "current hero image" for a product.

The `lambda/upload/update-status.ts` handler uses that second table to atomically move a relation to the newest successful upload while keeping a reference to the previous one for cleanup.

## Ingestion: Generating the Presigned Upload

The upload path starts in `lambda/upload/generate-presigned-post.ts`. A key detail here is binding metadata directly into the presigned POST conditions. By forcing `x-amz-meta-author-id` and `x-amz-meta-relation-key` into the upload itself, we ensure downstream processors never have to "guess" the context.

```ts
const { url, fields } = await createPresignedPost(s3, {
  Bucket: stagingBucket,
  Key: key,
  Conditions: [
    ["content-length-range", 0, 100 * 1024 * 1024],
    ["starts-with", "$Content-Type", contentType ?? ""],
    ["eq", "$x-amz-meta-relation-key", relationKey],
    ["eq", "$x-amz-meta-author-id", userId],
  ],
  Fields: {
    ...(contentType ? { "Content-Type": contentType } : {}),
    "x-amz-meta-relation-key": relationKey,
    "x-amz-meta-author-id": userId,
  },
  Expires: 600,
});
```

## Security First: GuardDuty Malware Protection for S3

We use **GuardDuty Malware Protection for S3** as our security gate. It scans objects asynchronously and emits EventBridge events, which is far more efficient and scalable than writing custom antivirus logic in Lambda.

From `lib/guard-duty-stack.ts`:

```ts
const plan = new CfnMalwareProtectionPlan(this, "S3MalwareProtectionPlan", {
  role: role.roleArn,
  protectedResource: {
    s3Bucket: {
      bucketName: stagingUploadBucket.bucketName,
    },
  },
  actions: {
    tagging: { status: "ENABLED" },
  },
});
```

The stack also grants the GuardDuty service role the minimum S3, EventBridge, and optional KMS permissions it needs.

Why this design works well:

*   S3 remains the system of record for raw files.
    
*   GuardDuty performs the malware check asynchronously.
    
*   The result shows up as an EventBridge event.
    
*   Object tagging can record scan outcomes such as `NO_THREATS_FOUND` and `THREATS_FOUND`.
    

## EventBridge as the Backbone

The service uses EventBridge in two different ways.

First, AWS service events kick off work:

*   S3 `Object Created` events register the upload as pending.
    
*   GuardDuty malware scan result events start the Step Functions workflow.
    

Second, the service emits its own internal domain events on a dedicated bus: `UploadStatusChanged`.

That gives the system a very clean shape: inbound events drive orchestration, and outbound events describe state changes for consumers.

```ts
new Rule(this, "S3ObjectCreatedRule", {
  eventPattern: {
    source: ["aws.s3"],
    detailType: ["Object Created"],
    detail: {
      bucket: { name: [stagingUploadBucket.bucketName] },
    },
  },
  targets: [
    new LambdaFunction(registerUploadLambda, {
      event: RuleTargetInput.fromObject({
        bucket: EventField.fromPath("$.detail.bucket.name"),
        key: EventField.fromPath("$.detail.object.key"),
      }),
      retryAttempts: 4,
      deadLetterQueue: s3ObjectCreatedDeliveryDlq,
    }),
  ],
});
```

The `lambda/upload/register-upload.ts` handler calls `HeadObject`, reads metadata, and creates a DynamoDB row with a `PENDING_SCAN` status.

## Orchestration: Step Functions Express Workflow

This is the "brain" of the system. The actual architectural center of the repo, which manages the complex branching logic of the pipeline:

**Security Check:** If GuardDuty finds a threat, delete the file and fail. **Deep Validation:** Use file-type to inspect the "magic numbers" of the file, ignoring spoofed MIME headers. **Parallel Processing:** Transform images (using `sharp`) and delete the staging file simultaneously.

The workflow is triggered by GuardDuty scan results. If the object is clean, the pipeline validates the file, resolves the final key, copies the object to the clean bucket, optionally generates image variants, stores metadata, updates the upload record, emits a status event, and cleans up a previously replaced upload if needed.

If the object is malicious or invalid, it deletes the staging object and marks the upload as failed.

### Why Express Workflow?

We use Express Workflows with `StateMachineType.EXPRESS` for this orchestration because file processing is a high-volume, short-lived task where latency and cost-efficiency are paramount. Having said that, express workflows do not support [long-running callback patterns](https://docs.aws.amazon.com/step-functions/latest/dg/connect-to-resource.html) like `.waitForTaskToken` or `.sync`. If you need to wait for external human approval (HITL) or a long asynchronous job, switch to Standard.

### The Key Tasks in the Workflow

`lib/upload-processing-stack.ts` wires several Lambdas into the state machine:

*   `validate.ts` - checks file signature using `file-type`.
    
*   `resolve-final-key.ts` - generates the durable object key.
    
*   `transform-image.ts` - creates WebP derivatives with `sharp`.
    
*   `add-metadata.ts` - persists final key, dimensions, and formats.
    
*   `update-status.ts` - marks the upload successful or failed and updates the relation state.
    
*   `cleanup-replaced-upload.ts` - deletes the previous file version if the relation now points elsewhere.
    

It also uses direct AWS SDK integrations for S3 copy and delete operations, which keeps the workflow explicit and avoids extra Lambda glue.

### The Workflow Definition

This is the part many teams hand-wave. This demo does not. The state machine is modeled directly in CDK:

```ts
const coreWorkflow = new Choice(this, "ScanResultOK")
  .when(
    Condition.stringEquals("$.scanResultStatus", "NO_THREATS_FOUND"),
    validateFileTask.next(
      new Choice(this, "IsFileValid")
        .when(
          Condition.booleanEquals("$.isValid", true),
          resolveFinalKeyTask
            .next(copyToUploadBucketTask)
            .next(transformAndDelete),
        )
        .otherwise(deleteInvalidStagingObjectTask.next(markValidationFailedTask)),
    ),
  )
  .otherwise(deleteThreatStagingObjectTask.next(markThreatDetectedTask));

const definition = new Parallel(this, "MainWorkflowGroup", {
  outputPath: "$.[0]",
})
  .branch(coreWorkflow)
  .addCatch(
    updateUploadStatusFailureTask
      .next(uploadStatusEmitFailureOnCatch)
      .next(workflowFailState),
    {
      errors: [Errors.ALL],
      resultPath: "$.error",
    },
  )
  .next(postProcessingFlow);

const stateMachine = new StateMachine(this, "FileUploadStateMachine", {
  definitionBody: DefinitionBody.fromChainable(definition),
  timeout: Duration.minutes(5),
  stateMachineType: StateMachineType.EXPRESS,
  logs: {
    destination: logGroup,
    level: LogLevel.ALL,
    includeExecutionData: true,
  },
});
```

There are a few design choices here worth highlighting.

First, the workflow branches early on the security outcome. That means a malware-positive file never reaches the rest of the processing path.

Second, image transformation and staging-object deletion run inside a `Parallel` state:

```ts
const transformAndDelete = new Parallel(this, "TransformAndDeleteStagingFile", {
  outputPath: "$.[0]",
});

transformAndDelete.branch(
  new Choice(this, "IsImage")
    .when(Condition.stringMatches("$.mime", "image/*"), transformImageTask)
    .otherwise(new Pass(this, "SkipTransformForNonImage"))
    .afterwards()
    .next(addMetadataTask),
);

transformAndDelete.branch(deleteStagingObjectOnCopySuccessTask);
```

That is a nice example of Step Functions as a real orchestration engine, not just a fancy switch statement.

Third, the stack centralizes retry policies for Lambda, S3, and EventBridge tasks:

```ts
private addServiceRetry(task: CallAwsService | LambdaInvoke | EventBridgePutEvents, errors: string[]) {
  task.addRetry({
    errors,
    interval: Duration.seconds(2),
    backoffRate: 2,
    maxAttempts: 3,
  });
}
```

That small helper keeps resilience consistent across the workflow.

## File Validation Beyond the Content-Type Header

One of the easiest upload mistakes is trusting the browser-provided MIME type. This demo does better. The `lambda/upload/validate.ts` reads the first few kilobytes of the object and uses `file-type` to inspect the file signature:

```ts
const object = await s3.send(new GetObjectCommand({
  Bucket: bucket,
  Key: key,
  Range: "bytes=0-4095",
}));

const buffer = await object.Body.transformToByteArray();
const detected = await fileTypeFromBuffer(buffer);
const detectedMime = detected?.mime ?? null;
const mime = detectedMime ?? head.ContentType ?? "application/octet-stream";
const isValid = !(detectedMime && head.ContentType && detectedMime !== head.ContentType);
```

That is still lightweight, but it already closes a very common trust gap. If the content is invalid, the workflow deletes the staging object and marks the upload as `VALIDATION_FAILED`.

## Image Transformation and Derivative Generation

If the file is an image, the workflow creates optimized WebP variants with `sharp`.

From `lambda/upload/transform-image.ts`:

```ts
async function createVariant(width: number, prefix: string) {
  const transformed = await sharp(buffer)
    .resize({ width, withoutEnlargement: true })
    .webp({ quality: 75 })
    .toBuffer({ resolveWithObject: true });

  const parts = event.finalKey.split("/");
  const fileName = parts.pop()!;
  const newKey = [...parts, `${prefix}-${fileName}`].join("/");

  await s3.send(new PutObjectCommand({
    Bucket: uploadBucket,
    Key: newKey,
    Body: transformed.data,
    ContentType: "image/webp",
  }));

  return {
    key: newKey,
    width: transformed.info.width,
    height: transformed.info.height,
    size: transformed.info.size,
    mime: "image/webp",
  };
}
```

The handler returns the original dimensions plus a `formats` array, and `add-metadata.ts` persists that back into DynamoDB. That means the final upload record is not just "file uploaded"; it is an asset descriptor that a frontend can actually use.

### The `sharp` Challenge

Bundling native modules like `sharp` for Lambda can be tricky. We use `NodejsFunction` with Docker-based bundling to ensure the binary is compiled for the Lambda Linux environment:

```ts
bundling: {
  nodeModules: ["sharp"],
  forceDockerBundling: true,
  environment: {
    NPM_CONFIG_BIN_LINKS: "false",
  },
}
```

> **Note:** If you're on Windows, you may have to set `NPM_CONFIG_BIN_LINKS` environment variable to `false` to disable symlink creation.

## Relation-aware Status Tracking and Atomic Replacement

One of the more sophisticated patterns in this architecture is the decoupling of **Uploads** from **Relations**.

*   **An Upload** is an immutable historical record. It tracks the journey of a specific file from staging to final storage, including its metadata and processing results.
    
*   **A Relation** is a logical pointer within your application domain (e.g., `user:42:avatar` or `product:99:hero-image`) that resolves to a specific, successful upload.
    

### Why This Decoupling Matters

In many systems, updating a file means overwriting the existing object or manually deleting the old one before uploading the new one. This often leads to race conditions, orphaned files, or broken UI states.

By using a dedicated `UploadRelationsTable`, we implement a **fail-safe replacement strategy**:

1.  **Late Binding:** The application relation only updates *after* the entire processing pipeline succeeds.
    
2.  **Reference Tracking:** When `lambda/upload/update-status.ts` marks a new upload as complete, it updates the relation to point to the new asset while capturing the `previousUploadId`.
    
3.  **Automatic Cleanup:** This hand-off allows the `lambda/upload/cleanup-replaced-upload.ts` task to safely delete the old file and all its generated variants (WebP, thumbnails, etc.) without impacting the live application.
    

This approach ensures that your application always points to a valid, scanned asset, and your storage never becomes a graveyard of abandoned "Version 1" files. It provides the benefits of object versioning without the complexity of S3 Bucket Versioning for your application logic.

## Real-time Communication with WebSockets

The last mile is status delivery. Because the pipeline is asynchronous, the user needs immediate feedback. The service emits `UploadStatusChanged` events to an internal EventBridge bus. The `lambda/upload/emit-upload-status.ts` subscriber Lambda then looks up active connections in DynamoDB and pushes the update via **API Gateway WebSockets**.

```ts
private createUploadStatusEmitTask(id: string, uploadStatusBus: EventBus) {
  return new EventBridgePutEvents(this, id, {
    entries: [
      {
        eventBus: uploadStatusBus,
        detailType: "UploadStatusChanged",
        detail: TaskInput.fromJsonPathAt("$"),
        source: "com.file-processing.upload",
      },
    ],
    resultPath: "$.eventBridgeResult",
  });
}
```

The WebSocket stack stores connections in DynamoDB with a TTL so stale sessions age out automatically.

From `lib/websocket-stack.ts`:

```ts
const connectionsTable = new TableV2(this, "WebSocketConnectionsTable", {
  partitionKey: { name: "PK", type: AttributeType.STRING },
  sortKey: { name: "SK", type: AttributeType.STRING },
  timeToLiveAttribute: "ttl",
  pointInTimeRecoverySpecification: {
    pointInTimeRecoveryEnabled: true,
  },
  dynamoStream: StreamViewType.NEW_AND_OLD_IMAGES,
});
```

And the connection handler refreshes TTL on inbound messages to keep live sockets active.

> **Note:** The `lambda/websocket/authorizer.ts` in the demo is explicitly mocked to return a hard-coded demo subject after "verification". For a real deployment, replace that with actual JWT verification against Cognito or your identity provider.

## End-to-End Flow

Here is the mental model I would use when explaining the pipeline:

```text
Client
  -> Request presigned POST
  -> Upload file to staging bucket

S3 Object Created
  -> EventBridge rule
  -> Register-upload Lambda
  -> DynamoDB status = PENDING_SCAN

GuardDuty scans staging object
  -> EventBridge scan result
  -> Step Functions Express workflow

If clean
  -> Validate file signature
  -> Resolve final key
  -> Copy to clean bucket
  -> Optionally transform image
  -> Save metadata
  -> Mark upload complete
  -> Clean previous version
  -> Emit UploadStatusChanged

If invalid or malicious
  -> Delete staging object
  -> Mark failed status
  -> Emit UploadStatusChanged

Internal event bus
  -> WebSocket notifier Lambda
  -> Connected clients receive status update
```

### Step Functions Graph for a Successful File Upload.

![Step Functions success graph.](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/9fd624b7-8f1a-4058-81f0-9a3703420688.png align="center")

### WebSocket Notification for a Successful File Upload

![WebSocket notification for success.](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/777da2fb-6f3e-4545-8a1f-166b510b7203.png align="center")

## Architectural Benefits

The strength of this architecture lies in its strict adherence to the **Single Responsibility Principle (SRP)** at the infrastructure level. Rather than building a monolithic "Upload Lambda," we've distributed concerns across managed services that are natively optimized for their respective tasks.

*   **Decoupled Storage (S3):** S3 is used strictly for what it does best—durable, scalable object storage—rather than being cluttered with orchestration logic.
    
*   **Security as a Service (GuardDuty):** By offloading malware scanning to GuardDuty, we eliminate the operational burden and scaling limits of maintaining custom antivirus signatures in Lambda.
    
*   **Explicit Orchestration (Step Functions):** Complex branching, retries, and parallel execution are managed in a visual state machine, making the workflow observable and easy to modify without touching code.
    
*   **State vs. Event (DynamoDB & EventBridge):** DynamoDB maintains the source of truth for asset metadata, while EventBridge acts as the nervous system, routing state changes to downstream consumers without tight coupling.
    
*   **Asynchronous Communication (WebSockets):** Real-time feedback is treated as a reactive side-effect of domain events, not a synchronous dependency of the processing pipeline.
    

This modularity ensures that the service is not just easy to build, but easy to evolve. You can swap the image processor, add new security gates, or change how notifications are delivered without ever disrupting the core ingestion path.

## Scaling to Production: Roadmap and Optimizations

While this architecture provides a robust foundation, transitioning to a high-scale production environment often requires additional optimizations for efficiency, cost, and observability.

### 1\. Extending the Processing Pipeline

The Step Functions backbone makes it trivial to insert new processing stages. As your application grows, you can easily integrate:

*   **AI/ML Insights:** Add Amazon Rekognition for moderation or Textract for document analysis.
    
*   **Frontend Optimization:** Generate **Blurhash** values for instant placeholders or low-resolution image previews.
    
*   **Rich Metadata:** Extract EXIF data, strip sensitive location tags, or generate multi-format derivatives (AVIF, WebP, PDF).
    
*   **Compliance:** Implement automated PII (Personally Identifiable Information) detection before files reach the clean bucket.
    

### 2\. Implementing Content De-duplication

To optimize storage costs and minimize redundant processing (like malware scanning or image transformation), implement **content-addressable storage**:

1.  **Client-side Hashing:** Compute a SHA-256 digest on the client before upload.
    
2.  **Dedupe Check:** Query DynamoDB to see if the digest already exists in the `Clean Bucket`.
    
3.  **Logical Mapping:** If a match is found, create a new `Relation` record pointing to the existing `finalKey` instead of initiating a new upload.
    

### 3\. Proactive Observability and Alarming

Relying on DLQs is the first step, but production systems require active monitoring. Implement CloudWatch Alarms for:

*   **Pipeline Failures:** Monitor Step Functions `ExecutionsFailed` and `ExecutionsTimedOut`.
    
*   **Infrastructure Health:** Track Lambda `Errors` and `Throttles`, specifically for the image processing and metadata tasks.
    
*   **Queue Backlog:** Alarm when DLQ message counts exceed zero to ensure human intervention for unhandled edge cases.
    

### 4\. Explicit Business-level Retries

Beyond infrastructure retries, your workflow should handle transient business failures (e.g., a third-party moderation API being temporarily unavailable). Use Step Functions' `Retry` logic with custom error codes like `TransientServiceError` to implement sophisticated backoff strategies without cluttering your Lambda code.

### 5\. Decoupling Notification Infrastructure

In a multi-service architecture, WebSocket management should often be extracted into a dedicated **Notification Microservice**. The file-processing service would remain a pure producer, publishing `UploadStatusChanged` events, while the notification service handles delivery across multiple channels (WebSockets, Push Notifications, Email).

### 6\. Cross-Account Event Distribution

As your organization grows, consumers of your file-processing events may live in different AWS accounts. Leverage **EventBridge Bus-to-Bus routing** to forward domain events to a central integration bus or directly to consumer accounts, maintaining a clean event-driven contract across team boundaries.

## Production Check-list

*   Replace mock authorizers with robust JWT/OIDC verification.
    
*   Implement customer-managed KMS keys for encryption at rest (S3 & DynamoDB).
    
*   Protected CloudFront distribution by associating it with a web ACL that includes AWS WAF managed rules and IP-based rate limiting.
    
*   Define granular IAM boundaries and S3 Bucket Policies (Least Privilege).
    
*   Add structured logging and X-Ray tracing for end-to-end correlation.
    
*   Enforce S3 Object Lock or Versioning for compliance-heavy domains.
    

## Final Thoughts

In modern web applications, handling file uploads is a multi-stage challenge. You need to ensure security, perform transformations, and keep the user informed, all while maintaining a serverless, cost-effective footprint.

The beauty of this architecture lies in **AWS Service Alignment**. We aren't fighting the platform; we're using each service for what it's naturally good at:

*   **S3** for durable storage.
    
*   **GuardDuty** for specialized security.
    
*   **Step Functions** for stateful orchestration.
    
*   **EventBridge** for decoupled, event-driven communication.
    

By separating ingestion from processing and using security as a gatekeeper, you create a system that is secure by default, observable, and ready to scale.

👉 [**View the full project on GitHub**](https://github.com/haZya/upload-file-processing-pipeline)