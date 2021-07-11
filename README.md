<!-- note marker start -->
**NOTE**: This repo contains only the documentation for the private BoltsOps Pro repo code.
Original file: https://github.com/boltopspro/s3-antivirus/blob/master/README.md
The docs are publish so they are available for interested customers.
For access to the source code, you must be a paying BoltOps Pro subscriber.
If are interested, you can contact us at contact@boltops.com or https://www.boltops.com

<!-- note marker end -->

# S3 AntiVirus with ClamAV Blueprint

[![BoltOps Badge](https://img.boltops.com/boltops/badges/boltops-badge.png)](https://www.boltops.com)

![CodeBuild](https://codebuild.us-west-2.amazonaws.com/badges?uuid=eyJlbmNyeXB0ZWREYXRhIjoiMG5xajh6RU50LzVGL0FERE9xS1RNKzE4SDNNUDFBb3REZENwTHBuTTQ4STI5eHJMcXUvOHJ5Y0pQYWhYZFF6WitZWE4zM0RKYU5PVUNrcDgwdHl2TXBzPSIsIml2UGFyYW1ldGVyU3BlYyI6IkhtZ1FabUswSmFqZ25BT3kiLCJtYXRlcmlhbFNldFNlcmlhbCI6MX0%3D&branch=master)

This blueprint is a part of the [S3 AntiVirus Setup Instructions](https://github.com/boltopspro-docs/s3-antivirus/blob/master/docs/instructions-overview.md). Please refer to that doc for the setup steps.

This blueprint provisions the AutoScaling EC2 Worker Instances and related resources to scan S3 files for viruses. Blueprint provisions:

* Fleet of EC2 instances. Supports spot, on-demand, or mixed instance types.
* SQS Queue to hold the payloads from S3 Bucket Event Notification Configurations.
* Scaling Alarms and Policies to scale the fleet dynamically based on the SQS Queue length.
* Alerts SNS Topic notify when viruses are found.

Features:

* Auto-Updating CloudWatch Dashboard. Provide monitoring visibility on S3 Antivirus scanning. Docs: [CloudWatch Dashboard](docs/cloudwatch-dashboard.md).
* Auto-delete or tag files as necessary. The default s3 tag is `scan-status`.
* Run EC2 Instances in existing VPC private or public subnets.
* Supports scanning buckets from multiple regions.
* You can run one s3-antivirus stack for all of your buckets regardless of region. Or you can run multiple stacks for each region if you prefer. More docs in the [Single or Multiple Scan Stacks?](https://github.com/boltopspro-docs/s3-antivirus#single-or-multiple-scan-stacks) section.
* [ClamAV](https://www.clamav.net/) is used for scanning.
* To automatically enable s3-antivirus protection for new buckets on a go-forward basis use the [s3-antivirus-autoenable](https://github.com/boltopspro-docs/s3-antivirus-autoenable) blueprint.

## Diagrams

Here's a diagram focusing on the resources this blueprint creates:

![](https://img.boltops.com/boltopspro/blueprints/s3-antivirus/s3-antivirus-blueprint.png)

Here's a diagram to explain the overall flow:

![](https://img.boltops.com/boltopspro/blueprints/s3-antivirus/s3-antivirus-overview.png)

This blueprint provisions the SQS Queue and Scanning EC2 Instance.  The SNS Topics are added with the companion [boltopspro/s3-antivirus-cli](https://github.com/boltopspro-docs/s3-antivirus-cli) tool.

## Usage

1. Add blueprint to your Gemfile
2. Configure configs/s3-antivirus values
3. Deploy blueprint

## Add

Add the blueprint to your lono project's `Gemfile`.

```ruby
gem "s3-antivirus", git: "git@github.com:boltopspro/s3-antivirus.git", submodules: true
```

## Configure

Configure the `configs/s3-antivirus/params` and `configs/s3-antivirus/variables` files.

    LONO_ENV=development lono seed s3-antivirus

The generated files in `configs/s3-antivirus` folder look something like this:

    configs/s3-antivirus/
    ├── params
    │   └── development.txt
    └── variables
        └── development.rb

Here some example parameters:

configs/s3-antivirus/params/development.txt:

    # Parameter Group: AWS::AutoScaling::AutoScalingGroup
    MaxSize=8
    MinSize=1
    VPCZoneIdentifier=<%= default_subnets.join(',') %> # (required)

    # Parameter Group: AWS::AutoScaling::AutoScalingGroup MixedInstancesPolicy InstancesDistribution
    OnDemandPercentageAboveBaseCapacity=0 # 0 means all spot instances, 100 means all on-demand instances. Default is 100

    # Parameter Group: AWS::EC2::LaunchTemplate LaunchTemplateData
    InstanceType=t3.large
    KeyName=<%= key_pairs(/default/).first %>

    # Parameter Group: AWS::EC2::SecurityGroup
    VpcId=<%= default_vpc %> # Required. Though only used if SecurityGroupIds not set to create Security Group. # (required)

## Deploy

Let's deploy now with [lono cfn deploy](https://lono.cloud/reference/lono-cfn-deploy/).

    LONO_ENV=development lono cfn deploy s3-antivirus --sure

### Server Size

Clamd loads the DB `/var/lib/clamav/main.cvd` into memory, so the server requires enough memory for that. Recommend running a server that has at least 4GB of RAM. Here's a fresh server's memory footprint with clamd loaded.

    # free -m
                  total        used        free      shared  buff/cache   available
    Mem:           3884        1100        1361           0        1422        2539
    Swap:             0           0           0

### Subscribing to Main Alerts

You can subscribe to the `AlertsTopic` created by the CloudFormation template to receive notifications when viruses are found. You can find it under the CloudFormation Console Outputs tab.  You can subscribe to the AlertsTopic with the console or you can do so with code with configs variable:

configs/s3-antivirus/variables/development.rb:

```ruby
# AWS::SNS::Topic Main Queue
@alerts_topic_subscription = [{
  Endpoint: "me@example.com", # String. Examples: http | https | email | email | sms | sqs | application | lambda
  Protocol: "email", # String
}]
```

### Subscribing to Dead Letter Queue Alerts

The worker EC2 instances will try to process the S3 objects 3 times, after the 3rd failure. It will move the message over to the Dead Letter Queue. You can also subscribe to the Dead Letter Queue to receive alerts when there are processing errors.

```ruby
# AWS::SNS::Topic DLQ Queue (processing errors)
@dlq_topic_subscription = [{
  Endpoint: "me@example.com", # String. Examples: http | https | email | email | sms | sqs | application | lambda
  Protocol: "email", # String
}]
```

### Bucket Notification Topic

Do not use the AlertsTopic as the S3 Bucket Event Notification Configuration. You need a separate SNS Topic for each region, and it is created with the companion [boltopspro/s3-antivirus-cli](https://github.com/boltopspro-docs/s3-antivirus-cli) tool. Here's an example of the command.

    s3-antivirus bucket enable BUCKET --sqs-arn SQS_ARN

There is also a batch command that can be use to enable s3-antivirus protection for a list of buckets quickly.

    s3-antivirus batch bucket enable FILE.TXT --sqs-arn SQS_ARN

More details: [boltopspro/s3-antivirus-cli](https://github.com/boltopspro-docs/s3-antivirus-cli)

## Automatic s3-antivirus Protection for New Buckets

To automatically enable s3-antivirus protection for new buckets on a go-forward basis use the [s3-antivirus-autoenable](https://github.com/boltopspro-docs/s3-antivirus-autoenable) blueprint.  You will use the SQS Queue ARN created by this template as a variable in that template.

## Single or Multiple Scan Stacks?

The blueprint is designed for flexibility and supports running:

1. one stack to scan all buckets from multiple regions or
2. multiple regional stacks to scan buckets within the same region

There are tradeoffs with each approach and which one you should use depends on your use-case:

* Pro: With #1, you only need to run a single stack and don't have to pay for duplicated stack resources in multiple regions. IE: ec2 instances for scanning.
* Con: With #1, there are bandwidth costs with s3 for inter-region data transfer.

So it depends if the cost of the s3 transfer would be more than the costs of running the EC2 instances.

It is common to have most of your buckets in one region and maybe have a few small buckets in the other regions. The buckets in other regions may not be used much also.  So generally, it will save money and be easier to run and manage one stack. So it is generally recommended to use a Single Scan Stack for simplicity. The design allows flexibility to take either approach, though.

## awslogs configuration

This blueprint leverages the [awslogs](https://github.com/boltopspro-docs/awslogs) configset to set up centralized logging. It is recommended to configure the `@log_group_name` and `@add_instance_id_as` variables with these values.

configs/s3-antivirus/configsets/variables/awslogs.rb:

```ruby
@log_group_name = "s3-antivirus"
@add_instance_id_as = "suffix"
```

This allow you to tail the logs across all instances with the [aws-logs](https://github.com/tongueroo/aws-logs) tool:

    aws-logs tail s3-antivirus --log-stream-name-prefix /var/log/messages --filter-pattern '"s3-antivirus"'

Note, we must surround s3-antivirus with both quotes because of the way bash treates quotes and also because CloudWatch Log search normally interpret the `-` as a reject filter.  You can also just run this less specific pattern:

    aws-logs tail s3-antivirus --log-stream-name-prefix /var/log/messages --filter-pattern antivirus

## Testing Instructions

Refer to [Testing Instructions](/docs/instructions-testing.md).
