# Serverless Contact Form on AWS

A fully serverless contact form backend built with API Gateway, Lambda, and SES.
Integrated into a static portfolio site hosted on S3 + CloudFront.

🌐 **Live site:** https://d34h7plwugd541.cloudfront.net

---

## Architecture

![Architecture](./screenshots/architecture-diagram.png)


---

## Services Used

- **API Gateway** — HTTP API that exposes a public `POST /contact` endpoint
- **Lambda** — Python function that receives the request and calls SES
- **SES** — Sends the email to the verified recipient address
- **IAM** — Role with `AmazonSESFullAccess` policy attached to the Lambda function
- **S3 + CloudFront** — Hosts the frontend that calls the API (see static-website project)

---

## Implementation

1. Verified sender email identity in SES (sandbox mode)
![SES verification](./screenshots/ses-verification.png)
2. Created Lambda function (`contactFormHandler`) in Python 3.12
![Lambda function](./screenshots/lambda-function.png)
3. Attached `AmazonSESFullAccess` IAM policy to the Lambda execution role
![IAM Policy](./screenshots/iam-policy-attached.png)
4. Created HTTP API in API Gateway with `POST /contact` route pointing to Lambda
![API Gateway POST](./screenshots/cors-configured-correctly.png)
5. Configured CORS on API Gateway (`Allow-Origin: *`, `Allow-Headers: content-type`, `Allow-Methods: POST`)
![Configured CORS](./screenshots/cors-configured-correctly.png)
7. Wired up the form in `index.html` to fetch the API endpoint on submit
![Form Success](./screenshots/cors-configured-correctly.png)

---

## Lambda Function

```python
import json
import boto3

ses = boto3.client('ses', region_name='us-east-1')

def lambda_handler(event, context):
    body = json.loads(event['body'])
    name    = body.get('name', '')
    email   = body.get('email', '')
    message = body.get('message', '')

    ses.send_email(
        Source='beardeddominican@gmail.com',
        Destination={'ToAddresses': ['beardeddominican@gmail.com']},
        Message={
            'Subject': {'Data': f'Portfolio Contact: {name}'},
            'Body': {'Text': {'Data': f'Name: {name}\nEmail: {email}\n\nMessage:\n{message}'}}
        }
    )

    return {
        'statusCode': 200,
        'headers': {'Access-Control-Allow-Origin': '*'},
        'body': json.dumps({'message': 'Email sent!'})
    }
```

---

## Challenges & Lessons Learned

**CORS error on form submit** — The browser was blocking the API response because
API Gateway wasn't returning the correct CORS headers. Fixed by configuring CORS
on the API Gateway HTTP API (`Allow-Origin: *`, `Allow-Headers: content-type`).
Note: when CORS is configured at the API Gateway level, it overrides any CORS
headers returned by Lambda — so the headers in the Lambda response are redundant
but kept for clarity.

**Distribution ID vs domain name** — When automating deployments via AWS CLI,
CloudFront invalidations require the Distribution ID (e.g. `EWQ0L2AO5RDW9`),
not the CloudFront domain name. Using the domain name returns an error.

**SES sandbox mode** — By default SES only sends to verified email addresses.
For a production use case, you would request SES production access to send
to any recipient.

---

## Deployment

Updates to the frontend are deployed via AWS CloudShell with a single command:

```bash
aws s3 cp index.html s3://gt-portfolio-2026/index.html && \
aws cloudfront create-invalidation --distribution-id EWQ0L2AO5RDW9 --paths "/*"
```

---

## AWS Region

All services deployed in `us-east-1` (N. Virginia).
