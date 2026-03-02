+++
title = "Real‑Time Orchestration Platform"
description = "A cloud‑native, fault-tolerant AWS orchestration platform for 24×7 regulatory equity derivatives processing, built on Apache Airflow, Python microservices, Kubernetes, and a versioned CI/CD pipeline."
+++

This platform is an end-to-end orchestration system for equity derivatives calculation at institutional scale. It
coordinates data collection, instrument conversion, pricing calculation, and results publication across multiple AWS
services — running continuously against live market positions and publishing structured output downstream for risk
management and regulatory reporting.

The platform was architected, built, and delivered from the ground up. It spans an Apache Airflow DAG layer for
workflow orchestration, a Python utility library that abstracts all AWS and API interactions, a versioned CI/CD
pipeline for reproducible deployment, and a Kubernetes-hosted calculation service tier. All core services,
calculations, and infrastructure were designed and implemented as a cohesive, extensible platform — reviewed by AWS
and assessed as highly scalable and well-designed.

---

## The Problem

Equity derivatives processing at a regulated institution requires more than running calculations on demand. It demands
provenance — knowing exactly which positions, market data, and instruments were used for any given result, and being
able to reproduce that result on request. It demands resilience — the system must continue processing through partial
failures without corrupting state or losing progress. It demands observability — every stage of a multi-hour run must
be traceable in real time. And it demands operability — teams across development, QA, and production environments need
to deploy and configure consistently without manual intervention.

Existing tooling was fragmented: ad hoc scripts, no shared status tracking, no unified client library, and no
versioned release process. Each environment was a snowflake. Deployments were manual and error-prone.

---

## Orchestration Architecture

The orchestration layer is built on **Apache Airflow (MWAA)** running on AWS. The core DAG defines the full
end-to-end workflow for a given business date and calculator configuration, structured as a sequence of task groups
that enforce data dependencies while maximising parallelism.

The DAG begins with a validation task that parses the user-supplied run configuration, merges it with the DAG-level
environment context, and validates all required fields before any downstream work begins. Fail-fast validation at the
entry point prevents wasted compute on malformed runs.

An event query task fetches all data events tagged with the requested business date from the data hub. These events
carry S3 URLs for positions, market data, instruments, and time series snapshots. A data collection task group
processes each event type in parallel — fetching the tagged S3 URLs and converting each to Parquet via the
calculation services API. Static data (model control parameters and scenario sets) is collected in the same group.

A branch task filters data sources based on what the selected calculator requires, skipping conversion steps for
unused data types. A configuration assembly task combines tag data references with static data and records calculator
metadata before passing control to the calculation tier.

The calculator task group creates the calculation payload, submits it to the calculation service, collects result
URLs, converts results to Parquet, and stages the output to the publication bucket. A final publish task assembles
the publication payload — including dataset name, calculation type, business date, and input tag references — and
posts it to the data hub for downstream consumption.

Every stage writes status updates to DynamoDB, recording `Accepted`, `In Progress`, `Complete`, or `Failed` with
full payload context and timestamp. This provides a live audit trail of every run, queryable by process ID and
correlated across parent and child stages.

---

## Python Utility Library

The platform utility package is the shared foundation all services and DAGs build on. It is published to an internal
Artifactory PyPI repository and versioned using a date-based scheme (`YYYY.DDD.HHMM.SS`). Services declare a pinned
version in their `requirements.txt`, ensuring deterministic environments across dev, QA, and production.

The library is organised into four domains. The **`aws`** package wraps DynamoDB and S3. The `Status` class provides
a full lifecycle state machine — accepted, in-progress, complete, failed, custom — persisting each transition to a
DynamoDB table with process ID, parent correlation, timestamp, and serialisable payload. The `S3` module provides
typed bucket and key management for structured object storage.

The **`client`** package abstracts all external service interactions. Calculation and data service clients extend a
base endpoint resolver that handles URL construction, HTTP method dispatch, exponential backoff with jitter, and a
configurable payload callback for observability. The `Calculator` interface defines the contract all calculation
implementations must satisfy: `sources()`, `payload()`, `run()`, `report()`, and `close()`. A `Provider` class
resolves calculator instances by dotted module path at runtime, making the calculator registry open for extension
without code changes. `DataSource.Name` enumerates the canonical data types — Positions, Market Data, Time Series,
Instruments, Model Control, Scenarios — giving the entire platform a shared vocabulary for data routing.

The **`core`** package contains shared pure functions and I/O utilities. The **`etc`** package provides a `@config`
decorator that injects YAML-configured values into function parameters by key path, eliminating ad hoc environment
variable parsing and keeping all configuration paths explicit and discoverable.

The library is containerised via Docker, built through a Jenkins CI pipeline with automated packaging and Artifactory
publication, and tested with pytest. Coverage, quality, and complexity badges are published alongside each release.

---

## API and Service Contracts

Service interfaces are defined as OpenAPI 3.0 specifications and versioned independently. The Calculation Services
API exposes endpoints for data conversion (positions, market data, instruments, time series, calculation results),
model control retrieval, scenario set creation, and result publication. All long-running operations follow a HATEOAS
link-following pattern: the initial request returns a `_link` that the client polls until the operation completes.
This decouples the client from service internals and allows the service tier to scale independently.

