## Context

[Oakestra](https://github.com/oakestra/oakestra) is an open-source, lightweight hierarchical orchestration framework for Edge Computing: a Root Orchestrator delegates to per-cluster orchestrators, which schedule workloads onto heterogeneous worker nodes — a design that tolerates the unreliable links and constrained devices that make stock Kubernetes a poor fit at the edge.

I contributed to the core platform and its delivery pipeline between 2024 and 2025. The full list is on GitHub: [my commits on oakestra/oakestra](https://github.com/oakestra/oakestra/commits?author=TheDarkPyotr).

---

## Contributions

**Cluster registration over gRPC** ([#329](https://github.com/oakestra/oakestra/pull/329)). Migrated the registration handshake between cluster orchestrators and the Root from WebSocket to gRPC — a typed, contract-first interface in place of an ad-hoc message exchange, on the path every cluster crosses to join the system.

**Alarming & logs monitoring service** ([#289](https://github.com/oakestra/oakestra/pull/289)). A monitoring component for the orchestration plane, surfacing alarms and centralising logs from the distributed components. Also a follow-up fix disabling the `MULTI_STATUS` flag ([#354](https://github.com/oakestra/oakestra/pull/354)).

**CI/CD and developer experience.** A series of changes to how Oakestra is built and tested: container image building triggered during PR development ([#314](https://github.com/oakestra/oakestra/pull/314)), Super-Linter adoption with language-specific linter configuration ([#350](https://github.com/oakestra/oakestra/pull/350), [#351](https://github.com/oakestra/oakestra/pull/351)), automatic propagation of the `getstarted.sh` setup script across the organisation's repositories ([#317](https://github.com/oakestra/oakestra/pull/317)), and improved startup scripts ([#308](https://github.com/oakestra/oakestra/pull/308)).

**Testbed integration** ([#360](https://github.com/oakestra/oakestra/pull/360)). A GitHub Action letting maintainers trigger custom testbed executions directly from the Oakestra repository — the bridge between the main repo and the project below.

---

## ⚙️ Oakestra Virtual Testbed

The largest piece of my work lives in its own repository: the [Oakestra Virtual Testbed](https://github.com/oakestra/awx-testbed), a CI-integrated tool that gives maintainers and contributors automated end-to-end testing of the whole platform — component deployment, configuration, application deployment, and result assessment — across predefined and customizable scenarios.

Scenarios are expressed as **Topology Descriptors**: JSON files that extend Oakestra's deployment descriptors with meta-information about the infrastructure to provision — how many clusters, how many workers per cluster, which applications land where, and an `expected_output` per microservice that the testbed checks against container logs as a basic health check. A descriptor selects one of three deployment modes: 1-DOC (one device, one cluster), M-DOC (M devices, one cluster), or MDNC (M devices, N clusters).

The testbed runs in two modes, implemented on an Ansible/AWX pipeline:

- **Custom Execution** 🔬 — maintainers trigger a GitHub Action against specific branches and commits of Oakestra and Oakestra-Net (forked repositories included), with the chosen topology, and follow the run on the AWX dashboard.
- **Oneshot Execution** 🎯 — wired into CI: every approved PR review runs a predefined set of scenarios and reports a pass/fail status back to the PR, without ever exposing repository secrets to forks.

Under the hood it's a six-phase Ansible pipeline — topology validation, host provisioning, mode-specific deployment, component tests, application deployment checks, and cleanup — but the point is the interface: a reviewer approves a PR, and a few minutes later there's a verdict on whether the change still deploys and runs across real multi-node topologies.
