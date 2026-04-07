**Candidate:** Omkar Gaikwad
**Date:** April 8, 2026
**Subject:** B2B Inventory Management Platform Backend Design & Debugging

---

## Part 1: Code Review & Debugging

### Identified Issues & Production Impact

| Issue | Category | Production Impact |
| :--- | :--- | :--- |
| **Lack of Atomicity** | Technical | Using two `db.session.commit()` calls means if the second insert fails, you're left with a "ghost" product that has no stock records, causing data inconsistency. |
| **Missing Input Validation** | Security/UX | Direct access to `request.json` without checks will 500 (Internal Server Error) if fields are missing, providing a poor experience and potential crash vectors. |
| **Schema/Logic Mismatch** | Business Logic | The `Product` model contains `warehouse_id`. This prevents a single product from existing in multiple warehouses simultaneously, violating the core requirement. |
| **SKU Uniqueness Logic** | Integrity | No pre-check for existing SKUs. While DB constraints might catch this, the API fails to provide a meaningful `409 Conflict` error to the user. |
| **Price Precision** | Financial | Handling prices as raw floats/strings instead of `Decimal` can lead to rounding errors in high-volume financial transactions. |
| **Missing Edge Case Handling** | Validation | Negative `initial_quantity` or `price` is not checked. This allows impossible data entry into the system. |

### Corrected Implementation (Flask/SQLAlchemy)

```python
from flask import request, jsonify
from decimal import Decimal, InvalidOperation
from sqlalchemy.exc import IntegrityError

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json or {}
    
    # 1. Comprehensive Validation
    required = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
    if not all(k in data for k in required):
        return jsonify({"error": "Missing required fields"}), 400

    try:
        price = Decimal(str(data['price']))
        qty = int(data['initial_quantity'])
        if price < 0 or qty < 0:
            return jsonify({"error": "Values must be non-negative"}), 400
    except (InvalidOperation, ValueError):
        return jsonify({"error": "Invalid numeric format"}), 400

    # 2. Check for Warehouse and SKU existence
    if Product.query.filter_by(sku=data['sku']).first():
        return jsonify({"error": "SKU already exists"}), 409
    
    if not Warehouse.query.get(data['warehouse_id']):
        return jsonify({"error": "Warehouse not found"}), 404

    try:
        # 3. Single-Transaction Atomic Operation
        new_product = Product(
            name=data['name'],
            sku=data['sku'],
            price=price
        )
        db.session.add(new_product)
        db.session.flush() # Populate new_product.id

        inventory = Inventory(
            product_id=new_product.id,
            warehouse_id=data['warehouse_id'],
            quantity=qty
        )
        db.session.add(inventory)
        
        # Add stock movement for audit trail
        db.session.add(StockMovement(
            product_id=new_product.id,
            warehouse_id=data['warehouse_id'],
            change=qty,
            reason="Initial Setup"
        ))

        db.session.commit()
        return jsonify({"message": "Created", "id": new_product.id}), 201

    except Exception as e:
        db.session.rollback()
        return jsonify({"error": "Internal database error"}), 500
```

---

## Part 2: Database Design

### Recommended Schema (PostgreSQL DDL)

