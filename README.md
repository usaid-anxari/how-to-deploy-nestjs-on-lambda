# NestJS Docker Lambda Deployment - Universal Guide 🐳

> **Cross-Platform Guide**: Deploy your NestJS backend to AWS Lambda using Docker on Windows, macOS, and Linux in under 30 minutes.

## 🖥️ System Requirements

### All Operating Systems:
- AWS Account with proper permissions
- Node.js 18+ installed
- Your NestJS project ready
- Internet connection

### Platform-Specific Requirements:
- **Windows**: PowerShell or Command Prompt
- **macOS**: Terminal
- **Linux/Ubuntu**: Terminal or Bash

---

## 📦 Step 1: Install Prerequisites

### 1.1 Install Docker (All Platforms)

**Windows:**
```powershell
# Download Docker Desktop from https://docker.com/products/docker-desktop
# Install and restart your computer
# Verify installation:
docker --version
```

**macOS:**
```bash
# Install via Homebrew (recommended)
brew install --cask docker
# Or download from https://docker.com/products/docker-desktop
# Verify installation:
docker --version
```

**Linux/Ubuntu:**
```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
# Logout and login again, then verify:
docker --version
```

### 1.2 Install AWS CLI (All Platforms)

**Windows:**
```powershell
# Download and install from: https://aws.amazon.com/cli/
# Or use chocolatey:
choco install awscli
# Verify:
aws --version
```

**macOS:**
```bash
# Install via Homebrew:
brew install awscli
# Or download installer from AWS website
# Verify:
aws --version
```

**Linux/Ubuntu:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
# Verify:
aws --version
```

### 1.3 Install Project Dependencies
```bash
# Navigate to your NestJS project
cd your-nestjs-project

# Install required packages
npm install --save serverless-http@3.2.0
npm install -g serverless@3
```

## 🚀 Step 2: Prepare Your Project

### 2.1 Create Lambda Handler
**Create file: `src/lambda.ts`**
```typescript
import { NestFactory } from '@nestjs/core';
import { ExpressAdapter } from '@nestjs/platform-express';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';
import * as express from 'express';
import serverlessExpress from 'serverless-http';

let cachedServer;

async function bootstrap() {
  if (!cachedServer) {
    const expressApp = express();
    const app = await NestFactory.create(AppModule, new ExpressAdapter(expressApp));

    app.enableCors({
      origin: ['http://localhost:3000', 'https://your-domain.com'],
      credentials: true,
    });

    app.useGlobalPipes(new ValidationPipe({
      whitelist: true,
      transform: true,
    }));

    app.setGlobalPrefix('api');
    await app.init();
    cachedServer = serverlessExpress(expressApp);
  }
  return cachedServer;
}

export const handler = async (event, context) => {
  const server = await bootstrap();
  return server(event, context);
};
```

---

## 🐳 Step 3: Create Docker Files

### 3.1 Lambda Dockerfile
**Create file: `Dockerfile.lambda`**
```dockerfile
FROM public.ecr.aws/lambda/nodejs:20

# Copy package files
COPY package*.json ${LAMBDA_TASK_ROOT}/

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . ${LAMBDA_TASK_ROOT}/

# Build application
RUN npm run build

# Set handler
CMD ["dist/lambda.handler"]
```

### 3.2 Build Scripts (Platform-Specific)

**For Windows - Create file: `build-lambda.bat`**
```batch
@echo off
echo 🔨 Building NestJS for Lambda...
npm run build
if %errorlevel% equ 0 (
    echo ✅ Build complete!
) else (
    echo ❌ Build failed!
    exit /b 1
)
```

**For macOS/Linux - Create file: `build-lambda.sh`**
```bash
#!/bin/bash
echo "🔨 Building NestJS for Lambda..."
npm run build
if [ $? -eq 0 ]; then
    echo "✅ Build complete!"
else
    echo "❌ Build failed!"
    exit 1
fi
```

---

## ⚙️ Step 4: Configure Serverless

### 4.1 Create Serverless Config
**Create file: `serverless.yml`**
```yaml
service: your-app-backend

frameworkVersion: '3'

