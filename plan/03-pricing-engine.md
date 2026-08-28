# 03 — Pricing engine

This is the part of the system with a provably correct answer, and the part jewellers judge you on. It is
built first, before any AI, and it has the heaviest test coverage in the codebase.

**Nothing in this document involves a language model.** That is the point.

---

## Two rules that are absolute

1. **`Decimal` everywhere.** No `float` under `app/pricing/`, enforced by a CI grep. Money is
   `numeric(14,2)`, weights `numeric(10,3)`.
2. **The model never sees, emits or influences a price.** It returns `{value: 5, unit: "pavun"}`;
   this module converts and computes.

---

## Unit conversion

The largest error magnitude in the system. A *pavun* / *sovereign* is **8 grams** — a mis-parse is an 8×
price error, not a rounding one.

```python
# app/pricing/units.py
from decimal import Decimal

GRAMS: dict[str, Decimal] = {
    "g":         Decimal("1"),
    "gram":      Decimal("1"),
    "grams":     Decimal("1"),
    "pavun":     Decimal("8"),      # Tamil
    "pavan":     Decimal("8"),
    "sovereign": Decimal("8"),
    "savaran":   Decimal("8"),
    "pon":       Decimal("8"),
    "tola":      Decimal("11.6638"),
    "kg":        Decimal("1000"),
}

def to_grams(value: Decimal, unit: str) -> Decimal:
    key = unit.strip().lower()
    if key not in GRAMS:
        raise UnknownUnitError(unit)
    return value * GRAMS[key]
```

Currency words matter just as much and belong in the same tested table:

```python
MULTIPLIER: dict[str, Decimal] = {
    "k": Decimal("1000"), "thousand": Decimal("1000"),
    "l": Decimal("100000"), "lakh": Decimal("100000"), "lac": Decimal("100000"),
    "lakhs": Decimal("100000"), "லட்சம்": Decimal("100000"),
    "cr": Decimal("10000000"), "crore": Decimal("10000000"),
}
```

`UnknownUnitError` must be raised, never defaulted. A silent fallback to grams is exactly the bug that
produces an 8× wrong quote.

---

## Rounding

One quantizer, applied at named steps only.

```python
# app/pricing/rounding.py
from decimal import Decimal, ROUND_HALF_UP

TWO = Decimal("0.01")

def money(x: Decimal) -> Decimal:
    return x.quantize(TWO, rounding=ROUND_HALF_UP)
```

Apply `money()` at exactly these points and nowhere else: `metal_value`, `wastage_amount`,
`making_charge_amount`, `line_subtotal`, `gst_amount`, `line_total`. Rounding at intermediate points not
on this list is how the PDF total ends up one rupee off the sum of its lines — and jewellers are extremely
numerate, so that destroys credibility faster than a missing feature.

---

## The formula

The spec's §8 formula, corrected for how Indian GST actually works.

```
metal_value          = gold_rate_per_gram × net_gold_weight_g
wastage_amount       = metal_value × wastage_pct / 100

making_charge_amount = depends on making_charge_type:
    percentage  →  metal_value × making_charge_value / 100
    per_gram    →  net_gold_weight_g × making_charge_value
    fixed       →  making_charge_value
  then: max(making_charge_amount, min_making_charge) if min_making_charge is set

stone_charge         = configured per product

line_subtotal        = metal_value + wastage_amount + making_charge_amount + stone_charge
```

### GST — the correction to the spec

Spec §8 models GST as `subtotal × configured_percentage`. That is wrong. Indian gold jewellery is taxed
at **3% on the gold/metal value and 5% on making charges**, and which applies depends on how the shop bills:

**`gst_mode = 'composite'`** — gold and making sold as a single supply. The whole invoice is taxed at 3%
because gold is the principal supply. Many shops bill this way.

```
gst_amount = line_subtotal × gst_metal_pct / 100
```

**`gst_mode = 'itemised'`** — making charges shown as a separate line.

```
metal_portion  = metal_value + wastage_amount + stone_charge
making_portion = making_charge_amount
gst_amount     = money(metal_portion  × gst_metal_pct  / 100)
               + money(making_portion × gst_making_pct / 100)
```

Both are legitimate and shops differ, so `gst_mode` is a per-organisation setting **frozen onto the
quotation at creation**. Computing tax on the subtotal with a single rate produces invoices that are
quietly non-compliant and slightly wrong on every quote — the kind of bug a shop's accountant finds six
months in.

```
line_total = line_subtotal + gst_amount
```

Discounts, when present, apply to `line_subtotal` **before** GST.

---

## Rule resolution

