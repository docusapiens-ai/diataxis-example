# Locations Service Data Model

## Overview

The Locations Service uses PostgreSQL with PostGIS extensions to manage geospatial data for all MyComputer repair center locations across Europe. The data model is optimized for geographic queries, proximity searches, and multi-language support.

## Database Configuration

```sql
-- Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;
CREATE EXTENSION IF NOT EXISTS postgis_tiger_geocoder;
CREATE EXTENSION IF NOT EXISTS fuzzystrmatch;

-- Enable UUID generation
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

## Core Tables

### 1. locations

Primary table storing all repair center locations with geospatial data.

```sql
CREATE TABLE locations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_code VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL, -- 'repair_center', 'service_point', 'warehouse', 'headquarters'
    status VARCHAR(50) NOT NULL DEFAULT 'active', -- 'active', 'inactive', 'maintenance', 'closed'
    
    -- Geographic data
    coordinates GEOGRAPHY(POINT, 4326) NOT NULL,
    geojson JSONB NOT NULL,
    
    -- Address information
    street_address VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state_province VARCHAR(100),
    postal_code VARCHAR(20) NOT NULL,
    country_code CHAR(2) NOT NULL, -- ISO 3166-1 alpha-2
    country_name VARCHAR(100) NOT NULL,
    
    -- Contact information
    phone_primary VARCHAR(50),
    phone_secondary VARCHAR(50),
    email VARCHAR(255),
    website_url VARCHAR(500),
    
    -- Capacity and features
    capacity_daily_repairs INTEGER,
    parking_available BOOLEAN DEFAULT false,
    wheelchair_accessible BOOLEAN DEFAULT true,
    
    -- Metadata
    opened_date DATE,
    last_renovation_date DATE,
    manager_id UUID,
    region_id UUID,
    timezone VARCHAR(50) NOT NULL,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP WITH TIME ZONE,
    
    -- Indexes
    CONSTRAINT chk_location_type CHECK (type IN ('repair_center', 'service_point', 'warehouse', 'headquarters')),
    CONSTRAINT chk_status CHECK (status IN ('active', 'inactive', 'maintenance', 'closed'))
);

