# Software Requirements Specification (SRS)

## Autonomous AI Software Engineer

**Document status:** Approved product requirements baseline (architecture choices remain separately documented)\
**Version:** 2.1\
**Date:** 2026-11-26\
**Document purpose:** This document defines the functional and
non-functional requirements for an autonomous software engineering
agent. It is intended to be kept in the repository and used as the
source of truth by AI coding agents (Codex, Claude Code, Antigravity,
and similar tools).

------------------------------------------------------------------------

## 1. Product overview

### 1.1 Purpose

The Autonomous AI Software Engineer (AIASE) is a system that receives a
software bug or engineering task from Asana, investigates the relevant
codebase, attempts to reproduce the problem, identifies and validates
likely root causes, proposes and implements a fix, verifies the changes,
and opens a merge request for human review.

If the system cannot confidently determine or implement a safe fix, it
must not fabricate certainty or make speculative code changes. Instead,
it must produce an evidence-based investigation and research report
containing possible causes, potential solutions, trade-offs, and
concrete next steps. The report should be added to the originating Asana
issue and made available to the assigned developer.

The system is intended to behave like a careful software engineer, not
merely a code-generation chatbot.

### 1.2 Product goals

1.  Reduce the manual effort required to investigate and resolve
    software bugs.
2.  Convert an issue description and acceptance criteria into a
    reproducible engineering investigation.
3.  Ground root-cause analysis in repository evidence, execution
    results, logs, tests, and relevant research.
4.  Generate small, reviewable, tested code changes.
5.  Keep a human developer in control of ambiguous, risky, or
    irreversible decisions.
6.  Provide useful engineering output even when autonomous
    implementation is unsuccessful.
7.  Make the complete workflow observable, auditable, and resumable.

### 1.3 Non-goals for the initial release

The initial release is not intended to:

-   Merge merge requests automatically.
-   Deploy code to production.
-   Modify production data or infrastructure.
-   Execute arbitrary commands with unrestricted host access.
-   Make unapproved architectural changes or broad refactors.
-   Guarantee that every reported bug can be reproduced or fixed.
-   Replace human code review, product decisions, or release approval.
-   Autonomously change issue priority, ownership, or acceptance
    criteria without authorization.

These capabilities may be considered later, but they are outside the
initial scope.

### 1.4 Terminology

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  Issue                               A bug or engineering task assigned
                                      in Asana

  Agent run                           One execution of the workflow for
                                      an issue

  Orchestrator                        Component responsible for workflow
                                      state, routing, retries, and
                                      checkpoints

  Hypothesis                          A possible explanation for the
                                      reported behaviour

  Reproduction                        A repeatable demonstration of the
                                      reported bug

  Root cause                          The underlying defect supported by
                                      sufficient evidence

  Work branch                         Isolated Git branch created for an
                                      agent run

  Verification                        Tests and checks used to evaluate
                                      the proposed change

  MR                                  merge request opened for human
                                      review

  Fallback report                     Research and investigation report
                                      produced when a safe fix cannot be
                                      completed

  Human intervention                  Input, approval, credentials, or
                                      decision required from a developer
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 2. Users and operating assumptions

### 2.1 Primary user

The primary user is a software developer who receives issues in Asana
and has access to the associated source-code repository.

### 2.2 External systems

The initial system is expected to integrate with:

-   **Asana:** issue discovery, issue details, comments, and
    notifications.
-   **Git provider:** repository access, branches, commits, pushes, and
    merge requests. GitLab is the initial target unless configured
    otherwise.
-   **Execution environment:** isolated workspace or container for
    repository inspection, builds, tests, and controlled reproduction.
-   **LLM provider(s):** reasoning, planning, code understanding,
    research synthesis, and code generation.
-   **Research sources:** official documentation, framework/library
    documentation, relevant release notes, and other approved sources.

### 2.3 Assumptions

1.  An Asana issue contains, or can be supplemented with, a description
    of the observed behaviour and expected behaviour.
2.  The agent has authorized access to the relevant repository.
3.  The repository contains enough setup information to install
    dependencies and run at least some checks, or the agent can identify
    what is missing.
4.  The Git provider supports creating branches and merge requests
    through an authorized integration.
5.  The agent operates with explicitly configured permissions.
6.  Secrets are supplied through a secure secret mechanism and are not
    stored in prompts, logs, or generated reports.

### 2.4 Configurable inputs

The system should support configuration for:

-   Asana workspace, project, and assignment filters.
-   Repository and default base branch.
-   Git provider and MR settings.
-   Allowed commands and network access.
-   Runtime, dependency installation, build, lint, and test commands.
-   Maximum investigation time, implementation attempts, and retry
    count.
-   LLM model/provider and token or cost limits.
-   Approval requirements for risky operations.
-   Research source allowlist, where applicable.

------------------------------------------------------------------------

## 3. High-level workflow

The normal workflow is:

1.  Detect an eligible issue assigned to the configured developer or
    agent.
2.  Retrieve and normalize the issue details and acceptance criteria.
3.  Create an agent run and record its initial state.
4.  Access the repository in an isolated workspace.
5.  Explore the codebase and gather relevant engineering context.
6.  Generate and rank root-cause hypotheses.
7.  Attempt to reproduce the bug in a controlled environment.
8.  Validate or reject hypotheses using evidence.
9.  If a root cause is sufficiently supported, plan a minimal, safe fix.
10. Implement the fix on an isolated work branch.
11. Add or update tests that cover the reported behaviour.
12. Run relevant tests and quality checks.
13. Review the diff and verify the acceptance criteria.
14. If checks pass and policy allows, push the branch and open a MR.
15. Post a summary and links to the Asana issue.
16. Wait for human review; process review feedback if that capability is
    enabled.

If the agent cannot safely complete the implementation, it must follow
the fallback research workflow in Section 7.

### 3.1 Workflow principles

-   Evidence must precede conclusions.
-   The agent must distinguish facts from hypotheses and
    recommendations.
-   The agent must not claim a bug is reproduced unless it has
    repeatable evidence.
-   The agent must not claim tests passed unless the relevant commands
    actually completed successfully.
-   The agent must prefer minimal, focused changes over unrelated
    refactoring.
