1. Archirecure diagram
    - > ![alt text](image.png)
2. Initialize the Frontend
    - > npx create-next-app@latest frontend --typescript --tailwind --app --no-src-dir
2.  Install Required Dependencies(frontend)
    - > npm install lucide-react
3. Fix Tailwind v4 Configuration:Next.js 15.5 comes with Tailwind CSS v4, which has a different configuration approach. We need to update two files:
    - > export default {
    plugins: {
        '@tailwindcss/postcss': {},
    },
}


## Step 1: Setup the Backend Server

1.  Start the Backend Server
    - > cd backend
    - > uv init --bare
    - > uv python pin 3.14
    - > uv add -r requirements.txt (run this command after adding all the dependencies inside requirements.txt)
    - > uv run uvicorn server:app --reload

## Step 2: Start the Frontend Development Server

1. Start frontend
    - > cd frontend
    - > npm run dev

## Step 3: Start/Restart Backend Server

1. Stop the backend server (Ctrl+C in the terminal) and restart it:
    - > uv run uvicorn server:app --reload


## Set Up AWS Environment

1. Create Environment Configuration: Create a .env file in your project root (twin/.env):
    - > AWS_ACCOUNT_ID=your_aws_account_id
    - > DEFAULT_AWS_REGION=us-east-1
    - > PROJECT_NAME=twin-bedrock

