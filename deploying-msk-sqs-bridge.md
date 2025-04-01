# Deploying MSK to SQS Bridge Applications

This guide explains how to deploy Java applications that consume messages from Amazon MSK and forward them to Amazon SQS in a hybrid architecture.

## Overview

The MSK to SQS bridge is a critical component in a hybrid messaging architecture. It allows you to:

1. Consume high-volume, ordered events from MSK
2. Forward specific events to SQS for simpler processing
3. Transform messages between the two systems as needed

## Deployment Options

There are several ways to deploy your MSK to SQS bridge application:

1. **Amazon ECS/Fargate** - Containerized, managed deployment
2. **Amazon EC2** - Traditional VM-based deployment
3. **AWS Lambda** - Serverless deployment (with limitations)
4. **Amazon EKS** - Kubernetes-based deployment

Let's explore each option with implementation details.

## Option 1: Deploying on Amazon ECS/Fargate

### Step 1: Create a Docker Image

First, create a Dockerfile for your Java application:

```dockerfile
FROM amazoncorretto:11

# Set working directory
WORKDIR /app

# Copy the JAR file
COPY target/msk-sqs-bridge-1.0.0.jar /app/app.jar

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Step 2: Build and Push the Docker Image

```bash
# Build the Docker image
docker build -t msk-sqs-bridge:latest .

# Tag the image for ECR
docker tag msk-sqs-bridge:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/msk-sqs-bridge:latest

# Login to ECR
aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com

# Create ECR repository if it doesn't exist
aws ecr create-repository --repository-name msk-sqs-bridge --region ${REGION}

