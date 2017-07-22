---
title:  "AWS Lambda configuration"
date: 2017-07-21T07:35:01+02:00
draft: true
tags: ["aws", "lambda"]
---

# Adding permission for Lambda function
```bash
aws lambda add-permission \
  --function-name "asg_route53_updater"  \
  --action lambda:invokeFunction \
  --principal sns.amazonaws.com \
  --source-arn "arn:aws:sns:us-west-2:240020657974:asg_route_notifications" \
  --statement-id $(uuidgen)
```

# Subscribe the AWS Lambda function to the SNS Topic
```bash
aws sns subscribe \
  --topic-arn "arn:aws:sns:us-west-2:240020657974:asg_route_notifications" \
  --protocol lambda \
  --notification-endpoint "arn:aws:lambda:us-west-2:240020657974:function:asg_route53_updater"
```
