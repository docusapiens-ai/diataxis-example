# Getting Started with MyComputer Systems

## Introduction

Welcome to MyComputer's enterprise documentation. This tutorial will guide you through your first steps with our integrated suite of applications for computer repair services. By the end of this tutorial, you will have a working understanding of our core systems and how they work together.

## What You'll Learn

In this tutorial, we will:
- Set up your development environment
- Access each of the four core applications
- Create your first entities in each system
- Understand how data flows between applications

## Prerequisites

Before starting, ensure you have:
- AWS account with appropriate permissions
- Node.js 18+ installed
- Docker Desktop running
- Access to MyComputer's GitLab repositories

## Step 1: Clone the Repositories

First, let's clone all four application repositories:

```bash
git clone https://gitlab.mycomputer.com/apps/inventory.git
git clone https://gitlab.mycomputer.com/apps/customer-base.git
git clone https://gitlab.mycomputer.com/apps/payments.git
git clone https://gitlab.mycomputer.com/apps/locations.git
```

You should see confirmation messages for each repository cloned.

## Step 2: Set Up Environment Variables

Each application requires environment configuration. Create `.env` files in each project:

```bash
cd inventory
cp .env.example .env

cd ../customer-base
cp .env.example .env

cd ../payments
cp .env.example .env

cd ../locations
cp .env.example .env
```

Notice how each application maintains its own configuration - this is part of our microservices architecture.

## Step 3: Install Dependencies

Now, install dependencies for each application:

```bash
cd inventory && npm install
cd ../customer-base && npm install
cd ../payments && npm install
cd ../locations && npm install
```

The installation will download all required packages. This typically takes 2-3 minutes per application.

## Step 4: Start the Applications

We'll use Docker Compose to start all services:

```bash
cd ..
docker-compose up -d
```

You should see output indicating that all containers are starting:
- `inventory-api` on port 3001
- `customer-base-api` on port 3002
- `customer-base-frontend` on port 3000
- `payments-api` on port 3003
- `locations-api` on port 3004

## Step 5: Verify the Installation

Let's verify each service is running:

```bash
curl http://localhost:3001/health  # Inventory
curl http://localhost:3002/health  # Customer Base
curl http://localhost:3003/health  # Payments
curl http://localhost:3004/health  # Locations
```

Each should return: `{"status":"healthy","timestamp":"..."}`

## Step 6: Access the Customer Base UI

Open your browser and navigate to:
```
http://localhost:3000
```

You'll see the MyComputer Customer Base login screen. Use the default development credentials:
- Username: `admin@mycomputer.com`
- Password: `DevPassword123!`

## Step 7: Create Your First Customer

Once logged in:

1. Click "New Customer" in the navigation
2. Fill in the customer details:
   - Name: "Test Customer"
   - Email: "test@example.com"
   - Phone: "+44 20 1234 5678"
3. Click "Save"

Notice the success message: "Customer created successfully"

## Step 8: Add an Inventory Item

Using the API (we'll use curl for this tutorial):

```bash
curl -X POST http://localhost:3001/api/items \
  -H "Content-Type: application/json" \
  -d '{
    "name": "RAM Module 8GB DDR4",
    "sku": "RAM-8GB-001",
    "quantity": 50,
    "price": 45.99
  }'
```

You'll receive a response with the created item's ID.

## Step 9: Create a Service Location

```bash
curl -X POST http://localhost:3004/api/locations \
  -H "Content-Type: application/json" \
  -d '{
    "name": "London Repair Center",
    "address": "123 Tech Street, London",
    "coordinates": {
      "type": "Point",
      "coordinates": [-0.1276, 51.5074]
    }
  }'
```

## Step 10: Process a Test Payment

```bash
curl -X POST http://localhost:3003/api/payments \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 99.99,
    "currency": "GBP",
    "customer_id": "cust_001",
    "description": "Computer repair service"
  }'
```

## What You've Accomplished

Congratulations! You have:
- ✅ Set up all four MyComputer applications
- ✅ Created your first customer record
- ✅ Added inventory items
- ✅ Registered a service location
- ✅ Processed a test payment

## Next Steps

Now that you have a working environment, explore:
- [AWS Deployment Tutorial](./aws-deployment.md) - Deploy to production
- [How to Configure Auto-scaling](../how-to/configure-scaling.md) - Set up scalability
- [System Architecture](../explanation/architecture-overview.md) - Understand the big picture

Remember: This is your development environment. Changes here won't affect production systems.
