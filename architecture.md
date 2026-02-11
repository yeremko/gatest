# Solution Architecture — Laravel Web Application

This document describes the architecture of the web solution built with PHP Laravel, including runtime topology, data flows, and deployment model.

---

## 1. High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        Browser[Web Browser]
    end

    subgraph Application["Application Layer"]
        Laravel[Laravel Application\nPHP · Front-end & Back-end]
        QueueWorker[Queue Worker\nphp artisan queue:work]
        Scheduler[Scheduler\nphp artisan schedule:work]
    end

    subgraph Data["Data & Storage"]
        MySQL[(MySQL Database)]
        Redis[(Redis\nCache · Sessions · Queues)]
        FS[File Storage\nLocal / S3]
    end

    Browser -->|HTTP/HTTPS| Laravel
    Laravel --> MySQL
    Laravel --> Redis
    Laravel --> FS
    QueueWorker --> MySQL
    QueueWorker --> Redis
    QueueWorker --> FS
    Scheduler --> MySQL
    Scheduler --> Redis
    Scheduler --> FS
```

---

## 2. Component Overview

| Component | Role |
|-----------|------|
| **Laravel Application** | Monolithic PHP app: serves web UI, API, and server-side logic. |
| **Queue Worker** | Runs `queue:work` to process jobs from Redis queues (async tasks). |
| **Scheduler** | Runs `schedule:work` to execute Laravel scheduled tasks (cron-like). |
| **MySQL** | Primary relational database for application data. |
| **Redis** | Cache, sessions, and queue driver (and optionally broadcast). |
| **File Storage** | Temp, cache, and data files; backend can be local disk or S3. |

---

## 3. Runtime Topology (Docker Compose)

```mermaid
flowchart LR
    subgraph DockerHost["Docker Host"]
        subgraph Network["Network: sail (bridge)"]
            App[laravel.test\n:80 → APP_PORT]
            Queue[queue\nqueue:work redis]
            Sched[scheduler\nschedule:work]
            MySQL[(mysql\n:3306)]
            Redis[(redis)]
        end
    end

    User[User] -->|APP_PORT| App
    App --> MySQL
    App --> Redis
    Queue --> MySQL
    Queue --> Redis
    Sched --> MySQL
    Sched --> Redis
```

- **laravel.test**: Web server (e.g. Nginx/Apache + PHP); exposes app port and optionally Vite dev port.
- **queue**: Same app image, entrypoint `php artisan queue:work redis`; processes jobs from Redis.
- **scheduler**: Same app image, entrypoint `php artisan schedule:work`; runs cron-like tasks.
- **mysql**: MySQL 8.0; port forwarded to host (e.g. 3309); persistent volume for data.
- **redis**: Redis; used by app, queue, and scheduler (no port exposed unless configured).

---

## 4. Data Flow

### 4.1 Web Request Flow

```mermaid
sequenceDiagram
    participant User
    participant Laravel
    participant Redis
    participant MySQL
    participant Storage

    User->>Laravel: HTTP Request
    Laravel->>Redis: Session / Cache lookup
    alt Cache hit
        Redis-->>Laravel: Cached data
    else Cache miss
        Laravel->>MySQL: Query
        MySQL-->>Laravel: Result
        Laravel->>Redis: Store in cache
    end
    Laravel->>Storage: Read/Write files (if needed)
    Laravel-->>User: HTTP Response
```

### 4.2 Asynchronous Job Flow

```mermaid
sequenceDiagram
    participant Laravel
    participant Redis
    participant QueueWorker
    participant MySQL
    participant Storage

    Laravel->>Redis: Dispatch job to queue
    Laravel-->>User: Response (job queued)
    QueueWorker->>Redis: Poll for jobs
    Redis-->>QueueWorker: Job payload
    QueueWorker->>MySQL: Process (DB ops)
    QueueWorker->>Storage: Process (files / S3)
    QueueWorker->>Redis: Mark complete / release
```

### 4.3 Scheduled Task Flow

```mermaid
sequenceDiagram
    participant Scheduler
    participant Laravel
    participant MySQL
    participant Redis
    participant Storage

    loop Every minute (schedule:work)
        Scheduler->>Laravel: Evaluate schedule
        Laravel->>Laravel: Run due tasks (closures, commands)
        Laravel->>MySQL: Task logic
        Laravel->>Redis: Cache / queue (if used)
        Laravel->>Storage: Files / exports (if used)
    end
```

---

## 5. Storage Architecture

```mermaid
flowchart TB
    subgraph LaravelApp["Laravel Application"]
        FS[Filesystem Facade\nStorage::]
    end

    subgraph Backends["Storage Backends"]
        Local[Local Disk\nstorage/app\nstorage/framework]
        S3[S3-Compatible\nObject Storage\nBucket]
    end

    FS -->|default / local| Local
    FS -->|s3 disk| S3
```

- **Local**: Temp files, cache, and app-generated files under `storage/` (and optionally public uploads).
- **S3**: Same API via Laravel filesystem; used for persistent or shared assets (e.g. uploads, exports, backups).
- Configuration is environment-driven (e.g. `FILESYSTEM_DISK`, `AWS_*`); same code can use local in dev and S3 in production.

---

## 6. Technology Stack Summary

```mermaid
flowchart LR
    subgraph Runtime["Runtime"]
        PHP[PHP 8.2]
        Laravel[Laravel]
        Sail[Laravel Sail / Docker]
    end

    subgraph Data["Data"]
        MySQL[(MySQL 8.0)]
        Redis[(Redis)]
    end

    subgraph Storage["Storage"]
        Local[Local FS]
        S3[S3]
    end

    subgraph Processes["Processes"]
        Web[Web]
        Queue[Queue Worker]
        Cron[Scheduler]
    end

    PHP --> Laravel
    Laravel --> Sail
    Laravel --> MySQL
    Laravel --> Redis
    Laravel --> Local
    Laravel --> S3
    Web --> Laravel
    Queue --> Laravel
    Cron --> Laravel
```

---

## 7. Deployment Context (Docker Compose)

- **Images**: Application uses PHP 8.2 Sail-based image; MySQL and Redis use official images.
- **Volumes**: App code mounted into `laravel.test`, `queue`, and `scheduler`; MySQL data in a dedicated volume.
- **Dependencies**: `laravel.test`, `queue`, and `scheduler` depend on `mysql` and `redis`; queue and scheduler start after DB/Redis are available.
- **Scaling**: Multiple `queue` replicas can be added for higher throughput; single `scheduler` is typically sufficient.

---

## 8. Document Info

- **Format**: Markdown with Mermaid diagrams in fenced blocks. Use **Markdown preview** (with Mermaid support) on this `.md` file to see both text and diagrams.
- **mermaid.live**: Paste **only** the diagram code there (no markdown). Copy the contents of one mermaid block above—from the first line like `flowchart TB` or `sequenceDiagram` to the last line of that block—and paste into mermaid.live. Do not paste the whole page or any lines starting with `#`.
- **Scope**: Architecture only; no code. Implementation details (routes, env vars, exact ports) follow from project configuration and this layout.
