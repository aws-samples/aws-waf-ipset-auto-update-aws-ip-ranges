## Automatically update WAF IP sets with AWS IP Ranges

This project creates two regional WAFv2 IP sets and automatically updates them with AWS service's IP ranges from the [ip-ranges.json](https://docs.aws.amazon.com/general/latest/gr/aws-ip-ranges.html) file. The ranges are configurable as well as the regions for EC2 ranges. Use cases include allowing CloudFront requests, Route53 health checker and EC2 IP range (which includes AWS Lambda and CloudWatch Synthetics). The IP sets are created in the region where the CloudFormation stack is created and the IP addresses in the IP sets are completely replaced on every update.

## Overview

The CloudFormation template `cloudformation/template.yml` creates a stack with the following resources:

1. Two WAFv2 IP sets. One for IPv4 addresses and one for IPv6 addresses.
1. AWS Lambda function with customizable environment variables. The function's code is in `lambda/update_wafv2_ipset.py` and is written in Python.
1. Lambda function's execution role.
1. SNS subscription and Lambda invocation permissions for the `arn:aws:sns:us-east-1:806199016981:AmazonIpSpaceChanged` SNS topic.

```
                          +-----------------+         +----------------+
                          | Lambda          |         |                |
                          | Execution Role  |    +--->+ WAFv2 IPv4 Set |
                          +--------+--------+    |    |                |
                                   |             |    +----------------+
                                   |             |
+--------------------+    +--------+--------+    |
|SNS Topic           +--->+ Lambda function +----+
|AmazonIpSpaceChanged|    +--------+--------+    |
+--------------------+             |             |    +----------------+
                                   |             |    |                |
                                   v             +--->+ WAFv2 IPv6 Set |
                          +--------+--------+         |                |
                          | CloudWatch Logs |         +----------------+
                          +-----------------+
```

## Setup

These are the overall steps to deploy:

1. Create an S3 bucket to store the code. The bucket must be in the same region where the stack will be created.
1. Package the Lambda code into a `.zip` file.
1. Upload the packaged code to S3.
1. Create the CloudFormation stack.
1. Trigger a test Lambda invocation.
1. Reference the IP sets in a web ACL rule or rule group
1. Clean-up

To simplify setup and deployment, assign the values to the following variables. Replace the values according to your deployment options.

```bash
REGION=<region>
CFN_STACK_NAME=<Stack name>
BUCKET_NAME=<bucket name>
OBJECT_NAME=auto-update_wafv2_ipset.zip
```

### 1. Create an S3 bucket

Skip this step if you already have an S3 bucket to store the code.

To create the S3 bucket, run the following commands, depending on the region:

For `us-east-1`:

```bash
aws s3api create-bucket --bucket $BUCKET_NAME \
    --region $REGION
```

For any other region outside of `us-east-1`:

```bash
aws s3api create-bucket \
    --bucket $BUCKET_NAME \
    --region $REGION \
    --create-bucket-configuration LocationConstraint=$REGION
```

### 2. Package the Lambda code

Run the following commands from the **repository root**:

```bash
cp lambda/update_wafv2_ipset.py .
zip $OBJECT_NAME update_wafv2_ipset.py
```

### 3. Upload the packaged code to S3

```bash
aws s3 cp $OBJECT_NAME s3://$BUCKET_NAME
```

### 4. Create the CloudFormation stack

The CloudFormation template has the following input parameters.

* `LambdaCodeS3Bucket`: The S3 bucket where the Lambda function's packaged code is stored.
* `LambdaCodeS3Object`: The S3 object name of Lambda function's packaged code. This must be a `.zip` file.
* `IPV4SetNameSuffix`: Enter the name for the Wafv2 IPv6 set. Default is `IPv4Set`.
* `IPV6SetNameSuffix`: Enter the name for the Wafv2 IPv6 set. Default is `IPv6Set`.
* `SERVICES`: Enter the name of the AWS services to add, separated by commas and as explained in <https://docs.aws.amazon.com/general/latest/gr/aws-ip-ranges.html>. The default is `ROUTE53_HEALTHCHECKS,CLOUDFRONT`.
* `EC2REGIONS`: For the "EC2" service only, specify the AWS regions to add, separated by commas. Use 'all' to add all AWS regions. Default is `all`.

