# Diamond 2 XML Upload — AWS Bucket Authentication Approach

## Overview

Diamond 2 will use AWS's recommended Cross-Account IAM Role Assumption method for secure file uploads. This is combined with a dedicated S3 bucket for each broadcaster. This document sets out how broadcasters will gain access to their Diamond 2 XML upload bucket. Diamond 2 will run on the platform provided by The Everyone Project — abbreviated as TEP below.

## What You Need to Provide TEP

- **Your AWS Account ID** (12-digit number)

## What TEP Will Provide You

When TEP sets up your access, you'll receive the following details:

| Item | Description | Example |
| --- | --- | --- |
| **Role ARN** | The IAM role you'll assume | `arn:aws:iam::060011828759:role/tep-xml-ingestion-{broadcaster}-upload` |
| **External ID** | Security token for role assumption | `tep-xml-upload-{broadcaster}` |
| **Bucket Name** | Your dedicated S3 bucket | `tep-xml-ingestion-staging-{broadcaster}` |
| **Upload Path** | Where to place files | `incoming/` |
| **Region** | AWS region | `eu-west-2` (London) |
| **SNS Topic ARN** | For processing notifications (optional) | `arn:aws:sns:eu-west-2:060011828759:tep-xml-ingestion-staging-{broadcaster}-notifications` |

**Note:** The External ID is a required security feature to prevent the ["confused deputy problem"](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html). You must provide this External ID when assuming the role.

## How to Upload Files

### For Production Systems (Recommended)

If your upload process runs on an AWS service (EC2, ECS, Lambda, etc.), you can attach an IAM role to that service which has permission to assume TEP's role. This means your application can assume the role automatically without needing to manually export credentials — the AWS SDK handles it all for you.

### For Testing/Development

For testing or one-off uploads, you can manually assume the role from the command line:

#### Step 1: Assume the Role

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::060011828759:role/tep-xml-ingestion-{broadcaster}-upload \
  --role-session-name xml-upload-session \
  --external-id tep-xml-upload-{broadcaster}
```

This returns a JSON response containing temporary credentials.

#### Step 2: Export the Temporary Credentials

Parse the JSON response and export the credentials using `jq`:

```bash
# Capture the credentials in a variable
CREDENTIALS=$(aws sts assume-role \
  --role-arn arn:aws:iam::060011828759:role/tep-xml-ingestion-{broadcaster}-upload \
  --role-session-name xml-upload-session \
  --external-id tep-xml-upload-{broadcaster})

# Extract and export the credentials
export AWS_ACCESS_KEY_ID=$(echo $CREDENTIALS | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDENTIALS | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo $CREDENTIALS | jq -r '.Credentials.SessionToken')
```

#### Step 3: Upload Your Files

```bash
aws s3 cp --sse AES256 your-file.xml s3://tep-xml-ingestion-staging-{broadcaster}/incoming/
```

You can now use any AWS S3 commands with these temporary credentials.

## Setting Up Notifications (Optional)

TEP will send notifications via SNS when your uploaded files have been processed and moved to either the `complete/` or `errors/` directory. You can subscribe to these notifications in several ways:

### Option 1: Email Notifications (Simple)

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:eu-west-2:060011828759:tep-xml-ingestion-staging-{broadcaster}-notifications \
  --protocol email \
  --notification-endpoint your-email@example.com
```

AWS will send you a confirmation email with a link you must click. To unsubscribe later, click the unsubscribe link in any notification email.

### Option 2: Email with JSON Payload

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:eu-west-2:060011828759:tep-xml-ingestion-staging-{broadcaster}-notifications \
  --protocol email-json \
  --notification-endpoint your-email@example.com
```

### Option 3: SQS Queue

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:eu-west-2:060011828759:tep-xml-ingestion-staging-{broadcaster}-notifications \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:eu-west-2:{YOUR_ACCOUNT_ID}:{YOUR_QUEUE_NAME}
```

### Option 4: Lambda Function

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:eu-west-2:060011828759:tep-xml-ingestion-staging-{broadcaster}-notifications \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:eu-west-2:{YOUR_ACCOUNT_ID}:function:{YOUR_FUNCTION_NAME}
```

### Note on Batch Upload

Broadcasters can opt to batch multiple projects / batches / episodes into a single XML file, but can equally well opt to split each individual project, or even episode, into its own XML file. The ingestion process will handle either, however notifications will be issued per file, not per project or per episode. For this reason, integrators may choose to opt for more granular XML files (e.g. one per project) to facilitate logging, debugging and interpreting SNS notifications.

## Per-directory Access Permissions

The role will be given the following per-directory permissions:

### In Production System

| Directory | Permissions |
| --- | --- |
| `/incoming/` | Read, Write, Delete |
| `/complete/` | Read only (the files will automatically be deleted no sooner than 7 days after ingestion) |
| `/errors/` | Read only (the files will automatically be deleted no sooner than 7 days after ingestion failure) |

### In Staging System

| Directory | Permissions |
| --- | --- |
| `/incoming/` | Read, Write, Delete |
| `/complete/` | Read, Write, Delete |
| `/errors/` | Read, Write, Delete |

The additional permissions for the complete and errors directories are intended to facilitate testing of SNS integration i.e. you can create your own files in `/complete/` or `/errors/` and check that this triggers the expected workflow.

## Key Points

### Security Features

- **No long-term credentials** — temporary credentials expire automatically
- **External ID** prevents the [confused deputy problem](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html)
- **Your AWS account** maintains full control over which users/services can assume the role
- **Dedicated bucket per broadcaster** — complete isolation from other broadcasters

### Important Limitations

- **Console access NOT supported** — the External ID requirement means you cannot assume the role through the AWS web console
- **Must use CLI, SDK, or tools** that support the STS AssumeRole flow
- **Credentials expire** — you'll need to re-assume the role periodically (typically after 1 hour)
