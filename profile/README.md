# 📑 Data Labeling Platform

We'll cover the following
+ [🎯 Project Overview & Goals](#🎯-project-overview-goals)
+ [🧑‍💻 Team Roles & Responsibilities](#🧑‍💻-team-roles-responsibilities)
+ [ 🛠️ Technology Stack](#🛠️-technology-stack)
+ [🏗️ System Architecture](#🏗️-system-architecture)
+ [🏷️ Labeling Data Workflow](#🏷️-labeling-data-workflow)
+ [🎨 Database Design](#🎨-database-design)
+ [📚 Document References](#📚-document-references)

## 🎯 Project Overview & Goals
This project aims to build a Data Labeling Support System for training and evaluating machine learning models. The system supports multiple labeling tasks, such as identifying objects in images, drawing bounding boxes around objects, and segmenting object regions. Its goal is to manage the entire data labeling lifecycle, from project creation and task assignment to labeling, quality review, and data export, ensuring high-quality labeled datasets that improve the accuracy and reliability of machine learning models.

<img src="./Resources/logo.png" alt="logo">

## 🧑‍💻 Team Roles & Responsibilities
### Team Structure
| Role              | Name / Placeholder |
| ------------------|:------------------:|
| Front-End & BA    | Quốc Thái          |
| Front-End & BA    | Thế Anh            |
| Back-End          | Huy Vũ             |
| Back-End          | Kim Ngân           |
| Back-End & DevOps | Trương Minh Nhật   |

### Github Workflow
<img src="./resources/Github_Workflow.png" alt="Github workflow">

## 🛠️ Technology Stack
### Front-End
**Technologies**
+ React 19 - UI library
+ Vite - Build tool
+ React Router DOM - URL Mangement
+ Axios - Http Client for API calls
+ Shadcn - Components UI
+ Phosphoricons - Icons library

**Tools**
+ Visual Studio Code

### Back-End
**Technologies**
+ Spring Boot - Main Framework
+ BCrypt.NET - Securely hashing and Verifying passwords
+ JWT Authentication - Security
+ MicrosoftSQL - Database

**Tools**
+ IntelliJ
+ Microsoft SQL Management

### API Testing
**Tools**
+ Postman - API documentation

### DevOps
**Technologies**
+ Docker - Containerization
+ Kubernetes - Container Orchestration
+ Traefik - Reverse proxy
+ NGINX - Load Balancer 
+ SonarQube - Code quality & Security reviews
+ Trivy - Vulnerability scanner for containers
+ Jenkins - Continuous Integration
+ ArgoCD - Continuous Delivery

## 🏗️ System Architecture
```
[ Frontend (React) ]
   ↓ REST API calls
[ Backend (Spring Boot) ]
   ↓
[ Database (MSSQL) ]
   ↓
[ File Storage (Local / S3 / MinIO) ]
```

## 🏷️ Labeling Data Workflow
1. Project Manager creates a labeling project and uploads raw data (images, text, audio, or video).

2. The system validates the dataset, assigns task IDs, and distributes labeling tasks to annotators.

3. Annotators perform labeling tasks (e.g., classification, bounding boxes, or segmentation) following defined guidelines.

4. Reviewer checks label quality and approves, requests revisions, or rejects the annotations.

5. The system updates labeling status, tracks progress, and exports the verified labeled data for training and evaluating AI models.

## 🎨 Database Design
Our system will have total none Entities:

## 📚 Document References
### DevOps & Deployment
+ [🚀 CI/CD Pipeline](CICD.md)
### Project Documentation
+ [⚙️ Project Main Flow](MAIN_FLOW.md)
+ [📰 Features](FEATURE.MD)
+ [🗄️ Database Overview](DATABASE.md)


