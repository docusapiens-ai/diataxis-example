# Inventory Data Model

## Database Schema

The Inventory Management System uses PostgreSQL 14+ with the following schema design.

## Entity Relationship Diagram

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│ inventory_items │──────│  stock_levels    │──────│   warehouses    │
└─────────────────┘      └──────────────────┘      └─────────────────┘
         │                        │                          │
         │                        │                          │
         ▼                        ▼                          │
┌─────────────────┐      ┌──────────────────┐              │
│    suppliers    │      │ stock_movements  │──────────────┘
└─────────────────┘      └──────────────────┘
         │                        │
         │                        │
         ▼                        ▼
┌─────────────────┐      ┌──────────────────┐
│ purchase_orders │──────│ purchase_order_  │
└─────────────────┘      │     items        │
                         └──────────────────┘
```

## Tables

### inventory_items

Stores master data for all inventory items.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY | Unique identifier |
| sku | VARCHAR(50) | UNIQUE, NOT NULL | Stock keeping unit |
| name | VARCHAR(200) | NOT NULL | Item name |
| description | TEXT | | Detailed description |
| category | VARCHAR(100) | NOT NULL | Item category |
| subcategory | VARCHAR(100) | | Item subcategory |
| brand | VARCHAR(100) | | Brand name |
| model | VARCHAR(100) | | Model number |
| unit_cost | DECIMAL(10,2) | NOT NULL | Cost per unit |
| selling_price | DECIMAL(10,2) | NOT NULL | Selling price |
| weight_kg | DECIMAL(8,3) | | Weight in kilograms |
| dimensions_cm | JSONB | | {length, width, height} |
| barcode | VARCHAR(50) | UNIQUE | Barcode/EAN |
| qr_code | VARCHAR(200) | | QR code data |
| min_stock_level | INTEGER | DEFAULT 0 | Minimum stock threshold |
| reorder_point | INTEGER | DEFAULT 0 | Automatic reorder trigger |
| reorder_quantity | INTEGER | DEFAULT 0 | Standard reorder amount |
| lead_time_days | INTEGER | DEFAULT 7 | Supplier lead time |
| is_active | BOOLEAN | DEFAULT true | Active status |
| is_serialized | BOOLEAN | DEFAULT false | Requires serial tracking |
| warranty_months | INTEGER | | Warranty period |
| metadata | JSONB | | Additional attributes |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |
| created_by | UUID | NOT NULL | User who created |
| updated_by | UUID | NOT NULL | User who last updated |

**Indexes:**
- `idx_inventory_items_sku` ON (sku)
- `idx_inventory_items_category` ON (category, subcategory)
- `idx_inventory_items_barcode` ON (barcode)
- `idx_inventory_items_active` ON (is_active)

### stock_levels

Current stock quantities by location.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY | Unique identifier |
| item_id | UUID | FOREIGN KEY | Reference to inventory_items |
| warehouse_id | UUID | FOREIGN KEY | Reference to warehouses |
| quantity_on_hand | INTEGER | NOT NULL DEFAULT 0 | Available stock |
| quantity_reserved | INTEGER | NOT NULL DEFAULT 0 | Reserved for orders |
| quantity_in_transit | INTEGER | NOT NULL DEFAULT 0 | Being transferred |
| quantity_damaged | INTEGER | NOT NULL DEFAULT 0 | Damaged items |
| bin_location | VARCHAR(50) | | Physical location in warehouse |
| last_counted_at | TIMESTAMP | | Last physical count date |
| last_movement_at | TIMESTAMP | | Last stock movement |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

**Indexes:**
- `idx_stock_levels_item_warehouse` UNIQUE ON (item_id, warehouse_id)
- `idx_stock_levels_warehouse` ON (warehouse_id)
- `idx_stock_levels_quantity` ON (quantity_on_hand)

**Constraints:**
- CHECK (quantity_on_hand >= 0)
- CHECK (quantity_reserved >= 0)
- CHECK (quantity_reserved <= quantity_on_hand)

### stock_movements

Historical record of all stock transactions.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY | Unique identifier |
| movement_type | VARCHAR(50) | NOT NULL | Type of movement |
| item_id | UUID | FOREIGN KEY | Reference to inventory_items |
| from_warehouse_id | UUID | FOREIGN KEY | Source warehouse |
| to_warehouse_id | UUID | FOREIGN KEY | Destination warehouse |
| quantity | INTEGER | NOT NULL | Quantity moved |
| unit_cost | DECIMAL(10,2) | | Cost at time of movement |
| reference_type | VARCHAR(50) | | Related entity type |
| reference_id | UUID | | Related entity ID |
| reason | VARCHAR(200) | | Movement reason |
| notes | TEXT | | Additional notes |
| serial_numbers | TEXT[] | | Array of serial numbers |
| batch_number | VARCHAR(50) | | Batch/lot number |
| expiry_date | DATE | | Expiration date |
| performed_by | UUID | NOT NULL | User who performed |
| approved_by | UUID | | User who approved |
| movement_date | TIMESTAMP | NOT NULL | When movement occurred |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |

**Movement Types:**
- `RECEIPT` - Goods received from supplier
- `ISSUE` - Issued for repair/sale
- `TRANSFER` - Inter-warehouse transfer
- `ADJUSTMENT` - Manual adjustment
- `RETURN` - Customer return
- `DAMAGE` - Damaged goods
- `DISPOSAL` - Disposed items
- `FOUND` - Found during count

**Indexes:**
- `idx_stock_movements_item` ON (item_id)
- `idx_stock_movements_warehouses` ON (from_warehouse_id, to_warehouse_id)
- `idx_stock_movements_date` ON (movement_date)
- `idx_stock_movements_type` ON (movement_type)

### warehouses

Physical warehouse/location information.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY | Unique identifier |
| code | VARCHAR(20) | UNIQUE, NOT NULL | Warehouse code |
| name | VARCHAR(100) | NOT NULL | Warehouse name |
| type | VARCHAR(50) | NOT NULL | Warehouse type |
| address_line1 | VARCHAR(200) | NOT NULL | Street address |
| address_line2 | VARCHAR(200) | | Additional address |
| city | VARCHAR(100) | NOT NULL | City |
| state_province | VARCHAR(100) | | State/Province |
| postal_code | VARCHAR(20) | | Postal code |
| country | VARCHAR(2) | NOT NULL | ISO country code |
| latitude | DECIMAL(10,8) | | GPS latitude |
| longitude | DECIMAL(11,8) | | GPS longitude |
| phone | VARCHAR(50) | | Contact phone |
| email | VARCHAR(100) | | Contact email |
| manager_id | UUID | | Warehouse manager |
| capacity_m3 | DECIMAL(10,2) | | Storage capacity |
| operating_hours | JSONB | | Operating schedule |
| is_active | BOOLEAN | DEFAULT true | Active status |
| metadata | JSONB | | Additional attributes |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

**Warehouse Types:**
- `MAIN` - Main distribution center
- `REGIONAL` - Regional warehouse
- `STORE` - Retail store location
- `REPAIR` - Repair center
- `RETURNS` - Returns processing

**Indexes:**
- `idx_warehouses_code` ON (code)
- `idx_warehouses_location` ON (country, city)
- `idx_warehouses_active` ON (is_active)

### suppliers

Supplier/vendor information.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY | Unique identifier |
| code | VARCHAR(20) | UNIQUE, NOT NULL | Supplier code |
| name | VARCHAR(200) | NOT NULL | Company name |
| contact_name | VARCHAR(100) | | Primary contact |
| email | VARCHAR(100) | | Email address |
| phone | VARCHAR(50) | | Phone number |
| website | VARCHAR(200) | | Website URL |
| address_line1 | VARCHAR(200) | | Street address |
| address_line2 | VARCHAR(200) | | Additional address |
| city | VARCHAR(100) | | City |
| state_province | VARCHAR(100) | | State/Province |
| postal_code | VARCHAR(20) | | Postal code |
| country | VARCHAR(2) | | ISO country code |
| tax_id | VARCHAR(50) | | Tax identification |
| payment_terms | VARCHAR(50) | | Payment terms |
| currency | VARCHAR(3) | DEFAULT 'EUR' | Currency code |
| credit_limit | DECIMAL(12,2) | | Credit limit |
| rating | INTEGER | | Supplier rating (1-5) |
| is_active | BOOLEAN | DEFAULT true | Active status |
| metadata | JSONB | | Additional attributes |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

**Indexes:**
- `idx_suppliers_code` ON (code)
- `idx_suppliers_name` ON (name)
- `idx_suppliers_active` ON (is_active)

### purchase_orders

Purchase order header information.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY | Unique identifier |
| order_number | VARCHAR(50) | UNIQUE, NOT NULL | PO number |
| supplier_id | UUID | FOREIGN KEY | Reference to suppliers |
| warehouse_id | UUID | FOREIGN KEY | Delivery warehouse |
| status | VARCHAR(50) | NOT NULL | Order status |
| order_date | DATE | NOT NULL | Order date |
| expected_date | DATE | | Expected delivery |
| received_date | DATE | | Actual receipt date |
| subtotal | DECIMAL(12,2) | NOT NULL | Subtotal amount |
| tax_amount | DECIMAL(12,2) | DEFAULT 0 | Tax amount |
| shipping_cost | DECIMAL(12,2) | DEFAULT 0 | Shipping cost |
| total_amount | DECIMAL(12,2) | NOT NULL | Total amount |
| currency | VARCHAR(3) | DEFAULT 'EUR' | Currency code |
| payment_terms | VARCHAR(50) | | Payment terms |
| notes | TEXT | | Order notes |
| created_by | UUID | NOT NULL | User who created |
| approved_by | UUID | | User who approved |
| received_by | UUID | | User who received |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

**Order Status:**
- `DRAFT` - Being created
- `PENDING_APPROVAL` - Awaiting approval
- `APPROVED` - Approved
- `SENT` - Sent to supplier
- `PARTIAL` - Partially received
- `RECEIVED` - Fully received
- `CANCELLED` - Cancelled

**Indexes:**
- `idx_purchase_orders_number` ON (order_number)
- `idx_purchase_orders_supplier` ON (supplier_id)
- `idx_purchase_orders_status` ON (status)
- `idx_purchase_orders_dates` ON (order_date, expected_date)

### purchase_order_items

Purchase order line items.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | UUID | PRIMARY KEY | Unique identifier |
| order_id | UUID | FOREIGN KEY | Reference to purchase_orders |
| item_id | UUID | FOREIGN KEY | Reference to inventory_items |
| line_number | INTEGER | NOT NULL | Line number |
| quantity_ordered | INTEGER | NOT NULL | Ordered quantity |
| quantity_received | INTEGER | DEFAULT 0 | Received quantity |
| unit_cost | DECIMAL(10,2) | NOT NULL | Cost per unit |
| tax_rate | DECIMAL(5,2) | DEFAULT 0 | Tax percentage |
| discount_percent | DECIMAL(5,2) | DEFAULT 0 | Discount percentage |
| line_total | DECIMAL(12,2) | NOT NULL | Line total amount |
| notes | TEXT | | Line notes |
| created_at | TIMESTAMP | NOT NULL | Creation timestamp |
| updated_at | TIMESTAMP | NOT NULL | Last update timestamp |

**Indexes:**
- `idx_po_items_order` ON (order_id)
- `idx_po_items_item` ON (item_id)
- `idx_po_items_order_line` UNIQUE ON (order_id, line_number)

## Views

### v_current_inventory

Consolidated view of current inventory across all locations.

```sql
CREATE VIEW v_current_inventory AS
SELECT 
    i.id,
    i.sku,
    i.name,
    i.category,
    SUM(s.quantity_on_hand) as total_on_hand,
    SUM(s.quantity_reserved) as total_reserved,
    SUM(s.quantity_on_hand - s.quantity_reserved) as total_available,
    SUM(s.quantity_in_transit) as total_in_transit,
    i.min_stock_level,
    i.reorder_point,
    CASE 
        WHEN SUM(s.quantity_on_hand) <= i.min_stock_level THEN 'CRITICAL'
        WHEN SUM(s.quantity_on_hand) <= i.reorder_point THEN 'LOW'
        ELSE 'OK'
    END as stock_status