# Push the image to ECR
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/msk-sqs-bridge:latest
```

### Step 3: Create ECS Task Definition

Create a task definition for your application:

```json
{
  "family": "msk-sqs-bridge",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::${AWS_ACCOUNT_ID}:role/MSKSQSBridgeRole",
  "containerDefinitions": [
    {
      "name": "msk-sqs-bridge",
      "image": "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/msk-sqs-bridge:latest",
      "essential": true,
      "environment": [
        {
          "name": "BOOTSTRAP_SERVERS",
          "value": "your-msk-bootstrap-servers:9092"
        },
        {
          "name": "KAFKA_TOPIC",
          "value": "source-topic"
        },
        {
          "name": "CONSUMER_GROUP",
          "value": "msk-to-sqs-bridge"
        },
        {
          "name": "SQS_QUEUE_URL",
          "value": "https://sqs.${REGION}.amazonaws.com/${AWS_ACCOUNT_ID}/target-queue"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/msk-sqs-bridge",
          "awslogs-region": "${REGION}",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024"
}
```

### Step 4: Create IAM Role for Task

Create an IAM role with permissions to read from MSK and write to SQS:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:Connect",
        "kafka-cluster:DescribeGroup",
        "kafka-cluster:AlterGroup",
        "kafka-cluster:DescribeTopic",
        "kafka-cluster:ReadData"
      ],
      "Resource": [
        "arn:aws:kafka:${REGION}:${AWS_ACCOUNT_ID}:cluster/your-cluster-name/*",
        "arn:aws:kafka:${REGION}:${AWS_ACCOUNT_ID}:topic/your-cluster-name/*/source-topic",
        "arn:aws:kafka:${REGION}:${AWS_ACCOUNT_ID}:group/your-cluster-name/*/msk-to-sqs-bridge"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:SendMessageBatch"
      ],
      "Resource": "arn:aws:sqs:${REGION}:${AWS_ACCOUNT_ID}:target-queue"
    }
  ]
}
```

### Step 5: Create ECS Service

Create an ECS service to run your task:

```bash
aws ecs create-service \
  --cluster your-cluster \
  --service-name msk-sqs-bridge \
  --task-definition msk-sqs-bridge:1 \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-12345678,subnet-87654321],securityGroups=[sg-12345678],assignPublicIp=DISABLED}" \
  --region ${REGION}
```

### Step 6: Monitor the Service

Monitor your service using CloudWatch Logs and ECS service metrics:

```bash
# View logs
aws logs get-log-events \
  --log-group-name /ecs/msk-sqs-bridge \
  --log-stream-name ecs/msk-sqs-bridge/latest \
  --region ${REGION}
```
## Option 2: Deploying on Amazon EC2

### Step 1: Create an EC2 Instance

Launch an EC2 instance with the appropriate IAM role:

```bash
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium \
  --key-name your-key-pair \
  --security-group-ids sg-12345678 \
  --subnet-id subnet-12345678 \
  --iam-instance-profile Name=MSKSQSBridgeRole \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MSK-SQS-Bridge}]' \
  --region ${REGION}
```

### Step 2: Install Java and Dependencies

Connect to your EC2 instance and install Java:

```bash
# Update packages
sudo yum update -y

# Install Java 11
sudo amazon-linux-extras install java-openjdk11 -y

# Verify Java installation
java -version
```

### Step 3: Deploy Your Application

Transfer your application JAR to the EC2 instance:

```bash
# From your local machine
scp -i your-key-pair.pem target/msk-sqs-bridge-1.0.0.jar ec2-user@your-ec2-instance-ip:/home/ec2-user/
```

### Step 4: Create a Systemd Service

Create a systemd service file to run your application as a service:

```bash
# On the EC2 instance
sudo tee /etc/systemd/system/msk-sqs-bridge.service > /dev/null << 'EOF'
[Unit]
Description=MSK to SQS Bridge Service
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user
ExecStart=/usr/bin/java -jar /home/ec2-user/msk-sqs-bridge-1.0.0.jar
SuccessExitStatus=143
TimeoutStopSec=10
Restart=on-failure
RestartSec=5

Environment=BOOTSTRAP_SERVERS=your-msk-bootstrap-servers:9092
Environment=KAFKA_TOPIC=source-topic
Environment=CONSUMER_GROUP=msk-to-sqs-bridge
Environment=SQS_QUEUE_URL=https://sqs.${REGION}.amazonaws.com/${AWS_ACCOUNT_ID}/target-queue

[Install]
WantedBy=multi-user.target
EOF

# Enable and start the service
sudo systemctl enable msk-sqs-bridge
sudo systemctl start msk-sqs-bridge

# Check status
sudo systemctl status msk-sqs-bridge
```

### Step 5: Monitor the Application

Set up CloudWatch Agent to monitor your application:

```bash
# Install CloudWatch Agent
sudo yum install -y amazon-cloudwatch-agent

# Create CloudWatch Agent configuration
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json > /dev/null << 'EOF'
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/home/ec2-user/msk-sqs-bridge.log",
            "log_group_name": "/ec2/msk-sqs-bridge",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  },
  "metrics": {
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent"]
      },
      "swap": {
        "measurement": ["swap_used_percent"]
      }
    }
  }
}
EOF

# Start CloudWatch Agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```
## Option 3: Deploying as AWS Lambda Function

### Step 1: Create a Lambda Function Project

Structure your project to work with Lambda:

```java
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.KafkaEvent;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.SendMessageRequest;

import java.util.Base64;

public class MskToSqsBridgeLambda implements RequestHandler<KafkaEvent, Void> {
    
    private final SqsClient sqsClient = SqsClient.create();
    private final String SQS_QUEUE_URL = System.getenv("SQS_QUEUE_URL");
    
    @Override
    public Void handleRequest(KafkaEvent kafkaEvent, Context context) {
        for (KafkaEvent.KafkaEventRecord record : kafkaEvent.getRecords().values().iterator().next()) {
            try {
                // Decode the Kafka message
                String message = new String(Base64.getDecoder().decode(record.getValue()));
                
                // Log the message
                context.getLogger().log("Processing message: " + message);
                
                // Send to SQS
                SendMessageRequest sendRequest = SendMessageRequest.builder()
                        .queueUrl(SQS_QUEUE_URL)
                        .messageBody(message)
                        .build();
                
                sqsClient.sendMessage(sendRequest);
                context.getLogger().log("Message sent to SQS: " + message);
                
            } catch (Exception e) {
                context.getLogger().log("Error processing message: " + e.getMessage());
            }
        }
        return null;
    }
}
```

### Step 2: Build the Lambda Package

Create a deployment package:

```bash
# Build with Maven
mvn clean package

# The JAR file will be in the target directory
```

### Step 3: Create Lambda Function

Create the Lambda function:

```bash
aws lambda create-function \
  --function-name MSKToSQSBridge \
  --runtime java11 \
  --handler com.example.MskToSqsBridgeLambda::handleRequest \
  --role arn:aws:iam::${AWS_ACCOUNT_ID}:role/MSKSQSBridgeLambdaRole \
  --zip-file fileb://target/msk-sqs-bridge-lambda-1.0.0.jar \
  --timeout 60 \
  --memory-size 512 \
  --environment "Variables={SQS_QUEUE_URL=https://sqs.${REGION}.amazonaws.com/${AWS_ACCOUNT_ID}/target-queue}" \
  --region ${REGION}
```

### Step 4: Create IAM Role for Lambda

Create an IAM role with permissions for Lambda to access MSK and SQS:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:Connect",
        "kafka-cluster:DescribeGroup",
        "kafka-cluster:AlterGroup",
        "kafka-cluster:DescribeTopic",
        "kafka-cluster:ReadData"
      ],
      "Resource": [
        "arn:aws:kafka:${REGION}:${AWS_ACCOUNT_ID}:cluster/your-cluster-name/*",
        "arn:aws:kafka:${REGION}:${AWS_ACCOUNT_ID}:topic/your-cluster-name/*/source-topic",
        "arn:aws:kafka:${REGION}:${AWS_ACCOUNT_ID}:group/your-cluster-name/*/msk-to-sqs-bridge-lambda"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:SendMessageBatch"
      ],
      "Resource": "arn:aws:sqs:${REGION}:${AWS_ACCOUNT_ID}:target-queue"
    }
  ]
}
```

### Step 5: Create Event Source Mapping

Connect your Lambda function to MSK:

```bash
aws lambda create-event-source-mapping \
  --function-name MSKToSQSBridge \
  --event-source-arn arn:aws:kafka:${REGION}:${AWS_ACCOUNT_ID}:cluster/your-cluster-name/your-cluster-uuid \
  --topics source-topic \
  --starting-position LATEST \
  --batch-size 100 \
  --region ${REGION}
