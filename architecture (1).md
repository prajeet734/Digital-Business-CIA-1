# Architecture Documentation — Inventory Management System

## 1. Business Problem and Target Users

**Business problem:** Small and mid-sized retail/warehouse businesses lose money through stockouts (lost sales), overstocking (tied-up capital, spoilage/obsolescence), and poor visibility into which products need reordering, from which supplier, and how urgently. This system digitizes stock tracking, purchase ordering, and reorder decision-making so staff and managers can operate from one shared, accurate source of truth instead of spreadsheets.

**Target users (2 roles):**

| Role | Who they are | Core needs |
|---|---|---|
| **Staff** | Warehouse/store employees who handle day-to-day stock movement | Look up products, record stock in/out, raise restock requests, see what's low |
| **Manager/Admin** | Inventory/operations manager | Approve/place purchase orders, manage products/suppliers/staff, view KPIs, see system-wide reorder recommendations |

## 2. Technology Stack

| Layer | Technology | Notes |
|---|---|---|
| Frontend | React (Vite), React Router, Axios | SPA with role-based views/routes |
| Backend / API | Node.js + Express.js | REST API, JSON over HTTPS |
| Authentication | JWT (JSON Web Tokens) + bcrypt for password hashing | Stateless auth, role embedded in token claims |
| Database | PostgreSQL | Relational integrity for products/suppliers/transactions |
| ORM / Query layer | Prisma (or Sequelize) | Schema migrations, parameterized queries |
| Storage | Local filesystem in dev; cloud object storage in production (see §8) | For supplier documents / product images, if used |
| External services | Email/notification service (e.g., for low-stock alerts) — optional | Can be mocked in dev |

## 3. Current System Architecture

```
┌──────────────────────┐        HTTPS/JSON        ┌───────────────────────┐
│   React Frontend     │ ───────────────────────▶ │   Express REST API   │
│  (Staff / Manager UI)│ ◀─────────────────────── │  (Node.js backend)    │
└──────────────────────┘                           └───────────┬───────────┘
                                                                │
                                    ┌───────────────────────────┼───────────────────────────┐
                                    ▼                           ▼                           ▼
                          ┌─────────────────┐        ┌───────────────────┐        ┌──────────────────┐
                          │  Auth middleware │        │  Business logic /  │        │  Data access      │
                          │  (JWT verify,    │        │  reorder algorithm  │        │  layer (Prisma)   │
                          │  role check)     │        │  module             │        │                   │
                          └─────────────────┘        └───────────────────┘        └────────┬──────────┘
                                                                                              ▼
                                                                                   ┌─────────────────────┐
                                                                                   │   PostgreSQL DB      │
                                                                                   │  (products, stock,   │
                                                                                   │   suppliers, orders)  │
                                                                                   └─────────────────────┘
```

**Components:**
- **Frontend:** React SPA. Staff dashboard (stock lookup, record movement, low-stock view). Manager dashboard (KPIs, purchase order approval, staff/product/supplier management).
- **Backend/API:** Express route handlers grouped by resource (`/auth`, `/products`, `/stock-transactions`, `/purchase-orders`, `/suppliers`, `/users`), each calling a service layer.
- **Authentication:** Login issues a JWT containing `userId` and `role`. Middleware verifies the token and enforces role-based access (e.g., only `manager` role can approve purchase orders or manage staff).
- **Database:** PostgreSQL, accessed only through the data access layer — no raw SQL from route handlers.
- **Storage:** Any uploaded files (e.g., supplier invoices) stored outside the DB, referenced by path/URL.
- **External services:** Email notifications for low-stock alerts and purchase-order approvals (optional integration).

## 4. Data Flow Between Major Components

1. User logs in from the React app → credentials sent to `/auth/login` → backend verifies password hash → issues JWT → frontend stores token and attaches it to subsequent requests.
2. Staff records a stock movement (e.g., stock-out for a sale) → frontend calls `POST /stock-transactions` → backend validates the request → writes a `StockTransaction` row → updates `Product.quantity_on_hand` in the same transaction → runs the reorder-check logic → returns updated stock level (and a reorder flag, if triggered) to the frontend.
3. Manager opens the dashboard → frontend calls `GET /dashboard/kpis` and `GET /purchase-orders/recommended` → backend aggregates data from `Products`, `StockTransactions`, and runs the reorder algorithm (§6 in project doc) → returns KPIs and a ranked reorder list.
4. Manager approves a suggested reorder → `POST /purchase-orders` → backend creates a `PurchaseOrder` linked to the `Supplier` and `Product` → status tracked through `Pending → Ordered → Received`.

This demonstrates the required **Interface → Application Logic → Data Layer → Business Output** chain end-to-end.

## 5. Current Hosting / Deployment Approach

