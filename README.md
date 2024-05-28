# AWS-Reasonable-Account-Defaults

I had a friend ask me recently what his son/daughter should do to their AWS account if they wanted to stay on the free tier.
I explained some basics of free tier, but it got me thinking about “rational” AWS account defaults.

Currently, the CloudFormation template creates Cost Anomaly Detection Monitors and Subscriptions. And it creates 2 Budgets. One budget is static based on a given input at the time of the CloudFormation Template stack creation and the other is a dynamic (Auto Adjusting) budget based on the previous 6 months of usage.

## Invocation

- [Download the CloudFormation Template](https://github.com/dubrowin/AWS-Reasonable-Account-Defaults/blob/main/account-default-project-OVERALL.yaml)
- Go to CloudFormation in the AWS Console
- Click Create Stack
  - Select "Choose an existing template"
  - Template Source, select "Upload a template file"
  - click "choose file" and select the downloaded CloudFormation Template file
  - Click Next
- Stack Name
  - Give a name to your CloudFormation Stack. "Reasonable Account Defaults"
- Parameters
  - EmailParam - this is the email address to receive notifications to. It's important that this is an accurate address or the cost notifications will not be received
  - AlertThresholdParam - this is the alert threshold that will be set in the Cost Anomaly Detector and in the Fixed Budget.
    - For a Free Tier or Student account, the recommendation is the minimum of $0.01.
    - For an account with any usage, some other value should be chosen that coincides with how much spend you expect to incur
  - Click Next
- While Tags are optional, they are super useful for identifying resources. I suggest at least adding
  - key = project
  - value = AWS Reasonable Account Defaults
- The remaining items stay as the default
- Click Next
- Review the page
- Click Submit
- The creation process will start
  - The process should not take more than 10 seconds
- You're done, you have taken your first step toward avoiding Cost Surprises

## Next Steps
- ~~Have an option for non-student/non-free-tier accounts that splits out the fixed budget and cost anomaly detector thresholds~~ - Done
- Implement a Lambda that pushes an S3 lifecycle policy to all new buckets so that Multipart Upload (MPU) fragments are removed after 7 days (configurable)
  - push Event Bridge rules to all active regions so that a bucket created in any active region will trigger the above Lambda
- Implement a Lambda that sets new CloudWatch (CW) Log Group retention to 30 days (configurable) so that any new Log Group created will not fill up forever
  - push Event Bridge rules to all active regions so that a CW Log Group created in any active region will trigger the above Lambda
- Implement a Lambda to run weekly (configurable) that will push the previous 2 Event Bridge rules to any newly activated region
