# 📑 Data Labeling Platform

We'll cover the following
+ [Project Overview & Goals](#project-overview)
+ [Product Requirement](#project-requirement)
+ [System Design](#system-design)
+ [API Contract]()
+ [Database Design](#database-design)
+ [Local Dev Setup Guide](#local-dev-setup-guide)
+ [CI/CD RunBook](#cicd-runbook)

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

```shell
Browser (Next.js)
      │  HTTPS/REST + JWT
      ▼
ASP.NET Core API  ──►  PostgreSQL   (relational data)
      │           ──►  Object Store (images/files)
      │           ──►  Redis        (sessions, job queue)
      ▼
Background Worker (ASP.NET Core hosted service)
```

### 2. Tech stack
| Layer | Technology |
| --- | --- |
| Frontend | React Vite
| Backend | Springboot
| ORM | 
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
1. Manager uploads ZIP -> frontend requests presigned upload URL from API → browser streams ZIP to MinIO.
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
<!-- ![ERD](/profile/resources/Annotationary_ERD.png) -->

### 2. Tables
**Users**

Stores all registered accounts. `role` determines route access and what UI sections are shown. `specialization` is a free-text field for noting a labeler's domain expertise (e.g. medical imaging, traffic scenes).
 
| Column | Type | Notes |
|--------|------|-------|
| user_id | VARCHAR(255) PK | |
| username | VARCHAR(255) | Unique display name |
| email | VARCHAR(255) | Unique, used as login identifier |
| password | VARCHAR(255) | bcrypt hash — never returned by API |
| profile_image | VARCHAR(255) | URL to profile image; nullable |
| cover_image | VARCHAR(255) | URL to cover image; nullable |
| role | ENUM | `Admin`, `Labeler`, `Reviewer` |
| specialization | VARCHAR(255) | Optional domain expertise tag |
| user_status | ENUM | `Active`, `Inactive`, `Suspended` |
| created_at | DATETIME | Set on insert |

**Project**

Top-level container for a labeling effort. Groups a dataset, assignments, guidelines, and members under one context.
 
| Column | Type | Notes |
|--------|------|-------|
| project_id | VARCHAR(255) PK | |
| project_name | VARCHAR(255) | |
| description | VARCHAR(255) | Nullable |
| project_status | ENUM | `Active`, `Completed`, `Archived` |
| created_at | DATETIME | Set on insert |
| updated_at | DATETIME | Updated via EF interceptor |

**Project Members**

Join table linking users to the projects they belong to. A user can be a member of multiple projects.
 
| Column | Type | Notes |
|--------|------|-------|
| project_member_id | VARCHAR(255) PK | |
| project_id | VARCHAR(255) FK → projects.project_id | |
| user_id | VARCHAR(255) FK → users.user_id | |
 
**Index:** `(project_id, user_id)` — unique constraint; a user can only be added to a project once.

**Activity Logs**

Audit trail for all meaningful user actions in the system. Append-only — rows are never updated or deleted.
 
| Column | Type | Notes |
|--------|------|-------|
| log_id | VARCHAR(255) PK | |
| user_id | VARCHAR(255) FK → users.user_id | Who performed the action |
| activity_type | ENUM | e.g. `Create`, `Update`, `Delete`, `Login`, `Export` |
| entity_type | ENUM | e.g. `Project`, `Dataset`, `Task`, `Annotation` |
| entity_name | VARCHAR(255) | Human-readable name of the affected entity |
| entity_id | VARCHAR(255) | ID of the affected entity |
| action | VARCHAR(255) | Short action label, e.g. `status_changed` |
| description | VARCHAR(255) | Human-readable summary of what happened |
| old_value | VARCHAR(255) | Previous value, for update events; nullable |
| new_value | VARCHAR(255) | New value, for update events; nullable |
| metadata | VARCHAR(255) | Extra JSON context; nullable |
| ip_address | VARCHAR(255) | Client IP at time of action |
| timestamp | TIMESTAMP | Set on insert |
 
**Index:** `(user_id, timestamp DESC)` — user activity feed.  
**Index:** `(entity_type, entity_id, timestamp DESC)` — entity history queries.

**Datasets**

A batch of raw data items (images or files) attached to a project. Each project has exactly one dataset.
 
| Column | Type | Notes |
|--------|------|-------|
| dataset_id | VARCHAR(255) PK | |
| project_id | VARCHAR(255) FK → projects.project_id | One-to-one with project |
| dataset_name | VARCHAR(255) | |
| description | VARCHAR(255) | Nullable |
| total_items | INTEGER | Populated after upload processing completes |
| dataset_status | ENUM | `Uploading`, `Processing`, `Ready`, `Failed` |
| created_at | DATETIME | Set on insert |

**Data_Items**

Individual items within a dataset — one row per image or file. Created in bulk by the background worker when a dataset is processed.
 
| Column | Type | Notes |
|--------|------|-------|
| item_id | VARCHAR(255) PK | |
| dataset_id | VARCHAR(255) FK → datasets.dataset_id | Cascade delete |
| index | INTEGER | Position of the item in the original upload |
| filename | VARCHAR(255) | Original filename |
| url | VARCHAR(255) | MinIO presigned or permanent URL |
| file_format | ENUM | `JPEG`, `PNG`, `WEBP`, `CSV`, `TXT` |
| file_size | INTEGER | File size in bytes |
| width | INTEGER | Image width in pixels; nullable for non-image types |
| height | INTEGER | Image height in pixels; nullable for non-image types |
| data_type | ENUM | `Image`, `Text`, `Video` |
| item_status | ENUM | `Pending`, `InProgress`, `Completed` |
| uploaded_at | DATETIME | Set on insert |
 
**Index:** `(dataset_id, item_status)` — used by progress queries.

**Labels**

The set of label classes available for annotation within a dataset. Defined by the admin; shared across all assignments in the project.
 
| Column | Type | Notes |
|--------|------|-------|
| label_id | VARCHAR(255) PK | |
| dataset_id | VARCHAR(255) FK → datasets.dataset_id | Cascade delete |
| label_name | VARCHAR(255) | e.g. `car`, `pedestrian`, `positive` |
| color | VARCHAR(255) | Hex color for canvas rendering, e.g. `#FF5733` |
| description | VARCHAR(255) | Optional description of when to apply this label |
| label_status | ENUM | `Active`, `Deprecated` |
| created_at | DATETIME | Set on insert |

**Assignments**

An assignment scopes a subset of dataset items to a specific labeler, with an optional reviewer and due date. One project can have multiple assignments (e.g. splitting a dataset across multiple labelers).
 
| Column | Type | Notes |
|--------|------|-------|
| assignment_id | VARCHAR(255) PK | |
| project_id | VARCHAR(255) FK → projects.project_id | |
| dataset_id | VARCHAR(255) FK → datasets.dataset_id | |
| assignment_name | VARCHAR(255) | |
| assigned_to | VARCHAR(255) FK → users.user_id | The labeler |
| assigned_by | VARCHAR(255) FK → users.user_id | Admin who created the assignment |
| reviewed_by | VARCHAR(255) FK → users.user_id | Reviewer; nullable |
| total_items | INTEGER | Number of items in this assignment |
| completed_items | INTEGER | Running count, updated as tasks complete |
| description | VARCHAR(255) | Nullable |
| assignment_status | ENUM | `Pending`, `InProgress`, `InReview`, `Completed`, `Returned` |
| due_date | DATETIME | Nullable |
| created_at | DATETIME | Set on insert |
| updated_at | DATETIME | Updated via EF interceptor |

**Guidelines**

Annotation guidelines attached to a project. Versioned — a new row is created for each revision rather than updating in place.
 
| Column | Type | Notes |
|--------|------|-------|
| guideline_id | VARCHAR(255) PK | |
| project_id | VARCHAR(255) FK → projects.project_id | Cascade delete |
| title | VARCHAR(255) | |
| content | VARCHAR(255) | Markdown content of the guideline |
| version | VARCHAR(255) | Semantic version string, e.g. `1.0`, `1.1` |
| guideline_status | ENUM | `Draft`, `Active`, `Archived` |
| created_at | DATETIME | Set on insert |
| updated_at | DATETIME | Updated via EF interceptor |
 
Only one guideline per project should have `guideline_status = Active` at a time. Enforce this at the application layer.

**Tasks**

A task represents a unit of annotation work within an assignment. One assignment produces one or more tasks depending on how work is subdivided (e.g. by batch size).
 
| Column | Type | Notes |
|--------|------|-------|
| task_id | VARCHAR(255) PK | |
| assignment_id | VARCHAR(255) FK → assignments.assignment_id | Cascade delete |
| task_type | ENUM | `Classification`, `BoundingBox`, `Polygon`, `Segmentation` |
| completed_count | INTEGER | Number of task items completed |
| task_status | ENUM | `Pending`, `InProgress`, `InReview`, `Approved`, `Returned` |
| created_at | DATETIME | Set on insert |

**Task_Items**

Join table linking a task to the specific data items it covers. Each row is one item in one task.
 
| Column | Type | Notes |
|--------|------|-------|
| task_id | VARCHAR(255) FK → tasks.task_id | Composite PK |
| dataitem_id | VARCHAR(255) FK → data_items.item_id | Composite PK |
| item_index | INTEGER | Order of this item within the task |
| item_status | ENUM | `Pending`, `InProgress`, `Completed`, `Skipped` |
| assigned_at | DATETIME | When the task was created |
| completed_at | DATETIME | When the labeler marked this item done; nullable |
 
**Primary key:** `(task_id, dataitem_id)`

**Annotations**

Stores the label data submitted by a labeler for a specific data item within a task. One data item can have multiple annotations (e.g. multiple bounding boxes in one image).
 
| Column | Type | Notes |
|--------|------|-------|
| annotation_id | VARCHAR(255) PK | |
| task_id | VARCHAR(255) FK → tasks.task_id | |
| dataitem_id | VARCHAR(255) FK → data_items.item_id | |
| user_id | VARCHAR(255) FK → users.user_id | The labeler who created this annotation |
| confidence | ENUM | `Low`, `Medium`, `High` — labeler's self-assessed confidence |
| comment | VARCHAR(255) | Optional labeler note; nullable |
| annotation_type | ENUM | `Classification`, `BoundingBox`, `Polygon`, `Segmentation` |
| annotation_data | JSON | Shape geometry — see format below |
| annotation_status | ENUM | `Draft`, `Submitted`, `Approved`, `Rejected` |
| created_at | DATETIME | Set on insert |
| updated_at | DATETIME | Updated via EF interceptor |
 
**Index:** `(task_id, annotation_status)` — reviewer queue and task completion checks.  
**Index:** `(dataitem_id)` — fetch all annotations for a given item.
 
**`annotation_data` format by type:**
 
```json
// BoundingBox
{ "type": "bbox", "x": 120, "y": 80, "width": 200, "height": 150 }
 
// Polygon
{ "type": "polygon", "points": [[10,20],[30,40],[50,20]] }
 
// Segmentation mask (run-length encoded)
{ "type": "segmentation", "rle": "...", "width": 640, "height": 480 }
 
// Classification (no geometry — label applied via annotation_labels)
{ "type": "classification" }
```

**Annotation_Labels**

Join table linking an annotation to one or more label classes. Supports multi-label classification and composite annotations.
 
| Column | Type | Notes |
|--------|------|-------|
| annotation_id | VARCHAR(255) FK → annotations.annotation_id | Composite PK |
| label_id | VARCHAR(255) FK → labels.label_id | Composite PK |
 
**Primary key:** `(annotation_id, label_id)`

**Reviews**

Records a reviewer's decision on a specific annotation. One review per annotation.
 
| Column | Type | Notes |
|--------|------|-------|
| review_id | VARCHAR(255) PK | |
| annotation_id | VARCHAR(255) FK → annotations.annotation_id | One-to-one |
| reviewer_id | VARCHAR(255) FK → users.user_id | Must have `role = Reviewer` |
| review_status | ENUM | `Approved`, `Rejected`, `NeedsRevision` |
| evidence | VARCHAR(255) | URL or reference to support the decision; nullable |
| comment | VARCHAR(255) | Feedback to the labeler; required when `review_status = Rejected` |
| reviewed_at | VARCHAR(255) | Timestamp of the review decision |

### 3. Migrations
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

### 4. ENUM reference
| Table | Column | Values |
|-------|--------|--------|
| users | role | `Admin`, `Labeler`, `Reviewer` |
| users | user_status | `Active`, `Inactive`, `Suspended` |
| projects | project_status | `Active`, `Completed`, `Archived` |
| activity_logs | activity_type | `Create`, `Update`, `Delete`, `Login`, `Export` |
| activity_logs | entity_type | `Project`, `Dataset`, `Task`, `Annotation`, `Assignment`, `User` |
| datasets | dataset_status | `Uploading`, `Processing`, `Ready`, `Failed` |
| data_items | file_format | `JPEG`, `PNG`, `WEBP`, `CSV`, `TXT` |
| data_items | data_type | `Image`, `Text`, `Video` |
| data_items | item_status | `Pending`, `InProgress`, `Completed` |
| labels | label_status | `Active`, `Deprecated` |
| assignments | assignment_status | `Pending`, `InProgress`, `InReview`, `Completed`, `Returned` |
| guidelines | guideline_status | `Draft`, `Active`, `Archived` |
| tasks | task_type | `Classification`, `BoundingBox`, `Polygon`, `Segmentation` |
| tasks | task_status | `Pending`, `InProgress`, `InReview`, `Approved`, `Returned` |
| task_items | item_status | `Pending`, `InProgress`, `Completed`, `Skipped` |
| annotations | confidence | `Low`, `Medium`, `High` |
| annotations | annotation_type | `Classification`, `BoundingBox`, `Polygon`, `Segmentation` |
| annotations | annotation_status | `Draft`, `Submitted`, `Approved`, `Rejected` |
| reviews | review_status | `Approved`, `Rejected`, `NeedsRevision` |

### 5. Soft deletes
`users` and `projects` use status-based soft disabling (`user_status = Inactive` / `project_status = Archived`). All other entities use hard deletes with cascade on the parent FK. EF Core global query filters exclude inactive/archived rows by default — pass `.IgnoreQueryFilters()` in admin queries that need to see them.

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
mkdir data-labeling && cd data-labeling
 
# Clone both repos
git clone https://github.com/your-org/backend.git
git clone https://github.com/your-org/frontend.git
git clone https://github.com/your-org/infra.git
```

### 3. Start infrastructure services
The `infra` repo has a `docker-compose.yml` that starts PostgreSQL, Redis, and MinIO locally. This is the only Docker piece you need to run manually — the app services run natively for a faster dev loop.

```bash
cd infra
docker compose up -d
```

This starts:
- PostgreSQL on `localhost:5432`
- Redis on `localhost:6379`
- MinIO on `localhost:9000` (API) and `localhost:9001` (console UI)

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

## CI/CD Runbook
### 1. Repository and pipeline layout
Each repo has its own Jenkinsfile at root. Pipelines are independent — a frontend change only triggers the frontend pipeline.

| Repo | Jenkinsfile | What it builds |
|------|------------|----------------|
| `frontend` | `Jenkinsfile` | Next.js Docker image |
| `backend` | `Jenkinsfile` | ASP.NET Core API + Worker Docker image |
| `infra` | `Jenkinsfile` | Applies K8s manifests (no image build) |

### 2. Pipeline stages
 
#### 2.1 Frontend pipeline (`frontend/Jenkinsfile`)
 
```
Git checkout → Install deps → Lint → Build → Docker build → Docker push → Deploy to K8s
```
 
| Stage | Command | Failure behaviour |
|-------|---------|-------------------|
| Install deps | `npm ci` | Fail pipeline |
| Lint | `npm run lint` | Fail pipeline |
| Build | `npm run build` | Fail pipeline |
| Docker build | `docker build -t registry.yourdomain.com/frontend:$GIT_SHA .` | Fail pipeline |
| Docker push | `docker push ...` | Fail pipeline |
| Deploy | `kubectl set image deployment/frontend-deployment frontend=...` | Fail pipeline, trigger alert |

### 2.2 Backend pipeline (`backend/Jenkinsfile`)
 
```
Git checkout → Restore → Build → Test → Docker build → Docker push → Deploy to K8s
```

| Stage | Command | Failure behaviour |
|-------|---------|-------------------|
| Restore | `dotnet restore` | Fail pipeline |
| Build | `dotnet build --no-restore -c Release` | Fail pipeline |
| Test | `dotnet test --no-build` | Fail pipeline, PR blocked |
| Docker build | `docker build -t registry.yourdomain.com/backend:$GIT_SHA .` | Fail pipeline |
| Docker push | `docker push ...` | Fail pipeline |
| Deploy API | `kubectl set image deployment/backend-deployment ...` | Fail pipeline, trigger alert |
| Deploy Worker | `kubectl set image deployment/worker-deployment ...` | Fail pipeline, trigger alert |

---

### 3. Branch and environment mapping
 
| Branch | Environment | Auto-deploy? |
|--------|------------|--------------|
| `main` | Production | Yes, on merge |
| `staging` | Staging | Yes, on merge |
| `dev` | Dev | Yes, on merge |
| `feature/*` | — | CI only (no deploy) |
 
Feature branches run all stages up to Docker push but skip the deploy stage. This lets you confirm the image builds before merging.
 
---

### 4. Image tagging strategy
Images are tagged with the Git commit SHA:

```
registry.yourdomain.com/frontend:a3f9b2c
registry.yourdomain.com/backend:a3f9b2c
```

Additionally, the branch name is applied as a floating tag:
 
```
registry.yourdomain.com/frontend:main
registry.yourdomain.com/frontend:staging
```

K8s deployments always reference the SHA tag, never a floating tag. The SHA is recorded in the Jenkins build log for every deploy.

---

### 5. Kubernetes namespace layout
 
```
cluster
├── namespace: production
│   ├── frontend-deployment     (image: frontend:$SHA, replicas: 2)
│   ├── backend-deployment      (image: backend:$SHA, replicas: 2)
│   ├── worker-deployment       (image: backend:$SHA, replicas: 1)
│   └── ingress                 (Nginx → frontend + backend services)
├── namespace: staging
│   ├── frontend-deployment     (replicas: 1)
│   ├── backend-deployment      (replicas: 1)
│   └── worker-deployment       (replicas: 1)
├── namespace: dev
│   ├── frontend-deployment     (replicas: 1)
│   ├── backend-deployment      (replicas: 1)
│   └── worker-deployment       (replicas: 1)
└── namespace: infra            (shared across all envs)
    ├── postgresql-statefulset
    ├── redis-statefulset
    └── minio-statefulset
```

Each environment has its own set of `ConfigMaps` and `Secrets`. Never share secrets between namespaces.
 
---
 
### 6. Triggering a deploy manually
 
Sometimes you need to deploy without a code change (e.g. applying a config change or recovering after an incident).
 
**Option A — Re-run the Jenkins build**
 
Go to Jenkins → the relevant pipeline → click "Build Now". The pipeline will re-tag and re-apply the current `main` image.
 
**Option B — kubectl directly**
 
```bash
# Force a rolling restart (picks up new ConfigMap values)
kubectl rollout restart deployment/backend-deployment -n production
 
# Deploy a specific image tag
kubectl set image deployment/backend-deployment \
  backend=registry.yourdomain.com/backend:a3f9b2c \
  -n production
 
# Watch the rollout
kubectl rollout status deployment/backend-deployment -n production
```
 
---
 
### 7. Rollback steps
 
K8s keeps the previous ReplicaSet, so rollback is fast.
 
```bash
# Roll back to the previous revision
kubectl rollout undo deployment/backend-deployment -n production
 
# Roll back to a specific revision
kubectl rollout history deployment/backend-deployment -n production
kubectl rollout undo deployment/backend-deployment --to-revision=3 -n production
 
# Confirm rollback completed
kubectl rollout status deployment/backend-deployment -n production
```
 
After rolling back, identify the broken image SHA from the Jenkins build log and create a hotfix branch. Do not leave the deployment running on a rolled-back revision longer than necessary.
 
---
 
### 8. Health checks
 
All deployments have `readinessProbe` and `livenessProbe` configured in the K8s manifests (`infra/k8s/`).
 
- **Backend readiness:** `GET /health/ready` — returns 200 when the DB connection and Redis connection are alive.
- **Backend liveness:** `GET /health/live` — returns 200 always; if the process is deadlocked this fails.
- **Frontend readiness:** `GET /api/health` — Next.js built-in.
Check health manually:
 
```bash
kubectl get pods -n production
kubectl describe pod <pod-name> -n production
kubectl logs <pod-name> -n production --tail=100
```
 
---
 
## 9. Secrets management
 
Secrets are stored as Kubernetes Secrets and mounted as environment variables. They are not committed to any repo.
 
To update a secret:
 
```bash
kubectl create secret generic backend-secrets \
  --from-literal=Jwt__SecretKey=new-value \
  --namespace production \
  --dry-run=client -o yaml | kubectl apply -f -
 
# Restart the deployment to pick up the new secret
kubectl rollout restart deployment/backend-deployment -n production
```
 
Keep a copy of all secret values in a shared password manager (e.g. Bitwarden) — not in any file, not in Slack.
 
---
 
## 10. Adding a new environment variable
 
1. Add to `.env.example` in the relevant repo.
2. Add to the K8s `ConfigMap` or `Secret` in `infra/k8s/<env>/`.
3. Apply the ConfigMap: `kubectl apply -f infra/k8s/production/configmap.yaml`.
4. Restart the relevant deployment.
5. Update `local-dev-setup.md` with the new variable.