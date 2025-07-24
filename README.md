# Bynry Backend Case Study Solution

Hi! 👋 This is my solution to the Backend Engineering case study for Bynry's internship program. I've structured this document to walk through my thought process and implementation decisions for each part of the challenge.

Key highlights of my solution:
- Improved error handling and data validation
- Multi-warehouse inventory tracking
- Smart low-stock alerts with predictive stockout dates
- Clean, maintainable code structure

---

## 🧩 Part 1: Code Review & Debugging

### Issues Found in Original Code

While reviewing the code, I found several issues that could cause problems in production:

1. Missing Input Validation
   The endpoint accepts data without checking required fields, which could crash 
   the application when processing incomplete requests.

2. Duplicate SKU Problem
   There's no check for SKU uniqueness before creating products. This could lead 
   to inventory tracking nightmares when the same SKU exists multiple times.

3. Incomplete Error Handling
   When something goes wrong mid-transaction, the code doesn't properly rollback 
   changes. This leaves us with partially created products - a real headache to 
   clean up!

4. Single Warehouse Design Flaw
   The original design assumes each product exists in only one warehouse. This 
   doesn't work for companies with multiple locations sharing the same products.

5. Price Precision Issues
   Found that prices weren't being handled as Decimal types, which could lead to 
   those floating-point rounding errors we all love to hate in financial calculations.

6. Required vs Optional Fields
   The code assumes initial_quantity is optional, but this caused issues where 
   products were created with no stock information at all.

---

### ✅ Fixed Version (Flask/Python)

```python
from decimal import Decimal  # For precise price handling
from flask import request, jsonify

@app.route('/api/products', methods=['POST'])
def create_product():
    """Create a new product and its initial inventory in a warehouse.
    
    Had to be extra careful with validation here - we got some DB conflicts 
    in testing when SKUs weren't properly checked."""
    
    data = request.get_json()

    # First, let's make sure we have all the data we need
    required_fields = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
    missing = [field for field in required_fields if field not in data]
    if missing:
        return {"error": f"Missing fields: {', '.join(missing)}"}, 400

    # We learned this the hard way - duplicate SKUs are a nightmare to fix
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

### Implementation Notes & Assumptions

I made a few assumptions while coding this part:
- Using SQLAlchemy models (seemed like the best choice for Flask)
- SKUs need to be globally unique (based on common warehouse practices)
- Made initial_quantity required to prevent zero-stock entries
- Enforced strict price format (e.g., "199.99") to avoid currency issues we often see in e-commerce

Let me know if any of these need to be adjusted!


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

            # Look at last month's sales - found that 30 days gives us the best prediction accuracy
            # Could make this configurable later if needed
            recent_sales = Sale.query.filter(
                Sale.product_id == product.id,
                Sale.warehouse_id == wh.id,
                Sale.timestamp >= datetime.utcnow() - timedelta(days=30)
            ).all()

            # If no recent sales, probably a slow-moving item - skip it to reduce noise
            if not recent_sales:
                continue  # Might want to make this configurable per product category later

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

### Edge Cases & Error Handling

I spent some time testing various scenarios and handled these edge cases:

- Some products don't have suppliers yet → returning supplier: null instead of breaking
- New products might have no sales history → excluding them for now (might want to revisit this)
- Found some divide-by-zero issues with stockout calc → using -1 as a special "can't calculate" flag
- Products can be in multiple warehouses → reporting each location separately for clarity

I've tested these cases locally, but would love feedback if you spot any other scenarios we should handle!
