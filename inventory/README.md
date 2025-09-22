# Inventory Management System

## Overview

The Inventory Management System is a core component of MyComputer's enterprise platform, responsible for tracking computer parts, managing stock levels, and coordinating supply chain operations across all European locations.

## Quick Start

### Prerequisites
- Node.js 18+
- PostgreSQL 14+
- Redis 6+
- Docker (optional)

### Installation

```bash
# Clone the repository
git clone https://gitlab.mycomputer.com/apps/inventory.git
cd inventory

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env

# Run database migrations
npm run migrate

# Start the application
npm run dev
```

### Docker Setup

```bash
# Build and run with Docker Compose
docker-compose up -d

# Check service health
curl http://localhost:3001/health
```

## Key Features

- **Real-time Stock Tracking**: Monitor inventory levels across all locations
- **Automated Reordering**: Intelligent stock replenishment based on usage patterns
- **Multi-warehouse Support**: Manage inventory across distributed warehouses
- **Barcode Integration**: Quick item lookup and tracking
- **Audit Trail**: Complete history of all inventory movements
- **Predictive Analytics**: Forecast demand based on historical data

## Architecture

The Inventory service follows a layered architecture:

```
┌─────────────────────────────────┐
│         API Gateway             │
├─────────────────────────────────┤
│      Controllers Layer          │
├─────────────────────────────────┤
│       Services Layer            │
├─────────────────────────────────┤
│     Repository Layer            │
├─────────────────────────────────┤
│    Database (PostgreSQL)        │
└─────────────────────────────────┘
```

## API Endpoints

### Core Endpoints

- `GET /api/items` - List all inventory items
- `POST /api/items` - Create new inventory item
- `GET /api/items/:id` - Get specific item details
- `PUT /api/items/:id` - Update item information
- `DELETE /api/items/:id` - Remove item from inventory

### Stock Management

- `POST /api/stock/receive` - Record stock receipt
- `POST /api/stock/transfer` - Transfer between locations
- `POST /api/stock/adjust` - Manual stock adjustment
- `GET /api/stock/levels` - Current stock levels

### Reporting

- `GET /api/reports/low-stock` - Items below reorder point
- `GET /api/reports/movement` - Stock movement history
- `GET /api/reports/valuation` - Inventory valuation

## Database Schema

### Main Tables

- `inventory_items` - Item master data
- `stock_levels` - Current stock by location
- `stock_movements` - Historical transactions
- `warehouses` - Warehouse locations
- `suppliers` - Supplier information
- `purchase_orders` - Order management

See [Data Model](./reference/data-model.md) for complete schema.

## Configuration

### Environment Variables

```bash
# Application
NODE_ENV=production
PORT=3001

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=inventory
DB_USER=inventory_user
DB_PASSWORD=secure_password

# Redis Cache
REDIS_URL=redis://localhost:6379

# AWS
AWS_REGION=eu-west-1
S3_BUCKET=mycomputer-inventory

# Monitoring
LOG_LEVEL=info
SENTRY_DSN=https://xxx@sentry.io/xxx
```

## Development

### Running Tests

```bash
# Unit tests
npm test

# Integration tests
npm run test:integration

# E2E tests
npm run test:e2e

# Coverage report
npm run test:coverage
```

### Code Quality

```bash
# Linting
npm run lint

# Type checking
npm run type-check

# Format code
npm run format
```

### Database Migrations

```bash
# Create new migration
npm run migrate:create add_new_field

# Run migrations
npm run migrate:up

# Rollback
npm run migrate:down
```

## Deployment

### Production Deployment

```bash
# Build application
npm run build

# Run production server
npm start
```

### AWS ECS Deployment

See [AWS Deployment Guide](../common/tutorials/aws-deployment.md) for detailed instructions.

## Monitoring

- **Health Check**: `GET /health`
- **Metrics**: `GET /metrics` (Prometheus format)
- **CloudWatch**: Integrated logging and metrics
- **X-Ray**: Distributed tracing enabled

## Security

- JWT authentication required for all endpoints
- Role-based access control (RBAC)
- Data encryption at rest and in transit
- Regular security audits and penetration testing

## API Documentation

- [OpenAPI/Swagger Specification](./reference/api.md)
- [Postman Collection](https://api.postman.com/collections/xxx)
- Interactive API Explorer: `http://localhost:3001/api-docs`

## Troubleshooting

### Common Issues

1. **Database Connection Failed**
   - Check PostgreSQL is running
   - Verify connection credentials
   - Ensure database exists

2. **Redis Connection Error**
   - Verify Redis server is running
   - Check connection string
   - Review firewall rules

3. **Stock Discrepancies**
   - Run reconciliation report
   - Check audit logs
   - Verify transaction completeness

## Support

- **Documentation**: [Full Documentation](./index.md)
- **Issues**: [GitLab Issues](https://gitlab.mycomputer.com/apps/inventory/issues)
- **Team Chat**: #inventory-team on Slack
- **On-call**: Via PagerDuty

## License

Copyright © 2024 MyComputer Ltd. All rights reserved.

## Contributing

Please read [CONTRIBUTING.md](./CONTRIBUTING.md) for details on our code of conduct and the process for submitting pull requests.

## Version History

- **v2.0.0** - Current version with multi-warehouse support
- **v1.5.0** - Added predictive analytics
- **v1.0.0** - Initial release

See [CHANGELOG.md](./CHANGELOG.md) for detailed version history.
