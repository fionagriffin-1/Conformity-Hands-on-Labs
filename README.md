# Cloud One: Conformity Hands on Labs
## Overview

This HoL demonstrates the two ways in which Conformity can be used:

1. Securing existing resources
2. Securing proposed resources by scanning CloudFormation templates in a CI/CD pipeline

## Set Up

**Note:** `<BaseBucketName>` appears multiple times in the snippets below. Replace it with a unique name before proceeding.

### AWS

1. Spin up the environment:

```
aws cloudformation create-stack \
--stack-name conformity-labs \
--template-body file://infra.yaml \
--parameters \
ParameterKey=AwsUserArn,ParameterValue="$(aws sts get-caller-identity --query Arn --output text)" \
ParameterKey=BaseBucketName,ParameterValue=<BaseBucketName> \
--capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```

2. The CloudFormation scanner uses the Conformity API. Log into Conformity and generate a Conformity API key. Now go to AWS Secrets Manager.

3. Click _"Other type of secrets"_ then _"Plaintext"_. Replace the predefined text with the Conformity API key. Click _"Next"_. 

4. Set the _"Secret name"_ field to `Conformity-template-scanner-api-key`. Continue clicking _"Next"_ and then finally _"Store"_.

### Conformity

1. Log into Conformity.
   
2. Follow the AWS deployment instructions.

3. Complete one or both of the below labs.


## Labs
### Existing resources

Intentionally insecure resources were spun up as part of the setup process. Use Conformity to identify and secure these resources. 

### CI/CD pipeline scanning

Conformity's pipeline scanner enables organisations to "shift secure left". This means that engineers are able to secure their cloud infrastructure **before** it even exists.

In this lab we'll see how Conformity prevents us from deploying an insecure resource. 

1. Navigate to Cloud9. Click "Open IDE". 

2. Navigate to `~/environment/codecommit/Conformity-S3-Bucket-IaC/`. Create a file named `cfn.yaml` in this directory.

3. Paste the below content into the file, then save it:

```
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: <BaseBucketName>-cfn-bucket
```

4. Run the following commands:

```
cd ~/environment/codecommit/Conformity-S3-Bucket-IaC/
git config --global user.name "Conformity"
git config --global user.email Conformity@example.com
git add cfn.yaml
git commit -m 'New lab bucket'
git push
```

5. Navigate to CodePipeline. After analysing the template, Conformity will fail the pipeline. This is due to the configuration being insecure.

6. Replace the contents of `cfn.yaml` with the below, then save it:

```
Resources:
  CfnBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: <BaseBucketName>-cfn-bucket
      LoggingConfiguration:
        DestinationBucketName: <BaseBucketName>-artifact-bucket
        LogFilePrefix: demo
      AccelerateConfiguration:
        AccelerationStatus: Enabled
      PublicAccessBlockConfiguration: 
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
        - ServerSideEncryptionByDefault:
            SSEAlgorithm: AES256
      Tags:
      - Key: Name
        Value: Conformity CFN Bucket
  
  SecureBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref CfnBucket
      PolicyDocument:
        Statement:
        - Effect: Deny
          Principal: "*"
          Action: "*"
          Resource: !Sub arn:aws:s3:::${CfnBucket}/*
          Condition:
            Bool:
              aws:SecureTransport: 'false'
```

7. Run the following commands:

```
git add cfn.yaml
git commit -m 'New lab bucket - now secured'
git push
```

8. Navigate to CodePipeline. As there are no security concerns this time around, Conformity will enable the pipeline to proceed. 
   
9. Next, navigate to CloudFormation and/or S3. You will see that a new, secured S3 bucket named `<BaseBucketName>-cfn-bucket` has been created.