Use the command below to create the stack with the default values. You can update the parameter values as required.

```bash
aws cloudformation create-stack \
  --stack-name $CFN_STACK_NAME \
  --template-body file://cloudformation/template.yml \
  --region $REGION \
  --parameters ParameterKey=LambdaCodeS3Bucket,ParameterValue=$BUCKET_NAME \
  ParameterKey=LambdaCodeS3Object,ParameterValue=$OBJECT_NAME \
  --capabilities CAPABILITY_IAM
```

Run the following command and wait to check if the CloudFormation stack is created successfully. The command exits with no error or output when the stack is successfully created.

```bash
aws cloudformation wait stack-create-complete \
    --stack-name $CFN_STACK_NAME \
    --region $REGION
```

If the stack creation fails, troubleshoot by reviewing the stack events. The typical failure reasons are insufficient IAM permissions and the code's S3 bucket in a different AWS region.

### 5a. Trigger a test Lambda invocation with the AWS CLI

After the stack is created, the WAF IP sets are not updated until a new SNS message is received. To test the function and update the IP sets with the current IP ranges for the first time, do a test invocation with the AWS CLI command below:

```bash
aws lambda invoke \
  --function-name $CFN_STACK_NAME-UpdateWAFIPSets \
  --region $REGION \
  --payload file://lambda/test_event.json lambda_return.json
```

After successful invocation, you should receive the response below with no errors.

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

The content of the `lambda_return.json` will list all IPv4 and IPv6 prefixes downloaded and updated by the Lambda function.

### 5b. Trigger a test Lambda invocation with the AWS Console

Alternatively, you can invoke the test event in the AWS Lambda console with sample event below. This event uses a `test-hash` md5 string that the function parses as a test event.

```json
{
  "Records": [
    {
      "EventVersion": "1.0",
      "EventSubscriptionArn": "arn:aws:sns:EXAMPLE",
      "EventSource": "aws:sns",
      "Sns": {
        "SignatureVersion": "1",
        "Timestamp": "1970-01-01T00:00:00.000Z",
        "Signature": "EXAMPLE",
        "SigningCertUrl": "EXAMPLE",
        "MessageId": "12345678-1234-1234-1234-123456789012",
        "Message": "{\"create-time\": \"yyyy-mm-ddThh:mm:ss+00:00\", \"synctoken\": \"0123456789\", \"md5\": \"test-hash\", \"url\": \"https://ip-ranges.amazonaws.com/ip-ranges.json\"}",
        "Type": "Notification",
        "UnsubscribeUrl": "EXAMPLE",
        "TopicArn": "arn:aws:sns:EXAMPLE",
        "Subject": "TestInvoke"
      }
    }
  ]
}
```

### 6. Reference the IP sets in a web ACL rule or rule group

See [Using an IP set in a rule group or Web ACL](https://docs.aws.amazon.com/waf/latest/developerguide/waf-ip-set-using.html).

### 7. Clean-up

Remove the temporary files.

```bash
rm update_wafv2_ipset.py
rm $OBJECT_NAME
rm lambda_return.json
```

## Lambda function customization

After the stack is created, you can customize the Lambda function's execution by editing the function's [environment variables](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html).

* `SERVICES`: **Optional**. Comma separated values for service's to get the ranges for. Value is set to `ROUTE53_HEALTHCHECKS,CLOUDFRONT` if it is not explicitly set.
* `EC2_REGIONS`. **Optional**. Comma separated values for EC2 regions to be added to the IP set.
* `INFO_LOGGING`: **Optional**. Set it to `true` for more detailed logging of the AWS Lambda Python script.
* `IPV4_SET_NAME`: The WAFv2 IPv4 set name.
* `IPV4_SET_ID`: The ID of the WAFv2 IPV4 set to update.
* `IPV6_SET_NAME`: The WAFv2 IPv4 set name.
* `IPV6_SET_ID`: The ID of the WAFv2 IPV6 set to update.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
