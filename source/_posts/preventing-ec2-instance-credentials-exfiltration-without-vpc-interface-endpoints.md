---
date: "2023-04-10 12:00"
title: "Disallowing EC2 instance credentials being (ab)used elsewhere (without paying for VPC Interface Endpoints)"
---

The IAM conditions `aws:ec2InstanceSourceVPC` and `aws:EC2InstanceSourcePrivateIPv4` allow to restrict the usage of EC2 instance credentials to the EC2 instance itself.

This is a great security feature, for example it would have prevented breaches like the [Capital One breach](https://krebsonsecurity.com/2019/08/what-we-can-learn-from-the-capital-one-hack/). 

But the current solution by AWS comes with a cost: You need to pay for VPC interface endpoints, if you want to use this feature in combination with `${aws:SourceVpc}` and `${aws:VpcSourceIp}`, as promoted by the [AWS blog](https://aws.amazon.com/blogs/security/how-to-use-policies-to-restrict-where-ec2-instance-credentials-can-be-used-from/) and [AWS documentation samples](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-ec2instancesourcevpc).

VPC interface endpoints **cost per hour per AWS service per AZ per region (per VPC)**. That means if you want to connect the EC2 with e.g. EC2, SQS, and SNS, and you have 3 AZs per region, you need to pay for 9 VPC interface endpoints. A VPC interface endpoint costs $7.20 per month and AZ plus $0.01 per GB. For 27 VPC interface endpoints, this means $193.20 per month.

This article shows how to prevent EC2 instance credentials from being used outside its VPC without paying for VPC Interface Endpoints.

While `${aws:SourceVpc}` and `${aws:VpcSourceIp}` only work when used in combination with VPC interface endpoints, the conditions `aws:ec2InstanceSourceVPC` and `aws:EC2InstanceSourcePrivateIPv4` work without VPC interface endpoints.

So, "hardcoding" the VPC ID in the IAM condition will work without VPC interface endpoints. Here is an [adapted](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-ec2instancesourcevpc) example policy that denies all actions if the VPC ID does not match:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireSameVPC",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:ec2InstanceSourceVPC": "vpc-12345678"
        },
        "Null": {
          "ec2:SourceInstanceARN": "false"
        },
        "BoolIfExists": {
          "aws:ViaAWSService": "false"
        }
      }
    }
  ]
}
```

Note: Additional explanations from the AWS docs regarding `ec2:SourceInstanceARN` and `aws:ViaAWSService` still apply. 

Now, hardcoding the VPC ID is not a good idea, because it will break if you change the VPC ID. So, we need to find a way to dynamically get the VPC ID.

This can be usually achived by using some sort of infrastructure as code tool, e.g. CDK or Terraform.

Using a role with this policy attached from an EC2 instance in VPC `vpc-12345678` will work:

```

```

Now, let's steal the credentials from the EC2 instance and use them from outside the VPC:

```
[root@ip-172-31-95-199 ~]# curl http://169.254.169.254/latest/meta-data/iam/security-credentials/ssmmanaged2
{
  "Code" : "Success",
  "LastUpdated" : "2023-04-10T07:55:42Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASIASQMR66HSQVGTS4N6",
  "SecretAccessKey" : "LlyFWriFT9HZ1lzlve8JQlCymwyjcD1kDU5MI9ft",
  "Token" : "...",
  "Expiration" : "2023-04-10T14:31:32Z"
}
```

```
{
    "UserId": "AROASQMR66HS3M2DTJIAS:i-0d7dd677a895625c2",
    "Account": "172641677797",
    "Arn": "arn:aws:sts::172641677797:assumed-role/ssmmanaged2/i-0d7dd677a895625c2"
}
```

`````

## Limitations

- The examples don't work with Service Control Policies (SCP) because the VPC IDs are probably different per AWS account. 
- `aws:EC2InstanceSourcePrivateIPv4` could also be hardcoded, but this would break if the EC2 instance gets a new private IP address which is very common in cloud environments.