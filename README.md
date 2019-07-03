# Overview
A serverless system that acts as a custom bridge to connect components from Sage Intacct with Salesforce APTO.

# Contents

[Overview](#Overview)  
[Getting Started Guide](#Getting-Started-Guide)  
[How to Run and Deploy](#How-to-Run-and-Deploy)  
[Troubleshooting](#Troubleshooting)  
[Design](#Design)  
[Implementation](#Implementation)  
[Logic - How does the Automation Work?](#Logic)  
[Testing](#Testing)  
[Next Steps](#Next-Steps)  
[Conclusion](#Conclusion)

# Getting Started Guide

## Intacct SmartEvent

To trigger a webhook event from Sage Intacct, A smartevent has to be configured. To setup a smart event, follow these steps:

1. Go to `Platform services > Smart Events > list` and click on `Add` button to add a new event

2. Select the Object which has the property that triggers the events. For this example, I am choosing `AP Bill` object. Click `Next`

3. On the events page, choose `Action` to be `http post` as we need to send POST requests to an external webhook.

4. Select the `Event` that should trigger the webhook and move it to the rightside column. Let's choose `Add` event here.
(Conditions are optional and are used to generate a trigger, only if certain system conditions are met. For example, trigger http post event, only if bill amount is greater than $2000
For this example we are skipping away condions, but here is the reference: https://developer.intacct.com/customization-services/smart-event-walkthrough/#conditions-on-the-smart-event
) 

Click `Next`

5. Refer to the `How to Run and Deploy` section below, and deploy the bridge app. It should deploy 2 endpoints, one of which is the webhook endpoint which responds to Intacct events.
For example `https://ye6l7cv9ng.execute-api.us-east-1.amazonaws.com/dev/intacct-hook`

Copy this URL and paste in the Intacct URL field.

6. Click on the dropdown box and select method to be `POST`

7. Arguments refer to the data that are to be sent to the Webhook. To add relevant fields, click on the `Arguments` text box and then click on `Field Lookup` button.

A new window should pop up, find and select the fields that you need to send. For example
a. `Bill Number`
b. `Total transaction amount due`
c. `Total transaction amount paid`

Click `OK`. You should see Fields adding in the text box. 

```text
{!APBILL.RECORDID!} {!APBILL.TRX_TOTALDUE!} {!APBILL.TRX_TOTALPAID!}
```

8. We need to format these fields, so that they are properly encoding (Form xml type encoding) and therefore recieved properly by the webhook. 
To format, make sure each field has a field name that is sent and also make sure there is only one item per line. Example:

```text
BillNumber={!APBILL.RECORDID!}
TotalAmountDue={!APBILL.TRX_TOTALDUE!}
TotalAmountPaid={!APBILL.TRX_TOTALPAID!}
```
click `Next`

9. We need to create an SmartLinkID for this event, For example `AP_PAY`. Add description if required, and make sure status is `active`.

Click `Done`

Now your event should be created and added to smart events list. 
You can check if it works as expected by manually clicking on `Pay` and completing a payment form for AP Bills.

If you check the CloudWatch event of the webhook Lambda, you can see a request such as this:

```json
{'body': {'TotalAmountDue': '1000', 'BillNumber': 'HT-10760'}, 'method': 'POST', 'principalId': '', 'stage': 'dev', 'cognitoPoolClaims': {'sub': ''}, 'enhancedAuthContext': {}, 'headers': {'Accept': '*/*', 'CloudFront-Forwarded-Proto': 'https', 'CloudFront-Is-Desktop-Viewer': 'true', 'CloudFront-Is-Mobile-Viewer': 'false', 'CloudFront-Is-SmartTV-Viewer': 'false', 'CloudFront-Is-Tablet-Viewer': 'false', 'CloudFront-Viewer-Country': 'US', 'Content-Type': 'application/x-www-form-urlencoded', 'Host': 'ye6l7cv9ng.execute-api.us-east-1.amazonaws.com', 'User-Agent': 'Intacct Standard HTTP Agent', 'Via': '1.1 e2af8a85927835558866752f53562ecd.cloudfront.net (CloudFront)', 'X-Amz-Cf-Id': 'RVyPQaFLBDV2p0tjhC_HAa3suQnbfvmQ3ulheFec-HQtTPQ4mTER4A==', 'X-Amzn-Trace-Id': 'Root=1-5c93bc82-0b0292bb97be8b903add5374', 'X-Forwarded-For': '4.7.16.188, 54.239.134.46', 'X-Forwarded-Port': '443', 'X-Forwarded-Proto': 'https'}, 'query': {}, 'path': {}, 'identity': {'cognitoIdentityPoolId': '', 'accountId': '', 'cognitoIdentityId': '', 'caller': '', 'sourceIp': '4.7.16.188', 'accessKey': '', 'cognitoAuthenticationType': '', 'cognitoAuthenticationProvider': '', 'userArn': '', 'userAgent': 'Intacct Standard HTTP Agent', 'user': ''}, 'stageVariables': {}}
```
On which you can see that the `body` key-value pair has the sent data.

10. Download the smart event DEF and save it in the repo in `/smart_event_defs` for reference.

## Finding APTO Field names to write actions using APTO API

1. Click on `Setup` and then `Object Manager`

2. Use `Quick Find` to Search for the object we want to perform an action on, say `Invoice`

3. Note down the API Name. This is used to form a query SOQL statement for Invoices. Example query used in a lambda function:

`SELECT Id, Transaction_Number__c FROM McLabs2__Invoice__c WHERE Transaction_Number__c = 'HT-10762'`

The Transaction number string `'HT-10762'` will be dynamically inserted based on the function call. It is a variable.

4. Click on the `Invoice` object and click on `Fields & Relationships` tab. You will find all the field labels as in a invoice seen by the user.
For programmatic read or write, we don't need the label, rather we need `Field Name`. For example, if we wish to read and update a field labelled `Amount Paid`,
then we need to use the field name `McLabs2__Amount_Paid__c` in our code.

Using library `simple_salesforce` a sample code such as one below can update a particular invoice object.

```python
# import and instantiate sf object using simple_salesforce

invoices = sf.query("SELECT Id, Transaction_Number__c FROM McLabs2__Invoice__c WHERE Transaction_Number__c = 'HT-10762'")
Invoice_objs = SFType('McLabs2__Invoice__c', sf.session_id, sf.sf_instance)
amount = 1000.0
Invoice_objs.update(record['Id'], data={'McLabs2__Amount_Paid__c': amount})
```

# How to Run and Deploy

To deploy the serverless app to your AWS Account, you need to first create a config file and add required credentials.

For example, to deploy to stage `dev`, we would have to create `config.dev.json` with contents like below:

```json
{
  "SALESFORCE_INSTANCE_URL": "https://aptotude-5138.cloudforce.com/a0H6100000gHRtNEAW",
  "SALESFORCE_USERNAME": "{SF_USERNAME}",
  "SALESFORCE_PASSWORD": "{SF_PASSWORD}",
  "INTACCT_SENDER_ID": "{SAGE_WEBSERVICES_SENDER_ID}",
  "INTACCT_SENDER_PASSWORD": "{SAGE_WEBSERVICES_SENDER_PASSWORD}",
  "INTACCT_USER_ID": "{SAGE_USER_ID}",
  "INTACCT_COMPANY_ID": "{SAGE_COMPANY_ID}",
  "INTACCT_USER_PASSWORD": "{SAGE_USER_PASSWORD}",
  "INTACCT_ENDPOINT_URL": "https://api.intacct.com/ia/xml/xmlgw.phtml"
}
```
`{SAGE_WEBSERVICES_SENDER_ID}` has a developer license attached to it. Ask your admin for details. Please
make sure that the `sender_id` is added to `Company > Security > Web Services authorizations` list. A single
webservices sender can be authorized by multiple companies.

Ref: [https://developer.intacct.com/blog/2018/05/2018-Release-2.html#web-services-enhancements](https://developer.intacct.com/blog/2018/05/2018-Release-2.html#web-services-enhancements)


```sh
cd api_gw_resources
sls deploy --stage dev
cd ..
sls deploy --stage dev
```
For production:

```sh
cd api_gw_resources
AWS_PROFILE=profile_name sls deploy --stage prod
cd ..
AWS_PROFILE=profile_name sls deploy --stage prod
```

`api_gw_resources` is a serverless service, that deploys API_GATEWAY resources. We need to deploy them only once,
so for any updates, we can directly run the `sls deploy` command.

Check the trouble shooting section if the deploy fails.

## Updating System credentials without code update

Sometimes we have to update secrets like `INTACCT_LOGIN_PASSWORD` for the webservices account and so on. These
secrets are maintained as lambda environment variables and can be modified in AWS console without a code redeploy.

To change environment variables:
 
1. you should head over to [AWS Console](https://naipartners.signin.aws.amazon.com/console)
and login with suitable credentials.

2. Search for `lambda` and go to `lambda > Functions > prod-intacct-webhook` (or apto webhook, which ever you wish to change)

![lambda_console](./docs/images/lambda_console.png)

3. Make sure you are in `Configuration` tab and scroll down to `Environment Variables`.

![env_vars](./docs/images/env_vars.png)

4. Update any variables and click `Save` button on top.

Any consequent calls would take up the new set of variables.
  
# Troubleshooting

## Serverless deploy fails

If the `sls deploy` fails, the most common reason is that the AWS CLI is not configured.

1. Go to AWS Console > IAM > Create new Admin user
Setup the Admin user for only programmatic access, make sure admin policy is attached
Download the credentials

2. Download and install `aws-cli` for your OS

3. Run `aws configure` from a shell prompt and give the above downloaded creds to configure the aws cli.

4. Run `deploy` command as specified in the above section again

## Cannot find Smart Events under Platform Services

**Switch to Action UI** Smart events are shown under platform services only under action UI

## Leave site? Changes may not be saved

Sometimes while triggering an event, like say clikcing on bill's `Pay` from `AP > Bills`, we get a pop up component
where we fill details and then click on something like `Save` that processes the event.

At this point, we might get an alert message asking if we are leaving the site.

This is how Intacct frontend works. Not sure if it is a bug, but feel free to click on `Leave`. The smartevent would have been triggered as we have verified on server logs.

## Not getting configured smartevent arguments in the server request

Sometimes, when we configure a POST request with multiple arguments, like say for `AR > Invoice > Apply Payment`,
Make sure the event `action` is right. For example `Apply Payment` would trigger `AR Payment` not `Invoice` object.

## Deployment IAM User Policy

Serverless framework is designed to be deployed by admin users and the docs do not say much about what policies should be attached to the deploying user.

Based on issue: [https://github.com/serverless/serverless/issues/1439](https://github.com/serverless/serverless/issues/1439)

This is a valid (pretty permissive, but not admin level permissive) user policy. [https://github.com/serverless/serverless/issues/1439#issuecomment-482474780](https://github.com/serverless/serverless/issues/1439#issuecomment-482474780)

To add the above permissions to a user, login into AWS console as an admin.

1. Go to `IAM`
2. Create a new user or select an existing user under `Users`. For example let the user name be `prash-contractor`
3. Click on `Add Inline Policy`
4. Click on `JSON` tab, you should see a JSON editor
5. Copy and paste the following policy JSON into the editor

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "NonDestructiveCoreReaderActionsUsedThroughoutServerless",
            "Effect": "Allow",
            "Action": [
                "apigateway:*",
                "cloudformation:CreateStack",
                "cloudformation:Describe*",
                "cloudformation:ValidateTemplate",
                "cloudformation:UpdateStack",
                "cloudformation:List*",
                "iam:GetRole",
                "iam:CreateRole",
                "iam:PutRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:DeleteRole",
                "iam:PassRole",
                "lambda:UpdateFunctionCode",
                "lambda:Get*",
                "lambda:CreateFunction",
                "lambda:InvokeFunction",
                "lambda:UpdateFunctionConfiguration",
                "lambda:PublishVersion",
                "lambda:DeleteFunction",
                "lambda:List*",
                "lambda:AddPermission",
                "s3:CreateBucket",
                "s3:DeleteObject",
                "s3:GetObject",
                "s3:GetBucketLocation",
                "s3:ListBucket",
                "s3:PutObject",
                "s3:DeleteBucket",
                "s3:GetEncryptionConfiguration",
                "s3:PutEncryptionConfiguration",
                "logs:Describe*",
                "logs:CreateLogGroup",
                "logs:DeleteLogGroup",
                "events:PutRule",
                "events:DescribeRule",
                "events:PutTargets"
            ],
            "Resource": "*"
        }
    ]
}
```
6. Click on Review policy
7. On the next page give the policy a name `intacct_bridge_production`
8. Click on `Create Policy` in the bottom of the screen


# Design

Here is the high level sofware design

![Architecture](./design.jpg)

# Implementation

## Intacct to APTO Flow

On Intacct the events seem to be `Fire and Forget`, which means that intacct doesn't wait for responses after events were triggered.

Similarly, the server is being made to be `Fire and Forget` what it means here is that, when the lambda functions (server) is triggered by an event (say AP Bill Pay), it executes the business logic.
Say we want to update an APTO invoice field. If it can do that based on the logic it will execute, otherwise it gracefully exits. For example, in one of the problems solved, 
we find APTO invoice by transaction number matched with Intacct bill number from event triggered. If and only if there is one or more positive matches, they will get updated.

While we will maintain logs of events triggered on AWS CloudWatch, the `fire and forget` here means that the design is unidirectional. 

> A more complex system could change Intacct objects also, but that has to be carefully crafted to avoid circular dependencies and other issues. We are not doing anything like that at the moment.

## APTO to Intacct Flow

APEX Triggers is a code that is run `before` or `after` an APTO operation. `after` events are read-only.

# Logic

## Automation of Invoice Payment - What it does and doesn't do?

1. When an invoice is paid in Sage, the `invoice number` is looked up against Comps in Apto, whose `Transaction number` matches with the Sage `Invoice Number`.

2. **Bracket Notation**: The Matching Comp could have multiple invoices under it. The program picks a primary matching invoice based on the Sage Invoice number.
The matching Apto invoice is found based upon numbered suffix in the Sage invoice number.

For example, if the Sage invoice number is `ABC-123`, this would match against the 1st invoice under a Comp with the same Transaction number. An Sage invoice with invoice number
`ABC-123 (2)` would match with the second invoice under the Comp and so on.

3. If the `AmountPaid` in Sage equals the `AmountDue` in the matching invoice in Apto and if the status of the Apto Invoice is `unpaid`, then the `AmountPaid` field in Apto is updated.

4. If the `AmountPaid` in Sage does not equal the `AmountDue` in the primary matching invoice in Apto, then subsequent invoices under the Comp are taken, *neglecting bracket notation* and: 
    
    A. If there is any unpaid invoice under the Comp with `AmountPaid` equal to `AmountDue`, that invoice is updated.
    
    B. If there is no other invoices under the Comp with `AmountPaid` equal to `AmountDue`, then the program starts looks for **Overpaid** condition.
    When a Sage invoice is overpaid, then the program starts clubbing two or more Apto invoices together such that their sum of AmountDue's would equal the AmountPaid.
    If such a set of Apto invoices are found, then all of them are updated with Corresponding Amounts in the `AmountPaid` fields.
    
    C. If both of the above conditions fails, then the primary matching invoice is updated. Since `AmountPaid` is not equal to `AmountDue`, it will lead to `Balance` field in Apto invoice becoming non-zero.
    **Please note: In this case, if the primary invoice is already PAID, then the `AmountPaid` field is OVER-WRITTEN with the new value**

 

# Testing

# Next Steps

# Conclusion


If there are any questions or problems in the future, please contact *Prashanth* at *neotheicebird@gmail.com*
