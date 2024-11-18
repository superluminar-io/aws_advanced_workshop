# Lab 2: Adding Application Load Balancer

## Step 1: Create ALB Security Group

Add the following code to your existing `index.ts`:
```typescript
// Create Security Group for ALB
const albSg = new aws.ec2.SecurityGroup("alb-sg", {
    vpcId: vpc.id,
    description: "Security group for ALB",
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
// Update ECS Task Security Group to allow traffic from ALB
const taskSg = new aws.ec2.SecurityGroup("task-sg", {
    vpcId: vpc.id,
    description: "Security group for ECS tasks",
    ingress: [{
        protocol: "tcp",
        fromPort: 80,
        toPort: 80,
        securityGroups: [albSg.id],
    }],
    egress: [{
        protocol: "-1",
        fromPort: 0,
    toPort: 0,
    cidrBlocks: ["0.0.0.0/0"],
    }],
});
```

## Step 2: Create ALB and Target Group
```typescript
// Create Target Group
const targetGroup = new aws.lb.TargetGroup("workshop-tg", {
    port: 80,
    protocol: "HTTP",
    targetType: "ip",
    vpcId: vpc.id,
    healthCheck: {
        enabled: true,
        path: "/",
        healthyThreshold: 2,
        unhealthyThreshold: 10,
    },
});
// Create Application Load Balancer
const alb = new aws.lb.LoadBalancer("workshop-alb", {
    internal: false,
    loadBalancerType: "application",
    securityGroups: [albSg.id],
    subnets: [publicSubnet1.id, publicSubnet2.id],
});
// Create ALB Listener
const listener = new aws.lb.Listener("workshop-listener", {
    loadBalancerArn: alb.arn,
    port: 80,
    protocol: "HTTP",
    defaultActions: [{
        type: "forward",
        targetGroupArn: targetGroup.arn,
    }],
});
```

## Step 3: Update ECS Service
```typescript
typescript
// Update ECS Service with Load Balancer
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
    loadBalancers: [{
        targetGroupArn: targetGroup.arn,
        containerName: "workshop-app",
        containerPort: 80,
    }],
});
// Export ALB DNS name
export const albDnsName = alb.dnsName;
```

## Verify the Deployment

1. **Deploy the Infrastructure**:
```bash
pulumi up
```

2. **Check ALB Status**:
   - Navigate to EC2 > Load Balancers in AWS Console
   - Verify the ALB is in "active" state
   - Check that target group has healthy targets

3. **Test the Application**:
   - Copy the ALB DNS name from Pulumi outputs
   - Open in a web browser
   - You should see the nginx welcome page

## Best Practices

1. **Security**:
   - Follow security group best practices from Lab 3 (reference lines 450-454)
   - Use HTTPS listeners in production
   - Implement WAF for additional security

2. **High Availability**:
   - Deploy across multiple AZs
   - Monitor health check thresholds
   - Configure appropriate scaling policies

3. **Monitoring**:
   - Enable access logs
   - Set up CloudWatch alarms for:
     - Unhealthy host count
     - Request count
     - Target response time
   - Monitor 5xx errors

## Troubleshooting

Common issues and solutions:

1. **Unhealthy Targets**:
   - Check security group rules
   - Verify health check path
   - Inspect target group settings
   - Review ECS task logs

2. **Connection Timeouts**:
   - Verify VPC routing
   - Check security group rules
   - Ensure NAT Gateway is working

3. **5xx Errors**:
   - Check application logs
   - Verify container health
   - Monitor resource utilization

## Next Steps

In Lab 3, we'll learn about Amazon ECR and how to build and push custom container images.