2.  Sign In to AWS Console
    - > Go to [aws.amazon.com](https://aws.amazon.com/)
    - > Sign in as root user (we'll switch to IAM user shortly)
3. Create IAM User Group with Permissions
    - > In AWS Console, search for IAM
    - > Click User groups → Create group
    - > Group name: TwinAccess
    - > Attach the following policies - IMPORTANT see the last one added in to avoid permission issues later!
        1. AWSLambda_FullAccess - For Lambda operations
        2. AmazonS3FullAccess - For S3 bucket operations
        3. AmazonAPIGatewayAdministrator - For API Gateway
        4. CloudFrontFullAccess - For CloudFront distribution
        5. IAMReadOnlyAccess - To view roles
        6. AmazonDynamoDBFullAccess
        7. AmazonDynamoDBFullAccess_v2 - Needed on Day 4
        8. CloudWatchFullAccessV2
        9. AmazonBedrockFullAccess 
        10. CloudWatchFullAccess 
    - > Click Create group
4.  Add User to Group
    - > In IAM, click Users → Select aiengineer (from Week 1)
    - > Click Add to groups
    - > Select TwinAccess
    - > Click Add to groups
5.  Sign In as IAM User
    - > Sign out from root account
    - > Sign in as aiengineer with your IAM credentials
6. Follow the Deploy.py for deployment
7. Build the Lambda Package
    - > cd backend
    - > uv run deploy.py

##  Deploy Lambda Function
1. Create Lambda Function
    - > In AWS Console, search for Lambda
    - > Click Create function
    - > Choose Author from scratch
    - > Configuration:
        1. Function name: twin-bedrock-api
        2. Runtime: Python 3.14
        3. Architecture: x86_64
    - > Click Create function
2. Upload Your Code
    - > In the Lambda function page, under Code source
    - > Click Upload from → .zip file
    - > Click Upload and select your backend/lambda-deployment.zip
    - > Click Save
3.  Configure Handler
    - > In Runtime settings, click Edit
    - > Change Handler to: lambda_handler.handler
    - > Click Save
4. Configure Environment Variables
    - > Click Configuration tab → Environment variables
    - > Click Edit → Add environment variable
    - > Add these variables:
        1. CORS_ORIGINS = * (we'll restrict this later)
        2. USE_S3 = true
        3. S3_BUCKET = twin-memory
        4. Key: `DEFAULT_AWS_REGION` | Value: `us-east-1` (or your region)
        5. Key `BEDROCK_MODEL_ID` | Value: `amazon.nova-lite-v1:0` and remember that this might
    - > Click Save
6. **Add Bedrock Permissions to Lambda** Your Lambda function needs permission to call Bedrock:
    1. In Lambda → **Configuration** → **Permissions**
    2. Click on the execution role name (opens IAM)
    3. Click **Add permissions → Attach policies**
    4. Search for and select: **AmazonBedrockFullAccess**, **AmazonS3FullAccess**
    5. Click **Add permissions**

7. Increase Timeout
    1. In Configuration → General configuration
    2. Click Edit
    3. Set Timeout to 30 seconds
    4. Click Save
8. Test the Lambda Function
    1. Click Test tab
    2. Create new test event: 
        - > Event name: HealthCheck
        - > Event template: API Gateway AWS Proxy (scroll down to find it)
        - > Modify the Event JSON to:
                    {
            "version": "2.0",
            "routeKey": "GET /health",
            "rawPath": "/health",
            "headers": {
                "accept": "application/json",
                "content-type": "application/json",
                "user-agent": "test-invoke"
            },
            "requestContext": {
                "http": {
                "method": "GET",
                "path": "/health",
                "protocol": "HTTP/1.1",
                "sourceIp": "127.0.0.1",
                "userAgent": "test-invoke"
                },
                "routeKey": "GET /health",
                "stage": "$default"
            },
            "isBase64Encoded": false
            }
        - > Click Save → Test
        - > You should see a successful response with a body containing {"status": "healthy", "use_s3": true}
 
    
## Create S3 Buckets

1. Create Memory Bucket
    - > In AWS Console, search for S3
    - > Click Create bucket
    - > Configuration:
        - > Bucket name: twin-memory-[random-suffix] (must be globally unique)
        - > Region: Same as your Lambda (e.g., us-east-1)
        - > Leave all other settings as default
    - > Click Create bucket
    - > Copy the exact bucket name
2. Update Lambda Environment
    - > Go back to Lambda → Configuration → Environment variables
    - > Update S3_BUCKET with your actual bucket name(ex: ![alt text](image-1.png))
    - > Click Save
3. Add S3 Permissions to Lambda
    - > In Lambda → Configuration → Permissions
    - > Click on the execution role name (opens IAM)
    - > Click Add permissions → Attach policies
    - > Search and select: **AmazonS3FullAccess**
    - > Click Attach policies
4. Create Frontend Bucket
    1. Back in S3, click Create bucket
    2. Configuration:
        - > Bucket name: twin-frontend-[random-suffix]
        - > Region: Same as Lambda
        - > Uncheck "Block all public access"
        - > Check the acknowledgment box
    3. Click Create bucket
5. Enable Static Website Hosting
    1. Click on your frontend bucket
    2. Go to Properties tab
    3. Scroll to Static website hosting → Edit
    4. Enable static website hosting:
        - > Hosting type: Host a static website
        - > Index document: index.html
        - > Error document: 404.html
    5. Click Save changes
    6. Note the Bucket website endpoint URL

6. Configure Bucket Policy
    1. Go to Permissions tab
    2. Under Bucket policy, click Edit
    3. Add this policy (replace YOUR-BUCKET-NAME):
        - > {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Sid": "PublicReadGetObject",
                        "Effect": "Allow",
                        "Principal": "*",
                        "Action": "s3:GetObject",
                        "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
                    }
                ]
            }
    4. Click Save changes

## Set Up API Gateway
1. Create HTTP API with Integration
    1. In AWS Console, search for API Gateway
    2. Click Create API
    3. Choose HTTP API → Build
    4. Step 1 - Create and configure integrations:
        - > Click Add integration
        - > Integration type: Lambda
        - > Lambda function: Select twin-bedrock-api from the dropdown
        - > API name: twin-bedrock-api-gateway
        - > Click Next
2. Configure Routes
    1. Step 2 - Configure routes:
    2. You'll see a default route already created. Click Add route to add more:
    **Existing route (update it):**
        - Method: `ANY`
        - Resource path: `/{proxy+}`
        - Integration target: `twin-bedrock-api` (should already be selected)

    **Add these additional routes (click Add route for each):**

    Route 1:
    - Method: `GET`
    - Resource path: `/`
    - Integration target: `twin-bedrock-api`

    Route 2:
    - Method: `GET`
    - Resource path: `/health`
    - Integration target: `twin-bedrock-api`

    Route 3:
    - Method: `POST`
    - Resource path: `/chat`
    - Integration target: `twin-bedrock-api`

    Route 4 (for CORS):
    - Method: `OPTIONS`
    - Resource path: `/{proxy+}`
    - Integration target: `twin-bedrock-api`

    Click **Next**

3. Configure Stages
    1. **Step 3 - Configure stages:**
        - > Stage name: $default (leave as is)
        - > Auto-deploy: Leave enabled
    2. Click **Next**
4. Review and Create
    1. **Step 4 - Review and create:**
        - > Review your configuration
        - > You should see your Lambda integration and all routes listed
    2. Click Create
5. Configure CORS: After creation, configure CORS:
    1. In your newly created API, go to **CORS** in the left menu
    2. Click **Configure**
    3. Settings:
        - > Access-Control-Allow-Origin: Type * and click Add (important: you must click Add!)
        - > Access-Control-Allow-Headers: Type * and click Add (don't just type - click Add!)
        - > Access-Control-Allow-Methods: Type * and click Add (or add GET, POST, OPTIONS individually)
        - > Access-Control-Max-Age: 300
    4. click **Save**
6. Test Your API
    1. Go to API details (or Stages → $default) (ex: ![alt text](image-2.png))
    2. Copy your Invoke URL (looks like: https://abc123xyz.execute-api.us-east-1.amazonaws.com)
    3. Test with a browser by visiting: https://YOUR-API-ID.execute-api.us-east-1.amazonaws.com/health

You should see: {"status": "healthy", "use_s3": true}

**Note**: If you get a "Missing Authentication Token" error, make sure you're using the exact path /health and not just the base URL.

## Build and Deploy Frontend
1. Update Frontend API URL: Update frontend/components/twin.tsx - find the fetch call and update: 
    - > 
        // Replace this line:
        const response = await fetch('http://localhost:8000/chat', {

        // **With your API Gateway URL:**
        const response = await fetch('https://YOUR-API-ID.execute-api.us-east-1.amazonaws.com/chat', {
2. Configure for Static Export: First, update frontend/next.config.ts to enable static export:
    - > 
        import type { NextConfig } from "next";

        const nextConfig: NextConfig = {
        output: 'export',
        images: {
            unoptimized: true
        }
        };

        export default nextConfig;

3. Build Static Export
    - > cd frontend
    - > npm install
    - > npm run build

This creates an out directory with static files.

**Note:** With Next.js 15.5 and App Router, you must set output: 'export' in the config to generate the out directory.

4. Upload to S3 : Use the AWS CLI to upload your static files:
    - > cd frontend
    - > aws s3 sync out/ s3://YOUR-FRONTEND-BUCKET-NAME/ --delete
The `--delete` flag ensures that old files are removed from S3 if they're no longer in your build.

5. Test Your Static Site
    1. Go to your S3 bucket → Properties → Static website hosting
    2. Click the Bucket website endpoint URL
    3. Your twin should load! But CORS might block API calls...

## Set Up CloudFront
1. Get Your S3 Website Endpoint: First, you need your S3 static website URL (not the bucket name):
    1. Go to S3 → Your frontend bucket
    2. Click Properties tab
    3. Scroll to Static website hosting
    4. Copy the Bucket website endpoint (looks like: http://twin-frontend-xxx.s3-website-us-east-1.amazonaws.com)
Save this URL - you'll need it for CloudFront
2. Create CloudFront Distribution
    1. In AWS Console, search for **CloudFront**
    2. Click **Create distribution**
    3. **Step 1 - Origin:**
    - Distribution name: `twin-distribution`
    - Click **Next**
    4. **Step 2 - Add origin:**
    - Choose origin: Select **Other** (not Amazon S3!)
    - Origin domain name: Paste your S3 website endpoint WITHOUT the http://
        - Example: `twin-frontend-xxx.s3-website-us-east-1.amazonaws.com`
    - **Origin protocol policy**: Select **HTTP only** (CRITICAL - not HTTPS!)
        - This is because S3 static website hosting doesn't support HTTPS
        - If you select HTTPS, you'll get 504 Gateway Timeout errors
    - Origin name: `s3-static-website` (or leave auto-generated)
    - Leave other settings as default
    - Click **Add origin**
    5. **Step 3 - Default cache behavior:**
    - Path pattern: Leave as `Default (*)`
    - Origin and origin groups: Select your origin
    - Viewer protocol policy: **Redirect HTTP to HTTPS**
    - Allowed HTTP methods: **GET, HEAD**
    - Cache policy: **CachingOptimized**
    - Click **Next**
    6. **Step 4 - Web Application Firewall (WAF):**
    - Select **Do not enable security protections** (saves $14/month)
    - Click **Next**
    7. **Step 5 - Settings:**
    - Price class: **Use only North America and Europe** (to save costs)
    - Default root object: `index.html`
    - Click **Next**
    8. **Review** and click **Create distribution**

3. Wait for Deployment: CloudFront takes 5-15 minutes to deploy globally. Status will change from "Deploying" to "Enabled".


### Amazon Bedrock Token quaotas issue **FIX**

1. log in as a root user

2. go to Service Quotas
3. make sure your region is selected in the top-right corner
4. then choose AWS services -> search for Amazon Bedrock

5. search for the model you want to use (in my case it was **Cross-region model inference tokens per minute for Amazon Nova 2 Lite**)
6. Request increase at account level
7. set a requested value (I chose 8 000 000)
8. Request


## Set Up AWS CLI (to run aws s3 sync out/ s3://YOUR-FRONTEND-BUCKET-NAME/ --delete )

We need AWS CLI to push our image.

#### Create Access Keys

1. In AWS Console, go to **IAM**
2. Click **Users** → click on `aiengineer`
3. Click **Security credentials** tab
4. Under **Access keys**, click **Create access key**
5. Select **Command Line Interface (CLI)**
6. Check the confirmation box → **Next**
7. Description: `Docker push access`
8. Click **Create access key**
9. **Critical**: Download CSV or copy both:
   - Access key ID (like: `AKIAIOSFODNN7EXAMPLE`)
   - Secret access key (like: `wJalrXUtnFEMI/K7MDENG/bPxRfiCY`)
10. Click **Done**

#### Configure AWS CLI

Install AWS CLI if you haven't:
- **Mac**: `brew install awscli` or download from [aws.amazon.com/cli](https://aws.amazon.com/cli/)
- **Windows**: Download installer from [aws.amazon.com/cli](https://aws.amazon.com/cli/)

Configure it:
```bash
aws configure
```

Enter:
- AWS Access Key ID: (paste your key)
- AWS Secret Access Key: (paste your secret)
- Default region: Choose based on your location:
  - **US East Coast**: `us-east-1` (N. Virginia)
  - **US West Coast**: `us-west-2` (Oregon)
  - **Europe**: `eu-west-1` (Ireland)
  - **Asia**: `ap-southeast-1` (Singapore)
  - **Pick the closest region for best performance!**
- Default output format: `json`

**Important**: Remember your region choice - you'll use it throughout this course!