-   The agent must stop and request human input when required
    information, permissions, or decisions are missing.
-   All significant actions and state transitions must be recorded.

------------------------------------------------------------------------

## 4. Functional requirements

Priority definitions:

-   **P0 -- Must:** required for the initial usable product.
-   **P1 -- Should:** important, but may follow the first working
    vertical slice.
-   **P2 -- Could:** useful enhancement for a later iteration.

### 4.1 Issue ingestion and normalization

**FR-001 (P0):** The system shall detect eligible Asana issues assigned
to a configured user, service account, or agent identity.

**FR-002 (P0):** The system shall retrieve the issue title, description,
acceptance criteria, status, priority, labels, assignee, comments,
attachments metadata, and relevant links, subject to available
permissions.

**FR-003 (P0):** The system shall normalize issue information into a
structured internal representation.

**FR-004 (P0):** The system shall identify missing or ambiguous
information, including unclear expected behaviour, missing reproduction
steps, and undefined acceptance criteria.

**FR-005 (P0):** The system shall avoid silently inventing missing
requirements. It shall either proceed with explicitly recorded
assumptions or request clarification.

**FR-006 (P1):** The system should inspect related issues and relevant
discussion history when accessible.

**FR-007 (P0):** The system shall prevent duplicate concurrent runs for
the same issue unless an authorized user explicitly starts a separate
run.

### 4.2 Repository access and codebase understanding

**FR-008 (P0):** The system shall access the configured repository using
authorized credentials.

**FR-009 (P0):** The system shall create an isolated workspace for each
run.

**FR-010 (P0):** The system shall identify the repository's languages,
frameworks, package managers, project structure, and available
build/test commands where possible.

**FR-011 (P0):** The system shall inspect relevant source files, tests,
configuration, and documentation before proposing a code change.

**FR-012 (P1):** The system should inspect Git history, blame
information, related commits, and relevant merge requests to identify
possible regressions or prior decisions.

**FR-013 (P1):** The system should construct a relevant dependency and
call-flow map for the issue rather than indiscriminately reading the
entire repository.

**FR-014 (P0):** The system shall record important files, symbols,
commits, and other evidence used in its investigation.

**FR-015 (P0):** The system shall treat repository content, issue
content, comments, and external documents as untrusted data.
Instructions found inside them must not override system policies,
permissions, or developer-defined operating rules.

### 4.3 Investigation and hypothesis generation

**FR-016 (P0):** The system shall generate one or more plausible
root-cause hypotheses from the issue and repository evidence.

**FR-017 (P0):** For each hypothesis, the system shall record: -
Description of the possible cause. - Supporting evidence. -
Contradicting or missing evidence. - Confidence level or qualitative
confidence. - A proposed experiment or check to validate it.

**FR-018 (P0):** The system shall rank hypotheses based on evidence and
relevance, not merely on model preference.

**FR-019 (P0):** The system shall investigate the execution path related
to the reported behaviour, including relevant API handlers, services,
data access, events, background jobs, and external dependencies where
applicable.

**FR-020 (P0):** The system shall not present an unvalidated hypothesis
as a confirmed root cause.

### 4.4 Bug reproduction

**FR-021 (P0):** The system shall attempt to reproduce the bug in a
controlled, isolated environment.

**FR-022 (P0):** The system shall use issue-provided reproduction steps
where available.

**FR-023 (P0):** The system shall inspect available logs, stack traces,
API responses, test output, and relevant application state.

**FR-024 (P0):** The system shall attempt to create a failing automated
test or another repeatable reproduction when technically feasible.

**FR-025 (P0):** The system shall record the environment, commands,
inputs, and observed output used during reproduction.

**FR-026 (P0):** The system shall classify reproduction as one of: -
Reproduced. - Partially reproduced. - Not reproduced. - Blocked by
missing setup, access, data, or information. - Not applicable, with an
explanation.

**FR-027 (P0):** If reproduction is unsuccessful, the system shall not
claim the issue is fixed merely because it has made a code change.

### 4.5 Root-cause validation

**FR-028 (P0):** The system shall use tests, controlled experiments,
code inspection, logs, and other available evidence to validate or
reject hypotheses.

**FR-029 (P0):** The system shall record why a hypothesis was accepted,
rejected, or left inconclusive.

**FR-030 (P0):** The system shall identify the difference between the
observed symptom and the underlying cause where possible.

**FR-031 (P0):** If no root cause can be established with sufficient
confidence, the system shall move to the fallback research workflow or
request human input rather than inventing certainty.

### 4.6 Solution planning

**FR-032 (P0):** Before implementation, the system shall create a
solution plan describing the proposed change and the files or components
likely to be affected.

**FR-033 (P0):** The plan shall describe relevant edge cases, failure
modes, compatibility considerations, and regression risks.

**FR-034 (P0):** The system shall identify required tests and explain
how they validate the issue's acceptance criteria.

**FR-035 (P0):** The system shall prefer the smallest safe change that
addresses the validated root cause.

**FR-036 (P0):** The system shall request human approval before
performing operations classified as high risk by policy, including
destructive data operations, production changes, or broad architectural
changes.

### 4.7 Code implementation

**FR-037 (P0):** The system shall create a uniquely named work branch
from the configured base branch.

**FR-038 (P0):** The system shall not commit directly to the protected
base branch.

**FR-039 (P0):** The system shall implement the approved or
policy-permitted solution in the isolated workspace.

**FR-040 (P0):** The system shall follow repository conventions and
reuse existing abstractions where appropriate.

**FR-041 (P0):** The system shall add or update unit, integration,
regression, or other relevant tests.

**FR-042 (P0):** The system shall avoid unrelated refactoring,
formatting churn, generated-file changes, or dependency upgrades unless
required and justified.

**FR-043 (P0):** The system shall keep implementation attempts bounded
by configurable limits.

**FR-044 (P0):** If implementation repeatedly fails or the agent's
assumptions are invalidated, the system shall stop autonomous
modification and enter fallback or human-intervention handling.

### 4.8 Testing and verification

**FR-045 (P0):** The system shall run the most relevant tests for the
changed code.

**FR-046 (P0):** The system shall run configured linting, formatting
checks, type checks, and build checks when available and relevant.

