# Scaling Amazon MSK Topics and Optimizing Consumer Processing

When dealing with high message volumes in Amazon MSK (Managed Streaming for Kafka), you need to optimize both the producer and consumer sides to ensure efficient processing. This guide covers key strategies for scaling MSK topics and improving consumer throughput.

## Topic-Level Scaling Strategies

### 1. Increase Partition Count

**Why it matters**: Partitions are the fundamental unit of parallelism in Kafka. More partitions allow more consumers to process messages in parallel.

**Implementation**:
```bash
# Check current partition count
aws kafka describe-topic --cluster-arn <cluster-arn> --topic-name <topic-name>

# Increase partitions (cannot be decreased later)
kafka-topics.sh --bootstrap-server <bootstrap-servers> --alter --topic <topic-name> --partitions <new-partition-count>
```

**Best practices**:
- Start with 1-2 partitions per broker and scale as needed
- Consider future growth when setting initial partition count
- Monitor partition distribution with CloudWatch metrics => <b>check which topic?</b>
- Ideal partition count = max(expected throughput ÷ single partition throughput, consumer parallelism)

**Limitations**:
- Cannot decrease partition count
- Too many partitions can increase overhead and leader elections
- Message ordering is only guaranteed within a partition

### 2. Configure Optimal Retention Settings

**Implementation**:
```bash
# Set retention by time (e.g., 7 days)
kafka-configs.sh --bootstrap-server <bootstrap-servers> --entity-type topics --entity-name <topic-name> --alter --add-config retention.ms=604800000

# Set retention by size (e.g., 1GB per partition)
kafka-configs.sh --bootstrap-server <bootstrap-servers> --entity-type topics --entity-name <topic-name> --alter --add-config retention.bytes=1073741824
```

### 3. Optimize Topic Configuration

**Key configurations**:
```bash
# Increase replication factor for durability (3 is recommended)
kafka-topics.sh --bootstrap-server <bootstrap-servers> --create --topic <topic-name> --partitions <count> --replication-factor 3

# Configure min.insync.replicas for durability
kafka-configs.sh --bootstrap-server <bootstrap-servers> --entity-type topics --entity-name <topic-name> --alter --add-config min.insync.replicas=2
```

## Consumer-Side Scaling Strategies

### 1. Increase Consumer Group Parallelism

**Implementation**:
- Create multiple consumer instances in the same consumer group
- Each consumer will automatically be assigned a subset of partitions
- Maximum effective consumers = number of partitions

**Example consumer configuration**:
```java
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "<bootstrap-servers>");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "my-consumer-group");
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false"); // Manual commit for better control
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500); // Adjust based on processing capacity
```

### 2. Optimize Consumer Configuration

**Key settings to tune**:
- `fetch.min.bytes`: Increase to reduce network round trips (e.g., 1MB)
- `fetch.max.wait.ms`: Maximum time to wait if fetch.min.bytes isn't satisfied
- `max.poll.records`: Control batch size per poll
- `max.partition.fetch.bytes`: Maximum bytes fetched per partition

**Example**:
```java
// Fetch at least 1MB of data per request
props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1048576);
// Wait up to 500ms if min bytes not available
props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);
// Process up to 1000 records per poll
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 1000);
// Fetch up to 5MB per partition
props.put(ConsumerConfig.MAX_PARTITION_FETCH_BYTES_CONFIG, 5242880);
```

### 3. Implement Parallel Processing Within Consumers

**Strategies**:
1. **Thread pool for message processing**:
```java
ExecutorService executor = Executors.newFixedThreadPool(10);
ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
List<Future<?>> futures = new ArrayList<>();

for (ConsumerRecord<String, String> record : records) {
    futures.add(executor.submit(() -> processRecord(record)));
}

// Wait for all tasks to complete
for (Future<?> future : futures) {
    future.get();
}

// Commit offsets after all processing is complete
consumer.commitSync();
```

