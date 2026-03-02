+++
title = "Synthetic Platform Versioning"
description = "A YAML-driven release coordination system that cuts synchronised git tags across every component repository multiple times a week — creating a single, auditable platform release from independently versioned services."
tags = ["DevOps", "CI/CD", "AWS", "Architecture", "PowerShell"]
+++

A distributed platform is made up of independently deployed components — a Python utility package, an Airflow
DAG bundle, a Kubernetes Helm chart, an EC2 application wrapper, API contracts, and supporting scripts. Each
component has its own build pipeline and its own version number. The challenge is that when you ask "what is
running in production right now?", you should be able to answer precisely — not just for one service, but for
every component simultaneously.

The synthetic versioning system solves this by treating the platform as a single releasable unit. A single YAML
manifest declares the authoritative version of every component. A PowerShell script reads that manifest, verifies
each component is at the correct build, and cuts a synchronised semantic version tag — `vN.M` — across every
repository at once. The result is that `v1.2` means exactly one thing: a specific, reproducible, fully
auditable snapshot of the entire platform.

---

## The Problem With Independent Versions

When services version independently, platform state becomes implicit. You know service A is at `2025.5.16` and
service B is at `2025.5.14`, but you have no artefact that says these two versions were tested together and are
safe to run together. Promotions from QA to production require someone to mentally reconcile a matrix of build
dates. Rollback means coordinating multiple independent deployments and hoping the combination you're rolling
back to was ever actually tested.

The conventional answer — a monorepo — trades independent deployment for coupling. All services move together,
which is exactly the wrong tradeoff for a platform where different components are owned by different teams and
deployed to different infrastructure.

The synthetic versioning approach keeps repositories independent while giving the platform a coherent release
identity. Each service moves at its own pace. The versioning system takes a point-in-time snapshot of all of
them and names it.

---

## The versions.yaml Manifest

The manifest is the single source of truth for a platform release. It declares the required configuration keys
the platform depends on, and the exact version of every deployable component:

```yaml
platform:
  version: 1.0

  parameters:
    - /ENV/PLATFORM/SERVICES/API/CALCULATION/BASE_URL
    - /ENV/PLATFORM/SERVICES/API/DATA/BASE_URL
    - /ENV/PLATFORM/SERVICES/API/STATUS/TABLE/NAME

  requirements:

    openapi:
      calculation:
        version: v0.1.1
        url: https://.../openapi/raw/calculation.yaml?at=refs%2Ftags%2Fv0.1.1
      data:
        version: v0.1.1
        url: https://.../openapi/raw/data.yaml?at=refs%2Ftags%2Fv0.1.1

    mwaa:
      repo:
        clone: https://.../orchestration-dags.git
        tag: v0.1.2
      python:
        packages:
          - name: platform-utilities
            version: 2025.5.16.184454

    ec2:
      pricing-service:
        tags:
          - v0.1.2
        urls:
          pricing-service: https://.../pricing-service-windows/2025.140.269.465/pricing-service-2025.140.269.465.zip
          scripts: https://.../deploy-scripts/2025.140.269.465/deploy-scripts-2025.140.269.465.zip
          nginx: https://.../nginx-1.28.0-windows/25.120.0.687/nginx-1.28.0-windows-25.120.0.687.zip
          dotnet: https://.../dotnet-6.0.36-windows/25.132.0.877/dotnet-6.0.36-windows-25.132.0.877.zip

    kubernetes:
      - helm:
          chart:
            name: helm-prod-local/platform-core
            version: 2025.5.20-9397
```

Every Artifactory URL encodes its own version in the path. Every Git reference is a tag. There are no floating
references, no `latest`, no ambiguity. Updating the manifest to a new component version is a one-line diff with
a clear, reviewable history.

---

## Cutting a Release

The release process is automated by a PowerShell script that accepts a `ReleaseType` parameter (`major`,
`minor`, or `patch`) and an optional explicit previous tag, then executes the following sequence.

**Version calculation.** The script clones the `versions` repository, fetches all existing tags matching the
`vN.M` pattern, sorts them by numeric value, and derives the next semantic version by incrementing the
appropriate component. A minor release of `v1.2` produces `v1.3`.

