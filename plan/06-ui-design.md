# 06 — UI design

The product interface is **white and gold**. A published reference implementation of this palette is the
plan artifact at <https://claude.ai/code/artifact/867da382-d10c-4ece-9058-47bc391774e4>.

---

## Tokens

Define these once in `globals.css` as CSS custom properties and mirror them in `tailwind.config.ts`.
**Never write a raw hex in a component.** That single discipline is what keeps a solo-built UI coherent
across fourteen weeks.

| Token | Light | Dark | Use |
|---|---|---|---|
| `ground` | `#FFFFFF` | `#14120E` | Page background |
| `surface` | `#FBF9F3` | `#1C1914` | Cards, table bodies |
| `surface-2` | `#F4F0E4` | `#26221A` | Table headers, active rows |
| `ink` | `#1A1712` | `#F2EEE4` | Primary text |
| `ink-soft` | `#45403A` | `#CFC8BA` | Secondary text |
| `muted` | `#7A736A` | `#968E80` | Labels, captions |
| `rule` | `#EAE4D6` | `#2E2A22` | Hairlines |
| `rule-strong` | `#D6CCB4` | `#453E32` | Borders, dividers |
| `gold` | `#9E7C1F` | `#D4AF37` | **Text and icons** |
| `gold-bright` | `#C9A227` | `#E3C765` | **Fills and rules only** |

```css
:root {
  --ground: #FFFFFF;  --surface: #FBF9F3;  --surface-2: #F4F0E4;
  --ink: #1A1712;     --ink-soft: #45403A; --muted: #7A736A;
  --rule: #EAE4D6;    --rule-strong: #D6CCB4;
  --gold: #9E7C1F;    --gold-bright: #C9A227;
  /* semantic — deliberately not gold */
  --cold: #5C7186;    --warm: #A66A1C;     --hot: #A03428;
  --won: #2F6B4F;     --lost: #7A736A;
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --ground: #14120E;  --surface: #1C1914;  --surface-2: #26221A;
    --ink: #F2EEE4;     --ink-soft: #CFC8BA; --muted: #968E80;
    --rule: #2E2A22;    --rule-strong: #453E32;
    --gold: #D4AF37;    --gold-bright: #E3C765;
    --cold: #8FA7BD;    --warm: #DDA84B;     --hot: #DE7264;
    --won: #6FAE8C;     --lost: #968E80;
  }
}

:root[data-theme="dark"] { /* same overrides again, so the toggle wins both directions */ }
```

Define both themes from day one. Retrofitting dark mode later means touching every component.

---

## The contrast trap

**Bright gold on white fails accessibility for small text.** That is why there are two golds.

- **`gold-bright`** — surfaces only: a 3px top rule, a selected-tab underline, a phase dot, a chart series.
  Never sentences.
- **`gold`** — the darker one, the only one you may set type in.

Getting this wrong produces a page that looks luxurious in a screenshot and is unreadable on a showroom
laptop in daylight. Which is where it will actually be used.

---

## Spend gold structurally, not decoratively

Gold earns its place on:

- a rule above the page title
- section eyebrows and active nav items
- the left border of the *Ready to send* card
- the primary button
- the score band for `very_hot` — and only that band

Everything else is white, near-black and hairline.

**Gold gradients, gold backgrounds behind body text, and gold-on-gold all read as costume jewellery** —
the exact wrong association for this customer. Restraint is the point: a real jeweller's shop is mostly
white light and glass, with the gold in the case.

---

## Semantic colour is separate from the accent

Lead score bands and statuses carry meaning, so they need their own hues that are **not** gold:

| State | Colour |
|---|---|
| Cold | blue-grey |
| Warm | amber |
| Hot / Very hot | warm red |
| Won | green |
| Lost | muted grey |

If gold also means "hot", the owner cannot scan the pipeline board — which is the one screen they will
actually look at every morning.

