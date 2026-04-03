# Design, Implementation, and Deployment of a Spotify API + AWS Cloud Music Rating Application

> This document is the Cloud Computing phase write-up covering architecture, Docker containerization, AWS deployment, and troubleshooting.
>
> - How to run the app (Docker quick start): see README.md
> - Semester 1 team-project documentation + screenshots: see README_CITS3403_PROJECT.md

## Authors 

E. Lin, R. Wang, H. Guan, X. Sun (15%), R. Lu

> Note: Student IDs were provided in chat, but GitHub repos are often public.
> Consider omitting student IDs from the public repo and keeping a “submission version” in a private repo or a private branch.

## Abstract

With the rapid adoption of music streaming services, users increasingly demand deeper, more personalized interaction and analysis of their listening data. This report studies Rankd, a music rating web application built on the Spotify Web API. The system uses Flask as the backend framework and integrates Spotify API endpoints to retrieve music metadata and user listening signals. It supports core features such as music search and rating, analytics and visualization, and friend-based comparison. For reliability and portability, the application is containerized with Docker and deployed to AWS using a scalable cloud architecture.

This document summarizes the core business logic and system architecture, provides an end-to-end containerization and deployment workflow, and highlights common deployment issues together with debugging and solutions. The results indicate that Rankd is practical and extensible in data integration, personalized analytics, and cloud deployment architecture, and that its containerization and AWS deployment experience can serve as a reference for similar music-focused web applications.

Keywords: music rating application; Spotify API; Flask; Docker; AWS deployment; data visualization

---

## 1. Introduction

### 1.1 Background and Motivation

In the digital era, streaming platforms have become the primary channel for music consumption. Spotify, as a leading streaming provider, offers a large catalog and strong personalization, but its mainstream user-facing features focus on playback and recommendations. Many users want to actively:

- Rate music and record their own preferences
- Understand taste patterns (genres, decades, durations, etc.)
- Share and compare insights with friends

Rankd addresses this gap by integrating the Spotify Web API with a custom rating system to form a complete loop: search → rating → analytics → visualization → sharing. From an engineering perspective, Docker ensures environment consistency, and AWS provides a secure, highly available, scalable runtime for multi-user access.

### 1.2 Related Work

- Last.fm provides listening history tracking and basic statistics but does not focus on explicit user rating.
- Rate Your Music supports ratings and reviews but lacks deep integration with Spotify and requires manual entry.
- Some domestic music apps emphasize social comments and light statistics but do not provide structured, multi-dimensional analytics.

Rankd’s key contributions:

1. Deep Spotify API integration for automated data retrieval
2. Multi-dimensional analytics (correlation, extremes, outliers) with clear visualization
3. Docker + AWS deployment architecture for portability, availability, and elastic scaling

### 1.3 Structure

This report covers requirements and design, implementation highlights, Docker containerization, AWS deployment, and common troubleshooting patterns.

---

## 2. Requirements and High-Level Design

### 2.1 Functional Requirements

- User management: registration/login, profile editing, password/email updates, Spotify OAuth authorization
- Rating: search tracks/albums/artists via Spotify, rate on a 0–10 scale (including decimals), create/update/delete ratings
- Analytics: compute a “listening score” from Spotify listening signals and compare it with user ratings; visualize results
- Social: friend requests, accept/reject, compare ratings and analytics with friends
- Accessibility & security: HTTPS transport, custom domain access, cross-device usability

### 2.2 Non-Functional Requirements

- Performance: responsive page loads, fast search, analytics within a reasonable time window
- Compatibility: major browsers and mobile-friendly layouts
- Security: hashed passwords, least-privilege access, basic defenses against common web vulnerabilities
- Scalability: modular codebase and infrastructure that supports horizontal scaling

### 2.3 Overall Architecture

- Presentation layer: HTML/CSS/JS (charts, responsive UI)
- Business logic: Flask blueprints (auth, music data, rating management, analytics, friends)
- Data access: SQLAlchemy ORM; local SQLite for development; shared Postgres (RDS) for production
- Infrastructure: Spotify API integration, Docker, AWS services (VPC/ECS/ALB/ECR/RDS/ACM/Route 53), logging/monitoring

#### Production evolution: SQLite → RDS Postgres

In ECS/Fargate, multiple running tasks behind a load balancer make container-local SQLite unsuitable (each task has an isolated database). Migrating to Amazon RDS (PostgreSQL) centralizes persistence so all tasks share the same database via a single `DATABASE_URL`, which enables true horizontal scaling and consistent behavior.

---

## 3. Technology Stack and Implementation Highlights

### 3.1 Tech Stack

