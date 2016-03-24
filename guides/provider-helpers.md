---
title: "Provider Helpers"
weight: 3
anchors:
  - title: "Overview"
    url: "#overview"
  - title: "AWS helpers"
    url: "#aws-helpers"
  - title: "Azure helpers
    url: "#azure-helpers"
  - title: "HEAT helpers"
    url: "#heat-helpers"
---

## Overview

The SparkleFormation library supports multiple orchestration APIs,
each with their own structures for defining intrinsic functions. SparkleFormation
provides helper methods to build these structures, and the available
methods are dependent on the provider defined for the given template.

The provider specific helpers methods included by SparkleFormation are
included to aid in the generation of common data structures. This helps
to reduce the size of templates as well as making composition easier.

## AWS helpers

Templates will automatically load AWS helper methods when no provider
is explicitly defined. The AWS helpers provide access to all intrinsic
functions provided by CloudFormation.

~~~ruby
SparkleFormation.new(:my_template, :provider => :aws) do

end
~~~

* [All AWS helpers](https://sparkleformation.github.io/sparkle_formation/SparkleFormation/SparkleAttribute/Aws.html)

## Azure helpers

## HEAT helpers