**FR-047 (P0):** The system shall run regression tests appropriate to
the scope of the change.

**FR-048 (P0):** The system shall record each executed command, exit
status, and a useful summary of its output.

**FR-049 (P0):** The system shall distinguish passed, failed, skipped,
not-run, and blocked checks.

**FR-050 (P0):** The system shall verify that the tests actually
exercise the reported bug or acceptance criteria; merely adding a test
file is not sufficient.

**FR-051 (P0):** The system shall inspect the final Git diff for
unintended changes, suspicious modifications, and scope violations.

**FR-052 (P0):** The system shall not claim successful verification if
required checks failed or were not run. Any exceptions must be clearly
disclosed.

**FR-053 (P1):** The system should run security or static-analysis
checks when configured and relevant.

### 4.9 merge request creation

**FR-054 (P0):** The system shall push the work branch only after the
applicable verification and policy checks.

**FR-055 (P0):** The system shall create a merge request against the
configured base branch when all required conditions are satisfied.

**FR-056 (P0):** The MR description shall include: - Original issue
link. - Problem summary. - Confirmed root cause, or an explicit
statement that it remains uncertain. - Solution and implementation
summary. - Files or components changed. - Tests and checks executed with
their outcomes. - Known limitations, risks, and checks not performed.

**FR-057 (P0):** The system shall not merge the MR automatically in the
initial release.

**FR-058 (P0):** The system shall add the MR link and a concise
execution summary to the originating Asana issue.

**FR-059 (P1):** The system should process human review comments,
identify requested changes, and create a bounded follow-up
implementation run.

### 4.10 Fallback research and investigation report

This capability is a core product requirement, not an exceptional error
message.

**FR-060 (P0):** If the system cannot identify a sufficiently supported
root cause, cannot derive a safe solution, cannot implement a solution
within configured limits, or cannot complete required verification, it
shall produce a fallback report when enough information exists to make
the report useful.

**FR-061 (P0):** The report shall include an executive summary of what
the agent could and could not establish.

**FR-062 (P0):** The report shall include investigation steps performed,
relevant files and symbols inspected, commands run, reproduction
attempts, and available evidence.

**FR-063 (P0):** The report shall list plausible root causes separately
from confirmed facts.

**FR-064 (P0):** Each possible cause shall include: - Explanation. -
Supporting evidence. - Missing or contradictory evidence. - Confidence
level and reason. - A suggested experiment to validate it.

**FR-065 (P0):** The system shall conduct additional research when
appropriate, prioritizing official documentation, release notes, source
code, and trustworthy technical references.

**FR-066 (P0):** The system shall include research references and links
in the report. It shall not fabricate citations, documentation, issue
links, or findings.

**FR-067 (P0):** The report shall propose one or more possible
solutions, including: - High-level implementation approach. - Relevant
modules or files, if known. - Advantages and disadvantages. - Risks and
side effects. - Prerequisites and validation steps.

**FR-068 (P0):** The report shall identify the most promising next step
and explain why, without presenting it as guaranteed to work.

**FR-069 (P0):** The report shall list additional information, access,
logs, test data, or human decisions needed to continue.

**FR-070 (P0):** The system shall publish the report as a comment or
other configured update on the originating Asana issue and notify the
relevant developer when permissions allow.

**FR-071 (P0):** The system shall not create a MR containing speculative
or unverified changes merely to make the run appear successful.

**FR-072 (P0):** The report shall distinguish among: - Confirmed
facts. - Evidence-supported hypotheses. - Unverified possibilities. -
Externally researched recommendations.

### 4.11 Human intervention and approval

**FR-073 (P0):** The system shall support pausing a run and requesting
human input.

**FR-074 (P0):** The system shall request clarification when issue
requirements are materially ambiguous.

**FR-075 (P0):** The system shall request authorized access or
configuration when required resources are unavailable.

**FR-076 (P0):** The system shall request approval for actions that
require human authorization under configured policy.

**FR-077 (P0):** A human response shall be associated with the relevant
run and workflow state.

**FR-078 (P1):** The system should support resuming a paused run from
its latest valid checkpoint.

### 4.12 Run history and observability

**FR-079 (P0):** The system shall maintain a record of each agent run
and its associated issue.

**FR-080 (P0):** The system shall record state transitions, model/tool
actions, timestamps, command outcomes, errors, retries, and final
outcome.

**FR-081 (P0):** The system shall expose a useful execution summary for
debugging and auditing.

**FR-082 (P0):** The system shall record model usage and estimated
execution cost where provider data is available.

**FR-083 (P0):** The system shall support correlation between an Asana
issue, agent run, workspace, Git branch, commits, and MR.

------------------------------------------------------------------------

## 5. Workflow states and terminal outcomes

### 5.1 Suggested states

The orchestrator should model the workflow as explicit, persisted
states. Suggested states:

1.  `RECEIVED`
2.  `ISSUE_ANALYSIS`
3.  `WAITING_FOR_CLARIFICATION`
4.  `REPOSITORY_SETUP`
5.  `CODEBASE_EXPLORATION`
6.  `HYPOTHESIS_GENERATION`
7.  `BUG_REPRODUCTION`
8.  `ROOT_CAUSE_VALIDATION`
9.  `SOLUTION_PLANNING`
10. `WAITING_FOR_APPROVAL`
11. `IMPLEMENTATION`
12. `TESTING`
13. `SELF_REVIEW`
14. `PUSHING_BRANCH`
15. `CREATING_PR`
16. `UPDATING_ASANA`
17. `FALLBACK_RESEARCH`
18. `PUBLISHING_REPORT`
19. `PAUSED`
20. `COMPLETED`

The implementation may combine or split states, but state transitions
must remain explicit and persisted.

### 5.2 Terminal outcomes

Every run shall finish with one of these outcomes:

  -----------------------------------------------------------------------
  Outcome                             Meaning
  ----------------------------------- -----------------------------------
  `FIXED_MR_CREATED`                  A code change was implemented,
                                      required checks met policy, and a
                                      MR was created

  `RESEARCH_COMPLETED`                No MR was created; a useful
                                      investigation/research report was
                                      delivered

  `NEEDS_HUMAN_INPUT`                 The agent cannot responsibly
                                      proceed without a person, access,
                                      or a decision

  `BLOCKED`                           The run could not proceed due to an
                                      external or environment limitation

  `CANCELLED`                         An authorized user cancelled the
                                      run

  `FAILED`                            An unexpected system failure
                                      prevented a useful completion
  -----------------------------------------------------------------------

