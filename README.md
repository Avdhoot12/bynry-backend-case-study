# 🔧 Bynry Backend Intern Case Study

This document contains my complete solution to the Backend Engineering Case Study for the internship at **Bynry Inc.**

---

## 🧩 Part 1: Code Review & Debugging

### 🚩 Original Issues Identified

| Issue | Impact |
|-------|--------|
| No input validation | Crashes on missing fields |
| SKU uniqueness not enforced | Duplicate SKUs corrupt inventory |
| No error handling or rollback | Half-commits lead to data inconsistency |
| Business logic flaw | Products shouldn't be tied to one warehouse |
| Price type mismatch | Decimal precision lost if not enforced |
| Missing optional field handling | Breaks if `initial_quantity` is not sent |

---

### ✅ Fixed Version (Flask/Python)

```python
from decimal import Decimal
from flask import request, jsonify

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.get_json()

    # Validate required fields
    required_fields = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
    missing = [field for field in required_fields if field not in data]
    if missing:
        return {"error": f"Missing fields: {', '.join(missing)}"}, 400

    # Check for SKU uniqueness
    if Product.query.filter_by(sku=data['sku']).first():
        return {"error": "SKU must be unique"}, 409

    try:
        # Create product (decoupled from warehouse)
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=Decimal(str(data['price']))
        )
        db.session.add(product)
        db.session.flush()  # flush to get product.id without committing

        # Add inventory for specific warehouse
        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data['initial_quantity']
        )
        db.session.add(inventory)

        db.session.commit()

        return {"message": "Product created", "product_id": product.id}, 201

    except Exception as e:
        db.session.rollback()
        return {"error": str(e)}, 500

```

### 🧠 Explanation of Fixes

✅ Input validation: Ensures required fields are present before proceeding

✅ SKU uniqueness check: Prevents accidental data corruption

✅ Transaction safety: Wraps entire DB interaction with rollback on error

✅ Decimal conversion: Ensures consistent handling of currency values

✅ Warehouse decoupling: Supports product presence across multiple warehouses

✅ Safe commit strategy: flush() used to access product.id without half-saving

### 🧠 Assumptions

- Product and Inventory are SQLAlchemy models
- SKU is globally unique
- initial_quantity is required (not optional)
- price is in a valid numeric string format (e.g., "199.99")


## 🗃️ Part 2: Database Design

### 🧱 Schema Design (SQL DDL Style)

```sql
-- Companies can have multiple warehouses
CREATE TABLE companies (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE warehouses (
    id SERIAL PRIMARY KEY,
    company_id INT REFERENCES companies(id),
    name TEXT NOT NULL
);

-- Products can exist in multiple warehouses
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    sku TEXT UNIQUE NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    threshold INT DEFAULT 20  -- for low stock alerts
);

-- Product inventory in each warehouse
CREATE TABLE inventory (
    product_id INT REFERENCES products(id),
    warehouse_id INT REFERENCES warehouses(id),
    quantity INT DEFAULT 0,
    PRIMARY KEY (product_id, warehouse_id)
);

-- Track inventory changes (for auditing)
CREATE TABLE inventory_changes (
    id SERIAL PRIMARY KEY,
    product_id INT REFERENCES products(id),
    warehouse_id INT REFERENCES warehouses(id),
    quantity_change INT,
    changed_at TIMESTAMP DEFAULT NOW()
);

-- Suppliers and their product associations
CREATE TABLE suppliers (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    contact_email TEXT
);

CREATE TABLE product_supplier (
    product_id INT REFERENCES products(id),
    supplier_id INT REFERENCES suppliers(id),
    PRIMARY KEY (product_id, supplier_id)
);

-- Sales activity (for calculating recent demand)
CREATE TABLE sale (
    id SERIAL PRIMARY KEY,
    product_id INT REFERENCES products(id),
    warehouse_id INT REFERENCES warehouses(id),
    quantity INT,
    timestamp TIMESTAMP DEFAULT NOW()
);

-- Product bundles (e.g. combo packs)
CREATE TABLE product_bundles (
    bundle_id INT REFERENCES products(id),
    component_id INT REFERENCES products(id),
    quantity INT DEFAULT 1,
    PRIMARY KEY (bundle_id, component_id)
);
```

