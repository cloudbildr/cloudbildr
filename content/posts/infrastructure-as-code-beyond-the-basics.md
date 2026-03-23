---
title: "Infrastructure as Code: Beyond the Basics"
date: 2026-03-16
draft: false
tags: ["iac", "terraform", "devops", "automation"]
categories: ["engineering"]
series: ["cloud-fundamentals"]
description: "Most teams adopt infrastructure as code and stop at 'it works.' Here's what separates mature IaC practices from configurations that become a liability."
---

Most teams that adopt infrastructure as code follow the same trajectory: they start by translating their existing setup into configuration files, feel good about the reproducibility, and then gradually accumulate a codebase that becomes harder to change than the manual processes it replaced.

This post is about avoiding that outcome.

## The Problem with "It Works"

When infrastructure as code is done poorly, it exhibits specific failure modes:

- **Drift** between what the code describes and what actually exists in the cloud, usually caused by manual changes made during incidents
- **Sprawl** as the codebase grows without consistent structure, making it hard to find things or understand dependencies
- **Fear of change** because nobody is confident what a modification will actually do
- **Long feedback loops** because running a plan or apply takes ten minutes and touches too many resources at once

These aren't tool problems. They're process and structure problems that any IaC tool can develop.

## State is the Hard Part

The central challenge in infrastructure as code is state management. Your IaC tool maintains a record of what it believes exists in the world. When that record diverges from reality, things break in confusing ways.

A few practices that help:

**Lock state during applies.** If two engineers run applies simultaneously against the same state, the results are unpredictable. Remote state backends with locking prevent this.

**Store state remotely, not locally.** Local state files get lost, go stale, and can't be shared. Remote backends with versioning give you a safety net.

**Import before you write.** If you're codifying existing infrastructure, import resources into state before writing the configuration. Writing the config first leads to duplicated resources.

**Treat state corruption seriously.** If your state file is corrupted or severely drifted, stop and fix it before making any other changes. Applying on top of bad state compounds the problem.

## Module Design

Modules are the primary unit of reuse in most IaC tools. They're also where most complexity lives, for better and worse.

A well-designed module:

- Has a single, clear purpose
- Exposes inputs for everything that legitimately varies between uses
- Hides implementation details that callers shouldn't need to know about
- Is versioned so that callers can upgrade deliberately

A poorly designed module:

- Does too many things and accumulates unrelated resources over time
- Exposes every input, making callers deal with details they don't care about
- Has implicit dependencies that aren't expressed in the interface
- Is modified in place, breaking callers unexpectedly

The right granularity is usually "one module per logical component" — a database, a service, a network — rather than "one module per resource type" or "one module for everything."

## Testing Infrastructure Code

Infrastructure code is harder to test than application code because side effects are the point. You can't meaningfully test a VPC configuration without creating a VPC.

That said, several layers of testing are worth implementing:

**Static analysis** catches formatting issues, deprecated syntax, and common mistakes without touching the cloud. This should run on every commit and be fast.

**Plan validation** runs a plan against a real (or ephemeral) environment and asserts on the output. You can check that expected resources will be created, that certain tags are present, that no unexpected destructive changes will occur.

**Integration testing** actually applies infrastructure and verifies it behaves correctly. This is slow and expensive, so it's usually reserved for CI on the main branch rather than every pull request.

**Compliance scanning** checks that resources will be configured according to your security and governance policies before they're created.

## The Pipeline Question

Infrastructure changes should go through a pipeline, not be applied from an engineer's laptop. This isn't just about consistency — it's about auditability. When something goes wrong, you want to know exactly what was applied, by whom, and when.

A reasonable pipeline:

1. Validate and lint on every commit
2. Plan on every pull request, with the plan output posted as a comment
3. Require approval before applying to production
4. Apply in CI, not locally
5. Notify on success or failure

The plan-then-apply workflow is particularly important. Engineers reviewing a pull request should see exactly what infrastructure changes will result from merging. Reviewing a diff of configuration files is not the same as reviewing a diff of what will actually change.

## Organizational Considerations

As infrastructure codebases grow, ownership becomes a challenge. A few patterns that help:

**Separate repositories by blast radius.** Core networking and shared services go in one repository, individual application infrastructure in another. Changes to the network shouldn't require touching the same codebase as changes to a specific service's database.

**Establish conventions early.** Naming conventions, tagging requirements, and module structure are much harder to retrofit than to establish from the start. Write them down and enforce them in CI.

**Make the safe path the easy path.** If following best practices requires significantly more work than ignoring them, they won't be followed. Invest in tooling and templates that make the right thing the easy thing.

Infrastructure as code is one of those practices that's worth doing well. Done poorly, it's overhead. Done well, it becomes the foundation that everything else is built on.