The API gateway is deployed on AWS API Gateway behind a VPC endpoint, restricting access to within the AWS network
boundary. Requests carry correlation IDs derived from the run's DynamoDB status ID, enabling end-to-end tracing from
the Airflow task through the API call to the calculation result.

---

## Versioned CI/CD Pipeline

Every component in the platform — the Python library, the DAGs, the EC2 pricing node, the Kubernetes services — is
built, versioned, packaged, and deployed through a unified CI/CD framework of PowerShell scripts backed by Jenkins
and Bamboo.

Build versions are generated deterministically from the current date and time (`YYYY.DDD.HHMM.SS`), giving every
artefact a unique, sortable, human-readable identifier. The deployment script downloads versioned bundles from
Artifactory, expands them into versioned installation directories, applies environment-specific configuration via
secrets substitution, and registers or restarts services. Rollback is a re-deploy of a prior version tag — no state
to unwind.

A `versions.yaml` manifest declares the exact versions of every dependency in a single file, serving as the
authoritative description of a platform release. Promoting from QA to production means updating one tag and
re-running the pipeline. The CI/CD framework was subsequently adopted by other teams as the standard build,
configuration, and release process across the broader platform.

---

## Infrastructure

The platform runs across three compute tiers on AWS. **MWAA** hosts the Airflow orchestration layer, configured via
SSM Parameter Store secrets and deployed by updating S3 with a new DAG bundle. **Kubernetes** (EKS, managed via Helm
charts) hosts the calculation and data services, deployed using a versioned Helm chart from the internal chart
repository. **EC2** hosts the pricing engine node on Windows, running nginx as a reverse proxy, the .NET 6
application wrapper, and the pricing engine as a Windows service — all installed and configured by the PowerShell
deployment scripts.

The AWS environment was architected from first principles and reviewed by AWS, who assessed it as highly scalable
and well-designed. Infrastructure configuration is managed through SSM Parameter Store with a structured naming
convention, ensuring environment promotion requires only a prefix change with no code modifications.

---

## Synthetic Versioning

The platform introduced **[synthetic versioning](/projects/synthetic-platform-versioning/)** for all core services,
enabling fully repeatable deployments to any environment — from local development to production. A single YAML
manifest encapsulates the exact version of every component, dependency, and orchestration artefact. Deployments are
deterministic and environment-agnostic, with a clear audit trail of changes across all repositories.

By decoupling service evolution from infrastructure constraints, synthetic versioning accelerates onboarding,
simplifies disaster recovery, and supports rapid experimentation without impacting live operations. Releases are cut
multiple times a week with a single script execution, covering every repository simultaneously.

---

## Team Enablement

Beyond the technical delivery, the project focused on enablement. Reusable templates, shared configuration
frameworks, and targeted training sessions empowered other teams to adopt and extend the platform. Over time it
became the default orchestration backbone for multiple groups, each leveraging it for their own specialised
workloads. An intern was onboarded onto the platform and delivered a fully AI-enabled UI for orchestration,
building directly on the backend services and APIs.

---

## Cost Reduction

Three targeted initiatives delivered measurable cost savings while improving performance:

- **Caching & Storage Efficiencies** — Reduced repeated data retrieval and optimised storage tiers, lowering I/O
  costs and improving throughput.
- **Dynamic EC2 Provisioning** — Implemented on‑demand scaling of compute resources, ensuring capacity matched
  workload demand without over‑provisioning.
- **EKS Optimisation** — Tuned Kubernetes cluster configurations for workload patterns, reducing idle resource
  consumption and improving pod scheduling efficiency.

---

## Deployment Environments

The platform has been delivered through DEV, QA, and integration environments, with further onboarding in progress.
Each environment is defined entirely by its SSM parameter prefix and versions manifest — there are no
environment-specific code branches. This design makes environment parity a property of the system rather than a
discipline to be enforced manually.

---

## Skills

**Languages & Frameworks**  
Python | PowerShell | YAML | OpenAPI 3.0 | Dockerfile | Bash

**AWS**  
MWAA (Managed Airflow) | DynamoDB | S3 | API Gateway | VPC Endpoints | SSM Parameter Store | EKS

**Orchestration & Workflow**  
Apache Airflow | DAG Design | Task Groups | XCom | Branching | Dynamic Task Generation | AI‑Driven Automation

**Architecture & Design**  
Microservices | Plugin-Based Calculator Registry | HATEOAS API Pattern | Status Lifecycle State Machine | Domain-Driven Package Structure | Modular Service Design | Reusable Frameworks

**CI/CD & DevOps**  
Jenkins | Bamboo | Artifactory | Versioned Builds | Helm | Kubernetes | Docker | Skaffold | PowerShell Automation | Infrastructure as Code

**Observability**  
DynamoDB Audit Trail | Correlation IDs | Structured Logging | Payload Callbacks | Per-Stage Status Tracking | CloudWatch

**Performance & Optimisation**  
Dynamic Scaling | Caching Strategies | Cost‑Efficient Resource Allocation | Parallel Processing | Efficient I/O Handling

**Testing & Quality**  
pytest | Coverage Reporting | Complexity Badges | Isolated Unit Tests | Integration Test Fixtures
