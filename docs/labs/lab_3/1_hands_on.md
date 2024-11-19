# Lab 3: Building and Deploying Custom Images

## Step 1: Create ECR Repository

Add this code to the beginning of your `index.ts`:
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

And deploy the changes:

```bash
pulumi up
```

## Step 2: Build and Push the Image

Create and prepare an application directory:

```bash
mkdir application
cd application
npm init -y
npm install express @types/express typescript ts-node
```

Create a `tsconfig.json` file:
```json
{
  "compilerOptions": {
    "target": "es6",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```

Create `src/app.ts`:
```typescript
import express from 'express';

const app = express();
const port = 8080;

app.get('/', (req, res) => {
  res.json({ message: 'Hello from ECS!' });
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```

Create a `Dockerfile`:
```dockerfile
FROM --platform=linux/amd64 node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

EXPOSE 80
CMD ["node", "dist/app.js"]
```

Update `package.json` scripts:
```json
{
...
  "scripts": {
    "build": "tsc",
    "start": "node dist/app.js"
  }
...
}
```

2. **Build and Push Commands**:
Get ECR login credentials (make sure you exported the `PULUMI_CONFIG_PASSPHRASE` environment variable):

```bash
aws ecr get-login-password --region eu-central-1 | docker login --username AWS --password-stdin $(pulumi stack output repositoryUrl)
```

Build the image
```bash
cd application/
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
        name: containerName,
        image: `${repoUrl}:latest`,
        portMappings: [{
            containerPort: 80,
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
cd ../ # go back to the root of the project
pulumi up
```

2. **Verify the Deployment**:
   - Navigate to ECR in AWS Console and check that the image was pushed successfully
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