-- Spatial index for geographic queries
CREATE INDEX idx_locations_coordinates ON locations USING GIST(coordinates);
CREATE INDEX idx_locations_status ON locations(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_locations_country ON locations(country_code);
CREATE INDEX idx_locations_city ON locations(city);
CREATE INDEX idx_locations_type ON locations(type);
```

### 2. location_services

Services available at each location.

```sql
CREATE TABLE location_services (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    service_code VARCHAR(50) NOT NULL,
    service_name VARCHAR(255) NOT NULL,
    category VARCHAR(100) NOT NULL, -- 'hardware_repair', 'software_support', 'data_recovery', 'consultation'
    
    -- Service details
    available BOOLEAN DEFAULT true,
    average_duration_minutes INTEGER,
    price_range_min DECIMAL(10, 2),
    price_range_max DECIMAL(10, 2),
    currency_code CHAR(3) DEFAULT 'EUR',
    
    -- Capabilities
    same_day_service BOOLEAN DEFAULT false,
    appointment_required BOOLEAN DEFAULT true,
    walk_in_available BOOLEAN DEFAULT false,
    remote_support_available BOOLEAN DEFAULT false,
    
    -- Metadata
    certification_required BOOLEAN DEFAULT false,
    technician_level_required VARCHAR(50), -- 'junior', 'mid', 'senior', 'specialist'
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(location_id, service_code)
);

CREATE INDEX idx_location_services_location ON location_services(location_id);
CREATE INDEX idx_location_services_category ON location_services(category);
CREATE INDEX idx_location_services_available ON location_services(available);
```

### 3. location_hours

Operating hours for each location.

```sql
CREATE TABLE location_hours (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    day_of_week INTEGER NOT NULL, -- 0=Sunday, 1=Monday, ..., 6=Saturday
    
    -- Regular hours
    open_time TIME,
    close_time TIME,
    is_closed BOOLEAN DEFAULT false,
    
    -- Break hours (e.g., lunch break)
    break_start_time TIME,
    break_end_time TIME,
    
    -- Special hours
    is_24_hours BOOLEAN DEFAULT false,
    appointment_only BOOLEAN DEFAULT false,
    
    -- Seasonal variations
    season_start_date DATE,
    season_end_date DATE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_day_of_week CHECK (day_of_week >= 0 AND day_of_week <= 6),
    UNIQUE(location_id, day_of_week, season_start_date)
);

CREATE INDEX idx_location_hours_location ON location_hours(location_id);
CREATE INDEX idx_location_hours_day ON location_hours(day_of_week);
```

### 4. location_coverage_areas

Service coverage areas for mobile repair services.

```sql
CREATE TABLE location_coverage_areas (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    area_name VARCHAR(255) NOT NULL,
    area_type VARCHAR(50) NOT NULL, -- 'primary', 'secondary', 'on_demand'
    
    -- Geographic boundary
    coverage_polygon GEOGRAPHY(POLYGON, 4326) NOT NULL,
    coverage_radius_km DECIMAL(10, 2),
    
    -- Service details
    mobile_service_available BOOLEAN DEFAULT true,
    pickup_service_available BOOLEAN DEFAULT true,
    additional_fee DECIMAL(10, 2),
    estimated_response_time_hours INTEGER,
    
    -- Population and demand metrics
    population_covered INTEGER,
    average_monthly_requests INTEGER,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_area_type CHECK (area_type IN ('primary', 'secondary', 'on_demand'))
);

CREATE INDEX idx_coverage_areas_location ON location_coverage_areas(location_id);
CREATE INDEX idx_coverage_areas_polygon ON location_coverage_areas USING GIST(coverage_polygon);
```

### 5. location_inventory_links

Links between locations and inventory items.

```sql
CREATE TABLE location_inventory_links (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    inventory_item_id UUID NOT NULL, -- References inventory service
    
    -- Stock levels
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    quantity_reserved INTEGER NOT NULL DEFAULT 0,
    quantity_available INTEGER GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved) STORED,
    
    -- Reorder points
    reorder_point INTEGER,
    reorder_quantity INTEGER,
    max_stock_level INTEGER,
    
    -- Location within facility
    storage_zone VARCHAR(50),
    bin_location VARCHAR(50),
    
    last_counted_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(location_id, inventory_item_id)
);

CREATE INDEX idx_inventory_links_location ON location_inventory_links(location_id);
CREATE INDEX idx_inventory_links_item ON location_inventory_links(inventory_item_id);
CREATE INDEX idx_inventory_links_available ON location_inventory_links(quantity_available);
```

### 6. location_staff

Staff assignments to locations.

```sql
CREATE TABLE location_staff (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    staff_id UUID NOT NULL, -- References user/staff service
    
    -- Role and position
    role VARCHAR(100) NOT NULL, -- 'manager', 'technician', 'support', 'admin'
    position_title VARCHAR(255),
    department VARCHAR(100),
    
    -- Assignment details
    assignment_type VARCHAR(50) NOT NULL, -- 'permanent', 'temporary', 'rotating'
    start_date DATE NOT NULL,
    end_date DATE,
    is_primary_location BOOLEAN DEFAULT true,
    
    -- Availability
    available_days VARCHAR(20), -- Comma-separated: 'mon,tue,wed,thu,fri'
    shift_pattern VARCHAR(50), -- 'morning', 'afternoon', 'evening', 'rotating'
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_assignment_type CHECK (assignment_type IN ('permanent', 'temporary', 'rotating'))
);

