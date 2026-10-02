# Hybreed

**Integration intelligence engine for the ELCI Group.** Rust-based,
Git-native semantic integration control plane, companion to Kaptaind.

> Git determines what *can* be integrated. Hybreed determines what
> *should* be integrated, how, and whether the resulting state is acceptable.

## Status: Phase III — Candidate synthesis

Phases I–II (deterministic + semantic integration intelligence) deliver:
git-plane comparison, the Integration Graph, four-class conflict taxonomy,
risk scoring, strongly-typed decisions, tree-sitter symbol analysis,
intent comparison, supersession/duplication detection, and constrained
model inference behind a provider-neutral interface.

Phase III adds (this release) — Hybreed can now answer *"here are the
possible integrated states; this one is the safest"*:

- **Worktree isolation** (`hybreed-synthesis`, §19) — every candidate
  gets its own worktree and `hybreed/*` candidate branch under
  `.hybreed/worktrees/`. The primary working tree is never touched.
  Failed candidates are disposable (worktree + branch removed, evidence
  kept); successful candidates are retained as promotable artefacts.
- **Candidate generation** (§27) — strategies justified by the decision:
  `merge`, `squash`, `rebase_merge` (cherry-pick replay), `take_a`/`take_b`
  (supersession). Conflict/review decisions authorise *no* synthesis —
  fail closed, nothing is created.
- **Automated validation** (`hybreed-validation`, §26) — a
  repository-configurable ladder in `.hybreed/validation.toml`; steps run
  sequentially inside the candidate worktree with fixed argv, no shell,
  per-step timeout, capped output. Opt-in only: no implicit builds.
  Unvalidated candidates are never promotable.
- **Adversarial verification** (§13) — the candidate is actively compared
  against base and both sources: lost symbols (public API loss is an
  error), silent side-overrides, change reversion, removal-vs-change
  clashes. Unanswerable questions (weaker tests, migration ordering,
  security regression) are recorded honestly and cap validation
  confidence at 0.9.
- **Candidate ranking** (§27) — score = validation (50) + adversarial
  (30, minus findings) + decision-aware strategy preference + risk
  bonus. The **best validated state** is chosen, not the first success.
- **Transactional shape** (§20) — prepare → integrate → validate →
  adversarial → rank; failures roll back the worktree but preserve
  evidence under `.hybreed/candidates/` + `.hybreed/staging.json`.
- **Separate confidences** (§13) — integration confidence (analysis) and
  validation confidence (post-synthesis) are reported independently; a
  high-confidence interpretation with failed validation is still a failure.
- **Decision upgrade** (§23) — file overlap with symbol-disjoint changes
  and no cross-side references is now a deterministic `MERGE`
  (complementary), no inference required.

Not yet (Phases IV–V): policy engine, confidence-gated autonomous
execution, promotion/rollback of chosen candidates, signed audit trails,
GitHub PR integration, integration learning.

## Layout

```
crates/hybreed-ast        # tree-sitter symbol extraction
crates/hybreed-model      # constrained inference provider interface
crates/hybreed-validation # validation ladder runner
crates/hybreed-synthesis  # worktree-isolated candidate synthesis
crates/hybreed-core       # git + semantic planes, graph, risk, decisions, staging
crates/hybreed-cli        # `hybreed` binary
```

## Usage

```bash
cargo build --release

hybreed analyse feature/a feature/b     # deterministic comparison
hybreed compare  feature/a feature/b
hybreed semantic feature/a feature/b    # symbols, cross-refs, intent
hybreed graph     feature/a feature/b   # integration graph
hybreed decide    feature/a feature/b   # explicit Integration Decision
hybreed plan      feature/a feature/b   # dry-run plan + manifest
hybreed explain   feature/a feature/b   # narrative report

# STAGE mode: candidates in isolated worktrees, validated and ranked
hybreed stage feature/a feature/b [--keep-failed] [--max-candidates N]
hybreed candidates [--prune]

# JSON contract for Kaptaind / CI / dashboards
hybreed decide --format json feature/a feature/b
hybreed stage  --format json feature/a feature/b

# Constrained reasoning provider (default: offline, zero inference)
hybreed decide --provider command:/path/to/provider feature/a feature/b
```

Validation ladder configuration (`.hybreed/validation.toml`):

```toml
steps = ["cargo fmt --check", "cargo build --quiet", "cargo test --quiet"]
timeout_secs = 600
```

Steps run inside each candidate worktree with `HYBREED_CANDIDATE=1`.
Without this file, candidates are staged but marked `unvalidated` and
are never promotable — Hybreed does not guess at correctness.

The provider protocol (`hybreed.semantic_compare/1`): a JSON
`JudgmentRequest` on stdin, a JSON `Judgment` on stdout
(`compatible`, `intent_conflict`, `confidence`, `rationale`). Providers
run with a fixed argv, no shell, 30s timeout — LLM output is evidence,
never authority (§12).

## Testing

The golden integration corpus lives in `crates/hybreed-core/tests/`:
Phase I (clean, mechanical, duplication, dependency), Phase II (ordering,
structural, supersession, semantic divergence, opposing intents, provider
escalation with a mock script), Phase III (staging with ranked candidates
including a real `cargo build`/`cargo test` ladder, conflict → no
synthesis, sequence → rebase candidate, supersession → take candidate,
failed-validation disposal with preserved evidence, unvalidated
non-promotability), plus adversarial-verification units. Every production
regression becomes a new fixture.

```bash
cargo test
```

Model evaluations stay separate from deterministic tests (§38); the
provider test uses a fixed script, never a live model.

## Design principles

- Never equate textual merge success with integration success.
- Deterministic evidence wins over generated reasoning.
- Generative inference is invoked only where it is materially useful,
  and can only escalate, never validate.
- Fail closed: when uncertain, do not merge.
- Optimise for validated repository improvement per unit of risk —
  a human escalation is not a failure; a wrong autonomous merge is.
# hybreed
