# Hybrid MSK and SQS Architecture

This document provides detailed information about implementing a hybrid architecture that leverages both Amazon MSK (Managed Streaming for Apache Kafka) and Amazon SQS (Simple Queue Service) to optimize your event-driven applications.

## Overview

A Hybrid MSK and SQS Architecture combines the strengths of both services:

- **Amazon MSK**: High-throughput event streaming with strong ordering guarantees and rich ecosystem
- **Amazon SQS**: Simple, fully-managed queuing with automatic scaling and native DLQ support

This hybrid approach allows you to use the right tool for each specific workload within your overall architecture.

```
[High-Volume/Ordered Events] → [MSK] → [Stream Processors]
                                          ↓
[Simple Messaging Workloads] → [SQS] → [Queue Processors]
```

## Key Scenarios for Hybrid Architecture

### Scenario 1: Different Workload Requirements

Use MSK for high-volume, ordered event streams that require Kafka's features, while using SQS for simpler, decoupled workloads.

**Example**: An e-commerce platform using:
- MSK for order processing (requires strict ordering and high throughput)
- SQS for email notifications (simple, can be retried, no ordering requirements)

### Scenario 2: MSK with SQS as DLQ

Use MSK as the primary event backbone but leverage SQS for its native DLQ capabilities.

**Example**: A financial transaction system using:
- MSK for processing all transactions
- SQS as a dead-letter queue for failed transactions that need manual review

### Scenario 3: Gradual Migration

When migrating from SQS to MSK (or vice versa), a hybrid approach allows for gradual transition.

**Example**: A company migrating from SQS to MSK:
- New event types go to MSK
- Legacy systems continue using SQS
- Bridge components connect the two systems during transition

### Scenario 4: Cost Optimization

Use the more cost-effective service for each workload type.

**Example**: A data processing pipeline:
- MSK for high-value, critical data streams
- SQS for lower-priority, sporadic workloads where Kafka features aren't needed

## Implementation Considerations

### 1. Service Boundaries

Clearly define which types of events/messages belong in each system:

- **MSK**: Complex event streams, high-volume data, ordered processing, event sourcing
- **SQS**: Simple point-to-point messaging, variable load patterns, when native DLQ is needed

### 2. Data Consistency

When events flow between systems, ensure consistency through:

- Idempotent processing
- Correlation IDs across systems
- Transaction logs or audit trails

### 3. Operational Complexity

Be aware that managing two messaging systems increases operational complexity:

- Different monitoring approaches
- Separate scaling considerations
- Different failure modes and recovery processes

### 4. Common Integration Patterns

#### Bridge Pattern
Use a service to read from one system and write to another:
- MSK → SQS bridge for sending specific events to simpler processors
- SQS → MSK bridge for aggregating messages into an event stream

#### Splitter Pattern
Route different types of events to the appropriate system:
- High-value/ordered events to MSK
- Simple notifications to SQS

#### Aggregator Pattern
Combine events/messages from both systems for unified processing:
- Consume from both MSK and SQS
- Process in a unified way (e.g., Lambda function that handles both sources)
## Implementation Examples

### Example 1: MSK to SQS Bridge

