# Kafka/MSK-Focused Decision Graph for AWS Event Services

This document provides an updated decision graph and recommendations for customers who are primarily focused on using Apache Kafka/Amazon MSK while integrating with other AWS messaging services where appropriate.

## Updated AWS Event Services Decision Graph

```
Start
│
├─ Are you already using Apache Kafka or Amazon MSK?
│  ├─ Yes ↓
│  │  │
│  │  ├─ What's your primary use case with Kafka/MSK?
│  │  │  │
│  │  │  ├─ High-volume data streaming → Continue with MSK, consider scaling strategies
│  │  │  │
│  │  │  ├─ Need to process events in strict order → Continue with MSK (partition-level ordering)
│  │  │  │
│  │  │  ├─ Need Kafka-compatible ecosystem tools → Continue with MSK
│  │  │  │
│  │  │  └─ Simple messaging without Kafka-specific features → Consider complementing with SQS/SNS for simpler workloads
│  │  │
│  │  └─ Do you need to improve your current MSK implementation?
│  │     │
│  │     ├─ Need better error handling → Implement custom DLQ with MSK (see implementing-dlq-with-msk.md)
│  │     │
│  │     ├─ Experiencing performance issues → Review scaling-msk-topics.md for optimization strategies
│  │     │
│  │     ├─ Need to integrate with AWS services → Consider EventBridge for AWS service integration while keeping MSK
│  │     │
│  │     └─ Need simpler queue-based processing for some workloads → Add SQS for those specific workloads
│  │
│  └─ No ↓
│
├─ Do you need to integrate with third-party SaaS applications?
│  ├─ Yes → Amazon EventBridge
│  └─ No ↓
│
├─ Do you need Dead Letter Queue (DLQ) functionality?
│  │
│  ├─ Yes ↓
│  │  │
│  │  ├─ Do you need native, managed DLQ support?
│  │  │  ├─ Yes → Amazon SQS (native DLQ support)
│  │  │  └─ No ↓
│  │  │
│  │  └─ Are you willing to implement custom DLQ logic?
│  │     ├─ Yes → Can use MSK/Kafka with custom DLQ implementation
│  │     └─ No → Use Amazon SQS for simplicity
│  │
│  └─ No ↓
│
├─ What's your primary integration pattern?
│  │
│  ├─ Pub/Sub (Fan-out) ↓
│  │  │
│  │  ├─ Do you need guaranteed delivery to all subscribers?
│  │  │  ├─ Yes → Amazon SNS with SQS for each subscriber
│  │  │  └─ No → Amazon SNS
│  │  │
│  │  └─ Do you need filtering capabilities?
│  │     ├─ Yes → Amazon EventBridge
│  │     └─ No → Amazon SNS
│  │
│  ├─ Queue-based messaging ↓
│  │  │
│  │  ├─ Do you need FIFO (exactly-once processing)?
│  │  │  ├─ Yes → Amazon SQS FIFO queues
│  │  │  └─ No → Amazon SQS Standard queues
│  │  │
│  │  └─ Do you need to retain compatibility with existing JMS/AMQP applications?
│  │     ├─ Yes → Amazon MQ
│  │     └─ No → Amazon SQS
│  │
│  ├─ Stream processing ↓
│  │  │
│  │  ├─ Do you need to process high-volume data streams?
│  │  │  │
│  │  │  ├─ Do you need Apache Kafka compatibility?
│  │  │  │  ├─ Yes → Amazon MSK
│  │  │  │  └─ No ↓
│  │  │  │
│  │  │  └─ What's your primary use case?
│  │  │     ├─ Real-time analytics → Kinesis Data Analytics
│  │  │     ├─ Data ingestion → Kinesis Data Firehose
│  │  │     └─ Custom processing → Kinesis Data Streams
│  │  │
│  │  └─ Do you need to process events in order?
│  │     ├─ Yes → Kinesis Data Streams or MSK
│  │     └─ No → Consider other options
│  │
│  └─ Workflow orchestration ↓
│     │
│     ├─ Do you need to coordinate multiple AWS services?
│     │  ├─ Yes → AWS Step Functions
│     │  └─ No → Consider simpler event services
│     │
│     └─ Do you need visual workflow design?
│        ├─ Yes → AWS Step Functions
│        └─ No → Consider Lambda with EventBridge
│
└─ What are your latency requirements?
   ├─ Near real-time (milliseconds) → SNS, EventBridge
   ├─ Sub-second → Kinesis, MSK
   └─ Seconds to minutes → SQS, Step Functions
```

