# Lab 3: Building and Deploying Custom Images

## Step 1: Create ECR Repository

Add to your existing `index.ts`:
```typescript
// Create ECR Repository
const repository = new aws.ecr.Repository("workshop-app", {
    name: "workshop-app",
    imageScanningConfiguration: {
        scanOnPush: true,
    },
    forceDelete: true,
});

// Export the repository URL
export const repositoryUrl = repository.repositoryUrl;
```

## Step 2: Build and Push the Image

1. **Create Application Files**:
   Add a `Application.kt` file to the root of your project:
```kotlin
package com.workshop.app

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@SpringBootApplication
class Application

fun main(args: Array<String>) {
    runApplication<Application>(*args)
}

@RestController
class HelloController {
    @GetMapping("/")
    fun hello() = mapOf("message" to "Hello from ECS!")
}
```

Add a `build.gradle.kts` file to the root of your project:
```kotlin
plugins {
    id("org.springframework.boot") version "3.2.3"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.spring") version "1.9.22"
}

group = "com.workshop"
version = "0.0.1-SNAPSHOT"

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
}
```

Create a `Dockerfile`:
```dockerfile
FROM gradle:8.6.0-jdk17 AS build
WORKDIR /app
COPY build.gradle.kts settings.gradle.kts ./
COPY src ./src
RUN gradle build --no-daemon

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

2. **Build and Push Commands**:
Get ECR login credentials
```bash
aws ecr get-login-password --region eu-central-1 | docker login --username AWS --password-stdin $(pulumi stack output repositoryUrl)
```

Build the image
```bash
docker build -t workshop-app .
```

Tag the image
```bash
docker tag workshop-app:latest $(pulumi stack output repositoryUrl):latest
```

Push to ECR
```bash
docker push $(pulumi stack output repositoryUrl):latest
```

## Step 3: Update ECS Task Definition

Update your task definition in `index.ts` to use the custom image:
```typescript
// Update Task Definition with custom image
const taskDefinition = new aws.ecs.TaskDefinition("workshop-task", {
family: "workshop-app",
cpu: "256",
memory: "512",
networkMode: "awsvpc",
requiresCompatibilities: ["FARGATE"],
executionRoleArn: taskExecutionRole.arn,
containerDefinitions: pulumi.all([repository.repositoryUrl])
.apply(([repoUrl]) => JSON.stringify([{
name: "workshop-app",
image: ${repoUrl}:latest,
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

## Step 4: Deploy and Verify

1. **Deploy the Changes**:
```bash
pulumi up
```

2. **Verify the Deployment**:
   - Navigate to ECR in AWS Console
   - Check image scan results
   - View running tasks in ECS
   - Access the application through ALB DNS (from Lab 2)

## Best Practices

1. **Image Security**:
   - Enable image scanning
   - Use multi-stage builds
   - Minimize image size
   - Keep base images updated

2. **ECR Management**:
   - Implement lifecycle policies
   - Tag images appropriately
   - Clean up unused images

3. **Application Configuration**:
   - Use environment variables
   - Implement health checks
   - Follow the 12-factor app methodology

## Troubleshooting

Common issues and solutions:

1. **Image Pull Failures**:
   - Check ECR permissions
   - Verify image tags
   - Review task execution role

2. **Application Errors**:
   - Check CloudWatch logs
   - Verify container port mappings
   - Review environment variables

3. **Performance Issues**:
   - Monitor container metrics
   - Review resource allocations
   - Check application logs

## Next Steps

In Lab 4, we'll add CloudFront distribution to our architecture for global content delivery and enhanced security.