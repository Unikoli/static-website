# Elevate — Modern Landing Page

A responsive static landing page built with **HTML5 and CSS3**. It does not require JavaScript, a framework, or a build process.

## Folder structure

```text
modern-landing-page/
├── .github/
│   └── workflows/
│       └── deploy-to-s3.yml
├── index.html
├── css/
│   └── style.css
├── images/
│   └── (add your local images here)
└── README.md
```

## Run locally

1. Open this folder in Visual Studio Code.
2. Open `index.html` in a browser, or use the Live Server extension.

## Deploy automatically to Amazon S3

The GitHub Actions workflow at `.github/workflows/deploy-to-s3.yml` syncs the website to an S3 bucket whenever code is pushed to the `main` branch. It also supports manual runs from the Actions tab.

### 1. Prepare the S3 bucket

1. Create an S3 bucket with a globally unique name in your preferred AWS Region.
2. In **Properties → Static website hosting**, enable website hosting.
3. Set the index document to `index.html`.
4. For the simplest direct S3 website endpoint, configure the bucket for public website reads and add a bucket policy that grants `s3:GetObject` to `Principal: "*"`. This requires disabling the relevant S3 Block Public Access settings. Only do this for a bucket intended to host public website assets.
5. Note the bucket name and Region. The S3 website endpoint is shown on the Static website hosting panel.

For a production site where public S3 access is not acceptable, use CloudFront with Origin Access Control instead; the S3 bucket can remain private. The workflow's upload step still targets the bucket.

### 2. Create an IAM role for GitHub Actions (recommended: OIDC)

Do not store long-lived AWS access keys in GitHub. Configure GitHub OIDC:

1. In IAM, add an identity provider for `https://token.actions.githubusercontent.com` if your AWS account does not already have one. Provider audience: `sts.amazonaws.com`.
2. Create an IAM role for web identity with this trust policy. Replace the account ID, GitHub owner, and repository placeholders:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_OWNER/YOUR_REPOSITORY:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

3. Attach this least-privilege permissions policy to the role. Replace `YOUR_BUCKET_NAME`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListWebsiteBucket",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME"
    },
    {
      "Sid": "ManageWebsiteObjects",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*"
    }
  ]
}
```

4. Copy the role ARN.

### 3. Add GitHub repository settings

In your GitHub repository, open **Settings → Secrets and variables → Actions**.

Create:

- **Repository secret** `AWS_ROLE_ARN` = the IAM role ARN.
- **Repository variable** `AWS_REGION` = your bucket's Region, for example `ap-south-1`.
- **Repository variable** `S3_BUCKET` = your exact S3 bucket name.

Do not commit credentials, secret values, or account-specific ARNs into the repository.

### 4. Push and verify

1. Commit and push the project to the repository's `main` branch.
2. Open the repository's **Actions** tab.
3. Open **Deploy static website to Amazon S3** and confirm the latest run completes successfully.
4. Open the S3 bucket's **Properties → Static website hosting** endpoint and confirm the site loads.
5. Make a visible change in `index.html`, commit and push it to `main`, then confirm a new workflow run succeeds and the change appears at the website endpoint. If a browser or CDN shows an old copy, hard-refresh or invalidate the CloudFront cache if CloudFront is in use.

### Workflow behavior

- Runs on every push to `main`.
- Can also be started manually using `workflow_dispatch`.
- Uses short-lived credentials from GitHub OIDC rather than stored AWS access keys.
- `aws s3 sync --delete` removes destination objects that are no longer present in the repository. Use a dedicated bucket or prefix for this website.
- The workflow excludes Git metadata, the workflow directory, and this README from the website upload.

## Customize

- Change page content and links in `index.html`.
- Update colors, spacing, typography, and responsive breakpoints in `css/style.css`.
- Add local images to `images/` and reference them with relative paths.
- Replace sample brand names, testimonial, and `hello@example.com` with your own details.

The page uses Google Fonts from an external CDN. Remove the Google Fonts links and use system fonts in `style.css` if fully offline operation is required.
