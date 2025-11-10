Working with AWS Lambda
Overview

This project demonstrates how to deploy and configure an AWS Lambda-based serverless computing solution that generates a daily sales analysis report.
The solution automatically pulls data from a café database running on an EC2 LAMP instance, formats the results, and sends a report by email using Amazon SNS.
It includes two Lambda functions, an SNS topic for notifications, and a CloudWatch Events (EventBridge) rule that triggers execution on a schedule.

🎯 Objectives & Learning Outcomes

After completing this lab, you will be able to:

Identify IAM roles and permissions required for Lambda functions to access AWS resources.

Create a Lambda layer to include external Python libraries (PyMySQL).

Deploy two Lambda functions:

salesAnalysisReportDataExtractor — extracts data from a MySQL database.

salesAnalysisReport — formats the results and emails the report.

Configure CloudWatch Events (EventBridge) to invoke Lambda on a schedule.

Use CloudWatch Logs to troubleshoot function execution and performance.

🏗️ AWS-Style Architecture Diagram

Workflow Summary

CloudWatch Events / EventBridge triggers the salesAnalysisReport Lambda function daily at 8 PM (Mon–Sat).

salesAnalysisReport invokes the salesAnalysisReportDataExtractor Lambda function.

salesAnalysisReportDataExtractor retrieves sales data from the café database on an EC2 LAMP instance.

The data is returned to the main function.

salesAnalysisReport formats the report and publishes it to an SNS topic.

The SNS topic sends the report email to the administrator.

💻 Commands & Configuration Steps (Copy-All)
# 1️⃣ Configure AWS CLI credentials on the CLI Host instance
aws configure
# Provide AccessKey, SecretKey, Region (us-west-2), and output (json)

# 2️⃣ Verify files exist
cd activity-files
ls
# Expect: salesAnalysisReport-v2.zip, pymysql-v3.zip, salesAnalysisReportDataExtractor-v3.zip

# 3️⃣ Create Lambda Layer for PyMySQL
aws lambda publish-layer-version \
  --layer-name pymysqlLibrary \
  --description "PyMySQL library modules" \
  --zip-file fileb://pymysql-v3.zip \
  --compatible-runtimes python3.9

# 4️⃣ Create Data Extractor Function
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

# 5️⃣ Test Extractor Function with DB credentials
# Replace placeholder values with Parameter Store outputs
aws lambda invoke \
  --function-name salesAnalysisReportDataExtractor \
  --payload '{"dbUrl":"<url>","dbName":"<name>","dbUser":"<user>","dbPassword":"<pwd>"}' \
  response.json

# 6️⃣ Create SNS Topic and Subscription
aws sns create-topic --name salesAnalysisReportTopic
aws sns subscribe \
  --topic-arn arn:aws:sns:us-west-2:<account-id>:salesAnalysisReportTopic \
  --protocol email \
  --notification-endpoint your_email@example.com

# Confirm the subscription in your inbox.

# 7️⃣ Create Main Lambda Function (salesAnalysisReport)
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

# 8️⃣ Create CloudWatch Event Rule for Schedule Trigger (Mon–Sat, 8 PM UTC)
aws events put-rule \
  --name salesAnalysisReportDailyTrigger \
  --schedule-expression "cron(0 20 ? * MON-SAT *)" \
  --description "Initiates report generation on a daily basis"

# Add Lambda as target
aws events put-targets \
  --rule salesAnalysisReportDailyTrigger \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-west-2:<account-id>:function:salesAnalysisReport"

🖼️ Screenshots to Include
#	Screenshot Name	Description
1️⃣	iam-roles-lambda.png	IAM roles showing salesAnalysisReportRole and salesAnalysisReportDERole.
2️⃣	lambda-layer-created.png	Confirmation of pymysqlLibrary Lambda layer.
3️⃣	data-extractor-test-success.png	Successful output from salesAnalysisReportDataExtractor.
4️⃣	sns-topic-and-subscription.png	SNS topic with confirmed email subscription.
5️⃣	sales-report-lambda-success.png	Test results showing report sent successfully.
6️⃣	daily-email-report.png	Screenshot of “Daily Sales Analysis Report” email received.
⚙️ Tools Used

AWS Lambda – Serverless functions execution

Amazon SNS – Notification delivery service

Amazon CloudWatch / EventBridge – Scheduled triggers and monitoring

AWS Systems Manager Parameter Store – Secure configuration management

Amazon EC2 (LAMP) – Application database backend

AWS CLI & IAM – Deployment and permissions

🧾 Key Takeaways

Lambda layers streamline dependency management and code reuse.

IAM roles must grant precise permissions for Lambda to access SNS, SSM, and CloudWatch.

EventBridge scheduling enables automated reporting workflows.

SNS integration turns Lambda output into real-time notifications.

CloudWatch logs simplify debugging and performance analysis.

🪄 What Actually Happened

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

🗣️ How to Explain It to an Interviewer

“I deployed a serverless reporting solution using AWS Lambda. One function extracted sales data from an EC2-hosted MySQL database, while another formatted and emailed daily reports through SNS. I created a Lambda layer for PyMySQL, configured IAM roles for permissions, and scheduled the Lambda function via CloudWatch Events. When triggered, the main function invoked the extractor, retrieved the data, and published a formatted report to the SNS topic. This design fully automated daily reporting without managing servers.”

👩🏽‍💻 Author

Amarachi Emeziem (Amara) – Cloud Security & AWS Engineer
GitHub Profile: @Amara-lorritta