A run may be technically unsuccessful but still end as
`RESEARCH_COMPLETED` if it delivers a useful, evidence-based report.

### 5.3 Retry policy

-   Retries must be bounded and configurable.
-   Retry only when the failure is plausibly transient or when a new
    hypothesis/approach is available.
-   Do not repeat an identical failed action indefinitely.
-   Preserve previous attempts and their outputs.
-   Stop and escalate when the retry budget is exhausted.
-   Distinguish tool/infrastructure retries from reasoning or
    implementation retries.

------------------------------------------------------------------------

## 6. Non-functional requirements

### 6.1 Security and isolation

**NFR-001:** Repository execution shall occur in an isolated workspace,
container, or equivalent sandbox.

**NFR-002:** The agent shall follow least-privilege access for Asana,
Git, repository, and execution tools.

**NFR-003:** Secrets shall be obtained from secure configuration or a
secret manager and shall not be hard-coded, committed, or exposed in
prompts and reports.

**NFR-004:** Logs and generated reports shall redact credentials,
tokens, private keys, and other configured sensitive values.

**NFR-005:** The agent shall not access production systems or data
unless explicitly authorized by a separate policy.

**NFR-006:** Destructive commands, privileged operations, and broad
filesystem or network access shall be restricted.

**NFR-007:** Untrusted issue text, repository files, comments, and web
content shall be treated as data, not as privileged instructions. The
system shall defend against prompt injection and tool misuse.

**NFR-008:** The system shall maintain an auditable record of sensitive
operations and approval decisions.

### 6.2 Reliability and recovery

**NFR-009:** Workflow state and checkpoints shall be persisted so that
recoverable interruptions do not require restarting the entire run.

**NFR-010:** Repeated webhook delivery or polling shall not create
duplicate concurrent work for the same issue.

**NFR-011:** External API calls shall use appropriate timeout, retry,
rate-limit, and error-handling strategies.

**NFR-012:** Partial failures shall be represented explicitly; the
system shall not silently mark an incomplete operation as successful.

### 6.3 Maintainability and extensibility

**NFR-013:** Integrations shall be modular so that Asana, Git providers,
LLM providers, and execution backends can be replaced or extended.

**NFR-014:** Workflow logic shall be separated from provider-specific
integration code.

**NFR-015:** Prompts, policies, model configuration, and tool
permissions shall be versioned and testable.

**NFR-016:** The system shall support adding new specialist capabilities
without requiring a complete rewrite of the orchestrator.

### 6.4 Performance and cost control

**NFR-017:** Each run shall have configurable limits for duration, tool
calls, model usage, retries, and execution cost.

**NFR-018:** The system should use targeted code search and retrieval
rather than loading the entire repository into model context
unnecessarily.

**NFR-019:** Long-running operations shall provide status updates or
remain observable through run history.

**NFR-020:** The system shall avoid redundant research, repeated
repository scans, and duplicate test execution where results remain
valid.

### 6.5 Transparency

**NFR-021:** Every final result shall clearly state what the agent did,
what it did not do, and what remains uncertain.

**NFR-022:** Every claimed test result shall be traceable to an actual
command execution or trusted test-runner result.

**NFR-023:** Research recommendations shall include sources where
external research was used.

**NFR-024:** Confidence labels shall be treated as qualitative
estimates, not calibrated probabilities unless a calibration mechanism
is implemented.

------------------------------------------------------------------------

## 7. Safety policies and guardrails

The following policies apply throughout the system:

1.  **No direct base-branch commits.** All code changes must occur on an
    isolated work branch.
2.  **No automatic merge.** A human owns final review and merge
    approval.
3.  **No production deployment.** Deployment is outside the initial
    scope.
4.  **No unapproved destructive operations.** Database deletion, data
    migrations with destructive effects, infrastructure changes, and
    similar actions require approval.
5.  **No fabricated verification.** The agent must report actual test
    and command outcomes.
6.  **No unsupported certainty.** Hypotheses must not be represented as
    confirmed root causes without evidence.
7.  **No speculative MR.** If the fix is not sufficiently supported or
    verification requirements are not met, the agent should use fallback
    reporting or human escalation.
8.  **Bounded autonomy.** All retries, tool usage, execution time, and
    spending must have limits.
9.  **Workspace isolation.** Repository code and commands must run in a
    controlled environment.
10. **Human escalation.** The agent must pause when a decision exceeds
    its configured authority.

------------------------------------------------------------------------

## 8. Suggested specialist responsibilities

The product may use a single agent with tools, a graph of workflow
nodes, multiple specialist agents, or a hybrid design. The following
responsibilities are logical boundaries, not a requirement to create a
separate LLM for each one.

  -----------------------------------------------------------------------
  Responsibility                      Main duties
  ----------------------------------- -----------------------------------
  Orchestrator                        State management, routing, retries,
                                      checkpoints, policy enforcement

  Issue Analyst                       Parse issue, acceptance criteria,
                                      comments, and missing information

  Codebase Explorer                   Understand project structure,
                                      relevant files, call paths, history

  Investigation Specialist            Generate and test root-cause
                                      hypotheses

  Reproduction Specialist             Set up controlled reproduction and
                                      failing tests

  Solution Planner                    Propose minimal fix, edge cases,
                                      risks, and test plan

  Coding Specialist                   Implement changes and tests

  Verification Specialist             Run tests/checks and inspect
                                      results

  Review Specialist                   Inspect diff for correctness,
                                      regressions, security, and scope

  Research Specialist                 Research unresolved causes and
                                      possible solutions; produce
                                      fallback report
  -----------------------------------------------------------------------

The orchestrator remains responsible for enforcing policy. A specialist
must not grant itself permissions or bypass approval requirements.

------------------------------------------------------------------------

## 9. Core data entities

The implementation should define persistent representations for at least
the following concepts. Exact database technology and schema are
implementation decisions.

### 9.1 AgentRun

Suggested fields:

