+++
title = "Projects"
description = "Technical projects and architectural frameworks by Mark Cooper"
+++

Welcome to my portfolio of projects. This section highlights a selection of systems I've designed and delivered across
cloud-native architecture, AI orchestration, financial risk platforms, and capital markets. Each project reflects a
strategic approach to modularity, resilience, and operational clarity -- whether it's orchestrating BAU workflows,
extracting behavioural signals from market data, or embedding governance into model-driven pipelines.

The portfolio is built around frameworks that are difficult to replicate and easy to extend. I focus on separating
mechanical complexity from human intent, allowing systems to remain composable, cache-centric, and runtime-aware.

---

## Cooper's Way

All architecture across this portfolio conforms to **[Cooper's Way](/projects/coopers-way/)** &mdash; an engineering
doctrine built around interface isolation, immutable state transitions, dynamic auto-scaling, and fail-fast resilience.

- **[Architectural Principles & Engineering Directives](/projects/coopers-way/)**  
  A unified architectural doctrine and concurrency methodology covering domain decoupling, immutable task payloads,
  blocking priority queues, dynamic auto-scaling worker pools, fail-fast circuit breakers, and zero-data-loss state recovery.

- **[Engineering Best Practices & Design Patterns](/projects/ai/best-practice/)**  
  A practical handbook covering the four foundations (SRP, DRY, KISS, Interface-Driven Design), dependency injection layers,
  builder-flow interfaces, and pattern catalogs.

---

## Featured Systems

- **[Production Equity Swap Pricing Engine](/projects/production-equity-swap-pricing-engine/)**  
  An enterprise-grade C# / .NET library for pricing equity total return swaps. Implements compile-time workflow correctness
  via the Type-State Pattern, immutable domain modeling with strict ISDA contract/position separation, and a layered validation
  pipeline validated with a 97.8% pass rate across a 5,000-position benchmark dataset.

- **[Enterprise Chromium Extension Ecosystem](/projects/chromium-extension-ecosystem/)**  
  A Manifest V3 monorepo ecosystem of five browser extensions powered by a shared core kernel. Implements immutable
  task payloads, a blocking priority queue, dynamic worker pool auto-scaling, a rate-limit circuit breaker, and 162
  automated unit tests.

- **[High-Performance Go Systems & Data Pipelines](/projects/high-performance-go-data-pipelines/)**  
  Zero-allocation stream processing engines and CLI pipelines built in Go. Features streaming JSON parsers with `sync.Pool`
  buffer recycling, a bounded concurrent IMAP harvester, programmatic EPUB3/PDF compilers, and a sub-millisecond Chromium
  Native Messaging host.

- **[Murex Capital Markets AI Agent & LLM Proxy](/projects/murex-capital-markets-ai-agent/)**  
  An enterprise AI integration bridge connecting internal Microsoft Teams channels to a specialized Murex MX.3 trading copilot. Built with
  a zero-dependency Go proxy, dynamic context window pruning, and a domain persona engine that prevents hallucinated software commands.

- **[Identity‑Centric Encryption with User Consent](/projects/identity-centric-encryption-with-user-consent/)**  
  Every record is protected with its own key, and decryption requires both verified identity and explicit user approval,
  ensuring security, consent, and a clear audit trail.

- **[Service Framework](/projects/service-framework/)**  
  An annotation-driven event orchestration engine for BAU workflows. It enables declarative routing, conditional
  execution, and modular service composition using Spring Boot.

- **[Reconciliation Framework](/projects/reconciliation-framework/)**  
  A provider-based framework for simplifying reconciliation logic. Engineers focus on domain-specific rules while the
  engine handles boilerplate execution, timing, and exit signalling.

- **[Permissions Service](/projects/permissions-service/)**  
  A showcase for managing fine-grained ingress permissions within Kubernetes where Angular, TypeScript are used as a
  frontend backed by a ReST API using GoLang for the permission and group management of IdP provided users via JWTs.

- **[Swaps Pricing Engine (MVP)](/projects/swaps-pricing-engine/)**  
  A C# library for pricing total return equity swaps, wrapping an unmanaged pricing engine behind
  idiomatic .NET domain objects, a fluent type-state builder API, and a layered validation framework. Achieved an
  81.6% pass rate across initial benchmark positions.

- **[Cloud Cost Optimisation & Platform Engineering](/projects/cloud-cost-optimisation/)**  
  £2M+ annual savings delivered through Kubernetes right-sizing, compute tuning, and a versioned CI/CD framework
  subsequently adopted by multiple teams. AWS architecture reviewed and assessed as highly scalable and well-designed.

- **[Synthetic Platform Versioning](/projects/synthetic-platform-versioning/)**  
  A YAML-driven release coordination system that cuts synchronised git tags across every component repository multiple
  times a week -- creating a single, auditable platform release from independently versioned services without monorepo
  coupling.

- **[MPC Project Analyser](/projects/optimising-ai-credits/)**  
  A deterministic Python analysis engine that generates compact AST metadata, file trees, and documentation maps so AI agents can plan software changes without consuming raw source code. Pairs with a three-persona workflow -- Reviewer, Planner, and Coder -- to keep premium model usage focused on reasoning and architecture.

- **[AI-Based Image Classification & Face Recognition](/projects/image-classification/)**  
  A GPU-accelerated, CSV-driven framework for enriching large image and video datasets. Generates cached thumbnails,
  assigns CLIP-based semantic labels, detects people and animals, and applies a hybrid AI/heuristic model to flag junk
  content -- all in a modular, reproducible pipeline.

- **[Murex On-Demand](/about/murex/)**  
  A containerized, CI/CD-driven Murex platform enabling rapid, reproducible environment provisioning for development,
  testing, and training. Built on orchestration and configuration-as-code principles, it integrates with Datamart
  pipelines, secure messaging flows, and managed task extensions to deliver full environments in minutes instead of
  days.

Each project page includes architecture notes, runtime behaviour, and design rationale -- with more to come as the
portfolio expands.
