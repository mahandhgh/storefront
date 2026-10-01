# Storefront

A Django backend for an online store. Storefront provides a REST API for the product catalog, shopping carts, customer accounts, and orders, with a Django admin interface for store management.

The main flow is simple: browse products, add items to a cart, sign in, and create an order from that cart.

## Features

- Product collections, prices, inventory, reviews, and image uploads.
- Product search, price and collection filters, sorting, and pagination.
- Shopping carts with UUID identifiers, item quantities, and calculated totals.
- User registration and JWT authentication.
- Customer profiles with phone numbers, birth dates, and membership levels.
- Order creation from a cart within a database transaction.
- Saved item prices for each order, with payment status updates for staff.
- Admin tools for products, collections, customers, and orders.

When an order is created, its items are copied from the cart and the cart is removed. Products linked to order items cannot be deleted through the product API. Collections that contain products cannot be deleted through the collection API.

## Stack

| Component | Technology |
| --- | --- |
| Backend | Django 4.2.23 |
| API | Django REST Framework 3.15.2 |
| Database | MySQL |
| Authentication | Djoser and Simple JWT |
| Filtering and nested routes | django-filter and drf-nested-routers |
| Task processing and cache configuration | Celery and Redis |
| Tests | pytest, pytest-django, and model-bakery |
| Load testing | Locust |
| Development tools | Django Debug Toolbar and Django Silk |
| Static files and application server | WhiteNoise and Gunicorn |

Dependencies are defined in `Pipfile` and `Pipfile.lock`. The Pipfile specifies Python 3.13. This is a configuration value; compatibility with that version has not been verified here.

## Local setup

Install Python, Pipenv, and MySQL. The `mysqlclient` package can also require MySQL client libraries and build tools on your operating system.

Run the commands below from the project directory that contains `manage.py`.

### 1. Install dependencies

```bash
python -m pip install pipenv
pipenv sync --dev
```

Create a fresh environment rather than using an environment copied from another computer.

### 2. Configure MySQL

Create the database in MySQL:

```sql
CREATE DATABASE storefront3 CHARACTER SET utf8mb4;
```

Update `DATABASES` in `storefront/settings/dev.py` with your local credentials:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'storefront3',
        'HOST': 'localhost',
        'USER': 'your_mysql_user',
        'PASSWORD': 'your_mysql_password',
    }
}
```

The development settings currently read these values directly from the settings file. An `.env` file is not loaded automatically.

### 3. Apply migrations and create an admin account

```bash
pipenv run python manage.py migrate
pipenv run python manage.py createsuperuser
```

To add sample collections and products to a fresh database:

```bash
pipenv run python manage.py seed_db
```

The seed command uses fixed IDs. Run it once on a fresh database to avoid conflicts with existing records.

### 4. Start the server

```bash
pipenv run python manage.py runserver
```

- API root: <http://127.0.0.1:8000/store/>
- Admin: <http://127.0.0.1:8000/admin/>
- Home page: <http://127.0.0.1:8000/>

The home page is a small template. Store operations are available through the API and admin interface.

## API routes

| Route | Purpose |
| --- | --- |
| `/store/products/` | Browse products; staff can create and edit products |
| `/store/products/{id}/reviews/` | Product reviews |
| `/store/products/{id}/images/` | Product images |
| `/store/collections/` | Browse collections; staff can create and edit collections |
| `/store/carts/` | Create a cart |
| `/store/carts/{uuid}/` | Retrieve or delete a cart |
| `/store/carts/{uuid}/items/` | List or add cart items |
| `/store/carts/{uuid}/items/{id}/` | Retrieve, update, or delete an item |
| `/store/customers/` | Staff access to customer records |
| `/store/customers/me/` | Read or update the signed-in customer's profile |
| `/store/orders/` | Create orders and view accessible orders |
| `/auth/users/` | Register a user |
| `/auth/users/me/` | Read the signed-in user's account |
| `/auth/jwt/create/` | Obtain access and refresh tokens |
| `/auth/jwt/refresh/` | Refresh an access token |
| `/auth/jwt/verify/` | Verify a token |

Customers can view their own orders. Staff can view all orders and update payment status or delete orders. Product lists return 10 items per page. Product images have a 500 KB upload limit.

### Search and filters

```text
/store/products/?search=coffee
/store/products/?collection_id=2
/store/products/?unit_price__gt=10&unit_price__lt=50
/store/products/?ordering=unit_price
/store/products/?ordering=-last_update
/store/products/?page=2
```

### Authentication

Send a POST request to `/auth/jwt/create/`:

```json
{
  "username": "your_username",
  "password": "your_password"
}
```

Use the returned access token in requests that require authentication:

```http
Authorization: JWT <access_token>
```

The configured header prefix is `JWT`. Access tokens expire after one day.

### Create an order

1. Send a POST request to `/store/carts/` and save the returned cart ID.
2. Add a product through `/store/carts/{uuid}/items/`:

   ```json
   {
     "product_id": 1,
     "quantity": 2
   }
   ```

3. Sign in and send a POST request to `/store/orders/` with the JWT header:

   ```json
   {
     "cart_id": "<cart_uuid>"
   }
   ```

Replace the product ID with an existing product. Empty carts are rejected. The order stores each item's quantity and unit price at the time of creation.

## Background tasks and Redis

Redis is configured at `localhost:6379`. Celery uses database `1`; the Django cache uses database `2`.

With Redis running, start a worker in a separate terminal:

```bash
pipenv run celery -A storefront worker --loglevel=info
```

To run scheduled tasks:

```bash
pipenv run celery -A storefront beat --loglevel=info
```

The current schedule calls `playground.tasks.notify_customers` every five seconds. That task prints messages and waits; it does not send email. Celery Beat is optional for the store API.

## Tests

With MySQL running and development dependencies installed:

```bash
pipenv run pytest store/tests/
```

The existing API tests cover collection creation permissions, input validation, and collection retrieval. The database user must have permission to create the test database.

To start Locust while the Django server is running:

```bash
pipenv run locust -f locustfiles/browse_products.py --host=http://127.0.0.1:8000
```

Open <http://localhost:8089/> to configure the load test. The script uses fixed product ID ranges and also calls a playground route that depends on an external HTTP service.

## Project layout

| Path | Responsibility |
| --- | --- |
| `store/` | Models, serializers, API views, admin configuration, and tests |
| `core/` | Custom user model, account serializers, signals, and home template |
| `tags/` | Generic tagging model |
| `likes/` | Generic likes model |
| `playground/` | Utility views and sample background task |
| `storefront/settings/` | Shared, development, and production settings |
| `storefront/celery.py` | Celery application configuration |
| `locustfiles/` | Load test scenarios |
| `store/management/commands/` | Database seed command and sample data |

## Current scope

Order payment status is stored in the database; payment processing is not integrated. The customer history action currently returns a placeholder response. Tags and likes have models but no dedicated API routes.

Production settings are separate from development settings, but deployment configuration is incomplete. Before deployment, configure the database, allowed hosts, secret key, media storage, email service, and access rules for public write endpoints. The production settings require `SECRET_KEY` from the environment. Local development settings are selected by default.
