# agent-design

1. Executive Summary

As AgentHub evolves to support more advanced Agent workloads, the requirements for execution environments are becoming increasingly diverse.

Traditional short-lived task execution is no longer sufficient to represent all Agent workload patterns. Modern Agents may operate autonomously, execute long-running tasks, access highly restricted enterprise data, or collaborate with multiple other Agents.

These different execution patterns introduce distinct requirements around isolation, security, lifecycle management, persistence, networking, resource governance, and observability.

This document aims to:

* Identify the key Agent workload characteristics and execution patterns.
* Define the corresponding Sandbox capabilities required to support these workloads.
* Clarify the integration responsibilities between AgentHub Runtime and the Sandbox platform.
* Identify potential capability gaps and areas requiring further technical alignment.

Key Principle

The Sandbox platform should not be designed around a single Agent execution model. Instead, it should provide configurable isolation, lifecycle management, resource governance, and security capabilities to accommodate different Agent workload profiles.

2. Agent Workload Classification

We identify five representative Agent workload profiles.

2.1 Standard Task-oriented Agent

Characteristics

* Executes relatively short-lived tasks.
* Typically follows a predefined workflow or user instruction.
* Invokes tools and performs code execution when required.
* Execution normally completes within seconds or minutes.
* Limited requirements for persistent execution state.

Sandbox Requirements

* Fast provisioning and startup.
* Basic process and filesystem isolation.
* Configurable CPU and memory limits.
* Controlled network access.
* Automatic cleanup after task completion.

Typical Use Cases

* Data transformation.
* Document processing.
* Code generation and execution.
* NL2SQL and analytical tasks.

2.2 Autonomous Agent

Characteristics

* Operates with a higher degree of autonomy.
* Performs planning, reasoning, and multi-step execution.
* Can independently invoke tools and execute commands.
* May dynamically determine its next actions based on intermediate results.
* Potentially executes untrusted or dynamically generated code.

Sandbox Requirements

* Strong execution isolation.
* Fine-grained network egress control.
* Restricted filesystem and process access.
* Controlled tool and command execution.
* Resource quotas and execution limits.
* Ability to terminate or interrupt execution.
* Comprehensive execution audit and observability.

Key Consideration

Autonomous Agents introduce a larger execution attack surface because their actions are not always fully predictable in advance.

The Sandbox should enforce security boundaries independently of the Agent’s reasoning and decision-making capabilities.

2.3 Long-running Agent

Characteristics

* Executes tasks that may last from several minutes to hours or potentially days.
* May involve multiple execution stages and external dependencies.
* Needs to maintain intermediate state and working files.
* May be interrupted by infrastructure events or resource constraints.
* Requires the ability to resume execution without restarting the entire workflow.

Sandbox Requirements

* Persistent execution workspace.
* Checkpoint and snapshot capabilities.
* Reliable state persistence and restoration.
* Durable storage for files and intermediate artifacts.
* Support for graceful termination and recovery.
* Lifecycle management independent of individual execution requests.
* Resource management for long-running workloads.

Key Consideration

The Sandbox lifecycle should not necessarily be coupled to a single Agent query or HTTP request.

The execution environment should support suspension, recovery, and restoration while preserving the necessary execution state.

2.4 Sensitive Data / Highly Restricted Data Agent

Characteristics

* Accesses confidential or highly restricted enterprise information.
* May interact with internal databases, APIs, or regulated data sources.
* Requires strict identity verification and access control.
* Must comply with enterprise security and data governance policies.
* May require different security controls depending on data classification.

Sandbox Requirements

* Strong workload isolation, potentially using MicroVM-based isolation.
* Identity-aware access control.
* Secure credential injection and short-lived credentials.
* Fine-grained network egress policies.
* Protection against credential and sensitive data leakage.
* Controlled data access and filesystem boundaries.
* Comprehensive audit logging.
* Policy enforcement independent of Agent application logic.

Key Consideration

Access to sensitive enterprise data should be governed by identity, authorization, and data classification policies rather than relying solely on the isolation level of the Sandbox.

The Sandbox should provide the enforcement mechanisms required to prevent unauthorized access and data exfiltration.

2.5 Multi-Agent / Collaborative Agent Workload

Characteristics

* Multiple Agents collaborate to complete a larger task.
* Each Agent may have different responsibilities and execution privileges.
* Agents may execute concurrently.
* Some workloads require communication or artifact sharing between Agents.
* Individual Agents may have different security and resource requirements.

Sandbox Requirements