CREATE INDEX idx_location_staff_location ON location_staff(location_id);
CREATE INDEX idx_location_staff_staff ON location_staff(staff_id);
CREATE INDEX idx_location_staff_role ON location_staff(role);
```

### 7. location_metrics

Performance and operational metrics for each location.

```sql
CREATE TABLE location_metrics (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    metric_date DATE NOT NULL,
    
    -- Customer metrics
    total_customers INTEGER DEFAULT 0,
    new_customers INTEGER DEFAULT 0,
    returning_customers INTEGER DEFAULT 0,
    
    -- Service metrics
    repairs_completed INTEGER DEFAULT 0,
    average_repair_time_hours DECIMAL(10, 2),
    customer_satisfaction_score DECIMAL(3, 2), -- 0.00 to 5.00
    
    -- Financial metrics
    daily_revenue DECIMAL(15, 2),
    daily_costs DECIMAL(15, 2),
    currency_code CHAR(3) DEFAULT 'EUR',
    
    -- Efficiency metrics
    staff_utilization_rate DECIMAL(5, 2), -- Percentage
    inventory_turnover_rate DECIMAL(10, 2),
    first_time_fix_rate DECIMAL(5, 2), -- Percentage
    
    -- Queue metrics
    average_wait_time_minutes INTEGER,
    max_queue_length INTEGER,
    appointments_scheduled INTEGER,
    walk_ins_served INTEGER,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(location_id, metric_date)
);

CREATE INDEX idx_location_metrics_location ON location_metrics(location_id);
CREATE INDEX idx_location_metrics_date ON location_metrics(metric_date);
```

### 8. location_translations

Multi-language support for location information.

```sql
CREATE TABLE location_translations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    location_id UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    language_code VARCHAR(5) NOT NULL, -- e.g., 'en', 'de', 'fr', 'es'
    
    -- Translated fields
    name VARCHAR(255),
    description TEXT,
    street_address VARCHAR(255),
    city VARCHAR(100),
    
    -- Additional localized information
    directions_text TEXT,
    parking_instructions TEXT,
    public_transport_info TEXT,
    special_instructions TEXT,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(location_id, language_code)
);

CREATE INDEX idx_location_translations_location ON location_translations(location_id);
CREATE INDEX idx_location_translations_language ON location_translations(language_code);
```

## Views

### 1. v_active_locations

View of all active locations with essential information.

```sql
CREATE VIEW v_active_locations AS
SELECT 
    l.id,
    l.location_code,
    l.name,
    l.type,
    l.coordinates,
    l.city,
    l.country_code,
    l.status,
    COUNT(DISTINCT ls.id) as services_count,
    COUNT(DISTINCT lst.id) as staff_count
FROM locations l
LEFT JOIN location_services ls ON l.id = ls.location_id AND ls.available = true
LEFT JOIN location_staff lst ON l.id = lst.location_id AND lst.end_date IS NULL
WHERE l.status = 'active' 
    AND l.deleted_at IS NULL
GROUP BY l.id;
```

### 2. v_location_availability

Real-time availability view.

```sql
CREATE VIEW v_location_availability AS
SELECT 
    l.id,
    l.location_code,
    l.name,
    l.coordinates,
    CASE 
        WHEN EXTRACT(DOW FROM CURRENT_TIMESTAMP AT TIME ZONE l.timezone) = lh.day_of_week
            AND CURRENT_TIME BETWEEN lh.open_time AND lh.close_time
            AND NOT lh.is_closed
        THEN 'open'
        ELSE 'closed'
    END as current_status,
    lh.open_time,
    lh.close_time,
    l.capacity_daily_repairs,
    COALESCE(
        (SELECT COUNT(*) 
         FROM location_metrics lm 
         WHERE lm.location_id = l.id 
            AND lm.metric_date = CURRENT_DATE), 
        0
    ) as repairs_today
FROM locations l
LEFT JOIN location_hours lh ON l.id = lh.location_id
WHERE l.status = 'active'
    AND l.deleted_at IS NULL;
