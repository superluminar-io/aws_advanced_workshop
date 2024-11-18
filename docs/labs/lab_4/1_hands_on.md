# Lab 4: Setting up CloudFront

## Step 1: Create CloudFront Distribution

Add to your existing `index.ts`:
```typescript
// Create CloudFront distribution
const distribution = new aws.cloudfront.Distribution("workshop-cdn", {
    enabled: true,
    defaultCacheBehavior: {
        allowedMethods: [
            "DELETE",
            "GET",
            "HEAD",
            "OPTIONS",
            "PATCH",
            "POST",
            "PUT",
        ],
        cachedMethods: [
            "GET",
            "HEAD",
        ],
        targetOriginId: "ALB",
        viewerProtocolPolicy: "redirect-to-https",
        forwardedValues: {
            queryString: true,
            cookies: {
                forward: "all",
            },
        },
        minTtl: 0,
        defaultTtl: 3600,
        maxTtl: 86400,
    },
    origins: [{
        domainName: alb.dnsName,
        originId: "ALB",
        customOriginConfig: {
            httpPort: 80,
            httpsPort: 443,
            originProtocolPolicy: "http-only",
            originSslProtocols: ["TLSv1.2"],
        },
    }],
    restrictions: {
        geoRestriction: {
            restrictionType: "none",
        },
    },
    viewerCertificate: {
        cloudfrontDefaultCertificate: true,
    },
    customErrorResponses: [{
        errorCode: 404,
        responseCode: 404,
        responsePagePath: "/404.html",
    }],
});
    // Export CloudFront domain
export const cloudfrontDomain = distribution.domainName;
```

## Step 2: Configure Security Headers

Add a response headers policy to enhance security:
```typescript
// Create Response Headers Policy
const responseHeadersPolicy = new aws.cloudfront.ResponseHeadersPolicy("security-headers", {
    customHeadersConfig: {
        items: [{
            header: "Strict-Transport-Security",
            override: true,
            value: "max-age=31536000; includeSubdomains; preload",
        }],
    },
    securityHeadersConfig: {
        contentTypeOptions: {
            override: true,
        },
        frameOptions: {
            frameOption: "DENY",
            override: true,
        },
        xssProtection: {
            modeBlock: true,
            override: true,
            protection: true,
        },
    },
});
```

## Verify the Deployment

1. **Deploy the Changes**:
```bash
pulumi up
```

2. **Verify in AWS Console**:
   - Navigate to CloudFront
   - Check distribution status
   - Verify security headers
   - Test the application through CloudFront URL

## Best Practices

Following the style from Lab 2 (lines 268-274), let's adapt security best practices for CloudFront:

1. **Security**:
   - Enable WAF integration
   - Use custom SSL certificates
   - Implement secure response headers
   - Configure geo-restrictions when needed

2. **Performance**:
   - Optimize cache behaviors
   - Configure appropriate TTLs
   - Use compression
   - Enable modern TLS protocols

3. **Monitoring**:
   - Enable access logging
   - Set up CloudWatch alarms
   - Monitor cache statistics
   - Track error rates

## Troubleshooting

Following the checkpoint style from Lab 3 (lines 498-509):

Common issues to check:
- Verify origin configuration
- Check security group settings
- Review cache behavior settings
- Validate SSL certificate setup
- Inspect CloudWatch logs

## Clean Up

Following the cleanup process from Lab 1 (lines 168-183).

## Next Steps

In Lab 5, we'll explore auto-scaling and monitoring to ensure our application can handle varying loads efficiently.