**Build version extraction.** The script parses `versions.yaml` with targeted regular expressions to extract
the build version of every component. The orchestration DAG bundle version is extracted from the S3 URL path.
The OpenAPI contract version is extracted from the Bitbucket tag reference in the URL. The EC2 application
wrapper version is extracted from the Artifactory path segment before the filename. Kubernetes service image
tags are extracted from the environment values file using a pattern that matches the `service.image.tag`
structure for each named service. For any component whose version cannot be automatically determined, the script
prompts the operator to enter it manually, then continues — making the system robust to structural variation
without silently skipping components.

**Repository validation.** The script clones each component repository and checks out the exact tag that
`versions.yaml` declares. If the fetch fails for any tag — because it does not exist or the repository is
unreachable — the script throws immediately. Every component must be provably at its declared version before
any tags are written. This is the guarantee that makes the release meaningful: the platform tag only exists if
all components are verified.

**Confirmation gate.** Before writing anything, the script prints a summary of every repository and the tag it
is about to receive, then requires explicit `y` confirmation. This prevents accidental releases and gives the
operator a final chance to verify the matrix.

**Tag application.** For each component repository, the script checks whether the release tag already exists
(idempotent if re-run) and creates it if not, using an annotated tag with a descriptive message. All tags are
pushed to origin immediately. The `versions` repository itself receives the same tag, making the manifest
version part of the platform release record.

**Release notes generation.** After all tags are applied, a diff script is called with the previous and new
tags, generating a changelog that spans every repository in the platform — a single document describing every
change included in the release.

---

## Release Cadence

Because the process is automated and the confirmation step takes seconds, releases are cut multiple times a week
— whenever a component reaches a stable, tested state and the platform is ready to snapshot it. There is no
release freeze, no manual coordination meeting, no spreadsheet of component versions. The manifest is updated,
the script is run, and the platform has a new named release.

This cadence is only possible because the versioning system is synthetic. The underlying repositories have not
changed their branching strategies or introduced release trains. The platform tag is layered on top of their
existing build tags without requiring any changes to how individual services operate.

---

## Environment Promotion

Promoting a release from DEV to QA to production is a single operation: update the `versions.yaml` tag reference
in the target environment's configuration and run the deployment pipeline. Because every component version is
pinned in the manifest, there is no ambiguity about what gets deployed. Because the platform tag exists in every
component repository, every team can see exactly which platform release their service version belongs to.

Rollback follows the same pattern. The previous `versions.yaml` state is already tagged in the repository. A
re-deploy against the prior tag returns the entire platform to its last known good state in one operation, with
no per-service coordination required.

---

## Kubernetes Service Versioning

The Kubernetes tier adds a second layer of versioning on top of the platform release. The `platform-core` Helm
chart is a meta-chart that composes individual service charts — each with its own image tag declared
independently in the environment values file. The Helm chart version itself is generated in CI using the same
date-based scheme, keeping the Kubernetes deployment version traceable to the build that produced it.

The CI pipeline bakes the chart version at build time using a date expression that produces a `YYYY.M.D-HHMMSS`
format, packages the chart, and pushes it to the internal Artifactory Helm registry. Deployment uses
`helm upgrade --install` with a per-namespace isolation strategy, ensuring that DEV and production can run
different chart versions against the same cluster without conflict.

---

## Skills

**Release Engineering**
Synthetic Versioning | Semantic Version Calculation | Multi-Repository Tag Coordination | Automated Release Notes

**CI/CD & Automation**
Jenkins | Bamboo | PowerShell | Bash | Helm | Artifactory | Git Tag Management | Idempotent Release Scripts

**Infrastructure as Code**
YAML-Driven Platform Configuration | SSM Parameter Store | Environment Parity | Single-Manifest Promotion

**Architecture**
Distributed Release Coordination | Independent Service Deployment | Platform Snapshot Pattern | Rollback Strategy

**Kubernetes**
Helm Chart Composition | Per-Service Image Tagging | Namespace Isolation | Versioned Chart Publication

