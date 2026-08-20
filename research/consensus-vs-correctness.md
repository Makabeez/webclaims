# When GenLayer consensus agrees and is wrong

A controlled study of how source format affects whether validator agreement tracks truth, on Testnet Bradbury.

**Contract under test:** [`0x8f5a035f6CF80190686C395E3B4d7dDB21C2bDae`](https://explorer-bradbury.genlayer.com/address/0x8f5a035f6CF80190686C395E3B4d7dDB21C2bDae) ([source](https://github.com/Makabeez/webclaims))

Run `exp036128`, 9 trials, one contract, one validator set.

---

## Summary

Optimistic Democracy guarantees validators **agree**. It does not guarantee they are **right**. This study isolates one variable that decides whether the two coincide.

Holding the contract, the claims, and the jury constant and changing only the URL:

- **Rendered HTML source** — 2 of 2 settled trials reached unanimous agreement on answers that were **false**.
- **Structured JSON source** — 2 of 2 settled trials reached unanimous agreement on answers that were **correct**.

Nothing on chain distinguishes the two cases. Both report `AGREE`, both write state, both store per-claim reasons that read as considered judgments.

A secondary result: **3 of 9 settlements finalized without writing any state**, in three distinct ways, none of which is documented.

## Method

One deployed contract. Three claims held constant within each arm. Two variables: claim wording (subjective vs objective) and source format (rendered HTML vs JSON API returning the same underlying facts).

| Condition | Claims | Source |
| --- | --- | --- |
| A | subjective — *"the README includes clear install steps"*, *"the project is well documented"*, *"the code is production ready"* | `github.com/Makabeez/webclaims` (HTML) |
| B | objective — *"the page shows a file named X"* for `web_claims.py`, `README.md`, `Cargo.toml` | `github.com/Makabeez/webclaims` (HTML) |
| C | same objective claims as B | `api.github.com/repos/Makabeez/webclaims/contents` (JSON) |

Three repetitions per condition, each with its own `check_id`, all nine in flight simultaneously so no trial could influence another.

The contract runs two sequential non-deterministic blocks: a tolerant `prompt_comparative` around the fetch, then a strict per-claim `prompt_comparative` around the ruling, requiring the boolean for every claim id to match across validators. The outcome is derived in deterministic code from the agreed booleans.

**Ground truth for B and C:** `web_claims.py` present, `README.md` present, `Cargo.toml` absent. A correct jury returns 2 of 3 true.

## Results

| Trial | Condition | Consensus | State written | True count | Correct? |
| --- | --- | --- | --- | --- | --- |
| A1 | subjective / HTML | AGREE | yes | 0 of 3 | n/a |
| A2 | subjective / HTML | AGREE | **no** | — | — |
| A3 | subjective / HTML | AGREE | yes | 0 of 3 | n/a |
| B1 | objective / HTML | AGREE | yes | 0 of 3 | **no** |
| B2 | objective / HTML | **TIMEOUT** | **no** | — | — |
| B3 | objective / HTML | AGREE | yes | 0 of 3 | **no** |
| C1 | objective / API | AGREE | **no** | — | — |
| C2 | objective / API | AGREE | yes | 2 of 3 | **yes** |
| C3 | objective / API | AGREE | yes | 2 of 3 | **yes** |

Aggregated over trials that produced a result:

| Condition | Settled | Correct |
| --- | --- | --- |
| B — objective / HTML | 2 of 3 | **0 of 2** |
| C — objective / JSON | 2 of 3 | **2 of 2** |

Every transaction reached `FINALIZED` with `FINISHED_WITH_RETURN`.

## Finding 1 — the source format decides whether agreement tracks truth

B and C differ in exactly one character sequence: the URL. Same contract, same three claims, same validator set, same equivalence principles.

Both settled B trials returned 0 of 3 true, including for `README.md`, which is unquestionably in the repository. Both settled C trials returned 2 of 3 — correct, including correctly rejecting the absent `Cargo.toml`.

The validators were not malfunctioning and were not disagreeing. They were **correct about the content they were handed**. The first 6000 characters of a rendered GitHub page are `<head>`, navigation, and inline script; the file table never enters the window. Consensus operated exactly as designed on an input that could not support a correct answer.

The failure is invisible on chain. `result: AGREE`, state written, per-claim reasons phrased as judgments ("File not found in page content"). A contract composing on this — releasing escrow, gating a listing, settling a market — receives no signal that anything went wrong.

> Consensus is a property of the validator set. Correctness is a property of the input. The fetch layer decides which one you get, and only the first is reported.

## Finding 2 — a correction to a single-trial result

An earlier single run of condition A produced `DISAGREE`, suggesting subjective claims cannot reach consensus. **Three repetitions do not support that.** A1 and A3 both reached `AGREE` and wrote state; only A2 failed to write.

The original result was noise. Reported here because a single-trial claim about consensus behaviour is not evidence, and anyone with a testnet could have falsified it in twenty minutes.

The open question this leaves: whether subjective wording raises the *rate* of non-settlement rather than guaranteeing it. Distinguishing that needs tens of trials per cell, not three.

## Finding 3 — one third of settlements silently wrote nothing

Three of nine settle calls finalized without storing results, in three distinct ways:

- **B2** — `consensus: TIMEOUT`. Self-explanatory, and at least legible.
- **A2, C1** — `consensus: AGREE`, `exec: FINISHED_WITH_RETURN`, state unchanged. The transaction reports success at every level the SDK exposes, and nothing was written.

This is not a race with the preceding `submit`. All three submits were confirmed `FINALIZED | AGREE` before their settles executed:

```
exp036128-A2 submit: FINALIZED AGREE | settle: AGREE
exp036128-B2 submit: FINALIZED AGREE | settle: TIMEOUT
exp036128-C1 submit: FINALIZED AGREE | settle: AGREE
```

A caller polling transaction status alone would conclude all nine succeeded. Only reading contract state reveals otherwise. At a 33% rate on this workload, any production contract must verify state after settlement rather than trusting the receipt.

## Implications for contract authors

1. **Point at structured data.** APIs, JSON feeds, plain-text files — never a rendered page when a machine-readable equivalent exists. This is the highest-leverage decision in a contract that reads the web, and the SDK surface makes both look equally reasonable.
2. **Verify state after settlement.** `AGREE` plus `FINISHED_WITH_RETURN` does not mean your write landed. Read the contract.
3. **Treat truncation as a correctness risk, not a performance one.** Any character cap decides what the jury can possibly know.
4. **Do not read `AGREE` as a correctness signal.** It reports that validators matched. Nothing more.

## Suggested protocol and tooling improvements

- **Surface fetch truncation to the contract.** If `gl.nondet.web.get` reported whether the response was cut off, a contract could refuse to rule on a partial view rather than ruling confidently on page chrome. This single change would have prevented every wrong answer in condition B.
- **A rendered-text fetch mode with content extraction**, so authors pointing at an HTML page receive content rather than markup.
- **Distinguish "consensus reached, state written" from "consensus reached, nothing applied"** in the transaction result. Findings 3's A2/C1 cases are indistinguishable from success through the SDK.
- **Document source selection** in the non-determinism pages. The equivalence-principle docs explain how validators compare answers, but not how the input determines whether an answer can be right.

## Reproducing

```bash
git clone https://github.com/Makabeez/webclaims
```

The runner script fires all nine trials and prints the result matrix. Every trial is a distinct `check_id` on the contract above; all transaction hashes are recoverable from contract state via `list_checks()`.

Expect 30–45 minutes for settlement when run in parallel.

## Incidental testnet notes

None of the following is documented:

- Settlement takes 30+ minutes per call — two nondet blocks across every validator.
- `waitForTransactionReceipt` is unreliable: it has timed out on transactions that had finalized, and returned early reporting `NOT_VOTED`/`IDLE`. Poll `getTransaction` instead.
- A call can finalize with `NOT_VOTED`, meaning no committee picked it up. Resubmitting the identical call worked.
- `FINISHED_WITH_ERROR` does not reliably indicate lost state — one settle reported it and wrote complete, correct results.
- GenVM rejects `try/except Exception` at schema generation while `genvm-lint` accepts it. The on-chain symptom is a failed deploy with a six-byte CBOR error.
