# CI Pipeline Case Study: VM-Based System Validation in QEMU

A public, sanitized technical case study of a CI pipeline I designed and implemented during an industry internship. The pipeline boots a Linux VM in QEMU using a deterministic **kernel + initrd** approach, provisions the instance, runs functional workflows, validates outputs using a **Go Cobra CLI** test harness, and publishes versioned artifacts for reproducible future runs.

> Note: Organization names, internal URLs, proprietary configs, and product-specific commands are intentionally generalized. The architecture, engineering decisions, and validation strategy reflect the work I delivered.

---

## Table of Contents
- [Overview](#overview)
- [Problem](#problem)
- [Goals](#goals)
- [Solution Summary](#solution-summary)
- [Architecture](#architecture)
- [Pipeline Stages](#pipeline-stages)
- [Key Design Decisions](#key-design-decisions)
- [Validation Strategy](#validation-strategy)
- [Failure Handling and Debuggability](#failure-handling-and-debuggability)
- [Repo Structure](#repo-structure)
- [Local Reproduction (Sanitized)](#local-reproduction-sanitized)
- [Extending the Test Suite](#extending-the-test-suite)
- [Security and Public-Repo Notes](#security-and-public-repo-notes)
- [Results](#results)
- [What I Would Improve Next](#what-i-would-improve-next)
- [Resume Bullets](#resume-bullets)
- [License](#license)

---

## Overview

This repo documents a CI-driven approach to validating a Linux-based system in a production-like VM environment. Instead of relying only on unit tests, the pipeline runs real workflows inside a QEMU VM and uses a CLI harness to produce deterministic pass/fail results with useful logs.

This pattern applies well to systems that need:
- VM-based integration tests in CI
- reproducible boot and provisioning
- workflow validation with strict output assertions
- versioned artifacts for repeatability

---

## Problem

Manual VM testing was:
- slow to run and hard to standardize
- sensitive to environment drift over time
- difficult to debug consistently across engineers
- not well integrated into CI feedback loops

CI needed a deterministic way to boot a VM, provision it, run real workflows, and produce a strict pass/fail signal.

---

## Goals

- Deterministic VM boot in CI with minimal fragile dependencies
- Non-interactive provisioning and test execution
- End-to-end validation of real workflows (state transitions, “check-in/check-out”-style operations, stats queries)
- Automated output verification with clear pass/fail behavior
- Artifact-based reproducibility by publishing versioned VM images and logs

---

## Solution Summary

On every pipeline run:

1. Fetch a baseline VM image from an artifact repository
2. Boot the VM in QEMU using an explicit kernel + initrd method
3. Provision and run smoke checks as a setup user (`user1`)
4. Execute functional workflows representing real operations
5. Validate expected outputs using a Go Cobra CLI harness as a test user (`user2`)
6. Publish the updated VM image and logs back to an artifact repository

---

## Architecture

### High-level flow

---

## Components

- **CI Orchestrator:** GitLab CI/CD (pattern is CI-agnostic)
- **Virtualization:** QEMU
- **Boot Strategy:** explicit kernel + initrd for repeatable startup
- **Provisioning:** scripted, non-interactive setup
- **Test Harness:** Go + Cobra CLI for consistent command execution
- **Artifact Store:** generic artifact repository (versioned VM images + logs)

---

## Pipeline Stages

### Stage 0: Fetch baseline VM artifact
**Purpose:** start from a known-good baseline and prevent environment drift.

**Outputs:**
- baseline VM image downloaded
- metadata captured (commit hash, pipeline id)

### Stage 1: Boot the VM (kernel + initrd)
**Purpose:** deterministic boot in CI, independent of guest bootloader configuration.

**Signals:**
- VM reaches “ready” state
- console logs are captured and saved as artifacts

### Stage 2: Provision + smoke checks as `user1`
**Purpose:** fail fast and keep provisioning separate from validation.

**Typical tasks:**
- instance setup
- dependency and config verification
- basic command smoke tests

### Stage 3: Execute functional workflows
**Purpose:** validate production-like behavior end to end.

**Representative workflow categories:**
- “check-out/check-in”-style stateful operations
- activation and deactivation transitions
- stats or status queries to confirm server-side state

### Stage 4: Validate outputs via CLI harness as `user2`
**Purpose:** deterministic pass/fail and consistent validation.

**Actions:**
- switch to `user2`
- run CLI commands for workflows
- compare outputs to expected results
- fail CI on mismatch (non-zero exit code)

### Stage 5: Publish updated VM image + logs
**Purpose:** reproducibility and rapid debugging for future runs.

**Published artifacts:**
- updated VM image
- console logs and per-stage logs
- output diffs for failed assertions
- metadata (version tags, build info)

---

## Key Design Decisions

### Why kernel + initrd boot
I initially explored SSH and serial-console automation, but it required bootloader-level changes and increased maintenance overhead.

Switching to a kernel + initrd approach improved:
- boot determinism in CI
- isolation from bootloader drift
- early boot visibility through captured console logs

### Why two users (`user1` and `user2`)
- `user1` handles provisioning and sanity checks
- `user2` executes validation workflows through a single stable entrypoint (the CLI)

This separation helps catch permission and environment assumptions and keeps validation independent from provisioning.

### Why a Go Cobra CLI harness
A CLI harness provides:
- a consistent interface for workflows
- uniform output formatting and exit codes
- a scalable way to add new tests without changing the pipeline shape

---

## Validation Strategy

### Deterministic pass/fail
Each workflow is executed by the CLI harness, and each command:
- runs one operation
- captures stdout and stderr
- normalizes known non-deterministic fields if needed (timestamps, ids)
- compares against expected results
- returns a non-zero exit code on mismatch

### Expected outputs
Expected results can be stored as:
- golden files under `ci/workflows/expected/`, or
- structured expectations (recommended long term), such as JSON schemas or key-value assertions

---

## Failure Handling and Debuggability

Failures are easy to localize by stage:
- **Boot failures:** QEMU logs, VM console output
- **Provisioning failures:** setup logs, missing dependency checks
- **Workflow failures:** command traces, service logs
- **Assertion failures:** expected vs actual diffs

Recommended CI artifacts:
- VM console logs
- per-stage logs
- diffs when output mismatch occurs
- metadata (commit hash, pipeline id)

---

## Extending the Test Suite

To add a new workflow:
1. Add a new scenario definition under `ci/workflows/scenarios/`
2. Add expected output under `ci/workflows/expected/`
3. Add a CLI subcommand or extend an existing command:
   - parse scenario inputs
   - run the operation
   - emit stable output and exit codes
4. Register the scenario in `ci/workflows/run_workflows.sh`

Recommended practices:
- keep provisioning logic out of validation logic
- make CLI outputs stable and machine-parseable
- include clear error messages and diffs on mismatch

---

## Security and Public-Repo Notes

This public repo intentionally omits:
- organization names and internal systems
- internal URLs and artifact endpoints
- proprietary command syntax and sensitive config values
- real VM images and production credentials

If you adapt this pattern:
- inject secrets via CI variables or secret managers
- store sanitized logs only
- keep images and build artifacts access-controlled

---

## Results

This approach enabled:
- repeatable VM-based validation in CI
- deterministic pass/fail signals via CLI-driven assertions
- reduced environment drift through versioned artifacts
- faster debugging via stage-specific logs and clear failure boundaries

---

## What I Would Improve Next

- move output matching toward structured assertions (JSON schemas or typed checks)
- add parallelization for independent workflows
- add a lightweight test summary (per workflow status, artifact links, metadata)
- improve normalization for non-deterministic output fields

---

## Resume Bullets

- Designed a CI pipeline that boots and validates a Linux VM in QEMU using a deterministic kernel + initrd approach, enabling repeatable system-level testing.
- Built a Go (Cobra) CLI harness for workflow execution and output validation, producing clear pass/fail signals and reducing manual verification.
- Implemented artifact versioning for VM images and logs to support reproducibility and faster debugging across CI runs.

---

## License

MIT License (recommended for a public case-study repo). See `LICENSE` for details.
