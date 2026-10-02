# Personal Budget API
 
A RESTful backend API built with **Node.js**, **Express**, and **PostgreSQL** that implements a budget management system based on the envelope budgeting method, including transaction tracking.
 
This project was completed as part of the **Codecademy Back-End Engineer Career Path** and focuses on clean API design, database integration, proper error handling, and separation of concerns.
 
---
 
## Live Demo
 
The API is deployed and available at:
 
**https://personal-budget-2-6x9k.onrender.com**
 
> Note: The free Render instance spins down after inactivity. The first request may take up to 50 seconds to respond.
 
---
 
## Features
 
- Initialize and update a total budget
- Create, read, update, and delete budget envelopes
- Transfer funds between envelopes
- Record, read, and delete transactions (payments from envelopes)
- Automatic envelope debit on transaction creation
- Automatic envelope credit on transaction deletion
- Immutable transactions (delete and re-create to correct errors)
- Dynamic computation of available (unallocated) budget
- Centralized error handling with meaningful HTTP status codes
- Full OpenAPI 3.0 documentation
---
 
## Tech Stack
 
- **Node.js** and **Express** for the HTTP layer
- **PostgreSQL** for persistent storage, accessed via the `pg` driver
- **dotenv** for environment-based configuration
- **nodemon** for development auto-reload
---
 
## Architecture
 
The project follows a clear separation of concerns across four layers:
 
- **Router**: HTTP validation, request parsing, response formatting
- **Controller**: business logic and domain rules (budget invariants, envelope debit and credit)
- **Database layer**: parameterized SQL queries via a shared `pg` connection pool
- **Error handling**: custom `httpError` factory and global error middleware
```
personal-budget-api/
├── server.js
├── openapi.yaml                # OpenAPI 3.0 documentation
├── .env                        # secrets (gitignored)
├── .env.example                # template for contributors
├── db/
│   ├── index.js                # shared pg Pool instance
│   ├── schema.sql              # versioned database schema
│   ├── budget.js               # budget queries
│   ├── envelopes.js            # envelope queries
│   └── transactions.js         # transaction queries
├── controllers/
│   ├── budget-controller.js
│   └── transactions-controller.js
├── routes/
│   ├── budget-router.js
│   └── transactions-router.js
├── errors/
│   └── httpError.js
├── middleware/
│   └── error-handler.js
└── package.json
```
 
---
 
## API Documentation
 
