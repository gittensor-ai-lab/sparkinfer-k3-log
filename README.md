# sparkinfer-k3-log

Immutable, verifiable evaluation log for
[sparkinfer-k3](https://github.com/gittensor-ai-lab/sparkinfer-k3) — Kimi K3 (2.8T, 896
experts) on one 8x H200 node.

One directory per eval run. Each carries the verdict, the raw harness output, and a
**Polaris receipt** binding that verdict to its provenance — code commit, scoring-scripts
commit, model SHA256, build hash, llama.cpp reference commit, GPU and driver.

## Why this repo exists separately

The verdict decides emissions. If it lives only in a pull-request comment it is editable,
deletable, and unattributable after the fact. Committing it here makes the record
append-only and independently checkable, and keeps it out of the repo whose scores it
judges — a log that the scored repo can rewrite is not a log.

## Layout

```
runs/<run_id>/
  result.json        the RESULT_JSON the harness emitted (verdict + measurements)
  attestation.json   canonical provenance: code, references, environment, verdict
  receipt.json       Polaris receipt over that attestation (TDX quote, or Ed25519)
  eval.log           raw harness output
index.json           newest-first summary, for the page
ledger.jsonl         append-only, one line per run
```

## Verifying a run without trusting anyone

A receipt is checkable offline. It carries the Intel DCAP quote and collateral, so
verification needs neither Polaris nor us:

```bash
git clone https://github.com/gittensor-ai-lab/sparkinfer-k3.git
cd sparkinfer-k3 && python3 - <<'PY'
import json, sys; sys.path.insert(0, "eval")
from polaris.verify import verify_strict
from polaris.receipt import ReceiptValidator
r = json.load(open("/path/to/runs/<run_id>/receipt.json"))
problems = verify_strict(r, ReceiptValidator(r))
print(problems or "receipt verifies")
PY
```

`verify_strict` is deliberately strict: it rejects a receipt with unpinned clocks, a
missing build hash, or an absent llama.cpp commit. A run that fails those is not
fraudulent — it is under-provenanced, which is a different and recoverable problem. The
verdict text says which.

## What is and is not attested

The benchmark runs **bare metal, not inside the enclave**. Running a GPU benchmark under
CC/TDX degrades the very performance being measured, so an in-enclave number would be a
hardware-signed receipt of a CC-degraded result. What the enclave attests is the
**scoring**: that these inputs, through this scoring code, yield this verdict. The link
back to the bare-metal run is the provenance binding — model SHA256, build hash, commit —
not the enclave. Read a receipt as "this verdict follows from these inputs and this
scorer", not as "this hardware witnessed this benchmark".

## Scoring

Tiers come from
[`bench/scripts/label.py`](https://github.com/gittensor-ai-lab/sparkinfer-k3/blob/main/bench/scripts/label.py),
a deterministic function of the measurements, so independent validators converge on the
same verdict. Correctness gates first: top-1 agreement with llama.cpp >= 0.90 and mean KL
<= 0.20, else REJECT regardless of speed.
