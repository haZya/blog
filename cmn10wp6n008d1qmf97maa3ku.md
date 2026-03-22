---
title: "Mastering Cross-Account SNS to SQS Subscriptions with AWS CDK"
datePublished: 2026-03-22T00:33:15.650Z
cuid: cmn10wp6n008d1qmf97maa3ku
slug: mastering-cross-account-sns-to-sqs-subscriptions-with-aws-cdk
cover: https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/2481c863-7673-434a-8b9c-47e9b13736ef.png
tags: microservices, aws, aws-cdk, cloud-architecture

---

In a modern microservices architecture, decoupled communication is king. AWS SNS (Simple Notification Service) and SQS (Simple Queue Service) are the bedrock of this decoupling. But as your organization grows and you move toward a multi-account strategy, you'll inevitably hit a common hurdle: **How do you subscribe an SQS queue in Account B to an SNS topic in Account A?**

Cross-account subscriptions can be tricky because of permission boundaries and the "Pending Confirmation" dance. In this post, we’ll explore two ways to handle this using AWS CDK, and why one is clearly superior.

## The Scenario

Imagine a **Core Service** in Account A that publishes "Task Status" events to an SNS Topic. A **Consumer Service** in Account B needs to process these events via an SQS Queue.

> **Important:** For native SNS-to-SQS subscriptions, make sure the Topic and Queue are in the **same AWS Region**.

* * *

## Method 1: Subscribe as the Queue Owner (Recommended)

In this approach, the owner of the SNS Topic (Account A) "opens the door" by granting permission, and the owner of the SQS Queue (Account B) "walks in" by creating the subscription.

### Granting Consumer Account the Permission to Subscribe

Instead of managing every single subscription, the topic owner grants `sns:Subscribe` permissions to the consumer account.

**CDK Implementation (Account A):**

```typescript
const topic = new sns.Topic(this, "StatusTopic", {
  fifo: true,
  contentBasedDeduplication: true,
});

// Grant a specific account permission to subscribe
topic.grantSubscribe(new iam.AccountPrincipal("123456789012"));

// OR: If you are in an AWS Organization, grant access to the entire Org
topic.addToResourcePolicy(new iam.PolicyStatement({
  actions: ["sns:Subscribe"],
  resources: [topic.topicArn],
  principals: [new iam.AnyPrincipal()],
  conditions: {
    StringEquals: { "aws:PrincipalOrgID": "o-xxxxxxxxxx" }
  }
}));
```

### Subscribing SQS Queue to the Topic

Now, the consumer can subscribe their queue to the topic using the Topic ARN. Since they have permission on the topic and own the queue, the subscription is confirmed **instantly**.

**CDK Implementation (Account B):**

```typescript
const queue = new sqs.Queue(this, "ConsumerQueue", { fifo: true });
const topicArn = "arn:aws:sns:us-east-1:111122223333:StatusTopic.fifo";

const topic = sns.Topic.fromTopicArn(this, "ImportedTopic", topicArn);

topic.addSubscription(new subs.SqsSubscription(queue, {
  rawMessageDelivery: true,
  filterPolicy: {
    status: sns.SubscriptionFilter.stringFilter({ allowlist: ["CREATED"] }),
  },
}));
```

> **Deployment tip:** Deploy Account A (Topic policy/grants) before Account B creates the subscription.

### The "Self-Service" Advantage

This is the **Gold Standard** for microservices. Account A provides the "Event Bus" (the Topic), and consumers can:

*   Create as many queues as they need.
    
*   Define their own Filter Policies without bothering the producer.
    
*   Manage their own DLQs and retry logic.
    
*   Scale independently without any manual "confirmation" steps.
    

* * *

## Method 2: Subscribe as the Topic Owner

Here, the producer in Account A explicitly creates the subscription to the remote queue in Account B.

### Creating a Subscription for the Remote SQS Queue

Instead of allowing the consumers to initiate the subscriptions, the topic owner create and manage the subscriptions to the consumers.

**CDK Implementation (Account A):**

```typescript
const queue = sqs.Queue.fromQueueArn(this, "RemoteQueue", "arn:aws:sqs:us-east-1:444455556666:ConsumerQueue.fifo");

topic.addSubscription(new subs.SqsSubscription(queue));
```

### The "Symmetric Permission" Requirement

For this to work, you must also update the **SQS Queue Policy** in Account B to allow the SNS Topic to send messages.

**CDK Implementation (Account B):**

```typescript
// In Account B's CDK code:
queue.addToResourcePolicy(new iam.PolicyStatement({
  actions: ["sqs:SendMessage"],
  resources: [queue.queueArn],
  principals: [new iam.ServicePrincipal("sns.amazonaws.com")],
  conditions: {
    ArnEquals: { "aws:SourceArn": "arn:aws:sns:us-east-1:111122223333:StatusTopic.fifo" },
    StringEquals: { "aws:SourceAccount": "111122223333" }
  }
}));
```

**The Catch:** When the producer creates the subscription cross-account, it enters a `Pending Confirmation` state. AWS sends a `SubscriptionConfirmation` message to the SQS queue. The consumer then needs a confirmation step (manual or automated, e.g., a Lambda that calls `ConfirmSubscription`). This introduces extra coupling and operational complexity versus Method 1.

### Subscription confirmation

