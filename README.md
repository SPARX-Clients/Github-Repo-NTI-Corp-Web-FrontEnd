# Static Website Infrastructure

Deploy a production-ready static website on AWS using CloudFront, S3, and ACM — with automated deployments via GitHub Actions and keyless OIDC authentication.

---

## Architecture Overview

```
                   Your External DNS Provider
                   (GoDaddy, Cloudflare, etc.)
                   CNAME example.com      ──┐
                   CNAME www.example.com  ──┤
                                            │
                        ┌───────────────────▼─────────────────┐
                        │      CloudFront Distribution        │
                        │  • HTTPS only (TLS 1.2+)            │
                        │  • HTTP/2 + HTTP/3                  │
                        │  • ACM Certificate (us-east-1)      │
                        │  • Security response headers        │
                        │  • SPA 403/404 → index.html         │
                        └───────────────────┬─────────────────┘
                                            │  Origin Access Control (OAC)
                        ┌───────────────────▼─────────────────┐
                        │        S3 Bucket (private)          │
                        │  • Static website hosting enabled   │
                        │  • No public access                 │
                        │  • Versioning enabled               │
                        │  • AES-256 encryption               │
                        └─────────────────────────────────────┘

  GitHub Actions ──OIDC──► IAM Role ──► S3 Sync + CF Invalidation
  (push/merge to main)
```

---

## Files

| File | Description |
|------|-------------|
| `.cloudformation/cloudformation-certificate.yaml` | Stack 1 — ACM Certificate only |
| `.cloudformation/cloudformation-infrastructure.yaml` | Stack 2 — S3, CloudFront, IAM Role |
| `.github/workflows/deploy.yaml` | GitHub Actions workflow — deploys site on merge to main |
| `README.md` | This file |

---

## Prerequisites

- An AWS account with permissions to create IAM, S3, CloudFront, and ACM resources
- A domain with access to your DNS provider to add CNAME records
- AWS CLI installed and configured (`aws configure`)
- A GitHub repository containing your website files in a `code/` folder

---

## AWS Infrastructure

Infrastructure is split across two CloudFormation stacks to allow certificate DNS validation to complete before the rest of the resources are provisioned.

### Stack 1 — ACM Certificate (`cloudformation-certificate.yaml`)

- Covers both the apex domain (`example.com`) and `www` subdomain (`www.example.com`)
- Uses DNS validation — after deploying, two CNAME records must be manually added to your DNS provider
- **Must be deployed in `us-east-1`** — AWS requirement for certificates used with CloudFront
- Stack will pause at `CREATE_IN_PROGRESS` until ACM detects the DNS records and validates the certificate (typically 5–30 minutes)
- Once validated the stack completes and its `CertificateArn` export is available for Stack 2

### Stack 2 — Infrastructure (`cloudformation-infrastructure.yaml`)

Imports the `CertificateArn` directly from Stack 1 via `Fn::ImportValue` — CloudFormation will refuse to deploy Stack 2 if Stack 1 has not completed successfully.

#### S3 Bucket

- Named `<DomainName>-website` (e.g. `example.com-website`)
- **Static website hosting enabled** with `index.html` as both the index and error document
- **Fully private** — no public access; all traffic is routed through CloudFront via OAC
- Versioning enabled for rollback capability
- Server-side encryption (AES-256) at rest

#### CloudFront Distribution

- Serves both `example.com` and `www.example.com`
- Forces HTTPS with `redirect-to-https` viewer protocol policy
- Supports HTTP/2 and HTTP/3
- Uses AWS-managed policies:
  - **CachingOptimized** — cache policy for best performance
  - **CORS-S3Origin** — origin request policy
  - **SecurityHeadersPolicy** — adds `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options`, etc.
- SPA-friendly error handling: 403 and 404 errors mapped to `index.html` with a 200 response
- Minimum TLS version: `TLSv1.2_2021`

#### GitHub Actions IAM Role

- Uses **OpenID Connect (OIDC)** — no long-lived AWS credentials stored in GitHub
- Scoped to a specific GitHub organisation, repository, and branch
- The GitHub OIDC provider ARN is constructed directly (`arn:aws:iam::<account>:oidc-provider/token.actions.githubusercontent.com`) so the role works whether the provider was created by this stack or already existed
- The `CreateOidcProvider` parameter controls whether the OIDC provider resource is created — set to `false` by default since only one can exist per AWS account
- Least-privilege permissions:
  - `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject`, `s3:ListBucket` on the website bucket only
  - `cloudfront:CreateInvalidation`, `cloudfront:GetInvalidation`, `cloudfront:ListInvalidations` on the distribution only