* Isolation between individual Actors.
* Configurable CPU, memory, and disk quotas.
* Concurrency management and execution scheduling.
* Controlled inter-Agent communication.
* Secure artifact and workspace sharing.
* Independent lifecycle management.
* Isolation of credentials and execution contexts.

Key Consideration

The platform should support collaboration without introducing unnecessary trust relationships between individual Agents.

⸻

3. Cross-cutting Sandbox Capability Requirements

Beyond individual workload profiles, several capabilities are common requirements across different Agent execution models.

3.1 Isolation and Execution Security

The Sandbox should provide configurable isolation levels based on workload risk and security classification.

Potential implementation approaches:

* Container-based isolation for relatively trusted workloads.
* gVisor for additional kernel-level isolation.
* MicroVM-based isolation for workloads requiring stronger security boundaries.

The selection should consider:

* Security isolation strength.
* Startup latency.
* Runtime overhead.
* Resource efficiency.
* Operational complexity.
* Compatibility with existing infrastructure.

The platform should allow AgentHub to select or request an appropriate isolation profile without embedding low-level infrastructure decisions into individual Agent implementations.

3.2 Lifecycle Management

The Sandbox should support a complete execution lifecycle:

* Create and initialize.
* Start and execute.
* Suspend or pause where supported.
* Checkpoint and snapshot.
* Restore and resume.
* Terminate and clean up.

A key requirement is to support lifecycle operations independently of individual Agent queries.

3.3 Persistence and Storage

Different Agent workloads have different persistence requirements.

We propose separating storage into three conceptual categories:

Ephemeral Workspace

* Temporary files and execution artifacts.
* Can be cleaned up after execution.
* Suitable for short-lived tasks.

Checkpoint / Snapshot State

* Runtime state required for recovery and resumption.
* Should support reliable snapshot and restore.
* Requires consistency between execution state and persisted files.

Durable Workspace

* Files and artifacts that need to survive Sandbox termination.
* Should support persistence through external storage.
* Should provide appropriate access control and lifecycle management.

The platform should clearly define the consistency, recovery, and retention guarantees for each storage category.

3.4 Identity and Credential Management

Agent execution may require access to enterprise services using different identities and permissions.

Required capabilities include:

* Actor-level identity.
* Secure credential injection.
* Short-lived token issuance and exchange.
* Credential isolation between Actors.
* Credential rotation and expiration.
* No persistence of sensitive credentials in durable workspace or snapshots.
* Auditable credential access.

Credentials should be retrieved dynamically based on the authenticated execution identity and required authorization context.

3.5 Network and Egress Control

Agents may need to access internal enterprise APIs, external services, or third-party tools.

Required capabilities include:

* Configurable network access policies.
* FQDN-based egress restrictions.
* IP and port-level restrictions.
* Identity-aware network policies where applicable.
* Centralized egress enforcement.
* Request-level observability and audit.
* Protection against unauthorized data exfiltration.

Where necessary, the platform should support an egress gateway or proxy-based architecture to enforce network policies consistently.

3.6 Resource Governance and Concurrency

The Sandbox should provide resource isolation and predictable execution limits.

Required capabilities include:

* CPU and memory quotas.
* Disk and storage limits.
* Execution timeout and termination policies.
* Per-Actor resource limits.
* Per-workload concurrency limits.
* Fair resource allocation.
* Monitoring of resource utilization.

Resource governance should operate at both individual Actor and overall workload levels.

3.7 Observability and Audit

The platform should provide sufficient visibility for troubleshooting, operations, and security investigations.

Required capabilities include:

* Execution lifecycle events.
* Sandbox provisioning and termination logs.
* Resource utilization metrics.
* Network access audit.
* Credential access audit.
* Snapshot and restoration status.
* Failure and termination reasons.
* Correlation between AgentHub requests, Actors, and Sandbox executions.

⸻

4. Agent Workload to Sandbox Capability Mapping

The following table summarizes the key requirements.

Capability	Standard Agent	Autonomous Agent	Long-running Agent	Sensitive Data Agent	Multi-Agent
Basic isolation	Required	Required	Required	Required	Required
Strong isolation	Optional	Recommended	Risk-based	Strongly recommended	Risk-based
Network egress control	Required	Required	Required	Strict	Required
Credential management	Basic	Advanced	Advanced	Strict	Per-Actor
Persistent workspace	Optional	Optional	Required	Controlled	Required
Checkpoint / Restore	Optional	Optional	Required	Recommended	Recommended
Resource quotas	Required	Required	Required	Required	Required
Execution termination	Required	Required	Required	Required	Required
Audit and observability	Required	Required	Required	Strict	Required
Inter-Agent communication	Not required	Optional	Optional	Restricted	Required