```

## Functions

### 1. Find Nearest Locations

```sql
CREATE OR REPLACE FUNCTION find_nearest_locations(
    user_lat DOUBLE PRECISION,
    user_lon DOUBLE PRECISION,
    max_distance_km INTEGER DEFAULT 50,
    limit_count INTEGER DEFAULT 10
)
RETURNS TABLE (
    location_id UUID,
    name VARCHAR,
    distance_km DOUBLE PRECISION,
    coordinates GEOGRAPHY,
    address TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        l.id,
        l.name,
        ST_Distance(l.coordinates, ST_MakePoint(user_lon, user_lat)::geography) / 1000 as distance_km,
        l.coordinates,
        CONCAT(l.street_address, ', ', l.city, ', ', l.country_name) as address
    FROM locations l
    WHERE l.status = 'active'
        AND l.deleted_at IS NULL
        AND ST_DWithin(
            l.coordinates,
            ST_MakePoint(user_lon, user_lat)::geography,
            max_distance_km * 1000
        )
    ORDER BY distance_km
    LIMIT limit_count;
END;
$$ LANGUAGE plpgsql;
```

### 2. Check Service Coverage

```sql
CREATE OR REPLACE FUNCTION check_service_coverage(
    user_lat DOUBLE PRECISION,
    user_lon DOUBLE PRECISION
)
RETURNS TABLE (
    location_id UUID,
    location_name VARCHAR,
    area_type VARCHAR,
    mobile_service BOOLEAN,
    additional_fee DECIMAL
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        l.id,
        l.name,
        lca.area_type,
        lca.mobile_service_available,
        lca.additional_fee
    FROM location_coverage_areas lca
    JOIN locations l ON lca.location_id = l.id
    WHERE ST_Contains(
        lca.coverage_polygon,
        ST_MakePoint(user_lon, user_lat)::geography
    )
    AND l.status = 'active'
    AND l.deleted_at IS NULL
    ORDER BY lca.area_type;
END;
$$ LANGUAGE plpgsql;
```

## Triggers

### 1. Update Timestamp Trigger

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to all tables
CREATE TRIGGER update_locations_updated_at BEFORE UPDATE ON locations
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_location_services_updated_at BEFORE UPDATE ON location_services
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Apply to other tables similarly...
```

## Indexes Summary

- **Spatial Indexes**: For geographic queries and proximity searches
- **Foreign Key Indexes**: For join performance
- **Status Indexes**: For filtering active/inactive locations
- **Date Indexes**: For time-based queries and metrics

## Data Integrity Constraints

- **Foreign Keys**: Maintain referential integrity across services
- **Check Constraints**: Validate enum values and business rules
- **Unique Constraints**: Prevent duplicate entries
- **NOT NULL**: Ensure required fields are populated

## Performance Considerations

1. **Partitioning**: Consider partitioning `location_metrics` by date for large datasets
2. **Materialized Views**: Create materialized views for complex geographic aggregations
3. **Connection Pooling**: Use PgBouncer for connection management
4. **Read Replicas**: Utilize AWS RDS read replicas for query distribution
5. **Caching Strategy**: Cache frequently accessed location data in Redis

## Migration Strategy

```sql
-- Example migration for adding new features
BEGIN;

-- Add new column with default
ALTER TABLE locations 
ADD COLUMN IF NOT EXISTS supports_express_service BOOLEAN DEFAULT false;

-- Backfill data based on business logic
UPDATE locations 
SET supports_express_service = true 
WHERE type = 'repair_center' 
    AND capacity_daily_repairs > 50;

COMMIT;
```

## Backup and Recovery

- **Continuous Backups**: AWS RDS automated backups with 7-day retention
- **Point-in-Time Recovery**: Available for the last 35 days
- **Geographic Redundancy**: Cross-region read replicas for disaster recovery
- **Export Strategy**: Regular exports to S3 in Parquet format for analytics
