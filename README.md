# 🛒 Global E-Commerce Platform

A production-grade, multi-region e-commerce platform built on AWS — demonstrating **Solution Architect level** cloud design with containerized microservices, automatic failover, multi-language support, and event-driven architecture.

> Built as a portfolio project to showcase real-world AWS architecture patterns.

---

## 📸 Architecture Overview

```
                         ┌─────────────────────┐
                         │     Route 53         │
                         │  Failover Routing    │
                         └────────┬────────────┘
                                  │
                    ┌─────────────┴──────────────┐
                    │        CloudFront           │
                    │   S3 Frontend (en/hi/bn)    │
                    └─────────────┬──────────────┘
                                  │
              ┌───────────────────┴──────────────────┐
              │                                       │
   ┌──────────▼──────────┐              ┌────────────▼────────────┐
   │  Region 1 — Mumbai   │◄────────────►│  Region 2 — Singapore   │
   │  (Primary)           │  DynamoDB   │  (Secondary / Failover) │
   │                      │  Global     │                         │
   │  ALB → ECS Fargate   │  Tables     │  ALB → ECS Fargate      │
   │  ┌──────────────┐    │             │  ┌──────────────┐       │
   │  │ User Service │    │             │  │ User Service │       │
   │  │ Product Svc  │    │             │  │ Product Svc  │       │
   │  │ Order Svc    │    │             │  │ Order Svc    │       │
   │  │ Payment Svc  │    │             │  │ Payment Svc  │       │
   │  └──────────────┘    │             │  └──────────────┘       │
   │  ElastiCache (Redis) │             │  ElastiCache (Redis)    │
   │  DynamoDB            │             │  DynamoDB               │
   └──────────────────────┘             └─────────────────────────┘
              │
   ┌──────────▼──────────────────────────────┐
   │  SQS (FIFO) → Payment Worker            │
   │  SNS → Email Lambda (SES)               │
   │  CloudWatch Logs + Alarms               │
   │  WAF + IAM (Least Privilege)            │
   └─────────────────────────────────────────┘
```

---

## ✨ Features

| Feature | Implementation |
|---|---|
| Multi-region deployment | Mumbai (primary) + Singapore (secondary) |
| Automatic failover | Route 53 health checks + failover routing |
| Multi-language UI | English, Hindi, Bengali (`/en`, `/hi`, `/bn`) |
| User self-registration | Amazon Cognito — email verification, JWT auth |
| Product management | Admin API, AWS Console, or seed script |
| Containerized services | Docker → ECR → ECS Fargate |
| Global data replication | DynamoDB Global Tables (active-active) |
| Caching layer | ElastiCache Redis (cache-aside, 5–10 min TTL) |
| Async order processing | SQS FIFO queue + payment worker |
| Email notifications | SNS → Lambda → SES |
| CI/CD pipeline | GitHub Actions → ECR → ECS (both regions) |
| Security | WAF, IAM least privilege, Cognito JWT, Secrets Manager |
| Observability | CloudWatch logs (JSON structured) + alarms + dashboard |

---

## 🗂 Project Structure

```
ecommerce-platform/
├── services/
│   ├── user-service/          # Port 3000 — profile, language preference
│   ├── product-service/       # Port 3001 — catalog, multi-language names
│   ├── order-service/         # Port 3002 — place orders, SQS publish
│   └── payment-service/       # Port 3003 — SQS worker, mock payment
├── frontend/
│   ├── src/
│   │   ├── i18n/              # en.json, hi.json, bn.json
│   │   ├── pages/             # ProductsPage, CartPage
│   │   └── App.jsx            # Amplify auth, language routing
│   └── package.json
├── infrastructure/
│   └── terraform/
│       ├── main.tf            # Route 53, CloudFront, ECS, DynamoDB, ElastiCache
│       ├── variables.tf
│       ├── outputs.tf
│       └── modules/
│           ├── vpc/
│           ├── ecs/
│           ├── dynamodb/
│           ├── elasticache/
│           └── cloudfront/
├── .github/
│   └── workflows/
│       └── deploy.yml         # test → build → push ECR → deploy both regions
├── seed-products.js           # Add products to DynamoDB
├── docker-compose.yml         # Local dev with LocalStack + DynamoDB Local
└── README.md
```

