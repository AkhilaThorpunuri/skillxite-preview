# 11. Databases, Authentication, and Authorization

[⬅️ Previous:](10-rest-apis-and-api-integration.md) ·

> 🔐 AI Developer — Web Development 

## 🎯 Learning Objectives

By the end of this topic, you will be able to:

- Understand relational and non-relational databases.
- Connect a Node.js backend to PostgreSQL.
- Create tables and perform CRUD operations.
- Explain authentication versus authorization.
- Hash passwords and verify user credentials.
- Understand sessions, cookies, access tokens, and role-based access control.
- Protect backend routes and prevent common security vulnerabilities.

## 1. 🗄️ Why Do Web Applications Need Databases?

A backend needs persistent storage for information such as users, products, orders, and payments.

An in-memory JavaScript array is useful for demonstrations, but its contents disappear when the process restarts. A database stores information so that it can be retrieved and updated across application sessions.

### Database Choices

| Database | Type | Common use |
|---|---|---|
| PostgreSQL | Relational SQL | Users, products, orders, financial records |
| MySQL | Relational SQL | Business applications and transactional data |
| MongoDB | Document database | Flexible document-oriented data |
| Redis | In-memory data store | Caching, temporary data, rate limits |

Choose a database based on data relationships, consistency requirements, query patterns, operational needs, and team familiarity.

## 2. 🧱 Understand Relational Database Design

In a relational database, information is stored in tables containing rows and columns. Tables can be connected using keys.

For a boutique application, consider the following tables:

```mermaid
erDiagram
    USERS ||--o{ ORDERS : places
    ORDERS ||--|{ ORDER_ITEMS : contains
    PRODUCTS ||--o{ ORDER_ITEMS : appears_in

    USERS {
        int id PK
        string email UK
        string password_hash
        string role
    }

    PRODUCTS {
        int id PK
        string name
        string category
        numeric price
        int stock
    }

    ORDERS {
        int id PK
        int user_id FK
        string status
    }

    ORDER_ITEMS {
        int order_id FK
        int product_id FK
        int quantity
        numeric unit_price
    }
```

- **Primary key:** Uniquely identifies a row.
- **Foreign key:** Connects a row to a related record.
- **Unique constraint:** Prevents duplicate values in a column or column combination.
- **Index:** Helps the database find records efficiently.
- **Transaction:** Groups operations so they can succeed or fail together.

For example, an order should reference the customer who placed it, and an order item should reference the product being purchased.

## 3. 🐘 Set Up PostgreSQL

