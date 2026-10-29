# Unknowns and validation record

[Overview / inventory](01-architecture-overview.md) · [Risks](07-change-impact-analysis.md) · [Contracts](05-api-and-data-contracts.md)

## Scope and evidence discipline

Snapshot inspected: commit `d6581c9770db2cd48fc022bb41800d15d58cca2d`, branch `subodh/training/baseline_repo`; initial working tree was clean. Analysis date: 2026-10-29. All 24 tracked files were inventoried; all Python source/test files and all twelve Markdown files plus the prompt template were read. `.DS_Store` was inventoried only, not parsed. `.git` internals, credentials and host environment values were not analyzed.

The task authorizes only creation of these eight analysis reports. Application source, tests, policy, dependencies, configuration and existing documentation were left untouched. Training walkthrough prompts propose future refactors; those are not implemented code or authorization to perform them. In particular, `src/shared/pricing.py` is mentioned in [walkthrough:37](../../learner/walkthrough.md#L37) but does not exist in the checkout.

Evidence labels used throughout:

| Label | Meaning | Example |
|---|---|---|
| Confirmed static fact | Directly inspected code, assertion, policy text or inventory | Invoice literal `0.12` at [invoice:6](../../learner/starter/booking_platform/src/invoice.py#L6) |
| Static inference | Consequence of an expression, not reproduced in runtime | Invoice's two passes can exhaust a one-shot iterable |
| Intended policy | Approved Markdown source, not necessarily implemented | Aggregate 10% fee and 48-hour cutoff at [policy:5-10](../../learner/starter/booking_platform/policy/business_rules.md#L5) |
| Unknown | Evidence does not establish answer | Deployment, runtime version, external callers |

No external deployment, HTTP route, database, job, caller or function has been assumed. Inferred numeric types and hand-traced examples are explicitly labeled; they are not annotations or runtime observations.

## Unknowns and next validations

These are proposals only. Running tests/application code, installing packages or changing implementation requires subsequent authorization under this discovery's constraints.

| ID | Unknown / unresolved decision | Current evidence | Smallest next validation |
|---|---|---|---|
| U1 | Actual Python version, module path and runtime test results | No runtime declaration; [README:3-6](../../learner/starter/booking_platform/README.md#L3) supplies unittest command | After execution approval, identify runtime version and run documented unittest discovery from app root using the available Python command |
| U2 | Real HTTP transport, deployment, users, external consumers | [api:1](../../learner/starter/booking_platform/src/api.py#L1) says network layer omitted; no launcher/deployment spec | Ask owner whether another repository implements dispatch/consumers; inspect that source if supplied |
| U3 | Supported input schema and invalid-input policy | No validators/annotations; only integer-list fixtures | Confirm allowed currency, numeric type, negative values, integral counts, missing values and error contract with owner |
| U4 | Standard cancellation fee under approved policy | Code charges 50%; [learner policy:20](../../learner/policy/business_rules.md#L20) does not define its numeric value | Obtain approved fee schedule; do not assume current 50% is authorized |
| U5 | Property timezone, hour derivation, DST and fractional boundary handling | Helpers accept numeric hours only; [policy:10](../../learner/starter/booking_platform/policy/business_rules.md#L10) requires local check-in basis | Find upstream date/time conversion or obtain required semantics and boundary examples |
| U6 | Whether invoice callers pass one-shot/mutable iterables | Two traversals at [invoice:4-6](../../learner/starter/booking_platform/src/invoice.py#L4); tests use lists | Inspect actual consumers; after approval, compare list vs iterator inputs using identical lines |
| U7 | Float rounding and aggregate-fee edge results in target runtime | Single rounding sites; no edge-case tests | After approval, characterize ties, multiple lines, large values, non-finite values and float representations |
| U8 | Usage of `pricing.cancellation_fee` outside checkout | No static callers found | Search consuming repositories/interfaces before deletion or signature change |
| U9 | Executed line/branch coverage and test baseline | Only static inspection of five methods | After approval, run existing tests; coverage instrumentation would require separate tooling/dependency approval if unavailable |
| U10 | Owner-approved migration from historical tests to policy-correct behavior | [Invoice test:6-8](../../learner/starter/booking_platform/tests/test_invoice.py#L6), [rules test:5-7](../../learner/starter/booking_platform/tests/test_rules.py#L5) conflict with policy | Product/policy owner approves expected behavior and compatibility scope before a code-change task |

Both approved policy copies agree on the fee and cancellation window. The app-local copy is the source named by [AGENTS.md:3](../../learner/starter/booking_platform/AGENTS.md#L3). Instructor challenge prose is not authoritative behavior evidence; [seeded explanation:5-7](../../learner/review_packet/seeded_codex_explanation.md#L5) explicitly identifies a misleading claim.

## Static validation performed

| Operation | Purpose | Result / limitation |
|---|---|---|
| `rg --files --hidden` excluding `.git`, dependency/virtualenv directories | Repository file inventory | Only the compact training tree found; no manifests or deployment/configuration files |
| `git ls-files`, `git status --short --branch`, `git rev-parse HEAD` | Tracked scope, initial cleanliness, snapshot identity | 24 initial tracked files; clean initial worktree; commit recorded above |
| `Get-Content` with numbered output | Inspect all source, tests, policies and learning material | Four implementation modules, six functions, four test classes, five test methods; source citations recorded |
| `rg -n` for imports, definitions, named call sites and integration indicators | Static dependency/caller cross-check | One application import edge and two application call edges; test edges inventoried; no integration implementation found |
| New-report link and line-anchor checks; `git diff` / status | Validate report references and unchanged existing files | Eight reports; 179 local links checked; zero missing targets or out-of-range line anchors; all code fences paired |

Final Git verification: `git diff --exit-code` and `git diff --cached --exit-code` both succeeded with no tracked changes; `git ls-files --others --exclude-standard` listed only these eight reports. `git diff --check` reported no tracked whitespace errors. Mermaid diagrams were inspected as source and their fences checked; no Mermaid renderer was run, so rendered layout is unverified.

No application import, test invocation, dependency installation, package audit, external service query, performance benchmark or coverage measurement was performed. The absence of third-party imports gives no basis to make claims about an undisclosed host environment's packages or security status.

## Report index

1. [Architecture and repository map](01-architecture-overview.md)
2. [Module and library dependencies](02-module-dependency.md)
3. [Function signatures, callers, exceptions and side effects](03-function-inventory.md)
4. [Caller/callee graph and fan-in counts](04-function-call-graph.md)
5. [API and data contracts, policy conflicts](05-api-and-data-contracts.md)
6. [Five execution flows and diagrams](06-execution-flow.md)
7. [Ten change-risk hotspots and test gaps](07-change-impact-analysis.md)
8. [Unknowns and validation record](08-unknowns-and-validations.md)
