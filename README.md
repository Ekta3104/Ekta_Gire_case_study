# StockFlow — Take-Home Assessment

---

## Part 1: Code Review & Debugging

When I first read through this code, it *looks* fine at a glance — and that's kind of what makes it dangerous. It'll probably work 80% of the time in testing, and then silently break in production in ways that are hard to trace.

Here's what I found:

The double-commit is the biggest problem. The code does db.session.commit() after creating the product, then again after creating the inventory record. These two operations are logically one unit of work — they should either both succeed or both fail. If the second commit fails (say, the warehouse_id doesn't exist or there's a DB hiccup), you're left with a product in the database that has no inventory record. Downstream, anything that queries product stock will either crash or return wrong data. The fix is straightforward: add everything to the session first, then commit once.

No input validation. The code does data['name'], data['sku'], etc. — direct dictionary access. If the client sends a request missing any of these, Python throws a KeyError, Flask returns a 500, and your logs just show an unhandled exception with no useful context. In a real system, you at minimum want to check required fields exist and return a meaningful 400 with a clear message. Using data.get() for optional fields is also good practice.

warehouse_id on the Product model is an architectural issue. The requirements explicitly say a product can exist in multiple warehouses — but if warehouse_id is a column on the products table, that's a direct 1-to-N relationship in the wrong direction. You can't have one product tracked in three warehouses without duplicating the row (which would break SKU uniqueness). This tells me whoever wrote this didn't think through the data model before writing the API. The product-warehouse relationship should live in the inventory table.

SKU uniqueness isn't handled gracefully. If someone tries to create a product with a duplicate SKU, the DB will throw an IntegrityError. Without catching it, that bubbles up as a 500. It should be a 409 with a message like "SKU already exists."

Here's the corrected version:

python
from flask import request, jsonify
from sqlalchemy.exc import IntegrityError

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json or {}

    # Basic validation — keep it simple but explicit
    required_fields = ['name', 'sku', 'price']
    missing = [f for f in required_fields if f not in data]
    if missing:
        return jsonify({"error": f"Missing required fields: {', '.join(missing)}"}), 400

    try:
        price = float(data['price'])
        if price < 0:
            return jsonify({"error": "Price cannot be negative"}), 400
    except (ValueError, TypeError):
        return jsonify({"error": "Price must be a valid number"}), 400

    try:
        # Product has no warehouse_id — that belongs in inventory
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=price,
            description=data.get('description')  # optional
        )
        db.session.add(product)

        # Only create inventory record if warehouse info was provided
        warehouse_id = data.get('warehouse_id')
        initial_quantity = data.get('initial_quantity', 0)

        if warehouse_id is not None:
            inventory = Inventory(
                product=product,  # SQLAlchemy handles the FK via relationship
                warehouse_id=warehouse_id,
                quantity=max(0, int(initial_quantity))  # sanity check on quantity
            )
            db.session.add(inventory)

        # Single commit — both operations are atomic
        db.session.commit()

        return jsonify({"message": "Product created", "product_id": product.id}), 201

    except IntegrityError:
        db.session.rollback()
        return jsonify({"error": "A product with this SKU already exists"}), 409

    except Exception as e:
        db.session.rollback()
        # In production I'd log this properly — app.logger.exception(e)
        return jsonify({"error": "Something went wrong"}), 500


One thing I'd add in a real system: a check that the warehouse_id actually belongs to the company making the request. Otherwise a tenant could attach their product to someone else's warehouse, which is a data isolation bug. That requires auth context I don't have here, but it's worth flagging.

---

## Part 2: Database Design

The requirements give a decent starting point, but there are some obvious gaps. Before designing anything, I'd want to clarify a few things with the product team (more on that below). But based on what's there, here's the schema I'd go with:

