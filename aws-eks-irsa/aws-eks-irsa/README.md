## Sending Emails from EKS Pods: Leveraging IRSA (IAM Roles for Service Account) with AWS SES Across Accounts

Managing email communications from Kubernetes pods within Amazon EKS (Elastic Kubernetes Service) can be challenging, especially when the email service (Amazon SES) is located in a different AWS account. Traditionally, managing IAM credentials and securely configuring permissions across accounts involves complex setups and potential security risks of using Access Keys that can be compromised. The problem intensifies when developers need to ensure that their applications can send emails efficiently and securely without compromising on the principles of least privilege and access management.

By the end of the article, readers will have a clear understanding of how to configure their EKS clusters to send emails using Amazon SES in a different AWS account, leveraging the power of IRSA to manage permissions securely and efficiently. This guide will help developers and DevOps engineers simplify their setup, enhance security, and streamline their email-sending workflows from Kubernetes pods. This is not only limited to SES but other AWS services that permissions can be managed by roles.

- [Introduction to IRSA and its Benefits](#introduction-to-irsa-and-its-benefits)
- [IRSA Configuration](#irsa-configuration)
- [Cross-Account Access](#cross-account-access)
- [SES Configuration](#ses-configuration)
- [Testing and Validation](#testing-and-validation)
- [Best Practices and Security Considerations](#best-practices-and-security-considerations)

### Introduction to IRSA and its Benefits

#### What is IAM Roles for Service Accounts (IRSA)

IAM Roles for Service Accounts (IRSA) allows Kubernetes service accounts to assume IAM roles, similar to the way that Amazon EC2 instance profiles provide credentials to Amazon EC2 instances. Instead of creating and distributing your AWS credentials to the containers or using the Amazon EC2 instance's role, you associate an IAM role with a Kubernetes service account and configure your Pods to use the service account. . This enables pods running on EKS to interact with AWS services securely without embedding AWS credentials in the pods. Applications in a Pod's containers can use an AWS SDK or the AWS CLI to make API requests to AWS services using AWS Identity and Access Management (IAM) permissions.

The following steps explains steps in how IRSA works to assign the pod temporary credentials.

![IRSA Flow Diagram](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hmns8dh9mpjn1szb53z5.jpg)

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
            "Action": [
                "ses:SendEmail",
                "ses:SendRawEmail"
            ],
            "Resource": [
                "arn:aws:ses:<REGION>:<SES ACCOUNT ID>:identity/<SENDER EMAIL>",
                "arn:aws:ses:<REGION>:<SES ACCOUNT ID>:identity/<RECEIVER EMAIL>"
            ]
        }
  ]
}
```

Since we are using the SES Sandbox environment, both the Sender and Receiver needs to be verified on SES. This is different on the Production environment where the receivers will not need any verification hence on the IAM Policy the arn of the receiver won't be needed.

#### Create an IAM Role with Trust Policy

Create an IAM role with a trust policy that allows the EKS service account to assume the role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<EKS ACCOUNT ID>:oidc-provider/<OIDC_PROVIDER>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<OIDC_PROVIDER>:sub":"system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT_NAME>",
          "<OIDC_PROVIDER>:aud":"sts.amazonaws.com"
        }
      }
    }
  ]
}
```

#### Associate the IAM Role with a Kubernetes Service Account

Create a Kubernetes service account and annotate it with the IAM role ARN created.

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <SERVICE_ACCOUNT_NAME>
  namespace: <NAMESPACE>
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<EKS ACCOUNT ID>:role/<ROLE_NAME>
```

### Cross-Account Access

Accessing AWS resources across different accounts can be complex due to different security boundaries and policies. The key challenge is to securely grant permissions to SES resources from the EKS cluster in another account.

- **Cross-Account Trust Policies:** Establish trust policies that allow IAM roles in EKS Account to access SES in the different account.
- **IRSA Configuration:** Use IRSA to link EKS service accounts with IAM roles, enabling secure communication.

### SES Configuration

#### Verify Your Domain or Email Address.

In SES we have two environments (Sandbox and Production). All new accounts come configured with the SES Sandbox and you need to request it to be converted to the SES Production. ([How to Request SES Production Access](https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html))

For the demo I will be using SES SandBox and email address to configure the email identities. I will create two identities one for the sender and the other for the receiver. In SES Sandbox, you need both sender and recipient emails to be verified  This restriction is in place to prevent spam and misuse of the service while you're testing.

When your account has moved out of the sandbox and into production, you can send email to any recipient, regardless of whether the recipient's address or domain is verified. However, you still have to verify all identities that you use as "From", "Source", "Sender", or "Return-Path" addresses.

![New SES Identity](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/b2qnwl58sg82tr409g1r.png)

#### Create an Authorization policy for Sending Email Identity

Create an authorization Policy for the Sender Email Address identity to allow the Service Account role created to send mails

![SES Sender Authorization Config](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zmumataol0d4s0rrdio7.png)


![SES Sender Policy Config](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/crelspxmagcz0f0r0555.png)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<EKS ACCOUNT ID>:role/<SERVICE_ACCOUNT_NAME_ROLE>"
            },
            "Action": [
                "ses:SendEmail",
                "ses:SendRawEmail"
            ],
            "Resource": "arn:aws:ses:<REGION>:<SES ACCOUNT ID>:identity/<SENDER EMAIL>"
        }
    ]
}
```

### Testing and Validation

#### Deploy a sample application in your EKS cluster that uses the service account to send emails via SES.

Use the following deployment file to create a pod that uses an image with AWS CLI so that you can test sending the mail via CLI.

This will create a deployment with one replica in the default namespace and use the Service Account created with the IAM role configured

``` bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ses-test
  labels:
    app: ses-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ses-test
  template:
    metadata:
      labels:
        app: ses-test
    spec:
      serviceAccountName: <SERVICE_ACCOUNT_NAME>
      containers:
      - name: ses-test
        image: amazon/aws-cli
        command: [ "sh", "-c", "sleep 3600" ]


```

Confirm the pod is running

``` bash
kubectl get pods | grep ses-test
```

#### Verify that your application can send emails by sending a test email

Once the pod is running, execute the below commands to confirm that you can send an email from the pod.

``` bash
kubectl exec -it <POD_NAME> -- /bin/sh
```

``` bash
aws ses send-email \
  --region <REGION>>\
  --from "<SENDER_EMAIL>" \
  --source-arn "arn:aws:ses:<REGION>:<SES_ACCOUNT_ID>:identity/<SENDER_EMAIL>" \
  --destination "ToAddresses=[\"<RECEPIENT_EMAIL>\"]" \
  --message "Subject={Data=Test email from EKS pod,Charset=utf-8},Body={Text={Data=This is a test email sent from a pod in EKS Account using SES in a different account.,Charset=utf-8}}"
```

![Validation of Mail Received](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5afz0amopyw9ennh19qi.png)


### Best Practices and Security Considerations

- **Limit Permissions:**  Assign only the necessary permissions to your IAM roles to follow the principle of least privilege.
- **Monitor Access:** Regularly monitor access and review IAM policies and roles for any changes or anomalies.