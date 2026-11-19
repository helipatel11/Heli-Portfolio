---
title: "Cloud Resume Challenge: My Resume on S3, CloudFront and Lambda"
date: 2025-03-14
weight: 30
description: "An early AWS project: my resume as a static site on S3 and CloudFront, a Lambda + DynamoDB visitor counter, Terraform and a GitHub Actions deploy, and what I'd do differently now."
tags: ["aws", "s3", "cloudfront", "lambda", "dynamodb", "terraform", "github-actions", "earlier-work"]
---

*Built between January and March 2025, before [Plant Pal](/projects/plant-pal/) and my [LLMOps Support Assistant](/projects/llmops-support-assistant/). I've kept it here as an earlier project.*

## What it is

The Cloud Resume Challenge asks you to put your resume online as a website in the cloud, with a visitor counter backed by a database, infrastructure as code and an automated deploy. Mine is live at [resume.helipatel.com](https://resume.helipatel.com/).

![Cloud Resume Challenge architecture](/images/cloud-resume-architecture.png)

- **Hosting:** the site is plain HTML, CSS and JavaScript in a **private** S3 bucket, served through CloudFront. The bucket is not public. Its policy lets only my CloudFront distribution read from it, and the files are encrypted at rest (SSE-S3).
- **HTTPS:** CloudFront serves `resume.helipatel.com` with an ACM certificate, HTTPS only, TLS 1.2 minimum.
- **DNS:** Squarespace answers for my domain, and `resume.helipatel.com` is a CNAME pointing at the CloudFront distribution. I had also created a Route 53 hosted zone with an alias record for it, but the domain's nameservers pointed to Squarespace, so Route 53 was never asked. I deleted that zone in November 2026.
- **Visitor counter:** when the page loads, `index.js` calls a Lambda Function URL. A Python Lambda adds 1 to the count stored in DynamoDB and returns it, and the page shows "Views: N".
- **Deploys:** on every push to `main`, GitHub Actions logs into AWS with OIDC, so no AWS keys are stored in GitHub, and copies the `Website/` folder to the bucket.
- **Terraform:** manages the bucket, its encryption and policy, and the CloudFront distribution. The counter (the Lambda, its Function URL, its IAM role and the DynamoDB table) is not in Terraform.

## How it fits together

The most important design choice is that the bucket stays private. CloudFront signs its requests to S3 (Origin Access Control), and the bucket policy only accepts requests from that one distribution:

```json
{
  "Sid": "AllowCloudFrontServicePrincipal",
  "Effect": "Allow",
  "Principal": { "Service": "cloudfront.amazonaws.com" },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::<BUCKET_NAME>/*",
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
    }
  }
}
```

So the only way to the files is through CloudFront and HTTPS. Nobody can read the bucket directly.

The counter is a few lines of JavaScript. The page shows "Couldn't read views" by default, and replaces it when the call succeeds:

```javascript
const counter = document.querySelector(".counter-number")
async function updateCounter(){
    let response = await fetch("https://<FUNCTION_URL_ID>.lambda-url.us-east-1.on.aws/")
    let data = await response.json()
    counter.innerHTML = `Views: ${data}`;
}

updateCounter();
```

The counter uses a Lambda Function URL, not API Gateway. For one endpoint that only returns a number, it is the simpler option: there's no API to configure, and CORS is set on the URL itself.

## What I ran into

- **CORS.** The browser blocked the counter call until the Function URL allowed my site's origin. Now only `https://resume.helipatel.com` gets the CORS header.
- **The deploy pipeline.** It took about a dozen commits to get the GitHub Actions deploy working. Once it did, every push updated the live site.
- **IAM.** The counter only worked once the Lambda had the right DynamoDB permissions.

## Looking back (November 2026)

The main thing I'd change is that **Terraform only covers half the project.** The hosting is in code, but the counter was set up by hand, so I couldn't rebuild it from the repo.

I'd also:

- **Use remote Terraform state.** The state is a local file on my laptop. An S3 backend with locking would keep it safe.
- **Add tests.** A unit test for the Lambda and a smoke test after each deploy would catch a broken counter before a visitor does.

My [LLMOps Support Assistant](/projects/llmops-support-assistant/) is what I'd build now: the infrastructure is created by Terraform, and the pipeline runs tests before anything ships.
