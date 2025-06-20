# Phase 6: Splunk Log Ingestion Setup
In This phase, I will use Terraform to build out a Linux server and install/configure Splunk Enterprise for log aggregation across the AWS environment. I will also create two additional Windows EC2 instances within the private subnet that will act as "workstations" that will be targeted later during our simulated attacks.

## 🎯 Main Target Goals for this Phase:
- Configure Splunk Ubuntu server so it can receive logs.
- Forward Sysmon logs from all Windows EC2s to Splunk
- Forward Active Directory logs to Splunk
- Forward all AWS services logs to Splunk (Guard Duty, Cloud Trail, Cloudwatch, VPC flow logs, and IAM Acces Analyser)

---

## 🔧 Technology Utilized
- Terraform
- AWS EC2 with Windows Server 2022
- Windows AD Domain Controller
- Active Directory and Group Policies
- AWS Security Groups
- Sysmon
- AWS Guard Duty, Cloud Trail, Cloudwatch, VPC flow logs, and IAM Acces Analyser

---

## 📽️ Video: Lessons Learned

[![Watch the video](https://img.youtube.com/vi/vQeKo0BsaKw/0.jpg)](https://youtu.be/vQeKo0BsaKw)

---

# Table of Contents

- [Creating Sec Group Outbound rule for workstations to communicate with AD Server](#creating-sec-group-outbound-rules-for-workstations-to-communicate-with-ad-server)
- [Adding Workstations to my Domain Controller](#setting-the-workstation-to-join-my-ad-domain-controller)
- [Installing Sysmon on all my EC2s for Logging](#installing-sysmon-on-my-4-windows-ec2s)
- [Creating AD Group Policy Objects for Windows Workstations](#creating-ad-group-policies-for-my-windows-workstations)
- [Creating a new IAM User with permissions to manage AWS monitoring services](#-step-2-creating-a-new-iam-user-with-permissions-to-configure-aws-security-monitoring-services)
- [Configuring AWS monitoring services withing AWS dashboard](#-step-3-configuring-aws-security-monitoring-services)


---

## ⭐ Step 1️: Configuring Splunk to Receive Logs

Before I setup log forwarding, I first need to ensure that my Splunk server is set up to actually receive logs. Specifically, I need to make sure that:
1. The correct ports are open in the Splunk/Ubuntu EC2 security group for log traffic
2. Setup my Splunk account to receive logs from sysmon and windows AD logs.
3. Enable HTTP Event Collector (HEC) so that AWS logs can be forwarded to Splunk
4. Install "Splunk Add Ons" for AWS services and Microsoft Windows so that Splunk can properly parse these logs.


### Ensuring the Correct Ports are Open in the Ubuntu Security Group

In order for my Splunk EC2 to receive logs, I first needed to open up the following inbound ports from the security group:
- TCP 9997 for Splunk Universal Forwarder logs generated by my EC2s (Sysmon, AD, etc.)
- TCP 8088 for AWS services logs forwarded through HEC.
- TCP 8000 for Splunk GUI access from my Bastion server.

![image](https://github.com/user-attachments/assets/4f3b2240-f872-4772-b26e-2bbbb02c34d3)


#### Setup My Splunk Account to Receive Sysmon/AD Logs via Port 9997

Now that I have opened up the security group, I next have to go into my Splunk account and enable the receiving of Sysmon/AD logs via port 9997.

After confirming that Splunk was running in my Ubuntu machine, I then used my Bastion VM to open up a browser and log into my Splunk account via the DNS of the Ubuntu/Splunk server (it doesn't have a Public IP) to access the Splunk web UI using the following:

`http://<SPLUNKEC2-PUBLIC-DNS>:8000`

![image](https://github.com/user-attachments/assets/0243d612-f4ff-490d-97b8-bb83bffcd8dc)

![image](https://github.com/user-attachments/assets/9d327a64-ba3c-4429-871b-a2b0d7c50193)


**⚠️NOTE⚠️**: 
Bastion could not connect to the internet initially, I kept getting error messages that DNS could not be found for the URLS. Eventually, I discovered that when I added my Bastion Ec2 to my Windows AD domain controller (and changed the domain name) it automatically stops using Public DNS servers like Google's 8.8.8.8, and instead, switches to my domain controller's private IP as a DNS server; and thus I lose internet name resolution/internet connection. To resolve this, I had to log into my Windows AD Domain controller EC2 and setup up domain forwarding to 8.8.8.8 and 8.8.4.4 (And update outbound DNS rules to connect to 8.8.8.8) so that I can re-establish internet access to my EC2s connected to my Domain Controller!

Once I fixed the DNS name resolution issue, I went ahead and visited `http://<SPLUNKEC2-PUBLIC-DNS>:8000` and logged into my Splunk account web UI. To enable port 9997, I navigated to the following settings:

`Settings → Forwarding and receiving → Configure receiving → Enable port 9997`

![image](https://github.com/user-attachments/assets/449a4a81-75c9-4fc7-8039-199f79fb0206)
![image](https://github.com/user-attachments/assets/d275e688-fd58-4e29-8ca0-3af90b33b022)
![image](https://github.com/user-attachments/assets/44caaa1a-58bc-4ab7-9d45-a0c09177223d)
![image](https://github.com/user-attachments/assets/88fc44f1-9210-4d75-8f1e-285049e39b23)


#### Configuring the HTTP Event Collector (HEC) to Forward AWS Logs into Splunk

Next I am going to configure HEC so that my AWS monitoring services has the ability to forward their logs to my Splunk. HEC allows AWS to transmit log data via HTTPS/HTTP to Splunk using an authentication token. The token replaces the need to utilize my Splunk credentials for convenience and extra security.

HEC setup was fairly simple, In my Splunk web UI I navigated to the following settings:

`Settings → Data Inputs → HTTP Event Collector`

From here I created a `New Token` and added the following configurations for EACH service:
- Name: `AWS_Log_Collector`
- Source Types: `aws:cloudtrail`, `aws:guardduty`, `aws:cloudwatch:vpcflow`, and `aws:accessanalyzer`
- Enable Token (I saved the tokens for later use)
- Indexed is as `aws_logs` (for querying in Splunk)

![image](https://github.com/user-attachments/assets/4419965a-7665-4606-991e-1f180386b84c)
![image](https://github.com/user-attachments/assets/b1a072b2-9db8-4e45-8991-1c7cb8f05ccd)
![image](https://github.com/user-attachments/assets/5471bef2-0882-4b9f-bd07-e44f3b448672)
![image](https://github.com/user-attachments/assets/c9a1988f-f485-4897-a754-16c366b9c241)
![image](https://github.com/user-attachments/assets/0d5239a6-ad39-47c4-be8b-7d92d3a5c28c)

I repeated this step for all the AWS services (Cloudtrail, GuardDuty, CloudWatch, VPC Flow logs, IAM access analyzer)

Finally, I had to enable these tokens by navigating into the `Global Settings` and clicking `Enabled`. If these are not enabled, then Splunk will reject any logs that are sent to them.

![image](https://github.com/user-attachments/assets/b51d6112-67d0-45f3-8f93-968bd16549fe)
![image](https://github.com/user-attachments/assets/aec07908-fd1c-42e9-8dee-736e403252fe)

---

### Installing Add-ons for Parsing AWS & Windows Logs Correctly

These add-ons tell Splunk how to parse and recognize fields from AWS and Windows log formats. To do this, I will download the `.tgz` from Splunk and upload them in the following location within the Splunk web UI:

`Apps → Manage Apps → Install App from File`

![image](https://github.com/user-attachments/assets/68f36b84-b95c-40eb-a7a2-24b03a06050f)
![image](https://github.com/user-attachments/assets/a96000f1-f7c3-4756-89a9-7d16be541d20)
![image](https://github.com/user-attachments/assets/893e3ccd-2382-4ec6-a7c7-84e753bad55e)

I repeated this for both AWS and Microsoft Add Ons. I then confirmed they were successfully installed by reviewing them in the `APP` section of Splunk:

![image](https://github.com/user-attachments/assets/8c9797c6-a6d0-4b8c-9c2a-1ab261ec53e2)


---
---
## ⭐ Step 2: Installing Splunk Universal Forwarder into All My EC2s

Next, I need to download the Splunk Universal Forwarder (UF) and install it into my EC2 machines so they can be configured to forward logs to my Splunk server. I will be using Powershell scripts to initiated the download and installation, so I first began by grabbing the "wget" links from Splunk:

![image](https://github.com/user-attachments/assets/2545e5dc-0ab5-44f8-9e8a-ef76d543c61c)

Then I opened up my EC2 outbound rules to allow connections to HTTPS access so that the UF could be downloaded:

![image](https://github.com/user-attachments/assets/120cf9fa-c0d6-4643-88ec-28b2d00f0c0b)

### Installing and Configuring The UF in My EC2s with Powershell 

For each Windows EC2, I performed the following steps to install and configure UF:

#### ✅ Step 1:
To Download the UF:
`Invoke-WebRequest -Uri "https://download.splunk.com/products/universalforwarder/releases/9.4.2/windows/splunkforwarder-9.4.2-e9664af3d956-windows-x64.msi" -OutFile "C:\splunkforwarder.msi"`

**NOTE:** My AD server needs to be running for my windows workstations to have internet connections since they use it my domain controller for DNS name resolutions

![image](https://github.com/user-attachments/assets/cdcba81e-e909-4d35-86f3-bc5f217b768b)


---
#### ✅ Step 2:
To Install the UF and set credentials for the local UF
`Start-Process msiexec.exe -ArgumentList '/i C:\splunkforwarder.msi AGREETOLICENSE=Yes SPLUNKUSERNAME=NewAdmin SPLUNKPASSWORD=NewPass /quiet' -Wait`

![image](https://github.com/user-attachments/assets/6b6cea21-81c4-4870-a6ab-14d2e586a6a6)


---
#### ✅ Step 3: 
To Launch The UF and enable start on boot:
`& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" enable boot-start`

`& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" start`

Note: in this case, the UF was automatically configured to start on boot, so I did not have to:

![image](https://github.com/user-attachments/assets/0eb845c3-59df-4f82-aba5-d1910e87ed7a)


![image](https://github.com/user-attachments/assets/6caeb5e0-ae02-4591-9489-687650e00c9b)


---
#### ✅ Step 4:
To Configure UF to send Logs to My Splunk Server's private IP via port 9997 (local UF credentials). 

**NOTE:** I had to create an outbound security group rule to allow my EC2s to send traffic to my Splunk server via port 9997

`& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" add forward-server 10.0.2.70:9997 -auth user:pass`

![image](https://github.com/user-attachments/assets/736bfff6-ba13-4360-a960-ad8bc22b4353)


---
#### ✅ Step 5:
Then I will manually create an `inputs.conf` file and add it to the following file path for each EC2. The `inputs.conf` file tells the UF **WHAT** data to send (e.g., Sysmon logs, Windows event logs, specific files, etc.).:

`C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`

This `inputs.conf` file (.txt file) will contain the following configurations:

`[default]
host = Windows-Bastion-EC2`

`[WinEventLog://Security]
disabled = 0`

`[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
sourcetype = XmlWinEventLog:Sysmon` 

`[WinEventLog://System]
disabled = 0`

`[WinEventLog://Application]
disabled = 0`

![image](https://github.com/user-attachments/assets/1d8cf019-1369-4971-806f-0cb561f9d81f)


These configurations enables (i.e. `disabled = 0`) the UF to collect logs from AD Windows Security Events, and Sysmon (`WinEventLog` `sourcetype = XmlWinEventLog:Sysmon`). Once I do this, UF should now be able to forward these logs to my Splunk server!




---
Next, I will create an `output.conf` file in the same directory. Your `outputs.conf` tells the UF **WHERE** to send data to, in this case it will be my Splunk server:

```
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 10.0.2.70:9997

[tcpout-server://10.0.2.70:9997]
```
![image](https://github.com/user-attachments/assets/57479cd6-ee74-4609-89ca-50aed1127851)


Once I add these .conf files, I will then restart the Splunk UF using this PS command:

`& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" restart`


---
#### ✅ Step 6:

Now that I have set-up UF, next I will confirm that logs are indeed being forwarded to my Splunk server. We can start the verification by running the following command in my Windows EC2s to see if my UFs are actually connected:

 `& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" list forward-server`

**NOTE:** I had to adjust my EC2 Outbound rules to communicate with my Splunk server via port 9997 to ensure logs are being sent.

![image](https://github.com/user-attachments/assets/15793fac-2e26-401a-886d-52d0e82bdbc6)

In the above screenshot, you can see that there is active forwarding to my Splunk Server (10.0.2.70)

Next, I went into the Splunk web GUI and looked up my EC2 directly to see if logs where actually being collected in splunk using the following SPL query

`index=* host=<Windows-EC2-Hostname>`

![image](https://github.com/user-attachments/assets/259c6c31-1d68-452f-b51e-da33645981b0)

As you can see, we have logs being generated in Splunk!!


---
---   
## ⭐ Step 3: Configure AWS Services to Send Logs to Splunk via AWS Splunk Add On
For this final step, I configured all monitoring AWS services to forward their logs to Splunk utilizing the AWS Splunk Add On for AWS

#### ✅ Step 1: Configure my S3 bucket to Collect Logs, Enable Encryption and Enable Object Creation Notifications

In order for Splunk to collect any generated logs from AWS services, I first need to properly configure the S3 bucket that I created in Phase 5 so that it is ready. Splunk will pull logs via HEC from this S3 bucket.

I begin this by making sure that the S3 Bucket has encryption enabled and public access disabled:

![image](https://github.com/user-attachments/assets/2475e6d8-9dba-4133-8b35-2db6ab65b501)

![image](https://github.com/user-attachments/assets/e81306de-de5f-40eb-b731-a79a92759aa2)

I will also setup Event Notifications that can forwarded to my Splunk via HEC using a Lambda Function later on. This help will help detect any tampering with my S3 Bucket:

![image](https://github.com/user-attachments/assets/806e02e5-e92a-4d1f-849b-0b86c3002fc5)

#### ✅ Step 2: Create a IAM Role for My Splunk Server to Access this S3 Bucket and Logs

Next, I will need to create a new IAM role for my Splunk EC2 server so that it has the proper permissions to read/access the following:
- AWS logs S3 Bucket
- Cloudwatch logs
- Guarduty logs
- Cloud trail logs
- IAM Accesss Analyzer logs
- VPC flow logs

I will do this by creating the following JSON file that outlines these permissions:

```json
{
	"Statement": [
		{
			"Action": [
				"s3:GetObject",
				"s3:ListBucket",
				"cloudwatch:Get*",
				"cloudwatch:List*",
				"logs:Get*",
				"logs:Describe*",
				"guardduty:Get*",
				"iam:GetAccessAnalyzer*",
				"cloudtrail:Get*",
				"ec2:Describe*",
				"sqs:GetQueueAttributes",
				"sqs:ReceiveMessage",
				"sqs:DeleteMessage",
				"sqs:ListQueues"
			],
			"Effect": "Allow",
			"Resource": "*"
		}
	],
	"Version": "2012-10-17"
}
```
NOTE: Had to add SQS S3 permissions later because Splunk did not have the necessary permissions to use `listques` and efficiently pull logs from AWS.

Then, I went ahead and uploaded the JSOn file into AWS IAM role for Splunk:

![image](https://github.com/user-attachments/assets/b91fecc1-7817-4d06-8239-c37b4b0c7077)

Once created, I then added this IAM role to my Splunk EC2 Instance:

![image](https://github.com/user-attachments/assets/889109f7-0788-4648-bbca-7777c96fc9fc)

---
#### ✅ Step 3: Configuring AWS SQS and SNS Service for Efficient S3 CloudTrail Log Forwarding to Splunk

Next I need to setup and configure the SQS service so that Splunk can efficiently pull CloudTrail logs from my S3 bucket. I start by looking up the AWS `SQS` service and create a new "queue"

![image](https://github.com/user-attachments/assets/4d7978ec-bb71-44a8-a187-6ca57f86e68b)

I am going to do a `standard` setup and keep all the default settings in place for our purposes:

![image](https://github.com/user-attachments/assets/18c17ee8-3d4d-43f3-83b6-ff475c06951a)

Next, I need to update the IAM privileges of this SQS so that it has the necessary permissions to send logs from my S3 bucket. I will use the following JSON file:

![image](https://github.com/user-attachments/assets/48fcec45-ac5c-4f9b-a645-762814b8fa66)

Then I need to go into my cloudtrail S3 bucket and create a new event notification that will allow my S3 bucket to send notifications to my SQS service, which will then forward these notifications to my Splunk. This will allow my S3 bucket to push new CloudTrail logs to my Splunk.

![image](https://github.com/user-attachments/assets/35b214f5-e7c0-4f56-8b31-964aa18ec316)

![image](https://github.com/user-attachments/assets/f6815263-65ee-4285-9232-4031eecfff6e)

![image](https://github.com/user-attachments/assets/fab004a4-d4fb-4886-9aaf-60f3e96b91f9)

![image](https://github.com/user-attachments/assets/fe23ec7b-435a-4827-8517-9a6899a7463c)

The last step in configuring CloudTrail log forwarding to my Splunk involves configuring the AWS Splunk Add On to receive these logs from my SQS service. To do this I have to navigate to my Splunk GUI → `Settings` → `Data Inputs` → `AWS Add-on` → `CloudTrail` > `New Input` and use the following settings to complete the configuration:

![image](https://github.com/user-attachments/assets/b5824693-b421-4796-885f-d329ee726df5)
![image](https://github.com/user-attachments/assets/18c3e3ce-1f74-4ca5-80f9-dbadf4778c31)
![image](https://github.com/user-attachments/assets/79c59715-e510-4206-9927-06febda651b3)

Once I have the SQS set up, I have to set up SNS (Simple Notification Service). The SNS service is what actually sends notifications to SQS so that it knows that new logs have been recorded and triggers them to be forwarded to Splunk. To do so, I had to go to the AWS Console → `SNS` → `Topics` → `create new topic`. Then I configured the new topic in the following way:

![image](https://github.com/user-attachments/assets/9ee80bdc-4543-4205-bd77-5e18eb4dc65a)

![image](https://github.com/user-attachments/assets/4b44a2de-4c6c-42d2-80d0-54971ec250b4)

Then I need to subscribe this "topic" to my SQS queue, finalizing the connection between SQS and SNS.

![image](https://github.com/user-attachments/assets/f74596a1-19b8-46f5-9bbd-3646fb637c39)

![image](https://github.com/user-attachments/assets/b98291e0-466e-4e32-8375-751182f78e05)

The last step is to go to my `splunktrail` (CloudTrail) and  . With this step complete, whenever a new log file is delivered to your S3 bucket, the SNS topic sends a notification → which goes to your SQS queue → which Splunk reads!


![image](https://github.com/user-attachments/assets/b64851dc-1c46-4576-ad9e-c77d0f1ab4b5)

![image](https://github.com/user-attachments/assets/4b02bc9b-0544-43a5-8926-607ad05343f5)


Once set up, you can see that CloudTrail Logs are being generated in Splunk!:


![image](https://github.com/user-attachments/assets/73cfef4e-972f-4279-9f25-b4075d11cabb)

Now that we have EventBridge sending logs to my Lambda Function, which will then forward those logs to my Splunk, LETS TEST IT! I triggered a sample event in GuardDuty, then I check my Splunk to see if it ingested the logs:

---
#### ✅ **Step 4: Configuring GuardDuty to send Logs to Splunk Via GuardDuty ➝ EventBridge ➝ Lambda ➝ Splunk HEC**

To begin, I have to log into my Spolunk web GUI and prep a new HEC token (`GuardDuty-HEC`) to allow logs from GuardDuty to be ingested by going to `Settings` → `Data Inputs` → `HTTP Event Collector` -> `add new token`:

![image](https://github.com/user-attachments/assets/2f8380be-f67a-45fc-92a8-90c2967600f4)
![image](https://github.com/user-attachments/assets/e7e75564-fb42-4471-9d4c-7f381f5cccf9)

Now that I have the HEC set up, next I need to create a IAM role for the Lambda function that will allow it to generate logs within CloudWatch for visibility. There will also be permissions that will allow the lambda function to communicate with the Splunk EC2 within my VPC:

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"logs:CreateLogGroup",
				"logs:CreateLogStream",
				"logs:PutLogEvents"
			],
			"Resource": "*"
		},
		{
			"Effect": "Allow",
			"Action": [
				"ec2:CreateNetworkInterface",
				"ec2:DescribeNetworkInterfaces",
				"ec2:DeleteNetworkInterface"
			],
			"Resource": "*"
		}
	]
}

```
NOTE: I had to add this lambda function to my VPC, the subnets and included my Splunk EC@ security group settings into it.

Once I have setup this policy and assigned it to a new Lambda Role, the next step is to actually create the Lambda function with a simple python script:

```py

import json
import urllib3

http = urllib3.PoolManager()

SPLUNK_HEC_URL = 'https://MY SPLUNK IP:8088/services/collector'
SPLUNK_TOKEN = 'MY GUARDDUTY HEC TOKEN'

def lambda_handler(event, context):
    for record in event['Records']:
        payload = {
            "event": record,
            "sourcetype": "_json"
        }
        encoded_data = json.dumps(payload).encode('utf-8')
        response = http.request(
            'POST',
            SPLUNK_HEC_URL,
            body=encoded_data,
            headers={
                'Authorization': f'Splunk {SPLUNK_TOKEN}',
                'Content-Type': 'application/json'
            }
        )
        print(f"Sent to Splunk, status code: {response.status}")

```

![image](https://github.com/user-attachments/assets/32377f19-73cd-4dab-b148-f83bc6e355b1)

![image](https://github.com/user-attachments/assets/015b6651-c1d1-46cf-b3e0-0f373010e477)

![image](https://github.com/user-attachments/assets/0ae2731d-8b18-4a4d-87b9-6cdb29061c0a)


Next, I will make an EventBridge rule that will send GuardDuty Logs to my Lambda function by default:

![image](https://github.com/user-attachments/assets/8147f396-accd-416b-89a8-dd40f0c9fd81)
![image](https://github.com/user-attachments/assets/89c10f16-1292-4cbe-b161-e77631df83dc)
![image](https://github.com/user-attachments/assets/70e862c7-248c-4238-9bd0-a0db6dc747e4)









---
#### ✅ Step sdfsdfdsfsdff Configuring the Splunk Add-on To Ingest Logs from My AWS Services

Now that we have configured the necessary IAM permissions for my Splunk EC2, the next step is to log into my Splunk UI and configure the Splunk AWS Add On (that we [installed earlier](#installing-add-ons-for-parsing-aws--windows-logs-correctly)) so that we can configure log ingestion from our AWS Services.

To begin, I go to the `Splunk AWS Add On` app in my Splunk GUI, then to `configuration` and edit my AWS account to add in my Splunks IAM User's secret access key (found in IAM Console -> Users -> Security Credentials)

![image](https://github.com/user-attachments/assets/6c389911-05d0-44d3-add4-20ca84458df5)
![image](https://github.com/user-attachments/assets/b2fac70d-efad-4013-bf76-67743f7d8c35)
![image](https://github.com/user-attachments/assets/e475b6a4-cbff-472f-8a91-9abe7f8cd073)
![image](https://github.com/user-attachments/assets/1b1df088-7007-45fd-a2ee-4835cc580384)

AND as a result, we know have GuardDuty logs being generated in my Splunk!!!!:





Next I begin with adding a `new input` for CloudTrail Logs by using the following configurations:









---
--



## Now that I have set up my monitoring and detection services/tools, we are ready for [Phase 6 - Splunk Log Ingestion Setup](https://github.com/ChrisHerrera90/Complete-AWS-Cloud-Security-Architecture-Design-and-Splunk-Detection-Against-Simlulated-Attacks/blob/main/Phase%206%20-%20Splunk%20Log%20Ingestion%20Setup/Phase%206%20README.md)