```

### Step 6: Monitor the Lambda Function

Monitor your Lambda function using CloudWatch Logs:

```bash
# View logs
aws logs filter-log-events \
  --log-group-name /aws/lambda/MSKToSQSBridge \
  --region ${REGION}
```

### Lambda Limitations

Note these limitations when using Lambda for MSK to SQS bridging:

1. Maximum execution time of 15 minutes
2. Limited concurrency and scaling options
3. Potential cold start delays
4. Less control over consumer behavior compared to dedicated applications
## Option 4: Deploying on Amazon EKS

### Step 1: Create a Docker Image

Use the same Dockerfile as in the ECS example:

```dockerfile
FROM amazoncorretto:11

# Set working directory
WORKDIR /app

# Copy the JAR file
COPY target/msk-sqs-bridge-1.0.0.jar /app/app.jar

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Step 2: Build and Push the Docker Image

```bash
# Build the Docker image
docker build -t msk-sqs-bridge:latest .

# Tag the image for ECR
docker tag msk-sqs-bridge:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/msk-sqs-bridge:latest

# Login to ECR
aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com

# Create ECR repository if it doesn't exist
aws ecr create-repository --repository-name msk-sqs-bridge --region ${REGION}

# Push the image to ECR
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/msk-sqs-bridge:latest
```

### Step 3: Create Kubernetes Deployment

Create a deployment manifest file (msk-sqs-bridge-deployment.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: msk-sqs-bridge
  namespace: messaging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: msk-sqs-bridge
  template:
    metadata:
      labels:
        app: msk-sqs-bridge
    spec:
      serviceAccountName: msk-sqs-bridge-sa
      containers:
      - name: msk-sqs-bridge
        image: ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/msk-sqs-bridge:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        env:
        - name: BOOTSTRAP_SERVERS
          value: "your-msk-bootstrap-servers:9092"
        - name: KAFKA_TOPIC
          value: "source-topic"
        - name: CONSUMER_GROUP
          value: "msk-to-sqs-bridge"
        - name: SQS_QUEUE_URL
          value: "https://sqs.${REGION}.amazonaws.com/${AWS_ACCOUNT_ID}/target-queue"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 5
```

### Step 4: Create Service Account with IAM Role

Create a service account manifest file (msk-sqs-bridge-sa.yaml):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: msk-sqs-bridge-sa
  namespace: messaging
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::${AWS_ACCOUNT_ID}:role/MSKSQSBridgeEKSRole
```

### Step 5: Create IAM Role for EKS Service Account

Create an IAM role with permissions to read from MSK and write to SQS:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kafka-cluster:Connect",
        "kafka-cluster:DescribeGroup",
        "kafka-cluster:AlterGroup",
        "kafka-cluster:DescribeTopic",
        "kafka-cluster:ReadData"
      ],
      "Resource": [
        "arn:aws:kafka:${REGION}:${AWS_ACCOUNT_ID}:cluster/your-cluster-name/*",
        "arn:aws:kafka:${REGION}:${AWS_ACCOUNT_ID}:topic/your-cluster-name/*/source-topic",
        "arn:aws:kafka:${REGION}:${AWS_ACCOUNT_ID}:group/your-cluster-name/*/msk-to-sqs-bridge"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:SendMessageBatch"
      ],
      "Resource": "arn:aws:sqs:${REGION}:${AWS_ACCOUNT_ID}:target-queue"
    }
  ]
}
```

### Step 6: Deploy to EKS

Apply the Kubernetes manifests:

```bash
# Create namespace if it doesn't exist
kubectl create namespace messaging

# Apply service account
kubectl apply -f msk-sqs-bridge-sa.yaml

