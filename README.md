# yourz-ugc-backend-architecture

Architectural documentation and system design for the Yourz UGC Marketplace backend.

> Documentation-only public showcase for a private commercial UGC advertising platform backend.

![Node.js](https://img.shields.io/badge/Node.js-Backend-339933?logo=node.js&logoColor=white)
![Express](https://img.shields.io/badge/Express.js-REST_API-000000?logo=express&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-Mongoose-47A248?logo=mongodb&logoColor=white)
![Socket.IO](https://img.shields.io/badge/Socket.IO-Realtime-010101?logo=socket.io&logoColor=white)
![Cloudflare R2](https://img.shields.io/badge/Cloudflare_R2-Object_Storage-F38020?logo=cloudflare&logoColor=white)
![Paymob](https://img.shields.io/badge/Paymob-Payments_&_Payouts-0047AB)

## Project Title & TL;DR

**Yourz is a UGC advertising marketplace backend connecting brands, customers, creators, and admins through paid content projects, media approvals, real-time communication, checkout, and creator payouts.**

This showcase repository intentionally does **not** include proprietary source code. It documents the backend architecture, data model, integrations, and engineering decisions that powered the private production system.

## Confidentiality Notice

- No proprietary source code is included.
- No secret keys, environment values, webhook secrets, payment credentials, or database credentials are included.
- Business-specific algorithms and commercial rules are described only at a high level.
- The goal is to demonstrate backend architecture, system design, and senior engineering judgment.

## System Architecture

Yourz was implemented as a **modular monolithic Node.js backend**. This architecture kept development velocity high while still enforcing clear boundaries across marketplace domains such as creators, customers, brands, projects, finance, media, notifications, and admin operations.

```text
Client Applications
  |-- Creator Dashboard
  |-- Customer / Brand Dashboard
  |-- Admin Dashboard
  |
  v
Express.js API Gateway Layer
  |-- Auth and RBAC Middleware
  |-- Project Access Middleware
  |-- Request Validation
  |-- REST Route Modules
  |
  v
Domain Modules
  |-- Creator and Customer Profiles
  |-- Brand Management
  |-- Project Lifecycle
  |-- Media Approval Workflow
  |-- Finance Records
  |-- Notifications
  |-- Admin Operations
  |
  +--------------------+
  | MongoDB / Mongoose |
  +--------------------+
  |
  +-----------------------+
  | External Integrations |
  +-----------------------+
  |-- Paymob Checkout
  |-- Paymob Payouts
  |-- Cloudflare R2
  |-- SMTP Email
  |-- Exchange Rate API
  |-- Socket.IO Realtime Layer
```

### Architectural Style

- **Backend type:** Modular monolith with separated route, controller, service, model, middleware, utility, and config layers.
- **API style:** REST APIs for core workflows, Socket.IO for real-time collaboration.
- **Persistence:** MongoDB with Mongoose schemas, embedded subdocuments for request histories, and references for cross-domain relationships.
- **Media strategy:** Hybrid media layer with direct-to-cloud uploads for large UGC files and metadata persistence in MongoDB.
- **Realtime strategy:** Authenticated Socket.IO rooms for project messaging, per-user notifications, and live workflow updates.
- **Payment strategy:** Paymob checkout for customer payments and Paymob payout integrations for creator settlements.

## Tech Stack

| Area           | Technologies                                               |
| -------------- | ---------------------------------------------------------- |
| Runtime        | Node.js                                                    |
| API Framework  | Express.js                                                 |
| Database       | MongoDB, Mongoose                                          |
| Realtime       | Socket.IO                                                  |
| Authentication | JWT, bcrypt, OTP flows                                     |
| Media          | Multer, Sharp, Cloudflare R2, AWS S3 SDK                   |
| Payments       | Paymob Checkout, Paymob Payouts, Webhooks                  |
| Messaging      | Socket.IO, persisted notifications, SMTP email             |
| Jobs           | node-cron                                                  |
| DevOps         | Docker, Vercel configuration, environment-based deployment |

## Core Backend Domains

### Creator Domain

The creator module manages creator identity, profile metadata, social links, verification status, package offerings, bank/wallet payout details, ratings, suspension state, and content portfolio.

Key backend responsibilities:

- Creator registration and authentication.
- Profile and portfolio management.
- Package configuration with pricing and platform fee calculations.
- Verification and moderation status tracking.
- Payout destination management for bank and wallet flows.

### Customer and Brand Domain

Customers and brands initiate advertising projects, manage campaign details, review creator submissions, approve media, request changes, and track project completion.

Key backend responsibilities:

- Brand ownership and customer-brand relationships.
- Project access control for customer-side users.
- Campaign lifecycle management.
- Review, report, and completion workflows.

### Project Domain

Projects are the core marketplace object. They connect customers, creators, packages, deadlines, media assets, financial records, status transitions, and collaboration messages.

Conceptually, a project contains:

- Customer reference.
- Creator reference.
- Brand reference.
- Package snapshot.
- Currency and pricing metadata.
- Project status.
- Media references.
- Request history for package/deadline changes.
- Access list for authorized users.

### Media Domain

UGC platforms are media-heavy by design. Yourz separates binary asset storage from metadata storage:

- Large files are uploaded to object storage.
- MongoDB stores metadata such as owner, project, media type, public URL, approval status, timestamps, and display metadata.
- Customers approve or reject submitted media.
- Realtime notifications keep both sides updated.

### Finance Domain

Finance records act as the bridge between project state and payment state. They track payment intent, checkout status, payout status, project references, creator/customer relationships, currency, and reconciliation status.

## Core Technical Challenges & Solutions

### 1. Multi-Role Marketplace Access Control

**Challenge:** A UGC platform has multiple user types: creators, customers, brand users, admins, and super admins. Each role needs different project visibility and actions.

**Solution:**

- Implemented JWT authentication as the base identity layer.
- Added role-aware middleware for admin and customer permissions.
- Added project-level access checks before sensitive operations.
- Used project membership/access lists to authorize messaging, media operations, and project state changes.

**Engineering impact:**

- Prevented unauthorized media access and project updates.
- Kept controller logic cleaner by centralizing access checks.
- Supported flexible project collaboration without exposing unrelated customer or creator data.

### 2. Direct-to-Cloud UGC Uploads

**Challenge:** UGC projects can include large images, videos, PDFs, identity documents, and portfolio assets. Uploading all files through the API server would increase latency and server bandwidth cost.

**Solution:**

- Designed a Cloudflare R2 direct upload flow using presigned URLs.
- Backend validates upload requests and returns time-limited upload URLs.
- Frontend uploads files directly to R2.
- Backend confirms upload completion and persists media metadata.
- Batch upload support reduces API round trips for multi-file campaigns.

```text
Creator Client
  -> Request upload URL from API
  -> Receive presigned R2 URL
  -> Upload binary file directly to R2
  -> Confirm upload metadata with API
  -> API creates Media record and links it to Project
```

**Engineering impact:**

- Reduced API server bandwidth usage.
- Improved upload reliability for large media.
- Kept media ownership and approval state controlled by backend metadata.

### 3. Media Approval Workflow

**Challenge:** Customers need to review and approve creator-submitted content before it becomes accepted project output.

**Solution:**

- Media records include approval status, approver reference, approval timestamp, project reference, and media type.
- Project media lists are filtered by access rules and approval state.
- Notifications are generated when media is uploaded, approved, or removed.

**Engineering impact:**

- Created auditable content review history.
- Supported asynchronous collaboration between creator and customer.
- Prevented unapproved deliverables from being treated as final campaign assets.

### 4. Paymob Checkout and Creator Payouts

**Challenge:** The marketplace needed customer payment collection and creator settlement across regional payment flows.

**Solution:**

- Integrated Paymob checkout for customer payments.
- Created finance records before redirecting to checkout.
- Processed payment callbacks to reconcile finance status and project state.
- Implemented creator payout workflows using country-specific payout payloads.
- Validated payout prerequisites such as bank or wallet details before settlement.

```text
Customer Confirms Project
  -> API creates Finance record
  -> API creates Paymob checkout intention
  -> Customer completes payment
  -> Paymob webhook updates Finance record
  -> Project status advances
  -> Creator payout record becomes eligible
```

**Engineering impact:**

- Separated business workflow state from payment gateway state.
- Made payment reconciliation traceable through finance records.
- Supported marketplace revenue sharing and creator proceeds.

### 5. Real-Time Project Messaging

**Challenge:** Project collaboration requires immediate communication without exposing project rooms to unauthorized users.

**Solution:**

- Socket.IO authentication verifies user identity.
- Users join project rooms only after project access checks.
- Messages are persisted before broadcast.
- Per-user notification rooms deliver new message alerts.

**Engineering impact:**

- Enabled real-time collaboration inside active projects.
- Preserved message history for late joiners and reloads.
- Kept unauthorized users out of project rooms.

### 6. Scheduled Operational Automation

**Challenge:** Marketplace state changes are time-sensitive: verification can expire, suspensions can end, promo codes can activate/deactivate, and exchange rates can change.

**Solution:**

- Used scheduled jobs for routine state transitions.
- Updated promo code availability based on dates.
- Refreshed currency conversion values through an external exchange-rate provider.
- Reset moderation/verification state when configured periods expire.

**Engineering impact:**

- Reduced manual admin operations.
- Kept pricing and status data current.
- Improved operational reliability.

## Database Modeling (High-Level)

Yourz uses MongoDB and Mongoose to model a marketplace with many role-specific workflows.

### Main Model Groups

| Group         | Conceptual Models                                   |
| ------------- | --------------------------------------------------- |
| Identity      | Creator, Customer, Admin, OTP, Verification         |
| Marketplace   | Brand, Project, YourzProject, Package, YourzPackage |
| Media         | Media, Collaboration Logo                           |
| Finance       | Finance, Promo Code, Currency                       |
| Collaboration | Message, Notification, Review, Report, Request      |

### Modeling Decisions

- **References for core ownership:** Projects reference creators, customers, brands, media, and packages.
- **Embedded request history:** Package and deadline requests can live as embedded subdocuments inside projects to preserve negotiation context.
- **Snapshot pricing:** Project packages store pricing and platform-fee snapshots so historical finance records remain stable even if package settings change later.
- **Indexes for discovery:** Text/name indexes support search use cases for creators and projects.
- **Notification persistence:** Notifications are stored so users can retrieve historical alerts even if they were offline.

## Security & DevOps

### Authentication and Authorization

- JWT-based authentication for API access.
- bcrypt password hashing.
- OTP-based account recovery flows.
- Socket.IO authentication for real-time features.
- Admin and customer permission middleware.
- Project membership checks before sensitive project actions.
- File MIME/type checks and upload size limits for media safety.

### Payment Security

- Webhook verification patterns for payment callbacks.
- Finance records used as the source of truth for reconciliation.
- Payout validation before initiating transfer requests.
- No secrets stored in this public documentation repository.

### Deployment Strategy

The private project supported deployment through environment-based configuration and container-ready infrastructure:

- Dockerfile for containerized execution.
- Vercel configuration for serverless-style deployment compatibility.
- Environment variables for database URLs, JWT secrets, object storage credentials, email credentials, and payment provider keys.
- Scheduled jobs run inside the backend runtime for operational automation.

## Recruiter / Tech Lead Highlights

- Designed a complete marketplace backend from authentication to payouts.
- Built media-heavy cloud upload architecture using presigned URLs.
- Implemented real-time collaboration with authenticated Socket.IO rooms.
- Modeled complex creator/customer/project/finance relationships in MongoDB.
- Integrated checkout, payout, webhook reconciliation, and finance records.
- Balanced production requirements with a modular monolithic architecture.

## Repository Scope

This showcase repository is intended to communicate architecture and engineering approach only. It is suitable for technical recruiters, engineering managers, and tech leads evaluating backend system design experience.