provider:
  name: aws
  region: us-east-1
  stage: ${opt:stage, 'dev'}
  timeout: 30
  memorySize: 1024
  
  # Environment variables
  environment:
    NODE_ENV: production
    DB_HOST: ${env:DB_HOST}
    DB_USERNAME: ${env:DB_USERNAME}
    DB_PASSWORD: ${env:DB_PASSWORD}
    DB_NAME: ${env:DB_NAME}
    AWS_S3_BUCKET: ${env:AWS_S3_BUCKET}
    STRIPE_SECRET_KEY: ${env:STRIPE_SECRET_KEY}

  # ECR Image
  ecr:
    images:
      app:
        path: ./
        file: Dockerfile.lambda

functions:
  api:
    image:
      name: app
    events:
      - httpApi:
          path: /{proxy+}
          method: ANY
      - httpApi:
          path: /
          method: ANY
    url: true
```

### 4.2 Environment Variables
**Create file: `.env.lambda`**
```env
# Database
DB_HOST=your-database-host.rds.amazonaws.com
DB_USERNAME=your_username
DB_PASSWORD=your_password
DB_NAME=your_database

# AWS
AWS_S3_BUCKET=your-bucket-name

# Stripe
STRIPE_SECRET_KEY=sk_test_your_stripe_key

# URLs
FRONTEND_URL=https://your-domain.com
BACKEND_URL=https://api.your-domain.com
```

---

## 🚀 Step 5: Configure AWS & Deploy

### 5.1 Configure AWS Credentials (All Platforms)
```bash
# Configure AWS credentials (same for all platforms)
aws configure
# Enter your AWS Access Key ID
# Enter your AWS Secret Access Key  
# Default region name: us-east-1
# Default output format: json
```

### 5.2 Create Deployment Scripts

**For Windows - Create file: `deploy.bat`**
```batch
@echo off
echo 🚀 Starting Docker Lambda deployment...

REM Check if .env.lambda exists
if not exist ".env.lambda" (
    echo ❌ .env.lambda file not found!
    echo Please create .env.lambda with your environment variables
    exit /b 1
)

echo ✅ Environment file found

REM Build application
echo 🔨 Building application...
npm run build
if %errorlevel% neq 0 (
    echo ❌ Build failed!
    exit /b 1
)

REM Deploy with Serverless
echo ☁️ Deploying to AWS Lambda...
serverless deploy --stage dev
if %errorlevel% neq 0 (
    echo ❌ Deployment failed!
    exit /b 1
)

REM Get deployment info
echo 📋 Getting deployment info...
serverless info --stage dev

echo 🎉 Deployment complete!
```

**For macOS/Linux - Create file: `deploy.sh`**
```bash
#!/bin/bash

echo "🚀 Starting Docker Lambda deployment..."

# Check if .env.lambda exists
if [ ! -f ".env.lambda" ]; then
    echo "❌ .env.lambda file not found!"
    echo "Please create .env.lambda with your environment variables"
    exit 1
fi

# Load environment variables
export $(cat .env.lambda | grep -v '^#' | xargs)
echo "✅ Environment loaded"

# Build application
echo "🔨 Building application..."
npm run build
if [ $? -ne 0 ]; then
    echo "❌ Build failed!"
    exit 1
fi

# Deploy with Serverless
echo "☁️ Deploying to AWS Lambda..."
serverless deploy --stage dev
if [ $? -ne 0 ]; then
    echo "❌ Deployment failed!"
    exit 1
fi

# Get deployment info
echo "📋 Getting deployment info..."
serverless info --stage dev

echo "🎉 Deployment complete!"
```

### 5.3 Execute Deployment

**Windows:**
```powershell
# Run deployment
.\deploy.bat
```

**macOS/Linux:**
```bash
# Make script executable
chmod +x deploy.sh build-lambda.sh

# Run deployment
./deploy.sh
```

---

## 🧪 Step 6: Test Your Deployment

### 6.1 Expected Deployment Success Response
**When deployment succeeds, you should see:**
```
✅ Deployment complete!

Service Information
service: your-app-backend
stage: dev
region: us-east-1
stack: your-app-backend-dev
resources: 8
api keys:
  None
endpoints:
  ANY - https://abc123def.execute-api.us-east-1.amazonaws.com/dev/{proxy+}
  ANY - https://abc123def.execute-api.us-east-1.amazonaws.com/dev/
functions:
  api: your-app-backend-dev-api
layers:
  None
