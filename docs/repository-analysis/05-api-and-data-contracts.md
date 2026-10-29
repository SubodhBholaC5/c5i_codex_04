# API and data contracts

[Inventory](03-function-inventory.md) · [Flows](06-execution-flow.md) · [Risks](07-change-impact-analysis.md)

## Transport boundary

`src/api.py` describes HTTP-facing orchestration but explicitly omits the network layer ([api.py:1](../../learner/starter/booking_platform/src/api.py#L1)). No URL, verb, HTTP status, JSON serializer, auth middleware, request parser, OpenAPI schema or error-response mapping is implemented. The verified API is the Python function boundary. No database schema, migrations, tables or persistence contract is present.

## Room input contract

All field types here are inferred from arithmetic and integer fixtures, not annotations or runtime validation.

| Field / boundary | Required? / default | Expected shape | Consumers / evidence |
|---|---|---|---|
| Function argument `rooms` | Required; no default | Iterable of mapping-like room lines; tests use lists of dicts | [api:4-6](../../learner/starter/booking_platform/src/api.py#L4), [test_quote:6](../../learner/starter/booking_platform/tests/test_quote.py#L6) |
| Line `nightly_rate` | Required | Numeric nightly price; currency and units not declared beyond the name | [pricing:4](../../learner/starter/booking_platform/src/pricing.py#L4), [invoice:4,6](../../learner/starter/booking_platform/src/invoice.py#L4) |
| Line `nights` | Required | Numeric multiplier, expected night count; integrality/positivity not enforced | Same expressions |
| Line `rooms` | Optional; defaults to `1` if absent | Numeric room-count multiplier; zero, negative or `None` not normalized | `room.get("rooms", 1)` in the same expressions |
| Other line keys | Ignored by current calculations | Unspecified | Only the three fields above are accessed |

`rooms` names both the outer collection argument and a per-line count. These are different values. Passing `None` as the count does not activate the missing-field default. Quote consumes the iterable once; invoice consumes it twice, so reusable collection input is required for consistent invoice results. This distinction is visible at [pricing:4](../../learner/starter/booking_platform/src/pricing.py#L4) and [invoice:4-6](../../learner/starter/booking_platform/src/invoice.py#L4).

## Output and arithmetic contracts

Let `L_i = nightly_rate_i * nights_i * rooms_i`, with absent `rooms_i = 1`.

| Function | Subtotal | Service fee | Total / shape |
|---|---|---|---|
| `room_subtotal` | `S = sum(L_i)` | N/A | Returns `S` directly |
| `calculate_quote` | `S` | `F = round(S * 0.10)` | `{"subtotal": S, "service_fee": F, "total": S + F}` |
| `create_quote` | Copies result key | Copies result key | Fresh dict containing exactly the three keys above |
| `preview_invoice` | `S = sum(L_i)` first traversal | `F = round(sum(L_i * 0.12))` second traversal | Same three-key dict using `S + F` |

Evidence: [pricing:3-9](../../learner/starter/booking_platform/src/pricing.py#L3), [api:6-7](../../learner/starter/booking_platform/src/api.py#L6), [invoice:4-7](../../learner/starter/booking_platform/src/invoice.py#L4). Invoice rounds the aggregate fee once. Summing floating-point line fees is not guaranteed to be bitwise identical to multiplying an aggregate subtotal, even apart from the rate difference.

For ordinary integer/float inputs, `round(value)` without digits produces an integer fee, while subtotal/total follow input arithmetic types. Empty `sum` starts at integer `0`. The tests assert an integer empty-input fee, not a universal schema for all possible numeric classes ([baseline:7-10](../../learner/starter/booking_platform/tests/test_baseline_contracts.py#L7)).

## Cancellation contract

| Argument / result | Current behavior | Limit |
|---|---|---|
| `hours_before_checkin` | Required; compared with literal `24` | No date, property timezone or timezone conversion accepted |
| `booking_total` | Required; charged branch multiplies by float `0.50` | Free branch returns without using it; no validation |
| Result | `0` if hours >= 24, else `round(booking_total * 0.50)` | Scalar numeric fee; no booking state mutation |

Both implementations: [pricing:11-15](../../learner/starter/booking_platform/src/pricing.py#L11), [rules:2-6](../../learner/starter/booking_platform/src/rules.py#L2). How hours are derived is unknown. The approved sheet establishes the 48-hour local-time window but does not specify the numeric standard fee; the current 50% is implementation evidence, not an independently approved policy ([learner policy:20](../../learner/policy/business_rules.md#L20)).

## Evidence-backed examples and policy conflicts

Examples marked derived are hand-traced from code, not executed.

| Input / case | Current quote | Current invoice / cancellation | Evidence / intent |
|---|---|---|---|
| `[{"nightly_rate":100,"nights":2}]` | `{200,20,220}` in subtotal/fee/total order | Invoice `{200,24,224}` | Asserted in [quote:6-7](../../learner/starter/booking_platform/tests/test_quote.py#L6), [invoice:7-8](../../learner/starter/booking_platform/tests/test_invoice.py#L7); invoice violates approved 10% |
| `[]` | Derived `{0,0,0}` | Derived `{0,0,0}` | Partial assertions in [baseline:7-10](../../learner/starter/booking_platform/tests/test_baseline_contracts.py#L7) |
| Two lines, each rate `5`, nights `1` | Derived subtotal `10`, fee `1`, total `11` | Derived subtotal `10`, fee `1`, total `11` | Agreement for this rounded example does not establish policy equivalence; see formulas above |
| `[{"nightly_rate":100,"nights":2,"rooms":2}]` | Derived `{400,40,440}` | Derived `{400,48,448}` | Count multiplier from [pricing:4](../../learner/starter/booking_platform/src/pricing.py#L4), [invoice:4-6](../../learner/starter/booking_platform/src/invoice.py#L4); no matching test |
| Cancellation `(24,200)` / `(23,200)` | N/A | `rules` assertions `0` / `100` | [rules test:6-7](../../learner/starter/booking_platform/tests/test_rules.py#L6) |
| Cancellation `(36,200)` | N/A | Derived `0` in both helpers | Conflicts with free-cancellation cutoff of 48 hours; numeric fee under intended policy unspecified |

Approved policy requires a single aggregate 10% fee, equal quote/invoice pricing, Python rounding once, and a 48-hour local-check-in cancellation window ([app policy:5-10](../../learner/starter/booking_platform/policy/business_rules.md#L5)). Quote follows the stated fee expression for ordinary valid inputs; invoice differs in rate and floating operation order. Both cancellation helpers differ in cutoff and omit timezone handling. No behavior was corrected during this analysis.

## Validation and error contract

No input validator, schema or custom error type is implemented. Built-in exceptions propagate and are not translated into HTTP responses. Negative values, fractional counts and non-finite numbers are not explicitly rejected. See the [error inventory](03-function-inventory.md) and [runtime validation proposals](08-unknowns-and-validations.md).
