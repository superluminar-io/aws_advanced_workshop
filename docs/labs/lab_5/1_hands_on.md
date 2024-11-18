# Lab 5: Implementing Message Queuing

## Step 1: Modify the Kotlin application to send messages
`MessageController.kt`
```kotlin
package com.workshop.app

import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RestController
import software.amazon.awssdk.services.sqs.SqsClient
import software.amazon.awssdk.services.sqs.model.SendMessageRequest

@RestController
class MessageController(private val sqsClient: SqsClient) {
    @PostMapping("/message")
    fun sendMessage(@RequestBody message: Message) {
        val request = SendMessageRequest.builder()
            .queueUrl(System.getenv("QUEUE_URL"))
            .messageBody(message.content)
            .build()
        sqsClient.sendMessage(request)
    }
}

data class Message(val content: String)
```

`build.gradle.kts`
```kotlin
// Add to existing dependencies
dependencies {
    implementation(platform("software.amazon.awssdk:bom:2.24.0"))
    implementation("software.amazon.awssdk:sqs")
}
```

## Step 2: Create the infrastructure

**Step 1: Create SQS Queue and Lambda Function**

Add to your existing `index.ts`:
```typescript
// Create SQS Queue
const queue = new aws.sqs.Queue("workshop-queue", {
    visibilityTimeoutSeconds: 30,
    messageRetentionSeconds: 86400,
    redrivePolicy: JSON.stringify({
        deadLetterTargetArn: new aws.sqs.Queue("workshop-dlq").arn,
        maxReceiveCount: 3
    })
});
// Create Lambda Role
const lambdaRole = new aws.iam.Role("message-processor-role", {
    assumeRolePolicy: JSON.stringify({
        Version: "2012-10-17",
        Statement: [{
            Action: "sts:AssumeRole",
            Effect: "Allow",
            Principal: {
                Service: "lambda.amazonaws.com"
            }
        }]
    })
});
// Add SQS permissions to Lambda Role
new aws.iam.RolePolicy("lambda-sqs-policy", {
    role: lambdaRole.id,
    policy: queue.arn.apply(arn => JSON.stringify({
        Version: "2012-10-17",
        Statement: [{
            Effect: "Allow",
            Action: [
                "sqs:ReceiveMessage",
                "sqs:DeleteMessage",
                "sqs:GetQueueAttributes"
            ],
            Resource: arn
        }]
    }))
});
// Create Lambda Function
const processor = new aws.lambda.Function("message-processor", {
    runtime: "nodejs18.x",
    handler: "index.handler",
    role: lambdaRole.arn,
    code: new pulumi.asset.AssetArchive({
        "index.js": new pulumi.asset.StringAsset( exports.handler = async (event) => { for (const record of event.Records) { console.log('Processing message:', record.body); } return { statusCode: 200 }; }; )
    })
});
// Add SQS trigger to Lambda
new aws.lambda.EventSourceMapping("queue-trigger", {
    eventSourceArn: queue.arn,
    functionName: processor.name,
    batchSize: 1
});
// Update ECS Task Role with SQS permissions
const taskRole = new aws.iam.Role("ecs-task-role", {
    assumeRolePolicy: JSON.stringify({
        Version: "2012-10-17",
        Statement: [{
            Action: "sts:AssumeRole",
            Effect: "Allow",
            Principal: {
                Service: "ecs-tasks.amazonaws.com"
            }
        }]
    })
});
new aws.iam.RolePolicy("task-sqs-policy", {
    role: taskRole.id,
    policy: queue.arn.apply(arn => JSON.stringify({
        Version: "2012-10-17",
        Statement: [{
            Effect: "Allow",
            Action: ["sqs:SendMessage"],
            Resource: arn
        }]
    }))
});
```

## Step 3: Update ECS Task Definition

Update your task definition to include the queue URL and permissions:
```typescript
// Update Task Definition with environment variables
const taskDefinition = new aws.ecs.TaskDefinition("workshop-task", {
    family: "workshop-app",
    cpu: "256",
    memory: "512",
    networkMode: "awsvpc",
    requiresCompatibilities: ["FARGATE"],
    executionRoleArn: taskExecutionRole.arn,
    taskRoleArn: taskRole.arn,
    containerDefinitions: pulumi.all([repository.repositoryUrl, queue.url]).apply(([repoUrl, queueUrl]) => JSON.stringify([{
        name: "workshop-app",
        image: ${repoUrl}:latest,
        environment: [{
            name: "QUEUE_URL",
            value: queueUrl
        }],
        portMappings: [{
            containerPort: 8080,
            protocol: "tcp",
        }],
        logConfiguration: {
            logDriver: "awslogs",
            options: {
                "awslogs-group": "/ecs/workshop-app",
                "awslogs-region": "eu-central-1",
                "awslogs-stream-prefix": "ecs",
            },
        },
    }])),
});
```


## Verify the Deployment

Following the checkpoint style from Lab 2 (lines 276-292):

1. **Deploy the Changes**:
```bash
pulumi up
```

2. **Test Message Processing**:

Send a test message

```bash
curl -X POST -H "Content-Type: application/json" \
-d '{"content":"Hello from ECS!"}' \
http://$(pulumi stack output loadBalancerDns)/message
```
Check Lambda logs in CloudWatch

## Best Practices

1. **Queue Management**:
   - Configure DLQ for failed messages
   - Set appropriate retention periods
   - Monitor queue metrics
   - Implement proper error handling

2. **Lambda Best Practices**:
   - Handle partial batch failures
   - Implement proper error handling
   - Monitor function performance
   - Set appropriate timeout values

3. **Security**:
   - Follow least privilege principle
   - Encrypt messages in transit
   - Implement proper access controls
   - Regular security audits

## Checkpoint

At this point, you should have:
- Created an SQS queue with DLQ
- Updated the ECS task definition
- Created a Lambda consumer
- Successfully sent and processed messages
- Verified message flow in CloudWatch