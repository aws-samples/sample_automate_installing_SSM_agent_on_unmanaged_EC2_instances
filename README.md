# Automate installing SSM agent on unmanaged EC2 instances in an AWS Organization

This repo hosts templates written for the AWS Blog Post [Automate installing AWS Systems Manager agent on unmanaged Amazon EC2 nodes](https://aws.amazon.com/blogs/mt/automate-installing-ssm-agent-on-unmanaged-ec2-instances-in-an-aws-organization/) published on the [AWS Cloud Operations & Migrations](https://aws.amazon.com/blogs/mt/) blog channel.

For details on how to use the corresponding CloudFormation templates, refer to the blog post.

CloudFormation Templates:

* [SSMAgentMultiAccountInstallation.yaml](/Templates/CloudFormation/SSMAgentMultiAccountInstallation.yaml)

![Architectural Diagram](/Images/ArchitecturalDiagram.png)

## Runbooks & Process Flow

### SSMAgentInstall-Orchestrator

**Purpose**: Coordinates the overall automation across multiple AWS accounts in your organization.

**Key Steps**:
1. **Input Processing**
   - Accepts targeting parameters (tag-based or S3 diagnosis results)
   - Validates inputs and determines execution path

2. **Target Method Branching**
   - **Path A**: For tag-based targeting
     - Processes tag key-value pairs to identify target instances
   - **Path B**: For diagnosis-based targeting
     - Reads S3 diagnosis file to identify unmanaged instances
     - Extracts account IDs from the diagnosis file
     - Copies diagnosis file to centralized S3 location for extraction.

3. **Multi-Account Execution**
   - Invokes SSMAgentInstall-Primary across target accounts
   - Concurrency set at max 5 concurrent account executions.

4. **Results Consolidation**
   - Generates comprehensive CSV reports by combining results collected by the SSMAgentInstall-Primary runbook.
   - Provides summary statistics (total instances, success/failure counts)

### SSMAgentInstall-Primary

**Purpose**: Manages the deployment process within a single AWS account across multiple regions.

**Key Steps**:
1. **Instance Targeting**
   - **For tag-based targeting**:
     - Uses provided tags to directly target instances
   - **For diagnosis-based targeting**:
     - Processes S3 diagnosis file to extract instance IDs by region
     - Adds temporary tags (SSMAgentInstall-AutomationExecutionID) to instances
     - Handles tag operations in batches with rollback capability

2. **Secondary Runbook Execution**
   - Invokes SSMAgentInstall-Secondary for each target instance for each target regions.
   - Controls concurrency (max 40 concurrent instance operations)

3. **Account-Level Results Collection**
   - Aggregates results from all instances in the account.
   - Compiles account-specific results into structured data.
   - Uploads consolidated account results to S3.

### SSMAgentInstall-Secondary

**Purpose**: Performs the actual SSM Agent installation on individual EC2 instances.

**Key Steps**:
1. **Instance Evaluation**
   - Validates instance prerequisites:
     - Verifies instance is running with EBS root volume
     - Checks instance is not in Auto Scaling group or a spot instance
     - Confirms shutdown behavior is set to 'stop'
     - Confirms instance is running.

2. **Instance Preparation**
   - Stops the instance safely if prerequisites are met
   - Updates instance userdata with customized installation scripts
   - Attaches temporary IAM instance profile for S3 access

3. **Installation Execution**
   - Starts the instance to trigger userdata execution
   - Waits for installation process to complete (3-minute sleep)
   - The userdata script:
     - Installs required dependencies (curl, unzip, AWS CLI)
     - Downloads and installs appropriate SSM Agent package
     - Configures agent to start automatically
     - Uploads detailed logs to central S3 bucket

4. **Instance Restoration**
   - Stops the instance to restore original configuration
   - Restores the original userdata
   - Detaches temporary IAM profile
   - Reattaches original IAM profile if one existed
   - Returns instance to its original state (running/stopped)

5. **Installation Verification**
   - Retrieves installation logs from S3.
   - Verifies installation success
   - Reports status and detailed output


## FAQs

### How can I increase concurrency to target more instances?
The default concurrency settings are:
```
TargetsMaxConcurrency: '40'
TargetsMaxErrors: '100%'
```
You can modify these values in the CloudFormation template to increase the number of instances targeted concurrently. **NOTE:** Increasing the targets may lead to API throttlin.

### Why are instances not available in Fleet Manager after automation, showing as "Connection Lost"?
This typically happens if Default Host Management Configuration (DHMC) is enabled on the region. When DHMC is enabled, it may take approximately 30 minutes for the DHMC credentials to be used by the agent. For more information, see the [DHMC documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/fleet-manager-default-host-management-configuration.html#dhmc-activate).

### What regions should I specify for the automation?
The regions you specify when executing the Orchestrator runbook should be the same as or a subset of the regions where you deployed the CloudFormation stack (defined by the TargetRegions parameter).

### How do I read the automation report?
The automation generates a CSV report file with the naming format `combined_results_YYYYMMDD_HHMMSS.csv`. This file contains detailed information about the SSM Agent installation attempts on each instance.

The FinalOutput column shows the installation progress:

**Successful Installation with Summary:**
```
Directory /var/log/UserdataSSMAgentInstallation/ created to store logs.
Successfully Installed SSM Agent.
amazon-ssm-agent-3.3.2471.0-1.x86_64
#### Post Installation Check ####
[SUCCESS] ssm-cli showed agent is running and connectivity to ssm endpoint passed
```

**Failed Instances:**
```
[SKIPPED]: Instance does not meet the prerequisites: The instance i-0a6c4570bc4ce49e4 is part of an Auto Scaling group. Exiting.
```
```
[SKIPPED]: Instance does not meet the prerequisites: The instance i-0a20de6d9b7299c15 is a spot instance. Exiting.
```

```
Couldn't find userdata logs in the S3 bucket. It is possible the installation was complete but log upload failed from the instance. Confirm if the instance is now managed from Fleet Manager console. You can check the Execution Log file under C:\Windows\TEMP\ssm\log
```
The last error message may occur when no logs were pushed from the instance. This can happen for various reasons, there is no cloud-init installed or running, Upload to s3 failed due to permissions or connectivity issue. Login to the instance and verify if log file is created under /var/log/UserdataSSMAgentInstallation/ or C:\Windows\TEMP\ssm\log.

### Which instances are targeted when using Diagnose and Remediate output?
From the DiagnoseAndRemediateS3Results parameter, the automation reads the CSV file and filters for unmanaged instances with "Unidentified Issues". "Unidentified Issues" means the instance's issue found doesn't belong to any of the categories defined in the [EC2 diagnosis category types documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/diagnosing-ec2-category-types.html).


## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