2. **Batch processing**:
```java
ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>();

// Process records in batches
List<ConsumerRecord<String, String>> batch = new ArrayList<>();
for (ConsumerRecord<String, String> record : records) {
    batch.add(record);
    if (batch.size() >= BATCH_SIZE) {
        processBatch(batch);
        batch.clear();
    }
    
    // Track offsets
    currentOffsets.put(
        new TopicPartition(record.topic(), record.partition()),
        new OffsetAndMetadata(record.offset() + 1)
    );
}

// Process any remaining records
if (!batch.isEmpty()) {
    processBatch(batch);
}

// Commit offsets
consumer.commitSync(currentOffsets);
```

## AWS-Specific Scaling Strategies

### 1. Scale MSK Cluster

**Options**:
- Increase broker count
- Upgrade broker instance types
- Enable MSK Provisioned Throughput mode for predictable performance

**Implementation**:
```bash
# Update broker count
aws kafka update-broker-count \
    --cluster-arn <cluster-arn> \
    --current-version <current-version> \
    --target-number-of-broker-nodes <new-broker-count>

# Update broker type
aws kafka update-broker-type \
    --cluster-arn <cluster-arn> \
    --current-version <current-version> \
    --target-instance-type <instance-type>
```

### 2. Use MSK Connect for Managed Consumers

MSK Connect provides managed Kafka Connect workers that can scale automatically:

```bash
# Create a connector with auto-scaling
aws kafkaconnect create-connector \
    --connector-name "my-scaling-connector" \
    --connector-configuration '{"connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector", "tasks.max": "10", "topics": "my-topic", "file": "/tmp/output.txt"}' \
    --capacity '{"provisionedCapacity": {"mcuCount": 2, "workerCount": 6}, "autoScaling": {"maxWorkerCount": 10, "mcuCount": 2, "minWorkerCount": 2, "scaleInPolicy": {"cpuUtilizationPercentage": 20}, "scaleOutPolicy": {"cpuUtilizationPercentage": 80}}}'
```

### 3. Implement AWS Lambda Consumers

For serverless processing that scales automatically:

```yaml
# CloudFormation example
Resources:
  KafkaConsumerFunction:
    Type: AWS::Lambda::Function
    Properties:
      Handler: index.handler
      Runtime: nodejs14.x
      Code:
        ZipFile: |
          exports.handler = async (event) => {
            for (const record of event.records) {
              const message = Buffer.from(record.value, 'base64').toString();
              console.log(`Processing message: ${message}`);
              // Process message here
            }
            return { statusCode: 200 };
          }
      
  MSKTrigger:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      FunctionName: !Ref KafkaConsumerFunction
      EventSourceArn: !Ref MSKClusterArn
      Topics:
        - my-topic
      StartingPosition: LATEST
      BatchSize: 100
      MaximumBatchingWindowInSeconds: 5
```

## Monitoring and Optimization

### 1. Key Metrics to Monitor

- **Consumer lag**: Difference between latest produced offset and consumed offset
- **Throughput**: Messages/second processed
- **Partition distribution**: Ensure even distribution across brokers
- **CPU/Memory usage**: On both brokers and consumers

**Implementation**:
```bash
# Check consumer group lag
kafka-consumer-groups.sh --bootstrap-server <bootstrap-servers> --describe --group <consumer-group>

# Set up CloudWatch dashboard for MSK metrics
aws cloudwatch put-dashboard \
    --dashboard-name MSKMonitoring \
    --dashboard-body file://msk-dashboard.json
```

### 2. Implement Lag-Based Auto-Scaling

For consumer applications running on ECS or EKS:

```yaml
# ECS Service Auto Scaling based on custom CloudWatch metric
Resources:
  ConsumerService:
    Type: AWS::ECS::Service
    Properties:
      # Service configuration...
  
  ScalableTarget:
    Type: AWS::ApplicationAutoScaling::ScalableTarget
    Properties:
      MaxCapacity: 20
      MinCapacity: 2
      ResourceId: !Sub service/${ECSCluster}/${ConsumerService.Name}
      ScalableDimension: ecs:service:DesiredCount
      ServiceNamespace: ecs
  
  ScalingPolicy:
    Type: AWS::ApplicationAutoScaling::ScalingPolicy
    Properties:
      PolicyName: KafkaLagScalingPolicy
      PolicyType: TargetTrackingScaling
      ScalingTargetId: !Ref ScalableTarget
      TargetTrackingScalingPolicyConfiguration:
        CustomizedMetricSpecification:
          MetricName: KafkaConsumerLag
          Namespace: Custom/Kafka
          Dimensions:
            - Name: ConsumerGroup
              Value: my-consumer-group
          Statistic: Average
        TargetValue: 1000  # Target lag value
```