- **Development:** Frontend and backend run locally (`npm run dev`), PostgreSQL via Docker container or local install. `.env` files hold DB connection string and JWT secret (not committed).
- **Demo/staging:** Frontend deployed as a static build (e.g., Vercel/Netlify or served by Express as static files); backend deployed as a single Node process (e.g., Render/Railway free tier); database as a managed PostgreSQL instance (e.g., Supabase/Neon free tier) — chosen for zero-ops simplicity, not for production scale.
- Containerization is not required for submission, but a `Dockerfile` + `docker-compose.yml` (frontend, backend, Postgres) is recommended so the whole system can be run with one command for evaluation.

## 6. Proposed Cloud Deployment Architecture (AWS)

```
                        ┌───────────────────────────┐
                        │   Amazon Route 53 (DNS)   │
                        └─────────────┬─────────────┘
                                      ▼
                        ┌───────────────────────────┐
                        │  Amazon CloudFront (CDN)   │──▶ Serves React static build
                        └─────────────┬─────────────┘
                                      ▼
                        ┌───────────────────────────┐
                        │  Application Load Balancer │
                        └─────────────┬─────────────┘
                                      ▼
              ┌───────────────────────────────────────────┐
              │   Auto Scaling Group of EC2 / ECS Fargate   │
              │        (stateless Node.js/Express API)      │
              └─────────────────────┬───────────────────────┘
                                    ▼
              ┌──────────────┐            ┌───────────────────┐
              │  Amazon      │            │  Amazon ElastiCache │
              │  RDS for     │◀──────────▶│  (Redis) — caching,  │
              │  PostgreSQL  │            │  session/rate-limit  │
              │  (Multi-AZ,  │            └───────────────────┘
              │  read        │
              │  replicas)   │
              └──────────────┘
                     ▲
                     │
              ┌──────────────┐
              │  Amazon S3    │  ← product images, supplier docs, backups
              └──────────────┘

     Cross-cutting: Amazon CloudWatch (monitoring/logs), AWS WAF + Shield (network security),
     IAM (access control), AWS Secrets Manager (credentials/JWT secret), SNS/SES (alerts/email)
```

- **Frontend:** Built React bundle served via **S3 + CloudFront** for global low-latency delivery.
- **Backend/API:** Containerized with Docker, run on **ECS Fargate** (or EC2 Auto Scaling Group) behind an **Application Load Balancer**, so instances scale horizontally with traffic.
- **Database:** **Amazon RDS for PostgreSQL** with Multi-AZ failover and read replicas for read-heavy dashboard/report queries.
- **Caching:** **ElastiCache (Redis)** for frequently-read data (product catalog, KPI aggregates) to reduce DB load.
- **Storage:** **S3** for files/images and for database backups.
- **Security/monitoring:** **WAF**, **IAM roles**, **Secrets Manager**, **CloudWatch** for logs/alarms.

### Scaling to 1,000,000 users
- Add more Fargate tasks/EC2 instances behind the ALB (horizontal scaling) — application layer is stateless (JWT, no server-side sessions) so this requires no code change.
- Introduce a read replica for RDS to offload dashboard/report queries from the primary write instance.
- Add Redis caching for hot reads (product lookups, KPI dashboard) to cut DB load significantly.
- Move all static assets fully to CloudFront so origin servers only handle API traffic.

### Scaling to 5,000,000 users
- Partition/shard the database (e.g., by tenant/warehouse region) or move high-volume write tables (`StockTransactions`) to a separate, horizontally-scalable store if write throughput on RDS becomes the bottleneck.
- Introduce a message queue (Amazon SQS) between stock-transaction writes and downstream processing (reorder recalculation, notifications) so writes aren't blocked by heavier computation — decouples "record the transaction" from "run the algorithm."
- Use multiple Availability Zones/regions with CloudFront + Route 53 latency-based routing for global users.
- Move session/rate-limiting fully into Redis cluster mode; consider API Gateway + Lambda for lightweight, spiky endpoints to reduce always-on compute cost.

## 7. Quantitative Scalability Analysis

All calculations follow: **Formula → Values → Calculation → Result → Interpretation.**

### 7.1 User growth (10,000 users, 25% annual growth)

**Formula:** `Users(n) = Users(0) × (1.25)^n`

| Year | Calculation | Result |
|---|---|---|
| 0 | 10,000 | 10,000 |
| 1 | 10,000 × 1.25 | 12,500 |
| 2 | 12,500 × 1.25 | 15,625 |
| 3 | 15,625 × 1.25 | 19,531 |
| 4 | 19,531 × 1.25 | 24,414 |
| 5 | 24,414 × 1.25 | 30,518 |

**Interpretation:** At 25% annual growth, the user base roughly **triples over 5 years**. This growth rate alone would not force architectural changes early on — but combined with the peak-concurrency and request-rate figures below, it shows why the *proposed* architecture (§6), not the current simple deployment, is needed well before reaching millions of users.

### 7.2 Peak concurrent users (10% of registered users active at peak)

**Formula:** `Peak concurrent = Registered users × 0.10`

