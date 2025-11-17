# Complete NestJS Lambda Deployment Guide 🚀

> **Team Guide**: Step-by-step instructions to deploy NestJS applications to AWS Lambda with Docker support, monitoring, and production best practices.

## 📋 Table of Contents
1. [Prerequisites & Setup](#prerequisites--setup)
2. [Method 1: Direct Lambda Deployment](#method-1-direct-lambda-deployment)
3. [Method 2: Docker Container Deployment](#method-2-docker-container-deployment)
4. [Environment Configuration](#environment-configuration)
5. [Deployment Scripts](#deployment-scripts)
6. [Testing & Monitoring](#testing--monitoring)
7. [Custom Domain Integration](#custom-domain-integration)
8. [Troubleshooting](#troubleshooting)
9. [Team Workflow](#team-workflow)

---

## 📋 Prerequisites & Setup

### 1. Install Required Tools
```bash
# Install Node.js 18+
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install Serverless Framework
npm install -g serverless@3.38.0
```

### 2. AWS Configuration
```bash
# Configure AWS credentials
aws configure
# AWS Access Key ID: [Your Access Key]
# AWS Secret Access Key: [Your Secret Key]
# Default region name: us-east-1
# Default output format: json

# Verify configuration
aws sts get-caller-identity
```

### 3. Required AWS IAM Permissions
Ensure your AWS user has these policies:
- `AWSLambdaFullAccess`
- `IAMFullAccess`
- `AmazonS3FullAccess`
- `AmazonVPCFullAccess`
- `CloudFormationFullAccess`
- `AmazonAPIGatewayAdministrator`
- `AmazonEC2ContainerRegistryFullAccess` (for Docker)

---

## 🚀 Method 1: Direct Lambda Deployment

### Step 1: Update package.json
```json
{
  "name": "your-app-backend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "start:dev": "nest start --watch",
    "build": "nest build",
    "build:lambda": "nest build",
    "start": "node dist/main.js",
    "deploy:lambda": "npm run build:lambda && serverless deploy",
    "deploy:lambda:prod": "npm run build:lambda && serverless deploy --stage prod",
    "package:lambda": "npm run build:lambda && serverless package",
    "logs:lambda": "serverless logs -f api --tail",
    "remove:lambda": "serverless remove",
    "docker:build": "docker build -t your-app-backend .",
    "docker:run": "docker run -p 4000:4000 your-app-backend"
  },
  "dependencies": {
    "@aws-sdk/client-s3": "^3.888.0",
    "@aws-sdk/s3-request-presigner": "^3.888.0",
    "@nestjs/axios": "^4.0.1",
    "@nestjs/common": "^10.0.0",
    "@nestjs/config": "^4.0.2",
    "@nestjs/core": "^10.0.0",
    "@nestjs/passport": "^11.0.5",
    "@nestjs/platform-express": "^10.4.20",
    "@nestjs/swagger": "^7.4.2",
    "@nestjs/typeorm": "^10.0.0",
    "serverless-http": "^3.2.0",
    "compression": "^1.8.1",
    "express": "^4.18.0",
    "typeorm": "^0.3.18",
    "pg": "^8.10.0",
    "stripe": "^18.5.0",
    "auth0": "^5.0.0",
    "bcrypt": "^6.0.0",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.0",
    "dotenv": "^16.0.0",
    "googleapis": "^160.0.0",
    "multer": "^2.0.2",
    "passport": "^0.6.0",
    "passport-jwt": "^4.0.1"
  },
  "devDependencies": {
    "serverless-esbuild": "^1.55.2",
    "@types/node": "^24.5.2",
    "@types/bcrypt": "^6.0.0",
    "@types/multer": "^1.4.7"
  }
}
```

### Step 2: Install Lambda Dependencies
```bash
# Install Lambda-specific packages
npm install --save serverless-http@3.2.0
npm install --save-dev serverless-esbuild@1.55.2
```

### Step 3: Create Lambda Handler
**File: `src/lambda.ts`**
```typescript
import { NestFactory } from '@nestjs/core';
import { ExpressAdapter } from '@nestjs/platform-express';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import * as express from 'express';
import * as compression from 'compression';
import serverlessExpress from 'serverless-http';

let cachedServer;

async function bootstrap() {
  if (!cachedServer) {
    const expressApp = express();
    const app = await NestFactory.create(AppModule, new ExpressAdapter(expressApp));

    // Enable compression
    app.use(compression());

    // Enable CORS
    app.enableCors({
      origin: [
        'http://localhost:3000',
        'https://your-domain.com',
        'https://www.your-domain.com',
        'https://api.your-domain.com'
      ],
      credentials: true,
    });

    // Global validation pipe
    app.useGlobalPipes(new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }));

    // Swagger setup (only in development)
    if (process.env.NODE_ENV !== 'production') {
      const config = new DocumentBuilder()
        .setTitle('Your App API')
        .setDescription('Your application API description')
        .setVersion('1.0')
        .addBearerAuth()
        .build();
      const document = SwaggerModule.createDocument(app, config);
      SwaggerModule.setup('api-docs', app, document);
    }

    // Set global prefix
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

### Step 4: Create Serverless Configuration
**File: `serverless.yml`**
```yaml
service: your-app-backend

frameworkVersion: '3'

provider:
  name: aws
  runtime: nodejs18.x
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
    AUTH0_DOMAIN: ${env:AUTH0_DOMAIN}
    AUTH0_AUDIENCE: ${env:AUTH0_AUDIENCE}
    AWS_S3_BUCKET: ${env:AWS_S3_BUCKET}
    AWS_REGION: ${env:AWS_REGION}
    STRIPE_SECRET_KEY: ${env:STRIPE_SECRET_KEY}
    STRIPE_WEBHOOK_SECRET: ${env:STRIPE_WEBHOOK_SECRET}
    STRIPE_PRICE_TIER_FREE: ${env:STRIPE_PRICE_TIER_FREE}
    STRIPE_PRICE_TIER_STARTER: ${env:STRIPE_PRICE_TIER_STARTER}
    STRIPE_PRICE_TIER_ENTERPRISE: ${env:STRIPE_PRICE_TIER_ENTERPRISE}
    STRIPE_PRICE_TIER_PROFESSIONAL: ${env:STRIPE_PRICE_TIER_PROFESSIONAL}
    GOOGLE_CLIENT_ID: ${env:GOOGLE_CLIENT_ID}
    GOOGLE_CLIENT_SECRET: ${env:GOOGLE_CLIENT_SECRET}
    GOOGLE_API_KEY: ${env:GOOGLE_API_KEY}
    FRONTEND_URL: ${env:FRONTEND_URL}
    BACKEND_URL: ${env:BACKEND_URL}

  # IAM Role for Lambda
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - s3:GetObject
            - s3:PutObject
            - s3:DeleteObject
          Resource: "arn:aws:s3:::your-s3-bucket-name/*"
        - Effect: Allow
          Action:
            - rds:DescribeDBInstances
          Resource: "*"
        - Effect: Allow
          Action:
            - ecr:GetAuthorizationToken
            - ecr:BatchCheckLayerAvailability
            - ecr:GetDownloadUrlForLayer
            - ecr:BatchGetImage
          Resource: "*"

  # VPC Configuration (if needed for RDS)
  vpc:
    securityGroupIds:
      - ${env:LAMBDA_SECURITY_GROUP_ID}
    subnetIds:
      - ${env:SUBNET_ID_1}
      - ${env:SUBNET_ID_2}

functions:
  api:
    handler: dist/lambda.handler
    events:
      - httpApi:
          path: /{proxy+}
          method: ANY
      - httpApi:
          path: /
          method: ANY
    url: true

plugins:
  - serverless-esbuild

custom:
  esbuild:
    bundle: true
    minify: false
    sourcemap: false
    exclude:
      - aws-sdk
      - pg-native
    target: node18
    define:
      'require.resolve': undefined
    platform: node
    concurrency: 10
```

---

## 🐳 Method 2: Docker Container Deployment

### Step 1: Create Dockerfile
**File: `Dockerfile`**
```dockerfile
# Multi-stage build for production
FROM node:18-alpine AS builder

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && npm cache clean --force

# Copy source code
COPY . .

# Build application
RUN npm run build

# Production stage
FROM node:18-alpine AS production

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

# Create app user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nestjs -u 1001

# Set working directory
WORKDIR /app

# Copy built application
COPY --from=builder --chown=nestjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nestjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nestjs:nodejs /app/package*.json ./

# Switch to non-root user
USER nestjs

# Expose port
EXPOSE 4000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# Start application
CMD ["dumb-init", "node", "dist/main.js"]
```

### Step 2: Create Docker Compose for Development
**File: `docker-compose.yml`**
```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "4000:4000"
    environment:
      - NODE_ENV=development
      - DB_HOST=postgres
      - DB_USERNAME=postgres
      - DB_PASSWORD=password
      - DB_NAME=your_app_name
    depends_on:
      - postgres
      - redis
    volumes:
      - ./src:/app/src
    networks:
      - app-network

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=your_app_name
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

### Step 3: Create Health Check
**File: `healthcheck.js`**
```javascript
const http = require('http');

const options = {
  host: 'localhost',
  port: 4000,
  path: '/api/health',
  timeout: 2000
};

const request = http.request(options, (res) => {
  console.log(`Health check status: ${res.statusCode}`);
  if (res.statusCode === 200) {
    process.exit(0);
  } else {
    process.exit(1);
  }
});

request.on('error', (err) => {
  console.log('Health check failed:', err.message);
  process.exit(1);
});

request.end();
```

### Step 4: Docker Lambda Configuration
**File: `serverless-docker.yml`**
```yaml
service: your-app-backend-docker

frameworkVersion: '3'

provider:
  name: aws
  region: us-east-1
  stage: ${opt:stage, 'dev'}
  timeout: 30
  memorySize: 1024
  
  # Use container image
  ecr:
    images:
      your-app:
        path: ./
        file: Dockerfile.lambda

functions:
  api:
    image:
      name: your-app
    events:
      - httpApi:
          path: /{proxy+}
          method: ANY
      - httpApi:
          path: /
          method: ANY
    url: true
    environment:
      NODE_ENV: production
      DB_HOST: ${env:DB_HOST}
      DB_USERNAME: ${env:DB_USERNAME}
      DB_PASSWORD: ${env:DB_PASSWORD}
      DB_NAME: ${env:DB_NAME}
      # ... other environment variables
```

### Step 5: Lambda-specific Dockerfile
**File: `Dockerfile.lambda`**
```dockerfile
FROM public.ecr.aws/lambda/nodejs:18

# Copy package files
COPY package*.json ${LAMBDA_TASK_ROOT}/

# Install dependencies
RUN npm ci --only=production

# Copy built application
COPY dist/ ${LAMBDA_TASK_ROOT}/dist/
COPY src/lambda.ts ${LAMBDA_TASK_ROOT}/

# Build lambda handler
RUN npm run build

# Set the CMD to your handler
CMD ["dist/lambda.handler"]
```

---

## 🔐 Environment Configuration

### File: `.env.lambda`
```env
# Database Configuration
DB_HOST=your-rds-endpoint.region.rds.amazonaws.com
DB_PORT=5432
DB_USERNAME=your_db_username
DB_PASSWORD=your_secure_password
DB_NAME=your_database_name

# Auth0 Configuration
AUTH0_DOMAIN=https://your-auth0-domain.auth0.com
AUTH0_AUDIENCE=https://your-auth0-domain.auth0.com/api/v2/

# AWS Configuration
AWS_S3_BUCKET=your-s3-bucket-name
AWS_REGION=us-east-1
AWS_DOMAIN_URL=https://your-s3-bucket-name.s3.amazonaws.com

# Stripe Configuration
STRIPE_SECRET_KEY=sk_test_your_stripe_secret_key
STRIPE_WEBHOOK_SECRET=whsec_your_stripe_webhook_secret
STRIPE_PRICE_TIER_FREE=price_your_free_tier_price_id
STRIPE_PRICE_TIER_STARTER=price_your_starter_tier_price_id
STRIPE_PRICE_TIER_ENTERPRISE=price_your_enterprise_tier_price_id
STRIPE_PRICE_TIER_PROFESSIONAL=price_your_professional_tier_price_id

# VPC Configuration
LAMBDA_SECURITY_GROUP_ID=sg-your_security_group_id
SUBNET_ID_1=subnet-your_subnet_id_1
SUBNET_ID_2=subnet-your_subnet_id_2

# Google Configuration
GOOGLE_CLIENT_ID=your_google_client_id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_API_KEY=your_google_api_key

# Application URLs
NODE_ENV=production
FRONTEND_URL=https://your-frontend-domain.com
BACKEND_URL=https://your-backend-domain.com
```

---

## 📜 Deployment Scripts

### Script 1: Direct Lambda Deployment
**File: `deploy-lambda.sh`**
```bash
#!/bin/bash

set -e  # Exit on any error

echo "🚀 Starting Lambda deployment..."

# Check prerequisites
command -v aws >/dev/null 2>&1 || { echo "❌ AWS CLI not installed"; exit 1; }
command -v serverless >/dev/null 2>&1 || { echo "❌ Serverless not installed"; exit 1; }

# Check environment file
if [ ! -f ".env.lambda" ]; then
    echo "❌ .env.lambda file not found!"
    echo "Please create .env.lambda with your environment variables"
    exit 1
fi

# Load environment variables
echo "📋 Loading environment variables..."
export $(cat .env.lambda | grep -v '^#' | xargs)

# Verify AWS credentials
echo "🔐 Verifying AWS credentials..."
aws sts get-caller-identity > /dev/null || { echo "❌ AWS credentials not configured"; exit 1; }

# Install dependencies
echo "📦 Installing dependencies..."
npm ci

# Build application
echo "🔨 Building NestJS application..."
npm run build:lambda

# Verify build
if [ ! -f "dist/lambda.js" ]; then
    echo "❌ Build failed! lambda.js not found in dist/"
    exit 1
fi

# Deploy to Lambda
echo "☁️ Deploying to AWS Lambda..."
serverless deploy --stage dev --verbose

# Get deployment info
echo "📋 Getting deployment information..."
serverless info --stage dev

echo "✅ Deployment completed successfully!"
echo "🌐 Your API is now available at the Lambda URL above"
```

### Script 2: Docker Lambda Deployment
**File: `deploy-docker-lambda.sh`**
```bash
#!/bin/bash

set -e

echo "🐳 Starting Docker Lambda deployment..."

# Check prerequisites
command -v docker >/dev/null 2>&1 || { echo "❌ Docker not installed"; exit 1; }
command -v aws >/dev/null 2>&1 || { echo "❌ AWS CLI not installed"; exit 1; }

# Load environment
export $(cat .env.lambda | grep -v '^#' | xargs)

# Get AWS account ID
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
IMAGE_NAME="your-app-backend"
IMAGE_TAG="latest"

echo "📦 Building Docker image..."
docker build -f Dockerfile.lambda -t ${IMAGE_NAME}:${IMAGE_TAG} .

# Create ECR repository if it doesn't exist
echo "🏗️ Creating ECR repository..."
aws ecr describe-repositories --repository-names ${IMAGE_NAME} --region ${AWS_REGION} || \
aws ecr create-repository --repository-name ${IMAGE_NAME} --region ${AWS_REGION}

# Login to ECR
echo "🔐 Logging into ECR..."
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}

# Tag and push image
echo "📤 Pushing image to ECR..."
docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
docker push ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}

# Deploy with Serverless
echo "☁️ Deploying to Lambda..."
serverless deploy --config serverless-docker.yml --stage dev

echo "✅ Docker Lambda deployment completed!"
```

### Script 3: Development Setup
**File: `setup-dev.sh`**
```bash
#!/bin/bash

echo "🛠️ Setting up development environment..."

# Install dependencies
echo "📦 Installing Node.js dependencies..."
npm install

# Setup Docker environment
echo "🐳 Starting Docker services..."
docker-compose up -d postgres redis

# Wait for services
echo "⏳ Waiting for services to start..."
sleep 10

# Run migrations
echo "🗄️ Running database migrations..."
npm run migrate:run

# Start development server
echo "🚀 Starting development server..."
npm run start:dev
```

---

## 🔍 Testing & Monitoring

### Testing Script
**File: `test-deployment.sh`**
```bash
#!/bin/bash

# Get Lambda URL
LAMBDA_URL=$(serverless info --stage dev | grep "HttpApi" | awk '{print $2}')

if [ -z "$LAMBDA_URL" ]; then
    echo "❌ Could not get Lambda URL"
    exit 1
fi

echo "🧪 Testing Lambda deployment at: $LAMBDA_URL"

# Test health endpoint
echo "Testing health endpoint..."
HEALTH_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" "${LAMBDA_URL}/api/health")
if [ "$HEALTH_RESPONSE" = "200" ]; then
    echo "✅ Health check passed"
else
    echo "❌ Health check failed (HTTP $HEALTH_RESPONSE)"
fi

# Test API documentation
echo "Testing API docs..."
DOCS_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" "${LAMBDA_URL}/api-docs")
if [ "$DOCS_RESPONSE" = "200" ]; then
    echo "✅ API docs accessible"
else
    echo "⚠️ API docs not accessible (expected in production)"
fi

# Test CORS
echo "Testing CORS..."
CORS_RESPONSE=$(curl -s -H "Origin: https://your-domain.com" -H "Access-Control-Request-Method: GET" -H "Access-Control-Request-Headers: X-Requested-With" -X OPTIONS "${LAMBDA_URL}/api/health")
echo "CORS test completed"

echo "🎉 Testing completed!"
echo "Lambda URL: $LAMBDA_URL"
```

### Monitoring Setup
**File: `setup-monitoring.sh`**
```bash
#!/bin/bash

echo "📊 Setting up monitoring..."

# Create CloudWatch dashboard
aws cloudwatch put-dashboard --dashboard-name "YourApp-Lambda" --dashboard-body '{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Lambda", "Duration", "FunctionName", "your-app-backend-dev-api"],
          ["AWS/Lambda", "Errors", "FunctionName", "your-app-backend-dev-api"],
          ["AWS/Lambda", "Invocations", "FunctionName", "your-app-backend-dev-api"]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "Lambda Metrics"
      }
    }
  ]
}'

# Create CloudWatch alarms
aws cloudwatch put-metric-alarm \
  --alarm-name "YourApp-Lambda-Errors" \
  --alarm-description "Lambda function errors" \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --statistic Sum \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=FunctionName,Value=your-app-backend-dev-api \
  --evaluation-periods 2

echo "✅ Monitoring setup completed"
```

---

## 🌐 Custom Domain Integration

### Step 1: Create Custom Domain Certificate
```bash
# Request SSL certificate
aws acm request-certificate \
  --domain-name api.your-domain.com \
  --validation-method DNS \
  --region us-east-1
```

### Step 2: Update Serverless for Custom Domain
**Add to `serverless.yml`:**
```yaml
plugins:
  - serverless-esbuild
  - serverless-domain-manager

custom:
  customDomain:
    domainName: api.your-domain.com
    certificateName: api.your-domain.com
    createRoute53Record: true
    endpointType: 'regional'
    securityPolicy: tls_1_2
    apiType: http
```

### Step 3: Deploy Custom Domain
```bash
# Create custom domain
serverless create_domain --stage dev

# Deploy with custom domain
serverless deploy --stage dev
```

---

## 🛠️ Troubleshooting

### Common Issues & Solutions

#### 1. VPC Timeout Issues
```yaml
# Remove VPC config if not needed for RDS access
# vpc:
#   securityGroupIds:
#     - ${env:LAMBDA_SECURITY_GROUP_ID}
#   subnetIds:
#     - ${env:SUBNET_ID_1}
#     - ${env:SUBNET_ID_2}
```

#### 2. Memory/Timeout Issues
```yaml
provider:
  memorySize: 2048  # Increase memory
  timeout: 60       # Increase timeout
```

#### 3. Docker Build Issues
```bash
# Clear Docker cache
docker system prune -a

# Rebuild with no cache
docker build --no-cache -f Dockerfile.lambda -t your-app-backend .
```

#### 4. Database Connection Issues
```bash
# Test database connection
psql "postgresql://your_db_username:your_password@your-rds-endpoint.region.rds.amazonaws.com:5432/your_database"

# Check security groups
aws ec2 describe-security-groups --group-ids sg-your_security_group_id
```

### Debugging Commands
```bash
# View Lambda logs
serverless logs -f api --tail --stage dev

# View CloudWatch logs
aws logs tail /aws/lambda/your-app-backend-dev-api --follow

# Test Lambda locally
serverless offline start

# Package without deploying
serverless package --stage dev

# Remove deployment
serverless remove --stage dev
```

---

## 👥 Team Workflow

### Development Workflow
1. **Local Development**
   ```bash
   # Setup development environment
   ./setup-dev.sh
   
   # Make changes and test locally
   npm run start:dev
   ```

2. **Testing**
   ```bash
   # Run tests
   npm test
   
   # Test Docker build
   docker-compose up --build
   ```

3. **Deployment**
   ```bash
   # Deploy to development
   ./deploy-lambda.sh
   
   # Test deployment
   ./test-deployment.sh
   
   # Deploy to production
   serverless deploy --stage prod
   ```

### Environment Management
- **Development**: Local Docker setup
- **Staging**: Lambda with dev stage
- **Production**: Lambda with prod stage

### Code Review Checklist
- [ ] Environment variables updated
- [ ] Docker builds successfully
- [ ] Tests pass
- [ ] Lambda deployment works
- [ ] API endpoints tested
- [ ] Monitoring configured
- [ ] Documentation updated

### Deployment Checklist
- [ ] AWS credentials configured
- [ ] Environment files created
- [ ] Dependencies installed
- [ ] Build successful
- [ ] Tests passing
- [ ] Lambda deployed
- [ ] Custom domain configured
- [ ] Monitoring active
- [ ] Team notified

---

## 📚 Additional Resources

### Useful Commands Reference
```bash
# AWS CLI
aws lambda list-functions
aws logs describe-log-groups
aws ecr describe-repositories

# Docker
docker ps
docker logs <container-id>
docker exec -it <container-id> /bin/sh

# Serverless
serverless info
serverless invoke -f api
serverless metrics

# Node.js/NPM
npm audit
npm outdated
npm run build
```

### Performance Optimization
- Use Lambda provisioned concurrency for consistent performance
- Implement connection pooling for database
- Use Redis for caching
- Optimize Docker image size
- Monitor cold start times

### Security Best Practices
- Use IAM roles instead of access keys
- Enable VPC for database access
- Implement proper CORS policies
- Use environment variables for secrets
- Regular security audits

---

**🎉 Your NestJS application is now ready for Lambda deployment!**

> **Next Steps**: Follow the deployment method that best fits your needs, test thoroughly, and set up monitoring for production use.