Note: These are proposed baseline requirements. The final capability level should be determined by workload risk, data classification, and operational requirements.

⸻

5. Proposed AgentHub – Sandbox Integration Architecture

AgentHub Runtime and the Sandbox platform should maintain clear separation of responsibilities.

5.1 High-level Architecture

+-----------------------------------------------+
|                  AgentHub                     |
|                                               |
|  +-----------------------------------------+  |
|  |           Agent Applications            |  |
|  |                                         |  |
|  |  Standard | Autonomous | Long-running   |  |
|  |  Sensitive Data | Multi-Agent           |  |
|  +--------------------+--------------------+  |
|                       |                       |
|  +--------------------v--------------------+  |
|  |             Agent Runtime               |  |
|  |                                         |  |
|  |  - Agent Execution Orchestration        |  |
|  |  - Actor / Session Management           |  |
|  |  - Workload Profile Selection           |  |
|  |  - Credential Integration               |  |
|  |  - Checkpoint / Snapshot Coordination   |  |
|  |  - Resource and Concurrency Management  |  |
|  +--------------------+--------------------+  |
+-----------------------|-----------------------+
                        |
                  Sandbox API
                        |
+-----------------------v-----------------------+
|               Sandbox Platform               |
|                                               |
|  +-----------------------------------------+  |
|  |          Sandbox Control Plane          |  |
|  |  - Provisioning & Lifecycle             |  |
|  |  - Isolation Profile                    |  |
|  |  - Resource Policy                      |  |
|  |  - Security Policy                      |  |
|  +--------------------+--------------------+  |
|                       |                       |
|  +--------------------v--------------------+  |
|  |          Sandbox Execution Layer        |  |
|  |                                         |  |
|  |  Container | gVisor | MicroVM           |  |
|  |                                         |  |
|  +--------------------+--------------------+  |
|                       |                       |
|  +--------------------v--------------------+  |
|  |       Storage / Network / Security      |  |
|  |                                         |  |
|  |  Persistent Storage                     |  |
|  |  Snapshot & Restore                     |  |
|  |  Network Egress Gateway                 |  |
|  |  Identity & Credential Integration      |  |
|  |  Monitoring & Audit                     |  |
|  +-----------------------------------------+  |
+-----------------------------------------------+

5.2 Responsibility Boundaries

AgentHub Runtime Responsibilities

* Understand Agent workload characteristics.
* Define and select execution profiles.
* Manage Actor and Session lifecycle.
* Coordinate Agent execution and recovery.
* Integrate identity and credential services.
* Define execution resource requirements.
* Provide execution context and required metadata to Sandbox.

Sandbox Platform Responsibilities

* Provision and manage isolated execution environments.
* Enforce isolation and security policies.
* Manage execution resource boundaries.
* Provide storage and snapshot capabilities.
* Enforce network access policies.
* Provide execution lifecycle APIs.
* Expose operational metrics and security audit events.

Joint Responsibilities

* Define workload and security profiles.
* Agree on API contracts and lifecycle semantics.
* Establish failure recovery and cleanup strategies.
* Define performance and scalability targets.
* Validate security boundaries and compliance requirements.

⸻

6. Potential Capability Gaps and Areas for Alignment

The following areas should be reviewed with the Sandbox team.

Area	Key Questions / Potential Gaps	Priority
Isolation	Can the platform support configurable gVisor and MicroVM execution profiles?	High
Provisioning	What are the startup latency and provisioning scalability targets?	High
Lifecycle	Can execution environments survive individual request termination?	High
Snapshot / Restore	Can snapshots be exported, persisted externally, and restored reliably?	High
Storage	Can durable workspace be decoupled from ephemeral execution storage?	High
Credential	Can credentials be dynamically injected per Actor without persisting secrets?	High
Network	Can FQDN-level egress policies be centrally enforced?	High
Concurrency	Can the platform support high Actor concurrency with predictable resource quotas?	High
Observability	Can execution events and security audit logs be correlated with AgentHub requests?	Medium
Multi-Agent	Can Actors communicate or share artifacts through controlled mechanisms?	Medium

These items are discussion points rather than confirmed platform limitations. Detailed validation is required against the current Sandbox implementation.

⸻

7. Proposed Next Steps

We propose the following activities to align AgentHub and Sandbox development.

