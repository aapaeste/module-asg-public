**Note**: This public repo contains the documentation for the private GitHub repo <https://github.com/gruntwork-io/module-asg>.
We publish the documentation publicly so it turns up in online searches, but to see the source code, you must be a Gruntwork customer.
If you're already a Gruntwork customer, the original source for this file is at: <https://github.com/gruntwork-io/module-asg/blob/master/modules/server-group/README.md>.
If you're not a customer, contact us at <info@gruntwork.io> or <http://www.gruntwork.io> for info on how to get access!

# Server Group Module

This module allows you to run a fixed-size cluster of servers that can:

1. Attach [EBS Volumes](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumes.html) to each server. 
1. Attach [Elastic Network Interfaces (ENIs)](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html) to
   each server.
1. Do a zero-downtime, rolling deployment, where each server is shut down, the EBS Volume and/or ENI detached, and new
   server is brought up that reattaches the EBS Volume and/or ENI.
1. Integrate with an [Application Load Balancer 
   (ALB)](https://aws.amazon.com/elasticloadbalancing/applicationloadbalancer/) or [Elastic Load Balancer 
   (ELB)](https://aws.amazon.com/elasticloadbalancing/classicloadbalancer/) for routing and health checks.   
1. Automatically replace failed servers.  

The main use case for this module is to run data stores such as MongoDB and ZooKeeper. See the [Background 
section](#background) to understand how this module works and in what use cases you should use it instead of an Auto 
Scaling Group (ASG).





## Quick start

Check out the [server-group examples](/examples/server-group) for sample code that demonstrates how to use this module.





## How do you use this module?

To use this module, you need to do the following:

1. [Add the module to your Terraform code](#add-the-module-to-your-terraform-code)
1. [Optionally create an ENI and EBS Volume for each server](#optionally-create-an-eni-and-ebs-volume-for-each-server)
1. [Optionally integrate a load balancer for health checks](#optionally-integrate-a-load-balancer-for-health-checks)
1. [Attach an ENI and EBS Volume during boot](#attach-an-eni-and-ebs-volume-during-boot)


### Add the module to your Terraform code

As with all Terraform modules, you include this one in your code using the `module` keyword and pointing the `source`
URL at this repo:

```hcl
module "servers" {
  source = "git::git@github.com:gruntwork-io/module-asg.git//modules/server-group?ref=v0.3.1"

  name          = "my-server-group"
  size          = 3
  instance_type = "t2.micro"
  ami_id        = "ami-abcd1234"

  aws_region = "us-east-1"
  vpc_id     = "vpc-abcd12345"
  subnet_ids = ["subnet-abcd1111", "subnet-abcd2222", "subnet-abcd3333"]
}
```

The code above will spin up 3 `t2.micro` servers in the specified VPC and subnets. Any server that fails EC2 status 
checks will be automatically replaced. Any time you update any of the parameters and run `terraform apply`, it will
kick off a zero-downtime rolling deployment (see [How does rolling deployment work?](#how-does-rolling-deployment-work)
for details).


### Optionally integrate a load balancer for health checks

By default, the server-group module uses [EC2 status 
checks](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html) to determine
server health. This is used both during a rolling deployment (i.e., only replace the next server when the previous 
server is healthy) and for auto-recovery (i.e., replace any server that has failed). While EC2 status checks are good
enough to detect when the EC2 Instance has completely died or is malfunctioning, they do NOT determine if the code 
running on that EC2 Instance is actually working (e.g., is your database or application actually running and capable
of serving traffic).
 
Therefore, we strongly recommend associating a load balancer with your server-group. The load balancer can perform 
health checks on your application code by actually making HTTP or TCP requests to the application, which is a far more
robust way to tell if the server is healthy. 

Here is how to associate an ELB with your server-group and use it for health checks:

```hcl
module "servers" {
  source = "git::git@github.com:gruntwork-io/module-asg.git//modules/server-group?ref=v0.3.1"
  
  # (other params omitted)
    
  health_check_type = "ELB"
  elb_names         = ["${aws_elb.my_elb.name}"]
}
```

And here is how to associate an ALB with your server-group module and use it for health checks:

```hcl
module "servers" {
  source = "git::git@github.com:gruntwork-io/module-asg.git//modules/server-group?ref=v0.3.1"
  
  # (other params omitted)
    
  health_check_type     = "ELB"
  alb_target_group_arns = ["${aws_alb_target_group.my_target_group.arn}"]
}
```

*Note: The `health_check_type` value above is not a typo, it should be `ELB` in both cases.*

Once you've associated a load balancer with your server-group, new servers will automatically register with the load
balancer while deploying, deregister while undeploying, and use the load balancer's health checks to determine when a
server is healthy or needs replacing.


### Optionally create ENIs and EBS Volumes for each server

By default, the server-group module does not create any ENIs or EBS Volumes. If you would like to create ENIs, set the 
`num_enis` parameter to the number of ENIs you want per server:

```hcl
module "servers" {
  source = "git::git@github.com:gruntwork-io/module-asg.git//modules/server-group?ref=v0.3.1"
  
  # (other params omitted)
    
  num_enis = 1
}
```

If you would like to create EBS Volumes, set the `ebs_volumes` parameter to a list of volumes to create for each 
server:

```hcl
module "servers" {
  source = "git::git@github.com:gruntwork-io/module-asg.git//modules/server-group?ref=v0.3.1"
  
  # (other params omitted)
    
  ebs_volumes = [{
    type      = "gp2"
    size      = 100
    encrypted = false
  },{
    type      = "standard"
    size      = 500
    encrypted = true
  }]
}
```

Each ENI and server pair will get a matching `eni-xxx` tag (e.g., `eni-0`, `eni-1`, etc). Each EBS Volume and server 
pair will get a matching `ebs-volume-xxx` tag (e.g., `ebs-volume-0`, `ebs-volume-1`, etc). You will need to attach 
these ENIs and Volumes while your server is booting, as described in the next section.


### Attach an ENI and EBS Volume during boot

While the server-group module can create ENIs and EBS Volumes for you, you have to attach them to your servers 
yourself. The easiest way to do that is to use the following modules from 
[module-server](https://github.com/gruntwork-io/module-server-public):

* [attach-eni](https://github.com/gruntwork-io/module-server-public/tree/master/modules/attach-eni)
* [persistent-ebs-volume](https://github.com/gruntwork-io/module-server-public/tree/master/modules/persistent-ebs-volume)

Here's how it works:
 
1. Install the `attach-eni` and/or `persistent-ebs-volume` modules in the AMI that gets deployed in your server-group.
   The easiest way to do this is to use the [Gruntwork Installer](https://github.com/gruntwork-io/gruntwork-installer)
   in a [Packer](https://www.packer.io/) template:
   
    ```bash
    gruntwork-install --module-name 'persistent-ebs-volume' --repo 'https://github.com/gruntwork-io/module-server' --tag 'v0.1.10'
    gruntwork-install --module-name 'attach-eni' --repo 'https://github.com/gruntwork-io/module-server' --tag 'v0.1.10'
    ```

1. Run the `attach-eni` and/or `mount-ebs-volume` scripts while each server is booting, typically as part of
   [User Data](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html#user-data-api-cli). You can use the
   `--eni-with-same-tag` and `--volume-with-same-tag` parameters of the scripts, respectively, to automatically mount
   the ENIs and/or EBS Volumes with the same `eni-xxx` and `ebs-volume-xxx` tags as the server.





## Background 

1. [Why not an Auto Scaling Group?](#why-not-an-auto-scaling-group)
1. [How does this module work?](#how-does-this-module-work)
1. [How does rolling deployment work?](#how-does-rolling-deployment-work)


### Why not an Auto Scaling Group?

The first question you may ask is, how is this different than an [Auto Scaling Group 
(ASG)](http://docs.aws.amazon.com/autoscaling/latest/userguide/AutoScalingGroup.html)? While an ASG does allow you to
run a cluster of servers, automaticaly replace failed servers, and do zero-downtime deployment (see the 
[asg-rolling-deploy module](/modules/asg-rolling-deploy)), attaching ENIs and EBS Volumes to servers in an ASG is very
tricky:
 
1. Using ENIs and EBS Volumes with ASGs is not natively supported by Terraform. The 
   [aws_network_interface_attachment](https://www.terraform.io/docs/providers/aws/r/network_interface_attachment.html)
   and [aws_volume_attachment](https://www.terraform.io/docs/providers/aws/r/volume_attachment.html) resources only 
   work with individual EC2 Instances and not ASGs. Therefore, you typically create a pool of ENIs and EBS Volumes in 
   Terraform, and your servers, while booting, use the AWS CLI to attach those ENIs and EBS Volumes.

1. Attaching ENIs and EBS Volumes from a pool requires that each server has a way to uniquely pick which ENI or EBS
   Volume belongs to it. Picking at random and retrying can be slow and error prone.

1. With EBS Volumes, attaching them from an ASG is particularly problematic, as you can only attach an EBS Volume in
   the same Availability Zone (AZ) as the server. If you have, for example, three AZs and five servers, it's entirely
   possible that the ASG will launch a server in an AZ that does not have any EBS Volumes available. 
   
The goal of this module is to give you a way to run a cluster of servers where attaching ENIs and EBS Volumes is easy.


### How does this module work?

The solution used in this module is to:

1. Create one ASG for each server. So if you create a cluster with five servers, you'll end up with five ASGs. Using
   ASGs gives us the ability to automatically integrate with the ALB and ELB and to replace failed servers.
1. Each ASG is assigned to exactly one subnet, and therefore, one AZ.
1. Create ENIs and EBS Volumes for each server, in the same AZ as that server's ASG. This ensures a server will
   never launch in an AZ that doesn't have an EBS Volume.
1. Each server and ENI pair and each server and EBS Volume pair get matching tags, so each server can always uniquely 
   identify the ENIs and EBS Volumes that belong to it. 
1. Zero-downtime deployment is done using a Python script in a [local-exec 
   provisioner](https://www.terraform.io/docs/provisioners/local-exec.html). See [How does rolling deployment 
   work?](#how-does-rolling-deployment-work) for more details.
   




## How does rolling deployment work?

The server-group module will perform a zero-downtime, rolling deployment every time you make a change to the code and
run `terraform apply`. This deployment process is implemented in a Python script called 
[rolling_deployment.py](rolling-deploy/rolling_deployment.py) which runs in a [local-exec 
provisioner](https://www.terraform.io/docs/provisioners/local-exec.html). 

Here is how it works:

1. [The rolling deployment process](#the-rolling-deployment-process)
1. [Deployment configuration options](#deployment-configuration-options)


### The rolling deployment process

The rolling deployment process works as follows:

1. Wait for the server-group to be healthy before starting the deployment. That means the server-group has the expected 
   number of servers up and running and passing health checks.

1. Pick one of the ASGs in the server-group and set its size to 0. This will terminate the Instance in that ASG, 
   respecting any connection draining settings you may have set up. It will also detach any ENI or EBS Volume.

1. Once the instance has terminated, set the ASG size back to 1. This will launch a new Instance with your new code
   and reattach its ENI or EBS Volume.

1. Wait for the new Instance to pass health checks.

1. Once the Instance is healthy, repeat the process with the next ASG, until all ASGs have been redeployed.


### Deployment configuration options

You can customize the way the rolling deployment works by specifying the following parameters to the server-group
module in your Terraform code:

* `script_log_level`: Specify the [logging level](https://docs.python.org/2/library/logging.html#logging-levels) the
  script should use. Default is `INFO`. To debug issues, you may want to turn it up to `DEBUG`. To quiet the script 
  down, you may want to turn it down to `ERROR`.

* `deployment_batch_size`: How many servers to redeploy at a time. The default is 1. If you have a lot of servers to
  redeploy, you may want to increase this number to do the deployment in larger batches. However, make sure that taking 
  down a batch of this size does not cause an unintended outage for your service!

* `skip_health_check`: If set to `true`, the rolling deployment process will not wait for the server-group to be 
  healthy before starting the deployment. This is useful if your server-group is already experiencing some sort of 
  downtime or problem and you want to force a deployment as a way to fix it.

* `skip_rolling_deploy`: If set to `true`, skip the rolling deployment process entirely. That means your Terraform 
  changes will be applied to the launch configuration underneath the ASGs, but no new code will be deployed until
  something triggers the ASG to launch new instances. This is primarily useful if the rolling deployment script turns
  out to have some sort of bug in it.
