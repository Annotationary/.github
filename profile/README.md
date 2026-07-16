# 📑 Data Labeling Platform

We'll cover the following
+ [Project Overview & Goals](#project-overview)
+ [Product Requirement](#project-requirement)
+ [System Design](#system-design)
+ [API Contract](#5-api-contract)
+ [Database Design](#database-design)
+ [Local Dev Setup Guide](#local-dev-setup-guide)
+ [CI/CD RunBook](./CICD-RUNBOOK.md)

## Project Overview
This project aims to build a Data Labeling Support System for training and evaluating machine learning models. The system supports multiple labeling tasks, such as identifying objects in images, drawing bounding boxes around objects, and segmenting object regions. Its goal is to manage the entire data labeling lifecycle, from project creation and task assignment to labeling, quality review, and data export, ensuring high-quality labeled datasets that improve the accuracy and reliability of machine learning models.

## Project Requirement
### 1. Problem statement
Machine learning teams need labeled training data. Existing tools (Scale AI, Label Studio) are either expensive or complex to self-host with custom workflows. This project provides a lightweight, self-hosted data labeling platform where a small team can upload datasets, assign annotation tasks, review quality, and export labeled data in standard formats.

### 2. User roles
**Admin**

Manage user creation, system analystics, monitoring system resourse avaliability, tracing activity logs.

**Manager**

Creates and manage the labeling projects. Responsible for uploading datasets, configuring label schemas (classes, attributes), assigning tasks to labelers, and triggering exports. 

**Annotator**

Receives assigned tasks and annotates items — drawing bounding boxes, polygons, or applying class labels depending on the project type. Submits completed tasks for review.

**Reviewer**

Inspects submitted annotations for quality. Approves correct annotations or rejects them with written feedback that is sent back to the labeler. Has read access to all tasks in a project.

![context-diagram](./resources/Context-diagram.drawio.svg)

### 3. Core features
#### 3.1 Project management
+ Manager can create a project with a name, description, an annotation type (classification / bounding box / polygon / segmentation).
+ Manager can define a label schema: a list of classes with optional attributes (e.g `car` with attribute `occluded: bool`).
+ Manager can archive or delete a project.

#### 3.2 Dataset upload
+ Manager uploads a ZIP of images or a CSV of text items.
+ System auto-splits the dataset into individual tasks (one item = one task).
+ Supported image formats: JPEG, PNG, WEBP. Max item size: 10 MB.

#### 3.3 Task assignment
+ Manager assigns tasks to specific labelers, or uses round-robin auto-assignment.
+ A task can only be assigned to one labeler at a time.
+ Manager can reassign a task if the labeler is unavailable.

#### 3.4 Annotation Interface
+ Annotator sees their task queue sorted by due date.
+ Canvas supports pan, zoom, and keyboard shortcuts for common actions.
+ Annotator can save progress (draft) and submit when complete.
+ Submitted task status changes to "In Review".

#### 3.5 Review workflow
+ Reviewer sees all tasks with status "In Review".
+ Reviewer can approve (status → Approved) or reject (status → Returned) with a comment.
+ Rejected tasks reappear in the labeler's queue with the reviewer's feedback visible.

#### 3.6 Export
+ Manager exports all Approved annotations for a project.
+ Supported formats: COCO JSON, flat CSV, raw JSON.
+ Export is scoped to a project; partial exports (by date range) planned for v2.

#### 3.7 Progress dashboard
+ Per-project stats: total tasks, pending, in progress, in review, approved, returned.
+ Per-labeler stats: tasks completed, average time per task, rejection rate.

### 4. Acceptance criteria
| Feature | Criteria |
| --- | --- |
| Dataset upload | ZIP of 1000 images processes within 30 seconds; each image become on task |
| Task assignment | Round-robin distributes tasks evenly; no task assigned to two labelers simultaneously
| Annotation save | Draft saves within 2 seconds; no data loss on browser refresh
| Review | Approving or rejecting a task updates its status within 1 second
| Export | COCO JSON export for 10,000 annotations completes within 10 seconds
| Auth | Unauthenticated requests to any API endpoint return 401

### 5. Out of scoped for this document 
API endpoint specs -> see Swagger at `/swagger`

## System Design
### 1. Overview
The system is a web application with a decoupled frontend and backend, deployed on Kubernetes. Users interact through a Next.js browser app. The Next.js app communicates exclusively with the ASP.NET Core REST API. The API handles business logic and persists data to PostgreSQL, stores raw assets in object storage, and uses Redis for session caching and background job queuing.

![System-design-diagram](./resources/System-design-diagram.drawio.svg)

### 2. Tech stack
| Layer | Technology |
| --- | --- |
| Frontend | React Vite
| Backend | ASAP .NET Core
| Database | MySQL
| Object storage | SeaweedFS
| Auth | JWT
| Container runtime | Docker, Docker compose
| CI/CD | Jenkins

### 3. Services
#### 3.1 Frontend
Runs as a stateless deployment in K8s. Fetches all data from the backend API at runtime; no database access. The annotation canvas is a client-side React component (no SSR needed for that view).

Key responsibilities:

Authentication pages (login, token refresh).
Admin pages: project creation, dataset upload, task assignment, export trigger.
Labeler pages: task queue, annotation canvas.
Reviewer pages: review queue, approve/reject UI.
Progress dashboard.

Environment variables: `NEXT_PUBLIC_API_URL`, `NEXTAUTH_SECRET`.

#### 3.2 Backend
Runs as a stateless deployment in K8s (multiple replicas behind a ClusterIP service). All state lives in PostgreSQL, MinIO, or Redis.

Key responsibilities:

JWT issuance and validation.
All CRUD operations for projects, datasets, tasks, annotations.
Presigned URL generation for direct browser-to-MinIO image uploads.
Triggering background jobs (dataset splitting, export generation).
Swagger UI served at `/swagger` in non-production environments.

Controllers: `AuthController`, `ProjectsController`, `DatasetsController`, `TasksController`, `AnnotationsController`, `ExportsController`, `UsersController`.

#### 3.3 Background worker
Runs as a separate Deployment in K8s (single replica). Shares the same Docker image as the API but starts with `--worker` flag.

Key responsibilities:

Dataset processing: unzip uploads, validate images, create Task rows in bulk.
Export generation: query approved annotations, serialize to COCO/CSV/JSON, upload to MinIO, write download URL to DB.

Picks up jobs from a Redis list. If a job fails, it retries up to 3 times with exponential backoff, then writes to a `failed_jobs` table.

### 4. Data flow - annotation lifecycle
1. Manager uploads ZIP -> frontend requests presigned upload URL from API → browser streams ZIP to SeaweedFS.
2. API writes `Dataset` row, pushes a `process_dataset` job to Redis queue.
3. Worker picks up job, extracts ZIP, creates one `Task` row per image, updates `Dataset.status = Ready`.
4. Admin assigns tasks (API writes `Task.assignee_id`).
5. Labeler loads task → API returns presigned GET URL for the image → browser loads image directly from MinIO.
6. Labeler submits → API writes `Annotation` rows, sets `Task.status = InReview`.
7. Reviewer approves → API sets `Task.status = Approved`.
8. Admin triggers export → API pushes `generate_export` job → Worker serializes, uploads file to MinIO, writes presigned download URL to `Export` row.

### 5. API contract
The full spec is auto-generated by Swashbuckle from controller XML comments. Run the backend locally and visit:
```
http://localhost:8080/swagger
```
Conventions:
+ Base path: `/api/v1/`
+ All responses: `application/json`
+ Error shape: `{ "error": "string", "details": {} }`
+ Pagination: `?page=1&pageSize=20, response includes { "items": [], "total": int }`
+ Timestamps: ISO 8601 UTC (2026-06-24T10:00:00Z)

### 6. Infrastructure layout
```
K8s cluster
├── namespace: dev
│   ├── frontend-deployment      (1 replica)
│   ├── backend-deployment       (1 replica)
│   └── worker-deployment        (1 replica)
├── namespace: staging
│   ├── frontend-deployment      (1 replica)
│   ├── backend-deployment       (1 replica)
│   └── worker-deployment        (1 replica)
└── namespace: production
    ├── frontend-deployment      (2 replicas, HPA max 5)
    ├── backend-deployment       (2 replicas, HPA max 5)
    └── worker-deployment        (1 replica)

Shared services (namespace: infra)
    ├── postgresql-statefulset
    ├── redis-statefulset
    └── minio-statefulset
```
Ingress: Nginx Ingress Controller routes `app.yourdomain.com` → frontend, `api.yourdomain.com` → backend.

### 7. Non-functional requirements
| Concern | Target |
| --- | --- |
| API response time (p95) | < 300 ms for read endpoints |
| Dataset processing | 1000-image ZIP < 60 seconds |
| Availability | 99.5% uptime (dev team, no on-call SLA) |
| Image storage | Up to 50 GB per project |
| Max concurrent labelers | 50 (v1)

## Database Design
### 1. Entities overview
![ERD](/profile/resources/Annotationary_ERD.png)

### 2. Migrations
EF Core migrations live in `/backend/Migrations/`. To apply:
 
```bash
# from the backend project root
dotnet ef database update
```
 
To create a new migration after changing a model:
 
```bash
dotnet ef migrations add <MigrationName>
```
 
Never edit a migration file after it has been applied to any shared environment. Write a new migration instead.

### 3. Additional database documents
[-> More about database configurations](./DATABASE.md)

## Local Dev Setup Guide
Get the full stack running locally in ~20 minutes. You need Docker Desktop (or Docker Engine + Compose), Node.js 20+, and .NET 8 SDK.

### 1. Prerequisites

| Tool | Version | Check |
|------|---------|-------|
| Docker Desktop | Latest | `docker --version` |
| Docker Compose | v2+ | `docker compose version` |
| Node.js | 20+ | `node --version` |
| .NET SDK | 8.0 | `dotnet --version` |
| Git | Any | `git --version` |

### 2. Clone the repos
```bash
# Create a workspace folder
mkdir annotationary && cd annotationary
 
# Clone both repos
git clone https://github.com/Annotationary/NET-BackEnd.git
git clone https://github.com/Annotationary/Spring_FrontEnd.git
git clone https://github.com/Annotationary/Infrastructure.git
```

### 3. Start infrastructure services
The `infra` repo has a `docker-compose.yml` that starts PostgreSQL, Redis, and MinIO locally. This is the only Docker piece you need to run manually — the app services run natively for a faster dev loop.

```bash
cd infra
docker compose up -d
```

This starts:
- MySQL on `localhost:3307`
- SeaweedFS on `localhost:9000` (master) and `localhost:9001` (filer)

Verify everything is up:
 
```bash
docker compose ps
```
 
All three services should show `running`.
 
**MinIO first-time setup:** Open `http://localhost:9001`, log in with `minioadmin / minioadmin`, create a bucket named `data-labeling`.

### 4. Configure the backend
```bash
cd ../backend
cp .env.example .env
```
 
Open `.env` and fill in:
 
```env
# Database
ConnectionStrings__DefaultConnection=Host=localhost;Port=5432;Database=datalabeling;Username=postgres;Password=postgres
 
# Redis
Redis__ConnectionString=localhost:6379
 
# MinIO / Object Storage
Storage__Endpoint=localhost:9000
Storage__AccessKey=minioadmin
Storage__SecretKey=minioadmin
Storage__BucketName=data-labeling
Storage__UseSSL=false
 
# JWT
Jwt__SecretKey=your-local-dev-secret-min-32-chars-long
Jwt__Issuer=data-labeling-api
Jwt__Audience=data-labeling-frontend
Jwt__AccessTokenExpiryMinutes=15
Jwt__RefreshTokenExpiryDays=7
 
# Environment
ASPNETCORE_ENVIRONMENT=Development
```
 
Run database migrations:
 
```bash
dotnet ef database update
```
 
Seed the database with a default admin user and sample data:
 
```bash
dotnet run --project src/DataLabeling.Api -- --seed
```
 
Default seeded admin credentials:
- Email: `admin@local.dev`
- Password: `Admin1234!`
Start the API:
 
```bash
dotnet run --project src/DataLabeling.Api
```
 
API is available at `http://localhost:5000`. Swagger UI at `http://localhost:5000/swagger`.
 
Start the background worker (in a separate terminal):
 
```bash
dotnet run --project src/DataLabeling.Api -- --worker
```

### 5. Configure the frontend
```bash
cd ../frontend
cp .env.local.example .env.local
```
 
Open `.env.local` and fill in:
 
```env
NEXT_PUBLIC_API_URL=http://localhost:5000/api/v1
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=any-local-secret-string
```
 
Install dependencies and start:
 
```bash
npm install
npm run dev
```
 
Frontend is available at `http://localhost:3000`.

### 6. Verify the setup
 
Open `http://localhost:3000` and log in with `admin@local.dev` / `Admin1234!`.
 
You should see the admin dashboard. Create a project, upload the sample ZIP from `infra/seed-data/sample-images.zip`, and verify tasks appear after ~10 seconds (the worker processes the upload).
 
---
 
### 7. Stopping everything
 
```bash
# Stop the Next.js dev server: Ctrl+C in its terminal
# Stop the .NET API: Ctrl+C in its terminal
# Stop the background worker: Ctrl+C in its terminal
 
# Stop Docker services
cd infra
docker compose down
```
 
To wipe the database and start fresh:
 
```bash
docker compose down -v   # -v removes volumes
docker compose up -d
cd ../backend && dotnet ef database update && dotnet run --project src/DataLabeling.Api -- --seed
```

