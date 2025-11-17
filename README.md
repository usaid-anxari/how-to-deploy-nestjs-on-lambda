# NestJS Docker Lambda Deployment - Simple Guide 🐳

> **Easy Step-by-Step Guide**: Deploy your NestJS backend to AWS Lambda using Docker containers in under 30 minutes.

## 📋 What You'll Need

- AWS Account with CLI configured
- Docker installed
- Node.js 18+ installed
- Your NestJS project ready

---

## 🚀 Step 1: Prepare Your Project

### 1.1 Install Required Dependencies
```bash
cd your-nestjs-project
npm install --save serverless-http@3.2.0
npm install -g serverless@3.38.0
```

### 1.2 Create Lambda Handler
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

## 🐳 Step 2: Create Docker Files

### 2.1 Lambda Dockerfile
**Create file: `Dockerfile.lambda`**
```dockerfile
FROM public.ecr.aws/lambda/nodejs:18

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

### 2.2 Build Script
**Create file: `build-lambda.sh`**
```bash
#!/bin/bash
echo "🔨 Building NestJS for Lambda..."
npm run build
echo "✅ Build complete!"
```

---

## ⚙️ Step 3: Configure Serverless

### 3.1 Create Serverless Config
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

### 3.2 Environment Variables
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

## 🚀 Step 4: Deploy to AWS

### 4.1 Configure AWS CLI
```bash
# Install AWS CLI if not installed
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure credentials
aws configure
# Enter your AWS Access Key ID
# Enter your AWS Secret Access Key
# Default region: us-east-1
# Default output format: json
```

### 4.2 One-Click Deploy Script
**Create file: `deploy.sh`**
```bash
#!/bin/bash

echo "🚀 Starting Docker Lambda deployment..."

# Load environment variables
if [ -f ".env.lambda" ]; then
    export $(cat .env.lambda | grep -v '^#' | xargs)
    echo "✅ Environment loaded"
else
    echo "❌ .env.lambda file not found!"
    exit 1
fi

# Build application
echo "🔨 Building application..."
npm run build

# Deploy with Serverless
echo "☁️ Deploying to AWS Lambda..."
serverless deploy --stage dev

# Get URL
echo "📋 Getting deployment info..."
serverless info --stage dev

echo "🎉 Deployment complete!"
```

### 4.3 Make Script Executable and Deploy
```bash
# Make scripts executable
chmod +x build-lambda.sh deploy.sh

# Deploy your application
./deploy.sh
```

---

## 🧪 Step 5: Test Your Deployment

### 5.1 Get Your Lambda URL
```bash
# Get the Lambda URL from deployment output
serverless info --stage dev
```

### 5.2 Test API Endpoints
```bash
# Replace YOUR_LAMBDA_URL with actual URL from step 5.1
curl https://YOUR_LAMBDA_URL/api/health

# Test with data
curl -X POST https://YOUR_LAMBDA_URL/api/your-endpoint \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'
```

---

## 🔧 Step 6: Common Commands

### Development Commands
```bash
# View logs
serverless logs -f api --tail --stage dev

# Remove deployment
serverless remove --stage dev

# Deploy to production
serverless deploy --stage prod

# Package without deploying
serverless package --stage dev
```

### Docker Commands
```bash
# Build Docker image locally
docker build -f Dockerfile.lambda -t your-app .

# Test Docker image locally
docker run -p 9000:8080 your-app

# Test Lambda function
curl "http://localhost:9000/2015-03-31/functions/function/invocations" \
  -d '{"httpMethod":"GET","path":"/api/health"}'
```

---

## 🛠️ Troubleshooting

### Issue 1: Build Fails
```bash
# Clean and rebuild
rm -rf node_modules dist
npm install
npm run build
```

### Issue 2: Docker Build Fails
```bash
# Clear Docker cache
docker system prune -a

# Rebuild with verbose output
docker build -f Dockerfile.lambda -t your-app . --no-cache
```

### Issue 3: Lambda Timeout
**Update `serverless.yml`:**
```yaml
provider:
  timeout: 60        # Increase timeout
  memorySize: 2048   # Increase memory
```

### Issue 4: Environment Variables Not Working
```bash
# Check if .env.lambda exists
ls -la .env.lambda

# Verify environment loading
export $(cat .env.lambda | grep -v '^#' | xargs)
echo $DB_HOST
```

---

## 📊 Step 7: Monitor Your Application

### 7.1 CloudWatch Logs
1. Go to AWS Console → CloudWatch → Log Groups
2. Find `/aws/lambda/your-app-backend-dev-api`
3. View real-time logs

### 7.2 Lambda Metrics
1. Go to AWS Console → Lambda → Functions
2. Select your function
3. Check Monitoring tab

---

## 🌐 Step 8: Custom Domain (Optional)

### 8.1 Install Domain Manager
```bash
npm install --save-dev serverless-domain-manager
```

### 8.2 Update serverless.yml
```yaml
plugins:
  - serverless-domain-manager

custom:
  customDomain:
    domainName: api.your-domain.com
    certificateName: api.your-domain.com
    createRoute53Record: true
```

### 8.3 Deploy Custom Domain
```bash
# Create domain
serverless create_domain --stage dev

# Deploy with domain
serverless deploy --stage dev
```

---

## ✅ Quick Checklist

Before deploying, make sure you have:

- [ ] AWS CLI configured with proper permissions
- [ ] Docker installed and running
- [ ] `.env.lambda` file with your environment variables
- [ ] `src/lambda.ts` handler created
- [ ] `Dockerfile.lambda` created
- [ ] `serverless.yml` configured
- [ ] All scripts are executable (`chmod +x *.sh`)

---

## 🎯 Summary

You've successfully deployed your NestJS application to AWS Lambda using Docker! Here's what you accomplished:

1. ✅ Created a Lambda-compatible NestJS handler
2. ✅ Built a Docker container for Lambda
3. ✅ Configured Serverless Framework
4. ✅ Deployed to AWS with one command
5. ✅ Set up monitoring and logging

**Your API is now running serverless on AWS Lambda!** 🎉

### Next Steps:
- Set up custom domain for production
- Configure CI/CD pipeline
- Add monitoring and alerts
- Optimize performance and costs

---

**Need help?** Check the troubleshooting section or refer to the complete deployment guide for advanced configurations.
