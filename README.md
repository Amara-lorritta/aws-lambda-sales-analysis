## **Working with AWS Lambda**

## **Overview**

This project demonstrates how to deploy and configure an AWS Lambda-based serverless computing solution that generates a daily sales analysis report.
The solution automatically pulls data from a café database running on an EC2 LAMP stack (Linux, Apache, MySQL, and PHP) deployed on an Amazon EC2 instance, which is a virtual server in the cloud. LAMP instance, formats the results, and sends a report by email using Amazon SNS.
It includes two Lambda functions, an SNS topic for notifications, and a CloudWatch Events (EventBridge) rule that triggers execution on a schedule.

## **Objectives & Learning Outcomes**

After completing this lab,I was able to:

Identify IAM roles and permissions required for Lambda functions to access AWS resources.

Create a Lambda layer to include external Python libraries (PyMySQL).

Deploy two Lambda functions:

salesAnalysisReportDataExtractor — extracts data from a MySQL database.

salesAnalysisReport — formats the results and emails the report.

Configure CloudWatch Events (EventBridge) to invoke Lambda on a schedule.

Use CloudWatch Logs to troubleshoot function execution and performance.

## **Architecture Diagram**

<img width="1024" height="517" alt="ChatGPT Image Nov 10, 2025, 06_03_54 AM" src="https://github.com/user-attachments/assets/139cd15e-24e8-4c1a-8c84-fbb5a9f65b5b" />

### Workflow Summary

CloudWatch Events / EventBridge triggers the salesAnalysisReport Lambda function daily at 8 PM (Mon–Sat).

salesAnalysisReport invokes the salesAnalysisReportDataExtractor Lambda function.

salesAnalysisReportDataExtractor retrieves sales data from the café database on an EC2 LAMP instance.

The data is returned to the main function.

salesAnalysisReport formats the report and publishes it to an SNS topic.

The SNS topic sends the report email to the administrator.

## **Commands & Configuration Steps**

```bash
# Configure AWS CLI credentials on the CLI Host instance
aws configure
# Provide AccessKey, SecretKey, Region (us-west-2), and output (json)

# Verify files exist
cd activity-files
ls
# Expect: salesAnalysisReport-v2.zip, pymysql-v3.zip, salesAnalysisReportDataExtractor-v3.zip

# Create Lambda Layer for PyMySQL
aws lambda publish-layer-version \
  --layer-name pymysqlLibrary \
  --description "PyMySQL library modules" \
  --zip-file fileb://pymysql-v3.zip \
  --compatible-runtimes python3.9

# Create Data Extractor Function
aws lambda create-function \
  --function-name salesAnalysisReportDataExtractor \
  --runtime python3.9 \
  --role arn:aws:iam::<account-id>:role/salesAnalysisReportDERole \
  --handler salesAnalysisReportDataExtractor.lambda_handler \
  --zip-file fileb://salesAnalysisReportDataExtractor-v3.zip

# Attach the Lambda layer
aws lambda update-function-configuration \
  --function-name salesAnalysisReportDataExtractor \
  --layers arn:aws:lambda:us-west-2:<account-id>:layer:pymysqlLibrary:1

# Test Extractor Function with DB credentials
# Replace placeholder values with Parameter Store outputs
aws lambda invoke \
  --function-name salesAnalysisReportDataExtractor \
  --payload '{"dbUrl":"<url>","dbName":"<name>","dbUser":"<user>","dbPassword":"<pwd>"}' \
  response.json

# Create SNS Topic and Subscription
aws sns create-topic --name salesAnalysisReportTopic
aws sns subscribe \
  --topic-arn arn:aws:sns:us-west-2:<account-id>:salesAnalysisReportTopic \
  --protocol email \
  --notification-endpoint your_email@example.com

# Confirm the subscription in your inbox.

# Create Main Lambda Function (salesAnalysisReport)
aws lambda create-function \
  --function-name salesAnalysisReport \
  --runtime python3.9 \
  --zip-file fileb://salesAnalysisReport-v2.zip \
  --handler salesAnalysisReport.lambda_handler \
  --region us-west-2 \
  --role arn:aws:iam::<account-id>:role/salesAnalysisReportRole

# Add environment variable for SNS Topic
aws lambda update-function-configuration \
  --function-name salesAnalysisReport \
  --environment Variables="{topicARN=arn:aws:sns:us-west-2:<account-id>:salesAnalysisReportTopic}"

# Create CloudWatch Event Rule for Schedule Trigger (Mon–Sat, 8 PM UTC)
aws events put-rule \
  --name salesAnalysisReportDailyTrigger \
  --schedule-expression "cron(0 20 ? * MON-SAT *)" \
  --description "Initiates report generation on a daily basis"

# Add Lambda as target
aws events put-targets \
  --rule salesAnalysisReportDailyTrigger \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-west-2:<account-id>:function:salesAnalysisReport"
```

## **Screenshots**

IAM roles showing salesAnalysisReportRole and salesAnalysisReportDERole.
<img width="1408" height="284" alt="lab 178_1" src="https://github.com/user-attachments/assets/f25a53e1-a937-4e04-8cc3-f8ae355a5819" />

Confirmation of pymysqlLibrary Lambda layer.
<img width="1747" height="353" alt="178_2" src="https://github.com/user-attachments/assets/cecc178a-d664-4d80-8d7e-8f5a02953d55" />

Successful output from salesAnalysisReportDataExtractor.
<img width="1759" height="625" alt="178_4" src="https://github.com/user-attachments/assets/e58a4b46-b82f-4bb3-b60b-b5e541567e1e" />

SNS topic with confirmed email subscription.
<img width="903" height="435" alt="178_6" src="https://github.com/user-attachments/assets/891b3bf4-937d-4e41-bb53-b15545b21e79" />

Test results showing report sent successfully.
<img width="1055" height="464" alt="178_7" src="https://github.com/user-attachments/assets/bf0079cc-3bf8-4d0a-b7e2-d068fa4176e9" />


## **Tools Used**

AWS Lambda – Serverless functions execution

Amazon SNS – Notification delivery service

Amazon CloudWatch / EventBridge – Scheduled triggers and monitoring

AWS Systems Manager Parameter Store – Secure configuration management

Amazon EC2 (LAMP) – Application database backend

AWS CLI & IAM – Deployment and permissions

## **Key Takeaways**

Lambda layers streamline dependency management and code reuse.

IAM roles must grant precise permissions for Lambda to access SNS, SSM, and CloudWatch.

EventBridge scheduling enables automated reporting workflows.

SNS integration turns Lambda output into real-time notifications.

CloudWatch logs simplify debugging and performance analysis.

## **What Actually Happened**

Created IAM roles with the correct trust policies and permissions.

Built a Lambda layer to include PyMySQL.

Created the Data Extractor function to query the café MySQL database.

Fixed initial time-out by adding an inbound rule for port 3306 to the security group.

Populated the café database by placing sample orders.

Verified data extraction JSON output from Lambda.

Created the SNS Topic and confirmed the email subscription.

Created the Sales Report function to format and publish data to SNS.

Added a CloudWatch Events rule to trigger the function on a schedule.

Confirmed daily report delivery via email.


## **Author**

Amarachi Emeziem (Amara) 

Cloud Security & AWS Engineer

LinkedIn profile: https://www.linkedin.com/in/amarachilemeziem/