# Apply deployment
kubectl apply -f msk-sqs-bridge-deployment.yaml

# Check deployment status
kubectl get deployments -n messaging

# Check pods
kubectl get pods -n messaging

# View logs
kubectl logs -f deployment/msk-sqs-bridge -n messaging
```

### Step 7: Set Up Horizontal Pod Autoscaler

Create an HPA manifest file (msk-sqs-bridge-hpa.yaml):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: msk-sqs-bridge-hpa
  namespace: messaging
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: msk-sqs-bridge
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

Apply the HPA:

```bash
kubectl apply -f msk-sqs-bridge-hpa.yaml
```
## Deployment Comparison

| Deployment Option | Pros | Cons | Best For |
|------------------|------|------|----------|
| **ECS/Fargate** | - Fully managed<br>- No infrastructure to manage<br>- Easy scaling<br>- Simple deployment | - Higher cost than EC2<br>- Less control over infrastructure | Teams that want simplicity and don't need fine-grained control |
| **EC2** | - Full control over infrastructure<br>- Cost-effective for long-running workloads<br>- Customizable configuration | - Infrastructure management overhead<br>- Manual scaling<br>- OS patching required | Cost-sensitive workloads or when specific OS/hardware requirements exist |
| **Lambda** | - Serverless, no infrastructure<br>- Pay only for execution time<br>- Auto-scaling<br>- Simple deployment | - 15-minute execution limit<br>- Cold start latency<br>- Less control over consumer behavior<br>- Limited concurrency | Simple integrations with low to medium throughput |
| **EKS** | - Container orchestration<br>- Advanced scaling and resilience<br>- Consistent with Kubernetes workflows<br>- Multi-region deployment | - More complex setup<br>- Kubernetes expertise required<br>- Higher operational overhead | Large-scale deployments or when already using Kubernetes |

## Best Practices for MSK to SQS Bridge Deployments

### 1. Consumer Group Management

Ensure proper consumer group configuration:

```java
// In your Java code
Properties props = new Properties();
props.put(ConsumerConfig.GROUP_ID_CONFIG, "msk-to-sqs-bridge");
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"); // or "latest"
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false"); // Manual commit for better control
```

### 2. Error Handling and Retries

Implement robust error handling:

```java
try {
    // Process message and send to SQS
    sendToSqs(sqsClient, record.value());
    
    // Commit offset after successful processing
    consumer.commitSync();
} catch (SqsException sqsException) {
    // Handle SQS-specific errors
    logger.error("SQS error: " + sqsException.getMessage());
    
    // Implement retry logic with backoff
    if (isRetryable(sqsException)) {
        retryWithBackoff(record, retryCount);
    } else {
        // Log permanent failure
        logger.error("Permanent failure sending to SQS: " + sqsException.getMessage());
    }
} catch (Exception e) {
    // Handle other errors
    logger.error("Unexpected error: " + e.getMessage());
}
```

### 3. Monitoring and Alerting

Set up comprehensive monitoring:

1. **CloudWatch Metrics**:
   - MSK metrics: BytesInPerSec, BytesOutPerSec
   - SQS metrics: NumberOfMessagesSent, NumberOfMessagesReceived
   - Application metrics: ProcessingTime, ErrorCount

2. **CloudWatch Alarms**:
   - Consumer lag alarms
   - Error rate alarms
   - Processing time alarms

3. **Logging**:
   - Structured logging with correlation IDs
   - Log message counts and processing times
   - Log errors with context

### 4. Scaling Considerations

Optimize scaling based on your deployment model:

- **ECS/Fargate**: Use Service Auto Scaling with CloudWatch metrics
- **EC2**: Use EC2 Auto Scaling groups
- **Lambda**: Configure appropriate batch size and concurrency
- **EKS**: Use Horizontal Pod Autoscaler with custom metrics

### 5. Security Best Practices

Implement security best practices:

- Use IAM roles with least privilege
- Enable encryption in transit for MSK
- Enable encryption in transit for SQS
- Use VPC endpoints for SQS access
- Deploy in private subnets with appropriate security groups

## Conclusion

Deploying a Java application that consumes from MSK and pushes to SQS can be accomplished through various methods, each with its own advantages and trade-offs. The best deployment option depends on your specific requirements, existing infrastructure, operational preferences, and team expertise.

For most production use cases, ECS/Fargate provides a good balance of simplicity and control, while EC2 offers more customization at the cost of higher operational overhead. Lambda is ideal for simpler integrations with lower throughput, and EKS is best for teams already invested in Kubernetes.

Regardless of the deployment option chosen, ensure you implement proper error handling, monitoring, scaling, and security best practices to build a robust and reliable MSK to SQS bridge.
