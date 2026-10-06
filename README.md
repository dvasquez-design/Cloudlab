# CloudLab

A hybrid lab (on-premise + AWS) where I learn cloud, networking and security by building, breaking, investigating and documenting real environments.

> 🚧 **Work in progress.** This README describes the goals and structure of the project. Content is being published environment by environment.

## What it is

CloudLab is a personal lab built around one idea: virtual machines running on my own hardware (KVM) connected securely to infrastructure in AWS, so I can practice the way real hybrid environments work.

Each use case is an **independent environment** in its own subdirectory, with its own documentation, so the lab can grow without turning into a single tangled project.

## Objectives

- **Learn by doing.** Understand how larger systems work by starting small: first a network, then a service, then how they connect and how they are monitored.
- **Hybrid connectivity.** Connect on-premise VMs with an AWS VPC through a WireGuard tunnel started from the local side, with no inbound ports exposed.
- **Infrastructure as code.** Define the AWS side with CloudFormation so every environment can be deployed, destroyed and redeployed reproducibly.
- **Security from both sides.** Practice red team and blue team in a controlled environment, with monitoring, detection and a written report for each exercise.
- **Document everything.** Every environment records not only what worked, but the decisions, the problems found and how they were solved.
- **Share reusable templates.** Publish the templates so others can replicate the environments, with a static web portfolio planned to present them.

## Architecture (high level)

```
  LOCAL NETWORK (on-premise)                         AWS
 +---------------------------+              +---------------------------+
 |  KVM host                 |              |  VPC                      |
 |   +-------+  +-------+    |   WireGuard  |   +--------------------+  |
 |   | VM    |  | VM    |    |   tunnel     |   | Tools / monitoring |  |
 |   +-------+  +-------+    |<============>|   +--------------------+  |
 |                           |  started     |   deployed with           |
 |  (no inbound ports open)  |  from local  |   CloudFormation          |
 +---------------------------+              +---------------------------+
```

## Environments

| Environment | Status | Description |
|---|---|---|
| [`pentestlab/`](./pentestlab) | 🟡 In progress | Analysis of vulnerable VMs as red team and blue team, with a monitoring stack in AWS connected to the on-premise VMs. |
| More environments | 🔜 Planned | They will be added as independent subdirectories as they are ready. |

### PentestLab status

- ✅ Mr. Robot: full analysis published
- 🔄 Vulnerability analysis with Nessus
- 🔄 Final report
- 🔄 Purple-team approach: cross what the attack sees with what the monitoring detects

## Repository structure

```
Cloudlab/
├── README.md
└── pentestlab/
```

More subdirectories will be added with each new environment.

## Principles

- Everything is deployed in a controlled environment I own.
- Each exercise is documented with its reasoning, not just its result.
- Nothing is published as finished until it is.

## Author

Diego Vásquez · [GitHub](https://github.com/dvasquez-design) · [LinkedIn](https://linkedin.com/in/diego-vasquez-cloud)
