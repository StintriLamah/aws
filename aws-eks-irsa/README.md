## Sending Emails from EKS Pods: Leveraging IRSA (IAM Roles for Service Account) with AWS SES Across Accounts

Managing email communications from Kubernetes pods within Amazon EKS (Elastic Kubernetes Service) can be challenging, especially when the email service (Amazon SES) is located in a different AWS account. Traditionally, managing IAM credentials and securely configuring permissions across accounts involves complex setups and potential security risks of using Access Keys that can be compromised. The problem intensifies when developers need to ensure that their applications can send emails efficiently and securely without compromising on the principles of least privilege and access management.

By the end of the article, readers will have a clear understanding of how to configure their EKS clusters to send emails using Amazon SES in a different AWS account, leveraging the power of IRSA to manage permissions securely and efficiently. This guide will help developers and DevOps engineers simplify their setup, enhance security, and streamline their email-sending workflows from Kubernetes pods. Tjis is not only limited to SES but other AWS services that permissions can be managed by roles.

- [Introduction to IRSA and its Benefits](#introduction-to-irsa-and-its-benefits)
- [Cross-Account Access](#cross-account-access)
- [IRSA Configuration](#irsa-configuration)
- [SES Configuration](#ses-configuration)
- [Testing and Validation](#testing-and-validation)
- [Best Practices and Security Considerations](#best-practices-and-security-considerations)




### Introduction to IRSA and its Benefits

#### What is IAM Roles for Service Accounts (IRSA)

IAM Roles for Service Accounts (IRSA) allows Kubernetes service accounts to assume IAM roles, similar to the way that Amazon EC2 instance profiles provide credentials to Amazon EC2 instances. Instead of creating and distributing your AWS credentials to the containers or using the Amazon EC2 instance's role, you associate an IAM role with a Kubernetes service account and configure your Pods to use the service account. . This enables pods running on EKS to interact with AWS services securely without embedding AWS credentials in the pods. Applications in a Pod's containers can use an AWS SDK or the AWS CLI to make API requests to AWS services using AWS Identity and Access Management (IAM) permissions.

The following steps explains steps in how IRSA works to assign the pod temporary credentials.

1. A reference between the EKS Cluster and IAM is established via OIDC. This is a one-time setup per cluster.

2. A reference between a Kubernetes service account and a IAM Role has to be created.

3. The Kubernetes resource is configured with an appropriate service account annotation

4. As soon a Pod with a service account annotation comes up, the Pod Identity Webhook will be triggered and reconfigure (mutate) the Pod to use IRSA

5. The Pod assumes the specified IAM Role and connects to the AWS Security Token Service

6. AWS STS verifies the request by contacting AWS IAM

7. If the request could be verified and is valid, AWS STS assigns temporary credentials

#### Benefits of using IRSA for managing permissions in EKS

- **Security:** No need to store long-term AWS credentials in pods by using Access Keys.
- **Fine-Grained Access:** Assign least privilege permissions to specific workloads. You can scope IAM permissions to a service account, and only Pods that use that service account have access to those permissions.
- **Credential isolation:** A Pod's containers can only retrieve credentials for the IAM role that's associated with the service account that the container uses. A container never has access to credentials that are used by other containers in other Pods.
- **Auditability** – Access and event logging is available through AWS CloudTrail to help ensure retrospective auditing


### IRSA Configuration
Before configuring IRSA, ensure that the EKS cluster has OIDC enabled.

The following configurations will be done on the account where the EKS Cluster and pods are running.

#### Create IAM Policy

Create an IAM policy that allows sending emails using SES.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ses:SendEmail",
      "Resource": "*"
    }
  ]
}
```

#### Create an IAM Role with Trust Policy

Create an IAM role with a trust policy that allows the EKS service account to assume the role.

````json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<Account_B_ID>:oidc-provider/<OIDC_PROVIDER>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<OIDC_PROVIDER>:sub": "system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT_NAME>"
        }
      }
    }
  ]
}
````

#### Associate the IAM Role with a Kubernetes Service Account

Create a Kubernetes service account and annotate it with the IAM role ARN.

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <SERVICE_ACCOUNT_NAME>
  namespace: <NAMESPACE>
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<Account_B_ID>:role/<ROLE_NAME>
```

### Cross-Account Access

Accessing AWS resources across different accounts can be complex due to different security boundaries and policies. The key challenge is to securely grant permissions to SES resources from the EKS cluster in another account.

- **Cross-Account Trust Policies:** Establish trust policies that allow IAM roles in EKS Account to access SES in the different account.
- **IRSA Configuration:** Use IRSA to link EKS service accounts with IAM roles, enabling secure communication.


### SES Configuration

- Verify Your Domain and Email Address. Ensure that your domain and email address are verified in Amazon SES.
- Create an IAM policy in Account A that allows sending emails
- Update the SES policy in Account A to allow the role from Account B to send emails
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::<Account_B_ID>:role/<ROLE_NAME>"
        },
        "Action": "ses:SendEmail",
        "Resource": "*"
      }
    ]
  }
  ```

### Testing and Validation

- Deploy a sample application in your EKS cluster that uses the service account to send emails via SES.
- Verify that your application can send emails by sending a test email.

### Best Practices and Security Considerations

- **Limit Permissions:**  Assign only the necessary permissions to your IAM roles to follow the principle of least privilege.
- **Monitor Access:** Regularly monitor access and review IAM policies and roles for any changes or anomalies.