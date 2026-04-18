# IAM Access Denied (S3 Upload Failure)

## Context
A data processing application (Python) running on an EC2 instance with an IAM Role is failing to upload logs to a centralized S3 bucket.

## Symptoms
- App logs report: `An error occurred (AccessDenied) when calling the PutObject operation: Access Denied`.
- Status Code: 403.
- Incident frequency: 100% of upload attempts.

## Initial Triage
1. Check instance role identity: `aws sts get-caller-identity`. Result: Correct ARN: `arn:aws:iam::12345678:role/LogCollectorRole`.
2. Verify local permissions via AWS CLI: `aws s3 cp test.log s3://logs-bucket-prod/test.log`. Result: `Access Denied`.
3. Check bucket existence: `aws s3 ls s3://logs-bucket-prod/`. Result: Success (Can list).

## Investigation
1. Examine IAM Role policies:
   ```bash
   aws iam list-role-policies --role-name LogCollectorRole
   aws iam get-role-policy --role-name LogCollectorRole --policy-name S3UploadPolicy
   ```
   Policy allows: `s3:PutObject` on `arn:aws:s3:::logs-bucket-prod`. (Wait, resource is `arn:aws:s3:::logs-bucket-prod`, but `PutObject` requires `/*` on the end).
2. Check S3 Bucket Policy:
   ```bash
   aws s3api get-bucket-policy --bucket logs-bucket-prod
   ```
   Bucket policy is empty (Default).
3. Check for VPC Endpoints or SCPs: No SCP restrictions found for S3.

## Root Cause
The IAM Policy resource ARN `arn:aws:s3:::logs-bucket-prod` only grants permissions at the bucket level. Operations on objects (like `PutObject`) require permissions on the object path level (`arn:aws:s3:::logs-bucket-prod/*`).

## Resolution
1. Correct the IAM Policy to include the object-level ARN:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": ["s3:PutObject", "s3:ListBucket"],
         "Resource": [
           "arn:aws:s3:::logs-bucket-prod",
           "arn:aws:s3:::logs-bucket-prod/*"
         ]
       }
     ]
   }
   ```
2. Re-apply the policy.

## Validation
- Re-run CLI test: `aws s3 cp test.log s3://logs-bucket-prod/test.log` -> `upload: ./test.log to s3://logs-bucket-prod/test.log`.
- Verify in app logs: Uploads are succeeding (HTTP 200).

## Prevention
- Use IAM Access Analyzer during development to validate policy resource targets.
- Implement standard IAM policy templates for common patterns (e.g., S3 read/write) to avoid manual ARN mistakes.
- Add unit tests for IAM policies in Terraform using `terraform plan` and policy validation tools.