The full API is documented in `openapi.yaml` following the OpenAPI 3.0 specification. You can explore it by importing the file into [Swagger Editor](https://editor.swagger.io/) or [Postman](https://www.postman.com/).
 
---
 
## Database Schema
 
Three tables with a foreign key from `transactions` to `envelopes`:
 
- **budget**: singleton row (enforced with `CHECK (id = 1)`) holding the global `total_amount`
- **envelopes**: user-created categories, each with an allocated amount
- **transactions**: immutable records of payments, each linked to an envelope
The available budget is never stored. It is computed on demand as `total_amount - SUM(envelope_amount)` to avoid synchronization issues.
 
See `db/schema.sql` for the full DDL.
 
---
 
## API Endpoints
 
Base URLs:
- Budget and envelopes: `/api/budget`
- Transactions: `/api/transactions`
### Budget
 
#### Initialize or update total budget
**POST** `/api/budget/init`
 
```json
{
  "totalBudget": 1000
}
```
 
The first call creates the budget row. Subsequent calls update it.
 
---
 
### Envelopes
 
#### Create an envelope
**POST** `/api/budget/envelopes`
 
```json
{
  "name": "Food",
  "allocatedAmount": 200
}
```
 
Rejected with `400` if the allocated amount exceeds the available budget.
 
#### Get all envelopes
**GET** `/api/budget/envelopes`
 
#### Get a single envelope
**GET** `/api/budget/envelopes/:id`
 
#### Get full status (envelopes, total, available)
**GET** `/api/budget/status`
 
#### Update an envelope (partial update)
**PUT** `/api/budget/envelopes/:id`
 
```json
{
  "name": "Groceries",
  "amount": 250
}
```
 
Either field is optional. Only provided fields are updated.
 
#### Delete an envelope
**DELETE** `/api/budget/envelopes/:id`
 
#### Transfer funds between envelopes
**POST** `/api/budget/envelopes/transfer`
 
```json
{
  "fromEnvelopeId": 1,
  "toEnvelopeId": 2,
  "amount": 50
}
```
 
---
 
### Transactions
 
#### Create a transaction
**POST** `/api/transactions`
 
```json
{
  "recipient": "Carrefour",
  "transactionAmount": 25,
  "transactionDate": "2026-04-24",
  "envelopeId": 1
}
```
 
The designated envelope is automatically debited. Rejected with `400` if the envelope has insufficient funds, or `404` if the envelope does not exist.
 
#### Get all transactions
**GET** `/api/transactions/status`
 
#### Get a transaction by ID
**GET** `/api/transactions/:id`
 
#### Delete a transaction
**DELETE** `/api/transactions/:id`
 
Deleting a transaction credits the associated envelope back with the payment amount.
 
---
 
## Error Handling
 
- Uses a centralized global error middleware
- Controllers and routers throw typed HTTP errors via `httpError(status, message)`
- Async handlers propagate errors with `try/catch` and `next(error)`
- Returned status codes:
  - `400` Bad Request (invalid input or business rule violation)
  - `404` Not Found (resource does not exist)
  - `500` Internal Server Error (unexpected failure)
Example error response:
```json
{
  "success": false,
  "message": "Insufficient funds in the source envelope"
}
```
 
---
 
## Testing
 
All endpoints were tested manually using **Postman** across nominal and edge cases (invalid IDs, insufficient funds, missing envelopes, malformed bodies).
 
---
 
## Installation and Usage
 
### Prerequisites
 
- Node.js 18+
- PostgreSQL 13+ running locally
### Clone and install dependencies
 
```bash
git clone git@github.com:julperret/personal-budget-api.git
cd personal-budget-api
npm install
```
 
### Configure environment
 
Copy the provided template and fill in your local credentials:
 
```bash
cp .env.example .env
```
 
Expected variables:
 
```
PGHOST=localhost
PGPORT=5432
PGDATABASE=personal_budget
PGUSER=postgres
PGPASSWORD=your_password
```
 
### Initialize the database
 
Create the database and apply the schema:
 
```bash
createdb personal_budget
psql -d personal_budget -f db/schema.sql
```
 
### Run the server
 
```bash
npm start
```
 
### Run in development mode (with auto-reload)
 
```bash
npm run dev
```
 
The API will be available at `http://localhost:3000`.
 
---
 
## Deployment
 
The API is deployed on **Render** using:
 
- A **PostgreSQL** instance (Frankfurt, EU Central)
- A **Web Service** connected to this GitHub repository, with auto-deploy on push to `main`
To deploy your own instance:
1. Create a PostgreSQL database on Render
2. Apply the schema: `psql "[External Database URL]?sslmode=require" -f db/schema.sql`
3. Create a Web Service pointing to this repo
4. Set the five `PG*` environment variables using the Render database credentials
---
 
## Design Notes
 
- **Transactions are immutable.** To correct a transaction, delete it and create a new one. This mirrors standard accounting practices and avoids complex rebalancing logic.
- **The available budget is computed, not stored.** This removes a whole class of synchronization bugs between the total and the sum of envelope allocations.
- **Database names leak into API responses.** The current implementation returns raw database rows (`envelope_amount`, `payment_recipient`, etc.). A dedicated DTO layer would clean this up and decouple the API contract from the database schema. This is tracked as a future improvement.
- **No authentication.** The API assumes a single-user context, consistent with the project scope.
---
 
## What I Learned
 
- Designing a REST API with proper HTTP semantics
- Migrating an in-memory data model to a persistent relational database
- Writing parameterized SQL queries with `pg` to prevent injection
- Using `RETURNING *` to retrieve inserted or updated rows in a single round-trip
- Structuring an Express application in clearly separated layers
- Handling asynchronous errors with `try/catch` and `next(error)`
- Configuring environments with `.env` and keeping secrets out of version control
- Handling PostgreSQL type quirks (`NUMERIC` as string, `DATE` and timezones)
- Writing OpenAPI 3.0 documentation from an existing codebase
- Deploying a Node.js/PostgreSQL API to Render
---
 
## Possible Extensions
 
- DTO layer to normalize API responses (camelCase, typed values)
- Authentication and multi-user support
- Monthly budget periods and historical reports
- Categories and tags for transactions
- Automated test suite (unit and integration)
---
 
## Author
 
**Julien Perret**
GitHub: [julperret](https://github.com/julperret)
 