Phase 1 – Requirement Alignment

* Validate the proposed Agent workload classifications.
* Confirm security and isolation expectations.
* Identify mandatory versus optional capabilities.
* Agree on common workload profiles.

Phase 2 – Technical Assessment

* Review current Sandbox capabilities against the requirements.
* Assess gVisor and MicroVM deployment models.
* Validate lifecycle, persistence, and snapshot mechanisms.
* Review network and credential integration.

Phase 3 – Integration Design

* Define AgentHub-to-Sandbox API contracts.
* Define execution profile and resource policy schemas.
* Align Actor lifecycle and recovery semantics.
* Establish observability and audit integration.

Phase 4 – Proof of Concept
Validate representative scenarios:

1. Autonomous Agent executing dynamically generated code with controlled egress.
2. Long-running Agent with checkpoint, termination, and restoration.
3. Sensitive Data Agent accessing restricted enterprise services using short-lived credentials.
4. Multiple concurrent Actors with independent resource and security policies.

⸻

8. Key Discussion Questions

The following questions are proposed for the next cross-team discussion.

1. What isolation levels should be supported as standard Sandbox execution profiles?
2. Can the Sandbox support both short-lived execution and persistent Actor workloads?
3. What are the supported guarantees and limitations of Snapshot / Restore?
4. How should Actor identity and short-lived credentials be integrated into the Sandbox?
5. What level of network egress control can be enforced natively?
6. How should CPU, memory, disk, and concurrency quotas be managed?
7. What are the expected performance and scalability limits under high Actor concurrency?
8. Which capabilities should be owned by AgentHub Runtime, and which should be provided natively by the Sandbox platform?

⸻

9. Conclusion

The evolution of Agent workloads introduces requirements that extend beyond basic isolated code execution.

Autonomous execution, long-running tasks, sensitive enterprise data access, and multi-Agent collaboration each introduce distinct security, lifecycle, and operational challenges.

For AgentHub, the Sandbox should be treated as a foundational execution infrastructure rather than simply a container hosting environment.

A configurable and policy-driven Sandbox architecture will enable AgentHub to support different Agent workload profiles while maintaining consistent security, operational governance, and scalability.

The objective is to establish a common execution foundation that allows AgentHub to evolve its Agent capabilities without compromising enterprise security and operational requirements.
。

我建议把架构图进一步细化，不只是画 AgentHub 到 Sandbox 的简单调用关系，而是突出 Agent Runtime、Agent Substrate、Actor Execution、Sandbox Isolation，以及底层安全和基础设施服务 之间的关系。

这张图要让  和 Sandbox 团队一眼看懂：

* AgentHub 负责 Agent 的编排和生命周期管理。
* Agent Substrate 提供 Agent 执行所需的基础运行环境。
* Sandbox 负责实际执行隔离和安全边界。
* gVisor / MicroVM 是不同等级的隔离执行后端。
* Storage、Credential、Network Egress 是独立的基础设施能力。

1. AgentHub – Agent Substrate – Sandbox 整体架构图

flowchart TB
    U["Users / Enterprise Applications"]
    subgraph AH["AgentHub Platform"]
        APP["Agent Applications<br/>Standard · Autonomous · Long-running<br/>Sensitive Data · Multi-Agent"]
        RT["Agent Runtime<br/>Session & Actor Lifecycle<br/>Workload Profile Selection<br/>Execution Orchestration"]
        APP --> RT
    end
    subgraph SUB["Agent Execution Layer"]
        DA["DeepAgent Harness<br/>Planning · Reasoning · Tool Calling"]
        SK["Skills / Tools / Custom CLI<br/>Git · ZIP · PDF · APIs"]
        DA --> SK
    end
    subgraph AS["Agent Substrate"]
        API["Substrate API / Control Plane"]
        ACT["Actor Management<br/>Provision · Start · Stop · Recover"]
        POLICY["Execution Profile & Policy<br/>Isolation · CPU · Memory · Disk · Network"]
        API --> ACT
        ACT --> POLICY
    end
    subgraph SB["Sandbox Execution Infrastructure"]
        SVC["Sandbox Execution Service"]
        WP["Worker / Execution Environment"]
        subgraph ISO["Isolation Backends"]
            GV["gVisor Sandbox"]
            MV["MicroVM Sandbox<br/>Firecracker / Equivalent"]
        end
        ACT --> SVC
        SVC --> WP
        WP --> GV
        WP --> MV
        GV --> DA
        MV --> DA
    end
    subgraph INFRA["Supporting Infrastructure Services"]
        ST["Storage Services<br/>Ephemeral Workspace<br/>Durable Directory<br/>Snapshot / Restore"]
        CR["Credential Service<br/>Actor Identity<br/>Short-lived Token Exchange"]
        EG["Network Egress Gateway<br/>Envoy / Proxy<br/>FQDN & Destination Policies"]
        OBS["Observability & Audit<br/>Logs · Metrics · Events"]
    end
    U --> APP
    RT --> API
    DA <--> ST
    DA --> CR
    DA --> EG
    SVC --> OBS
    ST --> OBS
    CR --> OBS
    EG --> OBS
    style AH fill:#eaf2ff,stroke:#4785c7,color:#17365d
    style SUB fill:#eaf7ef,stroke:#41966a,color:#174b2c
    style AS fill:#fff5df,stroke:#c99735,color:#68470b
    style SB fill:#f3edff,stroke:#8b6ac5,color:#422c70
    style INFRA fill:#f1f3f5,stroke:#7b8794,color:#303b46

