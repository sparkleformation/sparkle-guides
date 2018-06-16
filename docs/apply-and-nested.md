---
title: "Stack Dependencies"
weight: 2
anchors:
  - title: "Overview"
    url: "#overview"
  - title: "Apply stack"
    url: "#apply-stack-implementation"
  - title: "Nested stack"
    url: "#nested-stack-implementation"
  - title: "Apply nested stack"
    url: "#apply-nested-stack"
  - title: "Cross location"
    url: "#cross-location"
---

## Overview

### Apply stack

The SparkleFormation CLI includes functionality to ease the sharing
of resources between isolated stacks. This functionality enables
groupings of similar resources which can define logical units of
a full architecture. The sharing is accomplished using a combination
of stack outputs and stack parameters. This feature is commonly
referred to as the _apply stack_ functionality.

One drawback of the _apply stack_ functionality is that it introduces
dependencies between disparate stacks. These dependencies must be
manually resolved when resource modifications occur. When an infrastructure
is composed of few stacks, the _apply stack_ approach can be sufficient
to manage stack dependencies leaving the user to ensure updates are
properly applied. For larger implementations composed of many interdependent
stacks, the _nested stack_ functionality is recommended.

### Nested stack

_Nested stack_ functionality is a generic feature provided by the SparkleFormation
library which the SparkleFormation CLI then specializes based on the target provider.
The _nested stack_ functionality utilizes a core feature provided by orchestration
APIs. This features allows nesting stack resources within a stack allowing a
parent stack to have many child stacks. By nesting stack resources, the provider
API will be aware of child stack interdependencies and automatically apply updates
when resources have been modified. This removes the requirement of stack updates
being tracked and applied manually.

The behavior of the _nested stack_ functionality is based directly on the
behavior of the _apply stack_ functionality. This commonality in behavior
allows for initial testing development using the _apply stack_ functionality
and, once stable, migrating to _nested stack_ functionality without requiring any
modifications to existing templates. The commonality also allows for the two
functionalities to be mixed in practice.

This guide will first display template implementations using the _apply stack_
functionality. The example templates will then be used to provide a _nested stack_
implementation.

_NOTE: This guide targets the AWS provider for simplicity to allow focus on the
features discussed. All providers support this behavior._

## Apply stack implementation

Lets start by defining a simple infrastructure. Our infrastructure will be composed
of a "network" and a collection of "computes" that utilizes the network. This
infrastructure can be easily defined within two units:

1. network
2. computes

Lets start by creating a simple `network` template.

Create a new file: `./sparkleformation/network.rb`

#### Template sparkles AWS

~~~ruby
SparkleFormation.new(:network) do

  parameters do
    cidr_prefix do
      type 'String'
      default '172.20'
    end
  end

  dynamic!(:ec2_vpc, :network) do
    properties do
      cidr_block join!(ref!(:cidr_prefix), '.0.0/24')
      enable_dns_support true
      enable_dns_hostnames true
    end
  end

  dynamic!(:ec2_dhcp_options, :network) do
    properties do
      domain_name join!(region!, 'compute.internal')
      domain_name_servers ['AmazonProvidedDNS']
    end
  end

  dynamic!(:ec2_vpc_dhcp_options_association, :network) do
    properties do
      dhcp_options_id ref!(:network_ec2_dhcp_options)
      vpc_id ref!(:network_ec2_vpc)
    end
  end

  dynamic!(:ec2_internet_gateway, :network)

  dynamic!(:ec2_vpc_gateway_attachment, :network) do
    properties do
      internet_gateway_id ref!(:network_ec2_internet_gateway)
      vpc_id ref!(:network_ec2_vpc)
    end
  end

  dynamic!(:ec2_route_table, :network) do
    properties.vpc_id ref!(:network_ec2_vpc)
  end

  dynamic!(:ec2_route, :network_public) do
    properties do
      destination_cidr_block '0.0.0.0/0'
      gateway_id ref!(:network_ec2_internet_gateway)
      route_table_id ref!(:network_ec2_route_table)
    end
  end

  dynamic!(:ec2_subnet, :network) do
    properties do
      availability_zone select!(0, azs!)
      cidr_block join!(ref!(:cidr_prefix), '.0.0/24')
      vpc_id ref!(:network_ec2_vpc)
    end
  end

  dynamic!(:ec2_subnet_route_table_association, :network) do
    properties do
      route_table_id ref!(:network_ec2_route_table)
      subnet_id ref!(:network_ec2_subnet)
    end
  end

  outputs do
    network_vpc_id.value ref!(:network_ec2_vpc)
    network_subnet_id.value ref!(:network_ec2_subnet)
    network_route_table.value ref!(:network_ec2_route_table)
    network_cidr.value join!(ref!(:cidr_prefix), '.0.0/24')
  end

