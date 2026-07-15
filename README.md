# cloud-itonami-isic-1623: Manufacture of wooden containers

Open Business Blueprint for **ISIC Rev.5 1623**: manufacture of wooden containers — an autonomous "actor" (LLM advisor behind an independent Governor, langgraph-clj StateGraph, append-only audit ledger) that coordinates back-office wooden-container-shop **plant operations**: production-batch data logging (dimensional-spec/unit-count/output-quality for crates, pallets, and barrels/casks), cutting/assembly-line-equipment maintenance scheduling, safety-concern flagging (including ISPM-15 heat-treatment-compliance), and outbound wooden-container shipment coordination.

This repository designs a forkable OSS business for wooden-container-
shop plant operations: run by a qualified operator so a wooden-
container shop (crates, pallets, barrels/casks manufactured via
cutting/assembly lines and cooperage) keeps its own operating records
instead of renting a closed SaaS.

## What this actor does

Proposes **plant operations coordination**, not equipment operation:
- `:log-production-batch` — crate/pallet/barrel batch dimensional-spec/output-quality data logging (administrative, not an operational decision)
- `:schedule-maintenance` — cutting/assembly-line-equipment maintenance scheduling proposal
- `:flag-safety-concern` — surface a materials-safety/equipment-safety/ISPM-15 heat-treatment-compliance concern (always escalates)
- `:coordinate-shipment` — outbound wooden-container shipment coordination proposal

## What this actor does NOT do

**CRITICAL SCOPE BOUNDARY — this is a safety-critical domain**
(nailing-machine and stave-jointer blade/cutter injury risk, wood-dust
fire/explosion hazard, and ISPM-15 heat-treatment noncompliance risk
for internationally shipped wooden packaging):

- Does NOT control cutting or assembly-line equipment directly
- Does NOT make plant-safety or hazard decisions (that's the plant supervisor's exclusive human authority)
- Does NOT authorize or finalize a cutting/assembly-line run (human plant supervisor decides)
- ONLY proposes/coordinates operations back-office; all actuation requires explicit human approval
- Safety-concern flagging ALWAYS escalates — never auto-decided, no confidence threshold or phase below escalation

## Architecture

Classic governed-actor pattern (`woodcontainer.operation/build`, a langgraph-clj StateGraph):
1. **`woodcontainer.advisor`** (sealed intelligence node, `WoodenContainerAdvisor`): proposes decisions only, never commits
2. **`woodcontainer.governor`** (independent, `Wooden Container Shop Plant Operations Governor`): validates against domain rules, re-derived from `woodcontainer.registry`'s pure functions and `woodcontainer.store`'s SSoT -- never trusts the advisor's own self-report
   - HARD invariants (always `:hold`, no override):
     - Shop/batch record must be independently verified/registered (`:verified?` AND `:registered?`) before any action is taken against it (equipment before maintenance scheduling, batch before shipment coordination)
     - The request's own `:effect` must be `:propose` (never a direct-write bypass)
     - `:op` must be in the closed four-op allowlist
     - The proposal's own `:effect` must be one of the four propose-shaped effects (no direct cutting/assembly-line-equipment control)
     - Finalizing a cutting/assembly-line run (`:finalize? true`) is a PERMANENT, unconditional block
     - A shipment may not push a batch's own recorded shipped unit count past its own logged production unit count (independently recomputed)
     - No double-scheduling the same maintenance record
     - No fabricated `:dimensional-spec` value on a production-batch patch
     - No physically implausible `:output-quality-percent` value on a production-batch patch
   - ESCALATE (always human sign-off, overridable by a human):
     - `:flag-safety-concern` always escalates, regardless of confidence
     - Low-confidence proposals
3. **`woodcontainer.phase`** (Phase 0->3 rollout): `:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` are NEVER in any phase's `:auto` set (permanent, matching the governor's own posture); only `:log-production-batch` may auto-commit at phase 3 when clean
4. **`woodcontainer.store`** (append-only audit ledger + SSoT): a single `MemStore` backend behind a `Store` protocol (see ns docstring for why a second Datomic-backed backend is out of scope for this build)

## Development

```bash
# Run tests (top-level deps.edn already pins langgraph+langchain local/root)
clojure -M:test

# Run tests via the workspace :dev override alias (equivalent, kept for sibling-repo parity)
clojure -M:dev:test

# Run the demo
clojure -M:dev:run

# Lint
clojure -M:lint
```

## Status

`:implemented` — `governor.cljc`/`store.cljc`/`advisor.cljc`/`registry.cljc` + `deps.edn` complete the module set; tests green, demo runnable, langgraph-clj integration verified.

## License

AGPL-3.0-or-later