This pattern is useful when you want to take specific events from MSK and route them to SQS for simpler processing.

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.SendMessageRequest;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class MskToSqsBridge {
    private static final String BOOTSTRAP_SERVERS = "your-msk-bootstrap-servers:9092";
    private static final String KAFKA_TOPIC = "orders-topic";
    private static final String CONSUMER_GROUP = "msk-to-sqs-bridge";
    private static final String SQS_QUEUE_URL = "https://sqs.region.amazonaws.com/account-id/queue-name";
    
    public static void main(String[] args) {
        // Create Kafka consumer
        final Consumer<String, String> consumer = createConsumer();
        
        // Create SQS client
        final SqsClient sqsClient = SqsClient.create();
        
        // Subscribe to the Kafka topic
        consumer.subscribe(Collections.singletonList(KAFKA_TOPIC));
        
        try {
            while (true) {
                // Poll for new messages
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
                
                for (ConsumerRecord<String, String> record : records) {
                    try {
                        // Process the message if needed
                        String messageBody = record.value();
                        
                        // Forward to SQS
                        sendToSqs(sqsClient, messageBody);
                        
                    } catch (Exception e) {
                        System.err.println("Error processing message: " + e.getMessage());
                    }
                }
                
                // Commit offsets
                consumer.commitSync();
            }
        } finally {
            consumer.close();
            sqsClient.close();
        }
    }
    
    private static void sendToSqs(SqsClient sqsClient, String messageBody) {
        SendMessageRequest sendRequest = SendMessageRequest.builder()
                .queueUrl(SQS_QUEUE_URL)
                .messageBody(messageBody)
                .build();
        
        sqsClient.sendMessage(sendRequest);
        System.out.println("Message sent to SQS: " + messageBody);
    }
    
    private static Consumer<String, String> createConsumer() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, CONSUMER_GROUP);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        
        return new KafkaConsumer<>(props);
    }
}
```

### Example 2: SQS to MSK Bridge

This pattern is useful for aggregating messages from SQS into an MSK topic.

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.*;

import java.util.List;
import java.util.Properties;
import java.util.concurrent.ExecutionException;

public class SqsToMskBridge {
    private static final String BOOTSTRAP_SERVERS = "your-msk-bootstrap-servers:9092";
    private static final String KAFKA_TOPIC = "aggregated-events";
    private static final String SQS_QUEUE_URL = "https://sqs.region.amazonaws.com/account-id/queue-name";
    
    public static void main(String[] args) {
        // Create Kafka producer
        final Producer<String, String> producer = createProducer();
        
        // Create SQS client
        final SqsClient sqsClient = SqsClient.create();
        
        try {
            while (true) {
                // Poll for SQS messages
                ReceiveMessageRequest receiveRequest = ReceiveMessageRequest.builder()
                        .queueUrl(SQS_QUEUE_URL)
                        .maxNumberOfMessages(10)
                        .waitTimeSeconds(20) // Long polling
                        .build();
                
                List<Message> messages = sqsClient.receiveMessage(receiveRequest).messages();
                
                for (Message message : messages) {
                    try {
                        // Send to Kafka
                        sendToKafka(producer, message.body());
                        
                        // Delete from SQS after successful processing
                        DeleteMessageRequest deleteRequest = DeleteMessageRequest.builder()
                                .queueUrl(SQS_QUEUE_URL)
                                .receiptHandle(message.receiptHandle())
                                .build();
                        
                        sqsClient.deleteMessage(deleteRequest);
                        
                    } catch (Exception e) {
                        System.err.println("Error processing message: " + e.getMessage());
                        // Message will return to queue after visibility timeout
                    }
                }
                
                // Small delay if no messages to avoid tight loop
                if (messages.isEmpty()) {
                    Thread.sleep(1000);
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            producer.close();
            sqsClient.close();
        }
    }
    
    private static void sendToKafka(Producer<String, String> producer, String messageBody) 
            throws ExecutionException, InterruptedException {
        // Use a unique key or null key depending on your partitioning needs
        ProducerRecord<String, String> record = new ProducerRecord<>(KAFKA_TOPIC, messageBody);
        
        // Synchronous send to ensure delivery
        producer.send(record).get();
        System.out.println("Message sent to Kafka: " + messageBody);
    }
    
    private static Producer<String, String> createProducer() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        
        return new KafkaProducer<>(props);
    }
}
```
### Example 3: Hybrid Consumer Application

This example shows an application that processes events from both MSK and SQS.

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.*;

import java.time.Duration;
import java.util.Collections;
import java.util.List;
import java.util.Properties;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicBoolean;

public class HybridConsumerApplication {
    private static final String BOOTSTRAP_SERVERS = "your-msk-bootstrap-servers:9092";
    private static final String KAFKA_TOPIC = "high-priority-events";
    private static final String CONSUMER_GROUP = "hybrid-consumer";
    private static final String SQS_QUEUE_URL = "https://sqs.region.amazonaws.com/account-id/low-priority-queue";
    
