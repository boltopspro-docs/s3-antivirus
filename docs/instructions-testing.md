<!-- note marker start -->
**NOTE**: This repo contains only the documentation for the private BoltsOps Pro repo code.
Original file: https://github.com/boltopspro/s3-antivirus/blob/master/docs/instructions-testing.md
The docs are publish so they are available for interested customers.
For access to the source code, you must be a paying BoltOps Pro subscriber.
If are interested, you can contact us at contact@boltops.com or https://www.boltops.com

<!-- note marker end -->

# S3 AntiVirus Testing Instructions

After completing the [S3 AntiVirus Overview](/docs/instructions-overview.md) setup instructions. You can test by uploading an object an s3 bucket that has S3 AntiVirus protection enabled.

## Example Test Commands

Here are some commands to help.

    aws s3 cp a.txt s3://YOUR_BUCKET/
    aws s3api get-object-tagging --bucket YOUR_BUCKET --key a.txt

Replace the `YOUR_BUCKET` placeholder with your actual bucket.

Commands with output:

    $ aws s3 cp a.txt s3://YOUR_BUCKET/
    upload: ./a.txt to s3://YOUR_BUCKET/a.txt
    $ aws s3api get-object-tagging --bucket YOUR_BUCKET --key a.txt
    {
        "TagSet": [
            {
                "Key": "scan-at",
                "Value": "2020-04-13T19:18:08Z"
            },
            {
                "Key": "scan-status",
                "Value": "clean"
            }
        ]
    }
    $

## Tailing CloudWatch Logs

Note, if you are tailing the Cloudwatch logs, it may take some time for it to show up - sometimes even 10s of seconds. This because CloudWatch logs events come in asynchronously.  So the object is tagged on s3 before the events show up on CloudWatch logs.

Here's the [aws-logs tail](https://github.com/tongueroo/aws-logs) command

    aws-logs tail s3-antivirus --log-stream-name-prefix /var/log/messages --filter-pattern '"s3-antivirus"'

Note, you need to have configured the configset variables:

configs/s3-antivirus/configsets/variables/awslogs.rb:

```ruby
@log_group_name = "s3-antivirus"
@add_instance_id_as = "suffix"
```
