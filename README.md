**Note**: This public repo contains the documentation for the private GitHub repo <https://github.com/gruntwork-io/module-asg>.
We publish the documentation publicly so it turns up in online searches, but to see the source code, you must be a Gruntwork customer.
If you're already a Gruntwork customer, the original source for this file is at: <https://github.com/gruntwork-io/module-asg/blob/master/README.md>.
If you're not a customer, contact us at <info@gruntwork.io> or <http://www.gruntwork.io> for info on how to get access!

# Auto Scaling Group Modules

This repo contains modules for creating and managing a best-practices [Auto Scaling Group
(ASG)](https://aws.amazon.com/autoscaling/).

The main modules are:

* [asg-rolling-deploy-dynamic](/modules/asg-rolling-deploy-dynamic): This module creates an ASG that can do a
  zero-downtime rolling deployment using [CloudFormation](https://aws.amazon.com/cloudformation/). This allows you to
  size the ASG both statically (e.g. by manually setting a fixed size) or dynamically (e.g. via Auto Scaling Policies).
* [asg-rolling-deploy-static](/modules/asg-rolling-deploy-static): This module creates an ASG that can do a
  zero-downtime rolling deployment using pure Terraform code. This only allows you to set the size of the ASG
  statically, by manually setting a fixed size.

The supporting modules are:

* [cfn-signal-go](/modules/cfn-signal-go): A Go implementation of Amazon's Python `cfn-signal` script that can be used
  to signal CloudFormation when an EC2 Instance is up and running so CloudFormation knows if a rolling deployment
  worked or needs to be rolled back. This Go implementaiton allows us to create cross-platform standalone binaries
  instead of relying on Python 2.x.
* [cloudformation-scripts](/modules/cloudformation-scripts): Install the Amazon [CloudFormation Helper
  Scripts](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-helper-scripts-reference.html), on your
  EC2 Instances, including the Python `cfn-signal` script, which can be used to signal CloudFormation when an EC2
  Instance is up and running so CloudFormation knows if a rolling deployment worked or needs to be rolled back.
* [render-if-equal](/modules/render-if-equal): This module contains a hacky way to support if-statements in Terraform.
  Terraform does not have support for conditional logic, and usually you don't need it, but in the rare cases where you
  do, this module contains a workaround.

## What is a module?

At [Gruntwork](http://www.gruntwork.io), we've taken the thousands of hours we spent building infrastructure on AWS and
condensed all that experience and code into pre-built **packages** or **modules**. Each module is a battle-tested,
best-practices definition of a piece of infrastructure, such as a VPC, ECS cluster, or an Auto Scaling Group. Modules
are versioned using [Semantic Versioning](http://semver.org/) to allow Gruntwork clients to keep up to date with the
latest infrastructure best practices in a systematic way.

## How do you use a module?

Most of our modules contain either:

1. [Terraform](https://www.terraform.io/) code
1. Scripts & binaries

#### Using a Terraform Module

To use a module in your Terraform templates, create a `module` resource and set its `source` field to the Git URL of
this repo. You should also set the `ref` parameter so you're fixed to a specific version of this repo, as the `master`
branch may have backwards incompatible changes (see [module
sources](https://www.terraform.io/docs/modules/sources.html)).

For example, to use `v1.0.8` of the ecs-cluster module, you would add the following:

```hcl
module "ecs_cluster" {
  source = "git::git@github.com:gruntwork-io/module-ecs.git//modules/ecs-cluster?ref=v1.0.8"

  // set the parameters for the ECS cluster module
}
```

*Note: the double slash (`//`) is intentional and required. It's part of Terraform's Git syntax (see [module
sources](https://www.terraform.io/docs/modules/sources.html)).*

See the module's documentation and `vars.tf` file for all the parameters you can set. Run `terraform get -update` to
pull the latest version of this module from this repo before runnin gthe standard  `terraform plan` and
`terraform apply` commands.

#### Using scripts & binaries

You can install the scripts and binaries in the `modules` folder of any repo using the [Gruntwork
Installer](https://github.com/gruntwork-io/gruntwork-installer). For example, if the scripts you want to install are
in the `modules/ecs-scripts` folder of the https://github.com/gruntwork-io/module-ecs repo, you could install them
as follows:

```bash
gruntwork-install --module-name "ecs-scripts" --repo "https://github.com/gruntwork-io/module-ecs" --tag "0.0.1"
```

See the docs for each script & binary for detailed instructions on how to use them.

## What's an Auto Scaling Group?

An [Auto Scaling Group](https://aws.amazon.com/autoscaling/) (ASG) is used to manage a cluster of EC2 Instances. It
can enforce pre-defined rules about how many instances to run in the cluster, scale the number of instances up or
down depending on traffic, and automatically restart instances if they go down.

## Developing a module

#### Versioning

We are following the principles of [Semantic Versioning](http://semver.org/). During initial development, the major
version is to 0 (e.g., `0.x.y`), which indicates the code does not yet have a stable API. Once we hit `1.0.0`, we will
follow these rules:

1. Increment the patch version for backwards-compatible bug fixes (e.g., `v1.0.8 -> v1.0.9`).
2. Increment the minor version for new features that are backwards-compatible (e.g., `v1.0.8 -> 1.1.0`).
3. Increment the major version for any backwards-incompatible changes (e.g. `1.0.8 -> 2.0.0`).

The version is defined using Git tags.  Use GitHub to create a release, which will have the effect of adding a git tag.

#### Tests

See the [test](/test) folder for details.