# Cloud Deployment (AWS & GCP)

## Overview

Cloud deployment enables applications to run on scalable, managed infrastructure from providers like AWS and Google Cloud Platform. Cloud platforms offer on-demand resources, high availability, automatic scaling, and pay-as-you-go pricing, eliminating the need for physical infrastructure management.

**Key Services:**

- **Compute**: EC2, ECS, Lambda (AWS) | Compute Engine, Cloud Run, Cloud Functions (GCP)
- **Databases**: RDS, DynamoDB (AWS) | Cloud SQL, Firestore (GCP)
- **Storage**: S3 (AWS) | Cloud Storage (GCP)
- **Networking**: VPC, Load Balancers, CloudFront (AWS) | VPC, Load Balancing, Cloud CDN (GCP)
- **Container Orchestration**: EKS (AWS) | GKE (GCP)

## Practical Use Cases

### 1. **Scalable Web Applications**

Handle variable traffic loads

- E-commerce platforms
- SaaS applications
- Content delivery
- API services

### 2. **Microservices Architecture**

Deploy independent services

- Container orchestration
- Service mesh
- API gateways
- Inter-service communication

### 3. **Serverless Applications**

Event-driven computing

- API backends
- Data processing
- Scheduled tasks
- Webhook handlers

### 4. **Global Distribution**

Low-latency worldwide access

- Multi-region deployment
- CDN integration
- Geographic load balancing
- Disaster recovery

### 5. **Data Analytics & ML**

Process large datasets

- Data lakes
- Real-time analytics
- Machine learning pipelines
- Big data processing

## AWS Deployment

### 1. Deploy to EC2 with User Data

```bash
#!/bin/bash
# user-data.sh - EC2 instance initialization script

# Update system
yum update -y

# Install Docker
amazon-linux-extras install docker -y
systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Clone application
cd /home/ec2-user
git clone https://github.com/username/myapp.git
cd myapp

# Create environment file
cat > .env << EOF
NODE_ENV=production
DB_HOST=${db_host}
DB_PASSWORD=${db_password}
REDIS_HOST=${redis_host}
EOF

# Start application
docker-compose up -d

# Setup CloudWatch agent
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
rpm -U ./amazon-cloudwatch-agent.rpm
```

### 2. Deploy to ECS (Elastic Container Service)

```json
// task-definition.json
{
  "family": "myapp-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "myapp",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -f http://localhost:3000/health || exit 1"
        ],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

```bash
# Deploy to ECS
aws ecs register-task-definition --cli-input-json file://task-definition.json

aws ecs create-service \
  --cluster myapp-cluster \
  --service-name myapp-service \
  --task-definition myapp-task:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc123],securityGroups=[sg-xyz789],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789:targetgroup/myapp/abc123,containerName=myapp,containerPort=3000"
```

### 3. AWS Lambda Deployment

```typescript
// lambda/handler.ts
import { APIGatewayProxyHandler } from "aws-lambda";

export const handler: APIGatewayProxyHandler = async (event) => {
  console.log("Event:", JSON.stringify(event, null, 2));

  const body = JSON.parse(event.body || "{}");

  try {
    // Your application logic
    const result = await processRequest(body);

    return {
      statusCode: 200,
      headers: {
        "Content-Type": "application/json",
        "Access-Control-Allow-Origin": "*",
      },
      body: JSON.stringify(result),
    };
  } catch (error) {
    console.error("Error:", error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: "Internal server error" }),
    };
  }
};

async function processRequest(data: any) {
  // Implementation
  return { success: true, data };
}
```

```yaml
# serverless.yml
service: myapp-api

provider:
  name: aws
  runtime: nodejs18.x
  region: us-east-1
  memorySize: 512
  timeout: 30
  environment:
    NODE_ENV: production
    DB_HOST: ${ssm:/myapp/db-host}
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:Query
            - dynamodb:GetItem
            - dynamodb:PutItem
          Resource: "arn:aws:dynamodb:us-east-1:*:table/myapp-*"

