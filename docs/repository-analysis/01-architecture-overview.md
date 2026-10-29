# Architecture overview

> Archived pre-refactor discovery snapshot at `d6581c9770db2cd48fc022bb41800d15d58cca2d`. Findings, test-execution status and source line references below describe that revision, not the current working tree. For the final shared-helper architecture and runtime evidence, see [current flow report](../../flow_diagram.md) and [guarded refactor](../../guarded_refactor.md).

Analysis date: 2026-10-29. Method: read-only source inspection and repository-wide text searches; application and tests were not executed. Only these new analysis documents were written. Paths below are repository-relative; linked source citations include line anchors. Statements about absent components apply only to this checkout.

## Executive summary

This is a compact Python booking-pricing training fixture, rather than a deployed web application. It implements procedural, in-memory calculations with one orchestration layer: `src.api.create_quote` calls `src.pricing.calculate_quote`, which calls `room_subtotal`. Invoice preview calculates independently. Two separate cancellation helpers have no connection to the quote/invoice paths. There are no application classes, framework imports, database clients, external API clients, background jobs, or executable launchers in the inspected source. [api.py:1-7](../../learner/starter/booking_platform/src/api.py#L1), [pricing.py:1-15](../../learner/starter/booking_platform/src/pricing.py#L1), [invoice.py:1-7](../../learner/starter/booking_platform/src/invoice.py#L1), [rules.py:1-6](../../learner/starter/booking_platform/src/rules.py#L1).

Two business-policy conflicts are confirmed: invoice uses 12%, rather than the approved 10%; both cancellation helpers allow free cancellation from 24 hours, rather than 48. Existing invoice and cancellation tests preserve historical behavior. These are documented findings, not changes. [Invoice implementation:6](../../learner/starter/booking_platform/src/invoice.py#L6), [pricing:13-15](../../learner/starter/booking_platform/src/pricing.py#L13), [rules:4-6](../../learner/starter/booking_platform/src/rules.py#L4), [approved policy:5-10](../../learner/starter/booking_platform/policy/business_rules.md#L5).

## Inventory and analysis plan

The initial inventory contained 24 tracked files: ten Python files (four implementation modules, four test modules, two empty package initializers), twelve Markdown files, one extensionless analysis-prompt template, and `.DS_Store`. No dependency manifest, lockfile, application configuration, CI workflow, container spec, or deployment spec was found using `rg --files --hidden -g '!.git'` and `git ls-files`. These eight new reports are additional files.

| Phase | Inspection scope | Verified result / report |
|---|---|---|
| 1. Discover | All tracked files, local guidance, source, policy | This overview; repository is a training fixture |
| 2. Inventory functions | All source definitions, signatures, tests | [Function inventory](03-function-inventory.md) |
| 3. Trace dependencies | Imports and all named call sites, including tests | [Module map](02-module-dependency.md), [call graph](04-function-call-graph.md) |
| 4. Trace execution | Quote, invoice, both cancellation functions, baseline checks | [Contracts](05-api-and-data-contracts.md), [flows](06-execution-flow.md) |
| 5. Assess impact | Compare implementation, assertions, approved policy | [Ten hotspots](07-change-impact-analysis.md) |
| 6. Validate documentation | Source citations, report links, unchanged tracked files | [Unknowns and validation record](08-unknowns-and-validations.md) |

## Repository module map

```text
repository/
├── master_template_brownfield application  # Analysis prompt, not executable code
├── .DS_Store                             # OS metadata
├── learner/
│   ├── README_FOR_LEARNERS.md, brief.md, walkthrough.md
│   ├── policy/business_rules.md           # Approved learning policy
│   ├── review_packet/                    # Instructor challenge materials
│   ├── worksheets/                       # Blank evidence/review templates
│   └── starter/booking_platform/
│       ├── AGENTS.md, README.md           # Local guidance and test command
│       ├── policy/business_rules.md       # App-local approved policy
│       ├── src/
│       │   ├── __init__.py                # Empty package marker
│       │   ├── api.py                     # Quote response orchestration
│       │   ├── pricing.py                 # Subtotal, quote fee, cancellation
│       │   ├── invoice.py                 # Independent invoice calculation
│       │   └── rules.py                   # Independent cancellation helper
│       └── tests/
│           ├── __init__.py                # Empty package marker
│           ├── test_quote.py             # Two quote assertions/methods
│           ├── test_invoice.py           # One historical invoice test
│           ├── test_rules.py             # One historical cancellation test
│           └── test_baseline_contracts.py # One empty-input contract test
└── docs/repository-analysis/              # This analysis, eight new reports
```

Folder purpose is supported by inspected content: [learner route:3-10](../../learner/README_FOR_LEARNERS.md#L3), [brief:3-15](../../learner/brief.md#L3), [review challenge:3-7](../../learner/review_packet/seeded_codex_explanation.md#L3), [worksheet:3-9](../../learner/worksheets/architecture_and_drift.md#L3), and source/test citations in the inventories.

## Entry points, configuration, integrations

| Area | Confirmed finding | Evidence / limit |
|---|---|---|
| Public application boundary | `create_quote(rooms)`, `preview_invoice(rooms)` are callable functions, not registered HTTP routes | [README:9](../../learner/starter/booking_platform/README.md#L9); [api:4-7](../../learner/starter/booking_platform/src/api.py#L4); [invoice:3-7](../../learner/starter/booking_platform/src/invoice.py#L3) |
| Other callable boundaries | `calculate_quote`, `room_subtotal`, and two cancellation helpers | [pricing:3-15](../../learner/starter/booking_platform/src/pricing.py#L3); [rules:2-6](../../learner/starter/booking_platform/src/rules.py#L2) |
| Test runner | Documented `python3 -m unittest discover -s tests -v` from application root; not run | [README:3-6](../../learner/starter/booking_platform/README.md#L3) |
| Configuration | Rates, room-count default, cancellation threshold and fee fraction are inline literals; policy Markdown is not loaded by code | [pricing:4-15](../../learner/starter/booking_platform/src/pricing.py#L4); [invoice:4-6](../../learner/starter/booking_platform/src/invoice.py#L4); [rules:4-6](../../learner/starter/booking_platform/src/rules.py#L4) |
| Environment | No environment-variable reads in the four source modules; no `.env` in inventory | All source inspected; no host environment values collected |
| Services/storage/network/jobs | No database, HTTP transport, external API, queue, scheduler, or background worker implementation | All source inspected; omitted runtime integrations cannot be inferred |
| Runtime/dependencies | Python syntax; standard-library `unittest` only in tests; no declared Python/library versions | [test_quote:1-4](../../learner/starter/booking_platform/tests/test_quote.py#L1); full import inventory in [dependencies](02-module-dependency.md) |

## Confidence boundaries

**Confirmed:** symbols, formulas, static call sites, test assertions and policy conflicts. **Static inference:** expected numeric input/output types, propagation of built-in exceptions, iterable exhaustion and potential downstream change effects. **Unknown:** runtime version, execution/test outcomes, deployment, external consumers, currency, validation requirements and timezone conversion. No external architecture is assumed. See [unknown register](08-unknowns-and-validations.md).
