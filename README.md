# Portage

**Agent-driven EKS → GKE migrations. PSO-grade outcomes, no PSO required.**

Portage is an open-source library of Skills 2.0 — modular, composable agent prompts — that automate the work a Professional Services engagement would normally do to move a Kubernetes estate from Amazon EKS to Google Kubernetes Engine. Drop the skills into Claude (or any agent runtime that understands the [Skill format](docs/architecture.md#the-skill-format)), point them at a target environment, and they walk through discovery, design, translation, cutover, and day‑2 with you.

> A *portage* is the act of carrying a boat overland between two waterways. You put in on one side, you take out on the other, and in between you carry the load on your shoulders. That's what this is.

---

## Why Portage exists

Migrating a non-trivial EKS estate to GKE is a six-figure, six-month engagement at most consultancies. The work is high-leverage but highly repetitive: every migration re-discovers the same translation patterns (IRSA → Workload Identity, ALB → Gateway API, EBS → Persistent Disk, ECR → Artifact Registry, CloudWatch → Cloud Operations) and re-writes the same runbooks against the same cutover failure modes.

Portage encodes that pattern library as agent skills. An SRE with a coding agent and Portage installed can:

- inventory an EKS account in an afternoon instead of two weeks of interviews
- produce a graded readiness report with named blockers and a phased plan
- translate manifests, Helm charts, and Terraform modules with auditable diffs
- generate a cutover runbook tailored to their actual workloads, not a template
- execute weighted-traffic cutovers and rollbacks with validated gates
- finish with a hardened, FinOps-tuned, observable GKE estate

This is not a wizard. It is a library of pattern-aware agent prompts that an SRE or platform team uses interactively.

## What's in the box

```
portage/
├── skills/                    # 14 production-ready skills (the core library)
│   ├── portage-orchestrator/      # Phase 0: program manager
│   ├── eks-discovery/             # Phase 1: inventory
│   ├── migration-assessment/      # Phase 1: readiness scorecard
│   ├── gke-landing-zone/          # Phase 2: target design
│   ├── network-translation/       # Phase 2: VPC, Gateway, DNS
│   ├── identity-translation/      # Phase 2: IRSA → Workload Identity
│   ├── workload-translation/      # Phase 3: manifests, Helm, kustomize
│   ├── storage-translation/       # Phase 3: EBS → PD, EFS → Filestore
│   ├── registry-migration/        # Phase 3: ECR → Artifact Registry
│   ├── observability-translation/ # Phase 3: CW → Cloud Ops, Prometheus
│   ├── data-migration/            # Phase 4: RDS, secrets, stateful
│   ├── traffic-cutover/           # Phase 4: weighted DNS, blue/green
│   ├── rollback-playbook/         # Phase 4: bail-out paths
│   └── post-migration-ops/        # Phase 5: day-2, FinOps, hardening
├── runbooks/                  # Phase-by-phase human-readable runbooks
├── reference/
│   ├── sources.md             # Canonical external references per skill
│   ├── lessons-from-the-field.md  # Citable practitioner war stories
│   ├── service-mapping.md     # AWS↔GCP service map
│   ├── api-translation.md     # Annotation-by-annotation map
│   └── terraform/             # Reusable HCL modules
├── templates/                 # Readiness report, migration plan, postmortem
├── examples/                  # End-to-end walkthrough on a sample app
└── docs/                      # Architecture, glossary, ADRs
```

### Two reference documents worth reading on their own

- **[reference/sources.md](reference/sources.md)** — the canonical external references the skills cite, organized A–L. The map of cloud.google.com / kubernetes.io / sre.google / CNCF pages each skill leans on.
- **[reference/lessons-from-the-field.md](reference/lessons-from-the-field.md)** — a citable knowledge base of real-world failure modes from EKS↔GKE migrations and adjacent moves. Every entry cites a verifiable practitioner source (HN, Reddit, engineering blogs, GitHub issues, conference talks), names the owning Portage skill, and rates severity. Read the severity-3 items before kicking off a migration.

## Quickstart

### 1. Install

Portage skills are plain folders with a `SKILL.md` and (optionally) helper assets. To use them with Gemini CLI:

```bash
git clone https://github.com/rohinikrishna05/portage.git ~/.gemini/skills/portage
```
To use them with Antigravity:
```bash
git clone https://github.com/rohinikrishna05/portage.git <workspace-root>/.agents/skills/portage/
```

Or vendor the skills you want into your team's plugin marketplace. See [docs/architecture.md](docs/architecture.md) for the skill loader contract.

### 2. Kick off a migration

Open an agent session in your workspace, then:

```
> Use the portage-orchestrator skill to plan a migration of my EKS estate to GKE.
> The source account is 123456789012, the target GCP org is example.com,
> and we have a 90-day window. I'll tell you what I want migrated first.
```

The orchestrator chains the 14 sub-skills, asks clarifying questions where the answer is irreducibly human, and produces artifacts under `./portage-output/<run-id>/`.

### 3. Review and ratchet

Every skill produces a deliverable: a discovery report, a Terraform plan, a translated chart, a runbook. Nothing is applied without your sign-off. The orchestrator's job is to keep the migration *moving*, not to surprise you.

## Design principles

1. **Auditable, not automatic.** Every artifact is human-readable Markdown, YAML, or HCL. Every change is a diff. No black boxes.
2. **Skills compose.** Each skill has typed inputs and outputs. The orchestrator chains them; you can also run any skill standalone.
3. **The agent narrates the work.** Each skill emits a structured trace: what it found, what it decided, what it would do next. That trace is the deliverable a PSO consultant would have written.
4. **Escalation is a first-class outcome.** When a skill encounters something it can't safely translate (a custom CNI, a cross-account peering, a privileged DaemonSet), it stops and writes an *escalation note* — what it saw, why it stopped, what the human needs to decide.
5. **Production-grade defaults.** Every recommendation matches the public GKE hardening guidance: regional clusters, Workload Identity, Dataplane V2, private nodes, Binary Authorization where applicable, separate fleets per environment.

## What Portage does not do

- It does not run privileged commands without confirmation.
- It does not migrate data it does not understand. RDS → Cloud SQL homogeneous-engine moves are scripted; heterogeneous data store moves (DynamoDB → Spanner, Aurora → AlloyDB with engine change) are scoped, planned, and handed back to a human.
- It does not invent compliance. If you have HIPAA, PCI, FedRAMP, or sovereignty constraints, those are inputs to the assessment skill — Portage will surface incompatibilities, not silently paper over them.
- It does not replace your platform team. It replaces the *first 80%* of a PSO engagement so your platform team can spend its time on the last 20% — the part that's actually unique to your business.

## Contributing

Portage is intentionally small. Fourteen skills, written carefully, beat fifty skills that drift. Read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a PR. The bar for new skills is *can two SREs at two different companies execute this end-to-end without our help*.

## License

Apache 2.0. See [LICENSE](LICENSE).

## Roadmap

The library is the substrate. The runnable agent tool that wraps it is documented in [ROADMAP.md](ROADMAP.md):

- **Stage 1 (today)** — skills + Claude Code / Cowork / any compliant agent runtime.
- **Stage 2** — `portage` CLI on the Claude Agent SDK (typed artifacts, cross-session resume, executable cloud-side actions with confirmation).
- **Stage 3** — multi-agent control plane with MCP servers per skill, real-time dashboard, parallel execution.
- **Stage 4 (optional)** — hosted service for teams that want PSO outcomes without standing up the control plane.

The skills themselves do not change as the runtime evolves. They are the stable substrate.

## Status

Pre-1.0. Skills are reviewed against real migrations as they happen — expect tightening of the rough edges, especially around stateful workload patterns and multi-cluster topologies. Friction logs and PRs welcome.