## Best Practices Summary

1. **Right-size partitions**: Match partition count to expected throughput and consumer parallelism
2. **Balance consumer instances**: Ensure you have enough consumers to process messages, but not more than partitions
3. **Batch processing**: Process messages in batches where possible
4. **Commit strategy**: Use manual commits after successful processing
5. **Monitor lag**: Set up alerts for consumer lag exceeding thresholds
6. **Scale proactively**: Add capacity before lag becomes critical
7. **Consider message key distribution**: Ensure even distribution across partitions
8. **Use compression**: Enable compression for large messages
9. **Implement backpressure**: Have mechanisms to slow down producers if consumers can't keep up
10. **Consider message lifecycle**: Set appropriate retention policies

## Advanced Scaling Techniques

### 1. Implementing Consumer Backpressure

When consumers can't keep up with the message flow, implementing backpressure mechanisms helps maintain system stability:

```java
// Dynamic poll control based on processing capacity
int maxPollRecords = calculateOptimalBatchSize(); // Dynamic calculation based on current performance
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, maxPollRecords);

// Adjust poll interval based on processing time
long processingStartTime = System.currentTimeMillis();
ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
processRecords(records);
long processingTime = System.currentTimeMillis() - processingStartTime;

// If processing is taking too long, reduce batch size
if (processingTime > MAX_PROCESSING_TIME) {
    int newMaxPollRecords = Math.max(MIN_POLL_RECORDS, maxPollRecords / 2);
    props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, newMaxPollRecords);
}
```

### 2. Implementing Partition Reassignment for Balance

When adding brokers or experiencing uneven load:

```bash
# Generate reassignment plan
kafka-reassign-partitions.sh --bootstrap-server <bootstrap-servers> \
  --topics-to-move-json-file topics.json \
  --broker-list "0,1,2,3,4" \
  --generate

# Execute reassignment
kafka-reassign-partitions.sh --bootstrap-server <bootstrap-servers> \
  --reassignment-json-file reassignment.json \
  --execute

# Verify reassignment
kafka-reassign-partitions.sh --bootstrap-server <bootstrap-servers> \
  --reassignment-json-file reassignment.json \
  --verify
```

### 3. Implementing Tiered Storage with S3 (MSK Tiered Storage)

For cost-effective storage of large volumes of data:

```bash
# Enable tiered storage when creating a cluster
aws kafka create-cluster \
  --cluster-name "tiered-storage-cluster" \
  --broker-node-group-info '{"instanceType": "kafka.m5.large", "clientSubnets": ["subnet-1", "subnet-2", "subnet-3"], "storageInfo": {"ebsStorageInfo": {"volumeSize": 100}}}' \
  --kafka-version "2.8.1" \
  --number-of-broker-nodes 3 \
  --storage-mode "TIERED"

# Configure topic for tiered storage
kafka-configs.sh --bootstrap-server <bootstrap-servers> \
  --entity-type topics \
  --entity-name <topic-name> \
  --alter \
  --add-config "remote.storage.enable=true"
```

## Real-World Scaling Architectures

### 1. High-Throughput Event Processing Architecture

For systems processing millions of events per minute:

```
[Producers] → [MSK (r5.4xlarge, 9 brokers)] → [Primary Consumers (ECS Fargate)] → [Processing Service] → [DynamoDB]
                      ↓
            [Secondary Consumers] → [S3 Data Lake] → [Athena/EMR for Analytics]
```