See [02](02-database-schema.md#gold-rates-and-pricing-rules) for the SQL. Precedence:

```
product-level override  >  category+purity (3)  >  category (2)  >  purity (1)  >  org default (0)
```

Whole-record, never field-merged. Every org is seeded with a specificity-0 default at signup so resolution
can never return NULL.

---

## The snapshot

Every quotation must remain reproducible after the gold rate changes. This is the spec's central
correctness requirement.

**At creation**, copy these onto `quotation_items` as typed columns:

```
purity_label · gross_weight_g · net_gold_weight_g · stone_weight_g
gold_rate_per_gram · wastage_pct · making_charge_type · making_charge_value
stone_charge · gst_metal_pct · gst_making_pct
```

plus the six computed outputs.

**At read**, the API returns the frozen columns verbatim. **Stored quotes are never recomputed on read.**

A separate `POST /quotations/{id}/reproduce` endpoint exists solely so the test suite can prove the
snapshot still reproduces. The UI never calls it.

`pricing_rule_id` and `gold_rate_id` are stored for provenance — so an audit can answer *"which rule was
in force"* — and are **never dereferenced when reproducing**.

### Engine versioning

```python
PRICING_ENGINES: dict[str, type[PricingEngine]] = {"1.0.0": EngineV1}
CURRENT_ENGINE_VERSION = "1.0.0"
```

`quotations.pricing_engine_version` records which one ran. A formula change means a new version and a new
class; old quotes reprint through the old one. This is what makes a refactor safe.

---

## Interface

```python
# app/pricing/engine.py

@dataclass(frozen=True)
class LineInput:
    purity_label: str
    net_gold_weight_g: Decimal
    gross_weight_g: Decimal
    stone_weight_g: Decimal = Decimal(0)
    stone_charge: Decimal = Decimal(0)
    category_id: UUID | None = None
    product_id: UUID | None = None
    description: str = ""

@dataclass(frozen=True)
class ResolvedPricing:      # everything frozen onto the row
    gold_rate_per_gram: Decimal
    wastage_pct: Decimal
    making_charge_type: Literal["percentage", "per_gram", "fixed"]
    making_charge_value: Decimal
    min_making_charge: Decimal | None
    gst_metal_pct: Decimal
    gst_making_pct: Decimal
    gst_mode: Literal["composite", "itemised"]
    pricing_rule_id: UUID
    gold_rate_id: UUID

@dataclass(frozen=True)
class LineResult:
    metal_value: Decimal
    wastage_amount: Decimal
    making_charge_amount: Decimal
    line_subtotal: Decimal
    gst_amount: Decimal
    line_total: Decimal

class PricingEngine(Protocol):
    version: str
    def price_line(self, inp: LineInput, res: ResolvedPricing) -> LineResult: ...
```

`resolve_pricing(org_id, category_id, purity_label, at) -> ResolvedPricing` lives in
`app/pricing/rules.py` and does the SQL lookup. Keeping resolution separate from calculation is what makes
`price_line` a pure function — and therefore trivially testable and reproducible.

---

## Budget → weight inversion

The retriever needs "which products fit a ₹3 lakh budget" without reimplementing pricing in SQL. Resolve
the rule and rate once in Python, then **invert the formula to a maximum affordable net weight** and pass
that as a plain bound:

```python
def max_net_weight_for_budget(budget: Decimal, res: ResolvedPricing) -> Decimal:
    r = res.gold_rate_per_gram
    w = res.wastage_pct / 100
    g = res.gst_metal_pct / 100

    per_gram = r * (1 + w)
    if res.making_charge_type == "percentage":
        per_gram += r * res.making_charge_value / 100
    elif res.making_charge_type == "per_gram":
        per_gram += res.making_charge_value
    # 'fixed' contributes a constant, handled by the caller subtracting it from budget first

    return budget / (per_gram * (1 + g))
```

This keeps exactly one implementation of the price formula in the codebase. Exact prices are then computed
with the real engine over the ~12 candidates the SQL returns.

---

## Tests

This module is where tests actually pay. Put them in `tests/unit/test_pricing.py` and
`tests/unit/test_units.py`.

### Table-driven cases

- Each making-charge type: `percentage`, `per_gram`, `fixed`
- `min_making_charge` floor applied and not applied
- Both `gst_mode` values against the same inputs
- Rule precedence: all four specificity levels plus the product override
- Unit conversion across gram / pavun / sovereign / tola, and `UnknownUnitError` on garbage
- Currency words: `3 lakh`, `3L`, `300k`, `₹3,00,000`

### Property tests (Hypothesis)

```python
@given(lines=st.lists(line_inputs(), min_size=1, max_size=10))
def test_total_equals_sum_of_lines(lines):
    q = build_quotation(lines)
    assert sum(i.line_total for i in q.items) == q.total_amount   # exactly, not approximately

@given(line=line_inputs())
def test_snapshot_reproduces_exactly(line):
    item = create_item(line)
    assert recompute_from_snapshot(item) == LineResult(
        item.metal_value, item.wastage_amount, item.making_charge_amount,
        item.line_subtotal, item.gst_amount, item.line_total,
    )
```

### Golden files

About 20 realistic quotes in `tests/fixtures/golden_quotes/*.json`. Any engine change that alters an
expected output fails CI and forces an explicit `pricing_engine_version` bump plus a new golden set — which
makes the version bump impossible to forget.

### The reproducibility test that matters most

```python
def test_quote_survives_a_rate_change(db):
    quote = create_quote(rate=Decimal("7200"))
    original_total = quote.total_amount
    set_gold_rate(Decimal("7800"))              # new append-only row
    assert reload(quote).total_amount == original_total
```

### CI guard

```yaml
- name: no floats in pricing
  run: |
    ! grep -rnE '\bfloat\b' backend/app/pricing/ \
      || (echo "float found under app/pricing — use Decimal" && exit 1)
```

---

## Gold rate entry — a product decision, not a technical one

**Do not build an API integration first.**

Indian retail 22K rates are not spot gold. Every shop sets its own daily rate and it differs between shops
on the same street. The correct primary mechanism is the shop owner typing today's rate into a box each
morning — ten seconds, and something they already do.

An API feed (IBJA, GoldAPI.io) is a *convenience that prefills that box*, added later. Building it the
other way round produces wrong prices and an external dependency you did not need.

Entry writes a new `gold_rates` row and closes the previous one by setting `effective_to`. Never update.
