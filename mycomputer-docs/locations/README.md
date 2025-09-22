# Locations Service

## Overview

The Locations Service is a comprehensive geospatial database system that manages all MyComputer repair center locations across Europe. This service provides real-time location data, store information, and geographic search capabilities using PostgreSQL with PostGIS extensions and GeoJSON data formats.

## Key Features

- **Geospatial Database**: PostgreSQL with PostGIS for advanced geographic queries
- **GeoJSON Support**: Native support for GeoJSON data formats
- **Multi-region Coverage**: Complete coverage of European locations
- **Real-time Updates**: Live synchronization of location status and availability
- **Advanced Search**: Proximity-based searches and geographic filtering
- **High Performance**: Optimized spatial indexing for fast queries

## Technology Stack

- **Runtime**: Node.js 20.x LTS
- **Framework**: Next.js 14
- **Database**: PostgreSQL 15 with PostGIS 3.3
- **Container**: AWS ECS Fargate
- **Data Format**: GeoJSON
- **API**: RESTful with OpenAPI 3.0 specification

## Quick Start

### Prerequisites

- Node.js 20.x or higher
- PostgreSQL 15 with PostGIS extension
- AWS CLI configured
- Docker for local development

### Installation

```bash
# Clone the repository
git clone https://github.com/mycomputer/locations-service.git

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env

# Run database migrations
npm run migrate

# Start development server
npm run dev
```

### Environment Variables

```env
DATABASE_URL=postgresql://user:password@localhost:5432/locations
POSTGIS_VERSION=3.3
AWS_REGION=eu-west-1
ECS_CLUSTER=mycomputer-cluster
SERVICE_NAME=locations-service
```

## Architecture

The Locations Service follows a microservices architecture pattern with:

- **API Layer**: RESTful endpoints for location management
- **Business Logic**: Location validation and geographic calculations
- **Data Layer**: PostgreSQL with PostGIS for spatial data
- **Caching**: Redis for frequently accessed location data
- **Message Queue**: SQS for asynchronous location updates

## API Documentation

Full API documentation is available at:
- [API Reference](./reference/api.md)
- [OpenAPI Specification](./reference/openapi.yaml)

Key endpoints:
- `GET /api/locations` - List all locations
- `GET /api/locations/nearby` - Find nearby locations
- `GET /api/locations/{id}` - Get specific location details
- `POST /api/locations` - Create new location
- `PUT /api/locations/{id}` - Update location
- `DELETE /api/locations/{id}` - Remove location

## Database Schema

The service uses PostgreSQL with PostGIS extensions. Key tables include:
- `locations` - Store location details
- `location_services` - Available services per location
- `location_hours` - Operating hours
- `location_coverage` - Service coverage areas

See [Data Model](./reference/data-model.md) for complete schema.

## Deployment

The service is deployed on AWS ECS using Fargate:

```bash
# Build Docker image
docker build -t locations-service .

# Push to ECR
aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI
docker tag locations-service:latest $ECR_URI/locations-service:latest
docker push $ECR_URI/locations-service:latest

# Deploy to ECS
aws ecs update-service --cluster mycomputer-cluster --service locations-service --force-new-deployment
```

## Monitoring

- **CloudWatch**: Application logs and metrics
- **X-Ray**: Distributed tracing
- **Custom Dashboards**: Geographic query performance
- **Alerts**: Location availability and response times

## Security

- **Authentication**: JWT tokens via AWS Cognito
- **Authorization**: Role-based access control (RBAC)
- **Data Encryption**: TLS 1.3 in transit, AES-256 at rest
- **API Rate Limiting**: Per-client throttling
- **Input Validation**: GeoJSON schema validation

## Testing

```bash
# Run unit tests
npm test

# Run integration tests
npm run test:integration

# Run E2E tests
npm run test:e2e

# Generate coverage report
npm run test:coverage
```

## Contributing

Please read our [Contributing Guide](../common/CONTRIBUTING.md) for details on our code of conduct and the process for submitting pull requests.

## Support

For issues and questions:
- Internal Wiki: https://wiki.mycomputer.internal/locations
- Slack: #team-locations
- Email: locations-team@mycomputer.com

## License

Copyright © 2024 MyComputer. All rights reserved.