sql
-- Core tenant table
CREATE TABLE companies (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE warehouses (
    id          SERIAL PRIMARY KEY,
    company_id  INT NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name        VARCHAR(255) NOT NULL,
    location    VARCHAR(255),
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE suppliers (
    id            SERIAL PRIMARY KEY,
    name          VARCHAR(255) NOT NULL,
    contact_email VARCHAR(255),
    phone         VARCHAR(50),
    created_at    TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Products belong to a company, not a warehouse
CREATE TABLE products (
    id           SERIAL PRIMARY KEY,
    company_id   INT NOT NULL REFERENCES companies(id),
    name         VARCHAR(255) NOT NULL,
    sku          VARCHAR(100) NOT NULL,
    product_type VARCHAR(20) NOT NULL DEFAULT 'standard', -- 'standard' | 'bundle'
    price        NUMERIC(12, 2) NOT NULL,
    created_at   TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(sku)  -- global uniqueness per the requirement
);

-- Links products to their primary supplier
-- Using a join table here in case a product can have multiple suppliers
CREATE TABLE product_suppliers (
    product_id  INT NOT NULL REFERENCES products(id),
    supplier_id INT NOT NULL REFERENCES suppliers(id),
    is_primary  BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (product_id, supplier_id)
);

-- The actual stock per product per warehouse
CREATE TABLE inventory (
    id                  SERIAL PRIMARY KEY,
    product_id          INT NOT NULL REFERENCES products(id),
    warehouse_id        INT NOT NULL REFERENCES warehouses(id),
    quantity            INT NOT NULL DEFAULT 0 CHECK (quantity >= 0),
    low_stock_threshold INT NOT NULL DEFAULT 10,
    updated_at          TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(product_id, warehouse_id)
);

-- Every stock movement gets logged here
CREATE TABLE inventory_transactions (
    id               SERIAL PRIMARY KEY,
    inventory_id     INT NOT NULL REFERENCES inventory(id),
    quantity_change  INT NOT NULL,  -- negative for sales, positive for restocks
    transaction_type VARCHAR(50) NOT NULL,  -- 'sale', 'restock', 'adjustment', 'return'
    reference_id     VARCHAR(100),  -- optional: order ID, PO number, etc.
    created_at       TIMESTAMP NOT NULL DEFAULT NOW()
);

-- For bundles: which products make up this bundle, and how many of each
CREATE TABLE bundle_components (
    bundle_id     INT NOT NULL REFERENCES products(id),
    component_id  INT NOT NULL REFERENCES products(id),
    quantity      INT NOT NULL DEFAULT 1,
    PRIMARY KEY (bundle_id, component_id),
    CHECK (bundle_id != component_id)  -- a product can't bundle itself
);

-- Useful indexes
CREATE INDEX idx_inventory_warehouse ON inventory(warehouse_id);
CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_transactions_inventory ON inventory_transactions(inventory_id);
CREATE INDEX idx_transactions_created ON inventory_transactions(created_at);
CREATE INDEX idx_products_company ON products(company_id);


Why I made these choices:

The inventory_transactions table is the most important part of this design. Just storing the current quantity wouldn't satisfy "track when inventory levels change," and it also makes it impossible to calculate things like sales velocity later. Every change — sale, restock, whatever — gets its own row. The current quantity is derived from summing these, or kept denormalized in inventory.quantity for fast reads (which is the trade-off I'd make here).

I used NUMERIC(12, 2) for price instead of FLOAT because floating point math on money is a well-known footgun. NUMERIC is exact.

The CHECK (quantity >= 0) constraint on inventory is a safety net. In a real system you'd want to handle this in application logic too, but having the DB enforce it means even direct SQL writes can't create negative stock.

Questions I'd ask the product team before finalizing:

The requirements don't say whether SKUs need to be globally unique or just per-company. I went with global uniqueness since the spec says "across the platform," but that's an unusual constraint for multi-tenant SaaS and might cause issues if two companies happen to use the same SKU format.

The bundle behavior is unclear — when a bundle is sold, do we deduct stock from the bundle's own inventory, or from each component separately? These are two very different fulfillment models. I'd also want to know whether bundles can contain other bundles (and how deep that nesting can go).

Can a product have multiple suppliers? I built a join table for it (product_suppliers) which handles both cases, but if it's always one supplier, a simple FK on products is cleaner.

And finally — is there a concept of "reserved" inventory (e.g., items in an order but not yet shipped)? That would significantly change the quantity tracking model.

---

## Part 3: API Implementation

My approach here was to push as much of the filtering logic down to the database as possible rather than pulling everything into Node and filtering in JavaScript. For a company with thousands of products across dozens of warehouses, fetching everything and looping over it is a quick way to make this endpoint slow and memory-hungry.

The trickiest part was the "recent sales activity" rule — I interpreted this as: the product must have had at least one sale transaction in the last 30 days. I handled that with an inner join against a subquery of recent sales. If there are no recent sales for a given inventory record, it simply won't appear in the join result, which is exactly the behavior we want.

I went with raw SQL via node-postgres (pg) instead of an ORM. For a query this complex with subqueries and multiple joins, I find raw SQL is cleaner and easier to reason about than trying to express it through an ORM's query builder. It's also easier to debug when something goes wrong.

Assumptions I made:

"Recent" means the last 30 days. "Sales activity" means there's at least one row in inventory_transactions with transaction_type = 'sale' in that window. Days until stockout is calculated as current_stock / (total_sold_last_30_days / 30). The low-stock threshold lives on the inventory row, so it can vary per product per warehouse.

js
// routes/alerts.js
const express = require('express');
const router = express.Router();
const { pool } = require('../db'); // pg Pool instance

router.get('/api/companies/:companyId/alerts/low-stock', async (req, res) => {
  const { companyId } = req.params;

  // Quick sanity check — make sure the company actually exists
  const companyCheck = await pool.query(
    'SELECT id FROM companies WHERE id = $1',
    [companyId]
  );
  if (companyCheck.rowCount === 0) {
    return res.status(404).json({ error: 'Company not found' });
  }

  try {
    /*
     * The subquery (recent_sales) aggregates total units sold per inventory
     * record over the last 30 days. quantity_change is negative for sales,
     * so we use ABS() to get a positive number.
     *
     * The inner join with recent_sales is intentional — it means only
     * inventory records with actual recent sales will be returned.
     * Products sitting idle won't trigger alerts even if stock is low.
     *
     * We use a LEFT JOIN for suppliers because not every product is
     * guaranteed to have one, and we don't want to silently drop alerts
     * just because supplier data is missing.
     */
    const query = 
      WITH recent_sales AS (
        SELECT
          it.inventory_id,
          ABS(SUM(it.quantity_change)) AS units_sold
        FROM inventory_transactions it
        WHERE
          it.transaction_type = 'sale'
          AND it.created_at >= NOW() - INTERVAL '30 days'
        GROUP BY it.inventory_id
      )
      SELECT
        p.id            AS product_id,
        p.name          AS product_name,
        p.sku,
        w.id            AS warehouse_id,
        w.name          AS warehouse_name,
        inv.quantity    AS current_stock,
        inv.low_stock_threshold AS threshold,
        rs.units_sold,
        s.id            AS supplier_id,
        s.name          AS supplier_name,
        s.contact_email AS supplier_email
      FROM inventory inv
      JOIN products p         ON p.id = inv.product_id
      JOIN warehouses w        ON w.id = inv.warehouse_id
      JOIN recent_sales rs     ON rs.inventory_id = inv.id
      LEFT JOIN product_suppliers ps
        ON ps.product_id = p.id AND ps.is_primary = TRUE
      LEFT JOIN suppliers s    ON s.id = ps.supplier_id
      WHERE
        w.company_id = $1
        AND inv.quantity <= inv.low_stock_threshold
      ORDER BY inv.quantity ASC
    ;

    const { rows } = await pool.query(query, [companyId]);

    const alerts = rows.map((row) => {
      const dailyRate = row.units_sold / 30;

      // Guard against divide-by-zero — shouldn't happen given the inner join,
      // but defensive code is better than a NaN sneaking into the response.
      const daysUntilStockout = dailyRate > 0
        ? Math.floor(row.current_stock / dailyRate)
        : null;

      return {
        product_id: row.product_id,
        product_name: row.product_name,
        sku: row.sku,
        warehouse_id: row.warehouse_id,
        warehouse_name: row.warehouse_name,
        current_stock: row.current_stock,
        threshold: row.threshold,
        days_until_stockout: daysUntilStockout,
        supplier: row.supplier_id
          ? {
              id: row.supplier_id,
              name: row.supplier_name,
              contact_email: row.supplier_email,
            }
          : null,
      };
    });

    return res.status(200).json({
      alerts,
      total_alerts: alerts.length,
    });

  } catch (err) {
    // In production I'd use a proper logger (Winston, Pino, etc.) here
    console.error('low-stock-alerts error:', err);
    return res.status(500).json({ error: 'Something went wrong' });
  }
});

module.exports = router;


And the db setup is straightforward — just a shared pool:

js
// db.js
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

module.exports = { pool };


A few things I'd add in a real system:

This endpoint could get expensive at scale. I'd add pagination (?page=1&limit=50) pretty quickly, and probably a short cache (Redis, even just 2-3 minutes) since stock levels don't change on every page load. If this becomes a dashboard widget that gets polled frequently, it's worth precomputing alerts on a schedule and serving from a cache instead of re-running the join every time.

Auth middleware is also missing here — any authenticated user could query any companyId. In practice you'd verify that the requesting user belongs to that company before hitting the DB.

The days_until_stockout: null when the rate is zero is a deliberate choice. Returning null is honest — it means we don't have enough data to calculate it. Returning a magic number like 999 or -1 just pushes the problem to the frontend.

