---
name: project-skill-planner
description: Plans and creates a tailored suite of skills for a new project or task by interviewing the user about their goal. Use when someone describes something they're about to build, migrate, or set up — e.g. "I'm going to create a new AWS service", "we need to migrate this app", "I'm building a data pipeline", "I want to set up a CI/CD pipeline", "I'm starting a new feature". Not for single-task requests where one existing skill already covers the work.
---

# Project Skill Planner

Interviews the user about a new project or task, then designs and scaffolds a tailored suite of skills to support it — one skill per phase or concern, each with a clear trigger and workflow.

## When to use

- User describes something they're about to start: a new service, migration, pipeline, feature, or integration
- The work spans multiple phases (understand → design → build → ship) and no single existing skill covers it
- User says things like "I'm going to...", "we need to build...", "I want to set up...", "help me plan..."

## When NOT to use

- The user's request is fully covered by a single existing skill (e.g. "create a Word doc", "fix this flaky test")
- The user just wants to do the task now, not build reusable skills for it

## Workflow

### Phase 1 — Interview (adaptive, 3–5 questions)

Ask one question at a time using `AskUserQuestion`. Don't front-load all questions at once.

**Q1 (always):** What are you going to build or do? (open, one sentence)

**Q2 (always):** Is this greenfield or does a legacy system exist that needs to be understood or replaced?

**Q3 (branch on domain):** Based on Q1, ask the most relevant follow-up:
- *Cloud/infra:* What's the target stack? (AWS services, deployment model — don't assume IaC tooling)
- *Software feature:* What's the tech stack and where does this live in the existing system?
- *Data/ML:* What are the data sources and sinks? What's the output (model, report, API)?
- *General:* What does "done" look like? What are the key deliverables?

**Q4 (conditional):** Do you have requirements, specs, or examples you can share or describe?

**Q5 (conditional, if legacy mentioned):** Can you point me at the existing code or describe how the current implementation works?

**Q6 (REQUIRED if legacy mentioned):** Why a new system instead of evolving the legacy one? Probe for the forcing function — what's broken, limiting, or blocking that the legacy can't address? What alternatives (refactor in place, extend, leave alone) were considered and ruled out? Who is driving the replacement (engineering, product, compliance, cost)? Is there a deadline or external trigger?

Do not move on until you have a concrete answer. "It's old" or "we want to modernise" is not sufficient — push for the specific pain or constraint. If the user genuinely doesn't know yet, capture that explicitly in the profile as `rationale: unclear` and flag it as a risk in Phase 4.

**Q7 (conditional, if legacy mentioned):** What's explicitly out of scope for the replacement? Which legacy behaviours are being intentionally dropped, and which must be preserved exactly? This prevents the new design from silently inheriting every legacy quirk.

Stop when you have enough to write a project profile. Don't over-interview — but never skip Q6 when legacy exists.

### Phase 2 — Project Profile

