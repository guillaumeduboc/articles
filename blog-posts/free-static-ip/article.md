---
published: false
title: 'Deploy a Lambda with a static IP for FREE'
cover_image: 'https://raw.githubusercontent.com/guillaumeduboc/articles/master/blog-posts/free-static-ip/cover.png'
description: 'A step-by-step guide and code snippets to assign a static IP to an AWS Lambda for free using the AWS CDK by extracting the Elastic Network Interface ID'
tags: aws, serverless, javascript, tutorial
---

## TL;DR

In this article, we will learn how to assign a static IP to a Lambda for free using the AWS CDK 💸

## The problem with Lambdas and static IPs

It's actually easy to assign a static IP to a Lambda, as shown by [this article][nat-static-ip].

❌ The only caveat is that we need to use a NAT Gateway and it is quite expensive ($30/month).

A few weeks ago I stumbled across this [great article by Yan Cui][bypassing-natgw]. He explains how to outsmart AWS and exploit the resources they create in the background for us.

Basically :

- every time you create a Lambda in a vpc, AWS creates an ENI (Elastic Network Interface)
- we can programmatically extract the ENI ID
- we can then attach an Elastic IP to the corresponding ENI

The idea in Yan's article is amazing but the Infrastructure as Code implementation is a bit complex. **Here is how I was able to do it using the AWS CDK.**

## Deploying a Lambda with a static IP using ENIs

### Step 1: Create a Lambda in a VPC

First, we need to create a Lambda in a VPC. It should be in a public subnet so that it can access the internet.

```typescript
import * as cdk from 'aws-cdk-lib';

const vpc = new cdk.aws_ec2.Vpc(this, 'Vpc', {
  natGateways: 0,
});

// we'll see later why we need this
const sg = new cdk.aws_ec2.SecurityGroup(this, 'SecurityGroup', { vpc });

const func = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'TestFunc', {
  vpc,
  // cdk wants to make sure that the lambda can access the internet
  allowPublicSubnet: true,
  // the lambda needs to be in a public subnet
  vpcSubnets: { subnets: vpc.publicSubnets },
  securityGroups: [sg],
  // ...
});
```

If your lambda already exists, make sure you move it to a public subnet.

### Step 2: Extract the Elastic Network Interface ID

Now that we have a Lambda in a VPC, we need to find the ENI ID aws created automatically.

Let's try like this

```typescript
const eni = func.connections.securityGroups[0].node.defaultChild as cdk.aws_ec2.CfnNetworkInterface;
```

Well actually, it doesn't work. 🙀

The ENI is not created yet when the CDK is executed. The resource is created by the AWS Cloudformation service when the lambda is deployed. We need to find a way to extract the ENI ID after the lambda has been created during the rest of the deployment.

The solution is to use a custom resource. It allows us to execute a lambda during the deployment, and fetch the ENI ID using the AWS SDK. We can do that easily using the [`AwsCustomResource`][aws-custom-resource] construct.

```typescript
const cr = new cdk.custom_resources.AwsCustomResource(this, 'customResource', {
  onCreate: {
    physicalResourceId: cdk.custom_resources.PhysicalResourceId.of(
      // adds a dependency on the security group and the subnet
      `${sg.securityGroupId}-${vpc.publicSubnets[0].subnetId}-CustomResource`,
    ),
    service: 'EC2',
    action: 'describeNetworkInterfaces',
    parameters: {
      Filters: [
        { Name: 'interface-type', Values: ['lambda'] },
        { Name: 'group-id', Values: [sg.securityGroupId] },
        { Name: 'subnet-id', Values: [vpc.publicSubnets[0].subnetId] },
      ],
    },
  },
  policy: cdk.custom_resources.AwsCustomResourcePolicy.fromSdkCalls({
    resources: cdk.custom_resources.AwsCustomResourcePolicy.ANY_RESOURCE,
  }),
});
// adds a dependency on the lambda function
cr.node.addDependency(func);

// we can now extract the ENI ID
const eniId = cr.getResponseField('NetworkInterfaces.0.NetworkInterfaceId');
```

In the snippet above we are calling [`describeNetworkInterfaces`][aws-sdk-describe-eni] function of the AWS SDK. We only keep the ENIs attached to a Lambda, in my security group and my public subnet. Adding our lambda as a dependency ensures that the custom resource is executed after the lambda is created.

### Step 3: Attach an Elastic IP to the ENI - (the easiest one)

Now that we have the ENI ID, we can attach an Elastic IP to it

```typescript
// const eniId = cr.getResponseField("NetworkInterfaces.0.NetworkInterfaceId");

const eip =  new cdk.aws_ec2.CfnEIP(subnet, "EIP", { domain: "vpc" });
new cdk.aws_ec2.CfnEIPAssociation(this, "EIPAssociation", {
  networkInterfaceId: eniId
  allocationId: eip.attrAllocationId,
});
```

### Step 4: Let's put it all together

We now have all the pieces we need to create a lambda with a static IP. We were able to do it for a single subnet, we just need to repeat the process for all the public subnets our lambda needs to be in.

You can find the full code [here][free-static-ip-github].

## Key takeaways

- We can use attach a static IP to a Lambda for FREE using ENIs 💸
- We can use the [`AwsCustomResource`][aws-custom-resource] construct to perform AWS SDK calls during the deployment

Make sure you enjoy your free static IP while it lasts. Elastic IPs will cost $3/month as of January 2024.

Although this is great for reducing your costs, you should not use it for production environments. AWS might change the way they manage ENIs in the future and break your infrastructure. Furthermore, [if you're Lambda is not used in a while AWS will delete the ENI][aws-heni-deletion]. To avoid this you can setup a cron job to keep the ENI.

[nat-static-ip]: https://dev.to/slsbytheodo/deploying-a-lambda-with-a-static-ip-has-never-been-so-simple-5dke
[bypassing-natgw]: https://theburningmonk.com/2023/09/static-ip-for-lambda-ingress-egress-and-bypassing-the-dreaded-nat-gateway/
[aws-custom-resource]: https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.custom_resources.AwsCustomResource.html
[aws-sdk-describe-eni]: https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/ec2/command/DescribeNetworkInterfacesCommand/
[free-static-ip-github]: https://github.com/guillaumeduboc/free-static-ip/blob/main/lib/free-static-ip-stack.ts
[aws-heni-deletion]: https://docs.aws.amazon.com/lambda/latest/dg/foundation-networking.html#:~:text=If%20a%20Lambda%20function%20remains%20idle%20for%20consecutive%20weeks%2C%20Lambda%20reclaims%20the%20unused%20Hyperplane%20ENIs
