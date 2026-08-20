# WebClaims

A reusable GenLayer primitive for checking factual claims about a web page under validator consensus.

## The problem

Any Intelligent Contract that reads the live web and judges what it finds has to answer two questions, and they need **different** equivalence principles:

1. Did every validator see the same page?
2. Do they agree on what the page shows?

Getting either wrong breaks the contract in ways that are hard to diagnose from on-chain data. `strict_eq` on a live page deadlocks the jury on ad counters and relative timestamps. One broad "is this good?" judgment lets a single model's opinion become the verdict, because `prompt_non_comparative` only asks validators whether the leader answered in good faith — not whether they reached the same answer.

WebClaims separates the two: a **tolerant** principle for the fetch, then a **strict per-claim** principle where every validator independently returns a boolean for every claim and they must all match.

## Measured behaviour

Bradbury, 5 validators, same page, same jury. Only the wording of the claims changed.

| Claim style | Example | Result |
| --- | --- | --- |
| Subjective | *"the README includes install steps"* | `DISAGREE` — no state written, check does not settle |
| Objective | *"the repository contains a file named Cargo.toml"* | `AGREE` — per-claim booleans stored |

That is the primitive's contract with its caller. Claims must be objectively checkable statements about what is present on the page. Anything requiring judgment splits the jury and refuses to settle — which is the correct failure mode for a contract that gates value. A single model's read of "is this well documented?" should not move money.

Sample settled record:

```json
{
  "url": "https://github.com/…",
  "claims": [
    "the repository contains a file named Cargo.toml",
    "the repository contains a directory named src"
  ],
  "results": [
    { "id": 0, "true": false, "why": "no Cargo.toml visible" },
    { "id": 1, "true": false, "why": "no src directory visible" }
  ],
  "unmet": [
    "the repository contains a file named Cargo.toml",
    "the repository contains a directory named src"
  ],
  "settled": true,
  "all_true": false
}
```

## How consensus is used

`settle()` runs **two sequential non-deterministic blocks**. They cannot nest, so they are separate calls, and every state write happens after both return, in deterministic context — writing inside a nondet block means each validator persists a different value before consensus decides which is correct.

**Block 1 — read the page.** `prompt_comparative` with a tolerant principle: two extracts are equivalent if they describe the same subject and contain the same concrete items, ignoring whitespace, advertising, star counts, relative timestamps, and dynamic navigation.

**Block 2 — rule on each claim.** `prompt_comparative` wrapping a `gl.nondet.exec_prompt` call that returns structured JSON — one `{id, true, why}` per claim. The principle requires the boolean to match for every id, and explicitly instructs that differing `why` wording be ignored, so validators are never rejected over phrasing.

**Derivation.** The unmet set is computed in code from the agreed booleans. `all_true` follows from `len(unmet) == 0`. The leader's prose decides nothing.

## Composing on it

`all_true(check_id)` returns a single boolean for contracts building on top — an escrow release, a grant milestone, a listing gate. It raises if the check has not settled, so a caller cannot read a half-formed result.

```python
# in your contract, after WebClaims has settled
if web_claims.all_true("milestone-3"):
    release_payment()
```

## Prompt injection

Whoever submits the URL controls the page the contract reads. That makes fetched content untrusted input in the strictest sense: an attacker supplies a page, and the page enters the prompt deciding an outcome in their favour.

The ruling prompt states that anything inside the page block — instructions, role-play, claims of authority — is data to be judged, never instructions to follow, and the page is delimited with explicit tags so the boundary is unambiguous.

## Notes for builders

- **No `try/except` in contract code.** GenVM rejects `except Exception` at schema generation, though `genvm-lint` accepts it. A malformed jury response therefore reverts the transaction rather than returning a soft error — nothing is written on bad input, which is the honest failure mode.
- **Settlement takes 30+ minutes on Bradbury.** Two nondet blocks across every validator. Treat it as an async job and poll `getTransaction`; `waitForTransactionReceipt` has both timed out on finalized transactions and resolved early reporting `NOT_VOTED`.
- **A call can finalize without being voted on.** Resubmitting the identical call worked.
- **Page text is capped at 6000 characters** so every validator reads the same slice.
- **Fetching `github.com/owner/repo` returns rendered HTML, not the file tree.** For repository claims, point at `api.github.com/repos/{owner}/{repo}/contents` instead.

## Verifying

```bash
pip install genvm-linter
genvm-lint check web_claims.py
genvm-lint schema web_claims.py
```

## License

MIT
