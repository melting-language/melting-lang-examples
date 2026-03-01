# Bootstrap Admin Panel (SQLite)

A simple admin panel example using **Melt** HTTP server, **Bootstrap 5**, and **SQLite**.

## Requirements

- Melt built with SQLite: `make with-sqlite`
- Melt built with HTTP server (default)

## Run

From the project root:

```bash
./bin/melt examples/admin_panel_sqlite/main.melt
```

Open **http://localhost:8080**

- **Login:** `admin` / `admin` (stored in SQLite; created on first run)
- **Dashboard:** list of users from the database
- **Products:** CRUD with optional image upload (JPEG, PNG, GIF, WebP). Images are stored in `uploads/` (created automatically).
- **Categories:** CRUD for product categories (name, description).
- **Logout:** clears session and redirects to login

## Structure

| File / folder        | Purpose |
|---------------------|--------|
| `main.melt`         | Entry point, HTTP handler, `listen(port)` |
| `config.melt`       | App name, port, SQLite DB path |
| `routes.melt`        | Router: `/`, `/login`, `/dashboard`, `/logout` |
| `db.melt`           | SQLite helpers: ensure schema, check user, list users |
| `controllers/`      | Login (form + POST), Dashboard, 404 |
| `views/`            | Layout (Bootstrap 5), login form, dashboard table, 404 |

## Database

- **Path:** `examples/admin_panel_sqlite/data.sqlite` (set in `config.melt`)
- **Table:** `users (id, username, password)` — created automatically; default user `admin`/`admin` is inserted if the table is empty.
- **Table:** `products (id, name, price, description, image)` — product image is a filename stored in `uploads/` (e.g. `product_1.jpg`).
- **Table:** `categories (id, name, description)` — product categories for grouping.

## Customization

- Change port or DB path in `config.melt`.
- Add more tables and controllers for a fuller admin (e.g. CRUD); use `db.melt` as a pattern for opening the DB, running SQL, and closing.