-   `id`
-   `asana_issue_id`
-   `repository_id`
-   `base_branch`
-   `work_branch`
-   `status`
-   `terminal_outcome`
-   `started_at`
-   `updated_at`
-   `completed_at`
-   `configuration_version`
-   `model/provider metadata`
-   `usage/cost metadata`
-   `error_summary`

### 9.2 IssueContext

-   Issue title and description.
-   Acceptance criteria.
-   Comments and relevant links.
-   Assumptions.
-   Missing information.
-   Normalized expected and actual behaviour.

### 9.3 InvestigationFinding

-   Finding ID.
-   Finding type: fact, hypothesis, experiment, or observation.
-   Description.
-   Evidence references.
-   Related files, symbols, logs, or commits.
-   Confidence and rationale.
-   Status: open, supported, rejected, inconclusive.

### 9.4 ReproductionAttempt

-   Environment description.
-   Setup steps.
-   Input data and reproduction steps.
-   Commands executed.
-   Observed output.
-   Result classification.
-   Logs or artifact references.

### 9.5 ImplementationChange

-   Branch and commit references.
-   Changed files.
-   Change summary.
-   Associated tests.
-   Implementation attempt number.
-   Review findings.

### 9.6 VerificationResult

-   Check name and command.
-   Start/end timestamps.
-   Exit code.
-   Status: passed, failed, skipped, not run, blocked.
-   Output summary and artifact references.

### 9.7 ResearchReport

-   Summary.
-   Confirmed facts.
-   Possible root causes.
-   Supporting and missing evidence.
-   Research sources.
-   Proposed solutions and trade-offs.
-   Recommended next steps.
-   Human input required.
-   Publication status and Asana comment reference.

### 9.8 ApprovalRequest

-   Requested action.
-   Reason and risk.
-   Requesting run and state.
-   Approver identity.
-   Decision and timestamp.
-   Optional comments.

------------------------------------------------------------------------

## 10. External interfaces

### 10.1 Asana integration

The integration should support, subject to API permissions:

-   Detecting eligible assigned issues through webhooks or polling.
-   Reading issue details and comments.
-   Reading accessible attachment metadata and links.
-   Posting investigation summaries and fallback reports.
-   Posting MR links and completion summaries.
-   Notifying or mentioning the relevant developer where supported.

Webhook handling must be idempotent. Polling should be available as a
fallback if webhooks are unavailable.

### 10.2 Git provider integration

The integration should support:

-   Repository metadata and clone/fetch.
-   Base branch inspection.
-   Branch creation.
-   Commit creation.
-   Branch push.
-   merge request creation.
-   merge request metadata retrieval.
-   Review-comment retrieval for a later feedback iteration.

### 10.3 Execution environment

The execution layer should support:

-   Creating and destroying isolated workspaces.
-   Installing dependencies using repository-defined instructions.
-   Running explicitly permitted commands.
-   Capturing stdout, stderr, exit codes, and execution duration.
-   Enforcing resource, time, filesystem, and network limits.
-   Persisting relevant artifacts and logs.

### 10.4 LLM and research interfaces

The model/research layer should support:

-   Structured outputs for issue analysis, hypotheses, plans, and
    reports.
-   Tool invocation through an allowlisted tool interface.
-   Provider-independent model configuration where practical.
-   Source capture for external research.
-   Usage and cost tracking where available.

------------------------------------------------------------------------

## 11. Acceptance criteria for the initial product

The initial product shall be considered ready for a controlled pilot
when the following end-to-end scenarios pass.

### AC-001: Successful bug fix

Given an eligible Asana issue with clear reproduction steps and
acceptance criteria, and a repository with a reproducible defect:

1.  The system ingests the issue.
2.  It accesses the correct repository and creates an isolated
    workspace.
3.  It identifies relevant code and records evidence.
4.  It reproduces the defect or creates a failing regression test.
5.  It identifies a supported root cause.
6.  It creates a solution plan and implements a focused fix on a work
    branch.
7.  It adds or updates tests.
8.  It runs configured checks and reports their actual outcomes.
9.  It inspects the final diff.
10. If policy conditions pass, it pushes the branch and opens a MR.
11. It posts a summary and MR link to the Asana issue.
12. It does not merge the MR.

### AC-002: Root cause remains uncertain

Given an issue that the agent cannot confidently diagnose:

1.  The system records investigation steps and evidence.
2.  It lists plausible causes as hypotheses, not facts.
3.  It conducts relevant research where appropriate.
4.  It proposes possible solutions with trade-offs and validation steps.
5.  It identifies missing information and recommended next actions.
6.  It publishes the fallback report to Asana.
7.  It does not create a speculative MR.

### AC-003: Bug cannot be reproduced

Given an issue whose reported bug cannot be reproduced:

1.  The system records reproduction attempts and environment details.
2.  It reports the reproduction status accurately.
3.  It investigates likely causes using available evidence.
4.  It requests logs, test data, or clarification when needed.
5.  It does not claim the bug is fixed without verification.

### AC-004: Tests fail after implementation

Given a code change that fails required tests:

1.  The system records the failing commands and output.
2.  It may attempt a bounded correction if a materially different
    approach is available.
3.  It does not report verification as successful.
4.  If it cannot resolve the failure within limits, it stops and
    publishes a report or requests human intervention.
5.  It does not open a MR that falsely claims all required checks
    passed.

### AC-005: Human approval required

Given an operation that policy marks as approval-required:

1.  The system pauses before performing the operation.
2.  It explains the requested action and risk.
3.  It records the approval request.
4.  It proceeds only after an authorized approval.
5.  It records the decision and resumes from a valid checkpoint.

### AC-006: Duplicate issue event

Given repeated delivery of the same Asana event:

1.  The system recognizes the duplicate event.
2.  It does not start duplicate concurrent runs.
3.  It retains a traceable record of event handling.

### AC-007: Interrupted run

Given a recoverable interruption:

1.  The system preserves the latest valid checkpoint.
2.  It can resume without losing the issue context and prior findings.
3.  It does not repeat completed side effects unnecessarily.
4.  It verifies the current workspace and Git state before continuing.

------------------------------------------------------------------------

## 12. MVP implementation plan

Build a working vertical slice before implementing every advanced
capability.

