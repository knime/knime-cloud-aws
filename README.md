
# Using KNIME Executors in AWS

This repository provides information about how to get started using KNIME Executors in the AWS cloud platform.
It also includes sample AWS CloudFormation and Terraform templates to help you get started quickly. Feel free to copy
the templates (or snippets from them) and reuse them as needed. See the *license.txt* file in the repository
for licensing specifics.

KNIME Executors are used by the [KNIME Server](https://www.knime.com/knime-server) to run KNIME Workflows.
In the KNIME Server, workflows can be executed
through a schedule, a REST API call, remotely using the KNIME Analytics Platform and through the KNIME WebPortal.
When a workflow is ready to run, the KNIME Server employs an Executor for the actual execution.

The diagram below illustrates a KNIME Server and multiple KNIME Executors running in AWS. A virtual private cloud network (VPC)
is required with at least one subnet. The KNIME Executors can be deployed using an Amazon Autoscaling Group (ASG).
The ASG manages the Executor instances and supports elastic scaling of Pay-as-you-go (PAYG) Executors based on the
average CPU utilization of instances in the ASG.

![Architecture Diagram](/images/executor_topology_aws.png)

You can run Executors without an ASG. However, an ASG provides reliability and recoverability features even for
BYOL Executors. The list below emphasis several important ASG features:

* Maintaining a balance of instances across availability zones
* Monitoring the health of instances and replacing unhealthy instances with healthy ones
* Replacing instances if they terminate unexpectedly
* Supporting autoscaling when using the PAYG edition of KNIME Executors

KNIME Executors are supported in the AWS Marketplace in the following forms.

* Pay As You Go (**PAYG**) instances are charged to your AWS account per hour. PAYG supports elastic scaling.
* Bring Your Own License (**BYOL**) instances are licensed through the KNIME Server using core tokens. Contact
KNIME at *sales@knime.com* for more information.

This repository contains an AWS CloudFormation template for both the **BYOL** and **PAYG** offerings.

---

## Getting Started

### Enable Marketplace Image Usage

To use a VM image from the AWS Marketplace in a CloudFormation template (i.e. programmatically) you first have to
subscribe to the marketplace offering. To enable access, go to the AWS Marketplace and find the KNIME Executor
of choice (BYOL or PAYG). Click on the *Continue to Subscribe* button to start the subscription process.

Until you subscribe for an Executor offering you will be unable to start Executors using the CloudFormation
templates contained in this project.

### Using a Template

Before deploying KNIME Executor(s), you'll first want to deploy the KNIME Server. When deploying one of the Executor templates,
ensure you use the same AWS VPC where the KNIME Server is deployed.

To use one of the CloudFormation templates you can either clone this project to your local filesystem or simply copy the text of
the template. There are multiple ways you can use CloudFormation templates, but we'll focus here on using them in the AWS Console.

Start by logging into the AWS Console. Search for *CloudFormation* and select the link to be forward to the CloudFormation console.
Select the *Create Stack* dropdown and the *With new resources (standard)* item. If you've uploaded the template to S3, provide the
S3 location. Otherwise browse your local filesystem for the template. Select *Next* and follow directions on the subsequent pages
to launch the Executor stack.

---

## AWS Autoscaling Group (ASG) Configuration

The AWS CloudFormation templates within the repository support the following ASG features. These must be enabled to allow the Executors
to function properly within the ASG deployment.

* Instance Monitoring
* Autoscaling with Metrics (**PAYG** only)
* Autoscaling Group (ASG) Lifecycle Hooks

### Instance Monitoring

The Instance Monitoring feature is enabled by default. The ASG monitors each instance to ensure the instance is healthy. Any
instances that are found to be unhealthy are terminated and replaced with a new instance.

The KNIME Executors don't support a health check that can be used by the ASG. The Instance Monitoring that is enabled checks the health
of the operating system and the underlying VM, but not the KNIME Executor. A health check script is periodically run on the Executor
to ensure the Executor process is running. If the Exeuctor process is not running, the health check fails and a command is run
to inform the ASG that the Executor is unhealthy and should be cycled.

### Autoscaling with Metrics (**PAYG only**)

[Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
supports elastic scaling. Elastic scaling allows you to provide a range of the number of Executor instances to deploy.  The ASG
will manage the number of deployed instances according to defined metrics. The **PAYG** example CloudFormation template
uses the average CPU utilization
of the ASG to support scaling events. When the upper CPU utilization threshold is passed, the ASG will create a scale event to add more
Executor instances. Likewise when the CPU utilization falls below the lower threshold, the ASG will pick instance(s) to terminate.

The **PAYG** template enables you to specify the minimum and maximum number of Executor instances to deploy. The ASG will initially
deploy with the minimum number of instances. As load increases, the ASG will automatically create new Executor instances.

You may also specify the upper and lower thresholds for the average CPU utilization metric. The upper threshold is the
point above which more Executor(s) are needed. The lower threshold is the point below which fewer Executors are needed.

The configuration of the ASG settings are fairly minimal. Other settings such as notifications can also be enabled by
extending the **PAYG** template.

### ASG Lifecycle Hooks

AWS Autoscaling Groups support lifecycle hooks that allow KNIME Executors to gracefully handle a terminate request.
An Executor may be picked for termination due to manually lowering the desired number of instances or due to a scaling
event such as the ASG CPU percentage falling below the lower threshold. Either way, Executors watch for a
change in lifecycle indicating the a termination is in progress.

When a termination is in progress, the Executor will stop accepting new job requests and will wait for any currently
running jobs to complete. The ASG has a maximum amount of time that it will wait for an Executor to allow the termination
to complete. If maximum time expires, the Executor will be forcefully terminated. As such, any long running jobs
on a terminated instance will fail.

---

## Custom Data

Amazon AWS supports providing custom data to a virtual machine. The custom data can be a shell script to execute,
a text file or a cloud-init directive.

KNIME Executors expect a cloud-init directive. This directive specifies the needed parameters that must be passed to the Executor
to allow it to find and connect to the KNIME Server.

The example AWS CloudFormation templates configure the custom data and pass the data to each VM within an autoscaling group.
There is need for you to change this. The purpose of this section is to inform you of the custom data in case you want to use a different method than CloudFormation to deploy KNIME Executors.

If you decide to use a different technology to create ASG instances for KNIME Executors, be sure to use the format specified below. Change
the values in the *knime-executor.config* section to match your configuration. The example below provides sample values to
demonstrate the structure of the *cloud-config* data.

```
#cloud-config
output : { all : '| tee -a /var/log/cloud-init-output.log' }
write_files:
  - owner: knime:knime
    path: /var/opt/knime-executor.config
    content: |
      KNIME_SERVER_HOST=10.0.0.4
      KNIME_VIRTUAL_HOST=knime-server
      KNIME_RMQ_USER=knime
      KNIME_RMQ_PASSWORD=knime
      KNIME_EXECUTOR_GROUP=payg-group
      KNIME_EXECUTOR_RESOURCES=
      KNIME_EXECUTOR_HEAP_USAGE_PERCENT_LIMIT=90
      KNIME_EXECUTOR_CPU_USAGE_PERCENT_LIMIT=85
runcmd:
  - /opt/knime/knime-utils/configure_executor.sh
```

The table below provides more information about each configuration item variable in the cloud-init configuration for KNIME Executors.

| Configuration Item | Description |
| ------------------ | ----------- |
| KNIME_SERVER_HOST | The private IP address of the KNIME Server or RabbitMQ server |
| KNIME_VIRTUAL_HOST | The RabbitMQ virtual host (vhost) configured for the KNIME Server |
| KNIME_RMQ_USER | The RabbitMQ user configured for the KNIME Server |
| KNIME_RMQ_PASSWORD | The RabbitMQ user password configured for the KNIME Server |
| KNIME_EXECUTOR_GROUP | The Executor Group membership of all Executors in the autoscaling group |
| KNIME_EXECUTOR_RESOURCES | The resources associated with all Executors in the autoscaling group |
| KNIME_EXECUTOR_HEAP_USAGE_PERCENT_LIMIT | Memory usage threshold above which an Executor stops accepting new work |
| KNIME_EXECUTOR_CPU_USAGE_PERCENT_LIMIT  | CPU usage threshold above which an Executor stops accepting new work |

For detailed information about these parameters, refer to the [KNIME Server Admin Guide](https://docs.knime.com/2020-07/server_admin_guide/index.html).

---

## CloudFormation Template Parameters

### **BYOL** Template

The **BYOL** CloudFormation template enables users to provide parameter values that affect the deployment. When using the template
you will have an opportunity to provide values that match your deployment. The table below provides more information about the
supported parameters.

| Parameter             | Description                                                                                         |
|-----------------------|-----------------------------------------------------------------------------------------------------|
| vpc                   | The Virtual Private Cloud (VPC) instance into which Executors should be deployed.                   |
| vpcCIDR               | The IP range (CIDR notation) for the selected VPC used for Executor deployment.                     |
| subnet1               | The identifier of the first subnet to use (which is tied to an Availability Zone).                  |
| subnet2               | The identifier of the second subnet to use (which is tied to an Availability Zone).                 |
| serverHost            | The private IP address of the KNIME Server of the RabbitMQ server (if they are different)           |
| virtualHost           | The RabbitMQ Virtual host (Vhost) configured for the KNIME Server                                   |
| rmqUser               | The RabbitMQ user configured for the KNIME Server                                                   |
| rmqPassword           | The RabbitMQ user password configured for the KNIME Server                                          |
| keyPair               | The name of the Key Pair used for all executor instances to be created                              |
| instanceType          | The EC2 instance type to use for executor instances                                                 |
| instanceCount         | The number of Executor instances wanted. Can be overridden by ASG settings.                         |
| executorGroup         | The Executor Group membership of all Executors in the autoscaling group                             |
| executorResources     | The resources associated with all Executors in the autoscaling group                                |
| heapUsagePercentLimit | Memory usage threshold above which an Executor stops accepting new work                             |
| cpuUsagePercentLimit  | CPU usage threshold above which an Executor stops accepting new work                                |
| heartbeatTimeout      | The amount of time an Executor has to respond to an ASG terminate request.                          |

### **PAYG** Template

The **PAYG** CloudFormation template enables users to provide parameter values that affect the deployment. When using the template
you will have an opportunity to provide values that match your deployment. The table below provides more information about the
supported parameters. The **PAYG** template supports elastic scaling. The *minNumberOfInstances* and *maxNumberOfInstances*
provide a range of the instance counts wanted in the ASG. Likewise the *upperCpuPercentLimit* and *lowerCpuPercentLimit* determine
the upper and lower thresholds used for elastic scaling of KNIME Executors in the ASG.

| Parameter             | Description                                                                                         |
|-----------------------|-----------------------------------------------------------------------------------------------------|
| vpc                   | The Virtual Private Cloud (VPC) instance into which Executors should be deployed.                   |
| vpcCIDR               | The IP range (CIDR notation) for the selected VPC used for Executor deployment.                     |
| subnet1               | The identifier of the first subnet to use (which is tied to an Availability Zone).                  |
| subnet2               | The identifier of the second subnet to use (which is tied to an Availability Zone).                 |
| serverHost            | The private IP address of the KNIME Server of the RabbitMQ server (if they are different)           |
| virtualHost           | The RabbitMQ Virtual host (Vhost) configured for the KNIME Server                                   |
| rmqUser               | The RabbitMQ user configured for the KNIME Server                                                   |
| rmqPassword           | The RabbitMQ user password configured for the KNIME Server                                          |
| keyPair               | The name of the Key Pair used for all executor instances to be created                              |
| instanceType          | The EC2 instance type to use for executor instances                                                 |
| executorGroup         | The Executor Group membership of all Executors in the autoscaling group                             |
| executorResources     | The resources associated with all Executors in the autoscaling group                                |
| heapUsagePercentLimit | Memory usage threshold above which an Executor stops accepting new work                             |
| cpuUsagePercentLimit  | CPU usage threshold above which an Executor stops accepting new work                                |
| minExecutorCount      | The minimum number of Executor instances to maintain in the ASG                                     |
| maxExecutorCount      | The maximum number of Executor instances to maintain in the ASG                                     |
| targetUtilization     | The target average CPU utilization for the scaling group of executors.                              |
| coolDown              | Period in seconds after a scaling event before another scaling event is allowed                     |
| heartbeatTimeout      | The amount of time an Executor has to respond to an ASG terminate request.                          |

For detailed information about these parameters, refer to the [KNIME Server Admin Guide](https://docs.knime.com/2020-07/server_admin_guide/index.html).

---

## Things You May Want to Change

The CloudFormation templates contain a *RegionMap* that maps from an AWS region to the AMI to use within the region.
An AMI or Amazon Machine Image is the Executor machine image that is pre-built by KNIME. For each release of KNIME Executors,
the mapping of AMI's will change.

The autoscaling group (ASG) for the *PAYG* Executor offering uses [target tracking scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html). With this option, the
ASG attempts to keep the average CPU utilization of member Executors at the specified level. As the utilization goes above the
threshold, more Executors will be added to the ASG. As it goes below, Executor(s) will be terminated. Other options are
available for auto-scaling that may fit your usage better.

---

## Useful Links

* [KNIME Software on AWS](https://docs.knime.com/2020-12/aws_marketplace_server_guide/index.html)
* [KNIME Server](https://www.knime.com/knime-server)
* [KNIME Server Admin Guide](https://docs.knime.com/2020-07/server_admin_guide/index.html)
* [Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
* [Instance Monitoring with Health Checks](https://docs.aws.amazon.com/autoscaling/ec2/userguide/healthcheck.html)
* [ASG Dynamic Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scale-based-on-demand.html)
* [ASG Lifecycle Hooks](https://docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks.html)
* [Running Commands on EC2 Instances at Launch](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
