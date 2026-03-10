+++
title = "Project Page Template & Guide"
date = 2026-03-10
categories = ["Internal", "Guides"]
tags = ["Template", "AI-Instructions"]
# This page is internal and should not be linked in any public menus or lists.
# To ensure it is not indexed or listed by Hugo, we use the following build settings:
[build]
  list = "never"
  render = "always"
+++

# Project Page Template & Guide

This document serves as a comprehensive template and architectural guide for generating new project pages for this portfolio. When tasked with creating or expanding a project page, an AI assistant must strictly adhere to the structure, tone, and technical depth defined herein.

---

## 1. Front Matter Requirements

Every project page must begin with a TOML front matter block. 

```toml
+++
title = "Exact Project Name"
date = YYYY-MM-DD
categories = ["Primary Category", "Secondary Category"]
tags = ["Technology1", "Pattern1", "Domain1"]
+++
```

- **Title**: Professional and concise.
- **Date**: The approximate completion or major update date.
- **Categories**: High-level domains (e.g., "Architecture", "FinTech", "AI/ML", "Cloud Infrastructure").
- **Tags**: Specific technologies (e.g., "Java", "Python", "AWS") and architectural patterns (e.g., "Event-Driven", "AOP").

---

## 2. Style and Narrative Constraints

- **Prose-First Approach**: The page must be written in human-readable, narrative paragraphs. Avoid bulleted lists for descriptive content.
- **Professional Tone**: The language should be technical, authoritative, and focused on engineering excellence.
- **Strict No-Emoji Policy**: Emojis are strictly prohibited. Do not use symbols like 🚀, 🛠️, or 📥. Use plain text or standard Markdown formatting (bold/italics) for emphasis.
- **Section Separators**: Use horizontal rules (`---`) to separate major logical sections.
- **Visual Hierarchy**: Use `##` for main headings and `###` for sub-headings.

---

## 3. Core Content Sections

### 3.1 Repository Access
If the project has a public or internal repository, include a direct link at the very top of the page (below the front matter but above the main title).
Format: `[browse this repository](URL)`

### 3.2 Introduction and High-Level Design
Describe the project's purpose, the problem it solves, and its high-level architecture. Focus on the "why" behind the design choices.

### 3.3 Technical Depth and Architectural Patterns
This is the core of the page. You must explicitly discuss:
- **Design Patterns**: Identify and explain the implementation of patterns (e.g., Observer, Strategy, Provider, Builder, Singleton, Factory, Adapter, AOP).
- **Algorithmic Complexity**: Discuss O-Notation for critical paths (e.g., "$O(N)$ time complexity for event dispatching", "$O(1)$ space complexity via streaming").
- **State Management**: Discuss whether the system is stateless or how it manages state/persistence.
- **Concurrency**: Describe how the system handles parallel processing or thread safety.

### 3.4 Code Snippets
Include 1-2 high-impact code snippets that demonstrate a key architectural feature. Snippets must be:
- **Focused**: Show only the relevant logic.
- **Explained**: Precede or follow the snippet with a paragraph explaining why this code is significant.
- **Syntactically Correct**: Use the appropriate language identifier for Markdown code blocks.

### 3.5 Resilience and Recovery
Detail how the system handles failures. Mention specific mechanisms like:
- **Persistence**: How data is saved and recovered (e.g., `ResilienceStore`).
- **Validation**: Startup checks, type-safety, and contract enforcement.
- **Error Propagation**: How exceptions are handled and communicated across the system.

---

## 4. Skills and Technologies (ATS Optimization)

The final section of every page must be a "Skills and Technologies" list. This section is specifically designed for ATS (Applicant Tracking Systems) and quick professional review. It should be a series of bullet points grouped by context.

**Format Example:**

```markdown
## Skills and Technologies

- **Languages & Core**: Java 11+, Spring Boot, GoLang, Python.
- **Architecture & Patterns**: Event-Driven Design, AOP (AspectJ), Strategy Pattern, Builder Pattern.
- **Infrastructure & DevOps**: AWS (EC2, S3), Kubernetes, Docker, CI/CD Pipelines.
- **Tools & Libraries**: Jackson (Custom Modules), SpEL (Spring Expression Language), JUnit, Mockito.
```

---

## 5. Verification Checklist for the AI

Before submitting a new page, verify the following:
1. [ ] Is the front matter complete and accurate?
2. [ ] Is the content primarily narrative paragraphs?
3. [ ] Are there ZERO emojis or decorative icons?
4. [ ] Have I discussed Design Patterns and O-Notation?
5. [ ] Is there at least one relevant code snippet?
6. [ ] Is the "Skills and Technologies" section at the end and correctly formatted?
7. [ ] Does the page follow the standard Markdown hierarchy?
8. [ ] If a repo link was provided, is it included at the top?
