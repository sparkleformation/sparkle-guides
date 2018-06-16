---
title: "Tips and Tricks"
weight: 3
anchors:
  - title: "Stubbing builtins"
    url: "#stubbing-builtins"
  - title: "Resource conflicts"
    url: "#resource-conflicts"
---

## Stubbing builtins

SparkleFormation provides a number of builtin resources using the
`dynamic!` helper. These known resources are collected when SparkleFormation
is released. Because this is a static list of resource types it is
common for new resources to be introduced within a cloud provider
and not be available within SparkleFormation. Since the new resource
types will not be available within SparkleFormation until the next
release, stubbing the builtin will provide the same behavior as if
it were included within the builtin list.

For example, lets assume that AWS CloudFormation introduces a new
resource type named `AWS::JVM::Instance`. To stub this resource a
new dynamic can be created locally:

~~~ruby
SparkleFormation.dynamic(:jvm_instance) do |name|
  resources.set!("#{name}_jvm_instance".to_sym) do
    type "AWS::JVM::Instance"
  end
end
~~~

With this dynamic defined it will behave the same as if it were
defined in the builtin list:

~~~ruby
SparkleFormation.new(:test) do
  dynamic!(:jvm_instance, :test) do
    properties.memory 512
  end
end
~~~

which results in:

~~~json
{
  "Resources": {
    "TestJvmInstance": {
      "Type": "AWS::JVM::Instance",
      "Properties": {
        "Memory": 512
      }
    }
  }
}
~~~

Once the `AWS::JVM::Instance` becomes available in the builtin
list within SparkleFormation the custom stub dynamic can be deleted.

## Resource conflicts

Resource conflicts can occur when the same resource name is used
in multiple resource namespaces. If a template has created resources
using the `dynamic!` helper and not included the resource's namespace,
conflicts will occur when the new resource becomes available. An
example of this is the `AWS::ElasticLoadBalancing::LoadBalancer` resource.

Assume that the following template is in use:

~~~ruby
SparkleFormation.new(:lb) do
  dynamic!(:load_balancer, :app)
end
~~~

and it generates:

~~~json
{
  "Resources": {
    "AppLoadBalancer": {
      "Type": "AWS::ElasticLoadBalancing::LoadBalancer"
    }
  }
}
~~~

Now, AWS releases a new resource with a conflicting name
`AWS::ElasticLoadBalancingV2::LoadBalancer` and SparkleFormation's internal
resource list has been updated. When the template is run again, it returns
an error about conflicting resource names. One solution is to update the
`dynamic!` call to include the namespace:

~~~ruby
SparkleFormation.new(:lb) do
  dynamic!(:elastic_load_balancing_load_balancer, :app)
end
~~~

and it will then generate:

~~~json
{
  "Resources": {
    "AppElasticLoadBalancingLoadBalancer": {
      "Type": "AWS::ElasticLoadBalancing::LoadBalancer"
    }
  }
}
~~~

This resolves the conflict issue, yet in practice the modification of the
resource name will likely cause issues in complex templates where the
resource is being referenced in other locations. To allow templates to
remain unchanged, a new dynamic can be introduced which will force matching
of the `:load_balancer` name to the local dynamic:

~~~ruby
SparkleFormation.dynamic(:load_balancer) do |name|
  dynamic!(:elastic_load_balancing_load_balancer, name, :resource_name_suffix => :load_balancer)
end
~~~

With this dynamic in place, all original templates will continue to behave
as expected without individual modifications.