end
~~~

Here we have an extremely simple VPC defined for our infrastructure. It is
important to note the outputs defined within our `network` template. These
outputs are values that will be required for other resources to effectively
utilize the VPC. With the `network` template defined, we can create that unit
of our infrastructure:

~~~
$ bundle exec sfn create sparkle-guide-network --file network
~~~

Having successfully built the `network` unit of the infrastructure, we can
now compose the `computes` template.

Create a new file: `./sparkleformation/computes.rb`

#### Template sparkles AWS

~~~ruby
SparkleFormation.new(:computes) do

  parameters do
    ssh_key_name.type 'String'
    network_vpc_id.type 'String'
    network_subnet_id.type 'String'
    image_id_name do
        type 'String'
        default 'ami-63ac5803'
    end
  end

  dynamic!(:ec2_security_group, :compute) do
    properties do
      group_description 'SSH Access'
      security_group_ingress do
        cidr_ip '0.0.0.0/0'
        from_port 22
        to_port 22
        ip_protocol 'tcp'
      end
      vpc_id ref!(:network_vpc_id)
    end
  end

  dynamic!(:ec2_instance, :micro) do
    properties do
      image_id ref!(:image_id_name)
      instance_type 't2.micro'
      key_name ref!(:ssh_key_name)
      network_interfaces array!(
        ->{
          device_index 0
          associate_public_ip_address 'true'
          subnet_id ref!(:network_subnet_id)
          group_set [ref!(:compute_ec2_security_group)]
        }
      )
    end
  end

  dynamic!(:ec2_instance, :small) do
    properties do
      image_id ref!(:image_id_name)
      instance_type 't2.micro'
      key_name ref!(:ssh_key_name)
      network_interfaces array!(
        ->{
          device_index 0
          associate_public_ip_address 'true'
          subnet_id ref!(:network_subnet_id)
          group_set [ref!(:compute_ec2_security_group)]
        }
      )
    end
  end

  outputs do
    micro_address.value attr!(:micro_ec2_instance, :public_ip)
    small_address.value attr!(:small_ec2_instance, :public_ip)
  end

end
~~~

Our `computes` template will create one micro and one small EC2
instance and output the public IP addresses of each resource. To
build the EC2 instances into the VPC created in the `sparkle-guide-network`
stack we need two pieces of information:

* VPC ID
* VPC subnet ID

In our `computes` template we define two parameters for the required
VPC information: `network_vpc_id` and `network_subnet_id`. When we
create a stack using this template we can copy the output values from the
`sparkle-guide-network` stack and paste them into these parameters, but
that is extremely cumbersome. The SparkleFormation CLI instead allows
"applying" a stack on creation or update.

Notice that the output names in our `network` template match the parameter
names in our `computes` template.

~~~ruby
# From ./sparkleformation/network.rb
  outputs do
    network_vpc_id.value ref!(:network_ec2_vpc)
    network_subnet_id.value ref!(:network_ec2_subnet)
    network_route_table.value ref!(:network_ec2_route_table)
    network_cidr.value join!(ref!(:cidr_prefix, '.0.0/24'))
  end

# From ./sparkleformation/computes.rb
  parameters do
    ssh_key_name.type 'String'
    network_vpc_id.type 'String'
    network_subnet_id.type 'String'
  end
~~~

The apply stack functionality will automatically collect outputs from
the stack names provided and use them to seed the parameters of the
stack being created or updated. This means instead of having to copy
and paste the VPC ID and subnet ID values, we can instruct the SparkleFormation
CLI to automatically use the outputs of our `sparkle-guide-network` stack:

~~~
$ bundle exec sfn create sparkle-guide-computes --file computes --apply-stack sparkle-guide-network
~~~

During the create process, the SparkleFormation CLI will prompt for parameters. The
default values for the VPC ID and subnet ID will be automatically inserted, matching
the outputs from the `sparkle-guide-network`.

You can destroy the sparkle-guide-compute and sparkle-guide-network stacks as they will not be used in the next section.

~~~
$ sfn destroy sparkle-guide-computes
$ sfn destroy sparkle-guide-network
~~~

## Nested stack implementation

