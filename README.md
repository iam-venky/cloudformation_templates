A virtual network in the AWS cloud
When you provision a service within a public cloud, it is effectively open to the world and can be at risk of attacks from the internet. In order to secure resources against attacks from the outside, you lock them within a VPC. VPC restricts what sort of traffic, IP addresses and also the users that can access your resources. In other words, VPC is your network but in the cloud.

Just to be clear, AWS has already created a default VPC for you to use so that you don't have to create and configure your own VPC. But, by default its subnet in each AZ is a public subnet, because the main route table sends the subnet's traffic that is destined for the internet to the internet gateway.

This tutorial walks through how to create a fully functional custom VPC from scratch with a pair of public and private subnets spread across two AZs using AWS CloudFormation. Here are core components that we are going to create:

a VPC
an Internet Gateway
a custom route table for all public subnets
two public subnets spread across two AZs
two NAT Gateways attached to public subnets
two custom route tables for each of private subnets
two private subnets spread across two AZs
At the end we will get the following architecture:

Let’s get started!

# Step 1. Create VPC and Internet Gateway
Firstly, we start by creating a VPC with a size /16 IPv4 CIDR block. This provides up to 65,536 private IPv4 addresses. We might hard code the CIDR block for VPC but it’s better to define it as an input parameter so a user can either customize the IP ranges or use a default value.
![image](https://github.com/iam-venky/cloudformation_templates/assets/160997274/633407a1-5016-45d4-bff7-58c3df8225d0)

CidrBlock - the IP address range available to our VPC.

EnableDnsSupport - if set to true, AWS will resolve DNS hostnames to any instance within the VPC’s IP address.

EnableDnsHostnames - if set to true, instances get allocated DNS hostnames by default.

Secondly, we need to create an Internet Gateway. An Internet Gateway is a logical connection between a VPC and the Internet. If there is no Internet Gateway, then there is no connection between the VPC and the Internet.
  # b) Create a Internet Gateway
![image](https://github.com/iam-venky/cloudformation_templates/assets/160997274/eb4f6b2d-f6f5-46d5-bc15-3d053bd30d84)

And finally, let’s attach the Internet Gateway to the VPC
  # c) Attach the Internet Gateway to the VPC
![image](https://github.com/iam-venky/cloudformation_templates/assets/160997274/7c181434-8aeb-43dc-9f43-e6df55cc805e)

With that, we have already built a very basic VPC.

Step 2. Create a public route table and public subnets across two AZs
Placing our instances in multiple AZs increase the availability of our resources and improves the fault tolerance in our applications. If one AZ experiences an outage, you might want to redirect the traffic to the other AZ. Having said that, we are going to follow AWS best practices to create subnets - each in a different AZ.

Firstly, let’s create a custom route table for all public subnets and name it public route table. We need it to control the routing for the public subnets which we are about to create.
  # a) Create a public route table for the VPC (will be public once it is associated with the Internet Gateway)
![image](https://github.com/iam-venky/cloudformation_templates/assets/160997274/da2d602f-e2c3-4de3-b856-2798a92509d5)


Secondly, we need to add a new route to the public route table that points all traffic (0.0.0.0/0) to the Internet Gateway.
  # b) Associate the public route table with the Internet Gateway
![image](https://github.com/iam-venky/cloudformation_templates/assets/160997274/3708c9a3-c462-4843-9a3c-12337b55b529)

DestinationCidrBlock - it specifies which traffic we want this route to be applied to. In this case, we apply it to all traffic using the 0.0.0.0/0 CIDR block.

GatewayId - it specifies where traffic matching the CIDR block should be directed.

Note, when you add a DependsOn attribute to a resource, that resource is created only after the creation of the resource specified in the DependsOn attribute.

Thirdly, once we are done with the public route table, time to create two public subnets with a size /24 IPv4 CIDR block in each of two AZs. This provides up to 256 addresses per subnet, a few of which are reserved for AWS use.

It’s better to define CIDR blocks for both subnets as input parameters so a user can either customize the IP ranges or use a default range.
![image](https://github.com/iam-venky/cloudformation_templates/assets/160997274/07904a93-924d-401d-b55f-fde67439f80c)

Important! The CIDR blocks of two subnets must not overlap with the CIDR block that's associated with the VPC.

Subnets must exist within a VPC, so this is how we associate these two subnets within our VPC:
  ![image](https://github.com/iam-venky/cloudformation_templates/assets/160997274/19ec716a-05fb-451e-8f82-02fe54ab5634)

By setting MapPublicIpOnLaunch to true instances launched into the subnet will be allocated a public IP address by default. This means that any instances in this subnet will be reachable from the Internet via the Internet Gateway attached to the VPC.

And finally, it’s important to associate the route table with both public subnets:
  ![image](https://github.com/iam-venky/cloudformation_templates/assets/160997274/69ed9620-d235-4fdf-8a88-ae88de883b72)

At that point, we are done with public subnets, now let’s create private subnets

Step 3. Create NAT Gateways, private route tables and private subnets across two AZs
We don't want instances within our private subnets to be reachable from the public internet. But we do want these instances to be able to initiate outbound connections. Also we want them to be able to do this without having public IP addresses. That is when NAT gateway comes in handy.

NAT gateway enables instances in a private subnet to connect to the internet or other AWS services, but prevent the internet from initiating a connection with those instances. It will have a public IP address and will be associated with a public subnet. Private instances in private subnets will be able to use this to initiate outbound connections. But the NAT will not allow the reverse, a party on the public internet cannot use the NAT to connect to our private instances.

Thus, we need an Elastic IP (EIP) address and a NAT gateway. Not even one, but two! Because a single NAT Gateway in a single AZ has redundancy within that AZ only, so if there are any zonal issues then instances in other AZs would have no route to the internet. To remain highly available, we need a NAT Gateway in each AZ and a different route table for each AZ.
![image](https://github.com/iam-venky/cloudformation_templates/assets/160997274/9b8709e7-08b1-4a47-a231-f660e0066919)

In the sample above, we created and specified an Elastic IP address to associate with the NAT gateway for each AZ. After creating a NAT gateway, we created and updated the route table associated with each private subnet to point internet-bound traffic to the NAT gateway. This enables instances in our private subnets to communicate with the internet.

And our final step is to create two private subnets with a size /24 IPv4 CIDR block in each of two AZs. And then associate the private route table with the private subnet for each AZ.

![image](https://github.com/iam-venky/cloudformation_templates/assets/160997274/e53ef051-7a71-43f9-b301-89b6fad5dd78)

Optionally, we can add an Output section to get the list of newly created private and public subnets

![image](https://github.com/iam-venky/cloudformation_templates/assets/160997274/ce7d8aba-6182-4543-aefe-114ac852a8bf)


Below, we see our AWS CloudFormation stack with the list of newly created resources:

If you need to understand how to create a stack in AWS Console, please see Part 1 of this series.

Look! We have a stack. At the end we got four subnets, including two public and two private within a newly created VPC:

Summary
If you made it all the way to the end, congrats, and happy CloudFormation construction!

Our VPC template allows to create two public and two private subnets, in different AZs for redundancy using AWS CloudFormation. It builds a private networking environment in which you can securely run AWS resources, along with related networking resources.

The ability to create, modify, and delete a stack of resources through AWS CloudFormation is a powerful example of the Infrastructure as Code concept.