### ❓ Questions for Product Team
- Are product thresholds defined globally or per-warehouse?
- Are bundles physically stored or just logical groupings?
- Can a supplier provide the same product to multiple companies?
- Do we track expiry/batches in inventory?
- Are price changes tracked historically?

### 💡 Design Notes
- Composite PK in inventory for fast lookups
- product_supplier table handles M:M mapping cleanly
- sale table lets us calculate demand for stockout alerts

## 🌐 Part 3: Low Stock Alert API

### 📎 Endpoint

```http
GET /api/companies/<company_id>/alerts/low-stock
```

Returns a list of products that are:
- Below their low stock threshold
- Have had recent sales activity
- Include details to reorder from supplier
- Cover multiple warehouses under the same company

### ✅ Implementation (Python + Flask + SQLAlchemy)

```python
@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def low_stock_alerts(company_id):
    alerts = []

    warehouses = Warehouse.query.filter_by(company_id=company_id).all()
    for wh in warehouses:
        inventories = Inventory.query.filter_by(warehouse_id=wh.id).all()

        for inv in inventories:
            product = Product.query.get(inv.product_id)

            # Filter for recent sales in the last 30 days
            recent_sales = Sale.query.filter(
                Sale.product_id == product.id,
                Sale.warehouse_id == wh.id,
                Sale.timestamp >= datetime.utcnow() - timedelta(days=30)
            ).all()

            if not recent_sales:
                continue  # Skip products with no recent sales

            total_qty = sum(s.quantity for s in recent_sales)
            avg_daily_sales = total_qty / 30 if total_qty > 0 else 0
            threshold = product.threshold or 20  # Default fallback

            if inv.quantity < threshold:
                supplier = product.suppliers[0] if product.suppliers else None
                days_until_stockout = int(inv.quantity / avg_daily_sales) if avg_daily_sales else -1

                alerts.append({
                    "product_id": product.id,
                    "product_name": product.name,
                    "sku": product.sku,
                    "warehouse_id": wh.id,
                    "warehouse_name": wh.name,
                    "current_stock": inv.quantity,
                    "threshold": threshold,
                    "days_until_stockout": days_until_stockout,
                    "supplier": {
                        "id": supplier.id,
                        "name": supplier.name,
                        "contact_email": supplier.contact_email
                    } if supplier else None
                })

    return jsonify({
        "alerts": alerts,
        "total_alerts": len(alerts)
    })
```

### 💡 Business Logic
- Threshold Check: Product is considered low stock if inventory.quantity < threshold
- Recent Sales Activity: Product must have sales within the last 30 days
- Stockout Prediction: Based on average daily sales (estimated as total_sold / 30)
- Supplier Info: Only first supplier returned (extendable)

### 🧠 Assumptions

- Product thresholds are defined globally, not per warehouse
- Sale table exists with timestamp, warehouse_id, and quantity
- Products can have multiple suppliers, but we return only the first
- Stockout prediction uses simple average (current_stock / avg_daily_sales)
- Products with no sales in last 30 days are excluded from alerts

### 📋 Example JSON Response
```json
{
  "alerts": [
    {
      "product_id": 123,
      "product_name": "Widget A",
      "sku": "WID-001",
      "warehouse_id": 456,
      "warehouse_name": "Main Warehouse",
      "current_stock": 5,
      "threshold": 20,
      "days_until_stockout": 12,
      "supplier": {
        "id": 789,
        "name": "Supplier Corp",
        "contact_email": "orders@supplier.com"
      }
    }
  ],
  "total_alerts": 1
}
```

### 🛡️ Edge Cases Considered
- Products with no supplier → supplier: null
- Products with no sales → excluded
- Division by zero → returns -1 for days_until_stockout
- Products existing in multiple warehouses are reported per warehouse