Install PostgreSQL using the [official PostgreSQL website](https://www.postgresql.org/download/) or use an existing development database.

If you already run PostgreSQL through Docker, you can reuse that database. Otherwise, install PostgreSQL and create a database named `boutique_db`.

Open `psql` or a database administration tool and run:

```sql
CREATE DATABASE boutique_db;
```

Connect to `boutique_db` and create the tables:

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    role TEXT NOT NULL DEFAULT 'customer'
        CHECK (role IN ('customer', 'manager', 'owner')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    category TEXT NOT NULL,
    price NUMERIC(12, 2) NOT NULL CHECK (price > 0),
    stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_category ON products(category);
```

The database enforces important constraints even if a frontend accidentally submits invalid data.

## 4. 🔌 Connect PostgreSQL to Node.js

Install the PostgreSQL driver:

```bash
npm install pg
```

Create a `.env` file in the backend root:

```env
DATABASE_URL=postgresql://YOUR_USER:YOUR_PASSWORD@localhost:5432/boutique_db
```

Replace the placeholders with your actual database credentials. Do not commit this file.

Create `src/db.js`:

```javascript
require("dotenv").config();

const { Pool } = require("pg");

if (!process.env.DATABASE_URL) {
  throw new Error("DATABASE_URL is not configured");
}

const pool = new Pool({
  connectionString: process.env.DATABASE_URL
});

module.exports = pool;
```

A connection pool reuses database connections rather than opening a new connection for every query.

## 5. 📋 Perform CRUD Operations

CRUD means Create, Read, Update, and Delete.

Create `src/routes/dbProductRoutes.js`:

```javascript
const express = require("express");
const pool = require("../db");

const router = express.Router();

// READ: Retrieve products
router.get("/", async (req, res, next) => {
  try {
    const result = await pool.query(
      `SELECT id, name, category, price, stock
       FROM products
       ORDER BY id DESC`
    );

    res.json({ products: result.rows });
  } catch (error) {
    next(error);
  }
});

// CREATE: Add a product
router.post("/", async (req, res, next) => {
  try {
    const { name, category, price, stock } = req.body;

    if (
      typeof name !== "string" ||
      !name.trim() ||
      typeof category !== "string" ||
      !category.trim() ||
      !Number.isFinite(price) ||
      price <= 0 ||
      !Number.isInteger(stock) ||
      stock < 0
    ) {
      return res.status(400).json({
        error: "Invalid product data"
      });
    }

    const result = await pool.query(
      `INSERT INTO products (name, category, price, stock)
       VALUES ($1, $2, $3, $4)
       RETURNING id, name, category, price, stock`,
      [name.trim(), category.trim(), price, stock]
    );

    res.status(201).json(result.rows[0]);
  } catch (error) {
    next(error);
  }
});

module.exports = router;
```

Register the routes in `server.js`:

```javascript
const dbProductRoutes = require("./routes/dbProductRoutes");

app.use("/api/db-products", dbProductRoutes);
```

This sample demonstrates database reads and inserts. For a production application, implement update and delete routes, pagination, database error classification, and authentication and authorization checks before allowing modifications.

### Why Use Parameterized Queries?

Notice the query uses `$1`, `$2`, `$3`, and `$4`.

Parameterized queries separate SQL instructions from user-provided values, helping prevent SQL injection.

Avoid this unsafe pattern:

```javascript
// UNSAFE: Do not concatenate user input into SQL.
const query = `SELECT * FROM products WHERE name = '${req.query.name}'`;
```

Use parameters instead:

```javascript
const result = await pool.query(
  "SELECT id, name, price FROM products WHERE name = $1",
  [req.query.name]
);
```

Table names and column names generally cannot be supplied as ordinary value parameters. If they must vary dynamically, use an explicit allowlist.

## 6. 🔑 Authentication vs. Authorization

These concepts are related but different.

- **Authentication:** Who are you? For example, verifying a user's password or validating their identity through an identity provider.
- **Authorization:** What are you allowed to do? For example, determining whether a manager can create products or whether an owner can view financial records.

```mermaid
flowchart TD
    A["Incoming request"] --> B["Authenticate identity"]
    B --> C{"Identity valid?"}
    C -- "No" --> D["401 Unauthorized"]
    C -- "Yes" --> E["Check permissions"]
    E --> F{"Permission granted?"}
    F -- "No" --> G["403 Forbidden"]
    F -- "Yes" --> H["Perform operation"]

    classDef auth fill:#dbeafe,stroke:#2563eb,color:#172554
    classDef deny fill:#fee2e2,stroke:#dc2626,color:#7f1d1d
    classDef allow fill:#dcfce7,stroke:#16a34a,color:#14532d

    class A,B,C,E,F auth
    class D,G deny
    class H allow
```

A `401` response usually means valid authentication is missing or failed. A `403` response means the server understood the request but will not permit the operation.

## 7. 🔒 Password Hashing

Never store passwords in plaintext or use reversible encryption as a substitute for password hashing.

Password hashing functions such as Argon2id, scrypt, or appropriately configured bcrypt are designed to make offline password guessing more difficult.

Install bcrypt for this example:

```bash
npm install bcrypt
```

Hash a password during account creation:

```javascript
const bcrypt = require("bcrypt");

async function hashPassword(password) {
  if (typeof password !== "string" || password.length < 12) {
    throw new Error("Password does not meet the minimum length");
  }

  const passwordHash = await bcrypt.hash(password, 12);

  return passwordHash;
}
```

The example uses a work factor of `12`; select the appropriate cost through performance testing and current security guidance. A production password policy should also account for long passwords, maximum input size, breached passwords, and user experience.

Verify a login password:

```javascript
async function verifyPassword(password, passwordHash) {
  return bcrypt.compare(password, passwordHash);
}
```

Store only the password hash in the database. Never log the submitted password or return the hash to the frontend.

## 8. 🍪 Sessions and Tokens

Two common authentication approaches are cookie-based sessions and token-based authentication.

### Session-Based Authentication

1. The user submits login credentials.
2. The backend verifies the credentials.
3. The server creates a session.
4. The browser stores a session identifier in a cookie.
5. Subsequent requests send the cookie.
6. The server validates the session and checks permissions.

Session cookies should generally use appropriate `HttpOnly`, `Secure`, and `SameSite` settings. Cookie-based authentication also requires protection against cross-site request forgery (CSRF) where applicable.

### Token-Based Authentication

A server or identity provider issues a token after successful authentication. The client sends the token to protected endpoints, commonly through an `Authorization: Bearer ...` header.

Tokens require careful handling of expiration, revocation, storage, and signing keys. A signed token is not necessarily encrypted, and its payload should not contain secrets.

For many browser-based applications, secure server-managed sessions are a straightforward choice. Use a well-maintained authentication framework or identity provider rather than inventing a custom authentication protocol.

## 9. 🛡️ Implement Role-Based Access Control

Suppose a boutique has three roles:

| Role | Example permissions |
|---|---|
| Customer | Browse products and manage their own orders |
| Manager | Create and update product catalog entries |
| Owner | Access permitted financial reports and manage business settings |

Roles should be assigned and validated by trusted server-side logic. Never accept a user-selected role from a registration form as proof of permission.

Example middleware for a backend that has already established `req.user` through a trusted authentication middleware:

```javascript
function requireRole(...allowedRoles) {
  return function (req, res, next) {
    if (!req.user) {
      return res.status(401).json({
        error: "Authentication required"
      });
    }

    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({
        error: "Insufficient permissions"
      });
    }

    next();
  };
}
```

Use it on protected routes:

```javascript
app.post(
  "/api/db-products",
  requireRole("manager", "owner"),
  createProductHandler
);

app.get(
  "/api/owner/financial-summary",
  requireRole("owner"),
  financialSummaryHandler
);
```

`createProductHandler` and `financialSummaryHandler` represent your actual route handlers. The middleware is only one part of the solution; the application must authenticate users securely and attach a trusted user identity before these routes execute.

For customer-specific records, also check ownership. A customer should not be able to retrieve another customer's order merely by changing an ID in the URL.

## 10. 🚨 Essential Security Protections

- **SQL injection:** Use parameterized database queries.
- **Broken access control:** Verify permissions for every protected operation.
- **Password exposure:** Store password hashes, not plaintext passwords.
- **Brute-force attacks:** Apply rate limiting and monitoring to login endpoints.
- **Session theft:** Use secure cookies, HTTPS, appropriate expiration, and session invalidation.
- **CSRF:** Protect cookie-authenticated state-changing requests.
- **XSS:** Escape output and avoid unsafe HTML rendering.
- **Secret leakage:** Keep credentials out of Git and frontend bundles.
- **Sensitive data exposure:** Return only the fields required by the caller.
- **Insecure object access:** Verify ownership or appropriate permissions on every record request.

Security should be enforced by the backend and database, not just by hiding buttons in the frontend.

## 11. 🏪 Real-World Scenario: Secure Boutique Management

A manager signs in and tries to add a new dress.

1. The server validates the manager's authentication.
2. Authorization checks whether the account has catalog-management permissions.
3. The server validates the product details.
4. PostgreSQL saves the product.
5. The API returns the new product.

If a customer tries the same request, the server rejects it even if the customer manually sends the API request outside the browser UI.

## 12. ⚠️ Common Mistakes

- Storing plaintext passwords.
- Using SQL string concatenation with user input.
- Treating a role supplied by the frontend as trustworthy.
- Protecting a page but forgetting to protect its API.
- Returning all database columns, including sensitive ones.
- Forgetting to verify record ownership.
- Using weak or permanent session credentials.
- Failing to handle duplicate emails or database constraint violations.
- Treating a successful login as proof that every operation is authorized.

## 🔑 Key Takeaways

- Databases provide persistent storage and relational constraints.
- Parameterized queries protect SQL values from injection.
- Authentication establishes identity; authorization determines permissions.
- Passwords must be stored as secure hashes.
- Every protected backend operation needs server-side access checks.
- Authentication, database design, and authorization must work together.