    private static final AtomicBoolean running = new AtomicBoolean(true);
    
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        
        // Start MSK consumer thread
        executor.submit(() -> consumeFromMsk());
        
        // Start SQS consumer thread
        executor.submit(() -> consumeFromSqs());
        
        // Add shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            running.set(false);
            executor.shutdown();
        }));
    }
    
    private static void consumeFromMsk() {
        // Create Kafka consumer
        final Consumer<String, String> consumer = createKafkaConsumer();
        
        // Subscribe to the Kafka topic
        consumer.subscribe(Collections.singletonList(KAFKA_TOPIC));
        
        try {
            while (running.get()) {
                // Poll for new messages
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
                
                for (ConsumerRecord<String, String> record : records) {
                    try {
                        // Process high-priority event from MSK
                        processHighPriorityEvent(record.value());
                        
                    } catch (Exception e) {
                        System.err.println("Error processing MSK message: " + e.getMessage());
                    }
                }
                
                // Commit offsets
                consumer.commitSync();
            }
        } finally {
            consumer.close();
        }
    }
    
    private static void consumeFromSqs() {
        // Create SQS client
        final SqsClient sqsClient = SqsClient.create();
        
        try {
            while (running.get()) {
                // Poll for SQS messages
                ReceiveMessageRequest receiveRequest = ReceiveMessageRequest.builder()
                        .queueUrl(SQS_QUEUE_URL)
                        .maxNumberOfMessages(10)
                        .waitTimeSeconds(20) // Long polling
                        .build();
                
                List<Message> messages = sqsClient.receiveMessage(receiveRequest).messages();
                
                for (Message message : messages) {
                    try {
                        // Process low-priority message from SQS
                        processLowPriorityMessage(message.body());
                        
                        // Delete from SQS after successful processing
                        DeleteMessageRequest deleteRequest = DeleteMessageRequest.builder()
                                .queueUrl(SQS_QUEUE_URL)
                                .receiptHandle(message.receiptHandle())
                                .build();
                        
                        sqsClient.deleteMessage(deleteRequest);
                        
                    } catch (Exception e) {
                        System.err.println("Error processing SQS message: " + e.getMessage());
                        // Message will return to queue after visibility timeout
                    }
                }
                
                // Small delay if no messages to avoid tight loop
                if (messages.isEmpty()) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            }
        } finally {
            sqsClient.close();
        }
    }
    
    private static void processHighPriorityEvent(String eventData) {
        // Business logic for processing high-priority events from MSK
        System.out.println("Processing high-priority event: " + eventData);
        // Add your processing logic here
    }
    
    private static void processLowPriorityMessage(String messageBody) {
        // Business logic for processing low-priority messages from SQS
        System.out.println("Processing low-priority message: " + messageBody);
        // Add your processing logic here
    }
    
    private static Consumer<String, String> createKafkaConsumer() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, CONSUMER_GROUP);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        
        return new KafkaConsumer<>(props);
    }
}
```
### Example 4: MSK with SQS as Dead Letter Queue

This example shows how to implement a custom DLQ pattern using SQS as the dead letter queue for failed MSK message processing.

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.SendMessageRequest;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class MskWithSqsDlq {
    private static final String BOOTSTRAP_SERVERS = "your-msk-bootstrap-servers:9092";
    private static final String KAFKA_TOPIC = "main-topic";
    private static final String CONSUMER_GROUP = "msk-consumer-with-sqs-dlq";
    private static final String SQS_DLQ_URL = "https://sqs.region.amazonaws.com/account-id/dlq-queue";
    private static final int MAX_RETRIES = 3;
    
    public static void main(String[] args) {
        // Create Kafka consumer
        final Consumer<String, String> consumer = createConsumer();
        
        // Create SQS client for DLQ
        final SqsClient sqsClient = SqsClient.create();
        
        // Subscribe to the Kafka topic
        consumer.subscribe(Collections.singletonList(KAFKA_TOPIC));
        
        try {
            while (true) {
                // Poll for new messages
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
                
                for (ConsumerRecord<String, String> record : records) {
                    boolean processed = false;
                    int retryCount = 0;
                    
                    // Retry logic
                    while (!processed && retryCount < MAX_RETRIES) {
                        try {
                            // Process the message
                            processMessage(record.value());
                            processed = true;
                            
                        } catch (Exception e) {
                            retryCount++;
                            System.err.println("Error processing message (attempt " + retryCount + "): " + e.getMessage());
                            
                            if (retryCount >= MAX_RETRIES) {
                                // Send to SQS DLQ after max retries
                                sendToSqsDlq(sqsClient, record, e.getMessage());
                            } else {
                                // Wait before retry
                                try {
                                    Thread.sleep(1000 * retryCount);
                                } catch (InterruptedException ie) {
                                    Thread.currentThread().interrupt();
                                }
                            }
                        }
                    }
                }
                
                // Commit offsets
                consumer.commitSync();
            }
        } finally {
            consumer.close();
            sqsClient.close();
        }
    }
    
    private static void processMessage(String message) {
        // Your business logic to process the message
        System.out.println("Processing: " + message);
        
        // Simulate random failures for demonstration
        if (Math.random() < 0.3) {
            throw new RuntimeException("Simulated processing failure");
        }
    }
    
    private static void sendToSqsDlq(SqsClient sqsClient, ConsumerRecord<String, String> record, String errorMessage) {
        // Create enhanced message with error details and original message metadata
        String dlqMessage = String.format(
            "{\"original_value\": %s, \"original_topic\": \"%s\", \"original_partition\": %d, " +
            "\"original_offset\": %d, \"error_message\": \"%s\", \"error_time\": %d}",
            record.value(),
            record.topic(),
            record.partition(),
            record.offset(),
            errorMessage,
            System.currentTimeMillis()
        );
        
        // Send to SQS DLQ
        SendMessageRequest sendRequest = SendMessageRequest.builder()
                .queueUrl(SQS_DLQ_URL)
                .messageBody(dlqMessage)
                .build();
        
        sqsClient.sendMessage(sendRequest);
        System.out.println("Message sent to SQS DLQ: " + dlqMessage);
    }
    
    private static Consumer<String, String> createConsumer() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, CONSUMER_GROUP);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        
        return new KafkaConsumer<>(props);
    }
}
```
## Best Practices for Hybrid MSK-SQS Architecture

