**Note**: This public repo contains the documentation for the private GitHub repo <https://github.com/gruntwork-io/module-asg>.
We publish the documentation publicly so it turns up in online searches, but to see the source code, you must be a Gruntwork customer.
If you're already a Gruntwork customer, the original source for this file is at: <https://github.com/gruntwork-io/module-asg/blob/master/examples/server-group/README.md>.
If you're not a customer, contact us at <info@gruntwork.io> or <http://www.gruntwork.io> for info on how to get access!

# Server Group Examples

This folder shows examples of how to use the [server-group module](/modules/server-group) to deploy a cluster of
servers that can automatically attach EBS Volumes and ENIs, do zero-downtime rolling deployment, and automatically 
replace failed servers:

* [ami](./ami): An example AMI meant to be used with all the other examples listed below. It installs the `attach-eni`
  and `mount-ebs-volume` scripts used by each server during boot to attach an ENI and EBS Volume, respectively.
* [with-elb](./with-elb): An example of how to deploy a server-group that uses an ELB for health checks.
* [with-alb](./without-alb): An example of how to deploy a server-group that uses an ALB for health checks.
* [without-load-balancer](./without-load-balancer): An example of how to deploy a server-group that uses EC2 status
  checks instead of load balancers for health checks.

On each EC2 Instance in the server-group examples, for demonstration and testing purposes, we run a dummy web server 
that just returns "Hello World". If you update this app (e.g. change the text to "Hello, World v2" or replace the app 
with a totally different AMI of your own), the next time you run `terraform apply`, the new version of the app will 
roll out automatically, with no downtime.




## How do you run these examples?

Prerequisites: Install [Packer](https://www.packer.io/) and [Terraform](https://www.terraform.io/). Also, make sure you 
have Python installed (version 2.x) and in your `PATH`.

1. Build the AMI: `packer build ami/server.json`.
1. Open `vars.tf`, set the environment variables specified at the top of the file, and fill in any other variables that
   don't have a default, including the ID of the AMI you built in the previous step.
1. Run `terraform get`.
1. Run `terraform plan`.
1. If the plan looks good, run `terraform apply`.




## Deploying a new version

Once you have the app up and running, to test out the rolling deployment, make a change to any of the variables passed
into the server-group module. For example, as a quick test, you could change the `server_text` variable to "Hello, 
World, v2.0!". Once you've made your changes, do the following:

1. Commit your changes to Git so your teammates have access to them.
1. Run `terraform plan`.
1. If the plan looks good, run `terraform apply`.

The new version of the app will automatically deploy across the server-group with no downtime. Check out the 
[server-group module documentation](/modules/server-group) for details on how this works. 
