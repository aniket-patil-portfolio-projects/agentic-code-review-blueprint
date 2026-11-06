
# Agentic Code Review System

## NOTE: This is an independently created technical blueprint based on concepts and learnings from a professional project developed during my employment. It is intended for technical reference and demonstration purposes and does not contain any proprietary source code or confidential information.

---

## Background

Modern enterprise software development teams face a structural tension: as codebases grow in size and complexity, the cognitive burden of meaningful code review increases, yet the time available per review does not. A reviewer who lacks deep familiarity with a module will naturally produce a shallower review — catching surface-level issues while missing architectural risks, dependency side-effects, and misalignment with stated requirements.

This initiative emerged from direct observation of that tension inside a large, active Java development organization. The team produces a high volume of pull requests against a codebase with complex interdependencies across modules, database layers, routing configurations, and service integrations. The organization also operates an internal knowledge system that has indexed structural relationships across the codebase — classes, entities, data schemas, routes, and their interconnections. That knowledge existed but was not systematically applied during code review.

The project began as an individual engineering initiative to answer a specific question: **can the review process be augmented by automatically combining ticket requirements, code change context, and codebase structural knowledge before the AI is asked to reason about a change?**

Two successive versions were designed, built, and validated in real use. This document describes the complete system as it stands — its purpose, architecture, capabilities, and direction.

---

## Problem Statement

### The Core Review Problem

A technically complete code review requires three distinct layers of context:

1. **Intent** — what was the developer asked to build? What are the acceptance criteria? What is the scope of the ticket?
2. **Change** — what lines of code were actually modified, added, or removed? Which files and functions were touched?
3. **Impact** — what does this code connect to? What calls it? What does it call? What data model does it operate on? What routes expose it?

In practice, reviewers assemble these layers manually. Intent lives in a ticket management system. Change lives in a source control pull request. Impact lives in the reviewer's mental model of the codebase — knowledge built over months or years of working in a specific module.

When a reviewer lacks full context across all three layers, the resulting review is predictably shallow: it identifies style issues and obvious bugs while missing dependency risks, logic gaps against requirements, and downstream effects that only reveal themselves when the change is traced through the dependency graph.

### Why Generic AI Tools Do Not Solve It

Pasting a diff into a general-purpose AI assistant addresses only the second layer (change). The AI has no knowledge of what the ticket required, what the function's callers depend on, or what data schema it modifies. The output is generic: standard best practices, common null-check observations, boilerplate recommendations that apply to any codebase anywhere.

This kind of review is not useless, but it is structurally limited. It cannot know that a particular method is called by three upstream services. It cannot verify whether the implementation satisfies specific acceptance criteria written in a ticket. It cannot assess whether a change to a data-access class has cascading effects on downstream processing logic. For that, it would need the codebase knowledge — the third layer.

### The Workflow Gap

Even when a good review is produced, a second problem appears: **the distance between the review document and the act of applying its findings**. A review that lists findings, files, and line numbers is useful documentation, but acting on it still requires the developer to manually locate each file, navigate to the relevant line, and make the change. In a large codebase with thousands of files across multiple repositories, this friction is real. Findings that are inconvenient to act on tend to get deferred or forgotten.

---

## Existing Limitations

Prior to this initiative, the available approaches and their specific shortcomings were as follows:

| Approach                                    | Limitation                                                                                                                                       |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Manual peer review                          | Quality depends entirely on reviewer's familiarity with the module; inconsistent depth and coverage across reviewers                             |
| Generic AI review (prompt-based)            | No knowledge of ticket requirements; no knowledge of codebase structure; produces boilerplate output disconnected from actual system context     |
| Basic automation (linting, static analysis) | Detects syntactic and stylistic violations; does not reason about requirements, dependencies, or architectural impact                            |
| Ad hoc knowledge system queries             | Requires the developer to manually formulate queries against the knowledge system; not integrated into the review workflow                       |
| Separate review tooling                     | Lives outside the IDE; produces a document that must be acted on manually; no write-back to the ticketing system; no persistence across sessions |

The underlying limitation across all prior approaches was the same: **no mechanism existed to automatically assemble all three layers of context — intent, change, and impact — and reason about them together before producing a structured, auditable review output**.

---

## Introduction to the Proposed System

The system described in this blueprint is an **agentic code review pipeline** that runs natively inside the developer's IDE as an AI-accessible tool. It is not a separate application the developer switches to — it is a set of tools callable from within the AI coding assistant the team already uses.

