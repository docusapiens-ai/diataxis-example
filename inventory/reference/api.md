# Inventory API Reference

## OpenAPI Specification

```yaml
openapi: 3.0.3
info:
  title: MyComputer Inventory Management API
  description: RESTful API for managing computer parts inventory across European locations
  version: 2.0.0
  contact:
    name: Inventory Team
    email: inventory-team@mycomputer.com
  license:
    name: Proprietary
    
servers:
  - url: https://api.mycomputer.com/inventory
    description: Production server
  - url: https://staging-api.mycomputer.com/inventory
    description: Staging server
  - url: http://localhost:3001
    description: Development server

security:
  - bearerAuth: []

tags:
  - name: Items
    description: Inventory item management
  - name: Stock
    description: Stock level operations
  - name: Movements
    description: Stock movement tracking
  - name: Warehouses
    description: Warehouse management
  - name: Suppliers
    description: Supplier management
  - name: Orders
    description: Purchase order management
  - name: Reports
    description: Inventory reports

paths:
  /api/items:
    get:
      tags: [Items]
      summary: List inventory items
      description: Retrieve a paginated list of inventory items with optional filters
      operationId: listItems
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
        - name: category
          in: query
          schema:
            type: string
          description: Filter by category
        - name: sku
          in: query
          schema:
            type: string
          description: Search by SKU
        - name: name
          in: query
          schema:
            type: string
          description: Search by name
        - name: is_active
          in: query
          schema:
            type: boolean
          description: Filter by active status
        - name: sort
          in: query
          schema:
            type: string
            enum: [name, sku, category, created_at, updated_at]
            default: name
        - name: order
          in: query
          schema:
            type: string
            enum: [asc, desc]
            default: asc
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
                      $ref: '#/components/schemas/InventoryItem'
                  pagination:
                    $ref: '#/components/schemas/Pagination'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '500':
          $ref: '#/components/responses/InternalError'
    
    post:
      tags: [Items]
      summary: Create inventory item
      description: Add a new item to the inventory catalog
      operationId: createItem
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateInventoryItem'
      responses:
        '201':
          description: Item created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/InventoryItem'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '409':
          $ref: '#/components/responses/Conflict'

  /api/items/{itemId}:
    get:
      tags: [Items]
      summary: Get item details
      description: Retrieve detailed information about a specific inventory item
      operationId: getItem
      parameters:
        - $ref: '#/components/parameters/itemId'
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/InventoryItemDetail'
        '404':
          $ref: '#/components/responses/NotFound'
    
    put:
      tags: [Items]
      summary: Update item
      description: Update inventory item information
      operationId: updateItem
      parameters:
        - $ref: '#/components/parameters/itemId'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateInventoryItem'
      responses:
        '200':
          description: Item updated successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/InventoryItem'
        '400':
          $ref: '#/components/responses/BadRequest'
        '404':
          $ref: '#/components/responses/NotFound'
    
    delete:
      tags: [Items]
      summary: Delete item
      description: Soft delete an inventory item
      operationId: deleteItem
      parameters:
        - $ref: '#/components/parameters/itemId'
      responses:
        '204':
          description: Item deleted successfully
        '404':
          $ref: '#/components/responses/NotFound'
        '409':
          description: Cannot delete item with existing stock

  /api/stock/levels:
    get:
      tags: [Stock]
      summary: Get stock levels
      description: Retrieve current stock levels across warehouses
      operationId: getStockLevels
      parameters:
        - name: item_id
          in: query
          schema:
            type: string
            format: uuid
        - name: warehouse_id
          in: query
          schema:
            type: string
            format: uuid
        - name: low_stock_only
          in: query
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
                      $ref: '#/components/schemas/StockLevel'

  /api/stock/receive:
    post:
      tags: [Stock]
      summary: Receive stock
      description: Record receipt of stock from supplier
      operationId: receiveStock
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/StockReceipt'
      responses:
        '201':
          description: Stock received successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/StockMovement'

  /api/stock/transfer:
    post:
      tags: [Stock]
      summary: Transfer stock
      description: Transfer stock between warehouses
      operationId: transferStock
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/StockTransfer'
      responses:
        '201':
          description: Stock transferred successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/StockMovement'
        '400':
          description: Insufficient stock or invalid transfer

  /api/stock/adjust:
    post:
      tags: [Stock]
      summary: Adjust stock
      description: Manual stock adjustment for corrections
      operationId: adjustStock
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/StockAdjustment'
      responses:
        '201':
          description: Stock adjusted successfully

  /api/movements:
    get:
      tags: [Movements]
      summary: List stock movements
      description: Retrieve history of stock movements
      operationId: listMovements
      parameters:
        - name: item_id
          in: query
          schema:
            type: string
            format: uuid
        - name: warehouse_id
          in: query
          schema:
            type: string
            format: uuid
        - name: movement_type
          in: query
          schema:
            type: string
            enum: [RECEIPT, ISSUE, TRANSFER, ADJUSTMENT, RETURN, DAMAGE, DISPOSAL]
        - name: date_from
          in: query
          schema:
            type: string
            format: date
        - name: date_to
          in: query
          schema:
            type: string
            format: date
        - $ref: '#/components/parameters/pagination'
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
                      $ref: '#/components/schemas/StockMovement'
                  pagination:
                    $ref: '#/components/schemas/Pagination'

  /api/warehouses:
    get:
      tags: [Warehouses]
      summary: List warehouses
      description: Retrieve all warehouse locations
      operationId: listWarehouses
      parameters:
        - name: is_active
          in: query
          schema:
            type: boolean
            default: true
        - name: type
          in: query
          schema:
            type: string
            enum: [MAIN, REGIONAL, STORE, REPAIR, RETURNS]
        - name: country
          in: query
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
                      $ref: '#/components/schemas/Warehouse'

  /api/suppliers:
    get:
      tags: [Suppliers]
      summary: List suppliers
      description: Retrieve supplier information
      operationId: listSuppliers
      parameters:
        - name: is_active
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
                      $ref: '#/components/schemas/Supplier'

  /api/purchase-orders:
    get:
      tags: [Orders]
      summary: List purchase orders
      description: Retrieve purchase orders with filters
      operationId: listPurchaseOrders
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [DRAFT, PENDING_APPROVAL, APPROVED, SENT, PARTIAL, RECEIVED, CANCELLED]
        - name: supplier_id
          in: query
          schema:
            type: string
            format: uuid
        - $ref: '#/components/parameters/pagination'
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
                      $ref: '#/components/schemas/PurchaseOrder'
    
    post:
      tags: [Orders]
      summary: Create purchase order
      description: Create a new purchase order
      operationId: createPurchaseOrder
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreatePurchaseOrder'
      responses:
        '201':
          description: Order created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PurchaseOrder'

  /api/reports/low-stock:
    get:
      tags: [Reports]
      summary: Low stock report
      description: Get items below reorder point
      operationId: getLowStockReport
      parameters:
        - name: warehouse_id
          in: query
          schema:
            type: string
            format: uuid
        - name: include_zero_stock
          in: query
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
                      $ref: '#/components/schemas/LowStockItem'

  /api/reports/valuation:
    get:
      tags: [Reports]
      summary: Inventory valuation
      description: Calculate total inventory value
      operationId: getValuationReport
      parameters:
        - name: warehouse_id
          in: query
          schema:
            type: string
            format: uuid
        - name: as_of_date
          in: query
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
                  total_value:
                    type: number
                    format: decimal
                  total_items:
                    type: integer
                  total_quantity:
                    type: integer
                  by_category:
                    type: array
                    items:
                      type: object
                      properties:
                        category:
                          type: string
                        value:
                          type: number
                        quantity:
                          type: integer

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  parameters:
    itemId:
      name: itemId
      in: path
      required: true
      schema:
        type: string
        format: uuid
      description: Inventory item ID
    
    pagination:
      name: page
      in: query
      schema:
        type: integer
        default: 1
        minimum: 1

  schemas:
    InventoryItem:
      type: object
      properties:
        id:
          type: string
          format: uuid
        sku:
          type: string
        name:
          type: string
        description:
          type: string
        category:
          type: string
        subcategory:
          type: string
        brand:
          type: string
        model:
          type: string
        unit_cost:
          type: number
          format: decimal
        selling_price:
          type: number
          format: decimal
        barcode:
          type: string
        min_stock_level:
          type: integer
        reorder_point:
          type: integer
        reorder_quantity:
          type: integer
        is_active:
          type: boolean
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time

    InventoryItemDetail:
      allOf:
        - $ref: '#/components/schemas/InventoryItem'
        - type: object
          properties:
            stock_levels:
              type: array
              items:
                $ref: '#/components/schemas/StockLevel'
            recent_movements:
              type: array
              items:
                $ref: '#/components/schemas/StockMovement'
            suppliers:
              type: array
              items:
                $ref: '#/components/schemas/Supplier'

    CreateInventoryItem:
      type: object
      required:
        - sku
        - name
        - category
        - unit_cost
        - selling_price
      properties:
        sku:
          type: string
        name:
          type: string
        description:
          type: string
        category:
          type: string
        subcategory:
          type: string
        brand:
          type: string
        model:
          type: string
        unit_cost:
          type: number
          format: decimal
        selling_price:
          type: number
          format: decimal
        weight_kg:
          type: number
        dimensions_cm:
          type: object
          properties:
            length:
              type: number
            width:
              type: number
            height:
              type: number
        barcode:
          type: string
        min_stock_level:
          type: integer
        reorder_point:
          type: integer
        reorder_quantity:
          type: integer
        lead_time_days:
          type: integer
        is_serialized:
          type: boolean
        warranty_months:
          type: integer

    UpdateInventoryItem:
      type: object
      properties:
        name:
          type: string
        description:
          type: string
        category:
          type: string
        subcategory:
          type: string
        unit_cost:
          type: number
        selling_price:
          type: number
        min_stock_level:
          type: integer
        reorder_point:
          type: integer
        reorder_quantity:
          type: integer
        is_active:
          type: boolean

    StockLevel:
      type: object
      properties:
        id:
          type: string
          format: uuid
        item_id:
          type: string
          format: uuid
        item_name:
          type: string
        item_sku:
          type: string
        warehouse_id:
          type: string
          format: uuid
        warehouse_name:
          type: string
        quantity_on_hand:
          type: integer
        quantity_reserved:
          type: integer
        quantity_available:
          type: integer
        quantity_in_transit:
          type: integer
        bin_location:
          type: string
        last_counted_at:
          type: string
          format: date-time
        last_movement_at:
          type: string
          format: date-time

    StockMovement:
      type: object
      properties:
        id:
          type: string
          format: uuid
        movement_type:
          type: string
          enum: [RECEIPT, ISSUE, TRANSFER, ADJUSTMENT, RETURN, DAMAGE, DISPOSAL, FOUND]
        item_id:
          type: string
          format: uuid
        item_name:
          type: string
        from_warehouse_id:
          type: string
          format: uuid
        from_warehouse_name:
          type: string
        to_warehouse_id:
          type: string
          format: uuid
        to_warehouse_name:
          type: string
        quantity:
          type: integer
        unit_cost:
          type: number
        reference_type:
          type: string
        reference_id:
          type: string
        reason:
          type: string
        notes:
          type: string
        performed_by:
          type: string
        movement_date:
          type: string
          format: date-time

    StockReceipt:
      type: object
      required:
        - item_id
        - warehouse_id
        - quantity
      properties:
        item_id:
          type: string
          format: uuid
        warehouse_id:
          type: string
          format: uuid
        quantity:
          type: integer
          minimum: 1
        unit_cost:
          type: number
        purchase_order_id:
          type: string
          format: uuid
        supplier_id:
          type: string
          format: uuid
        notes:
          type: string

    StockTransfer:
      type: object
      required:
        - item_id
        - from_warehouse_id
        - to_warehouse_id
        - quantity
      properties:
        item_id:
          type: string
          format: uuid
        from_warehouse_id:
          type: string
          format: uuid
        to_warehouse_id:
          type: string
          format: uuid
        quantity:
          type: integer
          minimum: 1
        reason:
          type: string
        notes:
          type: string

    StockAdjustment:
      type: object
      required:
        - item_id
        - warehouse_id
        - quantity
        - adjustment_type
        - reason
      properties:
        item_id:
          type: string
          format: uuid
        warehouse_id:
          type: string
          format: uuid
        quantity:
          type: integer
        adjustment_type:
          type: string
          enum: [INCREASE, DECREASE, SET]
        reason:
          type: string
        notes:
          type: string

    Warehouse:
      type: object
      properties:
        id:
          type: string
          format: uuid
        code:
          type: string
        name:
          type: string
        type:
          type: string
          enum: [MAIN, REGIONAL, STORE, REPAIR, RETURNS]
        address:
          type: object
          properties:
            line1:
              type: string
            line2:
              type: string
            city:
              type: string
            state_province:
              type: string
            postal_code:
              type: string
            country:
              type: string
        coordinates:
          type: object
          properties:
            latitude:
              type: number
            longitude:
              type: number
        contact:
          type: object
          properties:
            phone:
              type: string
            email:
              type: string
        is_active:
          type: boolean

    Supplier:
      type: object
      properties:
        id:
          type: string
          format: uuid
        code:
          type: string
        name:
          type: string
        contact_name:
          type: string
        email:
          type: string
        phone:
          type: string
        website:
          type: string
        payment_terms:
          type: string
        currency:
          type: string
        rating:
          type: integer
          minimum: 1
          maximum: 5
        is_active:
          type: boolean

    PurchaseOrder:
      type: object
      properties:
        id:
          type: string
          format: uuid
        order_number:
          type: string
        supplier:
          $ref: '#/components/schemas/Supplier'
        warehouse:
          $ref: '#/components/schemas/Warehouse'
        status:
          type: string
          enum: [DRAFT, PENDING_APPROVAL, APPROVED, SENT, PARTIAL, RECEIVED, CANCELLED]
        order_date:
          type: string
          format: date
        expected_date:
          type: string
          format: date
        received_date:
          type: string
          format: date
        items:
          type: array
          items:
            $ref: '#/components/schemas/PurchaseOrderItem'
        subtotal:
          type: number
        tax_amount:
          type: number
        shipping_cost:
          type: number
        total_amount:
          type: number
        currency:
          type: string

    CreatePurchaseOrder:
      type: object
      required:
        - supplier_id
        - warehouse_id
        - items
      properties:
        supplier_id:
          type: string
          format: uuid
        warehouse_id:
          type: string
          format: uuid
        expected_date:
          type: string
          format: date
        items:
          type: array
          items:
            type: object
            required:
              - item_id
              - quantity
              - unit_cost
            properties:
              item_id:
                type: string
                format: uuid
              quantity:
                type: integer
              unit_cost:
                type: number
        notes:
          type: string

    PurchaseOrderItem:
      type: object
      properties:
        item_id:
          type: string
          format: uuid
        item_name:
          type: string
        item_sku:
          type: string
        quantity_ordered:
          type: integer
        quantity_received:
          type: integer
        unit_cost:
          type: number
        tax_rate:
          type: number
        discount_percent:
          type: number
        line_total:
          type: number

    LowStockItem:
      type: object
      properties:
        item_id:
          type: string
          format: uuid
        sku:
          type: string
        name:
          type: string
        current_stock:
          type: integer
        reorder_point:
          type: integer
        reorder_quantity:
          type: integer
        days_until_stockout:
          type: integer
        suggested_order_quantity:
          type: integer

    Pagination:
      type: object
      properties:
        page:
          type: integer
        limit:
          type: integer
        total:
          type: integer
        total_pages:
          type: integer

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
              type: object

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
    
    InternalError:
      description: Internal server error
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

## Authentication

All API endpoints require JWT authentication. Include the token in the Authorization header:

```
Authorization: Bearer <your-jwt-token>
```

## Rate Limiting

- 1000 requests per minute per API key
- 10000 requests per hour per API key
- Headers returned: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

## Error Handling

Standard HTTP status codes are used. Error responses include:
- `code`: Machine-readable error code
- `message`: Human-readable error message
- `details`: Additional error context

## Versioning

API version is included in the URL path. Breaking changes require a new version.

## Webhooks

Configure webhooks for real-time events:
- `inventory.item.created`
- `inventory.item.updated`
- `inventory.stock.low`
- `inventory.order.received`
