## AWS CloudWatch to S3

Logging infrastructure for exporting all CloudWatch logs from multiple accounts to a single S3 bucket.

Available on AWS Serverless Application Repository for easy deployment:

* [CloudWatch2S3](https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:859319237877:applications~CloudWatch2S3)
* [CloudWatch2S3-additional-account](https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:859319237877:applications~CloudWatch2S3-additional-account)

### Deprecation

Most of this code is no longer required now that Kinesis Firehose supports decompression and CloudWatch log processing.

https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-properties-kinesisfirehose-deliverystream-processor.html

### Overview

![Architecture diagram](https://github.com/CloudSnorkel/CloudWatch2S3/raw/master/architecture.svg?sanitize=true)

This project supplies a CloudFormation template that setups Kinesis stream that takes log records from CloudWatch and
writes them to a specific S3 bucket as they are arrive. Log records can be retrieved from multiple AWS accounts using
the second CloudFormation template.

Log records are batched together across log groups and partitioned into folders based on time of ingestion. Log format
can be configured to either be raw log lines or compressed CloudWatch JSON. The raw log format is:

    LOG_GROUP:LOG_STREAM\tTIMESTAMP\tRAW_LOG_LINE

For example:

    /aws/lambda/SomeLambdaFunction:2019/02/05/[$LATEST]b346f603d7bb4b6aa77b53bc4050bc37 1549428326  INFO hello world

Subscription of CloudWatch log groups is done in two ways. If CloudTrail is enabled, every new log group will immediacy
be subscribed. In addition, every hour a subscription Lambda is executed to look for new log groups and subscribe them.
Finally, the same subscription Lambda is executed during deployment of the CloudFormation stack so all log groups
matching the configured prefix will be subscribed immediately on deployment.

If CloudTrail is not enabled, it may take up to an hour for new log groups to be subscribed. This time can be configured
in the CloudFormation stack using the `SubscribeSchedule` parameter. In CloudFormation UI it may be named _Look for New
Logs Schedule_. 

### Deploy

If you have just one AWS account, simply deploy `CloudWatch2S3.template` in CloudFormation.

If you have multiple AWS accounts, choose a central account where all logs will be stored in S3 and deploy
`CloudWatch2S3.template` in CloudFormation. Once done, go to the outputs tab and copy the value of `LogDestination`.
Then go to the other accounts and deploy `CloudWatch2S3-additional-account.template` in CloudFormation. You will need to
supply the value you copied as the `LogDestination` parameter.

#### Parameters

There are a lot of parameters to play with, but the defaults should be good enough for most. If you have a lot of log
records coming in (more than 1000/s or 1MB/s), you might want to increase Kinesis shard count.

### Known Limitations

Cross region export is not supported by CloudWatch Logs. If you need to gather logs from multiple regions, create the CloudFormation stack in each required region. You can use CloudFormation Stack Sets to deploy to all regions at once.

Single CloudWatch records can't be over 6MB when using anything else but raw log format. Kinesis uses Lambda to convert data and Lambda output is limited to 6MB. Note that data comes in compressed from CloudWatch but has to come out decompressed from Lambda. So the decompressed record can't be over 6MB. You will see record failures in CloudWatch metrics for the Kinesis stream for this and errors in the log for the processor Lambda function.

### Troubleshooting

* Make sure the right CloudWatch log groups are subscribed
* Look for errors in CloudWatch log group `/aws/kinesisfirehose/<STACK NAME>-DeliveryStream`
* Look for errors in CloudWatch log group `/aws/lambda/<STACK NAME>-LogProcessor-<RANDOM>`
* Make sure Kinesis, Firehose and S3 have access to your KMS key when using encryption
* Increase Kinesis shard count with `ShardCount` (_Kinesis Shard Count_) CloudFormation parameter

### Origin

This project is based on Amazon's [Stream Amazon CloudWatch Logs to a Centralized Account for Audit and Analysis](https://aws.amazon.com/blogs/architecture/stream-amazon-cloudwatch-logs-to-a-centralized-account-for-audit-and-analysis/) but adds:

* One step installation
* Zero scripts
* Works out of the box
* Easier configuration without editing files
* No hard dependency on CloudTrail
* Optional unpacking of CloudWatch JSON format
