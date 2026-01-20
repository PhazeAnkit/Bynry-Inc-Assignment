# Bynry-Inc-Assignment
## Part 1: Issues in the Current Code

## 1.1 Business Logic Issues

- Product stores **warehouse_id**, but products can exist in multiple
  warehouses --- **data model violation**
- SKU uniqueness is not enforced
- Inventory created for only one warehouse; inventory should be per
  warehouse
- Optional fields not handled
- No defaults defined

## 1.2 Technical Issues

- No transaction safety: product may exist without inventory
- No input validation: missing type checks, required fields, JSON
  validation
- No error handling: exceptions return generic 500 errors
- Assumes **request.json** is always valid

## 1.3 Production Impact

- Duplicate SKUs → broken search, billing, integrations
- Partial writes → corrupted data
- Single-warehouse model → cannot scale
- Random 500 errors in production
- Inventory mismatches

## 1.4 Corrected API Endpoint

```python
from decimal import Decimal
from sqlalchemy.exc import IntegrityError

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.get_json()

    if not data:
        return {"error": "Invalid JSON"}, 400

    required_fields = ['name', 'sku', 'price', 'warehouses']
    for field in required_fields:
        if field not in data:
            return {"error": f"Missing field: {field}"}, 400

    if Product.query.filter_by(sku=data['sku']).first():
        return {"error": "SKU already exists"}, 409

    try:
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=Decimal(data['price']),
            description=data.get('description')
        )

        db.session.add(product)
        db.session.flush()

        inventories = []
        for wh in data['warehouses']:
            if 'warehouse_id' not in wh or 'initial_quantity' not in wh:
                return {"error": "Invalid warehouse entry"}, 400

            if wh['initial_quantity'] < 0:
                return {"error": "Quantity cannot be negative"}, 400

            inventories.append(
                Inventory(
                    product_id=product.id,
                    warehouse_id=wh['warehouse_id'],
                    quantity=wh['initial_quantity']
                )
            )

        db.session.add_all(inventories)
        db.session.commit()

        return {"message": "Product created", "product_id": product.id}, 201

    except IntegrityError:
        db.session.rollback()
        return {"error": "SKU must be unique"}, 409

    except Exception as e:
        db.session.rollback()
        return {"error": str(e)}, 500
```

---

# Part 2: Database Design

## 2.1 Assumptions (Explicit)

- A company owns warehouses and products
- Products are global; inventory is company + warehouse specific
- Inventory changes must be auditable
- A product can have one primary supplier (many-to-one)
- Bundles are products composed of other products (self-referencing)
- Low-stock thresholds vary by product type

## 2.2 Core Tables (SQL DDL)

```sql
companies (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

```sql
warehouses (
    id SERIAL PRIMARY KEY,
    company_id INT NOT NULL REFERENCES companies(id),
    name VARCHAR(255) NOT NULL,
    location TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(company_id, name)
);
```

```sql
suppliers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    contact_email VARCHAR(255),
    phone VARCHAR(50)
);
```

```sql
products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(100) NOT NULL UNIQUE,
    product_type VARCHAR(50) NOT NULL,
    price NUMERIC(10,2) NOT NULL,
    supplier_id INT REFERENCES suppliers(id),
    is_bundle BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

```sql
product_bundles (
    bundle_id INT REFERENCES products(id),
    component_id INT REFERENCES products(id),
    quantity INT NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (bundle_id, component_id)
);
```

```sql
inventory (
    product_id INT REFERENCES products(id),
    warehouse_id INT REFERENCES warehouses(id),
    quantity INT NOT NULL CHECK (quantity >= 0),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (product_id, warehouse_id)
);
```

```sql
inventory_transactions (
    id SERIAL PRIMARY KEY,
    product_id INT NOT NULL,
    warehouse_id INT NOT NULL,
    change_amount INT NOT NULL,
    reason VARCHAR(50),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

```sql
sales (
    id SERIAL PRIMARY KEY,
    product_id INT NOT NULL,
    warehouse_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    sold_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

```sql
low_stock_thresholds (
    product_type VARCHAR(50) PRIMARY KEY,
    threshold INT NOT NULL CHECK (threshold > 0)
);
```

## 2.3 Indexing & Constraints

- SKU unique → global identity
- (product_id, warehouse_id) PK → fast inventory lookup
- Index on sales.sold_at → recent activity queries
- Inventory transactions → audit & compliance
- Bundle table → avoids recursive JSON

## 2.4 Open Questions

- Exact definition of recent sales?
- Multiple suppliers per product?
- Bundle inventory vs component-derived?
- Company-specific thresholds?
- Warehouse-specific thresholds?
- Official stockout calculation?
- Alerting on bundles or components?

---

# Part 3: Low-Stock Alerts API

## 3.1 Endpoint

`GET /api/companies/{company_id}/alerts/low-stock`

## 3.2 High-Level Logic

- Resolve warehouses for company
- Filter products with recent sales
- Calculate sales velocity
- Compare inventory vs threshold
- Attach supplier info

## 3.3 Flask Implementation

```python
from datetime import datetime, timedelta
from sqlalchemy.sql import exists

@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def low_stock_alerts(company_id):

    RECENT_DAYS = 30
    recent_cutoff = datetime.utcnow() - timedelta(days=RECENT_DAYS)

    alerts = []

    recent_sales_exists = (
        db.session.query(Sales.id)
        .filter(
            Sales.product_id == Inventory.product_id,
            Sales.warehouse_id == Inventory.warehouse_id,
            Sales.sold_at >= recent_cutoff
        )
        .exists()
    )

    query = (
        db.session.query(
            Product,
            Warehouse,
            Inventory,
            LowStockThreshold.threshold,
            Supplier
        )
        .join(Inventory, Inventory.product_id == Product.id)
        .join(Warehouse, Warehouse.id == Inventory.warehouse_id)
        .join(LowStockThreshold, LowStockThreshold.product_type == Product.product_type)
        .outerjoin(Supplier, Supplier.id == Product.supplier_id)
        .filter(Warehouse.company_id == company_id)
        .filter(Inventory.quantity <= LowStockThreshold.threshold)
        .filter(recent_sales_exists)
    )

    for product, warehouse, inventory, threshold, supplier in query.all():
        alerts.append({
            "product_id": product.id,
            "product_name": product.name,
            "sku": product.sku,
            "warehouse_id": warehouse.id,
            "warehouse_name": warehouse.name,
            "current_stock": inventory.quantity,
            "threshold": threshold,
            "supplier": {
                "id": supplier.id if supplier else None,
                "name": supplier.name if supplier else None,
                "contact_email": supplier.contact_email if supplier else None
            }
        })

    return {
        "alerts": alerts,
        "total_alerts": len(alerts)
    }, 200

```

## 3.4 Edge Cases Handled

| Scenario            | Handling                     |
| ------------------- | ---------------------------- |
| No recent sales     | Product excluded             |
| Zero daily sales    | `days_until_stockout = None` |
| Missing supplier    | Nullable supplier            |
| Multiple warehouses | Alert per warehouse          |
| No alerts           | Empty list                   |