架构图的核心逻辑：

层次	核心职责
AgentHub	Agent 应用、业务编排、Session 管理
Agent Runtime	DeepAgent 执行协调、Actor 生命周期、Workload Profile
Agent Substrate	Actor 创建、执行环境管理、隔离策略下发
Sandbox	实际隔离执行、资源限制、安全边界
Infrastructure Services	存储、凭证、网络出口、监控审计

这里需要特别说明：DeepAgent Harness 是运行在 Actor 执行环境内部的 Agent 执行逻辑；gVisor 和 MicroVM 则是提供隔离的执行后端，不是 Agent Framework。

2. Workload Profile 到 Sandbox Policy 的映射图

我建议再补一张图，因为 Ricky 特意提到 Autonomous、Long-running 和 Highly Restricted Data。这张图能直接解释为什么需要不同的 Sandbox 配置。

flowchart TD
    A["Agent Workload Profile"]
    A --> B["Standard Agent"]
    A --> C["Autonomous Agent"]
    A --> D["Long-running Agent"]
    A --> E["Sensitive Data Agent"]
    A --> F["Multi-Agent Workload"]
    B --> P1["Standard Isolation<br/>Basic Egress Control<br/>Resource Quotas"]
    C --> P2["Strong Isolation<br/>Strict Egress Policy<br/>Execution Limits"]
    D --> P3["Persistent Workspace<br/>Checkpoint / Snapshot<br/>Restore & Recovery"]
    E --> P4["MicroVM Profile<br/>Identity-based Access<br/>Short-lived Credentials<br/>Strict Audit"]
    F --> P5["Per-Actor Isolation<br/>Concurrency Control<br/>Controlled Communication"]
    P1 --> S["Sandbox Policy Engine"]
    P2 --> S
    P3 --> S
    P4 --> S
    P5 --> S
    S --> X["Sandbox Execution Service"]
    X --> G["gVisor"]
    X --> M["MicroVM"]

3. 我建议在图中强调的三个设计原则

原则一：Workload-driven isolation

不要让所有 Agent 默认使用同一种 Sandbox。根据 Workload Profile 和数据敏感等级选择隔离策略，允许 gVisor 和 MicroVM 作为不同执行后端。

原则二：Runtime and Sandbox separation

Agent Runtime 不应该直接管理底层 MicroVM 或 gVisor 的实现细节。Runtime 通过 Agent Substrate 的标准接口提交执行需求，由 Substrate 负责执行环境管理。

原则三：State and execution separation

长期运行的 Agent 不应该依赖 Sandbox 实例本身保存所有状态。需要将：

* Runtime Checkpoint
* Durable Workspace
* Sandbox Snapshot

分别管理，并定义一致性及恢复机制。

4. 放进原文档的位置

我建议调整为：

* Section 5.1：Overall Architecture（第一张图）
* Section 5.2：Workload Profile & Policy Mapping（第二张图）
* Section 5.3：Responsibility Boundaries（Runtime / Substrate / Sandbox 职责）
* Section 5.4：Integration Flow（后续可以进一步补充 API 调用时序）

一个重要提醒： 这两张图是目标逻辑架构，不代表 Agent Substrate 当前已经原生支持所有这些集成能力。特别是 Credential Injection、Envoy Egress、Snapshot Export，需要标注为 integration requirements 或 capabilities to be validated。

这样既能体现你们 AgentHub 的整体设计方向，也不会在下周会议上过早承诺 Sandbox 团队尚未交付的能力。