functions:
  api:
    handler: dist/handler.handler
    events:
      - http:
          path: /{proxy+}
          method: ANY
          cors: true
    vpc:
      securityGroupIds:
        - sg-abc123
      subnetIds:
        - subnet-xyz789

  scheduled:
    handler: dist/scheduled.handler
    events:
      - schedule: rate(1 hour)

resources:
  Resources:
    MyAppTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: myapp-data
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
```

### 4. AWS CDK Infrastructure

```typescript
// lib/myapp-stack.ts
import * as cdk from "aws-cdk-lib";
import * as ec2 from "aws-cdk-lib/aws-ec2";
import * as ecs from "aws-cdk-lib/aws-ecs";
import * as ecs_patterns from "aws-cdk-lib/aws-ecs-patterns";
import * as rds from "aws-cdk-lib/aws-rds";
import { Construct } from "constructs";

export class MyAppStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // VPC
    const vpc = new ec2.Vpc(this, "MyAppVPC", {
      maxAzs: 2,
      natGateways: 1,
    });

    // RDS Database
    const database = new rds.DatabaseInstance(this, "MyAppDatabase", {
      engine: rds.DatabaseInstanceEngine.postgres({
        version: rds.PostgresEngineVersion.VER_15,
      }),
      instanceType: ec2.InstanceType.of(
        ec2.InstanceClass.T3,
        ec2.InstanceSize.MICRO
      ),
      vpc,
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      allocatedStorage: 20,
      maxAllocatedStorage: 100,
      databaseName: "myapp",
      credentials: rds.Credentials.fromGeneratedSecret("postgres"),
      backupRetention: cdk.Duration.days(7),
      deleteAutomatedBackups: false,
      removalPolicy: cdk.RemovalPolicy.SNAPSHOT,
    });

    // ECS Cluster
    const cluster = new ecs.Cluster(this, "MyAppCluster", {
      vpc,
      containerInsights: true,
    });

    // Load Balanced Fargate Service
    const fargateService =
      new ecs_patterns.ApplicationLoadBalancedFargateService(
        this,
        "MyAppService",
        {
          cluster,
          cpu: 512,
          memoryLimitMiB: 1024,
          desiredCount: 2,
          taskImageOptions: {
            image: ecs.ContainerImage.fromAsset("./"),
            containerPort: 3000,
            environment: {
              NODE_ENV: "production",
              DB_HOST: database.dbInstanceEndpointAddress,
            },
            secrets: {
              DB_PASSWORD: ecs.Secret.fromSecretsManager(database.secret!),
            },
          },
          publicLoadBalancer: true,
        }
      );

    // Allow ECS to connect to RDS
    database.connections.allowFrom(fargateService.service, ec2.Port.tcp(5432));

    // Auto Scaling
    const scaling = fargateService.service.autoScaleTaskCount({
      minCapacity: 2,
      maxCapacity: 10,
    });

    scaling.scaleOnCpuUtilization("CpuScaling", {
      targetUtilizationPercent: 70,
    });

    scaling.scaleOnMemoryUtilization("MemoryScaling", {
      targetUtilizationPercent: 80,
    });

    // Outputs
    new cdk.CfnOutput(this, "LoadBalancerDNS", {
      value: fargateService.loadBalancer.loadBalancerDnsName,
    });

    new cdk.CfnOutput(this, "DatabaseEndpoint", {
      value: database.dbInstanceEndpointAddress,
    });
  }
}
```

## Google Cloud Platform Deployment

### 1. Deploy to Cloud Run

```bash
# Build and deploy
gcloud builds submit --tag gcr.io/PROJECT_ID/myapp
gcloud run deploy myapp \
  --image gcr.io/PROJECT_ID/myapp \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --memory 512Mi \
  --cpu 1 \
  --min-instances 1 \
  --max-instances 10 \
  --port 3000 \
  --set-env-vars NODE_ENV=production \
  --set-secrets DB_PASSWORD=db-password:latest
