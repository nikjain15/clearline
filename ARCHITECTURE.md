# Architecture

Clearline is an agent layer for a private-markets fund administrator: four narrow agents read the fund's bank statement overnight, book what the controller's policy allows, and hand every other line to a named person with the evidence, a proposed entry and one decision. This document is the overview; the live pages carry the pictures: [How it works](https://nikjain15.github.io/clearline/how-it-works.html) and [Architecture](https://nikjain15.github.io/clearline/architecture.html).

## Layers

```
  The human gate      fund accountant records or declines with a reason; controller sets policy and signs; GP approves refunds; two signatories release
        ^
  Agents              Intake (rules), Matching (model + rules), Reconciliation (rules), Posting (policy), Ask (phrases from released data)
        ^
  Engines             call arithmetic, bank tie-out, duplicate and tolerance checks, chart-of-accounts mapping, share allocation, statement builder, the audit
        ^
  Data                investors, call notice, bank lines, fund ledger, fund agreement chunks, policy with version, an append-only decision log
        ^
  Sources (read only) bank camt.053/054 over SWIFT, general ledger API, transfer-agency register, notice inbox attachments, uploaded templates
```

Every arrow points up. One arrow points down, and it is the only write path: a draft journal, released by a person, into the ledger's own validation and approval. No agent holds a payment, release or send capability.

## Principles

1. **Start from the material, not the agent.** Each case is a bank statement and a call notice first. The agents appear only in what they did to that material.
2. **One dataset, every figure derived.** Investors, call notice, bank lines and ledger are generated once from a seed in `index.html`; every count, total, statement row and client answer is computed from them. The Ingest screen's audit fold recomputes eight reconciliations on every render and offers the CSVs.
3. **Keep the engines deterministic; put the model at the edges.** Gates, tolerances, arithmetic and the posting path are code. A model may score a match, read free text into fields, and phrase a trace or an answer. It never posts, releases, sends or changes policy. Each agent card says which it is.
4. **The policy is a person's.** Gate, tolerance and the never-automatic list carry an owner, a date and a version. A new fund starts at a gate of 1.00. Agents propose changes; only the controller applies them, and the changelog records it.
5. **Never send, never pay.** Refunds need the GP, a call-back to the number on file and two signatories. The demo drafts the script and stops.
6. **Read the screen before trusting the test.** Every release is walked headlessly at 1280 and 400 pixels with every button pressed and every derived figure dumped and checked against the dataset.

## Where things live

Everything is one file, `index.html`, in this order:

| Section | Holds |
|---|---|
| `<style>` | Relay tokens verbatim, then components: documents, tables, the three-column case, the trace, folds, the feedback panel, the learning screen |
| Dataset | `FUND`, `NAMED`, the seeded generator for `INV` (180 investors) and `LINES` (183 bank lines), `EXC` (which lines become cases), `D` (derived totals), `LEDGER`, `stmtRows()`, `DATASET` (CSV downloads) |
| Material | `CASES`: four cases built from the dataset, each with the statement excerpt, the records, the proposed entry, four agent traces, the figure and the decision |
| Setup | `CONN`, `UP`, template generation, in-browser CSV checking, `connect()`, `ingRows()`, `dataset()`, `audit()`, `ingest()` |
| Feedback and learning | `FB`, `FBTO`, `fbPanel()`, `fbDone()`, `learn()` |
| Screens | `intro()`, `caseView()` with `stmtTable`, `recCard`, `entryBox`, `fig`, `beats`, `pipe`, `trace`; `statement()`; `client()` with `QA` |
| Router | `go()`, `render()`, `wire()`, history and keyboard handling |

The about pages (`how-it-works.html`, `architecture.html`) share `about.css`.

## Ingestion

| Source | Parse | Validate | Index |
|---|---|---|---|
| Bank statement | lines to amount, value date, counterparty, reference, UETR | opening plus movements equals closing; duplicate wire IDs flagged, not dropped | reference, exact amount, payer name |
| Call notice | CSV against template columns | sum of calls equals the call percentage of commitments; unique IDs; joined to the register | investor ID, amount |
| Register | investors, commitments, instructions, KYC | near-duplicate names flagged; instructions verified by call-back | ID, name tokens |
| Invoice, offset schedule | fields with a confidence | basis cross-checked to the fund agreement; version diffed | period, payee |
| Fund agreement | chunks at clause boundaries, about 400 tokens, 60-token overlap, page and clause on each | operational clauses tagged; re-indexed only on a new version | keyword and embedding; cited by chunk ID |

## The learning loop

Decide and explain (a decline needs a reason) → store as labelled examples in the tenant → evaluate weekly against a held-out set (precision above the gate, override rate, reversal rate) → propose a rule, tolerance or retrain with its evidence → the controller approves or rejects each line, the policy version bumps, the changelog is signed. The model retrains at most once a quarter and ships as a frozen version with its evaluation report. A rising override rate freezes the gate.

## Prototype versus production

| In the demo | In production |
|---|---|
| Dataset generated in the page from a seed | camt.053 feed, ledger and register APIs, inbox attachments; the same parsers against real files |
| Confidences fixed per case | Matching score from a calibrated model trained on the tenant's own history; the gate unchanged |
| Draft journal shown on screen | Draft journal posted through the ledger API; the ledger validates and approves |
| Feedback kept in page state | Append-only decision log; weekly evaluation; quarterly frozen model versions |
| One fund | Policy per fund, one model per administrator; exceptions queued and routed |