### Milestone 1 --- Issue to investigation

-   Asana integration.
-   Issue normalization.
-   Repository access.
-   Isolated workspace.
-   Codebase exploration.
-   Evidence-backed investigation report.

**Exit criterion:** Given an issue, the agent can explain which code
paths may be relevant and why, with references to actual repository
evidence.

### Milestone 2 --- Reproduction and root-cause analysis

-   Controlled environment setup.
-   Test and command execution.
-   Reproduction attempts.
-   Hypothesis generation and validation.
-   Failing regression test where feasible.

**Exit criterion:** The agent can demonstrate a reported defect or
clearly report why it could not reproduce it.

### Milestone 3 --- Fix implementation

-   Solution plan.
-   Work branch creation.
-   Code modification.
-   Test creation/update.
-   Bounded implementation retry.

**Exit criterion:** The agent can implement a focused fix for a
supported root cause and preserve a reviewable diff.

### Milestone 4 --- Verification and MR

-   Test/lint/type/build execution.
-   Self-review.
-   Git push.
-   MR creation.
-   Asana update.

**Exit criterion:** A verified change can be submitted for human review
without automatic merge.

### Milestone 5 --- Fallback research and resilience

-   Research workflow.
-   Structured fallback report.
-   Asana report publication.
-   Checkpoint/resume.
-   Observability, cost limits, and operational hardening.

**Exit criterion:** When the agent cannot safely fix an issue, it still
delivers an actionable report and stops safely.

The fallback report itself is a core requirement and should be developed
before the system is considered a complete MVP.

------------------------------------------------------------------------

## 13. Architecture baseline and open decisions

The companion `architecture.md` v1.0 defines the selected architecture: Express/TypeScript with Prisma, Python with LangGraph/LangChain, PostgreSQL, RabbitMQ, durable scheduling, Docker and Playwright. GitLab is the initial repository provider. Asana remains the issue provider. The architecture document governs technical choices; this SRS governs product behavior.

Open decisions: exact LLM models, approved research sources, deployment sizing, retention periods, approval UI beyond the initial authenticated API, and production sandbox hardening. Multiple explicitly registered repositories are supported; the first vertical slice may use one.

## 14. Instructions for AI coding agents working in this repository

Any AI coding agent using this document must follow these rules:

1.  Treat this SRS as the product requirements baseline.
2.  Before implementing a feature, inspect the existing repository and
    identify the current architecture, conventions, and implementation
    status.
3.  Do not assume a technology decision is final if Section 13 marks it
    open.
4.  Do not generate the entire application in one step. Implement small,
    verifiable milestones.
5.  Before making changes, state the requirement IDs being addressed and
    provide a concise implementation plan.
6.  Preserve existing working behaviour unless a requirement explicitly
    changes it.
7.  Write or update tests for behaviour changes.
8.  Run relevant tests and report exact outcomes. Never fabricate test
    execution.
9.  Do not add dependencies without explaining why they are needed.
10. Do not introduce unnecessary agents, abstractions, services, queues,
    or microservices. Prefer the simplest architecture that satisfies
    the requirements.
11. Enforce the security and approval rules in Section 7.
12. If requirements conflict, identify the conflict and ask for
    clarification rather than silently choosing.
13. If a requirement cannot be implemented safely, explain the
    limitation and propose alternatives.
14. Keep this document and implementation aligned. If an approved
    product decision changes a requirement, update the SRS in the same
    change or explicitly propose the required update.
15. At the end of each milestone, provide:
    -   Requirements completed.
    -   Architecture and files changed.
    -   Tests/checks executed and their results.
    -   Known limitations.
    -   Remaining work and recommended next milestone.

**First task for a coding agent:** Inspect the repository, report its
current state, map existing code to this SRS, identify gaps, and propose
a phased implementation plan. Do not begin a large implementation before
presenting that assessment.


---

## 15. Finalized product decisions (v2.1)

This section supersedes any conflicting earlier suggestions. Existing FR-001–FR-083 and NFR-001–NFR-024 remain applicable. New requirements extend rather than replace the earlier SRS.

### 15.1 Product scope

- The first release supports **one developer** and one connected Asana identity, with multiple explicitly configured repositories permitted.
- Architecture must preserve replaceable provider interfaces and per-run isolation for future multi-user support, but no multi-tenant UI, billing, or tenant administration is required in v1.
- Asana is the input and status/reporting surface; GitLab is the initial code-review output; a separate dashboard is optional.
- The agent must support backend, frontend/UI, and cross-stack bugs. No guarantee of fully autonomous resolution is implied.

### 15.2 Issue metadata and repository selection

The Asana issue should specify a repository and base branch; these are **issue metadata, not values hardcoded in this SRS**. The following is a recommended ticket template:

```yaml
Repository: organization/repository
Base Branch: develop
Environment: staging
Bug Type: frontend | backend | cross-stack | unknown
Expected Behavior: ...
Actual Behavior: ...
Steps to Reproduce: ...
Acceptance Criteria: ...
Evidence: screenshots, logs, traces, links
```

Repository and branch may alternatively be resolved from an explicit, versioned Asana-project mapping. If an issue's declared repository or branch conflicts with mapping, the agent must request clarification. The agent must verify access and branch existence. If the issue specifies no branch, use a configured default and record that assumption. The work branch is generated by the agent and is distinct from the issue's base branch.

### 15.3 Run isolation and runtime

The initial execution backend shall use a **disposable Docker-based workspace** per run, with Docker Compose or equivalent for test-only dependencies (database, message broker, frontend/backend services). The browser runner shall be isolated from host credentials and use Playwright. Resource ceilings, command allowlists/policies, timeouts, non-root execution, restricted mounts and controlled egress are mandatory. No Docker socket or privileged container access may be exposed to agent-controlled code. Stronger isolation (microVM, dedicated VM, hardened sandbox runtime) is required before accepting arbitrary untrusted repositories or workloads beyond the risk profile of the personal pilot. Never treat ordinary Docker as a perfect security boundary.

Dedicated test accounts, seeded synthetic data, short-lived credentials and secret redaction are required. Reproduction fixtures must be repeatable and cleaned up. Production credentials/data are not available to execution containers by default.

