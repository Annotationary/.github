# Database details
## Tables
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

## ENUM reference
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

## Soft deletes
`users` and `projects` use status-based soft disabling (`user_status = Inactive` / `project_status = Archived`). All other entities use hard deletes with cascade on the parent FK. EF Core global query filters exclude inactive/archived rows by default — pass `.IgnoreQueryFilters()` in admin queries that need to see them.