```sql
-- Core Tenancy
CREATE TABLE companies (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

-- Locations
CREATE TABLE warehouses (
    id SERIAL PRIMARY KEY,
    company_id INT REFERENCES companies(id),
    name VARCHAR(100) NOT NULL,
    UNIQUE(company_id, name)
);

-- Categories with Default Thresholds
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    default_threshold INT DEFAULT 10
);

-- Products (Multi-warehouse capable)
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    company_id INT REFERENCES companies(id),
    category_id INT REFERENCES categories(id),
    sku VARCHAR(100) NOT NULL,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(15, 2) NOT NULL,
    custom_threshold INT, -- Nullable, overrides category default
    is_bundle BOOLEAN DEFAULT FALSE,
    UNIQUE(company_id, sku)
);

-- Junction: Inventory Levels
CREATE TABLE inventory (
    product_id INT REFERENCES products(id),
    warehouse_id INT REFERENCES warehouses(id),
    quantity INT NOT NULL DEFAULT 0,
    PRIMARY KEY (product_id, warehouse_id)
);

-- Bundles: Recursive Components
CREATE TABLE bundle_components (
    bundle_product_id INT REFERENCES products(id),
    component_product_id INT REFERENCES products(id),
    quantity_needed INT NOT NULL,
    PRIMARY KEY (bundle_product_id, component_product_id)
);

-- External Relationships
CREATE TABLE suppliers (
    id SERIAL PRIMARY KEY,
    company_id INT REFERENCES companies(id),
    name VARCHAR(255) NOT NULL,
    contact_email VARCHAR(255)
);

-- Audit History
CREATE TABLE stock_movements (
    id SERIAL PRIMARY KEY,
    product_id INT REFERENCES products(id),
    warehouse_id INT REFERENCES warehouses(id),
    quantity_delta INT NOT NULL,
    movement_type VARCHAR(20), -- 'SALE', 'RESTOCK', 'ADJUSTMENT'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Identification of Gaps
1. **Unit of Measure (UOM):** Can we have partial stock (e.g., 0.5 kg)? Does a product have a base unit and a display unit?
2. **SKU Scope:** Is SKU uniqueness cross-tenant or per-company? My design assumes per-company.
3. **Currency Support:** For international SaaS, we need a `currency_code` for products and purchase orders.
4. **Bundle Behavior:** Does assembly happen at point-of-sale (decrementing components) or is it a separate "Manufacturing" step that creates new stock?

---

## Part 3: API Implementation

### Implementation: Low-Stock Alerts

```python
from datetime import datetime, timedelta
from flask import jsonify
from sqlalchemy import func

@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def get_low_stock_alerts(company_id):
    # Requirement: Recent sales activity (Past 30 days)
    lookback_window = datetime.utcnow() - timedelta(days=30)

    # 1. Join logic to find products below effective threshold
    # Effective threshold = Product.custom_threshold OR Category.default_threshold
    query = db.session.query(
        Product,
        Inventory.quantity.label('stock'),
        Warehouse.name.label('wh_name'),
        Warehouse.id.label('wh_id'),
        Supplier,
        func.coalesce(Product.custom_threshold, Category.default_threshold).label('threshold')
    ).join(Inventory, Product.id == Inventory.product_id)\
     .join(Warehouse, Inventory.warehouse_id == Warehouse.id)\
     .join(Category, Product.category_id == Category.id)\
     .outerjoin(Supplier, ...) # Logic for primary supplier\
     .filter(Product.company_id == company_id)\
     .filter(Inventory.quantity < func.coalesce(Product.custom_threshold, Category.default_threshold))

    # 2. Filter for recent sales velocity
    recent_activity = db.session.query(StockMovement.product_id)\
        .filter(StockMovement.movement_type == 'SALE', StockMovement.created_at >= lookback_window)\
        .distinct().subquery()
    
    results = query.filter(Product.id.in_(recent_activity)).all()

    # 3. Formatter and Business Logic
    alerts = []
    for res in results:
        # Helper: Days until stockout = Current Stock / (Total Sales in 30 days / 30)
        daily_velocity = calculate_sales_velocity(res.Product.id, 30)
        days_left = int(res.stock / daily_velocity) if daily_velocity > 0 else 999

        alerts.append({
            "product_id": res.Product.id,
            "product_name": res.Product.name,
            "sku": res.Product.sku,
            "warehouse_id": res.wh_id,
            "warehouse_name": res.wh_name,
            "current_stock": res.stock,
            "threshold": res.threshold,
            "days_until_stockout": days_left,
            "supplier": {
                "id": res.Supplier.id if res.Supplier else None,
                "name": res.Supplier.name if res.Supplier else "N/A"
            }
        })

    return jsonify({"alerts": alerts, "total_alerts": len(alerts)})

def calculate_sales_velocity(pid, days):
    cutoff = datetime.utcnow() - timedelta(days=days)
    total = db.session.query(func.sum(func.abs(StockMovement.quantity_delta)))\
              .filter(pid == pid, movement_type == 'SALE', created_at >= cutoff).scalar() or 0
    return total / days
```

### Approach & Assumptions
- **Heuristic for Stockout:** `days_until_stockout` is a forecast based on a 30-day moving average.
- **Supplier Selection:** Assumed a `primary_supplier` flag in the product relationship.
- **Edge Cases:** Handled items with zero sales velocity (infinite days left) and missing supplier links.
- **Performance:** Used subqueries to filter by activity before joining heavy warehouse data.