- Backend: Flask, SQLAlchemy, Flask-Login, Werkzeug
- Analytics: NumPy, SciPy, scikit-learn
- API client: Requests
- Frontend: HTML/CSS/JS (optional Bootstrap), charts (e.g., ECharts)
- Deployment: Docker; AWS (ECR, ECS Fargate, VPC, ALB, ACM, Route 53, RDS)

### 3.2 Key Features

- User authentication + Spotify OAuth2 authorization
- Search and rating CRUD for tracks/albums/artists
- Listening-history ingestion + listening-score computation + correlation/outlier analytics + visualization
- Friend relationship management and comparison views

---

## 4. Docker Containerization

### 4.1 Goals

Containerization ensures consistent runtime environments between local development and cloud deployment. The goal is portability, reproducibility, and simpler deployment.

### 4.2 Workflow

- Dockerfile: choose a slim Python base image, install dependencies, copy code, run via Gunicorn
- Local build/run: `docker build` and `docker run` to validate functionality
- Image distribution: optionally push to Docker Hub; push to Amazon ECR for AWS deployment

### 4.3 Benefits

- Environment consistency, simplified deploys, isolation, portability, and versioned releases

---

## 5. AWS Cloud Deployment

### 5.1 Architecture

- Custom VPC across availability zones (public + private subnets)
- ECS Fargate runs the container tasks
- Application Load Balancer (ALB) provides public access and health checks
- Security groups control ingress/egress
- NAT Gateway allows private tasks to reach the internet (e.g., image pulls, Spotify API)
- ACM provides TLS certificates for HTTPS
- Route 53 manages DNS and points the domain to the ALB
- RDS PostgreSQL provides shared persistence for production

### 5.2 Deployment Flow (Mermaid)

```mermaid
flowchart TD
  A[Local development] --> B[Write Dockerfile]
  B --> C[Build Docker image]
  C --> D1[Push to Docker Hub (optional)]
  C --> D2[Push to Amazon ECR]
  D2 --> E[Provision AWS infrastructure]
  E --> F[Create VPC (public/private subnets)]
  F --> G[Configure IGW and NAT Gateway]
  G --> H[Create security groups (ALB SG / ECS Task SG)]
  H --> I[Create ECS Fargate cluster and service]
  I --> J[Deploy ECS tasks (private subnet)]
  J --> K[Configure ALB (public subnet)]
  K --> L[Request TLS cert in ACM]
  L --> M[Configure DNS in Route 53]
  M --> N[Users access via custom domain over HTTPS]
```

---

## 6. Common Deployment Issues and Debugging

### 6.1 Port mismatch causing 503/504

- Symptom: target group health checks fail; users see 503/504
- Root cause: target group port does not match the container listening port
- Fix: align target group port with the application port (e.g., 5000), update rules, redeploy

### 6.2 Security group rules causing timeouts

- Symptom: timeouts even after port corrections
- Root cause: ECS task SG does not allow inbound traffic from ALB SG on the app port
- Fix: allow ALB SG → ECS Task SG on port 5000

### 6.3 Wrong image/task definition

- Symptom: unexpected pages or tasks failing to start
- Root cause: task definition points to the wrong image
- Fix: update to the correct ECR image and force a new deployment

### 6.4 Database initialization errors causing 500

- Symptom: 500 errors; logs indicate missing tables
- Root cause: initialization code only runs under `if __name__ == "__main__":`, which Gunicorn does not execute
- Fix: ensure schema creation/migrations run during startup (prefer Alembic migrations for production)

---

## 7. Conclusion and Future Work

### 7.1 Summary

- Implemented an end-to-end pipeline from Spotify-based features to cloud deployment
- Built reusable troubleshooting knowledge for AWS deployments (ports, SGs, images, DB, networking)

### 7.2 Future Improvements

- Features: recommendation system, playlist sharing, exportable listening reports
- Ops: autoscaling policies, end-to-end monitoring/alerting (CloudWatch), structured logging
- Security: WAF, stricter IAM least privilege, stronger network isolation, encryption at rest and in transit

---

## References

1. AWS ECS Fargate Developer Guide: <https://docs.aws.amazon.com/ecs/latest/developerguide/AWS_Fargate.html>
2. AWS VPC Best Practices: <https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Best_Practices.html>
3. Spotify Web API Documentation: <https://developer.spotify.com/documentation/web-api>
4. Amazon ECR User Guide: <https://docs.aws.amazon.com/ecr/latest/userguide/what-is-ecr.html>
5. Namecheap Support Docs: <https://www.namecheap.com/support/>

---

## Figures (To Be Added)

Two images were shared in chat:

1. App UI screenshot (Scores page)
2. AWS deployment flowchart

They are not yet present in the repository, so this section is a placeholder.

- Suggested path: `docs/images/ui-screenshot.png`
- Suggested path: `docs/images/aws-deployment-flow.png`

Once the image files are added, you can link them here.