**Implementation details:**
- 9-broker MSK cluster with r5.4xlarge instances
- Topics with 27+ partitions (3× broker count)
- Consumer groups with auto-scaling based on SQS queue depth
- Message batching with optimized serialization (Avro/Protobuf)
- Separate consumer groups for different processing priorities

### 2. Near Real-Time Analytics Architecture

For systems requiring both real-time processing and analytics:

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

## Troubleshooting Common Scaling Issues

### 1. High Consumer Lag

**Symptoms:**
- Increasing lag metrics in CloudWatch
- Delayed processing of messages

**Solutions:**
- Increase consumer count (up to partition count)
- Optimize consumer processing (batch processing, async processing)
- Increase partition count if at consumer limit
- Upgrade broker instance types for more network/disk throughput

### 2. Uneven Partition Load

**Symptoms:**
- Some partitions have significantly more messages
- Some consumers are overloaded while others are idle

**Solutions:**
- Review message key distribution strategy
- Implement custom partitioner for more even distribution
- Consider null keys with round-robin partitioning for even distribution

```java
// Custom partitioner example
public class EvenDistributionPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        
        // Use message content hash for distribution if no key
        if (key == null) {
            return Utils.toPositive(Utils.murmur2(valueBytes)) % numPartitions;
        }
        
        // Use key hash for consistent routing
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }
    
    @Override
    public void close() {}
    
    @Override
    public void configure(Map<String, ?> configs) {}
}
```

### 3. Broker Overload

**Symptoms:**
- High CPU/memory utilization on brokers
- Network throughput bottlenecks

**Solutions:**
- Increase broker count
- Upgrade broker instance types
- Enable MSK Provisioned Throughput
- Implement producer-side batching to reduce request rate

## Conclusion

Scaling MSK topics and optimizing consumer processing requires a multi-faceted approach addressing both infrastructure and application-level optimizations. By implementing the strategies outlined in this guide, you can build a high-throughput, resilient Kafka-based system on AWS that efficiently processes large volumes of messages.

Remember that scaling is an iterative process - start with reasonable defaults, monitor performance, and adjust based on observed behavior and changing requirements.

# Understanding and Optimizing MSK Throughput

## MSK Throughput Fundamentals

Amazon MSK throughput is determined by several factors that work together to define the overall capacity of your Kafka cluster.

### 1. Throughput Components

**Producer Throughput**: The rate at which data can be written to the cluster
- Measured in MB/s or messages/second
- Limited by network bandwidth, broker disk I/O, and broker CPU

**Consumer Throughput**: The rate at which data can be read from the cluster
- Measured in MB/s or messages/second
- Limited by network bandwidth and broker CPU

**Total Cluster Throughput**: The aggregate of all data flowing in and out of the cluster
- Sum of producer and consumer throughput
- Limited by the instance type and number of brokers

### 2. MSK Instance Type Throughput Capabilities

| Instance Type | Network Bandwidth | Approximate Max Throughput (Producer + Consumer) |
|---------------|-------------------|--------------------------------------------------|
| kafka.t3.small | Up to 5 Gbps | ~20-30 MB/s |
| kafka.m5.large | Up to 10 Gbps | ~60-80 MB/s |
| kafka.m5.xlarge | Up to 10 Gbps | ~100-120 MB/s |
| kafka.m5.2xlarge | Up to 10 Gbps | ~160-200 MB/s |
| kafka.m5.4xlarge | Up to 10 Gbps | ~300-350 MB/s |
| kafka.m5.12xlarge | 12 Gbps | ~600-800 MB/s |
| kafka.m5.24xlarge | 25 Gbps | ~1.2-1.5 GB/s |

*Note: Actual throughput varies based on message size, replication factor, acknowledgment settings, and other factors.*

### 3. MSK Provisioned Throughput Mode

For workloads requiring guaranteed throughput levels, MSK offers Provisioned Throughput:

