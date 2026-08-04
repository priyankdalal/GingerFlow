# GingerFlow

GingerFlow is a desktop visual automation platform for building, running, and monitoring workflows without writing orchestration code.

## GingerFlow Beta Status

- Current state: active development
- Release phase: Beta only
- Intended auth direction: support for major providers
- Current usage intent: testing and evaluation only

> Safety notice: The current GingerFlow Beta build is intended for testing purpose only. Incorrect workflow configuration or unsafe automation actions may affect files, services, or system state. Use in controlled environments and at your own risk.

## Motive

Modern teams spend too much time stitching scripts, APIs, files, and services by hand.
GingerFlow exists to make automation faster to build, easier to understand, and safer to operate.

## Objective

- Turn repetitive operational steps into reusable visual workflows.
- Reduce manual errors in data movement and system integration tasks.
- Help teams monitor execution status and performance in real time.
- Make automation accessible to both developers and technical operations users.

## What You Can Do

- Design workflows visually using drag-and-connect nodes.
- Connect APIs, files, data transformations, utility actions, and logic blocks.
- Run workflows interactively and inspect execution timelines.
- Observe runtime behavior with status, duration, and resource insights.
- Extend capabilities through installable plugins.

## Areas of Usage

- Data preparation and transformation pipelines.
- API orchestration and service integration.
- File processing and batch operations.
- Automation for QA, DevOps, and support workflows.
- Internal tooling for repeatable operational tasks.

## Why Teams Use GingerFlow

- Faster automation delivery compared to script-only approaches.
- Clear visual flow improves collaboration and onboarding.
- Reusable building blocks reduce duplicate implementation effort.
- Runtime visibility improves troubleshooting and confidence.

## Why GingerFlow Is a Desktop App (Not a Web App)

GingerFlow is intentionally desktop-first because automation often touches sensitive systems, local files, credentials, and production operations. For this use case, security comes first, then execution speed and scale.

### Security Before Execution

- Automation can change real systems quickly, so preventing unsafe access is more important than maximizing remote convenience.
- A desktop boundary reduces unnecessary exposure to internet-facing attack surfaces.
- Local execution makes permission scope clearer and easier to audit on a single machine.

### Why We Do Not Store Data Online by Default

- Workflow definitions may contain business logic, endpoint details, transformation rules, and operational intent.
- Runtime data may include internal records, file paths, metadata, and service responses.
- Keeping data local-by-default reduces data residency risk, third-party retention risk, and accidental cloud leakage.
- Teams can still version workflow files in their own controlled repositories without requiring a shared vendor cloud.

### Web App Security Concerns for Automation Platforms

- Browser sessions are exposed to risks such as token theft, session hijacking, and cross-site scripting vectors.
- Multi-tenant cloud storage increases blast radius if account or infrastructure boundaries are misconfigured.
- Credential handling in always-online architectures can create persistent high-value targets.
- Public endpoint dependence increases risk from supply-chain compromises and service outages outside user control.

### Web Limitations for Heavy Automation

- Browser sandboxes restrict direct system access to local files, background processes, and OS-level resources.
- Long-running, high-throughput automation can be constrained by browser process limits and tab lifecycle behavior.
- Network interruptions and browser refresh cycles can disrupt in-progress orchestration.
- Advanced plugin/runtime integration is harder when execution is constrained to browser capability and policy.

### Desktop Robustness for Real Operations

- Stable local runtime with stronger control over processes, memory, and resource management.
- Better support for long-running workflows, local dependencies, and system-level integrations.
- Predictable execution even in restricted or partially offline environments.
- Easier enterprise hardening with host controls, endpoint security tooling, and local policy enforcement.

## Authentication

GingerFlow is being built to support sign-in with major providers.
For local evaluation, the sign-in screen also includes a Test App option that enables a temporary session for the current run only.

## Binary Releases

This repository hosts prebuilt binaries for easy usage.

- Download the latest release package from the Releases section.
- Extract the package.
- Launch the application using the provided startup script or executable.

No separate language runtime setup is required for standard usage of the release package.

## Workflow Files

Workflows are saved as fg files and can be shared, versioned, and reused across teams.

## Vision

GingerFlow aims to become a practical automation workspace where teams can design, execute, and evolve real production workflows with clarity and control.

## Licensing

Licensing is currently under progress and will be finalized in an upcoming update.
