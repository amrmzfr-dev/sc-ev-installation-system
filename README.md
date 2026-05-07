# EV Charger Installation Platform

[![Go](https://img.shields.io/badge/Go-00ADD8?style=flat&logo=go&logoColor=white)](https://go.dev)
[![React](https://img.shields.io/badge/React-20232A?style=flat&logo=react&logoColor=61DAFB)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=flat&logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=flat&logo=postgresql&logoColor=white)](https://www.postgresql.org)

> Source code is private. This repository documents the system architecture and engineering decisions.

---

## Overview

Full-stack platform for managing end-to-end EV charger installation workflows across field operations in Malaysia. Connects field installers with admin coordinators through a web dashboard, tracking every stage from job creation to final approval.

The system handles installer dispatching via proximity-based search, structured site surveys, delivery order signing, document management, and notifications — all coordinated through a single Go REST API.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│             React Web Dashboard (Vite + TypeScript)          │
│                  Admin coordinator interface                  │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTPS / REST
┌───────────────────────────▼─────────────────────────────────┐
│                  Go REST API (Gin + GORM)                     │
│                  PostgreSQL · Redis                           │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTPS REST
                            │
                ┌───────────▼───────────┐
                │  Cloud / Local Storage │
                └───────────────────────┘
```

Admin coordinators use the web dashboard to create jobs, assign installers, review survey data, and approve completed work.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | Go / Gin / GORM |
| Database | PostgreSQL |
| Cache | Redis |
| Auth | JWT + refresh token rotation · RBAC |
| Frontend | React + TypeScript / Vite |
| Frontend State | Zustand (persisted) |
| Frontend UI | shadcn/ui + Tailwind CSS |
| File Storage | S3-compatible object storage (pluggable) |
| Geocoding | Multiple providers supported (pluggable) |

---

## Key Features

**Installation lifecycle management**
- Multi-stage state machine covering the full workflow from initial assignment through admin approval
- Every transition recorded append-only in an audit log with actor, timestamp, and prior state
- Admin and installer roles enforce separate permitted transitions at the API level

**Proximity-based installer dispatch**
- Admin triggers a radius search against job coordinates
- Backend queries installers within a configurable radius, returning results ranked by distance
- Selected installer is notified and assigned to the job

**Delivery order workflow**
- Admin populates the delivery order; balance load auto-calculates per phase from rating and measured full load
- Customer signature captured and linked to the installation record

**Document management**
- Documents categorised by workflow stage (survey, installation, handover)
- Cloud storage in production with a local filesystem fallback for development
- Uniform file access contract across both environments

**Notifications**
- Installers notified on job assignment
- Web dashboard shows an in-app notification feed with per-notification read state

---

## Engineering Highlights

**Pluggable geocoding and storage**

Both the geocoding and file storage layers are abstracted behind provider interfaces. Concrete implementations can be swapped via configuration with no application code changes — useful for keeping development lightweight while running a different provider in production.

**Token-based auth with per-session revocation**

Access tokens are short-lived. Refresh tokens are managed independently per client, so revoking one session does not affect other active sessions.

**State machine integrity via audit log**

Status transitions are persisted as append-only audit log entries rather than overwritten fields. Every transition records the actor, timestamp, and previous state — giving a complete immutable history for reporting and dispute resolution without additional event-sourcing infrastructure.

**Rate limiting decoupled from business logic**

Sensitive endpoints are rate-limited per IP at the middleware layer, before any handler logic runs. This keeps rate limiting independently testable and ensures consistent enforcement regardless of how routes are called.

---

## Project Status

**Production** — deployed and serving live field operations in Malaysia.