---

## Deployment

> ⚠️ Both stacks **must be deployed in `us-east-1`** because ACM certificates for CloudFront are required to be in that region.

### Step 1 — Deploy the Certificate Stack

```bash
aws cloudformation deploy \
  --template-file cloudformation-certificate.yaml \
  --stack-name my-website-cert \
  --region us-east-1 \
  --parameter-overrides \
      DomainName=example.com
```

The stack will pause at `CREATE_IN_PROGRESS` waiting for certificate validation. Retrieve the required DNS records:

```bash
aws acm describe-certificate \
  --certificate-arn $(aws cloudformation describe-stacks \
    --stack-name my-website-cert --region us-east-1 \
    --query 'Stacks[0].Outputs[0].OutputValue' --output text) \
  --region us-east-1 \
  --query 'Certificate.DomainValidationOptions[*].{Domain:DomainName,Name:ResourceRecord.Name,Value:ResourceRecord.Value}'
```

Add both CNAME records to your DNS provider and wait for the stack to complete (5–30 minutes).

#### Stack 1 Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `DomainName` | ✅ | — | Root domain (e.g. `example.com`) |

---

### Step 2 — Deploy the Infrastructure Stack

```bash
aws cloudformation deploy \
  --template-file cloudformation-infrastructure.yaml \
  --stack-name my-website \
  --region us-east-1 \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
      DomainName=example.com \
      CertStackName=my-website-cert \
      GitHubOrg=my-org \
      GitHubRepo=my-repo \
      GitHubBranch=main \
      CreateOidcProvider=false
```

#### Stack 2 Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `DomainName` | ✅ | — | Root domain (e.g. `example.com`) |
| `CertStackName` | ✅ | `my-website-cert` | Name of the deployed Stack 1 |
| `GitHubOrg` | ✅ | — | GitHub organisation or username |
| `GitHubRepo` | ✅ | — | GitHub repository name |
| `GitHubBranch` | ✅ | `main` | Branch allowed to assume the deploy role |
| `CreateOidcProvider` | ❌ | `false` | Set to `true` only if the GitHub OIDC provider does not yet exist in this account |
| `DefaultRootObject` | ❌ | `index.html` | CloudFront default root object |
| `PriceClass` | ❌ | `PriceClass_100` | CloudFront edge location coverage |

**CloudFront Price Classes:**

| Value | Coverage |
|-------|----------|
| `PriceClass_100` | US, Canada, Europe (cheapest) |
| `PriceClass_200` | + Asia, Middle East, Africa |
| `PriceClass_All` | All edge locations (most expensive) |

> **GitHub OIDC Provider:** Only one can exist per AWS account. If this is the first time setting up GitHub Actions OIDC in your account, set `CreateOidcProvider=true`. For all subsequent stacks or if it was created manually, leave it as `false`. To check if it already exists:
> ```bash
> aws iam list-open-id-connect-providers \
>   --query 'OpenIDConnectProviderList[?ends_with(Arn, `token.actions.githubusercontent.com`)].Arn' \
>   --output text
> ```

---

### Step 3 — Point Your DNS at CloudFront

After Stack 2 deploys, retrieve the CloudFront domain name:

```bash
aws cloudformation describe-stacks \
  --stack-name my-website \
  --region us-east-1 \
  --query 'Stacks[0].Outputs[?OutputKey==`CloudFrontDomainName`].OutputValue' \
  --output text
```

Add these records at your DNS provider:

| Record type | Name | Value |
|-------------|------|-------|
| `CNAME` | `example.com` | `<CloudFrontDomainName>` |
| `CNAME` | `www.example.com` | `<CloudFrontDomainName>` |

> **Note:** Some DNS providers do not support CNAME records on the apex domain (`example.com`). Use an `ALIAS` or `ANAME` record type instead if available, or check your provider's documentation.

---

### Step 4 — Retrieve Stack Outputs for GitHub

```bash
aws cloudformation describe-stacks \
  --stack-name my-website \
  --region us-east-1 \
  --query 'Stacks[0].Outputs'
```

| Output Key | Description |
|------------|-------------|
| `GitHubActionsRoleArn` | IAM Role ARN → set as `AWS_ROLE_ARN` in GitHub |
| `WebsiteBucketName` | S3 bucket name → set as `S3_BUCKET_NAME` in GitHub |
| `CloudFrontDistributionId` | Distribution ID → set as `CLOUDFRONT_DIST_ID` in GitHub |
| `CloudFrontDomainName` | CloudFront domain → use for DNS records |
| `WebsiteUrl` | Your live website URL |