| Registered users | Calculation | Peak concurrent |
|---|---|---|
| 100,000 | 100,000 × 0.10 | 10,000 |
| 500,000 | 500,000 × 0.10 | 50,000 |
| 1,000,000 | 1,000,000 × 0.10 | 100,000 |
| 5,000,000 | 5,000,000 × 0.10 | 500,000 |

**Interpretation:** A single Node.js process cannot reasonably serve 100,000–500,000 concurrent connections; this is the point at which horizontal scaling (load balancer + auto-scaling group, §6) becomes mandatory rather than optional.

### 7.3 Requests per minute/second (5 requests/min per active user at peak)

**Formula:** `Requests/min = Active users × 5`; `Requests/sec = Requests/min ÷ 60`

| Active users | Requests/min | Requests/sec |
|---|---|---|
| 10,000 | 50,000 | 833.3 |
| 50,000 | 250,000 | 4,166.7 |
| 100,000 | 500,000 | 8,333.3 |
| 500,000 | 2,500,000 | 41,666.7 |

**Interpretation:** At 500,000 active users the API must sustain roughly **41,700 requests per second**. No single server handles this; it requires many stateless API instances behind a load balancer, a caching layer (Redis) in front of the database for the most frequently requested data (product lookups, KPI dashboards), and read replicas so read traffic doesn't compete with write traffic (stock transactions) on the primary database.

## 8. Security Analysis (minimum 8 mechanisms)

| Security Mechanism | Component | Purpose | Threat Addressed |
|---|---|---|---|
| JWT-based authentication | API middleware | Verify user identity on every request without server-side session state | Unauthorized access, session hijacking |
| Password hashing (bcrypt, salted) | Auth service / User table | Store credentials irreversibly | Credential theft from DB breach |
| Role-based access control (RBAC) | API route middleware | Restrict manager-only endpoints (approve PO, manage staff) from staff role | Privilege escalation |
| Input validation & parameterized queries (via ORM) | API request handlers / data access layer | Prevent malformed or malicious input reaching the DB | SQL injection |
| HTTPS/TLS everywhere | Load balancer / CDN | Encrypt data in transit | Man-in-the-middle, eavesdropping |
| Rate limiting | API gateway/middleware (or Redis-backed) | Cap requests per IP/user | Brute-force login attempts, DoS |
| AWS Secrets Manager / environment variables | Deployment config | Keep DB credentials and JWT secret out of source code | Credential leakage via repo |
| Database access restriction (VPC, security groups) | RDS network config | Only application servers can reach the DB, not the public internet | Unauthorized DB access |
| Automated encrypted backups | RDS snapshot / S3 | Recoverable, encrypted point-in-time backups | Data loss, ransomware |
| Centralized logging & monitoring (CloudWatch) | All application/DB layers | Detect anomalous access patterns | Intrusion detection, audit trail |

## 9. Failure and Recovery Analysis (minimum 5 failures, one per required area)

| Failure | Impact | Detection | Recovery |
|---|---|---|---|
| **Application/server** — API process crashes or hangs (e.g., unhandled exception) | Users can't record stock movements or view dashboards | Health-check endpoint fails; CloudWatch alarm / process monitor (PM2) flags instance as unhealthy | Auto Scaling Group / process manager automatically restarts or replaces the unhealthy instance; load balancer stops routing to it in the meantime |
| **Database** — Primary RDS instance becomes unavailable (hardware fault, overload) | No reads/writes possible; all business operations blocked | RDS Multi-AZ health check detects primary failure | Automatic failover to the Multi-AZ standby (typically <60s); application reconnects using the same endpoint; restore from latest automated snapshot if failover also fails |
| **Network** — Load balancer/region network partition or DNS misconfiguration | Users can't reach the application at all | Route 53 health checks against the ALB fail | Route 53 fails over to a healthy region/ALB (if multi-region) or on-call alert triggers manual DNS/ALB fix; CloudFront cache continues serving static assets during the outage |
| **Storage** — S3 bucket unavailable or object corrupted (e.g., supplier document, backup file) | File uploads/downloads fail; backup restore fails | S3 error responses logged; scheduled integrity check on backup restore fails | S3 versioning restores the previous object version; cross-region replication provides a secondary copy for disaster recovery |
| **Security** — Credential leak or brute-force login attempt detected | Risk of unauthorized data access/modification | Rate-limiter/WAF flags abnormal login attempts; CloudWatch alarms on repeated 401s from one IP | Force password reset for affected accounts, rotate JWT signing secret (invalidating existing tokens), block offending IP via WAF, review audit logs for unauthorized changes |

## 10. Summary

This architecture satisfies the CIA-III requirement of a working system today (React + Express + PostgreSQL, role-based access, full CRUD, and a non-trivial reorder algorithm — see `docs/project-implementation.md` for where each task is tracked) while documenting a credible path to a cloud-native, horizontally scalable deployment on AWS capable of supporting 1M–5M users, backed by the quantitative analysis in §7.
