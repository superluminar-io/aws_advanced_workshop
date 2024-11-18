# Lab 1: Hands-on ECS Deployment

## Step 1: Create the VPC Infrastructure

First, let's set up our networking infrastructure. Add the following to your `index.ts`:

```typescript
import as pulumi from "@pulumi/pulumi";
import as aws from "@pulumi/aws";
// Create VPC
const vpc = new aws.ec2.Vpc("workshop-vpc", {
    cidrBlock: "10.0.0.0/16",
    enableDnsHostnames: true,
    enableDnsSupport: true,
});
// Create Internet Gateway
const internetGateway = new aws.ec2.InternetGateway("workshop-igw", {
    vpcId: vpc.id,
});
// Create Public Subnets in different AZs
const publicSubnet1 = new aws.ec2.Subnet("workshop-public-1", {
    vpcId: vpc.id,
    cidrBlock: "10.0.1.0/24",
    availabilityZone: "eu-central-1a",
    mapPublicIpOnLaunch: true,
});
const publicSubnet2 = new aws.ec2.Subnet("workshop-public-2", {
    vpcId: vpc.id,
    cidrBlock: "10.0.2.0/24",
    availabilityZone: "eu-central-1b",
    mapPublicIpOnLaunch: true,
});
// Create Private Subnets
const privateSubnet1 = new aws.ec2.Subnet("workshop-private-1", {
    vpcId: vpc.id,
    cidrBlock: "10.0.3.0/24",
    availabilityZone: "eu-central-1a",
});
const privateSubnet2 = new aws.ec2.Subnet("workshop-private-2", {
    vpcId: vpc.id,
    cidrBlock: "10.0.4.0/24",
    availabilityZone: "eu-central-1b",
});
// Create Route Tables and Routes
const publicRouteTable = new aws.ec2.RouteTable("workshop-public-rt", {
    vpcId: vpc.id,
    routes: [{
        cidrBlock: "0.0.0.0/0",
            gatewayId: internetGateway.id,
        }],
});
// Associate Public Subnets with Public Route Table
const publicRtAssoc1 = new aws.ec2.RouteTableAssociation("workshop-public-rt-assoc-1", {
    subnetId: publicSubnet1.id,
    routeTableId: publicRouteTable.id,
});
const publicRtAssoc2 = new aws.ec2.RouteTableAssociation("workshop-public-rt-assoc-2", {
    subnetId: publicSubnet2.id,
    routeTableId: publicRouteTable.id,
});
```

## Step 2: Create ECS Cluster

Add the ECS cluster configuration:
```typescript
// Create ECS Cluster
const cluster = new aws.ecs.Cluster("workshop-cluster", {
    name: "workshop-cluster",
    settings: [{
        name: "containerInsights",
        value: "enabled",
    }],
});
// Create Task Execution Role
const taskExecutionRole = new aws.iam.Role("ecs-task-execution-role", {
    assumeRolePolicy: JSON.stringify({
        Version: "2012-10-17",
        Statement: [{
            Action: "sts:AssumeRole",
            Effect: "Allow",
            Principal: {
                Service: "ecs-tasks.amazonaws.com",
            },
        }],
    }),
});
const taskExecutionRolePolicy = new aws.iam.RolePolicyAttachment(
    "ecs-task-execution-role-policy",
    {
        role: taskExecutionRole.name,
        policyArn: "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy",
    }
);
```

## Step 3: Create Task Definition and Service

Add the following code to create the task definition and ECS service:
```typescript
// Create Task Definition
const taskDefinition = new aws.ecs.TaskDefinition("workshop-task", {
    family: "workshop-app",
    cpu: "256",
    memory: "512",
    networkMode: "awsvpc",
    requiresCompatibilities: ["FARGATE"],
    executionRoleArn: taskExecutionRole.arn,
    containerDefinitions: JSON.stringify([{
        name: "workshop-app",
        image: "nginx:latest",
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
    }]),
});

// Create Security Group for ECS Tasks
const taskSg = new aws.ec2.SecurityGroup("task-sg", {
    vpcId: vpc.id,
    description: "Allow inbound HTTP traffic",
    ingress: [{
        protocol: "tcp",
        fromPort: 80,
        toPort: 80,
        cidrBlocks: ["0.0.0.0/0"],
    }],
    egress: [{
        protocol: "-1",
        fromPort: 0,
        toPort: 0,
        cidrBlocks: ["0.0.0.0/0"],
    }],
});

// Create ECS Service
const service = new aws.ecs.Service("workshop-service", {
    cluster: cluster.id,
    taskDefinition: taskDefinition.arn,
    desiredCount: 2,
    launchType: "FARGATE",
    networkConfiguration: {
        subnets: [privateSubnet1.id, privateSubnet2.id],
        securityGroups: [taskSg.id],
        assignPublicIp: false,
    },
});
```

## Verify the Deployment

1. **Deploy the Infrastructure**:
```bash
pulumi up
```

2. **Verify in AWS Console**:
   - Navigate to ECS service
   - Check cluster status
   - Verify running tasks
   - Monitor CloudWatch logs

## Best Practices

1. **Security**:
   - Follow least privilege principle for IAM roles
   - Use private subnets for tasks
   - Restrict security group rules

2. **Logging**:
   - Enable CloudWatch logs
   - Set appropriate retention periods
   - Use structured logging

3. **Networking**:
   - Use private subnets for containers
   - Implement proper security groups
   - Consider NAT Gateway costs

## Troubleshooting

Common issues and solutions:

1. **Task Failed to Start**:
   - Check CloudWatch logs
   - Verify security group rules
   - Ensure proper subnet configuration

2. **Network Connectivity**:
   - Verify VPC endpoints
   - Check NAT Gateway configuration
   - Validate security group rules

3. **Service Stability**:
   - Monitor service events
   - Check task definition compatibility
   - Verify resource allocation
