# EV Charger Installation Platform

[![Go](https://img.shields.io/badge/Go-00ADD8?style=flat&logo=go&logoColor=white)](https://go.dev)
[![React](https://img.shields.io/badge/React-20232A?style=flat&logo=react&logoColor=61DAFB)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=flat&logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![React Native](https://img.shields.io/badge/React_Native-20232A?style=flat&logo=react&logoColor=61DAFB)](https://reactnative.dev)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=flat&logo=postgresql&logoColor=white)](https://www.postgresql.org)

> Source code is private. This repository documents the system architecture and engineering decisions.

---

## Overview

Full-stack platform for managing end-to-end EV charger installation workflows across field operations in Malaysia. Connects field installers with admin coordinators through a web dashboard and mobile app, tracking every stage from job creation to final approval.

The system handles installer dispatching via proximity-based search, structured site surveys, delivery order signing, document management, and push notifications — all coordinated through a single Go REST API.

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
│                  PostgreSQL 14+ · Redis                       │
└──────┬──────────────────────────────────────┬───────────────┘
       │ HTTPS REST                            │ HTTPS REST
       │                                       │
┌──────▼──────────────────┐       ┌────────────▼──────────────┐
│   React Native App      │       │   AWS S3 / Cloudflare R2  │
│   Expo SDK 54           │       │   (or local filesystem)   │
│   Installer mobile app  │       │                           │
└─────────────────────────┘       └───────────────────────────┘
```

Field installers use the mobile app to accept jobs, submit site surveys, upload documents, and capture customer signatures. Admin coordinators use the web dashboard to create jobs, assign installers, review survey data, and approve completed work.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | Go / Gin / GORM |
| Database | PostgreSQL 14+ |
| Cache | Redis |
| Auth | JWT + refresh token rotation · RBAC |
| Frontend | React 18 + TypeScript / Vite 5 |
| Frontend State | Zustand (persisted) |
| Frontend UI | shadcn/ui + Tailwind CSS |
| Mobile | React Native + Expo SDK 54 |
| Mobile Navigation | React Navigation v6 |
| File Storage | AWS S3 / Cloudflare R2 (pluggable) |
| Geocoding | Nominatim · LocationIQ · Geoapify (pluggable) |
| Push Notifications | Firebase Cloud Messaging (FCM) |

---

## Key Features

**Installation lifecycle management**
- Eight-stage state machine from `pending_assignment` through `approved`
- Every transition recorded append-only in an audit log with actor, timestamp, and prior state
- Admin and installer roles enforce separate permitted transitions at the API level

**Proximity-based installer dispatch**
- Admin triggers a radius search against job coordinates
- Backend queries installers within a configurable radius, returning results ranked by distance
- Selected installer receives an FCM push notification on their mobile device and can accept or reject from the app

**Delivery order workflow**
- Admin populates the delivery order; balance load auto-calculates per phase from rating and measured full load
- Installer presents the DO to the customer on the mobile app
- Customer signs in-app via a canvas signature pad; signature is stored and linked to the installation record

**Document management**
- Documents categorised by workflow stage (survey, installation, handover)
- Files stored on S3/R2 in production; backend returns pre-signed URLs for direct client access
- Local filesystem fallback for development with no config changes required

**In-app notifications**
- FCM push notifications delivered to the installer's device on job assignment
- Web dashboard shows in-app notification feed with per-notification read state

---

## Engineering Highlights

**Pluggable geocoding provider**

Geocoding is abstracted behind a provider interface with concrete implementations for Nominatim, LocationIQ, and Geoapify. Switching providers is a single environment variable change with no application code touched. This also allows Nominatim (free, no key) in development and a commercial provider in production without separate build configurations.

**Pluggable file storage**

The storage layer uses the same abstraction pattern — an interface with S3/R2 and local filesystem implementations. Development runs entirely without cloud credentials. The same pre-signed URL contract is maintained across both backends, so the frontend code is identical in both environments.

**JWT refresh token rotation with individual revocation**

Access tokens are short-lived JWTs. Refresh tokens are stored in the database and can be revoked individually — logging out one device does not invalidate sessions on other devices. The mobile app and web dashboard each hold independent refresh tokens, so an admin revoking a field installer's session does not affect the coordinator's session.

**State machine integrity via audit log**

Status transitions are persisted as append-only audit log entries, not overwritten fields. Every transition records the acting user, timestamp, previous state, and new state. This gives a complete immutable history for dispute resolution and reporting without any additional event-sourcing infrastructure.

**Redis-backed rate limiting on sensitive routes**

Auth and geocoding endpoints are rate-limited per IP using a Redis sliding window counter. Limits are applied at the middleware layer before any handler logic runs, keeping the rate limit logic fully decoupled from business logic and trivially testable.

**Canvas signature capture on mobile**

The delivery order customer signature is captured in a React Native canvas component, encoded, and uploaded alongside the DO record. The backend links the signature blob to the installation ID and document type, treating it identically to any other uploaded document — no bespoke signature storage path required.

---

## Project Status

**Production** — deployed and serving live field operations in Malaysia.
