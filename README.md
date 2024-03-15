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
Parameters:
  paramVpcCIDR:
    Description: Enter the IP range (CIDR notation) for VPC
    Type: String
    Default: 10.192.0.0/16
Resources:
  # a) Create a VPC
  myVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref paramVpcCIDR
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
      - Key: CloudFormationLab
        Value:  !Ref paramUniqueName
CidrBlock - the IP address range available to our VPC
EnableDnsSupport - if set to true, AWS will resolve DNS hostnames to any instance within the VPC’s IP address
EnableDnsHostnames - if set to true, instances get allocated DNS hostnames by default
Secondly, we need to create an Internet Gateway. An Internet Gateway is a logical connection between a VPC and the Internet. If there is no Internet Gateway, then there is no connection between the VPC and the Internet.
  # b) Create a Internet Gateway
  myInternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
      - Key: CloudFormationLab
        Value:  !Ref paramUniqueName
And finally, let’s attach the Internet Gateway to the VPC
  # c) Attach the Internet Gateway to the VPC
  myVPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref myVPC
      InternetGatewayId: !Ref myInternetGateway
With that, we have already built a very basic VPC.

Step 2. Create a public route table and public subnets across two AZs
Placing our instances in multiple AZs increase the availability of our resources and improves the fault tolerance in our applications. If one AZ experiences an outage, you might want to redirect the traffic to the other AZ. Having said that, we are going to follow AWS best practices to create subnets - each in a different AZ.

Firstly, let’s create a custom route table for all public subnets and name it public route table. We need it to control the routing for the public subnets which we are about to create.
  # a) Create a public route table for the VPC (will be public once it is associated with the Internet Gateway)
  myPublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref myVPC
      Tags:
        - Key: Name
          Value: !Ref paramUniqueName
Secondly, we need to add a new route to the public route table that points all traffic (0.0.0.0/0) to the Internet Gateway.
  # b) Associate the public route table with the Internet Gateway
  myPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: myVPCGatewayAttachment
    Properties:
      RouteTableId: !Ref myPublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref myInternetGateway
DestinationCidrBlock - it specifies which traffic we want this route to be applied to. In this case, we apply it to all traffic using the 0.0.0.0/0 CIDR block
GatewayId - it specifies where traffic matching the CIDR block should be directed
Note, when you add a DependsOn attribute to a resource, that resource is created only after the creation of the resource specified in the DependsOn attribute.

Thirdly, once we are done with the public route table, time to create two public subnets with a size /24 IPv4 CIDR block in each of two AZs. This provides up to 256 addresses per subnet, a few of which are reserved for AWS use.

It’s better to define CIDR blocks for both subnets as input parameters so a user can either customize the IP ranges or use a default range.
Parameters:
  # etc
  paramPublicSubnet1CIDR:
    Description: Enter the IP range (CIDR notation)  for the public subnet in AZ A
    Type: String
    Default: 10.192.10.0/24
  paramPublicSubnet2CIDR:
    Description: Enter the IP range (CIDR notation)  for the public subnet in AZ B
    Type: String
    Default: 10.192.11.0/24
Important! The CIDR blocks of two subnets must not overlap with the CIDR block that's associated with the VPC.

Subnets must exist within a VPC, so this is how we associate these two subnets within our VPC:
   # c) Create a public subnet in AZ 1 (will be public once it is associated with public route table)
  myPublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref myVPC
      AvailabilityZone: !Select [ 0, !GetAZs '' ] # AZ 1
      CidrBlock: !Ref paramPublicSubnet1CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Ref paramUniqueName
  # Create a public subnet in AZ 2 (will be public once it is associated with public route table)
  myPublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref myVPC
      AvailabilityZone: !Select [ 1, !GetAZs  '' ] # AZ 2
      CidrBlock: !Ref paramPublicSubnet2CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Ref paramUniqueName
By setting MapPublicIpOnLaunch to true instances launched into the subnet will be allocated a public IP address by default. This means that any instances in this subnet will be reachable from the Internet via the Internet Gateway attached to the VPC.

And finally, it’s important to associate the route table with both public subnets:
  # d) Associate the public route table with the public subnet in AZ 1
  myPublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref myPublicRouteTable
      SubnetId: !Ref myPublicSubnet1
  # Associate the public route table with the public subnet in AZ 2
  myPublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref myPublicRouteTable
      SubnetId: !Ref myPublicSubnet2
At that point, we are done with public subnets, now let’s create private subnets

Step 3. Create NAT Gateways, private route tables and private subnets across two AZs
We don't want instances within our private subnets to be reachable from the public internet. But we do want these instances to be able to initiate outbound connections. Also we want them to be able to do this without having public IP addresses. That is when NAT gateway comes in handy.

NAT gateway enables instances in a private subnet to connect to the internet or other AWS services, but prevent the internet from initiating a connection with those instances. It will have a public IP address and will be associated with a public subnet. Private instances in private subnets will be able to use this to initiate outbound connections. But the NAT will not allow the reverse, a party on the public internet cannot use the NAT to connect to our private instances.

