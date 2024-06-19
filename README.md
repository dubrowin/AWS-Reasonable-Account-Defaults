# AWS-Reasonable-Account-Defaults

I had a friend ask me recently what his son/daughter should do to their AWS account if they wanted to stay on the free tier.
I explained some basics of free tier, but it got me thinking about “rational” AWS account defaults. This project is the culmination of 2 re:Invent chalk talks on "Ways to Avoid Cost Surprises" and the work I've done with startups around cost optimization. My expectation is that these resources should be a good start toward being alerted if and when costs are accumulating in an AWS account.

## Where to Deploy
Most of the services used by this CloudFormation are deployed in all regions, but I have found that AWS Budgets and Amazon EventBridge Scheduler are not deployed in all regions. Therefore, you should choose a region that has both of these. You can check availability for [AWS Budgets](https://www.aws-services.info/budgets.html) and [Amazon EventBridge Scheduler](https://www.aws-services.info/scheduler.html).

Deploying in the older regions like us-east-1 (Virginia), us-east-2 (Ohio), us-west-2 (Oregon), or eu-west-1 (Ireland) should not be a problem.

## Invocation

- [Download the CloudFormation Template](https://github.com/dubrowin/AWS-Reasonable-Account-Defaults/blob/main/account-default-project-OVERALL.yaml)
- Go to CloudFormation in the AWS Console

![cloudformation-service](https://github.com/dubrowin/AWS-Reasonable-Account-Defaults/assets/19818673/1194ce86-a6b3-4829-a61f-f5ec82795a5a)

  
- Click Create Stack

![create-stack](https://github.com/dubrowin/AWS-Reasonable-Account-Defaults/assets/19818673/a627a88f-10a6-451d-a0e8-3adec1c61d9a)


  - Select "Choose an existing template"
  - Template Source, select "Upload a template file"
  - click "choose file" and select the downloaded CloudFormation Template file
  - Click Next

![choose-file](https://github.com/dubrowin/AWS-Reasonable-Account-Defaults/assets/19818673/6ea0cbf9-4fd0-412e-86c3-030eb50c872a)

- Stack Details
  - Stack Name
    - Give a name to your CloudFormation Stack. "Reasonable-Account-Defaults" (Note: Spaces are not allowed)

![stack-details1](https://github.com/dubrowin/AWS-Reasonable-Account-Defaults/assets/19818673/e33fa817-0a6a-46ef-bbd8-959830a86515)


  - Parameters
    - Email Address - this is the email address to receive notifications to. It's important that this is an accurate address or the cost notifications will not be received
    - Spend Thresholds - this is the alert threshold that will be set in the Cost Anomaly Detector (first parameter) and in the Fixed Budget (second parameter).
      - For a Free Tier or Student account, the recommendation is the minimum of $0.01.
      - For an account with any usage, some other value should be chosen that coincides with how much spend you expect to incur
    - Log Retention - How long to retain CloudWatch Log Group data.
    - EventBridge Rule Lambda - this is the Lambda that will look for newly enabled regions and push the EventBridge rules enabling automatic S3 Bucket MPU Lifecycle Policies and CloudWatch Log Group Retentions. Default is daily (Days and 1).
 - Click Next

![stack-details2](https://github.com/dubrowin/AWS-Reasonable-Account-Defaults/assets/19818673/7f8b59a4-ab94-473e-bf89-39bfbfa20260)


- While Tags are optional, they are super useful for identifying resources. NOTE: the tag key and tag value are case sensitive. I suggest at least adding
  - key = project
  - value = AWS Reasonable Account Defaults

 ![tags](https://github.com/dubrowin/AWS-Reasonable-Account-Defaults/assets/19818673/f2afc6e5-eb94-484a-ade1-eb085751bd0a)

  
- The remaining items stay as the default
- Click Next

  
- Review the page
- Approve the IAM Permissions
  - "I acknowledge that AWS CloudFormation might create IAM resources with custom names."
- Click Submit

![approval](https://github.com/dubrowin/AWS-Reasonable-Account-Defaults/assets/19818673/1ff5f113-9be4-49fb-a08b-6f814a2fd97e)

  
- The creation process will start
  - The process should generaly take under 5 min. During my testing, the deployment took around 2 min.
- You're done, you have taken your first step toward avoiding Cost Surprises

## What Gets Deployed
- AWS Budgets
  - Auto Adjusting Budget
  - Forecasted Monthly Budget with a Target of the FixedBudgetParam you set above
- Cost Anomaly Detector
  - Monitor gets deployed with an Alert Threshold of AlterThresholdParam you set above
- CloudTrail
  - For the Event Bridge triggering rules to work, a CloudTrail Trail must be active.
  - A Lambda is deployed and run which checks all enabled regions in the account and searches for active CloudTrails. If no CloudTrail is found, it will create one.
    - The first CloudTrail Trail is available for free, therefore the Lambda checks that it will not incur an expense
    - The Lambda also creates and S3 bucket to store the CloudTrail logs in. No lifecycle policy is set to delete the logs since these are a security feature. If one is wanted, the user can go and set a lifecycle policy on the bucket created to remove all objects after X time.
- CloudWatch Log Group Retention
  - An Event Bridge Rule is added to the default Event Bridge Bus that traps all CloudWatch Log Group Creations and triggers the Lambda.
  - The Lambda that is deployed takes the Log Group from the event information and sets the Retention to what retention period was established when the CloudFormation template was run. This Lambda only finds Log Groups as creation, if the retention is changed later or if the Log Group existed already, the Lambda will not modify them.
- S3 Multipart Upload (MPU) Retention
  - An Event Bridge Rule is added to the default Event Bride Bus that traps all S3 Bucket Creations and triggers the Lambda.
  - The Lambda that is deployed takes the S3 Bucket from the event information and creates a Lifecycle Policy that deletes MPU Fragments after 7 days.
- Multi-Region / New Region Activation
  - In order for CloudWatch Log Groups and S3 Buckets created in regions other than the primary deployment to trigger the Lambdas, Event Bridge rules are deployed to all enabled regions by a Lambda.
  - The Lambda goes through all enabled regions other than the primary deployed region and if the Event Bridge forwarding rules are found, does nothing. If rules are not found, it creates them. This Lambda is scheduled to run regularly (as defined in the parameters) so that it will catch any newly enabled regions.
   
## Next Steps
- ~~Have an option for non-student/non-free-tier accounts that splits out the fixed budget and cost anomaly detector thresholds~~ - Done
- ~~A Cloudtrail trail is required by the Lambdas below. So I have created a Lambda that runs once and checks if there is a Cloudtrail Trail enabled. If not, it will create it.~~ - Done
- ~~Implement a Lambda that pushes an S3 lifecycle policy to all new buckets so that Multipart Upload (MPU) fragments are removed after 7 days~~ (configurable) - Done
  - ~~push Event Bridge rules to all active regions so that a bucket created in any active region will trigger the above Lambda~~ - Done
- ~~Implement a Lambda that sets new CloudWatch (CW) Log Group retention to 30 days (configurable) so that any new Log Group created will not fill up forever~~ - Done
  - ~~push Event Bridge rules to all active regions so that a CW Log Group created in any active region will trigger the above Lambda~~ - Done
- ~~Implement a Lambda to run weekly (configurable) that will push the previous 2 Event Bridge rules to any newly activated region~~ - Done
- Testing ability
  - automate the testing and cleanup of the resources deployed
- IAM Security Hardening
 - go through all the IAM permissions created and ensure they are only allowing required permissions and are scope limited to the required resources.
- Resource Tagging
 - automatically add the CloudFormation Name and StackID to all created resources and all derivative resources