## MSK-Focused Integration Patterns

For customers already invested in Kafka/MSK, here are recommended integration patterns:

### 1. MSK as Central Event Backbone with Complementary AWS Services

```
[Data Sources] → [MSK] → [Primary Event Processing]
                   ↓
     ┌────────────┴───────────────┐
     ↓                            ↓
[EventBridge]                    [SQS]
(AWS Service Integration)        (Simple Queue Processing)
     ↓                            ↓
[Lambda/Step Functions]         [EC2/ECS Consumers]
```

**When to use this pattern:**
- When MSK is your primary event backbone
- When you need to integrate with AWS services (use EventBridge)
- When you need simple queue processing for specific workloads (use SQS)

### 2. MSK with Custom DLQ Implementation

```
[Producers] → [MSK Main Topics] → [Consumers]
                    ↓
              [MSK DLQ Topics] → [DLQ Processors]
                                      ↓
                               [Monitoring/Alerting]
```

**When to use this pattern:**
- When you need error handling in your Kafka workloads
- When you want to maintain Kafka compatibility throughout your pipeline
- When you need custom error handling logic

### 3. Hybrid MSK and SQS Architecture

```
[High-Volume/Ordered Events] → [MSK] → [Stream Processors]
                                          ↓
[Simple Messaging Workloads] → [SQS] → [Queue Processors]
```

**When to use this pattern:**
- When you have different types of workloads with different requirements
- When some workloads benefit from Kafka features while others are simpler
- When you want to optimize cost by using the right service for each workload

## AWS Messaging Services Comparison with MSK Focus

| Feature | Amazon MSK | Amazon SQS | Amazon SNS | EventBridge | Kinesis |
|---------|------------|------------|------------|-------------|---------|
| **Primary Use Case** | Stream processing, event sourcing | Queue-based workload decoupling | Pub/sub notifications | Event routing and filtering | Real-time data streaming |
| **Throughput** | Very High (MB/s to GB/s) | High | High | Medium | High |
| **Latency** | Low (sub-second) | Variable (ms to seconds) | Low (ms) | Low (ms) | Low (sub-second) |
| **Ordering** | Guaranteed within partition | FIFO queues only | Not guaranteed | Not guaranteed | Guaranteed within shard |
| **Retention** | Configurable (hours to years) | 14 days max | No retention | 24 hours | 24 hours to 365 days |
| **Native DLQ** | No (requires custom implementation) | Yes | No | No | No |
| **Scaling** | Manual scaling (broker count/size) | Automatic | Automatic | Automatic | Manual (shard count) |
| **Ecosystem** | Rich Kafka ecosystem | AWS-specific | AWS-specific | AWS-specific | AWS-specific |
| **Best For** | High-volume streaming, event sourcing, when Kafka compatibility is required | Simple queue processing, when native DLQ is needed | Simple pub/sub messaging | AWS service integration, event filtering | Real-time analytics when Kafka not required |

## MSK Optimization Recommendations

For customers already using Amazon MSK, consider these optimization strategies:

### 1. Scaling Strategies
- Increase partition count to match expected throughput and consumer parallelism
- Scale horizontally by adding more brokers
- Scale vertically by using larger instance types
- Consider MSK Provisioned Throughput for predictable performance
- Monitor partition distribution with CloudWatch metrics
- Implement partition reassignment for balance when adding brokers