When a developer triggers a review, the system automatically:

- Retrieves the full ticket details including requirements, acceptance criteria, and comments
- Locates the associated pull request(s) across source control systems, regardless of PR state
- Extracts the complete code diff at the file and line level
- Queries the internal codebase knowledge system per modified file to retrieve callers, dependents, data model relationships, routing context, and entity definitions
- Assembles all of this into a structured data block and hands it to the AI to write a structured review
- Validates that the review meets a defined completeness standard before accepting it
- Persists the review and attaches it to the originating ticket automatically, making it accessible to the full team
- Makes the review's findings actionable — resolving file paths and enabling direct in-IDE application of recommended changes

The AI's role is to reason and write. The pipeline's role is to gather and structure. These responsibilities are deliberately separated.

---

## Key Capabilities

- **Requirement-Aware Code Review:** Evaluates code changes against the original development requirements.
- **Change Analysis:** Examines implementation changes to identify potential defects, inconsistencies, and areas requiring attention.
- **Application Context Integration:** Uses application-specific knowledge to understand relevant components, dependencies, and relationships.
- **Impact Analysis:** Identifies potential downstream effects of code changes across interconnected application components.
- **Structured Review Generation:** Produces organized review findings with explanations and recommendations.
- **Output Validation:** Applies programmatic validation to improve the consistency and completeness of generated reviews.
- **Optional Fix Application:** Supports developer-reviewed remediation by identifying suggested changes that can be applied to the local codebase.

---

### Agentic Workflow

The Generation 2 system follows a five-phase agentic loop. The AI acts autonomously across phases — it receives one instruction and drives the pipeline to completion, including self-correction on validation failure.

```
  Developer: "review ticket [ID]"
          |
          v
  +-------+-------+
  |  PHASE 1      |  AI identifies and invokes data gathering tool
  |  INITIATION   |
  +-------+-------+
          |
          v
  +-----------------------------------+
  |  PHASE 2   [code-controlled]      |
  |  DATA GATHERING                   |
  |                                   |
  |  Fetch ticket details             |
  |    (requirements, criteria,       |
  |     comments)                     |
  |          |                        |
  |          v                        |
  |  Discover associated PRs          |
  |    (any state, not open-only)     |
  |          |                        |
  |          v                        |
  |  Fetch PR metadata + full diff    |
  |          |                        |
  |          v                        |
  |  Per modified file:               |
  |    knowledge system queries       |
  |          |                        |
  |          v                        |
  |  Stage assembled data             |
  |  Return data reference to AI      |
  +-----------------------------------+
          |
          v
  +-----------------------------------+
  |  PHASE 3   [AI-controlled]        |
  |  REASONING AND REVIEW WRITING     |
  |                                   |
  |  AI reads assembled data             |
  |  Reasons over:                    |
  |    requirements vs implementation |
  |    dependency and impact risk     |
  |    security, performance, testing |
  |  Produces structured review       |
  +-----------------------------------+
          |
          v
  +-----------------------------------+
  |  PHASE 4   [code-controlled]      |
  |  VALIDATION AND PERSISTENCE       |
  |                                   |
  |  Integrity check                  |
  |    FAIL -> return missing list    |-----+
  |    PASS -> continue               |     |
  |          |                        |     |
  |          v                        |     |
  |  Normalize format                 |     |
  |  Publish review to ticket system  |     |
  |  Accessible to full team          |     |
  +-----------------------------------+     |
          |                                 |
          |           AI fills gaps,        |
          |           re-submits <----------+
          v
  +-----------------------------------+
  |  PHASE 5   [optional]             |
  |  FIX APPLICATION                  |
  |                                   |
  |  Developer: "apply the fixes"     |
  |  Parse findings from review       |
  |  Resolve findings to absolute     |
  |    local paths                    |
  |  Return structured fix list       |
  |          |                        |
  |          v                        |
  |  AI applies fixes in IDE editor   |
  |  No manual file navigation needed |
  +-----------------------------------+
```

---

### Key Architectural Decisions

