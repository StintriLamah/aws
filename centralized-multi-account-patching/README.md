### AWS Automated Centralized Multi Account Patching
This is a walkthrough on the steps needed to set up the Systems Manager Management Account and Target accounts for Patching.

  - [Setup Resource Groups to Logically Group your Managed Instances](#setup-resource-groups-to-logically-group-your-managed-instances)
  - [Setup the Required IAM permissions on Management account](#setup-the-required-iam-permissions-on-management-account)
  - [Create an Automation Document to Execute Patch Baselines](#create-an-automation-document-to-execute-patch-baselines)
  - [Execute Automation to Patch Multi Account Resources](#execute-automation-to-patch-multi-account-resources)
  - [Schedule Event Bridge Invocation during Patching Window](#schedule-event-bridge-invocation-during-patching-window)


![Architecture Design](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5d298395ezg12xdcay7z.png)


Many organizations struggle with effectively managing vulnerabilities and patching across their various environments, such as Production, UAT, and Staging. AWS Systems Manager Automation offers a solution by supporting multi-account and multi-Region actions, allowing centralized management of AWS resources. This capability streamlines configuration, operational tasks, and compliance efforts across the enterprise.

In this article, I'll illustrate how to utilize Resource Groups for organizing instances for patching purposes. For instance, you can create Resource Groups for different environments like development, test, and production. Additionally, I'll demonstrate the creation of a custom Automation Document that harnesses Patch Manager. Finally, I'll guide you through executing this custom Automation Document to install patches on your managed instances, which can be scheduled to run during a designated patching Maintenance Window.


Begin by selecting a single account to serve as your management account, alongside a designated AWS Region to act as your management Region. With this chosen management setup, you'll be able to schedule Automation tasks from this central location, directing them towards other AWS accounts and Regions as needed.

### Setup Resource Groups to Logically Group your Managed Instances


Resource groups serve as a helpful tool for organizing your AWS resources, providing a streamlined approach to managing and automating tasks across numerous resources simultaneously. By categorizing resources based on their function, such as distinguishing between web servers and databases, resource groups simplify operations and reduce the risk of applying patches incorrectly.
- Open your **Target account** and navigate to the service *Resource Groups & Tag Editor*.
- Create a new Resource Group using tags associated with your managed instances. It's essential to have previously tagged the instances you wish to manage.
- Specify the resource type as *AWS::EC2::Instance*.
- For the tags, ensure you've tagged your instances using the designated Tag Key (e.g., PatchGroup) and Tag Value (e.g., 1). This allows for accurate identification and grouping of instances within the resource group.


![Resource Group](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/u5ixfegxnf0d40zbjl8v.png)

- Preview the Resources and Create the Resource Group


![Create RG](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ah3ps2rgm8k8csvi9qb0.png)


### Setup the Required IAM permissions on Management account
From the selected **Management account**, you need to provision the administrator automation role that will assume the execution roles on the Targeted accounts.

- Navigate to *CloudFormation* Console and Create a stack from  [AWS-SystemsManager-AutomationAdministrationRole](https://github.com/StintriLamah/aws/blob/main/centralized-multi-account-patching/AWS-SystemsManager-AutomationAdministrationRole.yml). 


![Create Admin CFN Stack](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gteath73xar6bmrflhko.png)

- Ensure the user you are logged in with should have `AmazonSSMAutomationRole` attached, `iam:PassRole` and `resource-groups:ListGroupResources` actions in order to target Resource Groups and pass the role to the Automation Administrator Role created. Here's an example of an inline policy you can create to achieve this:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "resource-groups:ListGroupResources"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:PassRole"
            ],
            "Resource": "arn:aws:iam::<ManagementAccountId>:role/AWS-SystemsManager-AutomationAdministrationRole"
        }
    ]
}
```

Make sure to replace `<ManagementAccountId>` in the policy below with the account ID of the management account.

### Setup the Required IAM permissions on Target account

To provision the Execution IAM role in the account you wish to target for Automation tasks, you can use the provided CloudFormation template.

- Go to *CloudFormation* Console and Create a stack from the [AWS-SystemsManager-AutomationExecutionRole](https://github.com/StintriLamah/aws/blob/main/centralized-multi-account-patching/AWS-SystemsManager-AutomationExecutionRole.yml)


![Create Exec CFN Stack](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7poexr9p0ehtj50vsjal.png)
- Provide a name and the `<Management Account ID>` in the Parameters

![Provide ID for Exec](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/46a992khl91udhbrysi4.png)


### Create an Automation Document to Execute Patch Baselines
To create an automation document for executing the `AWS-RunPatchBaseline` command, follow these steps in the Management Account:

- Go to *Systems Manager* and open Documents section from the left Navigation Menu.
- Click on the **Create document** button to Start Creating an automation document and Select **Automation** as the document type.

![Automation Document Creation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/brqsown31c0tck3p3ogd.png)
- Provide a unique name for the Automation Runbook. 
- From [Automation-RunPatchBaseline](https://github.com/StintriLamah/aws/blob/main/centralized-multi-account-patching/Automation-RunPatchBaseline.json), copy the JSON content and replace the default content with JSON content copied.
![Automation Document JSON](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/a72cseaxllq8j582lqjs.png)
- Create runbook to create the Automation document

### Execute Automation to Patch Multi Account Resources
To execute the Automation Runbook on the **Management Account**:

- From the Left Navigation pane on *Systems Manager*, select **Automation**
- Click on **Execute Automation** to initiate the execution process.

- Choose the Automation Runbook you previously created (e.g., Automation-RunPatchBaseline) and click Execute Automation
- Select Multi account and Region 

![Execute Automation Runbook](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/alb18yl9dw37s0cjpdvx.png)

- In the **Target accounts and Regions** section:
  - Provide the *Account IDs* of the targeted accounts and specify the Region where EC2 instances are located
  - Specify the *Automation Execution Role Name* created in the target accounts
  - Optionally, specify the number or percentage of locations (account-Region pairs) on which to execute the task simultaneously.
  - Optionally, set an error threshold to stop the task after it fails on a specific number or percentage of locations.

![Target Accounts and Region](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/u1lm9puft4o22rc2uk7n.png)
- In the **Targets** Section:
  - Choose *InstanceID* as the parameter.
  - Select *Resource Group* as the targets.
  - Provide the name of the Resource Group you created earlier as the Resource group

![Targets](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/98ti6hz2ueak6khffznf.png)

- In the **Input parameters** section:
  - For the parameter *AutomationAssumeRole*, provide the IAM role "AWS-SystemsManager-AutomationAdministrationRole" that you previously created.
  - Specify *Install* as the operation to Scan and Install.
  - Optionally, you can provide an "InstallOverrideList" as a list of patches to be installed. This list will override the patches specified by the default patch baseline.
  - Optionally, if needed, you can specify the snapshot ID to use for retrieving a patch baseline snapshot.

![Input Parameters](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/woyox7c6acc6plhryqzc.png)

- In the **Rate control** section:
  - Specify the number or percentage of target instances on which to execute the task simultaneously
  - Set an error threshold, which will halt the task after it fails on a specific number or percentage of target instances.

![Rate Control](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/no81fo76znuwp9e3fmaw.png)	
- Execute the automation.

You can follow these execution steps using the AWS CLI:

- Make sure you have the AWS CLI installed and configured with the necessary permissions.
- Run the following command to execute the Automation task:

    ```bash
      aws ssm start-automation-execution 
      --document-name "Automation-RunPatchBaseline" 
      --target-parameter-name InstanceId 
      --document-version "\$DEFAULT"
      --parameters '{"AutomationAssumeRole":["arn:aws:iam::<Management Account ID>:role/AWS-SystemsManager-AutomationAdministrationRole"],"Operation":["Install"],"SnapshotId":[""],"InstallOverrideList":[""]}'
      --targets '[{"Key":"ResourceGroup","Values":["Test_Patching"]}]' 
      --target-parameter-name InstanceId 
      --max-errors "1" 
      --max-concurrency "1" 
      --target-locations '[{"Accounts":["<TargetAccountIDs>"],"Regions":["<Region>"],"ExecutionRoleName":"AWS-SystemsManager-AutomationExecutionRole"}]'
      --region <Region>
      ```
Replace **Management Account ID**, **TargetAccountIDs**, and **Region** with your actual values.


On the **Management account**, you can monitor the execution progress of the Automation task. This allows you to track the status of the task and any associated errors or issues. By keeping an eye on the execution progress, you can ensure that the patching process is proceeding as expected and take necessary actions if any problems arise.

![Execution Details](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/v7p85vs5029egd81y57p.png)

![Execution ID](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uw1n0gvs90kuee1sod3c.png)


On the **Target Account**, you can observe the execution of the AWS-RunPatchBaseline command triggered by the Automation Document created in the Management Account. This visibility allows you to monitor the progress of the patching operation directly within the targeted environment. 

![Run Command Execution](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e522fo3vg12za79n00g7.png)

![Run Command ID](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ld30a2tdpwx4ksp2ef67.png)

![Run Command 2 commands](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t7k5zkmz2vpwc4gd961o.png)



### Schedule Event Bridge Invocation during Patching Window

To enhance automation further, you can set up scheduled Patching Maintenance Windows for your organization. This can be achieved by creating a Scheduled EventBridge Rule that triggers a Lambda function. The Lambda function, in turn, initiates the Multi-Account Centralized Automation Flow, automating the patching process at designated times. It will ensure timely and consistent patching across your AWS environment.

This is the flow of tasks:

  - In the management account, the EventBridge rule is triggered based on the cron or rate-based expression specified.
  - The EventBridge rule then invokes a Lambda function, which, in turn, initiates a multi-account and multi-Region Automation workflow.
  - The Systems Manager administrator role assumes the execution role in each target account and Region.
  - The execution role initiates a Run Command task for AWS-RunPatchBaseline. This command scans for, or installs missing updates on target managed instances based on membership in the provided AWS Resource Group.

You need to deploy the CloudFormation Stack from [Scheduled-Patch-Automation](https://github.com/StintriLamah/aws/blob/main/centralized-multi-account-patching/Scheduled-Patch-Automation.yaml) to create the following resources: EventBridge rule, IAM service role for Lambda, Lambda function and Automation document to invoke the Command document `AWS-RunPatchBaseline`.

- In the **Management account**, navigate to the *CloudFormation* console and create a stack.
- Upload the `Scheduled-Patch-Automation.yaml`  template file, and then choose Next.
- Specify the stack details:
  - For `EventBridgeRuleSchedule`, enter a cron or rate-based expression for the schedule of the EventBridge rule. For example, `cron(30 22 ? * SAT *)` schedules the rule to initiate patching on *Saturdays at 22:30 UTC*. Choose a cron that matches your patching window.
  - Optionally modify the `ExecutionRoleName` to match the Automation execution role in target accounts.
  - Specify `MaximumConcurrency` and `MaximumErrors` as needed. You can specify a number, such as 10, or a percentage, such as 10%. The default value is 10%.
  - Provide the `ResourceGroupName` that includes the resources you want to target.
  - Optionally enter an HTTPS or S3 URL for `RunPatchBaselineInstallOverrideList`.
  - For `RunPatchBaselineOperation`, choose **Scan** to scan for missing updates only or **Install** to scan and install missing updates based on the rules of the patch baseline.
  - For `RunPatchBaselineRebootOption`, choose the reboot behavior for the patching operation. The valid options are **RebootIfNeeded** and **NoReboot**. 
  - Enter the list of `TargetAccounts` as comma-separated AWS account IDs (for example, *012345678901*, *987654321098*).
  - Optionally modify `TargetLocationMaxConcurrency` and `TargetLocationMaxErrors`. The default value is 1.
  - Enter the list of `TargetRegionIds` as comma-separated AWS Region names. (for example, *us-east-1*, *eu-west-1*).


![Schedule Automation CFN](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ioxwzxmexij7pr2semlk.png)
- Review the stack details, then click Next. Finally, acknowledge that CloudFormation might create IAM resources and click "Create stack".


Once the scheduled time is reached, you'll observe the automation being executed in the AWS Systems Manager Automation console.  Here, you'll notice the latest automation document triggered by Lambda and the previous executions initiated manually.

![Scheduled Automation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lr69yfy6w7z5fnzreyww.png)

