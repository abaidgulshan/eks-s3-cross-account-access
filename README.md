# EKS S3 Cross Account Access

This repository contains resources and documentation for enabling cross-account access from an Amazon Elastic Kubernetes Service (EKS) cluster to an Amazon S3 bucket in a different AWS account.

![image](https://github.com/abaidgulshan/eks-s3-cross-account-access/assets/7329596/2a21aa05-3b1b-48ab-b04c-18f46e42ec19)

## Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Instructions](#instructions)
- [Contributing](#contributing)
- [License](#license)

## Introduction

When you have an EKS cluster running in one AWS account and an S3 bucket residing in another AWS account, you may need to grant the EKS cluster access to the S3 bucket. This repository provides guidance and resources to set up cross-account access from the EKS cluster to the S3 bucket securely.

## Prerequisites

Before proceeding, ensure you have the following:

- An Amazon EKS cluster set up in one AWS account.
- An Amazon S3 bucket set up in a different AWS account.
- AWS IAM permissions to create IAM roles and policies in both AWS accounts.
- Kubernetes cluster access and the necessary permissions to create service accounts and roles.

## Instructions

To enable cross-account access from your EKS cluster to the S3 bucket, follow these steps:

1.    **Fetch the CI account cluster’s OIDC issuer URL**
  * `aws eks describe-cluster —name development-cluster --query "cluster.identity.oidc.issuer" --output text`
2.    **Create an OIDC provider for the cluster in the CI account**
    Navigate to the IAM console in the CI account, choose Identity Providers, and then select Create provider. Select OpenID Connect for provider type and paste the OIDC issuer URL for your cluster for provider URL. Enter

![image](https://github.com/abaidgulshan/eks-s3-cross-account-access/assets/7329596/8913afe9-38c8-474d-b01f-ffa6120ab42f)
![image](https://github.com/abaidgulshan/eks-s3-cross-account-access/assets/7329596/0c687cc5-7733-4857-82f3-1ac0290bbb8d)


3. **Configuring the CI account – IAM role and policy permissions**
   Create an IAM role in the CI account, ci-account-iam-role, with a trust relationship to the cluster’s OIDC provider and specify the service-account, namespace to restrict the access. In this case, I am specifying ci-namespace and ci-serviceaccount for namespace and serviceaccount respectively. Replace the OIDC_PROVIDER with the provider URL saved in the previous step.
   ```
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Federated": "arn:aws:iam::CI_ACCOUNT_ID:oidc-provider/OIDC_PROVIDER"
          },
          "Action": "sts:AssumeRoleWithWebIdentity",
          "Condition": {
            "StringEquals": {
              "<OIDC_PROVIDER>:sub": "system:serviceaccount:ci-namespace:ci-serviceaccount"
            }
          }
        }
      ]
    }
   ```
  attached follow S3 policy to access the other account S3 bucket
  ```
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Action": [
                  "s3:ListBucket"
              ],
              "Resource": [
                  "arn:aws:s3:::BucketName"
              ]
          },
          {
              "Effect": "Allow",
              "Action": [
                  "s3:GetObject",
                  "s3:PutObject",
                  "s3:PutObjectAcl"
              ],
              "Resource": [
                  "arn:aws:s3:::BucketName/*"
              ]
          }
      ]
  }
 ```

  For detailed instructions and example configurations, refer to the [instructions document](docs/instructions.md).
  In Kubernetes, you define the IAM role to associate with a service account in your cluster by adding the eks.amazonaws.com/role-arn annotation to the service account. In other words, annotate your service account associated with the cluster in the CI account with the role ARN as shown below.
  ```
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: ci-serviceaccount
  
    namespace: ci-namespace
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::CI_ACCOUNT_ID:role/ci-account-iam-role
  ```
4. **Target AWS Account S3 bucket Policy**
   ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::CI_ACCOUNT_ID:role/ci-account-iam-role"
                },
                "Action": [
                    "s3:GetObject",
                    "s3:PutObject",
                    "s3:PutObjectAcl",
                    "s3:ListBucket"
                ],
                "Resource": [
                    "arn:aws:s3:::BucketName",
                    "arn:aws:s3:::BucketName/*"
                ]
            }
        ]
    }
   ```
## Contributing

Contributions to this repository are welcome! If you have any suggestions, improvements, or additional examples, feel free to open an issue or pull request.

## License

This project is licensed under the [MIT License](LICENSE).
