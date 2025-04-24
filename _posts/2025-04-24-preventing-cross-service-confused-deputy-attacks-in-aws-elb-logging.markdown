---
layout: post
title:  "Preventing Cross-Service Confused Deputy Attacks in AWS ELB Logging"
date:   2025-04-24 17:00:0 +1000
---

Like many AWS services, the Elastic Load Balancing service allows for the delivery of logs to an S3 bucket. Despite this, each service seems to take a different approach to bucket permissions. Until earlier this year the approach taken by the Cloudfront service was particularly [idiosyncratic](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/standard-logging-legacy-s3.html#AccessLogsBucketAndFileOwnership), requiring the use of bucket ACLs. 

Bucket encryption is also a mixed bag, with some supporting `sse:kms` configuration while others only support `sse:s3`

Despite many of the things that AWS does well, I tend to agree with view that they lack consistency between services.

But back to ELB. The process of setting up cross-account S3 logging is fairly straightforward. This is helpful when you want to aggregate all logs to a centralised log/audit account. When I had to implement this a few weeks ago, the [docs](https://web.archive.org/web/20250319001906/https://docs.aws.amazon.com/elasticloadbalancing/latest/application/enable-access-logging.html#verify-bucket-permissions) provided the following example bucket policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "logdelivery.elasticloadbalancing.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "s3-bucket-arn/*"
    }
  ]
}
```

There is a slight caveat though - only regions created after 2022 support referencing the `logdelivery.elasticloadbalancing...` principal. For most regions, you need to provide access to a specific account id for that region that presumably hosts infrastructure for the ELB service. For Sydney (`ap-southeast-2`) the policy looks like:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::783225319266:root"
      },
      "Action": "s3:PutObject",
      "Resource": "s3-bucket-arn/*"
    }
  ]
}
```


Once the bucket policy is in place, it's a simple matter of turning on logging for the load balancer and supplying the bucket name. 

Despite the relatively simple process there is a flaw with the policies. It's so easy to setup cross-account logging that in fact *any* account can now direct their logs to your S3 bucket - assuming they know (or can guess) the name of the bucket. This is known as a Cross-Service Confused Deputy Attack, which is any situation where an attacker can leverage an AWS service to take actions on their behalf which they do not directly have the permissions to perform.

In the case of ELB access logging:
- A user creates a bucket in account `111122223333` and attaches one of the above bucket policies to allow the ELB service to write logs to the bucket
- An attacker in account `444455556666` has permissions to access the ELB service via their account, but no direct access to the account `111122223333`
- The attacker *confuses* the ELB service (*the deputy*) to write to the bucket in account `111122223333` on their behalf

The AWS docs on [Cross-Service Confused Deputy Attack](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html#cross-service-confused-deputy-prevention) have a helpful diagram of a very similar attack involving delivery of logs to S3 from the Cloudtrail service. 

For an example of a particularly nasty AppSync vulnerability that allowed confused deputy attacks see [this post](https://securitylabs.datadoghq.com/articles/appsync-vulnerability-disclosure/) from Datadog Labs

In comparison, the confused deputy issue with ELB is minor because the documented ELB policy only allows `s3:PutObject`. Additionally, the ELB service always logs under the path `AWSLogs/<source-account-id>/...` or optionally `<logging-prefix>/AWSLogs/<source-account-id>/`. 

There is no clear way that this situation could be leveraged to tamper with a victim account's existing logs unless the attacker were able to identify a vulnerability in the ELB service (e.g. by abusing the optional prefix somehow). Despite this, the idea of my bucket being writable to any AWS account (even in a limited capacity) is uncomfortable. 

The standard mitigation in this situation is to use IAM condition keys, like `aws:SourceAccount` which enforces that the request "originates" from a particular account. Per [AWS docs](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-sourceaccount):

> *This key provides a uniform mechanism for enforcing cross-service confused deputy control across AWS services. However, not all service integrations require the use of this global condition key. See the documentation of the AWS services you use for more information about service-specific mechanisms for mitigating cross-service confused deputy risks.*

I opened a case with AWS Support to clarify whether the `aws:SourceAccount` condition key was supported for ELB logging - because based on my testing it seemed to work as expected. The support engineer was certain that this condition key would *not* work, but was eventually able to replicate the behaviour that I saw. Perhaps the condition key is not *officially* supported by the service?

Support instead suggested updating the targeted `Resource` to include the ELB added prefix e.g. `arn:aws:s3:::s3-bucket-name/AWSLogs/111122223333/*`. If another account then tried to configure the bucket as a logging destination it would fail because the bucket path would not match the bucket policy. 

While I asked them to put in a request to have the docs updated to include the more restrictive policy, I half expected that it might sit in a backlog for some time. So when I came across the [awssecuritychanges.com]() site earlier this week, I was happy to see the change listed:

![Documentation Change](/images/elb-docs-change.png)

Per the updated docs:

> *Ensure your AWS account ID is always included in the resource path of your Amazon S3 bucket ARN. This ensures only Application Load Balancers from the specified AWS account are able to write access logs to the S3 bucket.*

Good Advice!