### 2. Error Handling
- Implement custom DLQ topics within MSK
- Add error metadata to DLQ messages
- Create monitoring for DLQ topics
- Implement retry mechanisms before sending to DLQ
- Consider implementing a secondary DLQ for critical systems

### 3. Performance Optimization
- Configure optimal producer batching (batch size, linger time)
- Use compression for large messages (snappy is a good balance)
- Optimize consumer fetch size and processing
- Implement parallel processing within consumers
- Consider message key distribution for even partition load
- Implement consumer backpressure mechanisms

### 4. Integration with AWS Services
- Use MSK Connect for managed Kafka Connect deployments
- Implement Lambda consumers for serverless processing
- Consider EventBridge for routing events to other AWS services
- Use SQS for simpler workloads that don't require Kafka features
- Use Kinesis Data Firehose for data delivery to S3, Redshift, etc.

### 5. Monitoring and Observability
- Monitor consumer lag as a key performance indicator
- Set up CloudWatch dashboards for MSK metrics
- Implement lag-based auto-scaling for consumer applications
- Monitor BytesInPerSec, BytesOutPerSec, and MessagesInPerSec metrics
- Create alerts for abnormal patterns in throughput or lag

## When to Consider Alternatives to MSK

While maintaining MSK as your primary event backbone, consider these complementary services for specific use cases:

1. **Use SQS when:**
   - You need simple queue processing without Kafka-specific features
   - You need native, managed DLQ functionality
   - You want automatic scaling without managing partitions
   - You have workloads with variable or unpredictable throughput

2. **Use SNS when:**
   - You need simple pub/sub messaging without complex routing
   - You need to fan out messages to multiple subscribers
   - You don't need message persistence beyond delivery

3. **Use EventBridge when:**
   - You need to integrate with AWS services or third-party SaaS applications
   - You need content-based filtering and routing
   - You need schema validation for events

4. **Use Step Functions when:**
   - You need to orchestrate complex workflows across multiple services
   - You need visual workflow design and monitoring
   - You need built-in error handling and retry logic

## Real-World MSK Architecture Examples

### High-Throughput Event Processing Architecture

```
[Producers] → [MSK (r5.4xlarge, 9 brokers)] → [Primary Consumers (ECS Fargate)] → [Processing Service] → [DynamoDB]
                      ↓
            [Secondary Consumers] → [S3 Data Lake] → [Athena/EMR for Analytics]
```

**Implementation details:**
- 9-broker MSK cluster with r5.4xlarge instances
- Topics with 27+ partitions (3× broker count)
- Consumer groups with auto-scaling based on lag metrics
- Message batching with optimized serialization (Avro/Protobuf)
- Separate consumer groups for different processing priorities

### Near Real-Time Analytics Architecture

```
[Data Sources] → [MSK] → [Lambda Consumers] → [ElastiCache] → [API Gateway] → [Real-time Dashboards]
                   ↓
           [Kinesis Firehose] → [S3] → [Athena/Redshift] → [Analytics Dashboards]
```

**Implementation details:**
- MSK Connect with custom SMT (Single Message Transforms)
- Lambda consumers with optimized batch size and concurrency
- Kinesis Firehose for S3 delivery with optimized buffer settings
- CloudWatch metrics for consumer lag monitoring

## Conclusion

For customers already invested in Kafka/MSK, the best approach is typically to maintain MSK as the central event backbone while strategically integrating with other AWS messaging services for specific use cases. This hybrid approach leverages the strengths of MSK for high-throughput, ordered event processing while taking advantage of the simplicity and managed features of services like SQS, SNS, and EventBridge where appropriate.

By following the decision graph and recommendations in this document, you can optimize your MSK implementation while making informed decisions about when to incorporate other AWS messaging services into your architecture.