```bash
# Create cluster with provisioned throughput
aws kafka create-cluster \
  --cluster-name "high-throughput-cluster" \
  --broker-node-group-info '{"instanceType": "kafka.m5.4xlarge", "clientSubnets": ["subnet-1", "subnet-2", "subnet-3"], "storageInfo": {"ebsStorageInfo": {"volumeSize": 1000}}}' \
  --kafka-version "3.3.1" \
  --number-of-broker-nodes 3 \
  --client-authentication '{"sasl": {"scram": {"enabled": true}}}' \
  --provisioned-throughput '{"enabled": true, "volumeThroughput": "250"}'
```

Benefits:
- Guaranteed throughput levels independent of storage size
- Predictable performance for high-volume applications
- Ability to scale throughput without changing instance types

## Measuring and Monitoring Throughput

### 1. Key Throughput Metrics

**BytesInPerSec**: Rate of incoming bytes
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/Kafka \
  --metric-name BytesInPerSec \
  --dimensions Name=Cluster Name,Value=<cluster-name> \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average
```

**BytesOutPerSec**: Rate of outgoing bytes
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/Kafka \
  --metric-name BytesOutPerSec \
  --dimensions Name=Cluster Name,Value=<cluster-name> \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average
```

**MessagesInPerSec**: Rate of incoming messages
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/Kafka \
  --metric-name MessagesInPerSec \
  --dimensions Name=Cluster Name,Value=<cluster-name> \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average
```

### 2. Throughput Testing Tools

**Kafka Producer Performance Test Tool**:
```bash
kafka-producer-perf-test.sh \
  --topic throughput-test \
  --num-records 10000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=<bootstrap-servers> acks=1 \
  --print-metrics
```

**Kafka Consumer Performance Test Tool**:
```bash
kafka-consumer-perf-test.sh \
  --bootstrap-server <bootstrap-servers> \
  --topic throughput-test \
  --messages 10000000 \
  --threads 1
```

## Throughput Optimization Strategies

### 1. Producer-Side Throughput Optimization

**Batch Messages**:
```java
// Configure producer for high throughput
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "<bootstrap-servers>");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

// Increase batch size (default: 16KB)
props.put(ProducerConfig.BATCH_SIZE_CONFIG, 131072); // 128KB

// Allow more time for batching (default: 0ms)
props.put(ProducerConfig.LINGER_MS_CONFIG, 5); // 5ms delay for batching

// Increase producer buffer memory (default: 32MB)
props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864); // 64MB

// Compression (snappy is good balance of CPU/compression ratio)
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
```

**Optimize Acknowledgments**:
```java
// For highest throughput but potential data loss
props.put(ProducerConfig.ACKS_CONFIG, "0");

// For good throughput with leader acknowledgment
props.put(ProducerConfig.ACKS_CONFIG, "1");

// For durability with performance impact
props.put(ProducerConfig.ACKS_CONFIG, "all");
```

### 2. Consumer-Side Throughput Optimization

**Fetch More Data Per Request**:
```java
// Increase minimum data fetch size (default: 1 byte)
props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1048576); // 1MB

// Maximum wait time if min bytes not available (default: 500ms)
props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);

// Maximum bytes per partition (default: 1MB)
props.put(ConsumerConfig.MAX_PARTITION_FETCH_BYTES_CONFIG, 5242880); // 5MB
```

**Parallel Processing**:
```java
// Process records in parallel with a thread pool
ExecutorService executor = Executors.newFixedThreadPool(10);
ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

CountDownLatch latch = new CountDownLatch(records.count());
for (ConsumerRecord<String, String> record : records) {
    executor.submit(() -> {
        try {
            processRecord(record);
        } finally {
            latch.countDown();
        }
    });
}