> **Step 1:** In Account B, search for the subscription confirmation message by polling the queue messages.

![SQS message with subscription confirmation URL.](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/29fdb864-c211-478d-aeba-dbdbd0e489b4.png align="center")

> Step 2: Extract the `SubscribeURL` from the message and enter it in a browser to confirm.

![Subscription confirmation result.](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/928f5931-d513-4831-82c9-08367bb5c6be.png align="center")

> Step 3: Now in Account A, verify that the subscription is confirmed.

![Verify subscription confirmation.](https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/2933be8a-5749-4532-b9f1-3dbbe2a5b154.png align="center")

* * *

## Essential Best Practices

### 1\. Use Filter Policies

Don't make your consumers process every single message. Use `SubscriptionFilter` to ensure they only get what they need.

```typescript
topic.addSubscription(new subs.SqsSubscription(queue, {
  filterPolicy: {
    status: sns.SubscriptionFilter.stringFilter({
      allowlist: ["CREATED", "COMPLETED"],
    }),
  },
}));
```

### 2\. Dead-Letter Queues (DLQs)

Always attach a DLQ to your SQS queue. If a message fails to process after several retries, it moves to the DLQ instead of blocking the queue.

```typescript
const dlq = new sqs.Queue(this, "DeadLetterQueue", {
  fifo: true,
  retentionPeriod: cdk.Duration.days(14),
});

const queue = new sqs.Queue(this, "ConsumerQueue", {
  fifo: true,
  deadLetterQueue: {
    queue: dlq,
    maxReceiveCount: 5, // Move to DLQ after 5 failed attempts
  },
});
```

### 3\. Encryption with KMS

When using Customer Managed Keys (CMKs) across accounts, KMS permissions depend on the exact service interactions:

*   **SNS-Encrypted Topics:** Your Publishers (IAM roles or AWS services) must have `kms:GenerateDataKey*` and `kms:Decrypt` on the Topic's key to encrypt payloads. The `sns.amazonaws.com` principal does not need permissions on this key to fan out messages.
    
*   **SQS-Encrypted Queues:** Because SNS writes to the queue, you must grant the `sns.amazonaws.com` service principal `kms:GenerateDataKey*` and `kms:Decrypt` on the Queue's key. Downstream Consumers also require `kms:Decrypt` on this key to read the messages.
    
*   **Cross-Account Setup:** Access requires two-way trust. You must explicitly allow the cross-account actions in both the KMS Key Policy (in the key-owning account) and the IAM Policy (in the calling account). If either is missing, access fails.
    

[Grant Amazon SNS KMS permissions to Amazon SNS to publish messages to the queue](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-least-privilege-policy.html#sqs-use-case-publish-messages-permissions)

```typescript
// Create the Customer Managed Key (CMK) for the SQS Queue
const queueKey = new kms.Key(this, "QueueKey", {
  enableKeyRotation: true,
  alias: "alias/my-queue-key",
});

// Allow SNS Service Principal to use the Queue's KMS Key
queueKey.addToResourcePolicy(new iam.PolicyStatement({
  effect: iam.Effect.ALLOW,
  principals: [new iam.ServicePrincipal("sns.amazonaws.com")],
  actions: ["kms:GenerateDataKey*", "kms:Decrypt"],
  resources: ["*"], // Evaluates to this specific key
  conditions: {
    // Security Best Practice: Restrict to the specific cross-account topic
    ArnEquals: { "aws:SourceArn": crossAccountTopicArn }
  }
}));
```

[Restrict message transmission to a specific Amazon SNS topic](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-least-privilege-policy.html#sqs-restrict-transmission-to-topic)

```typescript
// Create the Encrypted SQS Queue
const queue = new Queue(this, "MyEncryptedQueue", {
  encryption: sqs.QueueEncryption.KMS,
  encryptionMasterKey: queueKey,
});

// Allow the cross-account SNS topic to publish messages to the Queue
queue.addToResourcePolicy(new iam.PolicyStatement({
  effect: iam.Effect.ALLOW,
  principals: [new iam.ServicePrincipal("sns.amazonaws.com")],
  actions: ["sqs:SendMessage"],
  resources: [queue.queueArn],
  conditions: {
    ArnEquals: { "aws:SourceArn": crossAccountTopicArn }
  }
}));
```

> **Tip:** The `ArnEquals: { 'aws:SourceArn': crossAccountTopicArn }` condition is crucial to prevent the [confused deputy problem](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-least-privilege-policy.html#sqs-confused-deputy-prevention), ensuring only your specific topic can use this key and queue.

### 4\. FIFO Symmetry

If you're using FIFO (as in the examples), ensure both the Topic and the Queue are FIFO. A FIFO Topic cannot deliver to a standard Queue, and a standard Topic cannot deliver to a FIFO Queue. Also, make sure your publishers provide a `MessageGroupId`.

* * *

## Summary

Cross-account messaging is a powerful pattern for scaling. By **granting subscription permissions at the Topic level (Method 1)**, you enable a self-service model that empowers consumers and keeps your infrastructure code clean and automated.

## References

*   [AWS Docs: Sending SNS messages to an SQS queue in a different account](https://docs.aws.amazon.com/sns/latest/dg/sns-send-message-to-sqs-cross-account.html)
    

%[https://www.youtube.com/watch?v=xIcNTObKIuc]