---

### Step 5 — Configure GitHub Environment Variables

In your GitHub repository go to **Settings → Environments → New environment** and create an environment named `production`. Add the following **Environment variables**:

| Variable | Value |
|----------|-------|
| `AWS_ROLE_ARN` | Value of `GitHubActionsRoleArn` output |
| `AWS_REGION` | `us-east-1` |
| `S3_BUCKET_NAME` | Value of `WebsiteBucketName` output |
| `CLOUDFRONT_DIST_ID` | Value of `CloudFrontDistributionId` output |

> 💡 Using an **Environment** (rather than repository-level variables) lets you add required reviewers, restrict which branches can deploy, and maintain separate configs for staging vs. production.

---

### Step 6 — Add the GitHub Actions Workflow

Copy `deploy.yaml` into your repository at `.github/workflows/deploy.yaml` and commit it to `main`. The workflow will trigger automatically on every merge to `main`.

---

## GitHub Actions Workflow

The workflow (`.github/workflows/deploy.yaml`) triggers on every push or merge to `main` and can also be run manually from the GitHub Actions UI.

### Steps

1. **Checkout** — checks out the repository source
2. **Configure AWS credentials** — assumes the IAM role via OIDC (no AWS secrets stored in GitHub)
3. **Deploy to S3** — syncs the `./code` folder to S3 with split cache headers per file type:
   - HTML files: `Cache-Control: public, no-cache, must-revalidate` — browsers always fetch fresh
   - CSS & JS: `Cache-Control: public, max-age=31536000, immutable` — cached for 1 year
   - Images (`png`, `jpg`, `jpeg`, `gif`, `svg`, `webp`, `ico`): `Cache-Control: public, max-age=31536000, immutable`
4. **Invalidate CloudFront** — creates a `/*` invalidation and waits for it to complete
5. **Job summary** — writes a deployment summary to the GitHub Actions run

### Deployment Trigger

The workflow fires on a push to `main`, which includes merges from any branch (e.g. a pull request merge from `dev` into `main`). No additional configuration is needed for merge-based deployments.

---

## Security Notes

- **No AWS credentials are stored in GitHub** — authentication uses short-lived OIDC tokens valid for one job only
- The IAM role trust policy is scoped to a specific repo and branch — forked repositories cannot assume the role
- The S3 bucket has all public access blocked; objects are only reachable via CloudFront OAC
- CloudFront enforces HTTPS and adds security response headers on every request
- ACM handles certificate renewal automatically as long as the DNS CNAME validation records remain in place

---

## Troubleshooting

**Stack 1 times out waiting for certificate validation**

Check that the two CNAME records were added correctly to your DNS provider. You can verify DNS propagation with:
```bash
dig CNAME _<validation-token>.example.com
```

**`Export my-website-cert-CertificateArn does not exist` error on Stack 2**

Stack 1 has not completed successfully yet. Wait for the certificate to be validated and Stack 1 to reach `CREATE_COMPLETE` before deploying Stack 2.

**`GitHubOidcProvider already exists` error**

Set `CreateOidcProvider=false` (the default) and redeploy. The IAM role references the provider by its fixed ARN so it works regardless of which stack created it.

**`Not authorized to perform sts:AssumeRoleWithWebIdentity` in GitHub Actions**

The OIDC trust policy `sub` claim must match the workflow trigger. Since the workflow uses `environment: production`, ensure the trust policy includes the environment claim format. The template already handles both branch and environment claim formats via `StringLike`.

**CloudFront returns 403 for all requests**

The bucket policy is scoped to the CloudFront distribution ARN. If the distribution was recreated (new ARN), redeploy Stack 2 so the bucket policy is updated.

**Changes not appearing after deploy**

The workflow creates a `/*` CloudFront invalidation and waits for it to complete before finishing. If changes still don't appear, do a hard refresh (`Ctrl+Shift+R` / `Cmd+Shift+R`) to bypass the browser cache.

---

## Updating the Stacks

To change parameters or update either template, re-run the relevant `aws cloudformation deploy` command with new values. CloudFormation computes a changeset and updates only the affected resources.

---

## Teardown

```bash
# Delete Stack 2 first (it depends on Stack 1's export)
aws cloudformation delete-stack --stack-name my-website --region us-east-1

# Then delete Stack 1
aws cloudformation delete-stack --stack-name my-website-cert --region us-east-1
```

> ⚠️ Stack 2 deletion will fail if the S3 bucket is not empty. Empty it first:
> ```bash
> aws s3 rm s3://example.com-website --recursive
> ```