### 1. Message Format Standardization

When using both MSK and SQS, standardize your message format across both systems:

```java
// Example standardized message format
public class StandardizedMessage {
    private String messageId;
    private String messageType;
    private long timestamp;
    private String source;
    private Map<String, String> headers;
    private String payload;
    
    // Getters, setters, and methods to serialize/deserialize
    
    public String toJson() {
        // Convert to JSON string
    }
    
    public static StandardizedMessage fromJson(String json) {
        // Parse from JSON string
    }
}
```

### 2. Error Handling Strategy

Implement a consistent error handling strategy across both systems:

```java
public class ErrorHandler {
    private final SqsClient sqsClient;
    private final String dlqUrl;
    
    public ErrorHandler(SqsClient sqsClient, String dlqUrl) {
        this.sqsClient = sqsClient;
        this.dlqUrl = dlqUrl;
    }
    
    public void handleError(String messageSource, String messageId, String payload, Exception error) {
        // Create standardized error message
        Map<String, String> errorDetails = new HashMap<>();
        errorDetails.put("messageSource", messageSource); // "MSK" or "SQS"
        errorDetails.put("messageId", messageId);
        errorDetails.put("errorType", error.getClass().getName());
        errorDetails.put("errorMessage", error.getMessage());
        errorDetails.put("timestamp", String.valueOf(System.currentTimeMillis()));
        errorDetails.put("originalPayload", payload);
        
        // Convert to JSON
        String errorJson = convertToJson(errorDetails);
        
        // Send to DLQ
        SendMessageRequest sendRequest = SendMessageRequest.builder()
                .queueUrl(dlqUrl)
                .messageBody(errorJson)
                .build();
        
        sqsClient.sendMessage(sendRequest);
        
        // Log error
        System.err.println("Error handled for message " + messageId + ": " + error.getMessage());
    }
    
    private String convertToJson(Map<String, String> map) {
        // Implementation to convert map to JSON string
        return ""; // Placeholder
    }
}
```

