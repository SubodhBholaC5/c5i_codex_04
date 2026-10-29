# Function and argument inventory

> Archived pre-refactor discovery snapshot at `d6581c9770db2cd48fc022bb41800d15d58cca2d`. Findings, test-execution status and source line references below describe that revision, not the current working tree. For the final shared-helper architecture and runtime evidence, see [current flow report](../../flow_diagram.md) and [guarded refactor](../../guarded_refactor.md).

[Overview](01-architecture-overview.md) · [Call sites](04-function-call-graph.md) · [Data contracts](05-api-and-data-contracts.md)

All six application functions and all four test classes/five test methods are inventoried below. No application classes or HTTP handlers are registered. None of the functions has type annotations or argument defaults. Types below describe supported intent inferred from arithmetic and fixtures, not enforced contracts. No injected/default services, keyword-only parameters, `*args`, or `**kwargs` occur.

## Application signatures and responsibilities

| Function / source | Responsibility | Arguments and defaults | Return / output |
|---|---|---|---|
| `create_quote(rooms)` — [src/api.py:4-7](../../learner/starter/booking_platform/src/api.py#L4) | Assemble quote response from pricing result | `rooms`: iterable of room mappings; required | Fresh dict with `subtotal`, `service_fee`, `total`; exact values copied from pricing |
| `room_subtotal(rooms)` — [src/pricing.py:3-4](../../learner/starter/booking_platform/src/pricing.py#L3) | Aggregate room-line cost | `rooms`: iterable of room mappings; required. Each line's `rooms` field defaults to `1` only when missing | Numeric sum; `0` (int) for empty input |
| `calculate_quote(rooms)` — [src/pricing.py:6-9](../../learner/starter/booking_platform/src/pricing.py#L6) | Compute subtotal, 10% fee and total | Same required `rooms`; no default | Dict: numeric `subtotal`, rounded `service_fee`, numeric `total`; fee is int for ordinary int/float inputs |
| `preview_invoice(rooms)` — [src/invoice.py:3-7](../../learner/starter/booking_platform/src/invoice.py#L3) | Independently compute invoice preview at 12% | Required `rooms`; a re-iterable collection is needed for both passes to see identical data. Missing line `rooms` defaults to `1` | Dict with same three keys; subtotal from first pass, rounded fee from second pass |
| `cancellation_fee(hours_before_checkin, booking_total)` — [src/pricing.py:11-15](../../learner/starter/booking_platform/src/pricing.py#L11) | Compute historical cancellation charge | `hours_before_checkin`: number comparable with `24`; `booking_total`: number compatible with multiplication by `0.50`; both required | Literal int `0` if hours >= 24, otherwise `round(booking_total * 0.50)` |
| `cancellation_fee_due(hours_before_checkin, booking_total)` — [src/rules.py:2-6](../../learner/starter/booking_platform/src/rules.py#L2) | Separate implementation of same historical charge | Same two required arguments; no defaults | Same expression/branch outputs as preceding row |

Ordinary numeric types evidenced by tests are integers; float inputs are inferred to be usable from the expressions. Arbitrary numeric classes are not guaranteed. In particular, multiplying a Decimal subtotal by a float literal is not a supported contract established here.

## Calls, dependencies, errors and effects

All callers below are static call sites in this repository. Potential consumers outside the checkout are unknown. E1-E5 refer to the error table below.

| Function | Application callees / built-ins | Direct callers | External dependencies | Error behavior | Side effects |
|---|---|---|---|---|---|
| `create_quote` | `calculate_quote`; dictionary key access | QuoteTests' two methods; BaselineContracts' one method | None; local pricing import | E1-E4 propagated from pricing; E5 on changed result shape | No explicit I/O/state writes; passes input through to consuming code |
| `room_subtotal` | `sum`; generator expression; each mapping's `get` | `calculate_quote` only | None | E1-E3, no catches | Consumes input iteration; no explicit input writes |
| `calculate_quote` | `room_subtotal`, `round` | `create_quote` only | None | E1-E4, no catches | Input consumed via subtotal calculation; allocates result dict |
| `preview_invoice` | Two `sum` calls, one `round`, two generator expressions, mapping `get` | InvoiceTests' one method; BaselineContracts' one method | None | E1-E4, no catches | Traverses input twice; one-shot input can be exhausted; allocates result dict |
| `cancellation_fee` | `round` on charged branch only | None found | None | E3-E4, branch-dependent; no catches | No explicit state writes or I/O |
| `cancellation_fee_due` | `round` on charged branch only | RuleTests' one method (two calls) | None | E3-E4, branch-dependent; no catches | No explicit state writes or I/O |

Call evidence: [api:6-7](../../learner/starter/booking_platform/src/api.py#L6), [pricing:4-15](../../learner/starter/booking_platform/src/pricing.py#L4), [invoice:4-7](../../learner/starter/booking_platform/src/invoice.py#L4), [rules:4-6](../../learner/starter/booking_platform/src/rules.py#L4); caller line numbers are tabulated in [call graph](04-function-call-graph.md).

| ID | Inferred failure | Source operation | Handling |
|---|---|---|---|
| E1 | `TypeError` for non-iterable rooms or non-subscriptable line | Generator iteration and `room[...]` | Propagates to caller |
| E2 | `KeyError` for absent `nightly_rate` or `nights`; `AttributeError` for line without `.get` when preceding accesses succeed | [pricing:4](../../learner/starter/booking_platform/src/pricing.py#L4), [invoice:4,6](../../learner/starter/booking_platform/src/invoice.py#L4) | No validation/catch |
| E3 | `TypeError` for incompatible multiplication/addition/comparison | Line arithmetic, fee arithmetic, `hours >= 24` | Propagates; some inappropriate inputs may instead produce misleading values |
| E4 | `ValueError`/`OverflowError` when ordinary float fee becomes NaN/infinity and reaches `round` | [pricing:8,15](../../learner/starter/booking_platform/src/pricing.py#L8), [invoice:6](../../learner/starter/booking_platform/src/invoice.py#L6), [rules:6](../../learner/starter/booking_platform/src/rules.py#L6) | Propagates; not runtime-tested |
| E5 | `KeyError` if pricing result lacks any expected field | [api:7](../../learner/starter/booking_platform/src/api.py#L7) | Propagates; return-type changes could also produce `TypeError` |

There are no explicit raises, exception handlers, logs, database writes, network calls, filesystem writes or mutable globals in application source. User-supplied iterables/mappings can have their own effects or raise other exceptions; their implementations are outside this analysis.

## Test classes and methods

All classes inherit `unittest.TestCase`, declare no custom constructor or fields, and serve as test containers. All methods accept only implicit `self`, have no explicit return (normally `None`), and are discoverable by `unittest`. Assertion failures raise `AssertionError`; application exceptions are not caught in test bodies. No fixtures or teardown methods occur.

| Class / method and source | Purpose and argument fixture | Calls / output checked | Known caller and effects |
|---|---|---|---|
| `QuoteTests.test_one_room_quote` — [test_quote.py:4-7](../../learner/starter/booking_platform/tests/test_quote.py#L4) | `[{nightly_rate:100, nights:2}]` | `create_quote`; `self.assertEqual` dict `{subtotal:200, service_fee:20, total:220}` | `unittest` discovery by convention; no explicit I/O |
| `QuoteTests.test_quote_response_keys` — [test_quote.py:9-10](../../learner/starter/booking_platform/tests/test_quote.py#L9) | `[]` | `create_quote`, `set`, `self.assertEqual`; exact three keys | Same runner; local allocation only |
| `InvoiceTests.test_legacy_invoice_example` — [test_invoice.py:4-8](../../learner/starter/booking_platform/tests/test_invoice.py#L4) | `[{nightly_rate:100, nights:2}]` | `preview_invoice`, `self.assertEqual`; `{subtotal:200, service_fee:24, total:224}` | Same runner; explicitly historical expectation |
| `RuleTests.test_legacy_cancellation_boundary` — [test_rules.py:4-7](../../learner/starter/booking_platform/tests/test_rules.py#L4) | `(24,200)`, `(23,200)` | Two `cancellation_fee_due`, two `self.assertEqual`; `0`, `100` | Same runner; historical boundary only |
| `BaselineContracts.test_empty_inputs_preserve_numeric_shape` — [test_baseline_contracts.py:5-10](../../learner/starter/booking_platform/tests/test_baseline_contracts.py#L5) | Both `create_quote([])` and `preview_invoice([])` | Both calls evaluated before loop; per result `assertEqual(subtotal,0)`, `assertEqual(total,0)`, `assertIsInstance(service_fee,int)` | Same runner; fee numeric value itself is not asserted |

No application function is called by a test class definition at import time; test methods are invoked by the runner. Runtime discovery and assertion outcomes have not been verified.