Internally synthesise answers into a short profile:
- **Goal:** one sentence
- **Domain:** (aws-infra / software-feature / data-ml / general)
- **Has legacy:** yes/no
- **Replacement rationale:** the forcing function (or `unclear` if Q6 didn't yield one) — only present when legacy exists
- **Out-of-scope / preserved behaviours:** from Q7 — only present when legacy exists
- **Stack:** key technologies
- **Has requirements:** yes/no

### Phase 3 — Skill Suite Design

Design 3–7 skills. Group them into phases. For each skill, define:
- **Name** (kebab-case, verb-led where possible)
- **Description** (one sentence + trigger phrasings)
- **Workflow outline** (3–6 bullet steps)

**Common phase patterns by domain:**

*AWS / cloud infra:*
1. `understand-legacy-service` — reads existing code/config and produces a summary of current behaviour, dependencies, and gaps
2. `parse-requirements` — extracts structured requirements from a doc, ticket, or conversation
3. `aws-service-design` — proposes architecture, service selection, and trade-offs
4. `infrastructure-comparison` — compares two or more infra approaches (e.g. ECS vs Lambda, RDS vs Aurora)
5. `infrastructure-setup` — sets up the infrastructure using whatever tooling the team uses (IaC, runbooks, scripts, console — ask, don't assume)
6. `cicd-pipeline-design` — designs the build/test/deploy pipeline
7. `service-implementation-guide` — step-by-step guide to implement the agreed design

*Software feature:*
1. `understand-existing-codebase` — maps relevant modules, patterns, and entry points
2. `feature-requirements-parser` — turns a brief or ticket into structured acceptance criteria
3. `feature-design` — produces a technical design doc with trade-offs
4. `implementation-plan` — breaks design into ordered tasks with dependencies
5. `code-scaffold` — generates boilerplate following the repo's conventions
6. `test-plan` — defines unit, integration, and e2e test cases

*Data / ML:*
1. `data-source-audit` — documents schemas, volumes, and quality of inputs
2. `pipeline-design` — designs the ETL/ELT or feature-engineering pipeline
3. `model-selection` — compares candidate models or approaches with trade-offs
4. `pipeline-scaffold` — generates boilerplate pipeline code
5. `monitoring-plan` — defines data quality checks, drift detection, and alerting

Mix and match across domains when the project spans them.

**Infrastructure skill naming rule:** Never bake a specific tooling choice (CDK, Terraform, CloudFormation, etc.) into an infrastructure skill's name or description. The skill should ask what tooling the team uses at runtime and adapt accordingly. Name it `<context>-infrastructure-setup` or similar.

**Phase-sequencing rule (legacy → requirements → design):** When a legacy system exists, the canonical sequence is `understand-legacy-*` → `*-requirements-capture` → `*-design`. Never let the user jump straight from legacy understanding into design — the design skill will silently inherit decisions from legacy state instead of justifying them against explicit requirements. When presenting the plan in Phase 4, surface this warning verbatim:

> *"Legacy understanding tells you what exists. Requirements-capture tells you what the new service must do. Skipping requirements-capture causes design decisions to be implicitly inherited from legacy rather than explicitly justified."*

If the user tries to skip requirements-capture, push back once before complying.

**Iteration-expectation rule:** Skills that map an existing system (legacy-understanding, codebase-audit, data-source-audit) are **not one-shot**. The first output is wrong in specific ways only the engineer who knows the code can catch. When generating these skills, write the workflow as a **convergence loop** (draft → user corrects → re-investigate affected section → re-emit → repeat) rather than a single draft-and-ask. Budget the user's expectation for 5–10 rounds in a realistic mid-size codebase.

**Replacement-rationale rule:** When legacy exists, the suite MUST include a `replacement-rationale` (or equivalently-named) skill positioned between `understand-legacy-*` and `*-requirements-capture`. Its job is to interrogate WHY the replacement is justified, what alternatives were considered, and what forcing function drives it now — producing a written rationale that gates downstream work. Without this, teams drift into rewrite-for-rewrite's-sake and design decisions get retrofitted with motivations after the fact. If Q6 returned `rationale: unclear`, call this out explicitly in Phase 4 as a risk the user must resolve before scaffolding design skills.

**Design-implications rule:** Skills that produce a reference doc of current state (legacy-understanding, codebase-audit) must **not** emit "design implications" or "recommendations for the new service." That pre-commits decisions before requirements-capture runs. If observations naturally suggest design questions, file them under "Observations for design discussion" — observations, not implications.

**Pattern-service rule.** When the project sits inside an organisation that has shipped similar services before (e.g. another AWS microservice with the same shape — internal ALB + ECS + provider integrations), the planner should ask "is there a similar service in the org we can study as a pattern template?" and capture the answer in the project profile under `pattern_service`. Downstream skills (especially `*-architecture-design` and `*-infrastructure-setup`) should read this and mirror that service's conventions verbatim instead of inventing parallel ones. Without this, generated artefacts drift from house style in small ways the user has to keep correcting.

**Phase-iteration rule (design ↔ infrastructure).** Design and infrastructure-setup are not strictly sequential — discoveries during infrastructure-setup routinely feed back into design. Typical things that surface late and force a design revisit:

- DNS topology — which stack owns which record, and collisions (e.g. ALB and API Gateway both wanting the same hostname)
- Secrets convention — Secrets Manager vs SSM SecureString choice driven by org standard
- Image strategy — one container image for api + worker vs separate images
- Shared-template constraints — e.g. a shared ECS template that forces an ALB on every service
- Cross-account auth boundaries — which accounts the apply-side actor can reach

Plan a budget for one or two design revisits during infrastructure-setup. Present this expectation to the user in Phase 4 so they're not surprised by backtracking.

### Phase 4 — Present the Plan

Show the proposed skill suite as a structured list:

```
Phase 1 — Understand
  • understand-legacy-service: [one-line description]
  • parse-requirements: [one-line description]

Phase 2 — Design
  • aws-service-design: [one-line description]
  ...
```

Then say: "Ready to scaffold all of these? Say 'yes, build them' or tell me which ones to skip or rename."

Wait for explicit approval before building anything.

### Phase 5 — Scaffold

For each approved skill, create `~/.claude/skills/<skill-name>/SKILL.md` with:
- Full frontmatter (`name`, `description`)
- Workflow section with numbered steps
- When to use / When NOT to use sections

Follow the SKILL.md skeleton and writing guidelines from the `create-skill` skill. After all files are written, report the full list of paths created.

## Writing guidelines for generated skills

- Explain *why* constraints exist rather than issuing bare rules in caps
- Push lookup tables, edge cases, and deep context to `references/` — keep SKILL.md lean
- Every line should be high-signal; cut anything a frontier model already knows
- Descriptions must lean slightly pushy (combat undertriggering): include casual phrasings and adjacent intents

## Cross-cutting rules that ALL content-producing skills should encode

Any skill that produces a published artefact (rationale, requirements, design, reference doc) must include these in its Hard rules section:

- **Document hygiene — write current truth, not a changelog.** As the doc iterates with the user, weave corrections in as if the document always said what it now says. No "reframed" / "updated" / "v2" markers in the body; no warning/info panels flagging recent changes; no "previously this said X". Iteration history lives in the page version history.
- **No inline bold for emphasis in published prose.** Headings, table headers, and inline `code` formatting carry the visual structure. Bolding phrases inside sentences adds noise.
- **Section titles must read standalone.** Drop "Restated" / "Recap of" prefixes that imply cross-doc continuity. If the doc references prior context, the section is just "Constraints" / "Inputs" / "Context", not "Restated constraints".
- **Recommendation-after-analysis.** In any selection-style doc (stack, vendor, approach), present the comparison before the verdict. Constraints → Comparison → Deciding factor → Recommendation. Putting the verdict first reads as imposed.
- **Don't conflate v1 scope with architectural intent.** If the artefact is for a project that ships less than its design supports, distinguish explicitly: what's *delivered* by this project, what's *enabled* for future projects, what's untouched in other systems.
- **Don't bake another project's work into this project's success criteria.** If a criterion can only be met by changing a system that's out of scope, restate it as the effect inside the in-scope system.
- **Deciding factors must be intrinsic to the choice, not proxies for downstream decisions.** If you find yourself reasoning "we should pick X because of choice Y we haven't made yet", you're either making Y prematurely or X is actually tied with the alternative.
- **Cleanup / dead-code callouts go in their own section.** Don't litter the live narrative with "NOT IN USE" status lozenges; consolidate into a Dead Code / Cleanup Candidates section so the rest of the doc reads as living truth.
- **Title hygiene — no version markers in page titles.** Drop "(v1 Draft)", "(v3)", "(WIP)" etc. The page title is the durable name of the document, not its iteration state. Iteration lives in the platform's version history. Same principle as document hygiene — applied to titles.
- **Numeric / scope constraints have a propagation footprint.** When a load-bearing input changes (throughput target, deployment topology, in-scope list, timeout budget), identify every downstream doc that consumed it and update them in one sweep. Inconsistency between sibling docs is worse than a clean rewrite.
- **Walk back deferred decisions when they get made.** If an earlier-phase doc said "decision deferred to architecture", and architecture then made the call, revisit the earlier doc to (a) note the choice was made, and (b) check whether its reasoning still holds given the choice. A stack-selection that argued "Fargate keeps DB choice open" reads stale once DB is picked — say so explicitly rather than leaving the argument dangling.
- **Deferred-to-follow-up-PR is a first-class output.** Every content-producing skill that maps onto a deliverable (an architecture doc, a PR, a published page) must emit, alongside the main artefact, an explicit "Deferred to follow-up PR" / "Out of this artefact's scope" section listing the items it intentionally does not deliver. Without this, reviewers can't tell what's missing-by-design vs missing-by-oversight, and follow-up work is unclear.

## Confluence (or any platform with a body+title update API) — operational rules

When the content-producing skill publishes to a platform where `update(page)` requires both title and body:

- **Renaming requires re-sending the full body.** There is no title-only path. Before any rename, fetch (or have in context) the complete current body and submit it verbatim alongside the new title. A truncated body silently destroys content.
- **TOC-style auto-generated nodes are sacred once placed by the user.** If the user has configured a Table of Contents macro (or similar platform-rendered block), preserve it byte-for-byte in every subsequent update. Don't re-emit it, don't move it, don't "improve" it. If unsure, leave it where it is.
- **Renaming children means updating index links in the parent.** If a parent page lists its children by title, those link labels are now stale after a rename. Same sweep as the rename.