---

## 🚀 Quick Start

### Prerequisites

| Tool | Version | Install |
|---|---|---|
| Node.js | 20+ | [nodejs.org](https://nodejs.org) |
| Docker Desktop | latest | [docker.com](https://docker.com/products/docker-desktop) |
| AWS CLI | v2 | [docs.aws.amazon.com/cli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) |
| Terraform | 1.7+ | [developer.hashicorp.com/terraform](https://developer.hashicorp.com/terraform/downloads) |
| Git | any | [git-scm.com](https://git-scm.com) |

### 1. Clone and install

```bash
git clone https://github.com/YOUR_USERNAME/ecommerce-platform.git
cd ecommerce-platform

# Install dependencies for all services
for service in user-service product-service order-service payment-service; do
  cd services/$service && npm install && cd ../..
done

cd frontend && npm install && cd ..
```

### 2. Configure AWS CLI

```bash
aws configure
# AWS Access Key ID:     <your IAM key>
# AWS Secret Access Key: <your IAM secret>
# Default region:        ap-south-1
# Default output format: json
```

### 3. Run locally with Docker Compose

```bash
# Start all services + Redis + LocalStack + DynamoDB Local
docker compose up --build

# In a new terminal — create local DynamoDB tables
aws dynamodb create-table \
  --table-name ecom-products \
  --attribute-definitions AttributeName=productId,AttributeType=S \
  --key-schema AttributeName=productId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --endpoint-url http://localhost:8000

aws dynamodb create-table \
  --table-name ecom-users \
  --attribute-definitions AttributeName=userId,AttributeType=S \
  --key-schema AttributeName=userId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --endpoint-url http://localhost:8000

# Verify services are healthy
curl http://localhost:3000/health  # user-service
curl http://localhost:3001/health  # product-service
curl http://localhost:3002/health  # order-service
```

### 4. Add your first product

```bash
curl -X POST http://localhost:3001/products \
  -H 'Content-Type: application/json' \
  -d '{
    "name_en": "Wireless Headphones",
    "name_hi": "वायरलेस हेडफ़ोन",
    "name_bn": "ওয়্যারলেস হেডফোন",
    "description_en": "Premium sound with noise cancellation",
    "description_hi": "शोर रद्द करने के साथ प्रीमियम ध्वनि",
    "description_bn": "নয়েজ ক্যান্সেলেশন সহ প্রিমিয়াম সাউন্ড",
    "category": "electronics",
    "price": 2499,
    "stock": 100
  }'
```

---

## 🌍 Deploy to AWS

### Step 1 — Create infrastructure

```bash
# Create S3 bucket for Terraform state
aws s3 mb s3://ecommerce-tf-state-$(aws sts get-caller-identity --query Account --output text) \
  --region ap-south-1

# Update bucket name in infrastructure/terraform/main.tf
# Then deploy
cd infrastructure/terraform
terraform init
terraform plan
terraform apply   # takes ~20 min — creates VPC, ECS, DynamoDB, ElastiCache, ALB, CloudFront
```

### Step 2 — Set up Amazon Cognito (user registration)

Users register themselves — Cognito handles email verification and JWT tokens.

```
AWS Console → Cognito → Create user pool
  ├── Sign-in: Email
  ├── Self-registration: ENABLED  ← users can sign up themselves
  ├── Email verification: Automatic code
  └── App client: name it "ecom-frontend"

Note your:
  - User Pool ID  (ap-south-1_XXXXXXXXX)
  - App Client ID (long alphanumeric)
```

### Step 3 — Build and push Docker images

```bash
# Login to ECR
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS \
  --password-stdin YOUR_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com

# Build and push all services
for service in user-service product-service order-service payment-service; do
  docker build -t $service ./services/$service
  docker tag $service:latest YOUR_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/$service:latest
  docker push YOUR_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/$service:latest
done
```

### Step 4 — Deploy frontend

```bash
cd frontend

# Create production env file
cat > .env.production << EOF
REACT_APP_API_URL=https://YOUR_ALB_DNS
REACT_APP_COGNITO_USER_POOL_ID=ap-south-1_XXXXXXXXX
REACT_APP_COGNITO_CLIENT_ID=your-client-id
EOF

npm run build
aws s3 sync build/ s3://YOUR_FRONTEND_BUCKET --delete
aws cloudfront create-invalidation --distribution-id YOUR_CF_ID --paths "/*"
```

### Step 5 — Seed products in production

```bash
# Uses real AWS (no endpoint-url needed)
node seed-products.js
```

---

## 👤 User Registration Flow

Users register themselves — no admin action needed.

```
1. User visits your CloudFront URL
2. Clicks "Create account"
3. Enters email + password
4. Receives 6-digit verification code via email (sent by Cognito automatically)
5. Enters code → account confirmed
6. Cognito triggers post-registration Lambda
7. Lambda creates user record in DynamoDB (userId, email, language=en)
8. User receives JWT access token (1 hour validity)
9. All API calls include: Authorization: Bearer <token>
```

To add yourself as an **admin** (for product management):

```
AWS Console → Cognito → User Pools → Groups
  → Create group "admins"
  → Add your user to the group
```

---

## 📦 Product Management

Three ways to add products:

### Option A — Seed script (recommended for bulk adds)

```bash
# Edit seed-products.js with your product list, then:
node seed-products.js
```

### Option B — AWS Console (no code required)

```
AWS Console → DynamoDB → Tables → ecom-products
  → Explore table items → Create item → JSON view

Paste:
{
  "productId": {"S": "prod-001"},
  "name_en":   {"S": "Product Name"},
  "name_hi":   {"S": "उत्पाद का नाम"},
  "name_bn":   {"S": "পণ্যের নাম"},
  "category":  {"S": "electronics"},
  "price":     {"N": "999"},
  "stock":     {"N": "50"}
}
```

### Option C — Admin API (production approach)

```bash
# Must be in the "admins" Cognito group
curl -X POST https://YOUR_ALB_DNS/admin/products \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "name_en": "New Product", "price": 999, "stock": 50, "category": "electronics" }'
```

---

## ⚡ CI/CD Pipeline

Every push to `main` triggers the full pipeline:

```
Push to main
    │
    ▼
Run tests (jest)
    │
    ▼
Build Docker images
    │
    ▼
Push to ECR
    │
    ├──► Deploy to ECS Mumbai   (waits for stability)
    │
    └──► Deploy to ECS Singapore
         │
         └──► Deploy frontend to S3 + invalidate CloudFront
```

### Required GitHub Secrets

Set these in your repo under **Settings → Secrets and variables → Actions**:

| Secret | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |
| `AWS_ACCOUNT_ID` | 12-digit AWS account ID |
| `COGNITO_USER_POOL_ID` | `ap-south-1_XXXXXXXXX` |
| `COGNITO_CLIENT_ID` | Cognito app client ID |
| `API_URL` | ALB DNS or custom domain |
| `FRONTEND_BUCKET` | S3 bucket name |
| `CF_DISTRIBUTION_ID` | CloudFront distribution ID |
| `INTERNAL_API_KEY` | Random 32-char string |

---

## 🌐 Multi-Language Support

Language is set by the URL path and stored per-user in DynamoDB.

| URL | Language |
|---|---|
| `/en` | English |
| `/hi` | Hindi (हिंदी) |
| `/bn` | Bengali (বাংলা) |

Product names and descriptions are stored in all three languages in DynamoDB (`name_en`, `name_hi`, `name_bn`). The API returns the correct language based on the `?lang=` query parameter.

---

## 🔄 Failover Testing

Test that your multi-region setup actually works:

```bash
# Check current DNS — should show Mumbai
dig api.yourdomain.com

# Simulate Mumbai failure
aws route53 update-health-check \
  --health-check-id YOUR_HEALTH_CHECK_ID \
  --disabled \
  --region us-east-1

# Wait 60 seconds, then check — should now show Singapore
sleep 60 && dig api.yourdomain.com

# Re-enable after testing
aws route53 update-health-check \
  --health-check-id YOUR_HEALTH_CHECK_ID \
  --no-disabled \
  --region us-east-1
```

Route 53 automatically switches traffic in under 60 seconds when the primary region's health check fails 3 times in a row.

---

## 📊 Monitoring

All services write structured JSON logs to CloudWatch:

```json
{
  "timestamp": "2025-01-01T10:00:00.000Z",
  "level": "info",
  "service": "order-service",
  "region": "ap-south-1",
  "orderId": "uuid",
  "userId": "uuid",
  "total": 2499,
  "message": "Order placed"
}
```

CloudWatch alarms are configured for:
- HTTP 5xx error rate > 10/min
- ECS CPU utilization > 80%
- DynamoDB throttled requests > 5/min
- ElastiCache evictions spike

View the dashboard: **AWS Console → CloudWatch → Dashboards → ecommerce-overview**

---

## 💸 Cost Estimate

Running this in a dev/learning setup (minimal traffic):

| Service | Monthly Cost |
|---|---|
| ECS Fargate (4 services × 2 regions) | ~$30 |
| DynamoDB Global Tables (on-demand) | ~$5 |
| ElastiCache cache.t3.micro × 2 | ~$25 |
| Application Load Balancer × 2 | ~$20 |
| CloudFront + S3 | ~$5 |
| Route 53 | ~$2 |
| **Total** | **~$90/month** |

**Save money while not testing:** Scale ECS services to 0 tasks.

```bash
for service in user-service product-service order-service payment-service; do
  aws ecs update-service \
    --cluster global-ecommerce-mumbai \
    --service $service \
    --desired-count 0 \
    --region ap-south-1
done
```

---

## 🛡 Security

- **IAM**: ECS task roles with least-privilege policies — only the exact DynamoDB tables and SQS queues each service needs
- **Secrets**: All credentials stored in AWS Secrets Manager — never hardcoded or in environment variables in production
- **Auth**: Cognito JWT tokens verified on every protected endpoint using `aws-jwt-verify`
- **WAF**: AWS Managed Rules + rate limiting (1000 req/IP/min) in front of CloudFront
- **Network**: ECS tasks run in private subnets — only reachable through the ALB

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, AWS Amplify, i18next |
| Backend | Node.js 20, Express 4 |
| Auth | Amazon Cognito |
| Database | Amazon DynamoDB (Global Tables) |
| Cache | Amazon ElastiCache (Redis 7) |
| Container runtime | Amazon ECS Fargate |
| Container registry | Amazon ECR |
| Load balancing | Application Load Balancer |
| CDN | Amazon CloudFront + S3 |
| DNS + failover | Amazon Route 53 |
| Messaging | Amazon SQS (FIFO) + SNS |
| Email | Amazon SES |
| IaC | Terraform 1.7+ |
| CI/CD | GitHub Actions |
| Monitoring | Amazon CloudWatch |
| Security | AWS WAF, IAM, AWS Secrets Manager |

---

## 📋 API Reference

### User Service (`/users`)

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/health` | None | Health check |
| GET | `/users/:userId` | JWT | Get user profile |
| POST | `/users` | None | Create user (called by Cognito Lambda) |
| PATCH | `/users/:userId/language` | JWT | Update language preference |

### Product Service (`/products`)

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/health` | None | Health check |
| GET | `/products?lang=en&category=electronics` | None | List products |
| GET | `/products/:productId?lang=hi` | None | Get single product |
| POST | `/products` | None (add auth in prod) | Create product |
| POST | `/admin/products` | JWT + Admin group | Create product (admin) |

### Order Service (`/orders`)

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/health` | None | Health check |
| POST | `/orders` | JWT | Place an order |
| GET | `/orders` | JWT | List user's orders |
| PATCH | `/orders/:orderId/status` | Internal key | Update order status |

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -m 'Add your feature'`
4. Push to the branch: `git push origin feature/your-feature`
5. Open a Pull Request

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

---

## 🙏 Acknowledgements

Built as a learning project demonstrating AWS Solution Architect Associate level patterns including:
- Multi-region high availability architecture
- Event-driven microservices with SQS/SNS
- Infrastructure as Code with Terraform
- Containerized deployments with ECS Fargate
- Global data replication with DynamoDB Global Tables

---

<p align="center">
  Made with ☁️ on AWS &nbsp;|&nbsp; Mumbai + Singapore
</p>
