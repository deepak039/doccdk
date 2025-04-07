r'''

Amazon EC2 Construct Library

The aws-cdk-lib/aws-ec2 package contains primitives for setting up networking and
instances.

import aws_cdk.aws_ec2 as ec2

VPC

Most projects need a Virtual Private Cloud to provide security by means of
network partitioning. This is achieved by creating an instance of
Vpc:

vpc = ec2.Vpc(self, "VPC")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

All default constructs require EC2 instances to be launched inside a VPC, so
you should generally start by defining a VPC whenever you need to launch
instances for your project.

Subnet Types

A VPC consists of one or more subnets that instances can be placed into. CDK
distinguishes three different subnet types:

Public (SubnetType.PUBLIC) - public subnets connect directly to the Internet using an
Internet Gateway. If you want your instances to have a public IP address
and be directly reachable from the Internet, you must place them in a
public subnet.

Private with Internet Access (SubnetType.PRIVATE_WITH_EGRESS) - instances in private subnets are not directly routable from the
Internet, and you must provide a way to connect out to the Internet.
By default, a NAT gateway is created in every public subnet for maximum availability. Be
aware that you will be charged for NAT gateways.
Alternatively you can set natGateways:0 and provide your own egress configuration (i.e through Transit Gateway)

Isolated (SubnetType.PRIVATE_ISOLATED) - isolated subnets do not route from or to the Internet, and
as such do not require NAT gateways. They can only connect to or be
connected to from other instances in the same VPC. A default VPC configuration
will not include isolated subnets,

A default VPC configuration will create public and private subnets. However, if
natGateways:0 and subnetConfiguration is undefined, default VPC configuration
will create public and isolated subnets. See Advanced Subnet Configuration
below for information on how to change the default subnet configuration.

Constructs using the VPC will "launch instances" (or more accurately, create
Elastic Network Interfaces) into one or more of the subnets. They all accept
a property called subnetSelection (sometimes called vpcSubnets) to allow
you to select in what subnet to place the ENIs, usually defaulting to
private subnets if the property is omitted.

If you would like to save on the cost of NAT gateways, you can use
isolated subnets instead of private subnets (as described in Advanced
Subnet Configuration). If you need private instances to have
internet connectivity, another option is to reduce the number of NAT gateways
created by setting the natGateways property to a lower value (the default
is one NAT gateway per availability zone). Be aware that this may have
availability implications for your application.

Read more about
subnets.

Control over availability zones

By default, a VPC will spread over at most 3 Availability Zones available to
it. To change the number of Availability Zones that the VPC will spread over,
specify the maxAzs property when defining it.

The number of Availability Zones that are available depends on the region
and account of the Stack containing the VPC. If the region and account are
specified on
the Stack, the CLI will look up the existing Availability
Zones
and get an accurate count. The result of this operation will be written to a file
called cdk.context.json. You must commit this file to source control so
that the lookup values are available in non-privileged environments such
as CI build steps, and to ensure your template builds are repeatable.

If region and account are not specified, the stack
could be deployed anywhere and it will have to make a safe choice, limiting
itself to 2 Availability Zones.

Therefore, to get the VPC to spread over 3 or more availability zones, you
must specify the environment where the stack will be deployed.

You can gain full control over the availability zones selection strategy by overriding the Stack's get availabilityZones() method:

// This example is only available in TypeScript

class MyStack extends Stack {

  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    // ...
  }

  get availabilityZones(): string[] {
    return ['us-west-2a', 'us-west-2b'];
  }

}
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Text
IGNORE_WHEN_COPYING_END

Note that overriding the get availabilityZones() method will override the default behavior for all constructs defined within the Stack.

Choosing subnets for resources

When creating resources that create Elastic Network Interfaces (such as
databases or instances), there is an option to choose which subnets to place
them in. For example, a VPC endpoint by default is placed into a subnet in
every availability zone, but you can override which subnets to use. The property
is typically called one of subnets, vpcSubnets or subnetSelection.

The example below will place the endpoint into two AZs (us-east-1a and us-east-1c),
in Isolated subnets:

# vpc: ec2.Vpc


ec2.InterfaceVpcEndpoint(self, "VPC Endpoint",
    vpc=vpc,
    service=ec2.InterfaceVpcEndpointService("com.amazonaws.vpce.us-east-1.vpce-svc-uuddlrlrbastrtsvc", 443),
    subnets=ec2.SubnetSelection(
        subnet_type=ec2.SubnetType.PRIVATE_ISOLATED,
        availability_zones=["us-east-1a", "us-east-1c"]
    )
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

You can also specify specific subnet objects for granular control:

# vpc: ec2.Vpc
# subnet1: ec2.Subnet
# subnet2: ec2.Subnet


ec2.InterfaceVpcEndpoint(self, "VPC Endpoint",
    vpc=vpc,
    service=ec2.InterfaceVpcEndpointService("com.amazonaws.vpce.us-east-1.vpce-svc-uuddlrlrbastrtsvc", 443),
    subnets=ec2.SubnetSelection(
        subnets=[subnet1, subnet2]
    )
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Which subnets are selected is evaluated as follows:

subnets: if specific subnet objects are supplied, these are selected, and no other
logic is used.

subnetType/subnetGroupName: otherwise, a set of subnets is selected by
supplying either type or name:

subnetType will select all subnets of the given type.

subnetGroupName should be used to distinguish between multiple groups of subnets of
the same type (for example, you may want to separate your application instances and your
RDS instances into two distinct groups of Isolated subnets).

If neither are given, the first available subnet group of a given type that
exists in the VPC will be used, in this order: Private, then Isolated, then Public.
In short: by default ENIs will preferentially be placed in subnets not connected to
the Internet.

availabilityZones/onePerAz: finally, some availability-zone based filtering may be done.
This filtering by availability zones will only be possible if the VPC has been created or
looked up in a non-environment agnostic stack (so account and region have been set and
availability zones have been looked up).

availabilityZones: only the specific subnets from the selected subnet groups that are
in the given availability zones will be returned.

onePerAz: per availability zone, a maximum of one subnet will be returned (Useful for resource
types that do not allow creating two ENIs in the same availability zone).

subnetFilters: additional filtering on subnets using any number of user-provided filters which
extend SubnetFilter.  The following methods on the SubnetFilter class can be used to create
a filter:

byIds: chooses subnets from a list of ids

availabilityZones: chooses subnets in the provided list of availability zones

onePerAz: chooses at most one subnet per availability zone

containsIpAddresses: chooses a subnet which contains any of the listed ip addresses

byCidrMask: chooses subnets that have the provided CIDR netmask

byCidrRanges: chooses subnets which are inside any of the specified CIDR ranges

Using NAT instances

By default, the Vpc construct will create NAT gateways for you, which
are managed by AWS. If you would prefer to use your own managed NAT
instances instead, specify a different value for the natGatewayProvider
property, as follows:

The construct will automatically selects the latest version of Amazon Linux 2023.
If you prefer to use a custom AMI, use machineImage: MachineImage.genericLinux({ ... }) and configure the right AMI ID for the
regions you want to deploy to.

Warning
The NAT instances created using this method will be unmonitored.
They are not part of an Auto Scaling Group,
and if they become unavailable or are terminated for any reason,
will not be restarted or replaced.

By default, the NAT instances will route all traffic. To control what traffic
gets routed, pass a custom value for defaultAllowedTraffic and access the
NatInstanceProvider.connections member after having passed the NAT provider to
the VPC:

# instance_type: ec2.InstanceType


provider = ec2.NatProvider.instance_v2(
    instance_type=instance_type,
    default_allowed_traffic=ec2.NatTrafficDirection.OUTBOUND_ONLY
)
ec2.Vpc(self, "TheVPC",
    nat_gateway_provider=provider
)
provider.connections.allow_from(ec2.Peer.ipv4("1.2.3.4/8"), ec2.Port.HTTP)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

You can also customize the characteristics of your NAT instances, including their security group,
as well as their initialization scripts:

# bucket: s3.Bucket


user_data = ec2.UserData.for_linux()
user_data.add_commands(
    (SpreadElement ...ec2.NatInstanceProviderV2.DEFAULT_USER_DATA_COMMANDS
      ec2.NatInstanceProviderV2.DEFAULT_USER_DATA_COMMANDS), "echo \"hello world!\" > hello.txt", f"aws s3 cp hello.txt s3://{bucket.bucketName}")

provider = ec2.NatProvider.instance_v2(
    instance_type=ec2.InstanceType("t3.small"),
    credit_specification=ec2.CpuCredits.UNLIMITED,
    default_allowed_traffic=ec2.NatTrafficDirection.NONE
)

vpc = ec2.Vpc(self, "TheVPC",
    nat_gateway_provider=provider,
    nat_gateways=2
)

security_group = ec2.SecurityGroup(self, "SecurityGroup", vpc=vpc)
security_group.add_egress_rule(ec2.Peer.any_ipv4(), ec2.Port.tcp(443))
for gateway in provider.gateway_instances:
    bucket.grant_write(gateway)
    gateway.add_security_group(security_group)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
# Configure the `natGatewayProvider` when defining a Vpc
nat_gateway_provider = ec2.NatProvider.instance(
    instance_type=ec2.InstanceType("t3.small")
)

vpc = ec2.Vpc(self, "MyVpc",
    nat_gateway_provider=nat_gateway_provider,

    # The 'natGateways' parameter now controls the number of NAT instances
    nat_gateways=2
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

The V1 NatProvider.instance construct will use the AWS official NAT instance AMI, which has already
reached EOL on Dec 31, 2023. For more information, see the following blog post:
Amazon Linux AMI end of life.

# instance_type: ec2.InstanceType


provider = ec2.NatProvider.instance(
    instance_type=instance_type,
    default_allowed_traffic=ec2.NatTrafficDirection.OUTBOUND_ONLY
)
ec2.Vpc(self, "TheVPC",
    nat_gateway_provider=provider
)
provider.connections.allow_from(ec2.Peer.ipv4("1.2.3.4/8"), ec2.Port.HTTP)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Associate Public IP Address to NAT Instance

You can choose to associate public IP address to a NAT instance V2 by specifying associatePublicIpAddress
like the following:

nat_gateway_provider = ec2.NatProvider.instance_v2(
    instance_type=ec2.InstanceType("t3.small"),
    associate_public_ip_address=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

In certain scenarios where the public subnet has set mapPublicIpOnLaunch to false, NAT instances does not
get public IP addresses assigned which would result in non-working NAT instance as NAT instance requires a public
IP address to enable outbound internet connectivity. Users can specify associatePublicIpAddress to true to
solve this problem.

Ip Address Management

The VPC spans a supernet IP range, which contains the non-overlapping IPs of its contained subnets. Possible sources for this IP range are:

You specify an IP range directly by specifying a CIDR

You allocate an IP range of a given size automatically from AWS IPAM

By default the Vpc will allocate the 10.0.0.0/16 address range which will be exhaustively spread across all subnets in the subnet configuration. This behavior can be changed by passing an object that implements IIpAddresses to the ipAddress property of a Vpc. See the subsequent sections for the options.

Be aware that if you don't explicitly reserve subnet groups in subnetConfiguration, the address space will be fully allocated! If you predict you may need to add more subnet groups later, add them early on and set reserved: true (see the "Advanced Subnet Configuration" section for more information).

Specifying a CIDR directly

Use IpAddresses.cidr to define a Cidr range for your Vpc directly in code:

from aws_cdk.aws_ec2 import IpAddresses


ec2.Vpc(self, "TheVPC",
    ip_addresses=IpAddresses.cidr("10.0.1.0/20")
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Space will be allocated to subnets in the following order:

First, spaces is allocated for all subnets groups that explicitly have a cidrMask set as part of their configuration (including reserved subnets).

Afterwards, any remaining space is divided evenly between the rest of the subnets (if any).

The argument to IpAddresses.cidr may not be a token, and concrete Cidr values are generated in the synthesized CloudFormation template.

Allocating an IP range from AWS IPAM

Amazon VPC IP Address Manager (IPAM) manages a large IP space, from which chunks can be allocated for use in the Vpc. For information on Amazon VPC IP Address Manager please see the official documentation. An example of allocating from AWS IPAM looks like this:

from aws_cdk.aws_ec2 import IpAddresses

# pool: ec2.CfnIPAMPool


ec2.Vpc(self, "TheVPC",
    ip_addresses=IpAddresses.aws_ipam_allocation(
        ipv4_ipam_pool_id=pool.ref,
        ipv4_netmask_length=18,
        default_subnet_ipv4_netmask_length=24
    )
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

IpAddresses.awsIpamAllocation requires the following:

ipv4IpamPoolId, the id of an IPAM Pool from which the VPC range should be allocated.

ipv4NetmaskLength, the size of the IP range that will be requested from the Pool at deploy time.

defaultSubnetIpv4NetmaskLength, the size of subnets in groups that don't have cidrMask set.

With this method of IP address management, no attempt is made to guess at subnet group sizes or to exhaustively allocate the IP range. All subnet groups must have an explicit cidrMask set as part of their subnet configuration, or defaultSubnetIpv4NetmaskLength must be set for a default size. If not, synthesis will fail and you must provide one or the other.

Dual Stack configuration

To allocate both IPv4 and IPv6 addresses in your VPC, you can configure your VPC to have a dual stack protocol.

ec2.Vpc(self, "DualStackVpc",
    ip_protocol=ec2.IpProtocol.DUAL_STACK
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

By default, a dual stack VPC will create an Amazon provided IPv6 /56 CIDR block associated to the VPC. It will then assign /64 portions of the block to each subnet. For each subnet, auto-assigning an IPv6 address will be enabled, and auto-asigning a public IPv4 address will be disabled. An egress only internet gateway will be created for PRIVATE_WITH_EGRESS subnets, and IPv6 routes will be added for IGWs and EIGWs.

Disabling the auto-assigning of a public IPv4 address by default can avoid the cost of public IPv4 addresses starting 2/1/2024. For use cases that need an IPv4 address, the mapPublicIpOnLaunch property in subnetConfiguration can be set to auto-assign the IPv4 address. Note that private IPv4 address allocation will not be changed.

See Advanced Subnet Configuration for all IPv6 specific properties.

Reserving availability zones

There are situations where the IP space for availability zones will
need to be reserved. This is useful in situations where availability
zones would need to be added after the vpc is originally deployed,
without causing IP renumbering for availability zones subnets. The IP
space for reserving n availability zones can be done by setting the
reservedAzs to n in vpc props, as shown below:

vpc = ec2.Vpc(self, "TheVPC",
    cidr="10.0.0.0/21",
    max_azs=3,
    reserved_azs=1
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

In the example above, the subnets for reserved availability zones is not
actually provisioned but its IP space is still reserved. If, in the future,
new availability zones needs to be provisioned, then we would decrement
the value of reservedAzs and increment the maxAzs or availabilityZones
accordingly. This action would not cause the IP address of subnets to get
renumbered, but rather the IP space that was previously reserved will be
used for the new availability zones subnets.

Advanced Subnet Configuration

If the default VPC configuration (public and private subnets spanning the
size of the VPC) don't suffice for you, you can configure what subnets to
create by specifying the subnetConfiguration property. It allows you
to configure the number and size of all subnets. Specifying an advanced
subnet configuration could look like this:

vpc = ec2.Vpc(self, "TheVPC",
    # 'IpAddresses' configures the IP range and size of the entire VPC.
    # The IP space will be divided based on configuration for the subnets.
    ip_addresses=ec2.IpAddresses.cidr("10.0.0.0/21"),

    # 'maxAzs' configures the maximum number of availability zones to use.
    # If you want to specify the exact availability zones you want the VPC
    # to use, use `availabilityZones` instead.
    max_azs=3,

    # 'subnetConfiguration' specifies the "subnet groups" to create.
    # Every subnet group will have a subnet for each AZ, so this
    # configuration will create `3 groups × 3 AZs = 9` subnets.
    subnet_configuration=[ec2.SubnetConfiguration(
        # 'subnetType' controls Internet access, as described above.
        subnet_type=ec2.SubnetType.PUBLIC,

        # 'name' is used to name this particular subnet group. You will have to
        # use the name for subnet selection if you have more than one subnet
        # group of the same type.
        name="Ingress",

        # 'cidrMask' specifies the IP addresses in the range of of individual
        # subnets in the group. Each of the subnets in this group will contain
        # `2^(32 address bits - 24 subnet bits) - 2 reserved addresses = 254`
        # usable IP addresses.
        #
        # If 'cidrMask' is left out the available address space is evenly
        # divided across the remaining subnet groups.
        cidr_mask=24
    ), ec2.SubnetConfiguration(
        cidr_mask=24,
        name="Application",
        subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
    ), ec2.SubnetConfiguration(
        cidr_mask=28,
        name="Database",
        subnet_type=ec2.SubnetType.PRIVATE_ISOLATED,

        # 'reserved' can be used to reserve IP address space. No resources will
        # be created for this subnet, but the IP range will be kept available for
        # future creation of this subnet, or even for future subdivision.
        reserved=True
    )
    ]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

The example above is one possible configuration, but the user can use the
constructs above to implement many other network configurations.

The Vpc from the above configuration in a Region with three
availability zones will be the following:

Subnet Name	Type	IP Block	AZ	Features
IngressSubnet1	PUBLIC	10.0.0.0/24	#1	NAT Gateway
IngressSubnet2	PUBLIC	10.0.1.0/24	#2	NAT Gateway
IngressSubnet3	PUBLIC	10.0.2.0/24	#3	NAT Gateway
ApplicationSubnet1	PRIVATE	10.0.3.0/24	#1	Route to NAT in IngressSubnet1
ApplicationSubnet2	PRIVATE	10.0.4.0/24	#2	Route to NAT in IngressSubnet2
ApplicationSubnet3	PRIVATE	10.0.5.0/24	#3	Route to NAT in IngressSubnet3
DatabaseSubnet1	ISOLATED	10.0.6.0/28	#1	Only routes within the VPC
DatabaseSubnet2	ISOLATED	10.0.6.16/28	#2	Only routes within the VPC
DatabaseSubnet3	ISOLATED	10.0.6.32/28	#3	Only routes within the VPC
Dual Stack Configurations

Here is a break down of IPv4 and IPv6 specific subnetConfiguration properties in a dual stack VPC:

vpc = ec2.Vpc(self, "TheVPC",
    ip_protocol=ec2.IpProtocol.DUAL_STACK,

    subnet_configuration=[ec2.SubnetConfiguration(
        # general properties
        name="Public",
        subnet_type=ec2.SubnetType.PUBLIC,
        reserved=False,

        # IPv4 specific properties
        map_public_ip_on_launch=True,
        cidr_mask=24,

        # new IPv6 specific property
        ipv6_assign_address_on_creation=True
    )
    ]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

The property mapPublicIpOnLaunch controls if a public IPv4 address will be assigned. This defaults to false for dual stack VPCs to avoid inadvertant costs of having the public address. However, a public IP must be enabled (or otherwise configured with BYOIP or IPAM) in order for services that rely on the address to function.

The ipv6AssignAddressOnCreation property controls the same behavior for the IPv6 address. It defaults to true.

Using IPv6 specific properties in an IPv4 only VPC will result in errors.

Accessing the Internet Gateway

If you need access to the internet gateway, you can get its ID like so:

# vpc: ec2.Vpc


igw_id = vpc.internet_gateway_id
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

For a VPC with only ISOLATED subnets, this value will be undefined.

This is only supported for VPCs created in the stack - currently you're
unable to get the ID for imported VPCs. To do that you'd have to specifically
look up the Internet Gateway by name, which would require knowing the name
beforehand.

This can be useful for configuring routing using a combination of gateways:
for more information see Routing below.

Disabling the creation of the default internet gateway

If you need to control the creation of the internet gateway explicitly,
you can disable the creation of the default one using the createInternetGateway
property:

vpc = ec2.Vpc(self, "VPC",
    create_internet_gateway=False,
    subnet_configuration=[ec2.SubnetConfiguration(
        subnet_type=ec2.SubnetType.PUBLIC,
        name="Public"
    )]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Routing

It's possible to add routes to any subnets using the addRoute() method. If for
example you want an isolated subnet to have a static route via the default
Internet Gateway created for the public subnet - perhaps for routing a VPN
connection - you can do so like this:

vpc = ec2.Vpc(self, "VPC",
    subnet_configuration=[ec2.SubnetConfiguration(
        subnet_type=ec2.SubnetType.PUBLIC,
        name="Public"
    ), ec2.SubnetConfiguration(
        subnet_type=ec2.SubnetType.PRIVATE_ISOLATED,
        name="Isolated"
    )]
)

(vpc.isolated_subnets[0]).add_route("StaticRoute",
    router_id=vpc.internet_gateway_id,
    router_type=ec2.RouterType.GATEWAY,
    destination_cidr_block="8.8.8.8/32"
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Note that we cast to Subnet here because the list of subnets only returns an
ISubnet.

Reserving subnet IP space

There are situations where the IP space for a subnet or number of subnets
will need to be reserved. This is useful in situations where subnets would
need to be added after the vpc is originally deployed, without causing IP
renumbering for existing subnets. The IP space for a subnet may be reserved
by setting the reserved subnetConfiguration property to true, as shown
below:

vpc = ec2.Vpc(self, "TheVPC",
    nat_gateways=1,
    subnet_configuration=[ec2.SubnetConfiguration(
        cidr_mask=26,
        name="Public",
        subnet_type=ec2.SubnetType.PUBLIC
    ), ec2.SubnetConfiguration(
        cidr_mask=26,
        name="Application1",
        subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS
    ), ec2.SubnetConfiguration(
        cidr_mask=26,
        name="Application2",
        subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS,
        reserved=True
    ), ec2.SubnetConfiguration(
        cidr_mask=27,
        name="Database",
        subnet_type=ec2.SubnetType.PRIVATE_ISOLATED
    )
    ]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

In the example above, the subnet for Application2 is not actually provisioned
but its IP space is still reserved. If in the future this subnet needs to be
provisioned, then the reserved: true property should be removed. Reserving
parts of the IP space prevents the other subnets from getting renumbered.

Sharing VPCs between stacks

If you are creating multiple Stacks inside the same CDK application, you
can reuse a VPC defined in one Stack in another by simply passing the VPC
instance around:

#
# Stack1 creates the VPC
#
class Stack1(cdk.Stack):

    def __init__(self, scope, id, *, description=None, env=None, stackName=None, tags=None, notificationArns=None, synthesizer=None, terminationProtection=None, analyticsReporting=None, crossRegionReferences=None, permissionsBoundary=None, suppressTemplateIndentation=None):
        super().__init__(scope, id, description=description, env=env, stackName=stackName, tags=tags, notificationArns=notificationArns, synthesizer=synthesizer, terminationProtection=terminationProtection, analyticsReporting=analyticsReporting, crossRegionReferences=crossRegionReferences, permissionsBoundary=permissionsBoundary, suppressTemplateIndentation=suppressTemplateIndentation)

        self.vpc = ec2.Vpc(self, "VPC")

#
# Stack2 consumes the VPC
#
class Stack2(cdk.Stack):
    def __init__(self, scope, id, *, vpc, description=None, env=None, stackName=None, tags=None, notificationArns=None, synthesizer=None, terminationProtection=None, analyticsReporting=None, crossRegionReferences=None, permissionsBoundary=None, suppressTemplateIndentation=None):
        super().__init__(scope, id, vpc=vpc, description=description, env=env, stackName=stackName, tags=tags, notificationArns=notificationArns, synthesizer=synthesizer, terminationProtection=terminationProtection, analyticsReporting=analyticsReporting, crossRegionReferences=crossRegionReferences, permissionsBoundary=permissionsBoundary, suppressTemplateIndentation=suppressTemplateIndentation)

        # Pass the VPC to a construct that needs it
        ConstructThatTakesAVpc(self, "Construct",
            vpc=vpc
        )

stack1 = Stack1(app, "Stack1")
stack2 = Stack2(app, "Stack2",
    vpc=stack1.vpc
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Importing an existing VPC

If your VPC is created outside your CDK app, you can use Vpc.fromLookup().
The CDK CLI will search for the specified VPC in the the stack's region and
account, and import the subnet configuration. Looking up can be done by VPC
ID, but more flexibly by searching for a specific tag on the VPC.

Subnet types will be determined from the aws-cdk:subnet-type tag on the
subnet if it exists, or the presence of a route to an Internet Gateway
otherwise. Subnet names will be determined from the aws-cdk:subnet-name tag
on the subnet if it exists, or will mirror the subnet type otherwise (i.e.
a public subnet will have the name "Public").

The result of the Vpc.fromLookup() operation will be written to a file
called cdk.context.json. You must commit this file to source control so
that the lookup values are available in non-privileged environments such
as CI build steps, and to ensure your template builds are repeatable.

Here's how Vpc.fromLookup() can be used:

vpc = ec2.Vpc.from_lookup(stack, "VPC",
    # This imports the default VPC but you can also
    # specify a 'vpcName' or 'tags'.
    is_default=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Vpc.fromLookup is the recommended way to import VPCs. If for whatever
reason you do not want to use the context mechanism to look up a VPC at
synthesis time, you can also use Vpc.fromVpcAttributes. This has the
following limitations:

Every subnet group in the VPC must have a subnet in each availability zone
(for example, each AZ must have both a public and private subnet). Asymmetric
VPCs are not supported.

All VpcId, SubnetId, RouteTableId, ... parameters must either be known at
synthesis time, or they must come from deploy-time list parameters whose
deploy-time lengths are known at synthesis time.

Using Vpc.fromVpcAttributes() looks like this:

vpc = ec2.Vpc.from_vpc_attributes(self, "VPC",
    vpc_id="vpc-1234",
    availability_zones=["us-east-1a", "us-east-1b"],

    # Either pass literals for all IDs
    public_subnet_ids=["s-12345", "s-67890"],

    # OR: import a list of known length
    private_subnet_ids=Fn.import_list_value("PrivateSubnetIds", 2),

    # OR: split an imported string to a list of known length
    isolated_subnet_ids=Fn.split(",", ssm.StringParameter.value_for_string_parameter(self, "MyParameter"), 2)
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

For each subnet group the import function accepts optional parameters for subnet
names, route table ids and IPv4 CIDR blocks. When supplied, the length of these
lists are required to match the length of the list of subnet ids, allowing the
lists to be zipped together to form ISubnet instances.

Public subnet group example (for private or isolated subnet groups, use the properties with the respective prefix):

vpc = ec2.Vpc.from_vpc_attributes(self, "VPC",
    vpc_id="vpc-1234",
    availability_zones=["us-east-1a", "us-east-1b", "us-east-1c"],
    public_subnet_ids=["s-12345", "s-34567", "s-56789"],
    public_subnet_names=["Subnet A", "Subnet B", "Subnet C"],
    public_subnet_route_table_ids=["rt-12345", "rt-34567", "rt-56789"],
    public_subnet_ipv4_cidr_blocks=["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

The above example will create an IVpc instance with three public subnets:

| Subnet id | Availability zone | Subnet name | Route table id | IPv4 CIDR   |
| --------- | ----------------- | ----------- | -------------- | ----------- |
| s-12345   | us-east-1a        | Subnet A    | rt-12345       | 10.0.0.0/24 |
| s-34567   | us-east-1b        | Subnet B    | rt-34567       | 10.0.1.0/24 |
| s-56789   | us-east-1c        | Subnet B    | rt-56789       | 10.0.2.0/24 |

Restricting access to the VPC default security group

AWS Security best practices recommend that the VPC default security group should
not allow inbound and outbound
traffic.
When the @aws-cdk/aws-ec2:restrictDefaultSecurityGroup feature flag is set to
true (default for new projects) this will be enabled by default. If you do not
have this feature flag set you can either set the feature flag or you can set
the restrictDefaultSecurityGroup property to true.

ec2.Vpc(self, "VPC",
    restrict_default_security_group=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

If you set this property to true and then later remove it or set it to false
the default ingress/egress will be restored on the default security group.

Allowing Connections

In AWS, all network traffic in and out of Elastic Network Interfaces (ENIs)
is controlled by Security Groups. You can think of Security Groups as a
firewall with a set of rules. By default, Security Groups allow no incoming
(ingress) traffic and all outgoing (egress) traffic. You can add ingress rules
to them to allow incoming traffic streams. To exert fine-grained control over
egress traffic, set allowAllOutbound: false on the SecurityGroup, after
which you can add egress traffic rules.

You can manipulate Security Groups directly:

my_security_group = ec2.SecurityGroup(self, "SecurityGroup",
    vpc=vpc,
    description="Allow ssh access to ec2 instances",
    allow_all_outbound=True
)
my_security_group.add_ingress_rule(ec2.Peer.any_ipv4(), ec2.Port.tcp(22), "allow ssh access from the world")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

All constructs that create ENIs on your behalf (typically constructs that create
EC2 instances or other VPC-connected resources) will all have security groups
automatically assigned. Those constructs have an attribute called
connections, which is an object that makes it convenient to update the
security groups. If you want to allow connections between two constructs that
have security groups, you have to add an Egress rule to one Security Group,
and an Ingress rule to the other. The connections object will automatically
take care of this for you:

# load_balancer: elbv2.ApplicationLoadBalancer
# app_fleet: autoscaling.AutoScalingGroup
# db_fleet: autoscaling.AutoScalingGroup


# Allow connections from anywhere
load_balancer.connections.allow_from_any_ipv4(ec2.Port.HTTPS, "Allow inbound HTTPS")

# The same, but an explicit IP address
load_balancer.connections.allow_from(ec2.Peer.ipv4("1.2.3.4/32"), ec2.Port.HTTPS, "Allow inbound HTTPS")

# Allow connection between AutoScalingGroups
app_fleet.connections.allow_to(db_fleet, ec2.Port.HTTPS, "App can call database")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Connection Peers

There are various classes that implement the connection peer part:

# app_fleet: autoscaling.AutoScalingGroup
# db_fleet: autoscaling.AutoScalingGroup


# Simple connection peers
peer = ec2.Peer.ipv4("10.0.0.0/16")
peer = ec2.Peer.any_ipv4()
peer = ec2.Peer.ipv6("::0/0")
peer = ec2.Peer.any_ipv6()
peer = ec2.Peer.prefix_list("pl-12345")
app_fleet.connections.allow_to(peer, ec2.Port.HTTPS, "Allow outbound HTTPS")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Any object that has a security group can itself be used as a connection peer:

# fleet1: autoscaling.AutoScalingGroup
# fleet2: autoscaling.AutoScalingGroup
# app_fleet: autoscaling.AutoScalingGroup


# These automatically create appropriate ingress and egress rules in both security groups
fleet1.connections.allow_to(fleet2, ec2.Port.HTTP, "Allow between fleets")

app_fleet.connections.allow_from_any_ipv4(ec2.Port.HTTP, "Allow from load balancer")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Port Ranges

The connections that are allowed are specified by port ranges. A number of classes provide
the connection specifier:

ec2.Port.tcp(80)
ec2.Port.HTTPS
ec2.Port.tcp_range(60000, 65535)
ec2.Port.all_tcp()
ec2.Port.all_icmp()
ec2.Port.all_icmp_v6()
ec2.Port.all_traffic()
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

NOTE: Not all protocols have corresponding helper methods. In the absence of a helper method,
you can instantiate Port yourself with your own settings. You are also welcome to contribute
new helper methods.

Default Ports

Some Constructs have default ports associated with them. For example, the
listener of a load balancer does (it's the public port), or instances of an
RDS database (it's the port the database is accepting connections on).

If the object you're calling the peering method on has a default port associated with it, you can call
allowDefaultPortFrom() and omit the port specifier. If the argument has an associated default port, call
allowDefaultPortTo().

For example:

# listener: elbv2.ApplicationListener
# app_fleet: autoscaling.AutoScalingGroup
# rds_database: rds.DatabaseCluster


# Port implicit in listener
listener.connections.allow_default_port_from_any_ipv4("Allow public")

# Port implicit in peer
app_fleet.connections.allow_default_port_to(rds_database, "Fleet can access database")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Security group rules

By default, security group wills be added inline to the security group in the output cloud formation
template, if applicable.  This includes any static rules by ip address and port range.  This
optimization helps to minimize the size of the template.

In some environments this is not desirable, for example if your security group access is controlled
via tags. You can disable inline rules per security group or globally via the context key
@aws-cdk/aws-ec2.securityGroupDisableInlineRules.

my_security_group_without_inline_rules = ec2.SecurityGroup(self, "SecurityGroup",
    vpc=vpc,
    description="Allow ssh access to ec2 instances",
    allow_all_outbound=True,
    disable_inline_rules=True
)
# This will add the rule as an external cloud formation construct
my_security_group_without_inline_rules.add_ingress_rule(ec2.Peer.any_ipv4(), ec2.Port.SSH, "allow ssh access from the world")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Importing an existing security group

If you know the ID and the configuration of the security group to import, you can use SecurityGroup.fromSecurityGroupId:

sg = ec2.SecurityGroup.from_security_group_id(self, "SecurityGroupImport", "sg-1234",
    allow_all_outbound=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Alternatively, use lookup methods to import security groups if you do not know the ID or the configuration details. Method SecurityGroup.fromLookupByName looks up a security group if the security group ID is unknown.

sg = ec2.SecurityGroup.from_lookup_by_name(self, "SecurityGroupLookup", "security-group-name", vpc)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

If the security group ID is known and configuration details are unknown, use method SecurityGroup.fromLookupById instead. This method will lookup property allowAllOutbound from the current configuration of the security group.

sg = ec2.SecurityGroup.from_lookup_by_id(self, "SecurityGroupLookup", "sg-1234")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

The result of SecurityGroup.fromLookupByName and SecurityGroup.fromLookupById operations will be written to a file called cdk.context.json. You must commit this file to source control so that the lookup values are available in non-privileged environments such as CI build steps, and to ensure your template builds are repeatable.

Cross Stack Connections

If you are attempting to add a connection from a peer in one stack to a peer in a different stack, sometimes it is necessary to ensure that you are making the connection in
a specific stack in order to avoid a cyclic reference. If there are no other dependencies between stacks then it will not matter in which stack you make
the connection, but if there are existing dependencies (i.e. stack1 already depends on stack2), then it is important to make the connection in the dependent stack (i.e. stack1).

Whenever you make a connections function call, the ingress and egress security group rules will be added to the stack that the calling object exists in.
So if you are doing something like peer1.connections.allowFrom(peer2), then the security group rules (both ingress and egress) will be created in peer1's Stack.

As an example, if we wanted to allow a connection from a security group in one stack (egress) to a security group in a different stack (ingress),
we would make the connection like:

If Stack1 depends on Stack2

# Stack 1
# stack1: Stack
# stack2: Stack


sg1 = ec2.SecurityGroup(stack1, "SG1",
    allow_all_outbound=False,  # if this is `true` then no egress rule will be created
    vpc=vpc
)

# Stack 2
sg2 = ec2.SecurityGroup(stack2, "SG2",
    allow_all_outbound=False,  # if this is `true` then no egress rule will be created
    vpc=vpc
)

# `connections.allowTo` on `sg1` since we want the
# rules to be created in Stack1
sg1.connections.allow_to(sg2, ec2.Port.tcp(3333))
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

In this case both the Ingress Rule for sg2 and the Egress Rule for sg1 will both be created
in Stack 1 which avoids the cyclic reference.

If Stack2 depends on Stack1

# Stack 1
# stack1: Stack
# stack2: Stack


sg1 = ec2.SecurityGroup(stack1, "SG1",
    allow_all_outbound=False,  # if this is `true` then no egress rule will be created
    vpc=vpc
)

# Stack 2
sg2 = ec2.SecurityGroup(stack2, "SG2",
    allow_all_outbound=False,  # if this is `true` then no egress rule will be created
    vpc=vpc
)

# `connections.allowFrom` on `sg2` since we want the
# rules to be created in Stack2
sg2.connections.allow_from(sg1, ec2.Port.tcp(3333))
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

In this case both the Ingress Rule for sg2 and the Egress Rule for sg1 will both be created
in Stack 2 which avoids the cyclic reference.

Machine Images (AMIs)

AMIs control the OS that gets launched when you start your EC2 instance. The EC2
library contains constructs to select the AMI you want to use.

Depending on the type of AMI, you select it a different way. Here are some
examples of images you might want to use:

# Pick the right Amazon Linux edition. All arguments shown are optional
# and will default to these values when omitted.
amzn_linux = ec2.MachineImage.latest_amazon_linux(
    generation=ec2.AmazonLinuxGeneration.AMAZON_LINUX,
    edition=ec2.AmazonLinuxEdition.STANDARD,
    virtualization=ec2.AmazonLinuxVirt.HVM,
    storage=ec2.AmazonLinuxStorage.GENERAL_PURPOSE,
    cpu_type=ec2.AmazonLinuxCpuType.X86_64
)

# Pick a Windows edition to use
windows = ec2.MachineImage.latest_windows(ec2.WindowsVersion.WINDOWS_SERVER_2019_ENGLISH_FULL_BASE)

# Read AMI id from SSM parameter store
ssm = ec2.MachineImage.from_ssm_parameter("/my/ami", os=ec2.OperatingSystemType.LINUX)

# Look up the most recent image matching a set of AMI filters.
# In this case, look up the NAT instance AMI, by using a wildcard
# in the 'name' field:
nat_ami = ec2.MachineImage.lookup(
    name="amzn-ami-vpc-nat-*",
    owners=["amazon"]
)

# For other custom (Linux) images, instantiate a `GenericLinuxImage` with
# a map giving the AMI to in for each region:
linux = ec2.MachineImage.generic_linux({
    "us-east-1": "ami-97785bed",
    "eu-west-1": "ami-12345678"
})

# For other custom (Windows) images, instantiate a `GenericWindowsImage` with
# a map giving the AMI to in for each region:
generic_windows = ec2.MachineImage.generic_windows({
    "us-east-1": "ami-97785bed",
    "eu-west-1": "ami-12345678"
})
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

NOTE: The AMIs selected by MachineImage.lookup() will be cached in
cdk.context.json, so that your AutoScalingGroup instances aren't replaced while
you are making unrelated changes to your CDK app.

To query for the latest AMI again, remove the relevant cache entry from
cdk.context.json, or use the cdk context command. For more information, see
Runtime Context in the CDK
developer guide.

MachineImage.genericLinux(), MachineImage.genericWindows() will use CfnMapping in
an agnostic stack.

Special VPC configurations
VPN connections to a VPC

Create your VPC with VPN connections by specifying the vpnConnections props (keys are construct ids):

from aws_cdk import SecretValue


vpc = ec2.Vpc(self, "MyVpc",
    vpn_connections={
        "dynamic": ec2.VpnConnectionOptions( # Dynamic routing (BGP)
            ip="1.2.3.4",
            tunnel_options=[ec2.VpnTunnelOption(
                pre_shared_key_secret=SecretValue.unsafe_plain_text("secretkey1234")
            ), ec2.VpnTunnelOption(
                pre_shared_key_secret=SecretValue.unsafe_plain_text("secretkey5678")
            )
            ]),
        "static": ec2.VpnConnectionOptions( # Static routing
            ip="4.5.6.7",
            static_routes=["192.168.10.0/24", "192.168.20.0/24"
            ])
    }
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

To create a VPC that can accept VPN connections, set vpnGateway to true:

vpc = ec2.Vpc(self, "MyVpc",
    vpn_gateway=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

VPN connections can then be added:

vpc.add_vpn_connection("Dynamic",
    ip="1.2.3.4"
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

By default, routes will be propagated on the route tables associated with the private subnets. If no
private subnets exist, isolated subnets are used. If no isolated subnets exist, public subnets are
used. Use the Vpc property vpnRoutePropagation to customize this behavior.

VPN connections expose metrics (cloudwatch.Metric) across all tunnels in the account/region and per connection:

# Across all tunnels in the account/region
all_data_out = ec2.VpnConnection.metric_all_tunnel_data_out()

# For a specific vpn connection
vpn_connection = vpc.add_vpn_connection("Dynamic",
    ip="1.2.3.4"
)
state = vpn_connection.metric_tunnel_state()
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
VPC endpoints

A VPC endpoint enables you to privately connect your VPC to supported AWS services and VPC endpoint services powered by PrivateLink without requiring an internet gateway, NAT device, VPN connection, or AWS Direct Connect connection. Instances in your VPC do not require public IP addresses to communicate with resources in the service. Traffic between your VPC and the other service does not leave the Amazon network.

Endpoints are virtual devices. They are horizontally scaled, redundant, and highly available VPC components that allow communication between instances in your VPC and services without imposing availability risks or bandwidth constraints on your network traffic.

# Add gateway endpoints when creating the VPC
vpc = ec2.Vpc(self, "MyVpc",
    gateway_endpoints={
        "S3": cdk.aws_ec2.GatewayVpcEndpointOptions(
            service=ec2.GatewayVpcEndpointAwsService.S3
        )
    }
)

# Alternatively gateway endpoints can be added on the VPC
dynamo_db_endpoint = vpc.add_gateway_endpoint("DynamoDbEndpoint",
    service=ec2.GatewayVpcEndpointAwsService.DYNAMODB
)

# This allows to customize the endpoint policy
dynamo_db_endpoint.add_to_policy(
    iam.PolicyStatement( # Restrict to listing and describing tables
        principals=[iam.AnyPrincipal()],
        actions=["dynamodb:DescribeTable", "dynamodb:ListTables"],
        resources=["*"]))

# Add an interface endpoint
vpc.add_interface_endpoint("EcrDockerEndpoint",
    service=ec2.InterfaceVpcEndpointAwsService.ECR_DOCKER
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

By default, CDK will place a VPC endpoint in one subnet per AZ. If you wish to override the AZs CDK places the VPC endpoint in,
use the subnets parameter as follows:

# vpc: ec2.Vpc


ec2.InterfaceVpcEndpoint(self, "VPC Endpoint",
    vpc=vpc,
    service=ec2.InterfaceVpcEndpointService("com.amazonaws.vpce.us-east-1.vpce-svc-uuddlrlrbastrtsvc", 443),
    # Choose which availability zones to place the VPC endpoint in, based on
    # available AZs
    subnets=ec2.SubnetSelection(
        availability_zones=["us-east-1a", "us-east-1c"]
    )
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Per the AWS documentation, not all
VPC endpoint services are available in all AZs. If you specify the parameter lookupSupportedAzs, CDK attempts to discover which
AZs an endpoint service is available in, and will ensure the VPC endpoint is not placed in a subnet that doesn't match those AZs.
These AZs will be stored in cdk.context.json.

# vpc: ec2.Vpc


ec2.InterfaceVpcEndpoint(self, "VPC Endpoint",
    vpc=vpc,
    service=ec2.InterfaceVpcEndpointService("com.amazonaws.vpce.us-east-1.vpce-svc-uuddlrlrbastrtsvc", 443),
    # Choose which availability zones to place the VPC endpoint in, based on
    # available AZs
    lookup_supported_azs=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Pre-defined AWS services are defined in the InterfaceVpcEndpointAwsService class, and can be used to
create VPC endpoints without having to configure name, ports, etc. For example, a Keyspaces endpoint can be created for
use in your VPC:

# vpc: ec2.Vpc


ec2.InterfaceVpcEndpoint(self, "VPC Endpoint",
    vpc=vpc,
    service=ec2.InterfaceVpcEndpointAwsService.KEYSPACES
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Security groups for interface VPC endpoints

By default, interface VPC endpoints create a new security group and all traffic to the endpoint from within the VPC will be automatically allowed.

Use the connections object to allow other traffic to flow to the endpoint:

# my_endpoint: ec2.InterfaceVpcEndpoint


my_endpoint.connections.allow_default_port_from_any_ipv4()
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Alternatively, existing security groups can be used by specifying the securityGroups prop.

VPC endpoint services

A VPC endpoint service enables you to expose a Network Load Balancer(s) as a provider service to consumers, who connect to your service over a VPC endpoint. You can restrict access to your service via allowed principals (anything that extends ArnPrincipal), and require that new connections be manually accepted. You can also enable Contributor Insight rules.

# network_load_balancer1: elbv2.NetworkLoadBalancer
# network_load_balancer2: elbv2.NetworkLoadBalancer


ec2.VpcEndpointService(self, "EndpointService",
    vpc_endpoint_service_load_balancers=[network_load_balancer1, network_load_balancer2],
    acceptance_required=True,
    allowed_principals=[iam.ArnPrincipal("arn:aws:iam::123456789012:root")],
    contributor_insights=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

You can also include a service principal in the allowedPrincipals property by specifying it as a parameter to the  ArnPrincipal constructor.
The resulting VPC endpoint will have an allowlisted principal of type Service, instead of Arn for that item in the list.

# network_load_balancer: elbv2.NetworkLoadBalancer


ec2.VpcEndpointService(self, "EndpointService",
    vpc_endpoint_service_load_balancers=[network_load_balancer],
    allowed_principals=[iam.ArnPrincipal("ec2.amazonaws.com")]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Endpoint services support private DNS, which makes it easier for clients to connect to your service by automatically setting up DNS in their VPC.
You can enable private DNS on an endpoint service like so:

from aws_cdk.aws_route53 import PublicHostedZone, VpcEndpointServiceDomainName
# zone: PublicHostedZone
# vpces: ec2.VpcEndpointService


VpcEndpointServiceDomainName(self, "EndpointDomain",
    endpoint_service=vpces,
    domain_name="my-stuff.aws-cdk.dev",
    public_hosted_zone=zone
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Note: The domain name must be owned (registered through Route53) by the account the endpoint service is in, or delegated to the account.
The VpcEndpointServiceDomainName will handle the AWS side of domain verification, the process for which can be found
here

Client VPN endpoint

AWS Client VPN is a managed client-based VPN service that enables you to securely access your AWS
resources and resources in your on-premises network. With Client VPN, you can access your resources
from any location using an OpenVPN-based VPN client.

Use the addClientVpnEndpoint() method to add a client VPN endpoint to a VPC:

vpc.add_client_vpn_endpoint("Endpoint",
    cidr="10.100.0.0/16",
    server_certificate_arn="arn:aws:acm:us-east-1:123456789012:certificate/server-certificate-id",
    # Mutual authentication
    client_certificate_arn="arn:aws:acm:us-east-1:123456789012:certificate/client-certificate-id",
    # User-based authentication
    user_based_authentication=ec2.ClientVpnUserBasedAuthentication.federated(saml_provider)
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

The endpoint must use at least one authentication method:

Mutual authentication with a client certificate

User-based authentication (directory or federated)

If user-based authentication is used, the self-service portal URL
is made available via a CloudFormation output.

By default, a new security group is created, and logging is enabled. Moreover, a rule to
authorize all users to the VPC CIDR is created.

To customize authorization rules, set the authorizeAllUsersToVpcCidr prop to false
and use addAuthorizationRule():

endpoint = vpc.add_client_vpn_endpoint("Endpoint",
    cidr="10.100.0.0/16",
    server_certificate_arn="arn:aws:acm:us-east-1:123456789012:certificate/server-certificate-id",
    user_based_authentication=ec2.ClientVpnUserBasedAuthentication.federated(saml_provider),
    authorize_all_users_to_vpc_cidr=False
)

endpoint.add_authorization_rule("Rule",
    cidr="10.0.10.0/32",
    group_id="group-id"
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Use addRoute() to configure network routes:

endpoint = vpc.add_client_vpn_endpoint("Endpoint",
    cidr="10.100.0.0/16",
    server_certificate_arn="arn:aws:acm:us-east-1:123456789012:certificate/server-certificate-id",
    user_based_authentication=ec2.ClientVpnUserBasedAuthentication.federated(saml_provider)
)

# Client-to-client access
endpoint.add_route("Route",
    cidr="10.100.0.0/16",
    target=ec2.ClientVpnRouteTarget.local()
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Use the connections object of the endpoint to allow traffic to other security groups.

Instances

You can use the Instance class to start up a single EC2 instance. For production setups, we recommend
you use an AutoScalingGroup from the aws-autoscaling module instead, as AutoScalingGroups will take
care of restarting your instance if it ever fails.

# vpc: ec2.Vpc
# instance_type: ec2.InstanceType


# Amazon Linux 2
ec2.Instance(self, "Instance2",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=ec2.MachineImage.latest_amazon_linux2()
)

# Amazon Linux 2 with kernel 5.x
ec2.Instance(self, "Instance3",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=ec2.MachineImage.latest_amazon_linux2(
        kernel=ec2.AmazonLinux2Kernel.KERNEL_5_10
    )
)

# Amazon Linux 2023
ec2.Instance(self, "Instance4",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=ec2.MachineImage.latest_amazon_linux2023()
)

# Graviton 3 Processor
ec2.Instance(self, "Instance5",
    vpc=vpc,
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.C7G, ec2.InstanceSize.LARGE),
    machine_image=ec2.MachineImage.latest_amazon_linux2023(
        cpu_type=ec2.AmazonLinuxCpuType.ARM_64
    )
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Latest Amazon Linux Images

Rather than specifying a specific AMI ID to use, it is possible to specify a SSM
Parameter that contains the AMI ID. AWS publishes a set of public parameters
that contain the latest Amazon Linux AMIs. To make it easier to query a
particular image parameter, the CDK provides a couple of constructs AmazonLinux2ImageSsmParameter,
AmazonLinux2022ImageSsmParameter, & AmazonLinux2023SsmParameter. For example
to use the latest al2023 image:

# vpc: ec2.Vpc


ec2.Instance(self, "LatestAl2023",
    vpc=vpc,
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.C7G, ec2.InstanceSize.LARGE),
    machine_image=ec2.MachineImage.latest_amazon_linux2023()
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Warning
Since this retrieves the value from an SSM parameter at deployment time, the
value will be resolved each time the stack is deployed. This means that if
the parameter contains a different value on your next deployment, the instance
will be replaced.

It is also possible to perform the lookup once at synthesis time and then cache
the value in CDK context. This way the value will not change on future
deployments unless you manually refresh the context.

# vpc: ec2.Vpc


ec2.Instance(self, "LatestAl2023",
    vpc=vpc,
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.C7G, ec2.InstanceSize.LARGE),
    machine_image=ec2.MachineImage.latest_amazon_linux2023(
        cached_in_context=True
    )
)

# or
ec2.Instance(self, "LatestAl2023",
    vpc=vpc,
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.C7G, ec2.InstanceSize.LARGE),
    # context cache is turned on by default
    machine_image=ec2.AmazonLinux2023ImageSsmParameter()
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Kernel Versions

Each Amazon Linux AMI uses a specific kernel version. Most Amazon Linux
generations come with an AMI using the "default" kernel and then 1 or more
AMIs using a specific kernel version, which may or may not be different from the
default kernel version.

For example, Amazon Linux 2 has two different AMIs available from the SSM
parameters.

/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-ebs

This is the "default" kernel which uses kernel-4.14

/aws/service/ami-amazon-linux-latest/amzn2-ami-kernel-5.10-hvm-x86_64-ebs

If a new Amazon Linux generation AMI is published with a new kernel version,
then a new SSM parameter will be created with the new version
(e.g. /aws/service/ami-amazon-linux-latest/amzn2-ami-kernel-5.15-hvm-x86_64-ebs),
but the "default" AMI may or may not be updated.

If you would like to make sure you always have the latest kernel version, then
either specify the specific latest kernel version or opt-in to using the CDK
latest kernel version.

# vpc: ec2.Vpc


ec2.Instance(self, "LatestAl2023",
    vpc=vpc,
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.C7G, ec2.InstanceSize.LARGE),
    # context cache is turned on by default
    machine_image=ec2.AmazonLinux2023ImageSsmParameter(
        kernel=ec2.AmazonLinux2023Kernel.KERNEL_6_1
    )
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

CDK managed latest

# vpc: ec2.Vpc


ec2.Instance(self, "LatestAl2023",
    vpc=vpc,
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.C7G, ec2.InstanceSize.LARGE),
    # context cache is turned on by default
    machine_image=ec2.AmazonLinux2023ImageSsmParameter(
        kernel=ec2.AmazonLinux2023Kernel.CDK_LATEST
    )
)

# or

ec2.Instance(self, "LatestAl2023",
    vpc=vpc,
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.C7G, ec2.InstanceSize.LARGE),
    machine_image=ec2.MachineImage.latest_amazon_linux2023()
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

When using the CDK managed latest version, when a new kernel version is made
available the LATEST will be updated to point to the new kernel version. You
then would be required to update the newest CDK version for it to take effect.

Configuring Instances using CloudFormation Init (cfn-init)

CloudFormation Init allows you to configure your instances by writing files to them, installing software
packages, starting services and running arbitrary commands. By default, if any of the instance setup
commands throw an error; the deployment will fail and roll back to the previously known good state.
The following documentation also applies to AutoScalingGroups.

For the full set of capabilities of this system, see the documentation for
AWS::CloudFormation::Init.
Here is an example of applying some configuration to an instance:

# vpc: ec2.Vpc
# instance_type: ec2.InstanceType
# machine_image: ec2.IMachineImage


ec2.Instance(self, "Instance",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=machine_image,

    # Showing the most complex setup, if you have simpler requirements
    # you can use `CloudFormationInit.fromElements()`.
    init=ec2.CloudFormationInit.from_config_sets(
        config_sets={
            # Applies the configs below in this order
            "default": ["yumPreinstall", "config"]
        },
        configs={
            "yum_preinstall": ec2.InitConfig([
                # Install an Amazon Linux package using yum
                ec2.InitPackage.yum("git")
            ]),
            "config": ec2.InitConfig([
                # Create a JSON file from tokens (can also create other files)
                ec2.InitFile.from_object("/etc/stack.json", {
                    "stack_id": Stack.of(self).stack_id,
                    "stack_name": Stack.of(self).stack_name,
                    "region": Stack.of(self).region
                }),

                # Create a group and user
                ec2.InitGroup.from_name("my-group"),
                ec2.InitUser.from_name("my-user"),

                # Install an RPM from the internet
                ec2.InitPackage.rpm("http://mirrors.ukfast.co.uk/sites/dl.fedoraproject.org/pub/epel/8/Everything/x86_64/Packages/r/rubygem-git-1.5.0-2.el8.noarch.rpm")
            ])
        }
    ),
    init_options=ec2.ApplyCloudFormationInitOptions(
        # Optional, which configsets to activate (['default'] by default)
        config_sets=["default"],

        # Optional, how long the installation is expected to take (5 minutes by default)
        timeout=Duration.minutes(30),

        # Optional, whether to include the --url argument when running cfn-init and cfn-signal commands (false by default)
        include_url=True,

        # Optional, whether to include the --role argument when running cfn-init and cfn-signal commands (false by default)
        include_role=True
    )
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

InitCommand can not be used to start long-running processes. At deploy time,
cfn-init will always wait for the process to exit before continuing, causing
the CloudFormation deployment to fail because the signal hasn't been received
within the expected timeout.

Instead, you should install a service configuration file onto your machine InitFile,
and then use InitService to start it.

If your Linux OS is using SystemD (like Amazon Linux 2 or higher), the CDK has
helpers to create a long-running service using CFN Init. You can create a
SystemD-compatible config file using InitService.systemdConfigFile(), and
start it immediately. The following examples shows how to start a trivial Python
3 web server:

# vpc: ec2.Vpc
# instance_type: ec2.InstanceType


ec2.Instance(self, "Instance",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=ec2.MachineImage.latest_amazon_linux2023(),

    init=ec2.CloudFormationInit.from_elements(
        # Create a simple config file that runs a Python web server
        ec2.InitService.systemd_config_file("simpleserver",
            command="/usr/bin/python3 -m http.server 8080",
            cwd="/var/www/html"
        ),
        # Start the server using SystemD
        ec2.InitService.enable("simpleserver",
            service_manager=ec2.ServiceManager.SYSTEMD
        ),
        # Drop an example file to show the web server working
        ec2.InitFile.from_string("/var/www/html/index.html", "Hello! It's working!"))
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

You can have services restarted after the init process has made changes to the system.
To do that, instantiate an InitServiceRestartHandle and pass it to the config elements
that need to trigger the restart and the service itself. For example, the following
config writes a config file for nginx, extracts an archive to the root directory, and then
restarts nginx so that it picks up the new config and files:

# my_bucket: s3.Bucket


handle = ec2.InitServiceRestartHandle()

ec2.CloudFormationInit.from_elements(
    ec2.InitFile.from_string("/etc/nginx/nginx.conf", "...", service_restart_handles=[handle]),
    ec2.InitSource.from_s3_object("/var/www/html", my_bucket, "html.zip", service_restart_handles=[handle]),
    ec2.InitService.enable("nginx",
        service_restart_handle=handle
    ))
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

You can use the environmentVariables or environmentFiles parameters to specify environment variables
for your services:

ec2.InitConfig([
    ec2.InitFile.from_string("/myvars.env", "VAR_FROM_FILE=\"VAR_FROM_FILE\""),
    ec2.InitService.systemd_config_file("myapp",
        command="/usr/bin/python3 -m http.server 8080",
        cwd="/var/www/html",
        environment_variables={
            "MY_VAR": "MY_VAR"
        },
        environment_files=["/myvars.env"]
    )
])
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Bastion Hosts

A bastion host functions as an instance used to access servers and resources in a VPC without open up the complete VPC on a network level.
You can use bastion hosts using a standard SSH connection targeting port 22 on the host. As an alternative, you can connect the SSH connection
feature of AWS Systems Manager Session Manager, which does not need an opened security group. (https://aws.amazon.com/about-aws/whats-new/2019/07/session-manager-launches-tunneling-support-for-ssh-and-scp/)

A default bastion host for use via SSM can be configured like:

host = ec2.BastionHostLinux(self, "BastionHost", vpc=vpc)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

If you want to connect from the internet using SSH, you need to place the host into a public subnet. You can then configure allowed source hosts.

host = ec2.BastionHostLinux(self, "BastionHost",
    vpc=vpc,
    subnet_selection=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PUBLIC)
)
host.allow_ssh_access_from(ec2.Peer.ipv4("1.2.3.4/32"))
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

As there are no SSH public keys deployed on this machine, you need to use EC2 Instance Connect
with the command aws ec2-instance-connect send-ssh-public-key to provide your SSH public key.

EBS volume for the bastion host can be encrypted like:

host = ec2.BastionHostLinux(self, "BastionHost",
    vpc=vpc,
    block_devices=[ec2.BlockDevice(
        device_name="/dev/sdh",
        volume=ec2.BlockDeviceVolume.ebs(10,
            encrypted=True
        )
    )]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

It's recommended to set the @aws-cdk/aws-ec2:bastionHostUseAmazonLinux2023ByDefault
feature flag to true to use Amazon Linux 2023 as the
bastion host AMI. Without this flag set, the bastion host will default to Amazon Linux 2, which will be unsupported in
June 2025.

{
  "context": {
    "@aws-cdk/aws-ec2:bastionHostUseAmazonLinux2023ByDefault": true
  }
}
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Json
IGNORE_WHEN_COPYING_END
Placement Group

Specify placementGroup to enable the placement group support:

# instance_type: ec2.InstanceType


pg = ec2.PlacementGroup(self, "test-pg",
    strategy=ec2.PlacementGroupStrategy.SPREAD
)

ec2.Instance(self, "Instance",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=ec2.MachineImage.latest_amazon_linux2023(),
    placement_group=pg
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Block Devices

To add EBS block device mappings, specify the blockDevices property. The following example sets the EBS-backed
root device (/dev/sda1) size to 50 GiB, and adds another EBS-backed device mapped to /dev/sdm that is 100 GiB in
size:

# vpc: ec2.Vpc
# instance_type: ec2.InstanceType
# machine_image: ec2.IMachineImage


ec2.Instance(self, "Instance",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=machine_image,

    # ...

    block_devices=[ec2.BlockDevice(
        device_name="/dev/sda1",
        volume=ec2.BlockDeviceVolume.ebs(50)
    ), ec2.BlockDevice(
        device_name="/dev/sdm",
        volume=ec2.BlockDeviceVolume.ebs(100)
    )
    ]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

It is also possible to encrypt the block devices. In this example we will create an customer managed key encrypted EBS-backed root device:

from aws_cdk.aws_kms import Key

# vpc: ec2.Vpc
# instance_type: ec2.InstanceType
# machine_image: ec2.IMachineImage


kms_key = Key(self, "KmsKey")

ec2.Instance(self, "Instance",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=machine_image,

    # ...

    block_devices=[ec2.BlockDevice(
        device_name="/dev/sda1",
        volume=ec2.BlockDeviceVolume.ebs(50,
            encrypted=True,
            kms_key=kms_key
        )
    )
    ]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

To specify the throughput value for gp3 volumes, use the throughput property:

# vpc: ec2.Vpc
# instance_type: ec2.InstanceType
# machine_image: ec2.IMachineImage


ec2.Instance(self, "Instance",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=machine_image,

    # ...

    block_devices=[ec2.BlockDevice(
        device_name="/dev/sda1",
        volume=ec2.BlockDeviceVolume.ebs(100,
            volume_type=ec2.EbsDeviceVolumeType.GP3,
            throughput=250
        )
    )
    ]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
EBS Optimized Instances

An Amazon EBS–optimized instance uses an optimized configuration stack and provides additional, dedicated capacity for Amazon EBS I/O. This optimization provides the best performance for your EBS volumes by minimizing contention between Amazon EBS I/O and other traffic from your instance.

Depending on the instance type, this features is enabled by default while others require explicit activation. Please refer to the documentation for details.

# vpc: ec2.Vpc
# instance_type: ec2.InstanceType
# machine_image: ec2.IMachineImage


ec2.Instance(self, "Instance",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=machine_image,
    ebs_optimized=True,
    block_devices=[ec2.BlockDevice(
        device_name="/dev/xvda",
        volume=ec2.BlockDeviceVolume.ebs(8)
    )]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Volumes

Whereas a BlockDeviceVolume is an EBS volume that is created and destroyed as part of the creation and destruction of a specific instance. A Volume is for when you want an EBS volume separate from any particular instance. A Volume is an EBS block device that can be attached to, or detached from, any instance at any time. Some types of Volumes can also be attached to multiple instances at the same time to allow you to have shared storage between those instances.

A notable restriction is that a Volume can only be attached to instances in the same availability zone as the Volume itself.

The following demonstrates how to create a 500 GiB encrypted Volume in the us-west-2a availability zone, and give a role the ability to attach that Volume to a specific instance:

# instance: ec2.Instance
# role: iam.Role


volume = ec2.Volume(self, "Volume",
    availability_zone="us-west-2a",
    size=Size.gibibytes(500),
    encrypted=True
)

volume.grant_attach_volume(role, [instance])
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Instances Attaching Volumes to Themselves

If you need to grant an instance the ability to attach/detach an EBS volume to/from itself, then using grantAttachVolume and grantDetachVolume as outlined above
will lead to an unresolvable circular reference between the instance role and the instance. In this case, use grantAttachVolumeByResourceTag and grantDetachVolumeByResourceTag as follows:

# instance: ec2.Instance
# volume: ec2.Volume


attach_grant = volume.grant_attach_volume_by_resource_tag(instance.grant_principal, [instance])
detach_grant = volume.grant_detach_volume_by_resource_tag(instance.grant_principal, [instance])
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Attaching Volumes

The Amazon EC2 documentation for
Linux Instances and
Windows Instances contains information on how
to attach and detach your Volumes to/from instances, and how to format them for use.

The following is a sample skeleton of EC2 UserData that can be used to attach a Volume to the Linux instance that it is running on:

# instance: ec2.Instance
# volume: ec2.Volume


volume.grant_attach_volume_by_resource_tag(instance.grant_principal, [instance])
target_device = "/dev/xvdz"
instance.user_data.add_commands("TOKEN=$(curl -SsfX PUT \"http://169.254.169.254/latest/api/token\" -H \"X-aws-ec2-metadata-token-ttl-seconds: 21600\")", "INSTANCE_ID=$(curl -SsfH \"X-aws-ec2-metadata-token: $TOKEN\" http://169.254.169.254/latest/meta-data/instance-id)", f"aws --region {Stack.of(this).region} ec2 attach-volume --volume-id {volume.volumeId} --instance-id $INSTANCE_ID --device {targetDevice}", f"while ! test -e {targetDevice}; do sleep 1; done")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Tagging Volumes

You can configure tag propagation on volume creation.

# vpc: ec2.Vpc
# instance_type: ec2.InstanceType
# machine_image: ec2.IMachineImage


ec2.Instance(self, "Instance",
    vpc=vpc,
    machine_image=machine_image,
    instance_type=instance_type,
    propagate_tags_to_volume_on_creation=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Throughput on GP3 Volumes

You can specify the throughput of a GP3 volume from 125 (default) to 1000.

ec2.Volume(self, "Volume",
    availability_zone="us-east-1a",
    size=Size.gibibytes(125),
    volume_type=ec2.EbsDeviceVolumeType.GP3,
    throughput=125
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Configuring Instance Metadata Service (IMDS)
Toggling IMDSv1

You can configure EC2 Instance Metadata Service options to either
allow both IMDSv1 and IMDSv2 or enforce IMDSv2 when interacting with the IMDS.

To do this for a single Instance, you can use the requireImdsv2 property.
The example below demonstrates IMDSv2 being required on a single Instance:

# vpc: ec2.Vpc
# instance_type: ec2.InstanceType
# machine_image: ec2.IMachineImage


ec2.Instance(self, "Instance",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=machine_image,

    # ...

    require_imdsv2=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

You can also use the either the InstanceRequireImdsv2Aspect for EC2 instances or the LaunchTemplateRequireImdsv2Aspect for EC2 launch templates
to apply the operation to multiple instances or launch templates, respectively.

The following example demonstrates how to use the InstanceRequireImdsv2Aspect to require IMDSv2 for all EC2 instances in a stack:

aspect = ec2.InstanceRequireImdsv2Aspect()
Aspects.of(self).add(aspect)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Associating a Public IP Address with an Instance

All subnets have an attribute that determines whether instances launched into that subnet are assigned a public IPv4 address. This attribute is set to true by default for default public subnets. Thus, an EC2 instance launched into a default public subnet will be assigned a public IPv4 address. Nondefault public subnets have this attribute set to false by default and any EC2 instance launched into a nondefault public subnet will not be assigned a public IPv4 address automatically. To automatically assign a public IPv4 address to an instance launched into a nondefault public subnet, you can set the associatePublicIpAddress property on the Instance construct to true. Alternatively, to not automatically assign a public IPv4 address to an instance launched into a default public subnet, you can set associatePublicIpAddress to false. Including this property, removing this property, or updating the value of this property on an existing instance will result in replacement of the instance.

vpc = ec2.Vpc(self, "VPC",
    cidr="10.0.0.0/16",
    nat_gateways=0,
    max_azs=3,
    subnet_configuration=[ec2.SubnetConfiguration(
        name="public-subnet-1",
        subnet_type=ec2.SubnetType.PUBLIC,
        cidr_mask=24
    )
    ]
)

instance = ec2.Instance(self, "Instance",
    vpc=vpc,
    vpc_subnets=ec2.SubnetSelection(subnet_group_name="public-subnet-1"),
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.NANO),
    machine_image=ec2.AmazonLinuxImage(generation=ec2.AmazonLinuxGeneration.AMAZON_LINUX_2),
    detailed_monitoring=True,
    associate_public_ip_address=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Specifying a key pair

To allow SSH access to an EC2 instance by default, a Key Pair must be specified. Key pairs can
be provided with the keyPair property to instances and launch templates. You can create a
key pair for an instance like this:

# vpc: ec2.Vpc
# instance_type: ec2.InstanceType


key_pair = ec2.KeyPair(self, "KeyPair",
    type=ec2.KeyPairType.ED25519,
    format=ec2.KeyPairFormat.PEM
)
instance = ec2.Instance(self, "Instance",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=ec2.MachineImage.latest_amazon_linux2023(),
    # Use the custom key pair
    key_pair=key_pair
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

When a new EC2 Key Pair is created (without imported material), the private key material is
automatically stored in Systems Manager Parameter Store. This can be retrieved from the key pair
construct:

key_pair = ec2.KeyPair(self, "KeyPair")
private_key = key_pair.private_key
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

If you already have an SSH key that you wish to use in EC2, that can be provided when constructing the
KeyPair. If public key material is provided, the key pair is considered "imported" and there
will not be any data automatically stored in Systems Manager Parameter Store and the type property
cannot be specified for the key pair.

key_pair = ec2.KeyPair(self, "KeyPair",
    public_key_material="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIB7jpNzG+YG0s+xIGWbxrxIZiiozHOEuzIJacvASP0mq"
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Using an existing EC2 Key Pair

If you already have an EC2 Key Pair created outside of the CDK, you can import that key to
your CDK stack.

You can import it purely by name:

key_pair = ec2.KeyPair.from_key_pair_name(self, "KeyPair", "the-keypair-name")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Or by specifying additional attributes:

key_pair = ec2.KeyPair.from_key_pair_attributes(self, "KeyPair",
    key_pair_name="the-keypair-name",
    type=ec2.KeyPairType.RSA
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Using IPv6 IPs

Instances can be given IPv6 IPs by launching them into a subnet of a dual stack VPC.

vpc = ec2.Vpc(self, "Ip6VpcDualStack",
    ip_protocol=ec2.IpProtocol.DUAL_STACK,
    subnet_configuration=[ec2.SubnetConfiguration(
        name="Public",
        subnet_type=ec2.SubnetType.PUBLIC,
        map_public_ip_on_launch=True
    ), ec2.SubnetConfiguration(
        name="Private",
        subnet_type=ec2.SubnetType.PRIVATE_ISOLATED
    )
    ]
)

instance = ec2.Instance(self, "MyInstance",
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.T2, ec2.InstanceSize.MICRO),
    machine_image=ec2.MachineImage.latest_amazon_linux2(),
    vpc=vpc,
    vpc_subnets=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PUBLIC),
    allow_all_ipv6_outbound=True
)

instance.connections.allow_from(ec2.Peer.any_ipv6(), ec2.Port.all_icmp_v6(), "allow ICMPv6")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Note to set mapPublicIpOnLaunch to true in the subnetConfiguration.

Additionally, IPv6 support varies by instance type. Most instance types have IPv6 support with exception of m1-m3, c1, g2, and t1.micro. A full list can be found here: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI.

Specifying the IPv6 Address

If you want to specify the number of IPv6 addresses to assign to the instance, you can use the ipv6AddresseCount property:

# dual stack VPC
# vpc: ec2.Vpc


instance = ec2.Instance(self, "MyInstance",
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.M5, ec2.InstanceSize.LARGE),
    machine_image=ec2.MachineImage.latest_amazon_linux2(),
    vpc=vpc,
    vpc_subnets=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PUBLIC),
    # Assign 2 IPv6 addresses to the instance
    ipv6_address_count=2
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Credit configuration modes for burstable instances

You can set the credit configuration mode for burstable instances (T2, T3, T3a and T4g instance types):

# vpc: ec2.Vpc


instance = ec2.Instance(self, "Instance",
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
    machine_image=ec2.MachineImage.latest_amazon_linux2(),
    vpc=vpc,
    credit_specification=ec2.CpuCredits.STANDARD
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

It is also possible to set the credit configuration mode for NAT instances.

nat_instance_provider = ec2.NatProvider.instance(
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.T4G, ec2.InstanceSize.LARGE),
    machine_image=ec2.AmazonLinuxImage(),
    credit_specification=ec2.CpuCredits.UNLIMITED
)
ec2.Vpc(self, "VPC",
    nat_gateway_provider=nat_instance_provider
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Note: CpuCredits.UNLIMITED mode is not supported for T3 instances that are launched on a Dedicated Host.

Shutdown behavior

You can specify the behavior of the instance when you initiate shutdown from the instance (using the operating system command for system shutdown).

# vpc: ec2.Vpc


ec2.Instance(self, "Instance",
    vpc=vpc,
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.NANO),
    machine_image=ec2.AmazonLinuxImage(generation=ec2.AmazonLinuxGeneration.AMAZON_LINUX_2),
    instance_initiated_shutdown_behavior=ec2.InstanceInitiatedShutdownBehavior.TERMINATE
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Enabling Nitro Enclaves

You can enable AWS Nitro Enclaves for
your EC2 instances by setting the enclaveEnabled property to true. Nitro Enclaves is a feature of
AWS Nitro System that enables creating isolated and highly constrained CPU environments known as enclaves.

# vpc: ec2.Vpc


instance = ec2.Instance(self, "Instance",
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.M5, ec2.InstanceSize.XLARGE),
    machine_image=ec2.AmazonLinuxImage(),
    vpc=vpc,
    enclave_enabled=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

NOTE: You must use an instance type and operating system that support Nitro Enclaves.
For more information, see Requirements.

Enabling Termination Protection

You can enable Termination Protection for
your EC2 instances by setting the disableApiTermination property to true. Termination Protection controls whether the instance can be terminated using the AWS Management Console, AWS Command Line Interface (AWS CLI), or API.

# vpc: ec2.Vpc


instance = ec2.Instance(self, "Instance",
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.M5, ec2.InstanceSize.XLARGE),
    machine_image=ec2.AmazonLinuxImage(),
    vpc=vpc,
    disable_api_termination=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Enabling Instance Hibernation

You can enable Instance Hibernation for
your EC2 instances by setting the hibernationEnabled property to true. Instance Hibernation saves the
instance's in-memory (RAM) state when an instance is stopped, and restores that state when the instance is started.

# vpc: ec2.Vpc


instance = ec2.Instance(self, "Instance",
    instance_type=ec2.InstanceType.of(ec2.InstanceClass.M5, ec2.InstanceSize.XLARGE),
    machine_image=ec2.AmazonLinuxImage(),
    vpc=vpc,
    hibernation_enabled=True,
    block_devices=[ec2.BlockDevice(
        device_name="/dev/xvda",
        volume=ec2.BlockDeviceVolume.ebs(30,
            volume_type=ec2.EbsDeviceVolumeType.GP3,
            encrypted=True,
            delete_on_termination=True
        )
    )]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

NOTE: You must use an instance and a volume that meet the requirements for hibernation.
For more information, see Prerequisites for Amazon EC2 instance hibernation.

VPC Flow Logs

VPC Flow Logs is a feature that enables you to capture information about the IP traffic going to and from network interfaces in your VPC. Flow log data can be published to Amazon CloudWatch Logs and Amazon S3. After you've created a flow log, you can retrieve and view its data in the chosen destination. (https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html).

By default, a flow log will be created with CloudWatch Logs as the destination.

You can create a flow log like this:

# vpc: ec2.Vpc


ec2.FlowLog(self, "FlowLog",
    resource_type=ec2.FlowLogResourceType.from_vpc(vpc)
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Or you can add a Flow Log to a VPC by using the addFlowLog method like this:

vpc = ec2.Vpc(self, "Vpc")

vpc.add_flow_log("FlowLog")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

You can also add multiple flow logs with different destinations.

vpc = ec2.Vpc(self, "Vpc")

vpc.add_flow_log("FlowLogS3",
    destination=ec2.FlowLogDestination.to_s3()
)

# Only reject traffic and interval every minute.
vpc.add_flow_log("FlowLogCloudWatch",
    traffic_type=ec2.FlowLogTrafficType.REJECT,
    max_aggregation_interval=ec2.FlowLogMaxAggregationInterval.ONE_MINUTE
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

To create a Transit Gateway flow log, you can use the fromTransitGatewayId method:

# tgw: ec2.CfnTransitGateway


ec2.FlowLog(self, "TransitGatewayFlowLog",
    resource_type=ec2.FlowLogResourceType.from_transit_gateway_id(tgw.ref)
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

To create a Transit Gateway Attachment flow log, you can use the fromTransitGatewayAttachmentId method:

# tgw_attachment: ec2.CfnTransitGatewayAttachment


ec2.FlowLog(self, "TransitGatewayAttachmentFlowLog",
    resource_type=ec2.FlowLogResourceType.from_transit_gateway_attachment_id(tgw_attachment.ref)
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

For flow logs targeting TransitGateway and TransitGatewayAttachment, specifying the trafficType is not possible.

Custom Formatting

You can also custom format flow logs.

vpc = ec2.Vpc(self, "Vpc")

vpc.add_flow_log("FlowLog",
    log_format=[ec2.LogFormat.DST_PORT, ec2.LogFormat.SRC_PORT
    ]
)

# If you just want to add a field to the default field
vpc.add_flow_log("FlowLog",
    log_format=[ec2.LogFormat.VERSION, ec2.LogFormat.ALL_DEFAULT_FIELDS
    ]
)

# If AWS CDK does not support the new fields
vpc.add_flow_log("FlowLog",
    log_format=[ec2.LogFormat.SRC_PORT,
        ec2.LogFormat.custom("${new-field}")
    ]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

By default, the CDK will create the necessary resources for the destination. For the CloudWatch Logs destination
it will create a CloudWatch Logs Log Group as well as the IAM role with the necessary permissions to publish to
the log group. In the case of an S3 destination, it will create the S3 bucket.

If you want to customize any of the destination resources you can provide your own as part of the destination.

CloudWatch Logs

# vpc: ec2.Vpc


log_group = logs.LogGroup(self, "MyCustomLogGroup")

role = iam.Role(self, "MyCustomRole",
    assumed_by=iam.ServicePrincipal("vpc-flow-logs.amazonaws.com")
)

ec2.FlowLog(self, "FlowLog",
    resource_type=ec2.FlowLogResourceType.from_vpc(vpc),
    destination=ec2.FlowLogDestination.to_cloud_watch_logs(log_group, role)
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

S3

# vpc: ec2.Vpc


bucket = s3.Bucket(self, "MyCustomBucket")

ec2.FlowLog(self, "FlowLog",
    resource_type=ec2.FlowLogResourceType.from_vpc(vpc),
    destination=ec2.FlowLogDestination.to_s3(bucket)
)

ec2.FlowLog(self, "FlowLogWithKeyPrefix",
    resource_type=ec2.FlowLogResourceType.from_vpc(vpc),
    destination=ec2.FlowLogDestination.to_s3(bucket, "prefix/")
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Kinesis Data Firehose

import aws_cdk.aws_kinesisfirehose as firehose

# vpc: ec2.Vpc
# delivery_stream: firehose.CfnDeliveryStream


vpc.add_flow_log("FlowLogsKinesisDataFirehose",
    destination=ec2.FlowLogDestination.to_kinesis_data_firehose_destination(delivery_stream.attr_arn)
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

When the S3 destination is configured, AWS will automatically create an S3 bucket policy
that allows the service to write logs to the bucket. This makes it impossible to later update
that bucket policy. To have CDK create the bucket policy so that future updates can be made,
the @aws-cdk/aws-s3:createDefaultLoggingPolicy feature flag can be used. This can be set
in the cdk.json file.

{
  "context": {
    "@aws-cdk/aws-s3:createDefaultLoggingPolicy": true
  }
}
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Json
IGNORE_WHEN_COPYING_END
User Data

User data enables you to run a script when your instances start up.  In order to configure these scripts you can add commands directly to the script
or you can use the UserData's convenience functions to aid in the creation of your script.

A user data could be configured to run a script found in an asset through the following:

from aws_cdk.aws_s3_assets import Asset

# instance: ec2.Instance


asset = Asset(self, "Asset",
    path="./configure.sh"
)

local_path = instance.user_data.add_s3_download_command(
    bucket=asset.bucket,
    bucket_key=asset.s3_object_key,
    region="us-east-1"
)
instance.user_data.add_execute_file_command(
    file_path=local_path,
    arguments="--verbose -y"
)
asset.grant_read(instance.role)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Persisting user data

By default, EC2 UserData is run once on only the first time that an instance is started. It is possible to make the
user data script run on every start of the instance.

When creating a Windows UserData you can use the persist option to set whether or not to add
<persist>true</persist> to the user data script. it can be used as follows:

windows_user_data = ec2.UserData.for_windows(persist=True)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

For a Linux instance, this can be accomplished by using a Multipart user data to configure cloud-config as detailed
in: https://aws.amazon.com/premiumsupport/knowledge-center/execute-user-data-ec2/

Multipart user data

In addition, to above the MultipartUserData can be used to change instance startup behavior. Multipart user data are composed
from separate parts forming archive. The most common parts are scripts executed during instance set-up. However, there are other
kinds, too.

The advantage of multipart archive is in flexibility when it's needed to add additional parts or to use specialized parts to
fine tune instance startup. Some services (like AWS Batch) support only MultipartUserData.

The parts can be executed at different moment of instance start-up and can serve a different purpose. This is controlled by contentType property.
For common scripts, text/x-shellscript; charset="utf-8" can be used as content type.

In order to create archive the MultipartUserData has to be instantiated. Than, user can add parts to multipart archive using addPart. The MultipartBody contains methods supporting creation of body parts.

If the very custom part is required, it can be created using MultipartUserData.fromRawBody, in this case full control over content type,
transfer encoding, and body properties is given to the user.

Below is an example for creating multipart user data with single body part responsible for installing awscli and configuring maximum size
of storage used by Docker containers:

boot_hook_conf = ec2.UserData.for_linux()
boot_hook_conf.add_commands("cloud-init-per once docker_options echo 'OPTIONS=\"${OPTIONS} --storage-opt dm.basesize=40G\"' >> /etc/sysconfig/docker")

setup_commands = ec2.UserData.for_linux()
setup_commands.add_commands("sudo yum install awscli && echo Packages installed らと > /var/tmp/setup")

multipart_user_data = ec2.MultipartUserData()
# The docker has to be configured at early stage, so content type is overridden to boothook
multipart_user_data.add_part(ec2.MultipartBody.from_user_data(boot_hook_conf, "text/cloud-boothook; charset=\"us-ascii\""))
# Execute the rest of setup
multipart_user_data.add_part(ec2.MultipartBody.from_user_data(setup_commands))

ec2.LaunchTemplate(self, "",
    user_data=multipart_user_data,
    block_devices=[]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

For more information see
Specifying Multiple User Data Blocks Using a MIME Multi Part Archive

Using add*Command on MultipartUserData

To use the add*Command methods, that are inherited from the UserData interface, on MultipartUserData you must add a part
to the MultipartUserData and designate it as the receiver for these methods. This is accomplished by using the addUserDataPart()
method on MultipartUserData with the makeDefault argument set to true:

multipart_user_data = ec2.MultipartUserData()
commands_user_data = ec2.UserData.for_linux()
multipart_user_data.add_user_data_part(commands_user_data, ec2.MultipartBody.SHELL_SCRIPT, True)

# Adding commands to the multipartUserData adds them to commandsUserData, and vice-versa.
multipart_user_data.add_commands("touch /root/multi.txt")
commands_user_data.add_commands("touch /root/userdata.txt")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

When used on an EC2 instance, the above multipartUserData will create both multi.txt and userdata.txt in /root.

Importing existing subnet

To import an existing Subnet, call Subnet.fromSubnetAttributes() or
Subnet.fromSubnetId(). Only if you supply the subnet's Availability Zone
and Route Table Ids when calling Subnet.fromSubnetAttributes() will you be
able to use the CDK features that use these values (such as selecting one
subnet per AZ).

Importing an existing subnet looks like this:

# Supply all properties
subnet1 = ec2.Subnet.from_subnet_attributes(self, "SubnetFromAttributes",
    subnet_id="s-1234",
    availability_zone="pub-az-4465",
    route_table_id="rt-145"
)

# Supply only subnet id
subnet2 = ec2.Subnet.from_subnet_id(self, "SubnetFromId", "s-1234")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Launch Templates

A Launch Template is a standardized template that contains the configuration information to launch an instance.
They can be used when launching instances on their own, through Amazon EC2 Auto Scaling, EC2 Fleet, and Spot Fleet.
Launch templates enable you to store launch parameters so that you do not have to specify them every time you launch
an instance. For information on Launch Templates please see the
official documentation.

The following demonstrates how to create a launch template with an Amazon Machine Image, security group, and an instance profile.

# vpc: ec2.Vpc


role = iam.Role(self, "Role",
    assumed_by=iam.ServicePrincipal("ec2.amazonaws.com")
)
instance_profile = iam.InstanceProfile(self, "InstanceProfile",
    role=role
)

template = ec2.LaunchTemplate(self, "LaunchTemplate",
    launch_template_name="MyTemplateV1",
    version_description="This is my v1 template",
    machine_image=ec2.MachineImage.latest_amazon_linux2023(),
    security_group=ec2.SecurityGroup(self, "LaunchTemplateSG",
        vpc=vpc
    ),
    instance_profile=instance_profile
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

And the following demonstrates how to enable metadata options support.

ec2.LaunchTemplate(self, "LaunchTemplate",
    http_endpoint=True,
    http_protocol_ipv6=True,
    http_put_response_hop_limit=1,
    http_tokens=ec2.LaunchTemplateHttpTokens.REQUIRED,
    instance_metadata_tags=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

And the following demonstrates how to add one or more security groups to launch template.

# vpc: ec2.Vpc


sg1 = ec2.SecurityGroup(self, "sg1",
    vpc=vpc
)
sg2 = ec2.SecurityGroup(self, "sg2",
    vpc=vpc
)

launch_template = ec2.LaunchTemplate(self, "LaunchTemplate",
    machine_image=ec2.MachineImage.latest_amazon_linux2023(),
    security_group=sg1
)

launch_template.add_security_group(sg2)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

To use AWS Systems Manager parameters instead of AMI IDs in launch templates and resolve the AMI IDs at instance launch time:

launch_template = ec2.LaunchTemplate(self, "LaunchTemplate",
    machine_image=ec2.MachineImage.resolve_ssm_parameter_at_launch("parameterName")
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

Please note this feature does not support Launch Configurations.

Detailed Monitoring

The following demonstrates how to enable Detailed Monitoring for an EC2 instance. Keep in mind that Detailed Monitoring results in additional charges.

# vpc: ec2.Vpc
# instance_type: ec2.InstanceType


ec2.Instance(self, "Instance1",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=ec2.MachineImage.latest_amazon_linux2023(),
    detailed_monitoring=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Connecting to your instances using SSM Session Manager

SSM Session Manager makes it possible to connect to your instances from the
AWS Console, without preparing SSH keys.

To do so, you need to:

Use an image with SSM agent installed
and configured. Many images come with SSM Agent
preinstalled, otherwise you
may need to manually put instructions to install SSM
Agent into your
instance's UserData or use EC2 Init).

Create the instance with ssmSessionPermissions: true.

If these conditions are met, you can connect to the instance from the EC2 Console. Example:

# vpc: ec2.Vpc
# instance_type: ec2.InstanceType


ec2.Instance(self, "Instance1",
    vpc=vpc,
    instance_type=instance_type,

    # Amazon Linux 2023 comes with SSM Agent by default
    machine_image=ec2.MachineImage.latest_amazon_linux2023(),

    # Turn on SSM
    ssm_session_permissions=True
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
Managed Prefix Lists

Create and manage customer-managed prefix lists. If you don't specify anything in this construct, it will manage IPv4 addresses.

You can also create an empty Prefix List with only the maximum number of entries specified, as shown in the following code. If nothing is specified, maxEntries=1.

ec2.PrefixList(self, "EmptyPrefixList",
    max_entries=100
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

maxEntries can also be omitted as follows. In this case maxEntries: 2, will be set.

ec2.PrefixList(self, "PrefixList",
    entries=[ec2.CfnPrefixList.EntryProperty(cidr="10.0.0.1/32"), ec2.CfnPrefixList.EntryProperty(cidr="10.0.0.2/32", description="sample1")
    ]
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

For more information see Work with customer-managed prefix lists

IAM instance profile

Use instanceProfile to apply specific IAM Instance Profile. Cannot be used with role

# instance_type: ec2.InstanceType
# vpc: ec2.Vpc


role = iam.Role(self, "Role",
    assumed_by=iam.ServicePrincipal("ec2.amazonaws.com")
)
instance_profile = iam.InstanceProfile(self, "InstanceProfile",
    role=role
)

ec2.Instance(self, "Instance",
    vpc=vpc,
    instance_type=instance_type,
    machine_image=ec2.MachineImage.latest_amazon_linux2023(),
    instance_profile=instance_profile
)
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END

'''

how i use this
