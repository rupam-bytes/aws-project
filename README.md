# AWS Global E-Commerce Platform — Complete Build Guide
> Multi-region · Multi-language · Solution Architect Level

---

## Table of Contents
1. [Project Setup & Folder Structure](#1-project-setup--folder-structure)
2. [Infrastructure (Terraform)](#2-infrastructure-terraform)
3. [Microservices Code](#3-microservices-code)
4. [Frontend (React + i18n)](#4-frontend-react--i18n)
5. [DynamoDB Schemas](#5-dynamodb-schemas)
6. [ElastiCache (Redis) Integration](#6-elasticache-redis-integration)
7. [SQS + SNS Event-Driven Flow](#7-sqs--sns-event-driven-flow)
8. [Amazon Cognito Auth](#8-amazon-cognito-auth)
9. [CI/CD — GitHub Actions Pipeline](#9-cicd--github-actions-pipeline)
10. [Dockerfiles for All Services](#10-dockerfiles-for-all-services)
11. [Route 53 Failover Configuration](#11-route-53-failover-configuration)
12. [CloudWatch Monitoring & Alarms](#12-cloudwatch-monitoring--alarms)
13. [WAF + IAM Security](#13-waf--iam-security)
14. [Environment Variables & Secrets](#14-environment-variables--secrets)
15. [Local Dev with Docker Compose](#15-local-dev-with-docker-compose)
16. [Deployment Checklist](#16-deployment-checklist)

---

## 1. Project Setup & Folder Structure

```
ecommerce-platform/
├── services/
│   ├── user-service/
│   │   ├── src/
│   │   │   ├── app.js
│   │   │   ├── routes/
│   │   │   ├── middleware/
│   │   │   └── models/
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── product-service/
│   │   ├── src/
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── order-service/
│   │   ├── src/
│   │   ├── Dockerfile
│   │   └── package.json
│   └── payment-service/
│       ├── src/
│       ├── Dockerfile
│       └── package.json
├── frontend/
│   ├── public/
│   ├── src/
│   │   ├── i18n/
│   │   │   ├── en.json
│   │   │   ├── hi.json
│   │   │   └── bn.json
│   │   ├── components/
│   │   ├── pages/
│   │   └── App.jsx
│   └── package.json
├── infrastructure/
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── modules/
│   │   │   ├── vpc/
│   │   │   ├── ecs/
│   │   │   ├── dynamodb/
│   │   │   ├── elasticache/
│   │   │   └── cloudfront/
│   │   └── environments/
│   │       ├── mumbai.tfvars
│   │       └── singapore.tfvars
├── .github/
│   └── workflows/
│       └── deploy.yml
└── docker-compose.yml
```

---

## 2. Infrastructure (Terraform)

### `infrastructure/terraform/variables.tf`

```hcl
variable "project_name" {
  default = "global-ecommerce"
}

variable "primary_region" {
  default = "ap-south-1"  # Mumbai
}

variable "secondary_region" {
  default = "ap-southeast-1"  # Singapore
}

variable "environment" {
  default = "production"
}

variable "db_table_prefix" {
  default = "ecom"
}
```

### `infrastructure/terraform/main.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "ecommerce-terraform-state"
    key    = "global/terraform.tfstate"
    region = "ap-south-1"
  }
}

# Primary region provider
provider "aws" {
  alias  = "mumbai"
  region = var.primary_region
}

# Secondary region provider
provider "aws" {
  alias  = "singapore"
  region = var.secondary_region
}

# Route 53 always in us-east-1
provider "aws" {
  alias  = "global"
  region = "us-east-1"
}

# VPC — Mumbai
module "vpc_mumbai" {
  source    = "./modules/vpc"
  providers = { aws = aws.mumbai }
  region    = var.primary_region
  name      = "${var.project_name}-mumbai"
  cidr      = "10.0.0.0/16"
  azs       = ["ap-south-1a", "ap-south-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.10.0/24", "10.0.11.0/24"]
}

# VPC — Singapore
module "vpc_singapore" {
  source    = "./modules/vpc"
  providers = { aws = aws.singapore }
  region    = var.secondary_region
  name      = "${var.project_name}-singapore"
  cidr      = "10.1.0.0/16"
  azs       = ["ap-southeast-1a", "ap-southeast-1b"]
  public_subnets  = ["10.1.1.0/24", "10.1.2.0/24"]
  private_subnets = ["10.1.10.0/24", "10.1.11.0/24"]
}

# ECS Cluster — Mumbai
module "ecs_mumbai" {
  source    = "./modules/ecs"
  providers = { aws = aws.mumbai }
  name      = "${var.project_name}-mumbai"
  vpc_id    = module.vpc_mumbai.vpc_id
  subnets   = module.vpc_mumbai.private_subnets
  alb_subnets = module.vpc_mumbai.public_subnets
}

# ECS Cluster — Singapore
module "ecs_singapore" {
  source    = "./modules/ecs"
  providers = { aws = aws.singapore }
  name      = "${var.project_name}-singapore"
  vpc_id    = module.vpc_singapore.vpc_id
  subnets   = module.vpc_singapore.private_subnets
  alb_subnets = module.vpc_singapore.public_subnets
}

# DynamoDB Global Tables
module "dynamodb" {
  source   = "./modules/dynamodb"
  prefix   = var.db_table_prefix
  replicas = [var.primary_region, var.secondary_region]
}

# ElastiCache — Mumbai
module "cache_mumbai" {
  source    = "./modules/elasticache"
  providers = { aws = aws.mumbai }
  name      = "${var.project_name}-mumbai"
  vpc_id    = module.vpc_mumbai.vpc_id
  subnets   = module.vpc_mumbai.private_subnets
}

# ElastiCache — Singapore
module "cache_singapore" {
  source    = "./modules/elasticache"
  providers = { aws = aws.singapore }
  name      = "${var.project_name}-singapore"
  vpc_id    = module.vpc_singapore.vpc_id
  subnets   = module.vpc_singapore.private_subnets
}

# CloudFront + S3
module "cdn" {
  source    = "./modules/cloudfront"
  providers = { aws = aws.global }
  name      = var.project_name
  alb_primary   = module.ecs_mumbai.alb_dns
  alb_secondary = module.ecs_singapore.alb_dns
}

# Route 53
resource "aws_route53_zone" "main" {
  provider = aws.global
  name     = "yourdomain.com"
}

resource "aws_route53_health_check" "primary" {
  provider          = aws.global
  fqdn              = module.ecs_mumbai.alb_dns
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = "3"
  request_interval  = "30"
}

resource "aws_route53_record" "primary" {
  provider = aws.global
  zone_id  = aws_route53_zone.main.zone_id
  name     = "api.yourdomain.com"
  type     = "A"
  set_identifier = "primary"

  failover_routing_policy {
    type = "PRIMARY"
  }

  alias {
    name                   = module.ecs_mumbai.alb_dns
    zone_id                = module.ecs_mumbai.alb_zone_id
    evaluate_target_health = true
  }

  health_check_id = aws_route53_health_check.primary.id
}

resource "aws_route53_record" "secondary" {
  provider = aws.global
  zone_id  = aws_route53_zone.main.zone_id
  name     = "api.yourdomain.com"
  type     = "A"
  set_identifier = "secondary"

  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = module.ecs_singapore.alb_dns
    zone_id                = module.ecs_singapore.alb_zone_id
    evaluate_target_health = true
  }
}
```

### `infrastructure/terraform/modules/vpc/main.tf`

```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = var.name }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnets)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.public_subnets[count.index]
  availability_zone = var.azs[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "${var.name}-public-${count.index}" }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnets)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.azs[count.index]
  tags = { Name = "${var.name}-private-${count.index}" }
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = { Name = var.name }
}

resource "aws_eip" "nat" {
  count  = length(var.public_subnets)
  domain = "vpc"
}

resource "aws_nat_gateway" "this" {
  count         = length(var.public_subnets)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  tags          = { Name = "${var.name}-nat-${count.index}" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }
}

resource "aws_route_table_association" "public" {
  count          = length(var.public_subnets)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

### `infrastructure/terraform/modules/dynamodb/main.tf`

```hcl
resource "aws_dynamodb_table" "users" {
  name         = "${var.prefix}-users"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "userId"

  attribute {
    name = "userId"
    type = "S"
  }

  attribute {
    name = "email"
    type = "S"
  }

  global_secondary_index {
    name            = "email-index"
    hash_key        = "email"
    projection_type = "ALL"
  }

  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  dynamic "replica" {
    for_each = var.replicas
    content {
      region_name = replica.value
    }
  }

  tags = { Name = "${var.prefix}-users" }
}

resource "aws_dynamodb_table" "products" {
  name         = "${var.prefix}-products"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "productId"

  attribute {
    name = "productId"
    type = "S"
  }

  attribute {
    name = "category"
    type = "S"
  }

  global_secondary_index {
    name            = "category-index"
    hash_key        = "category"
    projection_type = "ALL"
  }

  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  dynamic "replica" {
    for_each = var.replicas
    content {
      region_name = replica.value
    }
  }
}

resource "aws_dynamodb_table" "orders" {
  name         = "${var.prefix}-orders"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "orderId"
  range_key    = "userId"

  attribute {
    name = "orderId"
    type = "S"
  }

  attribute {
    name = "userId"
    type = "S"
  }

  global_secondary_index {
    name            = "userId-index"
    hash_key        = "userId"
    projection_type = "ALL"
  }

  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  dynamic "replica" {
    for_each = var.replicas
    content {
      region_name = replica.value
    }
  }
}
```

### `infrastructure/terraform/modules/ecs/main.tf`

```hcl
resource "aws_ecs_cluster" "this" {
  name = var.name
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_security_group" "alb" {
  name   = "${var.name}-alb-sg"
  vpc_id = var.vpc_id
  ingress { from_port = 80  to_port = 80  protocol = "tcp" cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 443 to_port = 443 protocol = "tcp" cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0   to_port = 0   protocol = "-1"  cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_security_group" "ecs_tasks" {
  name   = "${var.name}-tasks-sg"
  vpc_id = var.vpc_id
  ingress {
    from_port       = 3000
    to_port         = 3003
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
  egress { from_port = 0 to_port = 0 protocol = "-1" cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_lb" "this" {
  name               = "${var.name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.alb_subnets
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.this.arn
  port              = 80
  protocol          = "HTTP"
  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# ECS Task Definition — User Service
resource "aws_ecs_task_definition" "user_service" {
  family                   = "${var.name}-user-service"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([{
    name  = "user-service"
    image = "${var.ecr_registry}/user-service:latest"
    portMappings = [{ containerPort = 3000, protocol = "tcp" }]
    environment = [
      { name = "PORT",        value = "3000" },
      { name = "AWS_REGION",  value = var.region },
      { name = "TABLE_USERS", value = "ecom-users" }
    ]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = "/ecs/${var.name}/user-service"
        awslogs-region        = var.region
        awslogs-stream-prefix = "ecs"
      }
    }
    healthCheck = {
      command  = ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
      interval = 30
      timeout  = 5
      retries  = 3
    }
  }])
}

resource "aws_ecs_service" "user_service" {
  name            = "user-service"
  cluster         = aws_ecs_cluster.this.id
  task_definition = aws_ecs_task_definition.user_service.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.subnets
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.user_service.arn
    container_name   = "user-service"
    container_port   = 3000
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }
}

# Auto-scaling
resource "aws_appautoscaling_target" "user_service" {
  max_capacity       = 10
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.this.name}/user-service"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "user_service_cpu" {
  name               = "${var.name}-user-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.user_service.resource_id
  scalable_dimension = aws_appautoscaling_target.user_service.scalable_dimension
  service_namespace  = aws_appautoscaling_target.user_service.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}
```

---

## 3. Microservices Code

### User Service — `services/user-service/src/app.js`

```javascript
const express = require('express');
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, GetCommand, PutCommand, QueryCommand } = require('@aws-sdk/lib-dynamodb');
const { CognitoJwtVerifier } = require('aws-jwt-verify');
const Redis = require('ioredis');
const { v4: uuidv4 } = require('uuid');

const app = express();
app.use(express.json());

const region = process.env.AWS_REGION || 'ap-south-1';
const TABLE = process.env.TABLE_USERS || 'ecom-users';

// DynamoDB client
const ddbClient = new DynamoDBClient({ region });
const ddb = DynamoDBDocumentClient.from(ddbClient);

// Redis client (ElastiCache)
const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: 6379,
  lazyConnect: true,
  retryStrategy: (times) => Math.min(times * 50, 2000),
});

// Cognito JWT verifier
const verifier = CognitoJwtVerifier.create({
  userPoolId: process.env.COGNITO_USER_POOL_ID,
  tokenUse: 'access',
  clientId: process.env.COGNITO_CLIENT_ID,
});

// Auth middleware
async function authenticate(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'No token' });
  try {
    req.user = await verifier.verify(token);
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
}

// Health check (required for ALB + Route 53)
app.get('/health', (req, res) => res.json({ status: 'ok', region, service: 'user-service' }));

// Get user profile
app.get('/users/:userId', authenticate, async (req, res) => {
  const { userId } = req.params;

  // 1. Check Redis cache
  const cached = await redis.get(`user:${userId}`);
  if (cached) return res.json(JSON.parse(cached));

  // 2. DynamoDB fallback
  try {
    const result = await ddb.send(new GetCommand({ TableName: TABLE, Key: { userId } }));
    if (!result.Item) return res.status(404).json({ error: 'User not found' });

    // Cache for 5 min
    await redis.setex(`user:${userId}`, 300, JSON.stringify(result.Item));
    res.json(result.Item);
  } catch (err) {
    console.error('DynamoDB error:', err);
    res.status(500).json({ error: 'Internal error' });
  }
});

// Create user (called post-Cognito registration)
app.post('/users', async (req, res) => {
  const { email, name, language = 'en' } = req.body;
  const userId = uuidv4();
  const createdAt = new Date().toISOString();

  const user = { userId, email, name, language, createdAt };

  try {
    await ddb.send(new PutCommand({
      TableName: TABLE,
      Item: user,
      ConditionExpression: 'attribute_not_exists(userId)',
    }));
    res.status(201).json(user);
  } catch (err) {
    if (err.name === 'ConditionalCheckFailedException') {
      return res.status(409).json({ error: 'User already exists' });
    }
    res.status(500).json({ error: 'Internal error' });
  }
});

// Update language preference
app.patch('/users/:userId/language', authenticate, async (req, res) => {
  const { userId } = req.params;
  const { language } = req.body;

  if (!['en', 'hi', 'bn'].includes(language)) {
    return res.status(400).json({ error: 'Invalid language. Use en, hi, or bn.' });
  }

  const { UpdateCommand } = require('@aws-sdk/lib-dynamodb');
  await ddb.send(new UpdateCommand({
    TableName: TABLE,
    Key: { userId },
    UpdateExpression: 'SET #lang = :lang',
    ExpressionAttributeNames: { '#lang': 'language' },
    ExpressionAttributeValues: { ':lang': language },
  }));

  // Invalidate cache
  await redis.del(`user:${userId}`);
  res.json({ updated: true });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`User service running on port ${PORT} in ${region}`));
module.exports = app;
```

### Product Service — `services/product-service/src/app.js`

```javascript
const express = require('express');
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, GetCommand, PutCommand, QueryCommand, ScanCommand } = require('@aws-sdk/lib-dynamodb');
const Redis = require('ioredis');
const { v4: uuidv4 } = require('uuid');

const app = express();
app.use(express.json());

const region = process.env.AWS_REGION || 'ap-south-1';
const TABLE = process.env.TABLE_PRODUCTS || 'ecom-products';

const ddb = DynamoDBDocumentClient.from(new DynamoDBClient({ region }));
const redis = new Redis({ host: process.env.REDIS_HOST, port: 6379, lazyConnect: true });

app.get('/health', (req, res) => res.json({ status: 'ok', region, service: 'product-service' }));

// List products with language-aware response
app.get('/products', async (req, res) => {
  const { category, lang = 'en', limit = 20 } = req.query;
  const cacheKey = `products:${category || 'all'}:${lang}`;

  // Check cache
  const cached = await redis.get(cacheKey);
  if (cached) return res.json(JSON.parse(cached));

  try {
    let result;
    if (category) {
      result = await ddb.send(new QueryCommand({
        TableName: TABLE,
        IndexName: 'category-index',
        KeyConditionExpression: 'category = :cat',
        ExpressionAttributeValues: { ':cat': category },
        Limit: parseInt(limit),
      }));
    } else {
      result = await ddb.send(new ScanCommand({ TableName: TABLE, Limit: parseInt(limit) }));
    }

    // Project language-specific name/description
    const products = (result.Items || []).map(p => ({
      productId: p.productId,
      category: p.category,
      price: p.price,
      stock: p.stock,
      name: p[`name_${lang}`] || p.name_en,
      description: p[`description_${lang}`] || p.description_en,
    }));

    // Cache for 10 min
    await redis.setex(cacheKey, 600, JSON.stringify({ products }));
    res.json({ products });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Internal error' });
  }
});

// Get single product
app.get('/products/:productId', async (req, res) => {
  const { productId } = req.params;
  const { lang = 'en' } = req.query;
  const cacheKey = `product:${productId}:${lang}`;

  const cached = await redis.get(cacheKey);
  if (cached) return res.json(JSON.parse(cached));

  try {
    const result = await ddb.send(new GetCommand({ TableName: TABLE, Key: { productId } }));
    if (!result.Item) return res.status(404).json({ error: 'Not found' });

    const p = result.Item;
    const product = {
      productId: p.productId,
      category: p.category,
      price: p.price,
      stock: p.stock,
      name: p[`name_${lang}`] || p.name_en,
      description: p[`description_${lang}`] || p.description_en,
    };

    await redis.setex(cacheKey, 600, JSON.stringify(product));
    res.json(product);
  } catch (err) {
    res.status(500).json({ error: 'Internal error' });
  }
});

// Create product (admin)
app.post('/products', async (req, res) => {
  const { name_en, name_hi, name_bn, description_en, description_hi, description_bn,
          category, price, stock } = req.body;

  const productId = uuidv4();
  await ddb.send(new PutCommand({
    TableName: TABLE,
    Item: { productId, name_en, name_hi, name_bn, description_en, description_hi,
            description_bn, category, price, stock, createdAt: new Date().toISOString() },
  }));

  // Invalidate category cache
  await redis.del(`products:${category}:en`);
  await redis.del(`products:${category}:hi`);
  await redis.del(`products:${category}:bn`);

  res.status(201).json({ productId });
});

app.listen(process.env.PORT || 3001);
```

### Order Service — `services/order-service/src/app.js`

```javascript
const express = require('express');
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutCommand, QueryCommand, GetCommand, UpdateCommand } = require('@aws-sdk/lib-dynamodb');
const { SQSClient, SendMessageCommand } = require('@aws-sdk/client-sqs');
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');
const { CognitoJwtVerifier } = require('aws-jwt-verify');
const { v4: uuidv4 } = require('uuid');

const app = express();
app.use(express.json());

const region = process.env.AWS_REGION || 'ap-south-1';
const TABLE = process.env.TABLE_ORDERS || 'ecom-orders';

const ddb = DynamoDBDocumentClient.from(new DynamoDBClient({ region }));
const sqs = new SQSClient({ region });
const sns = new SNSClient({ region });

const verifier = CognitoJwtVerifier.create({
  userPoolId: process.env.COGNITO_USER_POOL_ID,
  tokenUse: 'access',
  clientId: process.env.COGNITO_CLIENT_ID,
});

async function authenticate(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Unauthorized' });
  try {
    req.user = await verifier.verify(token);
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}

app.get('/health', (req, res) => res.json({ status: 'ok', region, service: 'order-service' }));

// Place order
app.post('/orders', authenticate, async (req, res) => {
  const { items, shippingAddress } = req.body;
  const userId = req.user.sub;
  const orderId = uuidv4();
  const createdAt = new Date().toISOString();

  // Calculate total
  const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);

  const order = {
    orderId,
    userId,
    items,
    shippingAddress,
    total,
    status: 'PENDING',
    createdAt,
  };

  try {
    // 1. Save order to DynamoDB
    await ddb.send(new PutCommand({ TableName: TABLE, Item: order }));

    // 2. Push to SQS for async payment processing
    await sqs.send(new SendMessageCommand({
      QueueUrl: process.env.ORDER_QUEUE_URL,
      MessageBody: JSON.stringify({ orderId, userId, total, items }),
      MessageGroupId: userId,  // FIFO queue: group by user
      MessageDeduplicationId: orderId,
    }));

    // 3. SNS notification (triggers email Lambda)
    await sns.send(new PublishCommand({
      TopicArn: process.env.ORDER_TOPIC_ARN,
      Message: JSON.stringify({ event: 'ORDER_PLACED', orderId, userId, total }),
      Subject: 'New Order Placed',
    }));

    res.status(201).json({ orderId, status: 'PENDING', total });
  } catch (err) {
    console.error('Order error:', err);
    res.status(500).json({ error: 'Order failed' });
  }
});

// Get orders for user
app.get('/orders', authenticate, async (req, res) => {
  const userId = req.user.sub;
  try {
    const result = await ddb.send(new QueryCommand({
      TableName: TABLE,
      IndexName: 'userId-index',
      KeyConditionExpression: 'userId = :uid',
      ExpressionAttributeValues: { ':uid': userId },
      ScanIndexForward: false, // newest first
    }));
    res.json({ orders: result.Items || [] });
  } catch (err) {
    res.status(500).json({ error: 'Internal error' });
  }
});

// Update order status (internal, called by payment worker)
app.patch('/orders/:orderId/status', async (req, res) => {
  const internalKey = req.headers['x-internal-key'];
  if (internalKey !== process.env.INTERNAL_API_KEY) {
    return res.status(403).json({ error: 'Forbidden' });
  }

  const { orderId } = req.params;
  const { status, userId } = req.body;

  await ddb.send(new UpdateCommand({
    TableName: TABLE,
    Key: { orderId, userId },
    UpdateExpression: 'SET #s = :status, updatedAt = :ts',
    ExpressionAttributeNames: { '#s': 'status' },
    ExpressionAttributeValues: { ':status': status, ':ts': new Date().toISOString() },
  }));

  res.json({ updated: true });
});

app.listen(process.env.PORT || 3002);
```

### Payment Mock Service — `services/payment-service/src/app.js`

```javascript
const express = require('express');
const { SQSClient, ReceiveMessageCommand, DeleteMessageCommand } = require('@aws-sdk/client-sqs');
const axios = require('axios');

const app = express();
app.use(express.json());

const region = process.env.AWS_REGION || 'ap-south-1';
const sqs = new SQSClient({ region });

app.get('/health', (req, res) => res.json({ status: 'ok', service: 'payment-service' }));

// SQS worker — polls the order queue and processes payments
async function processOrders() {
  console.log('Payment worker started...');
  while (true) {
    try {
      const result = await sqs.send(new ReceiveMessageCommand({
        QueueUrl: process.env.ORDER_QUEUE_URL,
        MaxNumberOfMessages: 5,
        WaitTimeSeconds: 20, // Long polling
      }));

      for (const message of result.Messages || []) {
        const order = JSON.parse(message.Body);
        console.log('Processing payment for order:', order.orderId);

        // Mock payment — 95% success rate
        const paymentSuccess = Math.random() > 0.05;
        const newStatus = paymentSuccess ? 'CONFIRMED' : 'PAYMENT_FAILED';

        // Update order status
        await axios.patch(
          `${process.env.ORDER_SERVICE_URL}/orders/${order.orderId}/status`,
          { status: newStatus, userId: order.userId },
          { headers: { 'x-internal-key': process.env.INTERNAL_API_KEY } }
        );

        // Delete from queue
        await sqs.send(new DeleteMessageCommand({
          QueueUrl: process.env.ORDER_QUEUE_URL,
          ReceiptHandle: message.ReceiptHandle,
        }));

        console.log(`Order ${order.orderId} → ${newStatus}`);
      }
    } catch (err) {
      console.error('Worker error:', err);
      await new Promise(r => setTimeout(r, 5000)); // backoff
    }
  }
}

processOrders();
app.listen(process.env.PORT || 3003);
```

---

## 4. Frontend (React + i18n)

### `frontend/src/i18n/en.json`
```json
{
  "app_name": "ShopGlobal",
  "home": "Home",
  "products": "Products",
  "cart": "Cart",
  "login": "Login",
  "logout": "Logout",
  "add_to_cart": "Add to Cart",
  "place_order": "Place Order",
  "order_placed": "Order placed successfully!",
  "loading": "Loading...",
  "error": "Something went wrong. Please try again."
}
```

### `frontend/src/i18n/hi.json`
```json
{
  "app_name": "शॉपग्लोबल",
  "home": "होम",
  "products": "उत्पाद",
  "cart": "कार्ट",
  "login": "लॉगिन",
  "logout": "लॉगआउट",
  "add_to_cart": "कार्ट में जोड़ें",
  "place_order": "ऑर्डर दें",
  "order_placed": "ऑर्डर सफलतापूर्वक दिया गया!",
  "loading": "लोड हो रहा है...",
  "error": "कुछ गलत हुआ। कृपया पुनः प्रयास करें।"
}
```

### `frontend/src/i18n/bn.json`
```json
{
  "app_name": "শপগ্লোবাল",
  "home": "হোম",
  "products": "পণ্য",
  "cart": "কার্ট",
  "login": "লগইন",
  "logout": "লগআউট",
  "add_to_cart": "কার্টে যোগ করুন",
  "place_order": "অর্ডার দিন",
  "order_placed": "অর্ডার সফলভাবে দেওয়া হয়েছে!",
  "loading": "লোড হচ্ছে...",
  "error": "কিছু একটা ভুল হয়েছে। আবার চেষ্টা করুন।"
}
```

### `frontend/src/App.jsx`

```jsx
import React, { useState, createContext, useContext, useEffect } from 'react';
import { Amplify } from 'aws-amplify';
import { Authenticator } from '@aws-amplify/ui-react';
import '@aws-amplify/ui-react/styles.css';
import en from './i18n/en.json';
import hi from './i18n/hi.json';
import bn from './i18n/bn.json';

Amplify.configure({
  Auth: {
    Cognito: {
      userPoolId: process.env.REACT_APP_COGNITO_USER_POOL_ID,
      userPoolClientId: process.env.REACT_APP_COGNITO_CLIENT_ID,
    }
  }
});

const translations = { en, hi, bn };

export const LangContext = createContext({ lang: 'en', t: (k) => k });
export const CartContext = createContext({ cart: [], addToCart: () => {} });

function useLang() { return useContext(LangContext); }
function useCart() { return useContext(CartContext); }

export default function App() {
  const [lang, setLang] = useState(() => {
    // Detect from URL path first: /hi, /bn, /en
    const path = window.location.pathname.split('/')[1];
    return ['en', 'hi', 'bn'].includes(path) ? path : 'en';
  });
  const [cart, setCart] = useState([]);

  const t = (key) => translations[lang][key] || key;

  function addToCart(product) {
    setCart(prev => {
      const existing = prev.find(i => i.productId === product.productId);
      if (existing) {
        return prev.map(i => i.productId === product.productId
          ? { ...i, quantity: i.quantity + 1 }
          : i);
      }
      return [...prev, { ...product, quantity: 1 }];
    });
  }

  function changeLanguage(newLang) {
    setLang(newLang);
    window.history.pushState({}, '', `/${newLang}`);
    // Persist to backend if logged in
  }

  return (
    <LangContext.Provider value={{ lang, t, setLang: changeLanguage }}>
      <CartContext.Provider value={{ cart, addToCart }}>
        <Authenticator>
          {({ signOut, user }) => (
            <div>
              <nav style={{ display: 'flex', gap: 16, padding: '12px 24px', borderBottom: '1px solid #eee' }}>
                <strong>{t('app_name')}</strong>
                <a href={`/${lang}`}>{t('home')}</a>
                <a href={`/${lang}/products`}>{t('products')}</a>
                <a href={`/${lang}/cart`}>{t('cart')} ({cart.length})</a>
                <div style={{ marginLeft: 'auto', display: 'flex', gap: 8 }}>
                  {['en', 'hi', 'bn'].map(l => (
                    <button key={l} onClick={() => changeLanguage(l)}
                      style={{ fontWeight: l === lang ? 'bold' : 'normal', background: 'none', border: 'none', cursor: 'pointer' }}>
                      {l.toUpperCase()}
                    </button>
                  ))}
                  <span>{user.signInDetails?.loginId}</span>
                  <button onClick={signOut}>{t('logout')}</button>
                </div>
              </nav>
              <Router lang={lang} />
            </div>
          )}
        </Authenticator>
      </CartContext.Provider>
    </LangContext.Provider>
  );
}

function Router({ lang }) {
  const path = window.location.pathname.replace(`/${lang}`, '') || '/';
  if (path === '/products' || path === '/') return <ProductsPage />;
  if (path === '/cart') return <CartPage />;
  return <ProductsPage />;
}

function ProductsPage() {
  const { lang, t } = useLang();
  const { addToCart } = useCart();
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`${process.env.REACT_APP_API_URL}/products?lang=${lang}`)
      .then(r => r.json())
      .then(data => { setProducts(data.products); setLoading(false); })
      .catch(() => setLoading(false));
  }, [lang]);

  if (loading) return <p style={{ padding: 24 }}>{t('loading')}</p>;

  return (
    <div style={{ padding: 24, display: 'grid', gridTemplateColumns: 'repeat(auto-fill, minmax(200px, 1fr))', gap: 16 }}>
      {products.map(p => (
        <div key={p.productId} style={{ border: '1px solid #eee', borderRadius: 8, padding: 16 }}>
          <h3 style={{ margin: '0 0 8px' }}>{p.name}</h3>
          <p style={{ color: '#666', fontSize: 14 }}>{p.description}</p>
          <strong>₹{p.price}</strong>
          <button
            onClick={() => addToCart(p)}
            style={{ display: 'block', marginTop: 12, padding: '8px 16px', background: '#0066cc', color: 'white', border: 'none', borderRadius: 6, cursor: 'pointer' }}>
            {t('add_to_cart')}
          </button>
        </div>
      ))}
    </div>
  );
}

function CartPage() {
  const { t } = useLang();
  const { cart } = useCart();
  const total = cart.reduce((s, i) => s + i.price * i.quantity, 0);
  const [placed, setPlaced] = useState(false);

  async function placeOrder() {
    const token = (await import('aws-amplify/auth').then(m => m.fetchAuthSession()))
      .tokens?.accessToken?.toString();

    await fetch(`${process.env.REACT_APP_API_URL}/orders`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
      body: JSON.stringify({ items: cart, shippingAddress: {} }),
    });
    setPlaced(true);
  }

  if (placed) return <p style={{ padding: 24 }}>{t('order_placed')}</p>;

  return (
    <div style={{ padding: 24 }}>
      {cart.map(i => (
        <div key={i.productId} style={{ display: 'flex', justifyContent: 'space-between', padding: '8px 0', borderBottom: '1px solid #eee' }}>
          <span>{i.name} × {i.quantity}</span>
          <span>₹{i.price * i.quantity}</span>
        </div>
      ))}
      <strong style={{ display: 'block', marginTop: 16 }}>Total: ₹{total}</strong>
      <button onClick={placeOrder}
        style={{ marginTop: 16, padding: '10px 24px', background: '#0066cc', color: 'white', border: 'none', borderRadius: 6, cursor: 'pointer' }}>
        {t('place_order')}
      </button>
    </div>
  );
}
```

---

## 5. DynamoDB Schemas

Sample seed data demonstrating multi-language structure:

```json
// ecom-products table item
{
  "productId": "prod-001",
  "category": "electronics",
  "price": 2499,
  "stock": 150,
  "name_en": "Wireless Headphones",
  "name_hi": "वायरलेस हेडफ़ोन",
  "name_bn": "ওয়্যারলেস হেডফোন",
  "description_en": "Premium sound quality with noise cancellation",
  "description_hi": "नॉइज़ कैंसलेशन के साथ प्रीमियम साउंड क्वालिटी",
  "description_bn": "নয়েজ ক্যান্সেলেশন সহ প্রিমিয়াম সাউন্ড কোয়ালিটি",
  "createdAt": "2025-01-01T00:00:00Z"
}

// ecom-users table item
{
  "userId": "uuid-here",
  "email": "user@example.com",
  "name": "Rahul Sharma",
  "language": "hi",
  "createdAt": "2025-01-01T00:00:00Z"
}

// ecom-orders table item
{
  "orderId": "order-001",
  "userId": "uuid-here",
  "items": [
    { "productId": "prod-001", "name_en": "Wireless Headphones", "price": 2499, "quantity": 1 }
  ],
  "total": 2499,
  "status": "CONFIRMED",
  "shippingAddress": { "line1": "123 MG Road", "city": "Mumbai", "pincode": "400001" },
  "createdAt": "2025-01-02T10:30:00Z"
}
```

---

## 6. ElastiCache (Redis) Integration

### Caching strategy used in services:

| Cache Key Pattern     | TTL    | Invalidated When          |
|-----------------------|--------|---------------------------|
| `user:{userId}`       | 5 min  | User profile update       |
| `product:{id}:{lang}` | 10 min | Product update            |
| `products:{cat}:{lang}` | 10 min | Any product in category updated |

### Redis connection setup shared across services:

```javascript
// shared/redis.js
const Redis = require('ioredis');

let client;

function getRedis() {
  if (!client) {
    client = new Redis({
      host: process.env.REDIS_HOST,
      port: 6379,
      retryStrategy: (times) => {
        if (times > 10) return null;
        return Math.min(times * 100, 3000);
      },
      enableOfflineQueue: false,
    });
    client.on('error', err => console.error('Redis error:', err));
    client.on('connect', () => console.log('Redis connected'));
  }
  return client;
}

// Wrapper with fallback — never let cache failure break the API
async function withCache(key, ttl, fetchFn) {
  const redis = getRedis();
  try {
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);
  } catch (err) {
    console.warn('Cache read failed, fetching from DB:', err.message);
  }

  const data = await fetchFn();

  try {
    await redis.setex(key, ttl, JSON.stringify(data));
  } catch (err) {
    console.warn('Cache write failed:', err.message);
  }

  return data;
}

module.exports = { getRedis, withCache };
```

---

## 7. SQS + SNS Event-Driven Flow

### Terraform for SQS + SNS:

```hcl
# FIFO queue for ordered processing
resource "aws_sqs_queue" "order_queue" {
  name                        = "ecom-orders.fifo"
  fifo_queue                  = true
  content_based_deduplication = false
  visibility_timeout_seconds  = 60
  message_retention_seconds   = 86400  # 1 day
  receive_wait_time_seconds   = 20     # Long polling
}

resource "aws_sqs_queue" "order_dlq" {
  name       = "ecom-orders-dlq.fifo"
  fifo_queue = true
}

resource "aws_sqs_queue_redrive_policy" "order_queue" {
  queue_url = aws_sqs_queue.order_queue.url
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.order_dlq.arn
    maxReceiveCount     = 3
  })
}

# SNS topic for order events
resource "aws_sns_topic" "orders" {
  name = "ecom-order-events"
}

# Email notification Lambda subscribes to SNS
resource "aws_sns_topic_subscription" "email_lambda" {
  topic_arn = aws_sns_topic.orders.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.email_notifier.arn
}
```

### Email notification Lambda — `services/email-notifier/handler.js`:

```javascript
const { SESClient, SendEmailCommand } = require('@aws-sdk/client-ses');
const ses = new SESClient({ region: process.env.AWS_REGION });

exports.handler = async (event) => {
  for (const record of event.Records) {
    const message = JSON.parse(record.Sns.Message);
    const { event: eventType, orderId, userId, total } = message;

    if (eventType === 'ORDER_PLACED') {
      // In production, fetch user email from DynamoDB first
      await ses.send(new SendEmailCommand({
        Source: 'noreply@yourdomain.com',
        Destination: { ToAddresses: [`${userId}@example.com`] },
        Message: {
          Subject: { Data: `Order Confirmed — #${orderId}` },
          Body: {
            Text: { Data: `Your order #${orderId} of ₹${total} has been placed successfully.` },
            Html: { Data: `<h2>Order Confirmed!</h2><p>Order ID: <strong>${orderId}</strong></p><p>Total: ₹${total}</p>` }
          }
        }
      }));
    }
  }
};
```

---

## 8. Amazon Cognito Auth

### Terraform Cognito setup:

```hcl
resource "aws_cognito_user_pool" "main" {
  name = "ecom-users"

  password_policy {
    minimum_length    = 8
    require_uppercase = true
    require_numbers   = true
  }

  auto_verified_attributes = ["email"]
  username_attributes      = ["email"]

  verification_message_template {
    default_email_option = "CONFIRM_WITH_CODE"
    email_subject        = "Your verification code"
    email_message        = "Your verification code is {####}"
  }

  lambda_config {
    post_confirmation = aws_lambda_function.post_registration.arn
  }

  tags = { Name = "ecom-users" }
}

resource "aws_cognito_user_pool_client" "frontend" {
  name         = "ecom-frontend"
  user_pool_id = aws_cognito_user_pool.main.id

  explicit_auth_flows = [
    "ALLOW_USER_SRP_AUTH",
    "ALLOW_REFRESH_TOKEN_AUTH",
  ]

  access_token_validity  = 1   # hours
  refresh_token_validity = 30  # days

  token_validity_units {
    access_token  = "hours"
    refresh_token = "days"
  }
}
```

### Post-registration Lambda — creates user in DynamoDB:

```javascript
// services/post-registration/handler.js
const axios = require('axios');

exports.handler = async (event) => {
  const { email, sub } = event.request.userAttributes;
  const name = event.request.userAttributes.name || email.split('@')[0];

  await axios.post(`${process.env.USER_SERVICE_URL}/users`, {
    userId: sub,
    email,
    name,
    language: 'en',
  });

  return event;
};
```

---

## 9. CI/CD — GitHub Actions Pipeline

### `.github/workflows/deploy.yml`

```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]

env:
  AWS_REGION_PRIMARY: ap-south-1
  AWS_REGION_SECONDARY: ap-southeast-1
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-south-1.amazonaws.com

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - name: Install and test user-service
        run: |
          cd services/user-service
          npm ci
          npm test
      - name: Install and test product-service
        run: |
          cd services/product-service
          npm ci
          npm test
      - name: Install and test order-service
        run: |
          cd services/order-service
          npm ci
          npm test

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region:            ${{ env.AWS_REGION_PRIMARY }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, push images
        run: |
          SERVICES=("user-service" "product-service" "order-service" "payment-service")
          for SERVICE in "${SERVICES[@]}"; do
            docker build -t $ECR_REGISTRY/$SERVICE:${{ github.sha }} \
                         -t $ECR_REGISTRY/$SERVICE:latest \
                         ./services/$SERVICE
            docker push $ECR_REGISTRY/$SERVICE:${{ github.sha }}
            docker push $ECR_REGISTRY/$SERVICE:latest
          done

  deploy-primary:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (Mumbai)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region:            ap-south-1

      - name: Deploy to ECS Mumbai
        run: |
          SERVICES=("user-service" "product-service" "order-service" "payment-service")
          for SERVICE in "${SERVICES[@]}"; do
            aws ecs update-service \
              --cluster global-ecommerce-mumbai \
              --service $SERVICE \
              --force-new-deployment \
              --region ap-south-1
          done

      - name: Wait for Mumbai deployment
        run: |
          aws ecs wait services-stable \
            --cluster global-ecommerce-mumbai \
            --services user-service product-service order-service payment-service \
            --region ap-south-1

  deploy-secondary:
    needs: deploy-primary
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (Singapore)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region:            ap-southeast-1

      - name: Deploy to ECS Singapore
        run: |
          SERVICES=("user-service" "product-service" "order-service" "payment-service")
          for SERVICE in "${SERVICES[@]}"; do
            aws ecs update-service \
              --cluster global-ecommerce-singapore \
              --service $SERVICE \
              --force-new-deployment \
              --region ap-southeast-1
          done

  deploy-frontend:
    needs: deploy-primary
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build frontend
        run: |
          cd frontend
          npm ci
          REACT_APP_API_URL=${{ secrets.API_URL }} \
          REACT_APP_COGNITO_USER_POOL_ID=${{ secrets.COGNITO_USER_POOL_ID }} \
          REACT_APP_COGNITO_CLIENT_ID=${{ secrets.COGNITO_CLIENT_ID }} \
          npm run build

      - name: Deploy to S3
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region:            us-east-1
      - run: aws s3 sync frontend/build s3://${{ secrets.FRONTEND_BUCKET }} --delete

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CF_DISTRIBUTION_ID }} \
            --paths "/*"
```

---

## 10. Dockerfiles for All Services

### `services/user-service/Dockerfile`

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/node_modules ./node_modules
COPY src/ ./src/
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "src/app.js"]
```

Same pattern for product-service (port 3001), order-service (port 3002), payment-service (port 3003).

### `services/product-service/package.json`

```json
{
  "name": "product-service",
  "version": "1.0.0",
  "scripts": {
    "start": "node src/app.js",
    "dev": "nodemon src/app.js",
    "test": "jest"
  },
  "dependencies": {
    "@aws-sdk/client-dynamodb": "^3.0.0",
    "@aws-sdk/lib-dynamodb": "^3.0.0",
    "aws-jwt-verify": "^4.0.0",
    "express": "^4.18.0",
    "ioredis": "^5.0.0",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "nodemon": "^3.0.0",
    "supertest": "^6.0.0"
  }
}
```

---

## 11. Route 53 Failover Configuration

Route 53 is already in the Terraform `main.tf` above. Key points:

- **Health check** polls `/health` on the Mumbai ALB every 30 seconds
- **Primary record** points to Mumbai ALB; marked `PRIMARY` in failover policy
- **Secondary record** points to Singapore ALB; marked `SECONDARY`
- If Mumbai fails 3 health checks in a row, Route 53 **automatically** routes all traffic to Singapore
- DynamoDB Global Tables ensure Singapore already has all the data

### Manual failover test:
```bash
# Force failover — disable the primary health check
aws route53 update-health-check \
  --health-check-id <HEALTH_CHECK_ID> \
  --disabled \
  --region us-east-1

# Verify traffic is going to Singapore
dig api.yourdomain.com

# Re-enable
aws route53 update-health-check \
  --health-check-id <HEALTH_CHECK_ID> \
  --no-disabled \
  --region us-east-1
```

---

## 12. CloudWatch Monitoring & Alarms

```hcl
# Alarm: high API error rate
resource "aws_cloudwatch_metric_alarm" "api_errors" {
  alarm_name          = "api-5xx-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Sum"
  threshold           = 10

  dimensions = {
    LoadBalancer = module.ecs_mumbai.alb_arn_suffix
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
  alarm_description = "More than 10 5xx errors per minute"
}

# Alarm: DynamoDB throttled requests
resource "aws_cloudwatch_metric_alarm" "ddb_throttle" {
  alarm_name          = "ddb-throttled"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ThrottledRequests"
  namespace           = "AWS/DynamoDB"
  period              = 60
  statistic           = "Sum"
  threshold           = 5

  alarm_actions = [aws_sns_topic.alerts.arn]
}

# CloudWatch dashboard
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "ecommerce-overview"
  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          title  = "ALB Request Count"
          metrics = [["AWS/ApplicationELB", "RequestCount"]]
          period = 60
          stat   = "Sum"
        }
      },
      {
        type = "metric"
        properties = {
          title  = "ECS CPU Utilization"
          metrics = [["AWS/ECS", "CPUUtilization", "ClusterName", "global-ecommerce-mumbai"]]
          period = 60
          stat   = "Average"
        }
      },
      {
        type = "metric"
        properties = {
          title  = "Cache Hit Rate (Redis)"
          metrics = [["AWS/ElastiCache", "CacheHits"], ["AWS/ElastiCache", "CacheMisses"]]
          period = 60
          stat   = "Sum"
        }
      }
    ]
  })
}
```

### Structured logs — all services log this format:

```javascript
function log(level, message, meta = {}) {
  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    level,
    service: process.env.SERVICE_NAME,
    region: process.env.AWS_REGION,
    ...meta,
    message,
  }));
}

// Usage:
log('info', 'Order placed', { orderId, userId, total });
log('error', 'DynamoDB error', { error: err.message, table: TABLE });
```

---

## 13. WAF + IAM Security

### WAF (Web ACL):

```hcl
resource "aws_wafv2_web_acl" "main" {
  provider = aws.global
  name     = "ecommerce-waf"
  scope    = "CLOUDFRONT"

  default_action { allow {} }

  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  rule {
    name     = "RateLimitRule"
    priority = 2
    action { block {} }
    statement {
      rate_based_statement {
        limit              = 1000
        aggregate_key_type = "IP"
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimit"
      sampled_requests_enabled   = true
    }
  }
}
```

### IAM — Task execution role (least privilege):

```hcl
resource "aws_iam_role" "ecs_task" {
  name = "${var.name}-ecs-task-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "ecs_task_dynamodb" {
  name = "dynamodb-access"
  role = aws_iam_role.ecs_task.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:UpdateItem",
                  "dynamodb:Query", "dynamodb:Scan"]
        Resource = [
          "arn:aws:dynamodb:*:*:table/ecom-*",
          "arn:aws:dynamodb:*:*:table/ecom-*/index/*"
        ]
      },
      {
        Effect   = "Allow"
        Action   = ["sqs:SendMessage", "sqs:ReceiveMessage", "sqs:DeleteMessage"]
        Resource = "arn:aws:sqs:*:*:ecom-*"
      },
      {
        Effect   = "Allow"
        Action   = ["sns:Publish"]
        Resource = "arn:aws:sns:*:*:ecom-*"
      }
    ]
  })
}
```

---

## 14. Environment Variables & Secrets

### Use AWS Secrets Manager — never hardcode:

```javascript
// shared/secrets.js
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');
const client = new SecretsManagerClient({ region: process.env.AWS_REGION });

const cache = {};

async function getSecret(secretName) {
  if (cache[secretName]) return cache[secretName];
  const result = await client.send(new GetSecretValueCommand({ SecretId: secretName }));
  const value = JSON.parse(result.SecretString);
  cache[secretName] = value;
  return value;
}

module.exports = { getSecret };
```

### GitHub Secrets to set:
```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_ACCOUNT_ID
COGNITO_USER_POOL_ID
COGNITO_CLIENT_ID
API_URL
FRONTEND_BUCKET
CF_DISTRIBUTION_ID
INTERNAL_API_KEY
```

### `.env.example` for local dev:
```bash
PORT=3000
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=your-key
AWS_SECRET_ACCESS_KEY=your-secret
TABLE_USERS=ecom-users
REDIS_HOST=localhost
COGNITO_USER_POOL_ID=ap-south-1_xxxxx
COGNITO_CLIENT_ID=xxxxxxxxx
ORDER_QUEUE_URL=https://sqs.ap-south-1.amazonaws.com/xxx/ecom-orders.fifo
ORDER_TOPIC_ARN=arn:aws:sns:ap-south-1:xxx:ecom-order-events
ORDER_SERVICE_URL=http://localhost:3002
INTERNAL_API_KEY=dev-internal-key-change-in-prod
SERVICE_NAME=user-service
```

---

## 15. Local Dev with Docker Compose

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  user-service:
    build: ./services/user-service
    ports:
      - "3000:3000"
    env_file: ./services/user-service/.env
    environment:
      - REDIS_HOST=redis
      - AWS_REGION=ap-south-1
    depends_on:
      - redis
      - dynamodb-local

  product-service:
    build: ./services/product-service
    ports:
      - "3001:3001"
    env_file: ./services/product-service/.env
    environment:
      - REDIS_HOST=redis
      - PORT=3001
    depends_on:
      - redis
      - dynamodb-local

  order-service:
    build: ./services/order-service
    ports:
      - "3002:3002"
    environment:
      - REDIS_HOST=redis
      - PORT=3002
      - ORDER_QUEUE_URL=http://localstack:4566/000000000000/ecom-orders.fifo
      - ORDER_TOPIC_ARN=arn:aws:sns:us-east-1:000000000000:ecom-order-events
    depends_on:
      - redis
      - localstack

  payment-service:
    build: ./services/payment-service
    ports:
      - "3003:3003"
    environment:
      - PORT=3003
      - ORDER_QUEUE_URL=http://localstack:4566/000000000000/ecom-orders.fifo
      - ORDER_SERVICE_URL=http://order-service:3002
      - INTERNAL_API_KEY=dev-internal-key
    depends_on:
      - localstack

  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    environment:
      - REACT_APP_API_URL=http://localhost:3001

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  dynamodb-local:
    image: amazon/dynamodb-local
    ports:
      - "8000:8000"
    command: -jar DynamoDBLocal.jar -inMemory

  localstack:
    image: localstack/localstack
    ports:
      - "4566:4566"
    environment:
      - SERVICES=sqs,sns,ses
      - DEFAULT_REGION=us-east-1
```

### Seed local DynamoDB:
```bash
# Create tables locally
aws dynamodb create-table \
  --table-name ecom-products \
  --attribute-definitions AttributeName=productId,AttributeType=S \
  --key-schema AttributeName=productId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --endpoint-url http://localhost:8000

# Seed a product
aws dynamodb put-item \
  --table-name ecom-products \
  --item '{"productId":{"S":"prod-001"},"category":{"S":"electronics"},"price":{"N":"2499"},"stock":{"N":"100"},"name_en":{"S":"Wireless Headphones"},"name_hi":{"S":"वायरलेस हेडफ़ोन"},"name_bn":{"S":"ওয়্যারলেস হেডফোন"}}' \
  --endpoint-url http://localhost:8000
```

---

## 16. Deployment Checklist

### Phase 1 — Infrastructure
- [ ] Fork/clone the repo
- [ ] Configure `terraform/environments/mumbai.tfvars` and `singapore.tfvars`
- [ ] Run `terraform init && terraform plan && terraform apply` for Mumbai
- [ ] Run same for Singapore
- [ ] Verify DynamoDB Global Table replication is active
- [ ] Create ECR repositories for all 4 services

### Phase 2 — Services
- [ ] Build and push Docker images to ECR
- [ ] Verify ECS services are RUNNING (2 tasks each, both regions)
- [ ] Hit `GET /health` on each service via ALB
- [ ] Verify DynamoDB read/write from ECS tasks

### Phase 3 — Frontend
- [ ] Create S3 bucket, enable static website hosting
- [ ] Deploy CloudFront distribution pointing to S3
- [ ] Configure Cognito User Pool and App Client
- [ ] Build React app with correct env vars
- [ ] Upload to S3, invalidate CloudFront

### Phase 4 — DNS + Failover
- [ ] Register or import domain in Route 53
- [ ] Create health check pointing to Mumbai `/health`
- [ ] Create PRIMARY A record (Mumbai ALB)
- [ ] Create SECONDARY A record (Singapore ALB)
- [ ] Test failover manually (disable health check, verify DNS switches)

### Phase 5 — Observability
- [ ] Verify CloudWatch log groups exist for all ECS services
- [ ] Create CloudWatch alarms (5xx, DynamoDB throttle, CPU)
- [ ] Create CloudWatch dashboard
- [ ] Set up SNS alert topic → your email

### Phase 6 — CI/CD
- [ ] Add all GitHub Secrets
- [ ] Push to `main` branch
- [ ] Watch GitHub Actions pipeline run end-to-end
- [ ] Verify new image deployed to both regions

---

## Cost Estimate (minimal/dev setup)

| Service | Monthly Cost |
|---------|-------------|
| ECS Fargate (4 services × 2 regions, 0.25 vCPU) | ~$30 |
| DynamoDB (on-demand, low traffic) | ~$5 |
| ElastiCache (cache.t3.micro × 2) | ~$25 |
| ALB (× 2 regions) | ~$20 |
| CloudFront + S3 | ~$5 |
| Route 53 | ~$2 |
| **Total** | **~$90/month** |

> Shut down ECS to 0 tasks when not testing to reduce cost significantly.