FROM inventory_items i
LEFT JOIN stock_levels s ON i.id = s.item_id
WHERE i.is_active = true
GROUP BY i.id;
```

### v_pending_orders

View of pending purchase orders with supplier details.

```sql
CREATE VIEW v_pending_orders AS
SELECT 
    po.order_number,
    s.name as supplier_name,
    po.order_date,
    po.expected_date,
    po.total_amount,
    po.status,
    COUNT(poi.id) as line_count,
    SUM(poi.quantity_ordered) as total_items
FROM purchase_orders po
JOIN suppliers s ON po.supplier_id = s.id
LEFT JOIN purchase_order_items poi ON po.id = poi.order_id
WHERE po.status IN ('APPROVED', 'SENT', 'PARTIAL')
GROUP BY po.id, s.name;
```

## Triggers

### update_stock_levels

Automatically updates stock levels when movements are recorded.

```sql
CREATE OR REPLACE FUNCTION update_stock_levels()
RETURNS TRIGGER AS $$
BEGIN
    -- Update source warehouse
    IF NEW.from_warehouse_id IS NOT NULL THEN
        UPDATE stock_levels 
        SET quantity_on_hand = quantity_on_hand - NEW.quantity,
            last_movement_at = NEW.movement_date,
            updated_at = NOW()
        WHERE item_id = NEW.item_id 
        AND warehouse_id = NEW.from_warehouse_id;
    END IF;
    
    -- Update destination warehouse
    IF NEW.to_warehouse_id IS NOT NULL THEN
        INSERT INTO stock_levels (item_id, warehouse_id, quantity_on_hand, last_movement_at)
        VALUES (NEW.item_id, NEW.to_warehouse_id, NEW.quantity, NEW.movement_date)
        ON CONFLICT (item_id, warehouse_id)
        DO UPDATE SET 
            quantity_on_hand = stock_levels.quantity_on_hand + NEW.quantity,
            last_movement_at = NEW.movement_date,
            updated_at = NOW();
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_stock_levels
AFTER INSERT ON stock_movements
FOR EACH ROW
EXECUTE FUNCTION update_stock_levels();
```

## Indexes Strategy

- Primary keys use UUID for global uniqueness
- Foreign keys indexed for join performance
- Composite indexes for common query patterns
- Partial indexes for filtered queries
- JSONB GIN indexes for metadata searches

## Data Retention

- `stock_movements`: 7 years (legal requirement)
- `purchase_orders`: 7 years (financial records)
- `inventory_items`: Soft delete (is_active flag)
- Audit logs: 10 years (compliance)

## Performance Considerations

- Table partitioning for `stock_movements` by year
- Materialized views for reporting dashboards
- Read replicas for analytical queries
- Connection pooling with PgBouncer
- Query optimization with EXPLAIN ANALYZE
