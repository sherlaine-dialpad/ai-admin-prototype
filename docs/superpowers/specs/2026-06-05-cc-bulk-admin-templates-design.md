# AI Admin Agent — Design Proposal

**Date:** 2026-06-05 · Updated 2026-06-08
**Author:** Sherlaine Lau
**Status:** Updated — incorporating Abhishek feedback
**Artifacts:** [presentation.html](../../presentation.html) · [api-inventory.html](../../api-inventory.html) · [qbr.html](../../qbr.html)

---

## Overview

Enterprise Dialpad customers managing 50–500+ contact centers have no scalable way to enforce consistent settings across their CC fleet. Today every CC is configured individually. One policy change (e.g. "enable recording for all NA CCs") requires touching each CC one by one.

The goal is an **AI Admin Agent** that understands natural language admin requests, determines which settings need to change, and executes bulk updates — with a human in the loop at every step. Templates are the Phase 1 foundation that makes this possible.

## Direction Update (2026-06-08) — Abhishek Feedback

Key pivots from EM review:

1. **Separate agents, not one** — Admin Agent (settings/config/bulk) and Analytics Agent (reporting/insights) must remain separate. Smaller domain = better accuracy, consistent with industry direction. The Admin Agent may surface analytics context but does not own it.

2. **Templates are Phase 1, not the end state** — Templates are validated by customer feedback and are the right foundation, but the long-term goal is a fully agentic admin assistant that can handle natural language requests, exceptions, and multi-step workflows beyond what templates cover.

3. **Align with new Global Admin IA** — Core UX is redesigning Admin with global search, URL-addressable settings, and a navigation metadata model. The Admin Agent should live inside this new experience and eventually consume the same navigation registry (agent knows where settings live without manual mappings).

4. **UI positioning** — Admin Agent lives in the new Global Admin experience, not as a side panel on existing pages.

5. **API inventory** — Continue and formalize as a matrix: Setting Area | API Exists | Writable | Bulk Supported | Notes. See [api-inventory.html](../../api-inventory.html).

### 3-Phase MVP

| Phase | Quarter | Scope |
|---|---|---|
| **Phase 1** | Q3 | Template CRUD, bulk apply, drift detection, AI analytics chat (read-only) |
| **Phase 2** | Q4 (pending API gaps + Core UX IA) | Natural language admin actions, AI-driven bulk changes, settings discovery via agent |
| **Phase 3** | Q5+ | Fully agentic admin, multi-step workflows, cross-setting orchestration, recommendations |

---

## Target User

**Large enterprise admins** managing their own Dialpad contact centers — e.g. a head of IT or CC operations at a company with 200 CCs across regions. They are responsible for compliance, consistency, and operational efficiency across the fleet. They are not Dialpad employees; they are Dialpad customers using the admin portal.

---

## Approach: Template-First (Option A)

Three options were evaluated:

| Option | Description | Decision |
|---|---|---|
| **A — Template-first** | Master template → child CCs, AI agent + analytics as accelerators on top | ✅ Selected for Q3 |
| B — AI-first | Chat is the primary interaction model, templates are saved prompts | Future vision (Q4/Q5) |
| C — Phased, settings dashboard first | No AI in Q3, AI deferred to later phases | Rejected — loses the narrative |

**Rationale for A:** Provides the clearest mental model for admins (familiar from RingCentral). AI agent and analytics chat layer in naturally without being the core infrastructure. Honest about Q3 scope while preserving the long-term AI vision.

---

## Section 1 — The Template Model

### What a template is

A **template** is a named configuration profile that defines a set of CC settings. A CC can be assigned to one template. Settings in the template flow down to assigned CCs.

Each setting in a template is either:

- **🔒 Locked** — the template owns this setting. Assigned CCs cannot change it locally. Any deviation surfaces as a "drift" warning.
- **✎ Adjustable** — the template sets the default value, but a local CC admin can override it for their specific CC. No drift is flagged for adjustable settings.

### Template fields (Q3 scope)

| Setting | Type | API status |
|---|---|---|
| Call recording | 🔒 Locked | ⚠️ Needs verification before Q3 commit — user-level toggle is public; if CC-level policy field exists in current API it's Q3, otherwise it moves to Q4 |
| Business hours (Mon–Sun) | 🔒 Locked | ✅ Supported via CC create/update |
| Ring seconds | ✎ Adjustable | ✅ Supported |
| Call routing mode | ✎ Adjustable | ✅ Supported (inside `routing_options`) |
| Overflow / hold queue | ✎ Adjustable | ✅ Supported (inside `hold_queue`) |
| Blocked numbers (company-wide) | 🔒 Locked | ✅ Supported |
| AI summaries / transcription (basic on/off) | ✎ Adjustable | ⚠️ `voice_intelligence` object exists in public API but sub-fields undocumented — best-effort Q3; deeper AI config (coaching, PII redaction, real-time assist) is Q4 |