// Wait for all processing to complete before committing
latch.await(MAX_PROCESSING_TIME_MS, TimeUnit.MILLISECONDS);
consumer.commitSync();
```

### 3. Network and Infrastructure Optimization

**Network Throughput**:
- Use placement groups for EC2-based consumers/producers
- Consider enhanced networking for EC2 instances
- Use private VPC endpoints for MSK access

**Client Proximity**:
- Place consumers/producers in the same AZ as MSK brokers when possible
- Use MSK's multi-AZ capability for resilience but be aware of cross-AZ data transfer costs

### 4. MSK-Specific Throughput Enhancements

**MSK Provisioned Throughput**:
- Provides dedicated I/O capacity independent of storage size
- Available in increments of MB/s per broker
- Enables consistent performance as data volume grows

**MSK Cluster Sizing**:
- Scale horizontally by adding more brokers
- Scale vertically by using larger instance types
- Balance between broker count and instance size based on partition requirements

## Throughput Bottleneck Identification

### 1. Common Throughput Bottlenecks

**Producer Bottlenecks**:
- High producer CPU usage
- Network saturation
- Slow acknowledgments from brokers

**Broker Bottlenecks**:
- High CPU utilization
- Disk I/O limitations
- Network bandwidth constraints

**Consumer Bottlenecks**:
- Slow message processing
- Inefficient deserialization
- Inadequate consumer parallelism

### 2. Throughput Monitoring Dashboard

Create a CloudWatch dashboard to monitor throughput metrics:

```bash
aws cloudwatch put-dashboard \
  --dashboard-name MSKThroughputDashboard \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "x": 0,
        "y": 0,
        "width": 12,
        "height": 6,
        "properties": {
          "metrics": [
            [ "AWS/Kafka", "BytesInPerSec", "Cluster Name", "YOUR_CLUSTER_NAME", { "stat": "Sum" } ],
            [ ".", "BytesOutPerSec", ".", ".", { "stat": "Sum" } ]
          ],
          "view": "timeSeries",
          "stacked": false,
          "region": "YOUR_REGION",
          "title": "MSK Throughput",
          "period": 60
        }
      },
      {
        "type": "metric",
        "x": 12,
        "y": 0,
        "width": 12,
        "height": 6,
        "properties": {
          "metrics": [
            [ "AWS/Kafka", "MessagesInPerSec", "Cluster Name", "YOUR_CLUSTER_NAME", { "stat": "Sum" } ]
          ],
          "view": "timeSeries",
          "stacked": false,
          "region": "YOUR_REGION",
          "title": "Messages Per Second",
          "period": 60
        }
      }
    ]
  }'
```

## Real-World Throughput Examples

### 1. High-Volume Event Processing (1M+ messages/minute)

**Cluster Configuration**:
- 6 brokers using kafka.m5.4xlarge
- 3 AZs (2 brokers per AZ)
- 30 partitions per topic (5× broker count)
- Provisioned throughput: 250 MB/s per broker

**Producer Configuration**:
- Batch size: 128KB
- Linger time: 10ms
- Compression: snappy
- acks=1 (leader acknowledgment)

**Consumer Configuration**:
- 30 consumer instances (matching partition count)
- Fetch size: 2MB
- Parallel processing with thread pools (10 threads per consumer)
- Commit frequency: Every 10,000 records or 5 seconds

**Achieved Throughput**:
- ~1.2 GB/s aggregate throughput
- ~1.5 million messages per minute
- Average end-to-end latency: 150ms

### 2. Real-time Analytics Pipeline

**Cluster Configuration**:
- 9 brokers using kafka.m5.2xlarge
- 3 AZs (3 brokers per AZ)
- 18 partitions per topic (2× broker count)

**Producer Configuration**:
- Multiple producer applications
- Avro serialization with Schema Registry
- Batch size: 64KB
- Linger time: 5ms

**Consumer Configuration**:
- Lambda consumers for real-time processing
- Batch size: 100 messages
- Maximum batching window: 3 seconds
- Kinesis Data Firehose for S3 delivery

**Achieved Throughput**:
- ~500 MB/s aggregate throughput
- ~800,000 messages per minute
- Average processing latency: <1 second

## Conclusion

Optimizing MSK throughput requires a holistic approach that addresses producer configuration, consumer processing, cluster sizing, and monitoring. By implementing the strategies outlined in this guide and continuously monitoring performance, you can achieve high-throughput Kafka processing on AWS MSK that scales with your application needs.

Remember that throughput optimization is a balance between performance, cost, and reliability. Always test your specific workload patterns and adjust configurations based on observed metrics rather than relying solely on theoretical maximums.