| Decision                                       | Rationale                                                                                                                                                                                                              |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AI engages only after all context is assembled | Prevents the AI from reasoning over incomplete information; every review is grounded in the same full data set regardless of PR size or complexity                                                                     |
| Context passed by reference, not inline        | Keeps the AI context window reserved for reasoning; eliminates token overflow on large PRs and prevents the model from truncating or ignoring parts of the input                                                       |
| Structured output schema enforced at save      | Without a defined schema, AI output varies in structure across runs and models; a fixed schema makes every review comparable, searchable, and parseable by downstream tools                                            |
| Validation gate rejects incomplete reviews     | A prompt instruction to include all sections is not reliable; a code gate that refuses to accept an incomplete review and forces a retry is the only consistent enforcement mechanism                                  |
| Gathered data staged as an auditable record    | Any finding in the review can be traced back to what was actually retrieved; this is the primary mechanism for detecting AI hallucination — if a claim is not in the staged data, it was fabricated                    |
| AI self-corrects on validation failure         | Rather than failing hard, the system returns the list of missing sections to the AI for completion; this handles natural variance in model output without requiring human intervention                                 |
| Output normalised before persistence           | AI formatting varies across runs — numbered headings, inline markdown in table cells, minor structural deviations; normalising at save time ensures downstream tools see consistent text regardless of model behaviour |

---

## Evaluation

AI agent evaluation in production systems differs fundamentally from traditional software testing. There is no binary pass/fail ground truth for reasoning quality. This section describes the evaluation dimensions applicable to this system, framed against the criteria used across the industry to assess agentic AI performance.

---

### Evaluation Summary

| Dimension                      | What It Measures                                | Observable Signal in This System                |
| ------------------------------ | ----------------------------------------------- | ----------------------------------------------- |
| Task Completion Rate           | End-to-end pipeline reliability                 | Review published vs. failed per run             |
| Schema Compliance Rate         | Structural output quality on first attempt      | Validation pass on attempt 1 vs. retry required |
| Faithfulness / Grounding       | Hallucination rate against staged source data   | Findings traceable to staged record             |
| Requirement Coverage Rate      | Completeness against ticket scope               | Acceptance criteria addressed in review         |
| Self-Correction Rate           | AI's ability to recover from validation failure | Retry convergence in 1 attempt                  |
| Context Utilization Efficiency | Use of high-signal gathered data                | Knowledge graph findings surfaced in review     |
| Output Consistency             | Variance across equivalent runs                 | Finding count and severity stability            |

---

## Learning

### Technical Learnings

- **Separating gathering from reasoning was the single most impactful design decision.** Early prototypes had the AI orchestrate its own data gathering. Output quality was inconsistent and hard to debug. Moving all gathering into deterministic code made the AI's job narrower and its output more reliable.
- **Staging context before engaging the AI solved two problems at once** — token overflow on large PRs and the inability to inspect what the AI actually received. The staged data is both a buffer and an audit trail.
- **Validation in code, not in prompts.** Instructions to "include all sections" are followed inconsistently across runs, models, and prompt variations. A code gate that refuses to save an incomplete review is the only reliable enforcement mechanism.
- **Generation 1 validated the concept; Generation 2 fixed the workflow friction.** The web app proved the pipeline produced useful output. The MCP server eliminated the context-switching cost that made the web app feel like a separate tool rather than part of the review workflow.
- **AI output quality is proportional to context quality.** The single biggest lever on review accuracy was not prompt engineering — it was how much relevant, structured context was assembled before the AI was engaged. Better data in, better reasoning out.
- **MCP as an architecture pattern changes how AI tools integrate.** Building the second generation as an MCP server rather than a standalone application meant zero friction for the developer — no new tool to learn, no context switch, no separate window. The AI assistant becomes the interface.

### Personal Learnings

- **Building end-to-end forces clarity.** Designing a system that actually runs in production — even at small scale — exposes assumptions that a design document never does. Every integration point that seemed simple in planning had at least one non-obvious edge case in implementation.
- **Agentic systems require a different mental model than traditional software.** The AI is not a deterministic function. Designing around that — by making non-deterministic parts narrow and well-bounded — is a skill that develops through iteration, not upfront design.
- **The value of the system is not the AI — it is the context assembly.** Any sufficiently capable model can write a code review given good context. The hard and valuable part is reliably gathering that context, structuring it, and delivering it without losing information or exceeding limits. That is an engineering problem, not an AI problem.
- **Iterating from a working prototype is faster than designing from scratch.** Generation 2 was designed knowing exactly what Generation 1 did well and where it created friction. That prior working system — even a simple one — was worth more than any amount of upfront architecture planning.
- **Developer tooling that reduces friction gets used; tooling that adds friction does not.** The web app worked but required switching context. The MCP server did not. Adoption followed directly from removing that single friction point.

---
