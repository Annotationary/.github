# 📑 Data Labeling Platform

We'll cover the following
+ [Project Overview & Goals](#🎯-project-overview-goals)
+ [Product Requirement](#)
+ [System Design]()
+ [API Contract]()
+ [Local Dev Setup Guide]()
+ [Database Schema]()
+ [CI/CD RunBook]()

## 🎯 Project Overview & Goals
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
| Frontend |
| Backend |
| ORM | 
| Database | 
| Object storge |
| Cache / queue |
| Auth |
| Container runtime |
| Orchestration |
| CI/CD |

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

Stores all registered accounts.

| Column | Type | Notes |
| --- | --- | -- |
| user_id | VARCHAR(255) PK | 
| username | VARCHAR(255) | Unique display name
| email | VARCHAR(255) | Unique, used as login indentifier
| password | VARCHAR(255) | bcrypt hash - never returned by API
| profile_image | VARCHAR(255) | URL to profile image, nullable
| cover_image | VARCHAR(255) | URL to cover image, nullable
| role | ENUM | Admin, Manager, Annotator, Reviewer
| specialization | VARCHAR(255) | Optional domain expertise tag
| user_status | ENUM | Active, Inactive
| created_at | DATETIME | Set on insert

**Project**

Top-level container for a labeling effort. Groups a dataset, assignemtns, guidelines, and members under one context.

| Column | Type | Notes |
| --- | --- | --- |
| project_id | VARCHAR(255) PK | 
| project_name | VARCHAR(255) | 
| description | VARCHAR(255) | Nullable
| project_status | ENUM | Active, Completed, Archived
| created_at | DATETIME | Set on insert
| updated_at | DATETIME | Updated via EF interceptor

**Project Members**

**Activity Logs**

**Datasets**

**Data_Items**

**Labels**

**Assignments**

**Guidelines**

**Tasks**

**Task_Items**

**Annotations**

**Annotation_Labels**

**Reviews**

### 3. Migrations
