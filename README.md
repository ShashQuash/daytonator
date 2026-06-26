# Detonator — pre-event scaffold

Fork-and-bait differential detonation for the AI software supply chain.
Snapshot a clean Daytona sandbox once, fork it into several *baited* environments,
and let the model judge the cross-fork behavioural differential. Malware that hides
on a clean run reveals itself in the fork that baits it.

This scaffold **already runs end to end offline** (no Daytona, no API key) and the
three demo packages classify correctly. Your job on the day is to swap the local
backend for Daytona forking and build the UI on top.

## Floor and ceiling
- **Floor (15:30 freeze target):** one package → control + AWS fork → strace capture
  → verdict. That alone is a complete, winning demo.
- **Ceiling (only on top of a working floor):** all five baits, AI-written
  provocation harnesses, and iterative re-forking when inconclusive.
Never sacrifice the floor for the ceiling.

## Setup
```bash
pip install -r requirements.txt          # daytona-sdk, anthropic
cp .env.example .env                      # fill in keys
# strace must be present locally for the LocalRunner: apt-get install -y strace
```

## Run it offline (do this tonight)
```bash
python tests/smoke_test.py                # all three demo packages should PASS
python -m detonator.cli --local demo_packages/evasive_stealer \
    --import-name evasive_stealer --full  # watch the star: silent on control, fires on AWS
```
With no `ANTHROPIC_API_KEY`, verdicts come from a transparent heuristic. Add the
key and the model writes the verdict + explanation instead (`via: ai`).

## Run it on Daytona (the day)
```bash
python -m detonator.cli requests --backend daytona            # real PyPI package
python -m detonator.cli evasive-pkg --backend daytona --full  # your demo
```
`DaytonaRunner` is written against the current `daytona-sdk`: `snapshot.create`,
`create(CreateSandboxFromSnapshotParams(snapshot=..., env_vars=...))`,
`process.exec`, `fs`. Three calls are marked `CONFIRM-ON-DAY` in `runner.py`
(file upload arg order, home-dir getter, network allow-list) — verify them against
live docs first thing, they're the only Daytona-specific risk.

## The UI data contract
`tests/sample_output_evasive.json` is the exact JSON the engine emits
(`--json`). Hand it to the designer/frontend so the console renders real shape:
`{ package, verdict{label,score,explanation,triggered_by,evidence,via}, traces[] }`.
Each trace has `bait`, `events[]` ({kind, detail}), `exit_code`. The mockup already
maps to this: per-package status, per-fork events, the highlighted differential,
verdict + explanation.

## File map
```
detonator/
  baits.py      bait environments (control/ssh/aws/prod/clock) + fake canaries
  trace.py      Trace model + strace parser (normalized paths)
  harness.py    in-sandbox provocation script + strace command
  runner.py     LocalRunner (offline) + DaytonaRunner (snapshot+fork)   <- spine
  analyze.py    cross-fork differential + AI verdict + AI provocation + fallback
  pipeline.py   package -> fork baits -> capture -> verdict
  cli.py        command-line entrypoint
demo_packages/  benign / canary_stealer / evasive_stealer (test fixtures)
tests/          smoke_test.py + sample_output_evasive.json (UI contract)
```

## Start sequence

**Tonight (everyone):**
1. `pip install -r requirements.txt`, get the smoke test passing on your machine.
2. Create Daytona accounts + keys, **set up billing** (extra $100 + parallel headroom),
   run the SDK hello-world so you know auth works.
3. Sushi + coder 2: read `runner.py`, resolve the three `CONFIRM-ON-DAY` calls against
   live Daytona docs, get one real package detonating on `--backend daytona`.
4. Designer: build the console from the mockup against `sample_output_evasive.json`.
5. QA: confirm the three demo packages behave (especially evasive firing only on AWS),
   and draft the demo script.

**On the day:**
- 11:00–12:00 — Sushi: real Daytona snapshot + fork working (control + AWS). Coder 2:
  wire AI verdict (`ANTHROPIC_API_KEY`). Designer: console on live data. QA: test harness.
- 12:45–14:30 — merge to a working floor end to end; one clean run of the evasive demo.
- 14:30–15:30 — add baits / AI provocation if ahead; fix breakages.
- 15:30 — FREEZE. Rehearse twice, record the backup video.

## Safety note
The demo packages are **test fixtures, not malware**. They read only fake planted
canary values and "exfiltrate" to unroutable RFC-5737 documentation IPs
(203.0.113.x). They reach nothing and steal nothing real; their only purpose is to
make the detector fire on demand.