```

### 6.2 Expected Failure Responses & Solutions

**❌ Environment Variables Missing:**
```
Error: Environment variable DB_HOST is not defined
```
**Solution:** Check your `.env.lambda` file exists and has all required variables.

**❌ AWS Credentials Invalid:**
```
Error: The security token included in the request is invalid
```
**Solution:** Run `aws configure` again with correct credentials.

**❌ Docker Not Running:**
```
Error: Cannot connect to the Docker daemon
```
**Solution:** Start Docker Desktop or Docker service.

**❌ Build Fails:**
```
Error: Command failed: npm run build
```
**Solution:** Check your TypeScript code for errors, run `npm install` first.

### 6.3 Test API Endpoints

**Get your Lambda URL from deployment output, then test:**

**Windows (PowerShell):**
```powershell
# Test health endpoint
Invoke-RestMethod -Uri "https://YOUR_LAMBDA_URL/api/health" -Method GET

# Test with data
$body = @{test="data"} | ConvertTo-Json
Invoke-RestMethod -Uri "https://YOUR_LAMBDA_URL/api/your-endpoint" -Method POST -Body $body -ContentType "application/json"
```

**macOS/Linux:**
```bash
# Test health endpoint
curl https://YOUR_LAMBDA_URL/api/health

# Test with data
curl -X POST https://YOUR_LAMBDA_URL/api/your-endpoint \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'
```

### 6.4 Expected API Responses

**✅ Successful Health Check:**
```json
{
  "status": "ok",
  "service": "your-app-backend",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

**❌ Lambda Function Error:**
```json
{
  "message": "Internal server error"
}
```
**Solution:** Check CloudWatch logs for detailed error information.

**❌ Environment Variable Missing in Lambda:**
```json
{
  "message": "Database connection failed"
}
```
**Solution:** Verify environment variables are set in `serverless.yml`.

---

## 🔧 Step 7: Common Commands (All Platforms)

### Development Commands (Same for All Platforms)
```bash
# View real-time logs
serverless logs -f api --tail --stage dev

# Remove deployment completely
serverless remove --stage dev

# Deploy to production
serverless deploy --stage prod

# Package without deploying (for testing)
serverless package --stage dev

# Get deployment information
serverless info --stage dev
```

### Docker Commands

**All Platforms:**
```bash
# Build Docker image locally
docker build -f Dockerfile.lambda -t your-app .

# Test Docker image locally
docker run -p 9000:8080 your-app
```

**Test Lambda Function Locally:**

**Windows (PowerShell):**
```powershell
# Test Lambda function
$body = @{
    httpMethod = "GET"
    path = "/api/health"
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:9000/2015-03-31/functions/function/invocations" -Method POST -Body $body -ContentType "application/json"
```

**macOS/Linux:**
```bash
# Test Lambda function
curl "http://localhost:9000/2015-03-31/functions/function/invocations" \
  -d '{"httpMethod":"GET","path":"/api/health"}'
```

---

## 🛠️ Step 8: Troubleshooting Guide

### Common Issues & Platform-Specific Solutions

#### Issue 1: Build Fails
**Error Message:**
```
❌ Build failed! 
Error: Command failed: npm run build
```

**Windows Solution:**
```powershell
# Clean and rebuild
Remove-Item -Recurse -Force node_modules, dist -ErrorAction SilentlyContinue
npm install
npm run build
```

**macOS/Linux Solution:**
```bash
# Clean and rebuild
rm -rf node_modules dist
npm install
npm run build
```

#### Issue 2: Docker Build Fails
**Error Message:**
```
❌ Docker build failed
Error: Cannot connect to the Docker daemon
```

**All Platforms Solution:**
```bash
# 1. Make sure Docker is running
docker --version

# 2. Clear Docker cache
docker system prune -a

# 3. Rebuild with verbose output
docker build -f Dockerfile.lambda -t your-app . --no-cache
```

#### Issue 3: AWS Credentials Error
**Error Message:**
```
❌ The security token included in the request is invalid
```

**Solution (All Platforms):**
```bash
# Reconfigure AWS credentials
aws configure
# Enter correct Access Key ID and Secret Access Key

# Test credentials
aws sts get-caller-identity
```

#### Issue 4: Lambda Timeout
**Error Message:**
```
❌ Task timed out after 30.00 seconds
```

**Solution - Update `serverless.yml`:**
```yaml
provider:
  timeout: 60        # Increase timeout to 60 seconds
  memorySize: 2048   # Increase memory to 2GB
```

#### Issue 5: Environment Variables Not Working
**Error Message:**
```
❌ Database connection failed
❌ Environment variable DB_HOST is not defined
```

**Windows Solution:**
```powershell
# Check if .env.lambda exists
Get-ChildItem .env.lambda -ErrorAction SilentlyContinue

# Verify file contents
Get-Content .env.lambda
```

**macOS/Linux Solution:**
```bash
# Check if .env.lambda exists
ls -la .env.lambda

# Verify environment loading
export $(cat .env.lambda | grep -v '^#' | xargs)
echo $DB_HOST
```

#### Issue 6: Serverless Framework Version Error
**Error Message:**
```
❌ No version found for 3.38.0
```

**Solution (All Platforms):**
```bash
# Uninstall and reinstall Serverless
npm uninstall -g serverless
npm install -g serverless@3

# Verify version
serverless --version
```

### Expected Log Messages

**✅ Successful Deployment Logs:**
```
🚀 Starting Docker Lambda deployment...
✅ Environment loaded
🔨 Building application...
✅ Build complete!
☁️ Deploying to AWS Lambda...
✅ Service deployed to stack your-app-backend-dev
📋 Getting deployment info...
🎉 Deployment complete!
```

**❌ Failed Deployment Logs:**
```
🚀 Starting Docker Lambda deployment...
❌ .env.lambda file not found!
```
or
```
🔨 Building application...
❌ Build failed!
npm ERR! Missing script: "build"
```

---

## 📊 Step 9: Monitor Your Application

### 9.1 CloudWatch Logs
1. **Go to AWS Console** → **CloudWatch** → **Log Groups**
2. **Find log group:** `/aws/lambda/your-app-backend-dev-api`
3. **Click on log group** to view real-time logs
4. **Expected log entries:**
   ```
   [INFO] Lambda function started
   [INFO] NestJS application initialized
   [INFO] Request: GET /api/health
   [INFO] Response: 200 OK
   ```

### 9.2 Lambda Metrics
1. **Go to AWS Console** → **Lambda** → **Functions**
2. **Select your function:** `your-app-backend-dev-api`
3. **Click "Monitoring" tab**
4. **Key metrics to watch:**
   - **Invocations:** Number of requests
   - **Duration:** Response time (should be < 5 seconds)
   - **Errors:** Should be 0 or very low
   - **Throttles:** Should be 0

### 9.3 Cost Monitoring
1. **Go to AWS Console** → **Billing & Cost Management**
2. **Check Lambda costs** (usually very low for development)
3. **Expected costs:** $0.00 - $5.00/month for development usage

---

## 🌐 Step 10: Custom Domain Setup (Optional)

### 10.1 Install Domain Manager
```bash
npm install --save-dev serverless-domain-manager
```

### 10.2 Update serverless.yml
```yaml
plugins:
  - serverless-domain-manager

custom:
  customDomain:
    domainName: api.your-domain.com
    certificateName: api.your-domain.com
    createRoute53Record: true
    endpointType: 'regional'
    securityPolicy: tls_1_2
```

### 10.3 Deploy Custom Domain

**All Platforms:**
```bash
# 1. Create SSL certificate (one-time)
aws acm request-certificate \
  --domain-name api.your-domain.com \
  --validation-method DNS \
  --region us-east-1

# 2. Create custom domain (one-time)
serverless create_domain --stage dev

# 3. Deploy with custom domain
serverless deploy --stage dev
```

### 10.4 Expected Custom Domain Results

**✅ Successful Custom Domain Setup:**
```
✅ Custom domain created successfully
Domain Name: api.your-domain.com
Target Domain: d-abc123def.execute-api.us-east-1.amazonaws.com
Hosted Zone ID: Z1D633PJN98FT9
```

**Your API will be available at:**
- `https://api.your-domain.com/api/health`
- `https://api.your-domain.com/api/auth/login`

**❌ Common Custom Domain Errors:**
```
Error: Certificate not found for domain api.your-domain.com
```
**Solution:** Create SSL certificate first using AWS Certificate Manager.

---

## ✅ Step 11: Pre-Deployment Checklist

### Platform-Specific Checklist

**Windows Users:**
- [ ] Docker Desktop installed and running
- [ ] AWS CLI installed and configured (`aws --version`)
- [ ] Node.js 18+ installed (`node --version`)
- [ ] PowerShell or Command Prompt available
- [ ] `.env.lambda` file created with all environment variables
- [ ] `src/lambda.ts` handler created
- [ ] `Dockerfile.lambda` created
- [ ] `serverless.yml` configured
- [ ] `deploy.bat` script created

**macOS Users:**
- [ ] Docker Desktop installed and running
- [ ] AWS CLI installed via Homebrew (`aws --version`)
- [ ] Node.js 18+ installed (`node --version`)
- [ ] Terminal access
- [ ] `.env.lambda` file created with all environment variables
- [ ] `src/lambda.ts` handler created
- [ ] `Dockerfile.lambda` created
- [ ] `serverless.yml` configured
- [ ] Scripts executable (`chmod +x *.sh`)

**Linux/Ubuntu Users:**
- [ ] Docker installed and user added to docker group
- [ ] AWS CLI installed (`aws --version`)
- [ ] Node.js 18+ installed (`node --version`)
- [ ] Bash shell access
- [ ] `.env.lambda` file created with all environment variables
- [ ] `src/lambda.ts` handler created
- [ ] `Dockerfile.lambda` created
- [ ] `serverless.yml` configured
- [ ] Scripts executable (`chmod +x *.sh`)

### Universal Verification Commands
```bash
# Test all prerequisites (run on any platform)
node --version          # Should show v18.x.x or higher
npm --version           # Should show 8.x.x or higher
docker --version        # Should show Docker version
aws --version           # Should show AWS CLI version
serverless --version    # Should show Serverless Framework version
```

---

## 🎆 Step 12: Success Summary & Next Steps

### 🎉 Congratulations! You've Successfully Deployed!

Your NestJS application is now running serverless on AWS Lambda across **Windows, macOS, and Linux**!

### What You've Accomplished:
1. ✅ **Cross-platform setup** - Works on Windows, macOS, and Linux
2. ✅ **Lambda-compatible NestJS handler** - Optimized for serverless
3. ✅ **Docker containerization** - Consistent deployment environment
4. ✅ **Serverless Framework configuration** - Infrastructure as code
5. ✅ **One-command deployment** - Platform-specific scripts
6. ✅ **Monitoring and logging setup** - CloudWatch integration
7. ✅ **Comprehensive troubleshooting** - Solutions for common issues

### Your API Endpoints Are Live:
- **Health Check:** `https://YOUR_LAMBDA_URL/api/health`
- **All API Routes:** `https://YOUR_LAMBDA_URL/api/*`
- **Expected Response Time:** < 3 seconds
- **Expected Costs:** $0-5/month for development

### 🚀 Recommended Next Steps:

#### Immediate (Next 24 hours):
- [ ] **Test all API endpoints** thoroughly
- [ ] **Set up monitoring alerts** in CloudWatch
- [ ] **Document your API endpoints** for your team
- [ ] **Update your frontend** to use the Lambda URL

#### Short-term (Next week):
- [ ] **Set up custom domain** (Step 10) for production
- [ ] **Configure environment-specific deployments** (dev/staging/prod)
- [ ] **Set up automated backups** for your database
- [ ] **Implement proper error handling** and logging

#### Long-term (Next month):
- [ ] **Configure CI/CD pipeline** with GitHub Actions or similar
- [ ] **Set up performance monitoring** and optimization
- [ ] **Implement security best practices** (WAF, rate limiting)
- [ ] **Cost optimization** and usage monitoring

### 📊 Performance Expectations:
- **Cold Start:** 2-5 seconds (first request after idle)
- **Warm Requests:** 100-500ms
- **Concurrent Users:** 1000+ (with proper scaling)
- **Monthly Free Tier:** 1M requests, 400,000 GB-seconds

### 🌐 Platform-Specific Resources:

**Windows Developers:**
- Use **PowerShell ISE** or **VS Code** for script editing
- Consider **WSL2** for Linux-like development experience
- **Docker Desktop** provides excellent Windows integration

**macOS Developers:**
- **Homebrew** makes package management easy
- **iTerm2** provides better terminal experience
- **Docker Desktop** integrates well with macOS

**Linux Developers:**
- **Native Docker** support provides best performance
- **Bash scripting** works out of the box
- **Package managers** (apt, yum) for easy tool installation

---

## 🆘 Need Help?

### Quick Support Options:
1. **Check Step 8 (Troubleshooting)** for common issues
2. **Review expected responses** in Step 6
3. **Verify your checklist** in Step 11
4. **Check AWS CloudWatch logs** for detailed error information

### Community Resources:
- **Serverless Framework Docs:** https://serverless.com/framework/docs/
- **AWS Lambda Docs:** https://docs.aws.amazon.com/lambda/
- **NestJS Docs:** https://docs.nestjs.com/

**🎉 Your serverless NestJS API is now live and ready for production use!**
