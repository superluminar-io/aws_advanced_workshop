# Lab 6: Implementing Auto Scaling and Monitoring

## Prerequisites

## Step 1: Configure Auto Scaling

Add to your existing `index.ts`:
```typescript
// Create Auto Scaling Target
const scalableTarget = new aws.appautoscaling.Target("workshop-scaling-target", {
    maxCapacity: 10,
    minCapacity: 2,
    resourceId: pulumi.interpolateservice/${cluster.name}/${service.name},
    scalableDimension: "ecs:service:DesiredCount",
    serviceNamespace: "ecs",
});
// Create CPU-based Scaling Policy
const cpuPolicy = new aws.appautoscaling.Policy("cpu-policy", {
    policyType: "TargetTrackingScaling",
    resourceId: scalableTarget.resourceId,
    scalableDimension: scalableTarget.scalableDimension,
    serviceNamespace: scalableTarget.serviceNamespace,
    targetTrackingScalingPolicyConfiguration: {
        predefinedMetricSpecification: {
            predefinedMetricType: "ECSServiceAverageCPUUtilization",
        },
        targetValue: 70.0,
        scaleInCooldown: 300,
        scaleOutCooldown: 300,
    },
});
```

## Verify the Deployment

1. **Deploy the Changes**:
```bash
pulumi up
```

2. **Verify in AWS Console**:
   - Check ECS service auto scaling configuration
   - Test scaling by generating load

## Best Practices

1. **Auto Scaling**:
   - Set appropriate scaling thresholds
   - Configure proper cooldown periods
   - Use target tracking for predictable workloads
   - Implement step scaling for specific scenarios

2. **Monitoring**:
   - Create comprehensive dashboards
   - Set up meaningful alerts
   - Monitor costs and resource utilization
   - Implement proper log retention

3. **Performance**:
   - Monitor application metrics
   - Track scaling events
   - Analyze resource utilization patterns
   - Optimize container configurations

## Troubleshooting

Common issues to check:

1. **Auto Scaling Issues**:
   - Verify scaling policy configuration
   - Check service CPU/memory metrics
   - Review scaling activity history
   - Confirm target group health checks

2. **CloudWatch Issues**:
   - Verify metric dimensions
   - Check alarm configurations
   - Review IAM permissions
   - Validate dashboard widgets

3. **Performance Issues**:
   - Monitor container insights
   - Review service logs
   - Check resource utilization
   - Analyze scaling patterns
