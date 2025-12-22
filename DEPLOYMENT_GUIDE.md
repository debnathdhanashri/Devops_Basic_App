# DevOps Deployment Guide - AWS CI/CD Pipeline

This guide will help you deploy your React app using a CI/CD pipeline on AWS.

## Architecture Overview

```
GitHub Push → GitHub Actions → Build App → Deploy to S3 → Serve via CloudFront
```

## Prerequisites

1. **GitHub Account** (to host your code)
2. **AWS Account** (free tier available)
3. **Git installed** on your machine

---

## Step 1: Set Up Git Repository

```bash
# Initialize git (if not already done)
git init

# Add all files
git add .

# Commit
git commit -m "Initial commit"

# Create a GitHub repository and push
# Go to github.com and create a new repository, then:
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git
git branch -M main
git push -u origin main
```

---

## Step 2: AWS Setup

### 2.1 Create S3 Bucket

1. Go to AWS Console → S3
2. Click "Create bucket"
3. **Bucket name**: `my-react-app-bucket-unique-name` (must be globally unique)
4. **Region**: Choose your preferred region (e.g., us-east-1)
5. **Uncheck** "Block all public access"
6. Click "Create bucket"

### 2.2 Configure S3 for Static Website Hosting

1. Click on your bucket
2. Go to "Properties" tab
3. Scroll to "Static website hosting" → Click "Edit"
4. Enable it:
   - **Index document**: `index.html`
   - **Error document**: `index.html` (for React Router)
5. Save changes

### 2.3 Set Bucket Policy

1. Go to "Permissions" tab
2. Scroll to "Bucket policy" → Click "Edit"
3. Paste this policy (replace YOUR_BUCKET_NAME):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*"
    }
  ]
}
```

### 2.4 Create CloudFront Distribution (CDN - Optional but Recommended)

1. Go to AWS Console → CloudFront
2. Click "Create distribution"
3. **Origin domain**: Select your S3 bucket
4. **Origin access**: Public
5. **Viewer protocol policy**: Redirect HTTP to HTTPS
6. **Default root object**: `index.html`
7. **Custom error responses**: Add error response
   - HTTP error code: 403
   - Response page path: `/index.html`
   - HTTP response code: 200
8. Click "Create distribution"
9. **Note the Distribution ID** (you'll need it)

---

## Step 3: Create IAM User for GitHub Actions

1. Go to AWS Console → IAM
2. Click "Users" → "Create user"
3. **User name**: `github-actions-deployer`
4. Click "Next"
5. **Permissions**: Click "Attach policies directly"
6. Search and add these policies:
   - `AmazonS3FullAccess`
   - `CloudFrontFullAccess`
7. Click "Next" → "Create user"

### Get Access Keys

1. Click on the user you just created
2. Go to "Security credentials" tab
3. Click "Create access key"
4. Choose "Application running outside AWS"
5. Click "Next" → "Create access key"
6. **IMPORTANT**: Copy both:
   - Access key ID
   - Secret access key
   - You won't be able to see the secret again!

---

## Step 4: Configure GitHub Secrets

1. Go to your GitHub repository
2. Click "Settings" → "Secrets and variables" → "Actions"
3. Click "New repository secret"
4. Add these secrets one by one:

| Name                         | Value                                      |
| ---------------------------- | ------------------------------------------ |
| `AWS_ACCESS_KEY_ID`          | Your AWS access key ID                     |
| `AWS_SECRET_ACCESS_KEY`      | Your AWS secret access key                 |
| `S3_BUCKET_NAME`             | Your S3 bucket name                        |
| `CLOUDFRONT_DISTRIBUTION_ID` | Your CloudFront distribution ID (if using) |

---

## Step 5: Test the Pipeline

1. Make a small change to your app (e.g., edit `src/App.jsx`)
2. Commit and push:

```bash
git add .
git commit -m "Test CI/CD pipeline"
git push
```

3. Go to GitHub → Your repo → "Actions" tab
4. Watch your workflow run!
5. Once complete, visit your CloudFront URL or S3 website URL

---

## Accessing Your Deployed App

### Via S3 (if no CloudFront):

```
http://YOUR_BUCKET_NAME.s3-website-REGION.amazonaws.com
```

### Via CloudFront (recommended):

```
https://YOUR_DISTRIBUTION_ID.cloudfront.net
```

---

## Troubleshooting

### Build fails

- Check Node.js version compatibility
- Run `npm ci` and `npm run build` locally first

### Deployment fails

- Verify AWS credentials in GitHub secrets
- Check IAM user has correct permissions
- Ensure S3 bucket name is correct

### Website shows 404

- Check S3 static hosting is enabled
- Verify bucket policy allows public access
- For CloudFront, check error response configuration

### Changes don't appear

- CloudFront caches content. Wait for invalidation to complete
- Can take 5-15 minutes for CloudFront to update

---

## Cost Estimation

- **S3**: ~$0.023 per GB/month + requests
- **CloudFront**: Free tier: 1TB data transfer out
- **GitHub Actions**: 2000 minutes/month free for public repos
- **Total for small app**: Usually under $1/month or free

---

## Next Steps for Learning

1. **Add Environment Variables**: Learn about different environments (dev/staging/prod)
2. **Custom Domain**: Set up Route 53 and SSL certificate
3. **Docker**: Containerize your app
4. **Infrastructure as Code**: Use Terraform or AWS CDK
5. **Monitoring**: Set up CloudWatch logs and alarms
6. **Testing**: Add automated tests to the pipeline

---

## Advanced: Adding Tests to Pipeline

Add this to your `.github/workflows/deploy.yml` before the build step:

```yaml
- name: Run tests
  run: npm test -- --passWithNoTests
```

## Alternative: AWS Amplify (Easier Setup)

If you want a simpler approach:

1. Go to AWS Amplify Console
2. Click "New app" → "Host web app"
3. Connect your GitHub repository
4. Amplify will auto-detect Vite settings
5. Click "Save and deploy"

Done! Amplify handles everything automatically.

---

## Resources

- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
- [GitHub Actions Documentation](https://docs.github.com/actions)
- [AWS CloudFront Documentation](https://docs.aws.amazon.com/cloudfront/)
- [Vite Deployment Guide](https://vitejs.dev/guide/static-deploy.html)

---

## Questions?

Common workflow:

1. Make code changes locally
2. Test locally: `npm run dev`
3. Commit and push to GitHub
4. Pipeline automatically builds and deploys
5. Check GitHub Actions for status
6. Visit your live site!

Happy DevOps learning! 🚀