### 3. Monitoring and Observability

Implement unified monitoring across both systems:

```java
public class HybridMessagingMetrics {
    private final CloudWatchClient cloudWatchClient;
    private final String namespace;
    
    public HybridMessagingMetrics(CloudWatchClient cloudWatchClient, String namespace) {
        this.cloudWatchClient = cloudWatchClient;
        this.namespace = namespace;
    }
    
    public void recordMessageProcessed(String source, String messageType) {
        putMetric("MessagesProcessed", 1.0, 
                 "Source", source,  // "MSK" or "SQS"
                 "MessageType", messageType);
    }
    
    public void recordProcessingTime(String source, String messageType, long milliseconds) {
        putMetric("ProcessingTimeMillis", (double) milliseconds, 
                 "Source", source,
                 "MessageType", messageType);
    }
    
    public void recordError(String source, String errorType) {
        putMetric("ProcessingErrors", 1.0, 
                 "Source", source,
                 "ErrorType", errorType);
    }
    
    private void putMetric(String metricName, double value, String... dimensions) {
        // Implementation to put metric to CloudWatch
    }
}
```

## Deployment Considerations

### Infrastructure as Code Example (AWS CDK)

Here's an example of how to provision both MSK and SQS resources using AWS CDK:

```java
import software.amazon.awscdk.core.*;
import software.amazon.awscdk.services.msk.*;
import software.amazon.awscdk.services.sqs.*;

public class HybridMessagingStack extends Stack {
    public HybridMessagingStack(final Construct scope, final String id) {
        super(scope, id);
        
        // Create MSK Cluster
        CfnCluster mskCluster = CfnCluster.Builder.create(this, "HybridMskCluster")
                .clusterName("hybrid-messaging-msk")
                .kafkaVersion("2.8.1")
                .numberOfBrokerNodes(3)
                .brokerNodeGroupInfo(CfnCluster.BrokerNodeGroupInfoProperty.builder()
                        .instanceType("kafka.m5.large")
                        .clientSubnets(Arrays.asList("subnet-1", "subnet-2", "subnet-3"))
                        .storageInfo(CfnCluster.StorageInfoProperty.builder()
                                .ebsStorageInfo(CfnCluster.EBSStorageInfoProperty.builder()
                                        .volumeSize(100)
                                        .build())
                                .build())
                        .build())
                .build();
        
        // Create SQS Queues
        Queue standardQueue = Queue.Builder.create(this, "StandardQueue")
                .queueName("hybrid-standard-queue")
                .visibilityTimeout(Duration.seconds(30))
                .retentionPeriod(Duration.days(14))
                .build();
        
        Queue dlqQueue = Queue.Builder.create(this, "DeadLetterQueue")
                .queueName("hybrid-dlq")
                .retentionPeriod(Duration.days(14))
                .build();
        
        // Create FIFO Queue for ordered processing
        Queue fifoQueue = Queue.Builder.create(this, "FifoQueue")
                .queueName("hybrid-fifo-queue.fifo")
                .fifo(true)
                .contentBasedDeduplication(true)
                .build();
    }
}
```

## Conclusion

A hybrid MSK and SQS architecture provides flexibility to use the right messaging service for each specific workload. By leveraging MSK for high-throughput, ordered event streams and SQS for simpler messaging needs, you can optimize your architecture for both performance and operational simplicity.

Key takeaways:
1. Use MSK for high-volume, ordered event streams that require Kafka's rich feature set
2. Use SQS for simpler messaging workloads, automatic scaling, and native DLQ support
3. Implement clear service boundaries based on workload characteristics
4. Standardize message formats and error handling across both systems
5. Implement unified monitoring and observability
6. Consider operational complexity when managing both systems

By following these guidelines and implementation examples, you can successfully build and maintain a hybrid MSK-SQS architecture that leverages the strengths of both services.
