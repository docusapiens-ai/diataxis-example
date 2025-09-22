# Locations Service API Reference

## Overview

The Locations Service API provides comprehensive endpoints for managing MyComputer repair center locations across Europe. This RESTful API supports geospatial queries, location management, and service availability checks.

## Base URL

```
Production: https://api.mycomputer.eu/locations/v1
Staging: https://api-staging.mycomputer.eu/locations/v1
Development: http://localhost:3000/api/locations/v1
```

## Authentication

All API requests require JWT authentication via Bearer token:

```http
Authorization: Bearer <jwt_token>
```

## OpenAPI Specification

```yaml
openapi: 3.0.3
info:
  title: MyComputer Locations Service API
  description: Geospatial location management system for MyComputer repair centers
  version: 1.0.0
  contact:
    name: Locations Team
    email: locations-team@mycomputer.com
servers:
  - url: https://api.mycomputer.eu/locations/v1
    description: Production server
  - url: https://api-staging.mycomputer.eu/locations/v1
    description: Staging server
  - url: http://localhost:3000/api/locations/v1
    description: Development server

security:
  - bearerAuth: []

paths:
  /locations:
    get:
      summary: List all locations
      description: Retrieve a paginated list of all locations with optional filtering
      operationId: listLocations
      tags:
        - Locations
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
            minimum: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            minimum: 1
            maximum: 100
        - name: status
          in: query
          schema:
            type: string
            enum: [active, inactive, maintenance, closed]
        - name: type
          in: query
          schema:
            type: string
            enum: [repair_center, service_point, warehouse, headquarters]
        - name: country_code
          in: query
          schema:
            type: string
            pattern: '^[A-Z]{2}$'
        - name: city
          in: query
          schema:
            type: string
        - name: services
          in: query
          description: Comma-separated list of service codes
          schema:
            type: string
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Location'
                  pagination:
                    $ref: '#/components/schemas/Pagination'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '500':
          $ref: '#/components/responses/InternalServerError'

    post:
      summary: Create a new location
      description: Add a new repair center or service point location
      operationId: createLocation
      tags:
        - Locations
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LocationCreate'
      responses:
        '201':
          description: Location created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Location'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '409':
          $ref: '#/components/responses/Conflict'
        '500':
          $ref: '#/components/responses/InternalServerError'

  /locations/{locationId}:
    get:
      summary: Get location details
      description: Retrieve detailed information about a specific location
      operationId: getLocation
      tags:
        - Locations
      parameters:
        - name: locationId
          in: path
          required: true
          schema:
            type: string
            format: uuid
        - name: include
          in: query
          description: Comma-separated list of related data to include
          schema:
            type: string
            enum: [services, hours, staff, metrics, coverage]
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/LocationDetail'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
        '500':
          $ref: '#/components/responses/InternalServerError'

    put:
      summary: Update location
      description: Update an existing location's information
      operationId: updateLocation
      tags:
        - Locations
      parameters:
        - name: locationId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LocationUpdate'
      responses:
        '200':
          description: Location updated successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Location'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
        '500':
          $ref: '#/components/responses/InternalServerError'

    delete:
      summary: Delete location
      description: Soft delete a location (marks as deleted but retains data)
      operationId: deleteLocation
      tags:
        - Locations
      parameters:
        - name: locationId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '204':
          description: Location deleted successfully
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
        '500':
          $ref: '#/components/responses/InternalServerError'

  /locations/nearby:
    get:
      summary: Find nearby locations
      description: Search for locations within a specified radius of coordinates
      operationId: findNearbyLocations
      tags:
        - Geographic Search
      parameters:
        - name: latitude
          in: query
          required: true
          schema:
            type: number
            format: double
            minimum: -90
            maximum: 90
        - name: longitude
          in: query
          required: true
          schema:
            type: number
            format: double
            minimum: -180
            maximum: 180
        - name: radius
          in: query
          description: Search radius in kilometers
          schema:
            type: integer
            default: 50
            minimum: 1
            maximum: 500
        - name: limit
          in: query
          schema:
            type: integer
            default: 10
            minimum: 1
            maximum: 50
        - name: services
          in: query
          description: Filter by required services (comma-separated)
          schema:
            type: string
        - name: open_now
          in: query
          description: Only return currently open locations
          schema:
            type: boolean
            default: false
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/NearbyLocation'
                  search_center:
                    type: object
                    properties:
                      latitude:
                        type: number
                      longitude:
                        type: number
                      radius_km:
                        type: integer
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '500':
          $ref: '#/components/responses/InternalServerError'

  /locations/coverage:
    post:
      summary: Check service coverage
      description: Verify if a specific location is within service coverage area
      operationId: checkCoverage
      tags:
        - Geographic Search
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - latitude
                - longitude
              properties:
                latitude:
                  type: number
                  format: double
                longitude:
                  type: number
                  format: double
                service_type:
                  type: string
                  enum: [mobile_repair, pickup, delivery]
      responses:
        '200':
          description: Coverage check result
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CoverageResult'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '500':
          $ref: '#/components/responses/InternalServerError'

  /locations/{locationId}/services:
    get:
      summary: List location services
      description: Get all services available at a specific location
      operationId: getLocationServices
      tags:
        - Location Services
      parameters:
        - name: locationId
          in: path
          required: true
          schema:
            type: string
            format: uuid
        - name: category
          in: query
          schema:
            type: string
            enum: [hardware_repair, software_support, data_recovery, consultation]
        - name: available_only
          in: query
          schema:
            type: boolean
            default: true
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/LocationService'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
        '500':
          $ref: '#/components/responses/InternalServerError'

    post:
      summary: Add service to location
      description: Add a new service offering to a location
      operationId: addLocationService
      tags:
        - Location Services
      parameters:
        - name: locationId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LocationServiceCreate'
      responses:
        '201':
          description: Service added successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/LocationService'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
        '409':
          $ref: '#/components/responses/Conflict'
        '500':
          $ref: '#/components/responses/InternalServerError'

  /locations/{locationId}/hours:
    get:
      summary: Get location hours
      description: Retrieve operating hours for a location
      operationId: getLocationHours
      tags:
        - Location Hours
      parameters:
        - name: locationId
          in: path
          required: true
          schema:
            type: string
            format: uuid
        - name: date
          in: query
          description: Specific date to check (for seasonal hours)
          schema:
            type: string
            format: date
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/LocationHours'
                  is_open_now:
                    type: boolean
                  next_open_time:
                    type: string
                    format: date-time
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
        '500':
          $ref: '#/components/responses/InternalServerError'

    put:
      summary: Update location hours
      description: Update operating hours for a location
      operationId: updateLocationHours
      tags:
        - Location Hours
      parameters:
        - name: locationId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: array
              items:
                $ref: '#/components/schemas/LocationHoursUpdate'
      responses:
        '200':
          description: Hours updated successfully
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/LocationHours'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
        '500':
          $ref: '#/components/responses/InternalServerError'

  /locations/{locationId}/metrics:
    get:
      summary: Get location metrics
      description: Retrieve performance metrics for a location
      operationId: getLocationMetrics
      tags:
        - Location Metrics
      parameters:
        - name: locationId
          in: path
          required: true
          schema:
            type: string
            format: uuid
        - name: start_date
          in: query
          required: true
          schema:
            type: string
            format: date
        - name: end_date
          in: query
          required: true
          schema:
            type: string
            format: date
        - name: aggregation
          in: query
          schema:
            type: string
            enum: [daily, weekly, monthly]
            default: daily
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/LocationMetrics'
                  summary:
                    $ref: '#/components/schemas/MetricsSummary'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
        '500':
          $ref: '#/components/responses/InternalServerError'

  /locations/geocode:
    post:
      summary: Geocode address
      description: Convert an address to geographic coordinates
      operationId: geocodeAddress
      tags:
        - Geographic Search
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - address
              properties:
                address:
                  type: string
                  description: Full address to geocode
                country_code:
                  type: string
                  pattern: '^[A-Z]{2}$'
                  description: ISO country code to restrict search
      responses:
        '200':
          description: Geocoding result
          content:
            application/json:
              schema:
                type: object
                properties:
                  latitude:
                    type: number
                    format: double
                  longitude:
                    type: number
                    format: double
                  formatted_address:
                    type: string
                  confidence:
                    type: number
                    format: float
                    minimum: 0
                    maximum: 1
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          description: Address not found
        '500':
          $ref: '#/components/responses/InternalServerError'

  /locations/bulk:
    post:
      summary: Bulk import locations
      description: Import multiple locations at once
      operationId: bulkImportLocations
      tags:
        - Bulk Operations
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                locations:
                  type: array
                  items:
                    $ref: '#/components/schemas/LocationCreate'
                validate_only:
                  type: boolean
                  default: false
                  description: If true, only validate without importing
      responses:
        '202':
          description: Import job accepted
          content:
            application/json:
              schema:
                type: object
                properties:
                  job_id:
                    type: string
                    format: uuid
                  status:
                    type: string
                    enum: [pending, processing, completed, failed]
                  total_records:
                    type: integer
                  validation_errors:
                    type: array
                    items:
                      type: object
                      properties:
                        row:
                          type: integer
                        errors:
                          type: array
                          items:
                            type: string
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '500':
          $ref: '#/components/responses/InternalServerError'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    Location:
      type: object
      properties:
        id:
          type: string
          format: uuid
        location_code:
          type: string
        name:
          type: string
        type:
          type: string
          enum: [repair_center, service_point, warehouse, headquarters]
        status:
          type: string
          enum: [active, inactive, maintenance, closed]
        coordinates:
          $ref: '#/components/schemas/Coordinates'
        address:
          $ref: '#/components/schemas/Address'
        contact:
          $ref: '#/components/schemas/Contact'
        features:
          $ref: '#/components/schemas/Features'
        timezone:
          type: string
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time

    LocationDetail:
      allOf:
        - $ref: '#/components/schemas/Location'
        - type: object
          properties:
            services:
              type: array
              items:
                $ref: '#/components/schemas/LocationService'
            hours:
              type: array
              items:
                $ref: '#/components/schemas/LocationHours'
            coverage_areas:
              type: array
              items:
                $ref: '#/components/schemas/CoverageArea'
            current_metrics:
              $ref: '#/components/schemas/LocationMetrics'

    LocationCreate:
      type: object
      required:
        - location_code
        - name
        - type
        - coordinates
        - address
        - timezone
      properties:
        location_code:
          type: string
        name:
          type: string
        type:
          type: string
          enum: [repair_center, service_point, warehouse, headquarters]
        coordinates:
          $ref: '#/components/schemas/Coordinates'
        address:
          $ref: '#/components/schemas/Address'
        contact:
          $ref: '#/components/schemas/Contact'
        features:
          $ref: '#/components/schemas/Features'
        timezone:
          type: string

    LocationUpdate:
      type: object
      properties:
        name:
          type: string
        status:
          type: string
          enum: [active, inactive, maintenance, closed]
        address:
          $ref: '#/components/schemas/Address'
        contact:
          $ref: '#/components/schemas/Contact'
        features:
          $ref: '#/components/schemas/Features'

    NearbyLocation:
      allOf:
        - $ref: '#/components/schemas/Location'
        - type: object
          properties:
            distance_km:
              type: number
              format: double
            travel_time_minutes:
              type: integer
            is_open:
              type: boolean

    Coordinates:
      type: object
      properties:
        latitude:
          type: number
          format: double
        longitude:
          type: number
          format: double

    Address:
      type: object
      required:
        - street_address
        - city
        - postal_code
        - country_code
      properties:
        street_address:
          type: string
        city:
          type: string
        state_province:
          type: string
        postal_code:
          type: string
        country_code:
          type: string
          pattern: '^[A-Z]{2}$'
        country_name:
          type: string

    Contact:
      type: object
      properties:
        phone_primary:
          type: string
        phone_secondary:
          type: string
        email:
          type: string
          format: email
        website_url:
          type: string
          format: uri

    Features:
      type: object
      properties:
        capacity_daily_repairs:
          type: integer
        parking_available:
          type: boolean
        wheelchair_accessible:
          type: boolean
        express_service:
          type: boolean
        appointment_booking:
          type: boolean

    LocationService:
      type: object
      properties:
        id:
          type: string
          format: uuid
        service_code:
          type: string
        service_name:
          type: string
        category:
          type: string
          enum: [hardware_repair, software_support, data_recovery, consultation]
        available:
          type: boolean
        average_duration_minutes:
          type: integer
        price_range:
          type: object
          properties:
            min:
              type: number
              format: decimal
            max:
              type: number
              format: decimal
            currency:
              type: string
        capabilities:
          type: object
          properties:
            same_day_service:
              type: boolean
            appointment_required:
              type: boolean
            walk_in_available:
              type: boolean
            remote_support_available:
              type: boolean

    LocationServiceCreate:
      type: object
      required:
        - service_code
        - service_name
        - category
      properties:
        service_code:
          type: string
        service_name:
          type: string
        category:
          type: string
          enum: [hardware_repair, software_support, data_recovery, consultation]
        average_duration_minutes:
          type: integer
        price_range_min:
          type: number
          format: decimal
        price_range_max:
          type: number
          format: decimal
        same_day_service:
          type: boolean
        appointment_required:
          type: boolean
        walk_in_available:
          type: boolean

    LocationHours:
      type: object
      properties:
        day_of_week:
          type: integer
          minimum: 0
          maximum: 6
        day_name:
          type: string
        open_time:
          type: string
          format: time
        close_time:
          type: string
          format: time
        is_closed:
          type: boolean
        is_24_hours:
          type: boolean
        break_start_time:
          type: string
          format: time
        break_end_time:
          type: string
          format: time
        appointment_only:
          type: boolean

    LocationHoursUpdate:
      type: object
      required:
        - day_of_week
      properties:
        day_of_week:
          type: integer
          minimum: 0
          maximum: 6
        open_time:
          type: string
          format: time
        close_time:
          type: string
          format: time
        is_closed:
          type: boolean
        is_24_hours:
          type: boolean
        break_start_time:
          type: string
          format: time
        break_end_time:
          type: string
          format: time

    LocationMetrics:
      type: object
      properties:
        metric_date:
          type: string
          format: date
        customer_metrics:
          type: object
          properties:
            total_customers:
              type: integer
            new_customers:
              type: integer
            returning_customers:
              type: integer
        service_metrics:
          type: object
          properties:
            repairs_completed:
              type: integer
            average_repair_time_hours:
              type: number
              format: double
            customer_satisfaction_score:
              type: number
              format: double
        financial_metrics:
          type: object
          properties:
            daily_revenue:
              type: number
              format: decimal
            daily_costs:
              type: number
              format: decimal
            currency_code:
              type: string
        efficiency_metrics:
          type: object
          properties:
            staff_utilization_rate:
              type: number
              format: double
            inventory_turnover_rate:
              type: number
              format: double
            first_time_fix_rate:
              type: number
              format: double

    MetricsSummary:
      type: object
      properties:
        period:
          type: object
          properties:
            start_date:
              type: string
              format: date
            end_date:
              type: string
              format: date
        totals:
          type: object
          properties:
            total_customers:
              type: integer
            total_repairs:
              type: integer
            total_revenue:
              type: number
              format: decimal
        averages:
          type: object
          properties:
            daily_customers:
              type: number
              format: double
            satisfaction_score:
              type: number
              format: double
            repair_time_hours:
              type: number
              format: double

    CoverageArea:
      type: object
      properties:
        id:
          type: string
          format: uuid
        area_name:
          type: string
        area_type:
          type: string
          enum: [primary, secondary, on_demand]
        coverage_radius_km:
          type: number
          format: double
        mobile_service_available:
          type: boolean
        pickup_service_available:
          type: boolean
        additional_fee:
          type: number
          format: decimal
        estimated_response_time_hours:
          type: integer

    CoverageResult:
      type: object
      properties:
        is_covered:
          type: boolean
        coverage_type:
          type: string
          enum: [primary, secondary, on_demand, not_covered]
        nearest_location:
          $ref: '#/components/schemas/NearbyLocation'
        service_options:
          type: array
          items:
            type: object
            properties:
              service_type:
                type: string
              available:
                type: boolean
              additional_fee:
                type: number
                format: decimal
              estimated_time:
                type: integer

    Pagination:
      type: object
      properties:
        page:
          type: integer
        limit:
          type: integer
        total_items:
          type: integer
        total_pages:
          type: integer
        has_next:
          type: boolean
        has_previous:
          type: boolean

    Error:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
            message:
              type: string
            details:
              type: array
              items:
                type: object
                properties:
                  field:
                    type: string
                  message:
                    type: string
            request_id:
              type: string
              format: uuid
            timestamp:
              type: string
              format: date-time

  responses:
    BadRequest:
      description: Bad request
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    
    Unauthorized:
      description: Unauthorized
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    
    Conflict:
      description: Resource conflict
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    
    InternalServerError:
      description: Internal server error
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

## Rate Limiting

API requests are rate-limited per client:
- **Standard tier**: 1000 requests per hour
- **Premium tier**: 5000 requests per hour
- **Enterprise tier**: Custom limits

Rate limit headers are included in all responses:
- `X-RateLimit-Limit`: Maximum requests per hour
- `X-RateLimit-Remaining`: Remaining requests in current window
- `X-RateLimit-Reset`: Unix timestamp when limit resets

## Webhooks

The Locations Service supports webhooks for real-time updates:

### Available Events
- `location.created`
- `location.updated`
- `location.deleted`
- `location.status_changed`
- `location.service_added`
- `location.service_removed`
- `location.hours_updated`

### Webhook Payload Example
```json
{
  "event": "location.updated",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "location_id": "123e4567-e89b-12d3-a456-426614174000",
    "changes": {
      "status": {
        "old": "active",
        "new": "maintenance"
      }
    }
  }
}
```

## Error Codes

| Code | Description |
|------|-------------|
| `LOC001` | Invalid coordinates |
| `LOC002` | Location code already exists |
| `LOC003` | Invalid country code |
| `LOC004` | Service not available at location |
| `LOC005` | Coverage area overlap detected |
| `LOC006` | Invalid operating hours |
| `LOC007` | Geocoding failed |
| `LOC008` | Maximum locations per region exceeded |
| `LOC009` | Invalid timezone |
| `LOC010` | Bulk import validation failed |

## SDK Examples

### JavaScript/TypeScript
```typescript
import { LocationsClient } from '@mycomputer/locations-sdk';

const client = new LocationsClient({
  apiKey: process.env.API_KEY,
  environment: 'production'
});

// Find nearby locations
const nearby = await client.findNearby({
  latitude: 52.520008,
  longitude: 13.404954,
  radius: 25,
  services: ['hardware_repair', 'data_recovery']
});

// Get location details
const location = await client.getLocation('location-id', {
  include: ['services', 'hours', 'metrics']
});
```

### Python
```python
from mycomputer_locations import LocationsClient

client = LocationsClient(
    api_key=os.environ['API_KEY'],
    environment='production'
)

# Check coverage
coverage = client.check_coverage(
    latitude=48.8566,
    longitude=2.3522,
    service_type='mobile_repair'
)

# Create new location
location = client.create_location({
    'location_code': 'PAR001',
    'name': 'MyComputer Paris Central',
    'type': 'repair_center',
    'coordinates': {
        'latitude': 48.8566,
        'longitude': 2.3522
    },
    'address': {
        'street_address': '123 Rue de la Paix',
        'city': 'Paris',
        'postal_code': '