### Template fields (Q4 — pending new APIs)

| Setting | Blocker |
|---|---|
| Holiday schedule | No public API — `/api/observedholiday` is internal only |
| CSAT survey configuration | No public API |
| AI settings (coaching, PII redaction) | `voice_intelligence` sub-fields undocumented |
| CRM integrations | No public API |
| Supervisor role assignment | Not documented in public API |
| CC-level recording policy | Needs new dedicated endpoint |

### Inheritance and drift detection

```
[Template: Enterprise Sales — NA]
        │
        ├── Sales East       ✓ In sync
        ├── Sales West       ✓ In sync
        ├── Sales Central    ⚠ Drifted (recording off — Locked setting changed locally)
        └── LATAM Support    — No template assigned
```

**Drift** = a CC whose local value for a Locked setting differs from the template value. Drifted CCs surface in the CC grid with a warning badge and a "Fix" action that re-applies the template.

---

## Section 2 — Bulk Apply Flow

Three steps. Human stays in the loop at every step — nothing applies without an explicit confirmation.

### Step 1: Select contact centers

From the CC grid, the admin selects one or more CCs using checkboxes. A bulk action bar appears at the top of the grid with an "Apply template" CTA. The grid supports filtering by region, current template assignment, and issue status to make targeting fast.

### Step 2: Preview changes

Before anything is applied, the admin sees a per-CC diff:

- **Will change** (yellow) — the setting differs from the template; it will be updated
- **Already compliant** (green) — the CC already matches the template; no change needed
- **Skipped** (orange) — the setting cannot be applied because the API doesn't support it yet; the rest of the template still applies

The preview shows the exact before/after value for each changed setting, with Locked/Adjustable labels visible.

### Step 3: Confirm and apply

One button applies the template to all selected CCs. Results appear immediately:

- Per-CC summary: how many settings changed, how many were already compliant, how many were skipped
- **Undo** button available immediately after apply
- Change is written to the activity log (what changed, when, who triggered it)

### Skipped settings — the honest API gap UX

When a setting can't be applied (API gap), it is shown clearly in the preview as "Skipped — will be applied when supported" rather than silently ignored or blocking the whole operation. This keeps the product useful today while making the gap visible to admins and the product team.

---

## Section 3 — AI Layer

The AI layer has two surfaces, sequenced by risk and dependency:

### AI analytics chat (Q3 — read-only)

Reuses the same chat pattern as Dialpad AI Analytics today, but pointed at admin settings data instead of call data.

**What it can answer:**
- "Which contact centers don't have a template assigned?"
- "Which CCs have recording off?"
- "Which CCs have drifted from their template?"
- "Show me all NA CCs without holiday hours set"

Results are returned as a list of CCs with relevant details. The list includes a direct CTA to act on the results — e.g. "Apply a template →" — which launches the bulk apply flow (Section 2) pre-filtered to those CCs.

**Why Q3:** Read-only, no write risk, reuses an existing Dialpad AI pattern and data infrastructure. Low engineering risk.

### AI agent (Q4 — write actions via natural language)

Natural language input triggers the same template apply flow. The AI interprets the command, identifies the target CCs, selects the appropriate template, and presents the standard preview/confirm screen. The admin confirms before anything is applied.

**Example:**
> "Apply the Enterprise Sales template to all NA contact centers that don't have recording on"

→ AI finds 4 matching CCs → shows preview (same as Section 2 Step 2) → admin clicks "Confirm & apply"

**Why Q4:** Depends on the bulk apply engine being stable and tested in Q3, and ideally on the batch API endpoint being available for reliable scale.

### Future vision (Option B — Q4/Q5)

One unified conversation surface where the admin can ask questions, get insights, and take actions without switching between the analytics chat and the settings UI. Templates become the "memory" the AI uses to understand intent. Scheduling ("apply holiday hours every December"), auditing ("who changed Sales East on April 3?"), and proactive recommendations ("3 CCs are out of policy — want to fix them?") all live in one thread.

---

