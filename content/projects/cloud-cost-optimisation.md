+++
title = "Cloud Cost Optimisation & Platform Engineering"
description = "£2M+ annual savings delivered through Kubernetes tuning, compute right-sizing, architectural refactoring, and a versioned CI/CD platform that cut deployment overhead across multiple teams."
tags = ["AWS", "Kubernetes", "DevOps", "Architecture", "CI/CD"]
+++

Cloud cost is rarely a billing problem — it is an architecture and operational discipline problem. The savings
delivered here came from eliminating waste built into the platform's foundations: over-provisioned compute,
manual deployment processes that required bespoke configuration per environment, and a lack of shared tooling
that caused every team to solve the same infrastructure problems independently.

The work spanned three interconnected areas: right-sizing and tuning the Kubernetes workloads running on AWS
EKS, designing and delivering a versioned CI/CD framework that was subsequently adopted across multiple teams,
and architecting the broader AWS environment — an architecture reviewed by AWS and assessed as highly scalable
and well-designed.

---

## Kubernetes Right-Sizing and Compute Optimisation

Kubernetes clusters running in AWS EKS accumulate cost through over-provisioned resource requests, idle node
capacity, and workloads that hold compute they no longer need between runs. The optimisation work addressed
each of these directly.

Resource requests and limits were tuned against observed peak and steady-state usage rather than conservative
estimates. This reduced the effective CPU and memory reservations per pod, which in turn reduced the number of
nodes required to schedule the same workload. Node groups were configured with appropriate instance types and
spot instance policies where workload characteristics allowed, capturing the price differential between on-demand
and spot without sacrificing availability.

Workloads were profiled and rescheduled to align compute-intensive jobs with off-peak pricing windows and to
avoid the idle-capacity cost of keeping large nodes warm between batch runs. Helm chart configurations were
standardised and versioned, ensuring that every environment ran the same resource policy and that tuning changes
propagated consistently rather than drifting back to defaults during the next manual deployment.

The Kubernetes configuration is managed through a versioned Helm chart published to an internal chart repository.
Each release pins the chart version alongside all other platform dependencies in a `versions.yaml` manifest,
making the exact compute configuration of any environment reproducible from a single file and a pipeline trigger.

---

## Versioned CI/CD Framework

Before the framework existed, each service had its own deployment approach — a mix of manual steps, per-service
scripts, and environment-specific configuration embedded in code. Promoting from DEV to QA to production required
human judgement at each step to reconcile differences. There was no standard for what a release looked like, no
guarantee that what passed QA was what reached production, and no fast path for rollback.

The CI/CD framework replaced this with a small, composable set of PowerShell functions and a Jenkins/Bamboo
pipeline that applies them consistently. Build versions are generated from the current date and time in the
format `YYYY.DDD.HHMM.SS` — deterministic, sortable, and human-readable without a registry lookup. Every
artefact produced by a build carries this version in its name and in its Artifactory path.

Deployment is driven by `versions.yaml`, a single manifest that declares the authoritative version of every
component in a platform release: the Python utility package, the Airflow DAG bundle, the Kubernetes Helm chart,
and the Windows EC2 node bundles (application, pricing engine, licence, nginx, .NET runtime). Promoting a release
from QA to production means updating one version tag and running the pipeline — no environment-specific branches,
no manual substitutions.

The `deploy.ps1` script downloads each versioned bundle from Artifactory, expands it into a versioned
installation directory, applies environment secrets through a user-secrets substitution step (resolving values
from the user's local credential store or a CI secret), and registers or restarts services. Rollback is a
re-deploy of the prior version tag. The installation directory structure preserves previous versions until
explicitly cleaned, giving operations a safe window for validation before removal.

Configuration values are externalised entirely to AWS SSM Parameter Store using a structured
`/ENV/SERVICE/COMPONENT/SETTING` naming convention. The application reads from SSM at startup; the only
difference between environments is the prefix. This means environment parity is a property of the system, not a
discipline that depends on engineers remembering which values to change.

The framework was subsequently adopted by the RDS team as their standard approach for configuration, local
builds, CI/CD pipelines, project templates, and versioned release processes — extending its impact well beyond
the platform it was originally built for.

---

## AWS Architecture

The AWS environment was designed to support 24×7 regulatory processing across multiple compute tiers while
remaining operationally manageable by a small team. The architecture separates concerns clearly: MWAA for
orchestration, EKS for stateless services, EC2 for the Windows-based pricing node, and managed services (S3,
DynamoDB, API Gateway) for storage, state, and ingress.

All inter-service communication routes through VPC endpoints, keeping traffic within the AWS network boundary
and eliminating data transfer costs for high-volume operations like S3 reads and DynamoDB writes. API Gateway
fronts the calculation and data services with a VPC endpoint configuration that restricts external access while
maintaining full internal reachability.

IAM roles follow least-privilege: MWAA's execution role grants only the SSM, S3, DynamoDB, and network
permissions it requires; service roles are scoped to their specific resources. SSM Parameter Store access is
granted via `ssm:GetParameter` on path-prefixed resources rather than broad policy statements.

The overall architecture was reviewed by AWS and formally assessed as highly scalable and well-designed.

---

## VDI Development Environment

Alongside the cloud infrastructure, a standardised VDI development environment was designed and delivered to
improve team velocity and consistency. The environment provides every developer with an identical baseline
— same tool versions, same credential handling, same build scripts — eliminating the "works on my machine"
class of problems that slow onboarding and CI diagnosis.

The VDI setup reduced the time for a new team member to reach a productive development state from days to hours
and materially improved the consistency of local builds versus CI outputs, reducing the frequency of
environment-specific failures reaching QA.

---

## Impact

The combined effect of Kubernetes right-sizing, architectural refactoring, and operational tooling delivered
over **£2 million in annual cloud cost savings**. The CI/CD framework and deployment automation reduced
deployment overhead across the platform and were adopted as the standard by a second team. The AWS architecture
review provided external validation of the design decisions and informed the roadmap for the next phase of
infrastructure investment.

---

## Skills

**Cloud & Infrastructure**
AWS EKS | AWS MWAA | AWS API Gateway | AWS DynamoDB | AWS S3 | SSM Parameter Store | VPC Endpoints | IAM Least-Privilege

**Kubernetes & Containers**
Helm | Skaffold | Docker | Resource Right-Sizing | Spot Instance Strategy | Node Group Configuration

**CI/CD & Automation**
Jenkins | Bamboo | Artifactory | PowerShell Automation | Versioned Build System | Rollback Strategy | Multi-Environment Promotion

**Architecture**
Platform Engineering | Multi-Tier AWS Design | Environment Parity | Configuration Externalisation | AWS Architecture Review

**Operational Excellence**
Cost Optimisation | Infrastructure as Code | Standardised Developer Environments | Cross-Team Framework Adoption