### 15.4 Repository consistency and reassignment

- Pin each investigation to a specific **base commit SHA**; record repository URL, branch and SHA before executing tests.
- Do **not** automatically pull changing upstream code during an active reproduction/implementation cycle.
- Before publishing a MR, fetch latest base, detect divergence, determine whether upstream already fixed the bug, then perform a controlled rebase or equivalent integration if necessary; rerun affected reproduction and regression checks against the updated base.
- If upstream changes make a fix redundant, stop implementation and report the finding rather than creating an unnecessary MR.
- Reassignment of an Asana issue **does not automatically cancel** a run. Continue to a useful result while permissions remain valid and no explicit cancellation has occurred; revalidate authorization before any write or publication. If permissions are revoked, preserve allowed artifacts and stop restricted actions.

### 15.5 Production-only incidents

Production is evidence-only by default: read authorized sanitized logs, traces, metrics, incident reports, feature-flag/config differences and relevant deployment history. Never mutate production, trigger disruptive tests, or access live personal data for reproduction without separately defined authorization and safety controls. Recreate suspected conditions in a sandbox with synthetic or appropriately sanitized data. Record whether the result is an exact reproduction, partial simulation, or merely a plausible hypothesis. If faithful reproduction or safe validation is impossible, produce a fallback research report, with explicit uncertainty and next experiments.

### 15.6 Verification and publication gate

Use a risk-based verification policy:

1. Capture an observable **pre-fix failure** where feasible; prefer a test that fails before and passes after the fix.
2. Ensure the same test assertions and materially equivalent conditions are used before and after; prohibit weakening assertions or deleting failing tests to manufacture success.
3. Run relevant existing unit/integration/e2e tests, and applicable build, lint, type and security checks.
4. Select specialized tests by bug type: concurrency/load, transaction consistency, message retry/idempotency, UI interaction/visual/responsive, accessibility, or network fault injection.
5. Inspect the final diff for scope, regressions, data/security risks, and acceptance-criteria coverage.
6. Record actual command outputs, status and environment; classify skipped/blocked checks explicitly.
7. Open a normal MR only if the configured mandatory checks and evidence gates pass. If reproduction cannot be established, **a draft MR requires explicit human approval** and must disclose all unverified claims. Otherwise use the fallback report.
8. Never merge automatically.

### 15.7 Asana input and output contract

Use Asana OAuth or another explicitly authorized supported connection method. Webhooks are the preferred event source; periodic reconciliation polling catches missed events. Verify webhook authenticity according to the provider's handshake/signature mechanism, fetch the latest task state, confirm the configured user is the assignee at ingestion, and apply eligibility/idempotency checks. Avoid launching on arbitrary comments or edits. Store credentials encrypted and do not log tokens.

The orchestrator shall persist a **typed, versioned run result**. Asana updates and GitLab MR descriptions must be rendered from this result rather than separately invented by specialists. Result states include `FIXED_MR_CREATED`, `RESEARCH_COMPLETED`, `NEEDS_HUMAN_INPUT`, `BLOCKED`, `CANCELLED`, and `FAILED`. Status updates should summarize observable milestones without revealing private reasoning or secrets.

Suggested result envelope (illustrative; implementation may use equivalent schema):

```json
{
  "schema_version": "1.0",
  "run_id": "RUN-1024",
  "asana_issue_id": "123",
  "outcome": "RESEARCH_COMPLETED",
  "repository": {"id": "org/repo", "base_branch": "develop", "base_sha": "..."},
  "summary": "Investigated but could not safely verify a fix",
  "reproduction": {"status": "NOT_REPRODUCED", "evidence": []},
  "hypotheses": [],
  "implementation": null,
  "verification": {"status": "NOT_RUN", "checks": []},
  "merge_request": null,
  "research_report": {"findings": [], "sources": [], "possible_solutions": [], "next_steps": []},
  "human_action_required": false,
  "artifacts": []
}
```

All externally displayed claims must trace to evidence or be explicitly labeled as hypothesis, simulation or recommendation. Artifacts may include sanitized logs, screenshots, traces, test results, patch/diff, and research citations.

---

## 16. Browser-based UI bug investigation and verification

### 16.1 Scope and strategy

The agent shall support browser-driven reproduction for frontend defects, including broken interactions, incorrect displayed state, form validation, navigation, layout/responsiveness, accessibility, and frontend/backend integration failures. Use Playwright as the initial browser automation engine. An LLM may choose actions from a bounded browser-tool interface, but tool execution and safety policies remain controlled by the orchestrator.

### 16.2 Additional functional requirements

**FR-084 (P0):** Authenticate the configured Asana identity, identify its user ID, and ingest eligible assigned tasks through webhook or polling with secure credential handling.

**FR-085 (P0):** Reconcile webhook and polling observations idempotently and avoid duplicate concurrent runs.

**FR-086 (P0):** Resolve and validate the issue's repository and base branch from explicit issue metadata or a configured mapping; do not guess among ambiguous repositories.

**FR-087 (P0):** Pin and persist the investigation base commit SHA and isolate the working tree; do not silently update it mid-investigation.

**FR-088 (P0):** Before MR creation, check upstream divergence, perform a controlled update when needed, and rerun relevant verification.

**FR-089 (P0):** Continue after issue reassignment only while authorized; recheck permissions before write operations and support explicit cancellation.

**FR-090 (P0):** Launch a browser against an isolated running application and execute bounded navigation, click, typing, selection, scrolling and viewport actions.

**FR-091 (P0):** Collect DOM/accessibility snapshots, screenshots, browser console errors, network requests/responses (with sensitive data redacted), and Playwright traces when relevant.

**FR-092 (P0):** Recreate browser context required for reproduction: user role, test login, cookies/session, feature flags, viewport, locale, timezone, seeded data and backend dependencies, where authorized and feasible.

**FR-093 (P0):** Compare actual UI behaviour with issue acceptance criteria and supplied screenshots/videos; distinguish visual differences from proven functional defects.

**FR-094 (P0):** Trace observed browser failures into frontend state, event handlers, API calls, backend responses, and server logs as needed.

**FR-095 (P0):** Generate a repeatable Playwright regression test where feasible and demonstrate red-before/green-after behaviour without weakening assertions.