Thus, we need an Elastic IP (EIP) address and a NAT gateway. Not even one, but two! Because a single NAT Gateway in a single AZ has redundancy within that AZ only, so if there are any zonal issues then instances in other AZs would have no route to the internet. To remain highly available, we need a NAT Gateway in each AZ and a different route table for each AZ.
# a) Specify an Elastic IP (EIP) address for a NAT Gateway in AZ 1
  myEIPforNatGateway1:
    Type: AWS::EC2::EIP
    DependsOn: myVPCGatewayAttachment
    Properties:
      Domain: vpc
  # Specify an Elastic IP (EIP) address for a NAT Gateway in AZ 2
  myEIPforNatGateway2:
    Type: AWS::EC2::EIP
    DependsOn: myVPCGatewayAttachment
    Properties:
      Domain: vpc

  # b) Create a NAT Gateway in the public subnet for AZ 1
  myNatGateway1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt myEIPforNatGateway1.AllocationId
      SubnetId: !Ref myPublicSubnet1
   # Create a NAT Gateway in the public subnet for AZ 2
  myNatGateway2:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt myEIPforNatGateway2.AllocationId
      SubnetId: !Ref myPublicSubnet2

  # c) Create a private route table for AZ 1
  myPrivateRouteTable1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref myVPC
      Tags:
        - Key: Name
          Value: !Ref paramUniqueName
  # Create a private route table for AZ 2
  myPrivateRouteTable2:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref myVPC
      Tags:
        - Key: Name
          Value: !Ref paramUniqueName

  # d) Associate the private route table with the Nat Gateway in AZ 1
  myPrivateRouteForAz1:
    Type: AWS::EC2::Route
    DependsOn: myVPCGatewayAttachment
    Properties:
      RouteTableId: !Ref myPrivateRouteTable1
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref myNatGateway1       
  #  Associate the private route table with the Nat Gateway in AZ 2
  myPrivateRouteForAz2:
    Type: AWS::EC2::Route
    DependsOn: myVPCGatewayAttachment
    Properties:
      RouteTableId: !Ref myPrivateRouteTable2
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref myNatGateway2
In the sample above, we created and specified an Elastic IP address to associate with the NAT gateway for each AZ. After creating a NAT gateway, we created and updated the route table associated with each private subnet to point internet-bound traffic to the NAT gateway. This enables instances in our private subnets to communicate with the internet.

And our final step is to create two private subnets with a size /24 IPv4 CIDR block in each of two AZs. And then associate the private route table with the private subnet for each AZ.
Parameters:
  # etc
  paramPrivateSubnet1CIDR:
    Description: Enter the IP range (CIDR notation)  for the private subnet in AZ A
    Type: String
    Default: 10.192.20.0/24
  paramPrivateSubnet2CIDR:
    Description: Enter the IP range (CIDR notation)  for the private subnet in AZ B
    Type: String
    Default: 10.192.21.0/24
  # e) Create a private subnet in AZ 1
  myPrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref myVPC
      AvailabilityZone: !Select [ 0, !GetAZs '' ] # AZ 1
      CidrBlock: !Ref paramPrivateSubnet1CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Ref paramUniqueName
  # Create a private subnet in AZ 2
  myPrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref myVPC
      AvailabilityZone: !Select [ 1, !GetAZs  '' ] # AZ 2
      CidrBlock: !Ref paramPrivateSubnet2CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Ref paramUniqueName

  # f) Associate the private route table with the private subnet in AZ 1
  myPrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref myPrivateRouteTable1
      SubnetId: !Ref myPrivateSubnet1
  #  Associate the private route table with the private subnet in AZ 2
  myPrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref myPrivateRouteTable2
      SubnetId: !Ref myPrivateSubnet2
Optionally, we can add an Output section to get the list of newly created private and public subnets
Outputs:
  outputVPC:
    Description: A reference to the created VPC
    Value: !Ref myVPC
  outputPublicSubnets:
    Description: A list of the public subnets
    Value: !Join [ ",", [ !Ref myPublicSubnet1, !Ref myPublicSubnet2 ]]
  outputPrivateSubnets:
    Description: A list of the private subnets
    Value: !Join [ ",", [ !Ref myPrivateSubnet1, !Ref myPrivateSubnet2 ]]
Here's the GitHub link to the whole template.

Below, we see our AWS CloudFormation stack with the list of newly created resources:

Alt Text

If you need to understand how to create a stack in AWS Console, please see Part 1 of this series.

Look! We have a stack. At the end we got four subnets, including two public and two private within a newly created VPC:

Alt Text

Summary
If you made it all the way to the end, congrats, and happy CloudFormation construction!

Alt Text

Our VPC template allows to create two public and two private subnets, in different AZs for redundancy using AWS CloudFormation. It builds a private networking environment in which you can securely run AWS resources, along with related networking resources.

The ability to create, modify, and delete a stack of resources through AWS CloudFormation is a powerful example of the Infrastructure as Code concept.