**Encode state in shape as well as colour** — a pill, a left stripe — so it survives a colourblind reader
and a bad monitor.

---

## Typography

| Role | Face | Notes |
|---|---|---|
| Headings, UI | **Archivo** | A grotesque with character; not the default Inter |
| Long-form reading | **Source Serif 4** | Quotation bodies, AI summaries |
| Numbers, labels | **IBM Plex Mono** | Every figure, every code, every timestamp |

Set `font-variant-numeric: tabular-nums` on **all** money and weight columns. Misaligned rupee figures are
the fastest way to look unserious to a jeweller, and they will notice before they notice anything else.

Keep running text near 65 characters. Give headings `text-wrap: balance`. Uppercase labels get a touch of
letter-spacing.

---

## It is a tool, not a brochure

The dashboard is scanned and operated, not read top to bottom. So the craft is information design, not
decoration:

- **Summary before detail.** The owner's first screen answers "what needs me today", not "here is
  everything".
- **State reads at a glance.** A hot lead should be identifiable without reading a number.
- **What is interactive should look interactive.**
- **Salesperson views are mobile-first.** That is where the actual work happens, standing in a showroom
  with a customer. The owner's dashboard can assume a laptop; the salesperson's cannot.

---

## Screen inventory

Built roughly in this order, matching [07](07-build-order.md).

### Phase 1 — foundation

| Screen | Notes |
|---|---|
| Login | Email + password; phone OTP later |
| App shell | Sidebar, org switcher, theme toggle |
| Products list + detail | Table with tabular figures; image upload |
| CSV import | Column mapping, dry-run preview, error report |
| Gold rate entry | **The ten-second morning task.** Big input, today's rate, history below |
| Pricing rules | One row per rule, showing resolved precedence |
| Quotation builder | Line items, live totals, the itemised GST breakdown |
| Quotation PDF | WeasyPrint; must match the on-screen numbers exactly |

### Phase 2 — CRM

| Screen | Notes |
|---|---|
| Public enquiry form | Standalone, no auth, mobile-first, Turnstile |
| Paste conversation | Big textarea + phone field; the training-data instrument |
| Pipeline board | Columns by lead status; the owner's daily screen |
| Lead detail | Requirements (**editable**), score with reasons, conversation, quotes, follow-ups |
| Conversation view | Message thread, draft composer, takeover toggle |
| *Ready to send* card | Gold left border, copy button, `wa.me` deep link |

### Phase 3–4 — AI

| Screen | Notes |
|---|---|
| Score reasons | The transparent breakdown, rendered from `score_reasons` |
| Matched products | With the deterministic "why it matches" bullets |
| Quote approval gate | Review the draft, edit, approve, then send |
| AI cost card | Per-org spend this month, from `ai_interactions` |

### Phase 5–6 — automation and pilot

| Screen | Notes |
|---|---|
| Follow-up schedule | On the lead: "next follow-up 3 Sep", cancellable |
| Follow-up rules settings | Steps, quiet hours, caps |
| Appointments | Calendar or list |
| Dashboard | The 8 metrics, today's priorities, **automation health strip** |
| Audit log viewer | Filterable; includes "follow-ups not sent, and why" |
| Onboarding wizard | The Phase 6 milestone depends on this being self-serve |

**The automation health strip is not cosmetic.** It shows *"last follow-up sent 2h ago · 6 scheduled
today"*. If the worker dies, the shop owner sees a stale number and tells you — turning a silent failure
into a loud one. See [08](08-capital-hosting-risk.md#7-solo-founder-concentration--high).

---

## The push-not-pull principle

The highest risk in this whole project is that shop owners never log in
([08](08-capital-hosting-risk.md#1-shop-owners-will-not-use-a-dashboard--critical)).

Design the owner's primary interface as **push**: a daily digest, an alert when a lead goes hot. The
dashboard is where they go when the push makes them curious — not the thing they must remember to visit.

Build the digest as soon as there is anything worth digesting.
