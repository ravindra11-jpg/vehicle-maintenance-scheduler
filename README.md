# Campus Notifications and Vehicle Maintenance Backend

A Node.js backend project with three parts: a reusable **logging middleware**, a **campus notification service** with a priority inbox, and a **vehicle maintenance scheduler**. The repo also includes a written system design for the notification platform.

## Table of contents

- [Overview](#overview)
- [Repository structure](#repository-structure)
- [Modules](#modules)
- [Notification system design](#notification-system-design)
- [Getting started](#getting-started)
- [API reference](#api-reference)
- [Design decisions and trade-offs](#design-decisions-and-trade-offs)
- [Limitations](#limitations)
- [Future work](#future-work)
- [License](#license)

## Overview

| Module | Folder | Purpose |
|---|---|---|
| Logging middleware | `logging_middleware/` | Reusable logging layer used by the other services |
| Notification service | `notification_app_be/` | Fetches student notifications and builds a priority inbox (top 10) |
| Vehicle maintenance scheduler | `vehicle_maintence_scheduler/` | Schedules vehicle maintenance tasks |
| System design | `notification_system_design.md` | Six-stage design for a scalable notification platform |

> The scheduler folder name is spelled `vehicle_maintence_scheduler` in the repository. The README uses the exact folder name so links and commands work.

## Repository structure

```
.
├── logging_middleware/              # logging middleware package
├── notification_app_be/             # notification backend (entry point: index.js)
├── vehicle_maintence_scheduler/     # vehicle maintenance scheduler
├── notification_system_design.md    # system design write-up (Stages 1 to 6)
├── package.json
├── package-lock.json
└── .gitignore
```

## Modules

### 1. Logging middleware (`logging_middleware/`)

A small, reusable logging layer that the other modules import so that every service logs in the same format.

<!-- TODO: add the log function signature, accepted levels/stacks/packages, and where logs are sent. -->

### 2. Notification service (`notification_app_be/`)

Implements the **priority inbox** from Stage 6 of the design doc (entry point: `notification_app_be/index.js`).

How it ranks notifications:

1. Fetch all notifications from the notifications API.
2. Rank by type, highest first: **Placement**, then **Result**, then **Event**.
3. Within the same type, sort by timestamp, newest first.
4. Return the top **10**.

### 3. Vehicle maintenance scheduler (`vehicle_maintence_scheduler/`)

Schedules and orders vehicle maintenance work.

<!-- TODO: describe the input (vehicles, tasks, durations, budget/hours), the selection logic, and the output format. -->

## Notification system design

The full write-up is in [`notification_system_design.md`](./notification_system_design.md). Summary of each stage:

| Stage | Topic | Decision |
|---|---|---|
| 1 | REST API and real-time delivery | `GET /notifications`, `PATCH /notifications/:id/read`, `GET /notifications/unread`; WebSockets push new notifications to the student |
| 2 | Storage | PostgreSQL with a `notifications` table (`id`, `student_id`, `type`, `message`, `is_read`, `created_at`); risks noted: full scans, load spikes, unbounded growth |
| 3 | Slow query | Composite index on `(student_id, is_read, created_at DESC)`; avoid indexing every column because it slows writes |
| 4 | Load on the database | Redis cache with a short TTL (about 60 s); bypass the cache for Placement notifications so they are never delayed |
| 5 | Notify-all at scale | Bulk insert, push jobs to a queue, workers retry failed jobs individually instead of failing the whole batch |
| 6 | Priority inbox | Rank by type then recency, return the top 10; keep a sorted structure in Redis and trim to 10 on insert |

## Getting started

### Prerequisites

- Node.js 18 or newer
- npm

### Install

```bash
git clone https://github.com/ravindra11-jpg/RA2311032010017.git
cd RA2311032010017
npm install
```

### Run the notification service

```bash
node notification_app_be/index.js
```

<!-- TODO: confirm the run commands for the scheduler and any required environment variables (API base URL, auth token). -->

### Environment variables

Keep tokens and credentials out of the repository. Put them in a `.env` file (already covered by `.gitignore` if listed there) and never commit it.

| Variable | Meaning |
|---|---|
| `<TODO>` | Add each variable the code reads, for example an API base URL or access token |

## API reference

Designed API for the notification platform (see Stage 1 of the design doc).

| Method | Path | Description |
|---|---|---|
| `GET` | `/notifications` | List the student's notifications |
| `GET` | `/notifications/unread` | List only unread notifications |
| `PATCH` | `/notifications/:id/read` | Mark one notification as read, returns `{ "success": true }` |

All requests send `Authorization: Bearer <token>`.

Example response:

```json
{
  "notifications": [
    {
      "id": "uuid",
      "type": "Placement",
      "message": "some message",
      "isRead": false,
      "createdAt": "2026-05-02 10:00:00"
    }
  ]
}
```

## Design decisions and trade-offs

- **PostgreSQL** was chosen because the data is structured and relational.
- **One composite index** serves the main "unread for this student, newest first" query, rather than many single-column indexes.
- **Caching** reduces database load but can show data up to a minute old, so urgent Placement notifications skip the cache.
- **Queues and workers** isolate failures, so one bad send does not block thousands of other students.
- **Priority inbox** favours importance first (type), then recency.

## Limitations

- The priority inbox ranks by type and time only. It does not yet consider read state or user preferences.
- Redis caching, WebSocket delivery and the queue workers are described in the design doc but are not part of the running code unless added later.

## Future work

- Unit tests for the ranking logic and the scheduler
- Real-time delivery over WebSockets
- Queue-based bulk notification with retries
- Pagination for large inboxes
- Docker setup and CI

## License

Add a license of your choice before sharing publicly.
