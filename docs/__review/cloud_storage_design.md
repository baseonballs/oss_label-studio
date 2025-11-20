# Technical Design Document: Cloud Storage Integration

**Author**: Antigravity
**Status**: Draft
**Last Updated**: 2025-11-20

## 1. Context and Scope

### 1.1 Background
Label Studio requires the ability to ingest large datasets (images, audio, video) and export labeled data without physically moving the raw assets into the application database. Users typically store these assets in cloud object stores like AWS S3, Google Cloud Storage (GCS), or Azure Blob Storage.

### 1.2 Objective
Design a scalable, extensible system to:
1.  **Import**: Automatically discover objects in a cloud bucket and create corresponding Tasks in Label Studio.
2.  **Export**: Automatically serialize and upload completed Annotations back to the cloud bucket.
3.  **Sync**: Maintain synchronization state to handle new files and avoid duplicates.

### 1.3 Non-Goals
*   **Real-time Bi-directional Sync**: The system relies on periodic polling or manual sync triggers, not real-time bucket event notifications (e.g., S3 Event Notifications).
*   **Bucket Management**: Creating or configuring buckets/permissions is out of scope; the system assumes existing accessible buckets.

## 2. Design Overview

### 2.1 System Context
The Cloud Storage system acts as a bridge between the Label Studio Core (Tasks/Annotations) and external Cloud Providers.

```mermaid
graph TD
    User[User] -->|Configures| API[Storage API]
    API -->|Creates| DB[(PostgreSQL)]
    
    subgraph "Label Studio"
        Worker[Background Worker (RQ)]
        Signal[Signal Handler]
    end
    
    DB -->|Reads Config| Worker
    Worker -->|Polls/Reads| Cloud[Cloud Storage (S3/GCS/Azure)]
    Cloud -->|Returns Keys| Worker
    Worker -->|Creates| Task[Task]
    
    Annotation[Annotation Created] -->|Triggers| Signal
    Signal -->|Uploads JSON| Cloud
```

### 2.2 High-Level Architecture
The system employs a **Polymorphic Architecture** using:
*   **Abstract Base Classes (`ImportStorage`, `ExportStorage`)**: Define the contract for syncing.
*   **Provider Mixins (`S3StorageMixin`, `GCSStorageMixin`)**: Encapsulate SDK-specific logic (auth, connection).
*   **Concrete Implementations**: Combine Base Classes and Mixins (e.g., `S3ImportStorage`).

## 3. Detailed Design

### 3.1 Data Models

#### 3.1.1 Storage Configuration (`Storage`)
Stores connection details and sync state.
*   **Inheritance**: `StorageInfo` (State) -> `Storage` (Abstract) -> `ImportStorage` / `ExportStorage`.
*   **Key Fields**:
    *   `type`: Discriminator (s3, gcs, azure).
    *   `bucket`, `prefix`, `regex_filter`: Scope definition.
    *   `last_sync`, `status` (initialized, queued, in_progress, failed, completed).
    *   `credentials`: (Encrypted) Access keys, secrets.

#### 3.1.2 Storage Links (`StorageLink`)
Tracks the relationship between a cloud object and a Label Studio entity to ensure idempotency.
*   **Fields**:
    *   `key`: The object key in the bucket.
    *   `storage`: FK to Storage.
    *   `task` (for Import) or `annotation` (for Export): FK to the entity.
    *   `created_at`: Timestamp.

### 3.2 API Design

The API follows RESTful principles using Django Rest Framework (DRF).

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/storages/{type}` | List all storages of a specific type. |
| `POST` | `/api/storages/{type}` | Create a new storage configuration. |
| `POST` | `/api/storages/{type}/{id}/sync` | Trigger a background sync job. |
| `POST` | `/api/storages/{type}/validate` | Validate connection credentials. |

### 3.3 Business Logic

#### 3.3.1 Import Workflow (`scan_and_create_links`)
Executed asynchronously via RQ worker.
1.  **Fetch**: Iterate over objects in the bucket using the provider SDK (`iter_objects`).
2.  **Filter**: Apply `prefix` and `regex_filter`.
3.  **Deduplicate**: Check `ImportStorageLink` to see if `key` already exists for this storage.
4.  **Process**:
    *   If `use_blob_urls=True`: Generate a URL (presigned or public) and create a Task with `{image: url}`.
    *   If `use_blob_urls=False`: Download the JSON content and parse it as Task data.
5.  **Persist**: Create `Task` and `ImportStorageLink` in a transaction.
6.  **Update State**: Update `StorageInfo` with progress and final status.

#### 3.3.2 Export Workflow (`save_annotation`)
Triggered via Django `post_save` signal on `Annotation`.
1.  **Check**: Verify if the Project has active `ExportStorage` configured.
2.  **Serialize**: Convert Annotation to the target JSON format (`ExportDataSerializer`).
3.  **Naming**: Generate object key (e.g., `{prefix}/{task_id}.json`).
4.  **Upload**: Put object to the bucket using provider SDK.
5.  **Link**: Create `ExportStorageLink`.

### 3.4 Scalability & Performance
*   **Asynchronous Processing**: All heavy I/O (scanning buckets) is offloaded to Redis Queue (RQ) to prevent blocking the web server.
*   **Batch Processing**: Import logic processes keys in batches to reduce DB round-trips (though `iter_objects` is often a generator).
*   **Pagination**: API list endpoints support pagination.
*   **Caching**: Cloud SDK clients are cached to avoid re-initialization overhead (`clients_cache`).

## 4. Cross-Cutting Concerns

### 4.1 Security
*   **Credential Storage**: Credentials (AWS keys, etc.) are stored in the database. *Recommendation: Integrate with a Secrets Manager or use Field-level encryption.*
*   **Presigned URLs**: For private buckets, the system generates short-lived presigned URLs for the frontend to access media, ensuring assets remain private.

### 4.2 Observability
*   **Status Tracking**: The `StorageInfo` model tracks the lifecycle of sync jobs (`queued` -> `in_progress` -> `completed/failed`).
*   **Tracebacks**: Failures store the full python traceback in the `traceback` field for debugging.

### 4.3 Reliability
*   **Atomic Transactions**: Task and Link creation happens inside `transaction.atomic()` to prevent orphaned records.
*   **Retries**: RQ handles job retries for transient failures.

## 5. Alternatives Considered

### 5.1 Event-Driven Architecture (Webhooks/Lambda)
*   **Idea**: Configure Cloud Provider to send a webhook to Label Studio on object creation.
*   **Pros**: Real-time, no polling overhead.
*   **Cons**: Complex setup for the user (requires configuring Lambda/PubSub), requires Label Studio to be publicly accessible.
*   **Decision**: **Polling** was chosen for ease of use and deployment simplicity (works behind firewalls).

### 5.2 Storing Blobs in Database
*   **Idea**: Upload files directly to Label Studio DB.
*   **Pros**: Simple backup/restore.
*   **Cons**: Bloats database, expensive storage, slow serving.
*   **Decision**: **Reference-based** approach (storing URLs/Keys) is standard for large datasets.

## 6. Test Plan (TDD)

### 6.1 Unit Tests
*   **`test_storage_state_machine`**: Verify status transitions.
*   **`test_s3_mixin`**: Mock `boto3` to verify `get_client` and `validate_connection`.
*   **`test_import_logic`**: Mock `iter_objects` to return dummy keys, verify `Task` creation.

### 6.2 Integration Tests
*   **`test_api_create_storage`**: Verify REST API creates DB records.
*   **`test_sync_job_execution`**: Verify calling `sync()` enqueues an RQ job.
