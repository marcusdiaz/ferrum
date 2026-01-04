# Design Document: Rust-Based Data Transformation Tool (Ferrum)

## 1. Introduction

### 1.1 Purpose
This design document outlines the architecture and high-level components for a new data transformation tool, tentatively named "Ferrum" (inspired by Rust's etymology). Ferrum aims to address limitations in existing tools like DBT by decoupling table definitions from update rules, enabling flexible mappings, reusable steps, and composable flows. Built entirely in Rust for performance, safety, and concurrency, Ferrum will provide a project-based configuration system, a metadata database for storing logic, a visual frontend for design, and an execution engine for running pipelines. It supports diverse data sources (files, databases) and scheduling triggers.

### 1.2 Key Requirements
- **Decoupling**: Table definitions (sources/targets) separate from mappings and update rules.
- **Flexibility**: Multiple sources per mapping (with joins), overridable defaults for targets.
- **Project Structure**: YAML/JSON-based configuration files, similar to DBT projects.
- **Storage**: Metadata in a dedicated database (e.g., SQLite/PostgreSQL for portability).
- **Execution**: Connect to target databases (e.g., Snowflake, PostgreSQL) or file systems for pipeline runs.
- **Frontend**: Web-based UI for visual creation/editing of tables, mappings, flows, and steps.
- **Flows and Steps**: Flows as sequences of reusable steps; steps encapsulate mappings and transformations.
- **Scheduling**: Time-based or event-based (e.g., file arrival) triggers.
- **Data Handling**: Support local files, S3 buckets, FTP servers for ingress/egress.
- **No Code in This Doc**: Focus on high-level design without implementation details.

### 1.3 Assumptions and Scope
- Ferrum assumes users have basic SQL knowledge for mappings.
- Initial scope: Support SQL-based transformations; future extensions for other languages (e.g., Python via Rust embeddings).
- Security: Role-based access in metadata DB; encryption for connections.
- Out of Scope: Real-time streaming (focus on batch); advanced ML integrations.

## 2. System Overview

### 2.1 High-Level Architecture
Ferrum consists of four main layers:
- **Metadata Layer**: A database storing definitions, mappings, and flows.
- **Backend Layer**: Rust-based services for CRUD operations on metadata, validation, and orchestration.
- **Frontend Layer**: A web application for visual interaction.
- **Execution Layer**: Runtime engine for pipeline execution, scheduling, and monitoring.

Data flow:
1. Users define entities via frontend or CLI.
2. Metadata is stored in the DB.
3. Flows are scheduled or triggered.
4. Execution engine reads metadata, connects to sources/targets, and runs transformations.

### 2.2 Components
- **CLI Tool**: For project init, metadata sync, and manual runs.
- **Web Server**: Hosts the frontend and API endpoints.
- **Scheduler Service**: Background process for triggers.
- **Executor**: Handles pipeline runs, potentially as a separate binary for scalability.

All components built in Rust, leveraging crates like `tokio` for async, `sqlx` for DB interactions, `rusoto` for S3, `reqwest` for FTP/HTTP, and `serde` for serialization.

## 3. Backend Design

### 3.1 Metadata Database Schema
The metadata is stored in a relational database (default: SQLite for local dev; configurable to PostgreSQL for production). Tables are designed for normalization, with foreign keys for integrity. Key tables include:

- **projects**: Stores project configurations.
  - Columns: id (PK, UUID), name (string), description (text), config_yaml (text), created_at (timestamp), updated_at (timestamp).

- **tables**: Defines sources and targets (unified table for both).
  - Columns: id (PK, UUID), project_id (FK to projects), name (string), type (enum: 'source'|'target'), schema (text, e.g., DDL SQL), defaults_yaml (text, e.g., update rules like 'updated_at: CURRENT_TIMESTAMP'), location (string, e.g., DB connection string or file path/S3 URI), created_at (timestamp), updated_at (timestamp).

- **mappings**: Decouples logic from tables; supports multi-source joins.
  - Columns: id (PK, UUID), project_id (FK to projects), name (string), source_ids (array of UUIDs, FK to tables), target_id (FK to tables), sql_logic (text, e.g., SELECT with joins), overrides_yaml (text, e.g., custom update rules overriding target defaults), created_at (timestamp), updated_at (timestamp).

- **steps**: Reusable units encapsulating a mapping or transformation.
  - Columns: id (PK, UUID), project_id (FK to projects), name (string), mapping_id (FK to mappings), params_yaml (text, e.g., runtime variables), created_at (timestamp), updated_at (timestamp).

- **flows**: Sequences of steps forming pipelines.
  - Columns: id (PK, UUID), project_id (FK to projects), name (string), description (text), step_sequence (array of UUIDs, FK to steps), trigger_type (enum: 'schedule'|'file_arrival'), trigger_config (text, e.g., cron string or watch path), created_at (timestamp), updated_at (timestamp).

- **executions**: Logs runs for auditing.
  - Columns: id (PK, UUID), flow_id (FK to flows), status (enum: 'pending'|'running'|'success'|'failed'), start_time (timestamp), end_time (timestamp), logs (text).

- **connections**: Stores secure connection details (e.g., DB creds, S3 keys).
  - Columns: id (PK, UUID), project_id (FK to projects), name (string), type (enum: 'db'|'s3'|'ftp'|'local'), config_yaml (text, encrypted), created_at (timestamp), updated_at (timestamp).

Indexes: On foreign keys and frequently queried fields (e.g., project_id, name). Views: For denormalized queries, like flow_with_steps.

### 3.2 Backend Components
The backend is structured as modular Rust libraries and executables:

- **ferrum-core**: Core library with data models (structs for tables, mappings, etc.), validation logic (e.g., SQL syntax checks), and serialization/deserialization.
  - Handles decoupling: Ensures mappings reference tables without embedding logic.
  - Provides APIs for overriding defaults (e.g., merge YAML configs at runtime).

- **ferrum-metadata**: Library for DB interactions.
  - Uses sqlx for async queries.
  - CRUD operations for all entities.
  - Migration system (inspired by Diesel) to evolve schema.

- **ferrum-cli**: Executable binary.
  - Commands: init (create project skeleton), sync (push/pull metadata to DB), run <flow_id> (manual execution).
  - Parses project YAML files (e.g., ferrum.yml with DB connections, defaults).

- **ferrum-api**: Web server binary (using Axum framework).
  - REST/GraphQL endpoints for frontend (e.g., GET /tables, POST /flows).
  - Authentication: JWT-based.
  - Validation middleware to enforce decoupling rules.

- **ferrum-executor**: Separate binary for pipeline runs.
  - Reads flow metadata, resolves steps/mappings.
  - Connects to sources (e.g., via rusoto for S3, sqlx for DBs, custom FTP client).
  - Executes SQL in target DBs, applying overrides.
  - Supports parallelism: Tokio tasks for concurrent steps if no dependencies.

- **ferrum-scheduler**: Daemon binary.
  - Watches for triggers: Cron-like for schedules (using cron crate), file watchers for arrivals (using notify crate).
  - Spawns executor instances on triggers.
  - Configurable: Polls metadata DB for flow triggers.

Error Handling: Centralized logging (tracing crate), retries for transient failures (e.g., DB connections).

## 4. Frontend Design

### 4.1 Overview
The frontend is a single-page application (SPA) built with a Rust-to-WebAssembly framework (e.g., Yew or Leptos) for consistency, or optionally integrated with a JS framework like React via WASM bindings. It provides a visual interface for non-technical users to design without YAML editing.

### 4.2 Key Features
- **Dashboard**: Overview of projects, flows, and recent executions.
- **Table Editor**: Drag-and-drop for defining sources/targets; form-based DDL input; YAML editor for defaults.
- **Mapping Builder**: Visual query builder (e.g., node-based like Node-RED) for sources, joins, SQL logic, and overrides.
- **Step/Flow Composer**: Canvas for sequencing steps; drag reusable steps into flows; configure triggers.
- **Execution Monitor**: Real-time logs, status, and history.
- **Config Pages**: For connections (e.g., S3/FTP setup) and project settings.

### 4.3 UI Components
- **Visual Paradigm**: Graph-based (using a Rust-WASM graph library like egui or custom).
  - Nodes: Represent tables, mappings, steps.
  - Edges: Show data flow (e.g., source to mapping to target).
- **Interactivity**: Real-time validation (e.g., highlight invalid overrides).
- **Accessibility**: Keyboard navigation, ARIA labels.
- **Deployment**: Served by ferrum-api; static assets bundled.

Integration: API calls to backend for persistence; optimistic updates for responsiveness.

## 5. Execution and Runtime

### 5.1 Pipeline Execution Flow
1. Trigger (schedule or event) activates ferrum-scheduler.
2. Scheduler queries metadata DB for flow details.
3. Spawns ferrum-executor with flow_id.
4. Executor:
   - Loads sequence of steps.
   - For each step: Resolve mapping, fetch sources (file/DB), execute SQL with overrides in target.
   - Handles multi-targets or chains (e.g., output of one step as input to next).
5. Logs to executions table; notifies via API (for frontend).

### 5.2 Data Ingestion/Egress
- Files: Local (std::fs), S3 (rusoto_s3), FTP (custom client with reqwest).
- DBs: sqlx for execution; supports transactions for atomicity.
- Overrides: At runtime, merge target defaults with mapping overrides (e.g., via serde_yaml merging).

### 5.3 Scalability and Monitoring
- Distributed: Executor can run on multiple nodes (e.g., via message queue like RabbitMQ bindings).
- Metrics: Integrate with Prometheus (prometheus crate) for runs, errors.
- Testing: Unit tests in libraries; integration tests for end-to-end flows.

## 6. Future Enhancements
- Versioning for metadata (git-like).
- Plugin system for custom triggers/transforms.
- Integration with orchestration tools (e.g., Airflow export).

This design provides a robust, extensible foundation for Ferrum, emphasizing decoupling and usability over DBT's monolithic models.