```

```yaml
# cloudbuild.yaml
steps:
  # Build Docker image
  - name: "gcr.io/cloud-builders/docker"
    args:
      - "build"
      - "-t"
      - "gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA"
      - "-t"
      - "gcr.io/$PROJECT_ID/myapp:latest"
      - "."

  # Push to Container Registry
  - name: "gcr.io/cloud-builders/docker"
    args:
      - "push"
      - "gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA"

  # Deploy to Cloud Run
  - name: "gcr.io/google.com/cloudsdktool/cloud-sdk"
    entrypoint: gcloud
    args:
      - "run"
      - "deploy"
      - "myapp"
      - "--image"
      - "gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA"
      - "--region"
      - "us-central1"
      - "--platform"
      - "managed"

images:
  - "gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA"
  - "gcr.io/$PROJECT_ID/myapp:latest"
```

### 2. Deploy to GKE (Google Kubernetes Engine)

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: gcr.io/PROJECT_ID/myapp:latest
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: "production"
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: db-host
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```bash
# Deploy to GKE
gcloud container clusters create myapp-cluster \
  --num-nodes=3 \
  --machine-type=e2-medium \
  --region=us-central1

kubectl apply -f kubernetes/
```

### 3. Terraform for Multi-Cloud

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

# AWS Resources
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "myapp-vpc"
  }
}

resource "aws_ecs_cluster" "main" {
  name = "myapp-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# GCP Resources
provider "google" {
  project = var.gcp_project_id
  region  = "us-central1"
}

resource "google_cloud_run_service" "main" {
  name     = "myapp"
  location = "us-central1"

  template {
    spec {
      containers {
        image = "gcr.io/${var.gcp_project_id}/myapp:latest"

        resources {
          limits = {
            memory = "512Mi"
            cpu    = "1"
          }
        }
      }
    }
  }

  traffic {
    percent         = 100
    latest_revision = true
  }
}
```

## Best Practices

### 1. **Use Infrastructure as Code**

- Terraform for multi-cloud
- AWS CDK for AWS
- Google Cloud Deployment Manager
- Version control all infrastructure

### 2. **Implement Auto-Scaling**

```bash
# AWS Auto Scaling
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/myapp-cluster/myapp-service \
  --min-capacity 2 \
  --max-capacity 10
```

### 3. **Use Managed Services**

- RDS/Cloud SQL for databases
- ElastiCache/Memorystore for caching
- S3/Cloud Storage for files
- CloudFront/Cloud CDN for distribution

### 4. **Implement Health Checks**

- Application health endpoints
- Load balancer health checks
- Container health checks
- Database connection checks

### 5. **Security Best Practices**

- Use IAM roles, not access keys
- Encrypt data at rest and in transit
- Use secrets management services
- Implement least privilege principle
- Enable VPC/network isolation

### 6. **Cost Optimization**

- Use reserved instances for stable workloads
- Implement auto-scaling
- Use spot instances for non-critical tasks
- Set up billing alerts
- Regular resource audits

### 7. **Monitoring & Logging**

- CloudWatch/Cloud Logging
- Distributed tracing
- Error tracking
- Performance monitoring
- Cost monitoring

### 8. **Disaster Recovery**

- Multi-region deployment
- Regular backups
- Database replication
- Automated failover
- Disaster recovery testing

## Key Takeaways

✅ **Cloud platforms provide scalable, managed infrastructure**  
✅ **Use containers for consistent deployments**  
✅ **Implement auto-scaling for variable loads**  
✅ **Leverage managed services to reduce operational overhead**  
✅ **Use Infrastructure as Code for reproducibility**  
✅ **Implement comprehensive monitoring and logging**  
✅ **Security should be built-in from the start**  
✅ **Multi-region deployment provides high availability**  
✅ **Regular cost optimization saves significant money**  
✅ **Test disaster recovery procedures regularly**

Cloud deployment enables applications to scale globally while reducing infrastructure management overhead and costs.