Now that our infrastructure has been successfully created using disparate stacks, lets
combine them to create a single `infrastructure` unit composed of sub-units. Using
nested stacks requires a bucket to store templates. Create a bucket in S3 and then
add the following to the `.sfn` configuration file:

~~~ruby
Configuration.new do
  ...
  nesting_bucket 'NAME_OF_BUCKET'
  ...
end
~~~

With the required bucket in place and SparkleFormation CLI configured we can
now create our `infrastructure` template.

Create a new file: `./sparkleformation/infrastructure.rb`

#### Template sparkles AWS

~~~ruby
SparkleFormation.new(:infrastructure) do
  nest!(:network, :infra)
  nest!(:computes, :infra)
end
~~~

This new `infrastructure` template is using SparkleFormation's builtin nesting
functionality to create stack resources within our `infrastructure` template
composed of our `network` and `computes` template. To see the conceptual result
of this nesting, we can print the `infrastructure` template:

~~~
$ bundle exec sfn print --file infrastructure
~~~

There are a few things of note in this output. First, the `Stack` property is
not a real resource property. It is used by SparkleFormation for template processing
and is included in print functions to display the template in its entirety. Next,
the generated URLs are not real URLs. This is due to the SparkleFormation CLI not
actually storing the templates in the remote bucket. Lastly, and most importantly,
the parameters property of the `ComputesInfra` resource.

~~~json
"Parameters": {
  "NetworkVpcId": {
    "Fn::GetAtt": [
      "NetworkInfra",
      "Outputs.NetworkVpcId"
    ]
  },
  "NetworkSubnetId": {
    "Fn::GetAtt": [
      "NetworkInfra",
      "Outputs.NetworkSubnetId"
    ]
  }
}
~~~

SparkleFormation registers outputs when processing templates and will automatically
map outputs to subsequent stack resource parameters if they match. Cross stack resource
dependencies are now explicitly defined allowing the orchestration API to automatically
determine creation order as well as triggering updates when required.

Now we can create our full infrastructure with a single command:

~~~
$ bundle exec sfn create sparkle-guide-infrastructure --file infrastructure
~~~

As SparkleFormation processes the nested templates for the `create` command, the SparkleFormation
CLI will extract the nested templates, store them in the configured nesting bucket, and updates
the template location URL in the resource.

Using nested templates, `update` commands follow the same behavior as `create` commands. All nested
templates are extracted and automatically uploaded prior to execution of the `update` request with
the orchestration API. This results in _all_ nested stacks being automatically updated by the API
as required based on dependent resource modifications.

## Apply nested stack

Nested stacks can be applied to disparate stacks in the same manner described in the apply
stack implementation section. When the stack to be applied is a nested stack, SparkleFormation
CLI will gather outputs from all the nested stacks, and then apply to the target command. This
means using the `sparkle-guide-infrastructure` stack we previously built can be used for creating
new `computes` stacks without being nested into the `infrastructure` template.

~~~
$ bundle exec sfn create sparkle-guide-computes-infra --file computes --apply-stack sparkle-guide-infrastructure
~~~

The ability to apply nested stacks to disparate stacks make it easy to provide resources to new
stacks, or to test building new stacks in isolation before being nested into the root stack.

You can destroy the all the `infrastructure` related stacks with the command:

~~~
sfn destroy sparkle-guide-infrastructure
~~~

## Cross location

SparkleFormation supports accessing stacks in other locations when using the
apply stack functionality. This is done by configuring named locations within
the `.sfn` configuration file and then referencing the location when applying
the stack. Using this example `.sfn` configuration file:

~~~ruby
Configuration.new do
  credentials do
    provider :aws
    aws_access_key_id 'KEY'
    aws_secret_access_key 'SECRET'
    aws_region 'us-east-1'
    west_stacks do
      provider :aws
      aws_access_key_id 'KEY'
      aws_secret_access_key 'SECRET'
      aws_region 'us-west-2'
    end
  end
end
~~~

This configuration connects to the AWS API in us-east-1. It also includes configuration
for connecting to us-west-2 via a named location of `west_stacks`. This allows referencing
stacks located in us-west-2 when using the apply stack functionality. Assume a new stack
is being created in us-east-1 (my-stack) and the outputs from other-stack in us-west-2
should be applied. This can be accomplished with the following command:

~~~
sfn create my-stack --apply-stack west_stacks__my-stack
~~~

When the stack is created sfn will first connect to the us-west-2 API and gather the
outputs from other-stack, then it will connect to us-east-1 to create the new stack.

_NOTE: Custom locations are not required to have a common provider._
