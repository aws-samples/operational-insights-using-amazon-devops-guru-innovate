# Gaining operational insights with AIOps using Amazon DevOps Guru

This lab is provided as part of **[AWS Innovate AI/ML and Data Edition](https://aws.amazon.com/events/aws-innovate/apj/aiml-data/)**.

ℹ️ You will run this lab in your own AWS account. Please follow directions at the end of the lab to remove resources to avoid future costs.

ℹ️ Please let us know what you thought of this session and how we can improve the experience for you in the future by completing [the survey](#survey) at the end of the lab.
Participants who complete the surveys from AWS Innovate Online Conference will receive a gift code for USD25 in AWS credits <sup> 1, 2 & 3 </sup>.

## **Overview**
[Amazon DevOps Guru](https://aws.amazon.com/devops-guru/) offers a fully managed AIOps platform service that enables developers and operators to improve application availability and resolve operational issues faster. It minimizes manual effort by leveraging machine learning (ML) powered recommendations. DevOps Guru automatically detects operational issues, predicts impending resource exhaustion, details likely causes, and recommends remediation actions.

This is a labified version of this [AWS Blog](https://aws.amazon.com/blogs/devops/gaining-operational-insights-with-aiops-using-amazon-devops-guru/)

It will walk you through how to enable DevOps Guru for your account in a typical serverless environment and observe the insights and recommendations generated for various activities. These insights are generated for operational events that could pose a risk to your application availability. DevOps Guru uses [AWS CloudFormation](http://aws.amazon.com/cloudformation) stacks as the application boundary to detect resource relationships and co-relate with deployment events.

## Setup
### Create Cloud9 environment via AWS CloudFormation

1. Log in your AWS Account
1. Click [this link](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=DevOpsGuru&templateURL=https://awsinnovatedata2022.s3.amazonaws.com/devopsguru/cloud9.yaml) and open a new browser tab
1. Click *Next* again to the stack review page, tick **I acknowledge that AWS CloudFormation might create IAM resources** box and click *Create stack*.
  ![Acknowledge Stack Capabilities](/images/acknowledge-stack-capabilities.png)
1. Wait for a few minutes for stack creation to complete.
1. Select the stack and note down the outputs (*Cloud9EnvironmentId* & *InstanceProfile*) on *outputs* tab for next step.
  ![Cloud9 Stack Output](/images/stack-cloud9-output.png)

### Assign instance role to Cloud9 instance

1. Launch [AWS EC2 Console](https://console.aws.amazon.com/ec2/v2/home?#Instances).
1. Use stack output value of *Cloud9EnvironmentId* as filter to find the Cloud9 instance.
  ![Locate Cloud9 Instance](/images/locate-cloud9-instance.png)
1. Right click the instance, *Security* -> *Modify IAM Role*.
1. Choose the profile name matches to the *InstanceProfile* value from the stack output, and click *Apply*.
  ![Set Instance Role](/images/set-instance-role.png)

### Disable Cloud9 Managed Credentials
1. Launch [AWS Cloud9 Console](https://console.aws.amazon.com/cloud9/home?region=us-east-1#)
1. Locate the Cloud9 environment created for this lab and click "Open IDE". The environment title should start with *DevOpsGuruCloud9*.
1. At top menu of Cloud9 IDE, click *AWS Cloud9* and choose *Preferences*.
1. At left menu *AWS SETTINGS*, click *Credentials*.
1. Disable AWS managed temporary credentials:

    ![Disable Cloud 9 Managed Credentials](/images/disable-cloud9-credentials.png)

### Install prerequisite packages
Run commands below on Cloud9 Terminal to install prerequisite packages:

```
sudo yum install jq -y

export AWS_REGION=$(curl -s \
169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')

sudo pip3 install requests
```
![Install Packages](/images/install-packages.png)

Clone the git repository to download the required CloudFormation templates:
```
git clone https://github.com/aws-samples/amazon-devopsguru-samples
cd amazon-devopsguru-samples/generate-devopsguru-insights/
```
### Deploy Serverless Infrastructure
As depicted in the following diagram, we use a CloudFormation stack to create a serverless infrastructure, comprising of [Amazon API Gateway](https://aws.amazon.com/api-gateway), [AWS Lambda](http://aws.amazon.com/lambda), and [Amazon DynamoDB](https://aws.amazon.com/dynamodb/), and inject HTTP requests at a high rate towards the API published to list records.

![architecture](/images/Architecture.png)

Deploy the CloudFormation template using the following command in Cloud 9:
```
aws cloudformation create-stack --stack-name myServerless-Stack \
--template-body file:///$PWD/cfn-shops-monitoroper-code.yaml \
--capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
--region ${AWS_REGION}
```
The AWS CloudFormation deployment creates an API Gateway, a DynamoDB table, and a Lambda function with sample code.
When it’s complete, go to the AWS CloudFormation Console, select stack named **myServerless-Stack**, and choose Outputs tab of the stack. 
Record the links for the two APIs: one of them to list the table contents and other one to populate the contents.
![stack-output](/images/stack-output.png)

### Populating the DynamoDB table
Run the following commands (simply copy-paste) to populate the DynamoDB table. The below three commands will identify the name of the DynamoDB table from the CloudFormation stack and populate the name in the populate-shops-dynamodb-table.json file.
```
dynamoDBTableName=$(aws cloudformation list-stack-resources \
--stack-name myServerless-Stack \
--region ${AWS_REGION} | \
jq '.StackResourceSummaries[]|select(.ResourceType == "AWS::DynamoDB::Table").PhysicalResourceId' | tr -d '"')
```
```
sudo sed -i s/"<YOUR-DYNAMODB-TABLE-NAME>"/$dynamoDBTableName/g \
populate-shops-dynamodb-table.json
```
```
aws dynamodb batch-write-item \
--request-items file://populate-shops-dynamodb-table.json --region ${AWS_REGION}
```

This populates the DynamoDB table with a few sample records, which you can verify by accessing the ListRestApiEndpointMonitorOper API URL published on the Outputs tab of the CloudFormation stack. The following screenshot shows the output.

![listrest](/images/listrest.png)

## Lab
### Enable DevOps Guru for the CloudFormation Stack
Run the command in Cloud9 to enable DevOps Guru for this CloudFormation stack:
```
aws cloudformation create-stack \
--stack-name EnableDevOpsGuruForServerlessCfnStack \
--template-body file:///$PWD/EnableDevOpsGuruForServerlessCfnStack.yaml \
--parameters ParameterKey=CfnStackNames,ParameterValue=myServerless-Stack \
--region ${AWS_REGION}
```
When the stack is created, navigate to the Amazon DevOps Guru console.
Choose Settings.
Under CloudFormation stacks, locate **myServerless-Stack**.

If you don’t see it, your CloudFormation stack has not been successfully deployed. You may remove and redeploy the EnableDevOpsGuruForServerlessCfnStack stack.

### Waiting for baselining of resources
This is a necessary step to allow DevOps Guru to complete baselining the resources and benchmark the normal behavior. For our serverless stack with 3 resources, we recommend waiting for 2 hours before carrying out next steps. 

### Proactive Insight from DevOps Guru for AWS Best Practice
From the DevOps Guru console, view the Dashboard and at the top you will see an ongoing proactive insight. You can click on the button to navigate to the insight dashboard. See the screenshot below for what that looks like.

![db-proactive](/images/db-proactive.png)

After you click on that, you are taken to the DevOps Guru console which displays the Insights. If you clicked on the Proactive insight button, you'll automatically be taken to the Proactive insight tab. You should see the following ongoing proactive insight for enabling Point In Time Recovery in your DynamoDB table. Click on the insight link to take a deeper look into what's going on (for reference, this is the bottom of the screenshot).

![dg-proactivepage](/images/dg-ProactiveInsightPage.png)

See the top of the insight details describing a summary of what DevOps Guru has detected. 

![dg-proactivedetails](/images/03_DG_ProactiveInsightDetails.png)

Scroll down on this page or click view recommendations to see what to do to fix it. 

![dg-proactiverecommendations](/images/04_DG_ProactiveRecommendation.png)

DevOps Guru provides a recommendation of enabling Point In Time Recovery as this is an AWS best practice for production database tables. It provides you a link to the documentation for the next steps. For example, you can configure this setting in the console, via API or using Infrastructure as Code such as CloudFormation. For purposes of today's workshop, let's make this fix in the console. Go to the search bar and type DynamoDB. From the DynamoDB console page, on the left, click on Tables. You should see something like this.

![ddb-tableview](/images/05_DynamoDB_TableView.png)

Click on the table. If there are multiple tables in your environment, click on the particular DynamoDB table that is referenced in the DevOps Guru insight/recommendation page. It should start with "myServerless-Stack." After clicking on the table, navigate to the Backups table and you will see Point-in-time-recovery is currently disabled.

![dg-ddbbackup](/images/06_DynamoDB_Backups.png)

Click the edit button, this brings you to the page below where you can check the Enable point-in-time-recovery checkbox and save changes.

![07_DynamoDB_EnablePITR](/images/07_DynamoDB_EnablePITR.png)

### Updating the CloudFormation stack
We will make a configuration change to simulate a typical operational event.
In Cloud 9, update update the CloudFormation template **cfn-shops-monitoroper-code.yaml** in folder **generate-devopsguru-insights** to change the read capacity for the DynamoDB table from 5 to 1.

![read-capacity](/images/update_read_capacity.png)

Run the following command to deploy the updated CloudFormation template:

```
aws cloudformation update-stack --stack-name myServerless-Stack \
--template-body file:///$PWD/cfn-shops-monitoroper-code.yaml \
--capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
--region ${AWS_REGION}
```

### Injecting HTTP requests into your API
In the **sendAPIRequest.py**, populate the url parameter with the correct API link for the ListRestApiEndpointMonitorOper API, listed in the CloudFormation stack’s output tab. 

![api-pre](/images/api-pre.png)


![api-after](/images/api-after.png)

After the script is saved, trigger the script using the following command:

```
python sendAPIRequest.py
```
After approximately 10 minutes of the script running in a loop, an operational insight is generated in DevOps Guru.

### Review DevOps Guru Reactive Insight
While we keep sending HTTP requests to our API endpoint, DevOps Guru monitors for anomalies, logs insights that provide details about the metrics indicating an anomaly, and prints actionable recommendations to mitigate the anomaly. In this section, we review some of these insights.

Under normal conditions, DevOps Guru dashboard will show the ongoing insights counter to be zero. It monitors a high number of metrics behind the scenes and offloads the operator from manually monitoring any counters or graphs. It raises an alert in the form of an insight, only when anomaly occurs.

The following screenshot shows an ongoing reactive insight for the specific CloudFormation stack. 
When you choose the insight, you see further details. The number under the Total resources analyzed last hour may vary, so for this workshop, you can ignore this number.

![dashboard-1](/images/Dashboard-1.png)

1. **Aggregated metrics**
The following screenshot shows the Aggregated metrics section, where it shows metrics for all the resources that it detected anomalies in (DynamoDB, Lambda, and API Gateway). This helps us understand the cause and symptoms, and prioritize the right anomaly investigation.

![dynamodb-metrics](/images/dynamodb-metrics.png)

Initially, you may see only two metrics listed, however, over time, it populates more metrics that showed anomalies. You can see the anomaly for DynamoDB started earlier than the anomalies for API Gateway and Lambda, thus indicating them as after effects. In addition to the information in the preceding screenshot, you may see Duration p90 and IntegrationLatency p90 (for Lambda and API Gateway respectively, due to increased backend latency) metrics also listed.

2. **Relevant events**
Now we check the Relevant events section, which lists potential triggers for the issue. The events listed here depend on the set of operations carried out on this CloudFormation stack’s resources in this Region. This makes it easy for the operator to be reminded of a change that may have caused this issue. The dots (representing events) that are near the Insight start portion of timeline are of particular interest.

![dynamodb-events2](/images/dynamodb-events2.png)

If you need to delve into any of these events, just click on any of these points, and it provides more details as shown in the screenshot below.

![dynamodb-events-updatestack](/images/dynamodb-events-updatestack.png)

You can choose the link for an event to view specific details about the operational event (configuration change via CloudFormation update stack operation).

3. **Recommendations**
Navigate to **Recommendations** section, it's where Amazon DevOps Guru provides recommendations on how to remediate issues. As seen in the following screenshot, it recommends to roll back the configuration change related to the read capacity for the DynamoDB table. It also lists specific metrics and the event as part of the recommendation for reference.

![dynamodb-recommendations2](/images/dynamodb-recommendations2.png)

You can follow the recommendations to remediate the issues

## **Conclusion**
Throughout this lab, you have learn how to enable Amazon DevOps Guru for your account in a typical serverless environment and observe the insights and recommendations generated for various activities.

As described in the [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected),it recommends to regularly analyze the collected metrics to proactively identify and mitigate issues before they affect business outcomes.

Visit the [AWS Well-Architected Framework white paper](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html) and below related Best Practices for more information:
- [OPS08-BP01 Analyze workload metrics](https://docs.aws.amazon.com/wellarchitected/latest/framework/ops_workload_observability_analyze_workload_metrics.html)

## Survey
Let us know what you thought of this session and how we can improve the presentation experience for you in the future by completing [this event session poll](https://amazonmr.au1.qualtrics.com/jfe/form/SV_bmgERAD8tX0Eb6m?Session=HAN1). 
Participants who complete the surveys from AWS Innovate Online Conference will receive a gift code for USD25 in AWS credits <sup> 1, 2 & 3 </sup>. AWS credits will be sent via email by **March 29, 2024**.
Note: Only registrants of AWS Innovate Online Conference who complete the surveys will receive a gift code for USD25 in AWS credits via email.

<sup>1</sup>AWS Promotional Credits Terms and conditions apply: https://aws.amazon.com/awscredits/

<sup>2</sup>Limited to 1 x USD25 AWS credits per participant.

<sup>3</sup>Participants will be required to provide their business email addresses to receive the gift code for AWS credits.
