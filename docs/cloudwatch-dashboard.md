<!-- note marker start -->
**NOTE**: This repo contains only the documentation for the private BoltsOps Pro repo code.
Original file: https://github.com/boltopspro/s3-antivirus/blob/master/docs/cloudwatch-dashboard.md
The docs are publish so they are available for interested customers.
For access to the source code, you must be a paying BoltOps Pro subscriber.
If are interested, you can contact us at contact@boltops.com or https://www.boltops.com

<!-- note marker end -->

# CloudWatch Dashboard

[![BoltOps Badge](https://img.boltops.com/boltops/badges/boltops-badge.png)](https://www.boltops.com)

This blueprint also provides an auto-updating CloudWatch Dashboard to monitor the S3 Antivirus processing. The Dashboard automatically updates:

* When S3 buckets are created and deleted.
* When AutoScaling adds and removes the worker instances.
* Also, updates every hour periodically to discovery S3 buckets across regions also. This is done because CloudWatch events are regional.

Custom metrics:

* Displays custom metrics like: ScannedCount, CleanCount, InfectedCount, OversizedCount, BytesScanned
* Metrics are provided on a by bucket or by region basis.
* The by-bucket metrics shows you which buckets are most scanned.
* The by-region metrics shows you which regions are most scanned. This is a useful datapoint to determine if it's worth it to run S3 Antivirus stacks in multiple regions.

SQS Metrics:

* The useful metrics for both SQS queue and the Dead Letter Queue metrics are surfaced up to the S3Antivirus dashboard for quick summarized visibility.

## Screenshot

![](https://img.boltops.com/boltopspro/blueprints/s3-antivirus/s3-antivirus-cloudwatch-dashboard.png)

## Configuration

The auto-updating function also runs on a periodic basis, every 1hr by default. This is necessary because s3 buckets created in regions outside of this stack will not receive a CloudWatch event. CloudWatch events are regional. You can adjust this frequency the job with `@dashboard_schedule_expression`.

configs/s3-antivirus/variables/development.rb:

```ruby
@dashboard_schedule_expression = "rate(30 minutes)"
```

### Lambda Function in VPC

To configure the Lambda function with VpcConfig set `@dashboard_subnet_ids` and either `@dashboard_vpc_id` or `@dashboard_security_group_ids`.

* When the `@dashboard_vpc_id` is set, the template creates a managed security group for you and the Lambda function is configured to use that security group.
* When `@dashboard_security_group_ids` is set, the Lambda function will use those existing security groups.
* The subnet must be a private subnet with configured with a NAT.

Here's an example of the managed security group.

configs/s3-antivirus/variables/development.rb:

```ruby
@subnet_ids = ["subnet-111"]
@vpc_id = "vpc-111"
```

For Lambda VPC to work, the subnet must be a private subnet configured with a NAT.

Note, Lambda functions configured with VPCs may take much longer to deploy, typically 30-45 minutes. This is because Lambda creates and attaches an ENI to the Lambda function to make the VPC feature possible. If the function is deleted or updated, requiring replacement, the ENI takes 30-45m to be removed. Because of this, it is recommended to write code for your Lambda function code without the VpcConfig first. Get it working and then add VpcConfig at the end.

### X-Ray Tracing

Lambda X-Ray tracing is set to `Active` by default. You can disable this by setting `@dashboard_tracing_config_mode = false`. Example:

configs/s3-antivirus/variables/development.rb:

```ruby
@tracing_config_mode = false
```

You can also change the mode with the same `@dashboard_tracing_config_mode` variable:

```ruby
@tracing_config_mode = "Active" # or "PassThrough"
```
