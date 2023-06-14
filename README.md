# How to: Setup AWS Access for ATG using Cross-Account Role
## Summary
This article will demonstrate how you can configure AWS access for Aligned Technology Group to access your AWS environment.

## Considerations
If we are analyzing multiple accounts, please ensure that the role you will be creating is created on every account and can be assumed by us.

## Method 1: Using AWS Console
You can set up a cross-account role for ATG so we can assume the role you’ve created from our AWS account.  This way you do not have to create/decommission any credentials for us to access your environment.

### Instructions
1. Sign in to the IAM console at https://console.aws.amazon.com/iam/. You must sign in as an IAM user, assume an IAM role, or sign in as the root user (not recommended) in the member account. The user or role must have permission to create IAM roles and policies.
1. In the IAM console, navigate to Roles and then choose Create Role.
1. For the Trusted entity type choose AWS Account and under An AWS account place a checkmark on Another AWS account.
1. Enter the 12-digit account ID number of the account ATG will provide to you and under Options check Require MFA and click Next.
1. On the Add Permissions page, search for ReadOnlyAccess once found, place a checkmark next to it.

### Optional Step: Create a deny policy
1. You can also create another policy to deny access to your data and certain services so this role isn’t too permissive, click Create Policy which will open another window.
1. Click on the JSON tab in the Create policy window and paste in our example deny policy.

```json
{
      "Version": "2012-10-17",
      "Statement": [
          {
              "Sid": "DenyData",
              "Effect": "Deny",
              "Action": [
                  "cloudformation:GetTemplate",
                  "dynamodb:GetItem",
                  "dynamodb:BatchGetItem",
                  "dynamodb:Query",
                  "dynamodb:Scan",
                  "ec2:GetConsoleOutput",
                  "ec2:GetConsoleScreenshot",
                  "ecr:BatchGetImage",
                  "ecr:GetAuthorizationToken",
                  "ecr:GetDownloadUrlForLayer",
                  "kinesis:Get*",
                  "lambda:GetFunction",
                  "logs:GetLogEvents",
                  "s3:GetObject",
                  "sdb:Select*",
                  "sqs:ReceiveMessage"
              ],
              "Resource": "*"
          }
      ]
  }
```

3. Click Next: Tags and create tags of your choosing (Optional) and click Next: Review.
4. Give a name to the policy, we’d recommend ATGDenyAccess and click Create Policy.
5. Switch back to the browser tab where we were adding permissions from Step 5 and click the Clear filters button and then refresh icon.  
6. Type in the name of the policy we’ve created in Step 9 to search for it, place a checkmark next to the policy (Ensure that ReadOnlyAccess policy is still checked as well) and click Next.
7. Ensure that both policies are present in the Review page, if not, please go back and ensure both policies are selected.

### Create the Role
1. On the Review page, specify a role name, an optional description and any tags you may want to add to the role. We recommend that you use ATGReadOnlyAccessRole as the role name. To commit your changes, choose Create role.
1. Your new role appears on the list of available roles. Choose the new role's name to view its details, paying special note to the link URL that is provided. Give this URL to ATG so we can access the role.

## Method 2: Using CloudFormation
You may also use CloudFormation to deploy this role.  Here is a template you may use to create this role.

```json
{
    "AWSTemplateFormatVersion": "2010-09-09",
    "Resources": {
        "ATGReadOnlyAccessRole": {
            "Type": "AWS::IAM::Role",
            "Properties": {
                "RoleName": "ATGReadOnlyAccessRole",
                "AssumeRolePolicyDocument": {
                    "Version": "2012-10-17",
                    "Statement": [
                        {
                            "Effect": "Allow",
                            "Principal": {
                                "AWS": "395963974293"
                            },
                            "Action": [
                                "sts:AssumeRole"
                            ],
                            "Condition": {
                                "Bool": {
                                    "aws:MultiFactorAuthPresent": true
                                }
                            }
                        }
                    ]
                },
                "ManagedPolicyArns": [
                    "arn:aws:iam::aws:policy/ReadOnlyAccess"
                ],
             "Path": "/"
            }
        },
        "RolePolicies": {
            "Type": "AWS::IAM::Policy",
            "Properties": {
                "PolicyName": "ATGDenyAccess",
                "PolicyDocument": {
                    "Version": "2012-10-17",
                    "Statement": [
                        {
                            "Effect": "Deny",
                            "Action": [
                                "cloudformation:GetTemplate",
                                "dynamodb:GetItem",
                                "dynamodb:BatchGetItem",
                                "dynamodb:Query",
                                "dynamodb:Scan",
                                "ec2:GetConsoleOutput",
                                "ec2:GetConsoleScreenshot",
                                "ecr:BatchGetImage",
                                "ecr:GetAuthorizationToken",
                                "ecr:GetDownloadUrlForLayer",
                                "kinesis:Get*",
                                "lambda:GetFunction",
                                "logs:GetLogEvents",
                                "s3:GetObject",
                                "sdb:Select*",
                                "sqs:ReceiveMessage"
                            ],
                            "Resource": "*"
                        }
                    ]
                },
                "Roles": [
                    {
                        "Ref": "ATGReadOnlyAccessRole"
                    }
                ]
            }
        }
    }
}
```

## Method 3: Using Terraform
You may use the example Terraform code to deploy this role.

```hcl
resource "aws_iam_policy" "atg_deny_policy" {
  name = "ATGDenyAccess"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "cloudformation:GetTemplate",
          "dynamodb:GetItem",
          "dynamodb:BatchGetItem",
          "dynamodb:Query",
          "dynamodb:Scan",
          "ec2:GetConsoleOutput",
          "ec2:GetConsoleScreenshot",
          "ecr:BatchGetImage",
          "ecr:GetAuthorizationToken",
          "ecr:GetDownloadUrlForLayer",
          "kinesis:Get*",
          "lambda:GetFunction",
          "logs:GetLogEvents",
          "s3:GetObject",
          "sdb:Select*",
          "sqs:ReceiveMessage"
        ]
        Effect   = "Deny"
        Resource = "*"
      },
    ]
  })
}

resource "aws_iam_role" "atg_readonly_access_role" {
  name = "ATGReadOnlyAccessRole"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        "Effect" : "Allow",
        "Principal" : {
          "AWS" : "395963974293"
        },
        "Action" : [
          "sts:AssumeRole"
        ],
        "Condition" : {
          "Bool" : {
            "aws:MultiFactorAuthPresent" : true
          }
        }
      },
    ]
  })
  managed_policy_arns = [
    "arn:aws:iam::aws:policy/ReadOnlyAccess",
    aws_iam_policy.atg_deny_policy.arn
  ]
}
```