**FR-096 (P1):** Support screenshot-based visual regression checks with stable fixtures, controlled viewport/fonts/animations, and review of legitimate rendering variation.

**FR-097 (P1):** Support cross-browser, mobile viewport, and accessibility checks according to issue scope and available runners.

**FR-098 (P0):** Save sanitized screenshots, traces, relevant console/network excerpts and test outcomes as investigation artifacts, linked from final report or MR.

**FR-099 (P0):** For production-only UI bugs, use approved observational evidence and sandbox simulations; never run unauthorized disruptive production browser experiments.

**FR-100 (P0):** Use dedicated test accounts and synthetic/seeded data; prevent test actions from sending real notifications, payments or external side effects unless explicitly permitted.

**FR-101 (P0):** Enforce the risk-based verification and MR publication policy in Section 15.6.

**FR-102 (P0):** Persist and publish the standardized versioned output contract for all outcomes, with evidence links and clear verification status.

**FR-103 (P0):** Allow a human-approved draft MR when reproduction remains unavailable but evidence supports a candidate change; otherwise provide research or escalation rather than falsely claiming a verified fix.

### 16.3 Browser investigation example

For a booking form issue where selecting check-out unexpectedly changes check-in:

1. Set up the application, seeded booking data and authorized test login.
2. Navigate to the booking form in a known browser/viewport.
3. Capture initial form values and UI state.
4. Enter check-in and check-out dates using accessible locators.
5. Assert expected values and capture actual behaviour, console/network output and trace.
6. Repeat against the pinned pre-fix commit to establish reproducibility.
7. Trace the event handler/state logic, form hypothesis and implement minimal fix.
8. Rerun the same test after the fix, then run relevant regression and static checks.
9. Attach evidence and disclose any unsupported browser/environment combinations.

### 16.4 UI-specific acceptance tests

**AC-008:** Given a deterministic broken UI interaction, the agent can navigate, reproduce the failure, capture evidence, add a failing Playwright test, fix it, rerun the same test successfully, and create a reviewable MR after required checks.

**AC-009:** Given a visual layout bug with an expected screenshot, the agent records viewport/environment, captures actual screenshot, explains differences, applies a targeted fix, and verifies the corrected layout without claiming pixel-perfect equivalence where rendering differs.

**AC-010:** Given a UI action that returns a successful API response but fails to update visible state, the agent captures both browser and API evidence and investigates across the frontend/backend boundary.

**AC-011:** Given a UI bug that requires unavailable production-only state, the agent attempts safe sandbox recreation and otherwise produces an evidence-backed fallback report without accessing or mutating production.

---

## 17. Final operational acceptance scenarios

**AC-012 — Reassignment:** An issue reassigned mid-run does not automatically terminate; the run rechecks authorization before writes, and stops restricted operations if access is revoked.

**AC-013 — Moving base branch:** A new upstream commit does not silently change the pinned reproduction environment. Before MR, divergence is detected and verification is repeated after controlled integration, or the run reports that the bug was already resolved upstream.

**AC-014 — Ambiguous repository:** An issue with conflicting repository metadata is paused for clarification rather than executed against an arbitrary repository.

**AC-015 — Production-only failure:** The agent uses sanitized observational evidence and synthetic sandbox reproduction; if not reproduced, it produces a report clearly separating facts, hypotheses and proposed tests.

**AC-016 — Verification honesty:** A passing test written after the fix, without a demonstrated pre-fix failure, is not represented as red–green evidence. Required skipped or blocked checks are disclosed; MR policy is enforced.

**AC-017 — Output consistency:** All terminal outcomes produce a versioned run result; Asana summaries and MR descriptions derive from that result, including links to available artifacts.

---

## 18. Production-readiness gates and outstanding architecture decisions

This document is the **final product-requirements baseline for starting architecture and implementation**, not a claim that the implementation is production-ready. Before deploying to untrusted workloads or additional users, the team must validate:

- Threat model, sandbox escape risk, credential boundaries, prompt injection, and outbound network policy.
- Repository/branch ownership, permissions, webhook validation, and idempotent side effects.
- Checkpoint/resume behaviour under process crashes, tool failures and API rate limits.
- A benchmark of historical pre-fix bugs with independent hidden tests; report reproduction rate, verified-fix rate, regression rate, false-success rate, useful fallback rate, runtime and cost.
- Operational SLOs, retention/redaction policy for logs/screenshots, backup/recovery, incident response and approval audit trail.
- Runtime compatibility and supported repository languages/frameworks; document unsupported environments.

**Architecture baseline:** See `architecture.md` v1.0 for finalized HLD choices and remaining validation decisions. Exact hosting, production isolation, secret-store selection, model selection, sizing and retention require validation before production rollout.

**Implementation instruction:** Build and benchmark a local Bug Investigation Engine (including browser reproduction) before adding broad platform infrastructure. Preserve the original FR-001–FR-083, NFR-001–NFR-024 and AC-001–AC-007, and implement the additional v2.1 requirements in Sections 15–17.


## 19. GitLab integration requirements (v2.1)

**FR-104 (P0):** Support configured GitLab project identifiers, instance base URLs and target branches; validate authorization before clone, push and merge request creation.

**FR-105 (P0):** Publish changes through the trusted repository integration, using a deterministic run-specific source branch. Reconcile existing branches and merge requests before retrying ambiguous external writes. Never automatically merge or deploy.

**FR-106 (P0):** Track GitLab pipeline status for the published commit, distinguish local verification from remote CI status, and report pending, failed, skipped or unavailable CI honestly.

**FR-107 (P0):** Keep repository-provider interfaces independent of GitLab-specific identifiers and API response shapes. GitLab is the only required initial implementation.

**AC-018:** Given a verified fix for a configured GitLab project, create exactly one logical MR for the run, link it to Asana and report the corresponding pipeline status. A retry after an ambiguous API response reconciles existing remote state rather than blindly creating another MR.

**AC-019:** Given revoked access, a protected target branch, or an unapproved GitLab host, the publisher must not bypass restrictions; it records the blocked operation and preserves authorized evidence.

**AC-020:** Given a failed or pending GitLab pipeline after local verification, the system reports the actual CI status and never merges or claims remote verification passed.