## Section 4 — Engineering Delta

### Settings coverage in Q3 templates

- **62%** of CC settings — fully supported by existing public API → can build now
- **14%** — partial coverage (fields exist but undocumented) → some Q3 risk
- **24%** — no public API → Q4 dependency

### Frontend work

| Item | Effort |
|---|---|
| Template library page (list, search, create) | M |
| Template editor (settings configuration + Locked/Adjustable toggles) | L |
| CC grid — template assignment column + drift badge | S |
| Bulk select + apply flow (bulk bar, flow entry point) | M |
| Preview / diff view | M |
| Results screen + undo | S |
| Activity log | S |
| AI analytics chat surface | M |

### Backend work

| Item | Effort |
|---|---|
| Template data model + storage | M |
| Template → CC assignment | S |
| Drift detection engine (compare CC state vs template per setting) | M |
| Bulk apply orchestration (apply template to N CCs, handle partial failures) | L |
| Settings query API for AI analytics | M |
| Activity log service | S |

### New public API endpoints needed (Q4 dependency)

| Endpoint | Why it matters | Priority |
|---|---|---|
| `GET/POST/PATCH/DELETE /api/v2/callcenters/{id}/holidays` | Holiday schedule is the #1 setting enterprise admins want to bulk-manage. Currently internal-only (`/api/observedholiday`). | High |
| `PATCH /api/v2/callcenters/{id}/recording` | Compliance teams need CC-level "always record" policy. User-level toggle exists; CC-level policy does not. | High |
| Document `voice_intelligence` sub-fields | The object already exists in the public API but sub-fields are undocumented. Lowest-effort gap to close — schema exposure + docs. | High |
| `GET/PATCH /api/v2/callcenters/{id}/csat` | Enterprises want consistent CSAT surveys across CCs. Zero public API coverage today. | Medium |
| `POST /api/v2/callcenters/batch` | Without a batch endpoint, applying a template to 200 CCs requires 200 sequential API calls. Needed for AI agent scale and reliability. | Medium |

---

## Open Questions (from Abhishek feedback)

1. **Core UX navigation schema** — what metadata/schema will Core UX expose? Can the Admin Agent consume the same registry? When is it available?

2. **API gap ownership** — who owns the 5 new public API endpoints (holiday schedule, CC recording policy, AI/voice intelligence fields, CSAT, batch)? Determines Q4 start date.

3. **First end-to-end agent workflow** — what is the first Phase 2 agentic workflow to prototype? Suggested: "Apply EMEA compliance settings to all European CCs."

4. **Template versioning and inheritance** — how does versioning work? How are templates applied during new CC creation?

5. **Global Admin IA timeline** — when does the new Global Admin experience launch? Phase 2 Admin Agent UI positioning depends on this.

---

## API Coverage Reference (from codebase + public API audit)

For full detail, see the contact center API audit conducted prior to this spec. Summary:

### What the public Dialpad Developer API (v2) supports today

- CC CRUD (create, read, update, delete)
- Basic business hours (per weekday open/close via CC create/update)
- Operator management (add, remove, list agents)
- Skill levels (get, update per operator)
- Duty status (get, update)
- Number assignment (assign, unassign, auto-assign, swap)
- Call routing — webhook-based dynamic routing (Call Router API)
- Custom IVR (create, update, delete, assign)
- Dispositions (create, read, update, delete, list)
- Blocked numbers (company-wide)
- Caller ID (get, set — CC scope unclear)
- Recording sharelinks + transcripts (access, not configuration)
- Stats API + scheduled reports
- Agent status types (custom CRUD)
- Access control policies
- Coaching teams
- Webhooks / event subscriptions

### What is internal-only today (no public API)

- Holiday hours / observed holidays (`/api/observedholiday`)
- Holiday routing rules
- CSAT survey configuration
- AI settings (coaching, PII redaction, real-time assist)
- CRM integrations (`/api/settings/integrations`)
- Data retention policy
- CC-level automatic recording policy
- Forwarding verification (`/api/verifyforwarding`)
- Auto-answer settings
- Supervisor/admin role assignment
- Agent wrap-up time / after-call work settings
- Per-CC spam prevention thresholds
- Disposition list management and per-CC assignment

### What exists but is opaque (fields undocumented)

- `advanced_settings` object on CC create/update
- `routing_options` object on CC create/update
- `hold_queue` object on CC create/update
- `alerts` object on CC create/update
- `voice_intelligence` object on CC create/update
