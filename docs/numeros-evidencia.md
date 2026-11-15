# Measured numbers

Figures **measured** on our machine (Windows 11, .NET SDK 8.0.428) and on Modal, on 14/11/2026. Source of each
figure: `fixtures/evidence_*.json`, `fixtures/report_2283_dotnet.json`, the `fixtures/measure_*` and `fixtures/capacity_*`
files, and `docs/lessons-log.md`. Stages: **T1** triage, **T2** executable evidence, **T3** cross-examination.

## Result per issue (T2, evidence)

| Issue | Type | Change test (written from the issue alone) | Generation invariance | Reading |
|---|---|---|---|---|
| #2283 | bug fix | ✅ red on base → green on PR | ✅ 21/21 general cases identical, deterministic | Complete evidence |
| #2234 | bug fix | ❌ also passes on base (weak assertion; the issue is one line long) | ❌ changes `general-duplicate-int-keys-not-suppressed` | A legitimate RED: no proof of the bug, and the fix changes behaviour outside the issue |
| #2297 | feature | ✅ green on PR (and red on base: it measures something new) | ✅ 30/31 identical after the **declared normalization**; only its own case changes | Complete evidence, with the transformation in plain sight |
| #2129 | feature | ❌ also fails on PR (the test picks the resolver's file) | ✅ 21/21 general cases identical; the formatter is generated on PR | A false red from the AI; the invariance shows the failure is in the test |

Corpus: 31 snippets (21 `general` + 10 issue-specific), run in **two isolated processes** (base twice, PR once).

## T3 cross-examination (#2283, full run with local dotnet)

Prosecutor `nvidia/nemotron-3.5-super-vl-120b-a12b`; Defender and mutants `gemini-4-hydrogen`.

- Objection 1: the test is **green on PR and red against the mutant** → valid (the PR covers that branch).
- Objections 2 and 3: the Defender's test **fails on the PR** → a real defect: the fix **moves** the crash
  (`IndexOutOfRangeException` → `InvalidOperationException`, `TypeCollector.cs:1013`) for a generic type from
  metadata with an invalid key, and the resolver is still not generated. Reproduced separately with `GeneratorProbe`.
- Result: `TypeCollector.cs` is RED, "the system refuses to decide", with the Prosecutor's exact question.

## Timings

| What | Local (Windows) | Modal |
|---|---|---|
| First build of the repo (cold) | ~2 min | image: 5.5 min **once** (8 precompiled worktrees) |
| One `dotnet test` of a generated test (incremental) | 10–25 s | 20–30 s per sandbox |
| Base + PR + mutant (`python -m ledger.evidence.bench --issue 2283`) | 54–63 s sequential | **28–29 s in parallel** (3 network-less sandboxes) |
| Invariance (31 cases, base ×2 + PR) | 35–70 s | (runs locally) |
| Change test with the LLM (includes 2 dotnet runs) | 0.8–5.5 min | — |
| Full #2283 pipeline with T3 | 18–22 min | **11 min** (`LEDGER_RUNNER=modal`; a first attempt stalled, see lessons log H6) |

## Full pipeline on Modal (#2283)

`LEDGER_RUNNER=modal`, same models. All AI-written code (change test, Defender tests and mutants) runs in
network-less sandboxes. Change test red on base and green on PR; invariance ok; T3 with **2 of 3 objections
validated** against their mutant and the third rejected by the anti-cheat rule (the test also passed against the
mutant). The Prosecutor's questions differ from the local run: LLMs do not repeat their answers, and the prompts had
moved to English. Report: `fixtures/report_2283_modal.json`; benchmark: `fixtures/bench_2283.json`.

## Full pipeline on Modal with T3 in parallel (14/11/2026)

`python -m ledger.evidence.measure <issue> par <dir>` with an empty LLM cache; limits: Gemini 32, Nemotron 16, Modal 60.

| | #2297 (237 files) |
|---|---|
| Wall time | **1024 s (17 min)**; sequentially it ran past 1 h and was cancelled |
| Cross-examination | 12 files, **36 objections**: 12 valid, 12 with a test that fails on the PR (a possible defect or a wrong test), 12 with a test that also passes against the mutant |
| Verdicts | 223 M4 + 2 M5 green · 10 M7 (refuses to decide) · `TransformUtilities.cs` M1 · `README.md` M2 |
| Modal | 126 sandboxes, **real peak 16** (limit 60), 0 failures |
| Gemini | 131 requests, peak 32 (= limit), **0 errors 429**, 1 timeout (300 s), 39 s spent waiting for a slot in total |
| Nemotron | 12 requests (one per file), peak 12 |

Sources: `fixtures/measure_2297_par_v1.json`, `fixtures/report_2297_modal.json`. Reading: the Gemini limit did fill
up, but it only cost 39 s of waiting; wall time is set by the slowest chain of long calls (up to 300 s each), not by
concurrency.

**#2283 in parallel.** Clean measurement (run alone, with Defender variants, D15): **622 s (10.4 min)**; 0/3 valid
objections and `TypeCollector.cs` red (`fixtures/measure_2283_par.json`). Reference: 11 min sequentially, with part of
the cache warm. The first two measurements were launched at the same time by mistake and shared the cache (447 s and
796 s; `fixtures/measure_2283_par_overlap_{a,b}.json`, lessons log H8): they are not valid as figures, but they show
that two runs of the same PR give different results (3/3 valid in one, 1/3 in the other).

### Optimising #2297: four versions, one run each

| Version | Wall time | Valid objections | Gemini: requests · errors · waiting for a slot | Backups (useful) | p90 of answers |
|---|---|---|---|---|---|
| v1: limits 32/16 (`_v1`) | 1024 s (17.1 min) | 12/36 | 131 · 1 timeout · 39 s | — | 456 s |
| v2: + T1 and T2 at the same time, parallel triage, adaptive variants (D15) (`_v2`) | 1054 s (17.6 min) | 8/36 | 142 · 2 × 504 · 48 s | — | 396 s |
| v3: + hedging with the median per provider (`_v3`, lessons log H9) | **833 s (13.9 min)** | 3/36 | 232 · 0 · 8983 s | 79 (6) | 595 s |
| v4: + hedging per model and only with spare capacity (`_v4`) | 1192 s (19.9 min) | 6/36 | 195 · 6 × 504 · 449 s | 40 (4) | 591 s |

Sources: `fixtures/measure_2297_par_v{1,2,3,4}.json`.

**Conclusion (what we can claim and what we cannot):**
- Proven: raising the Gemini limit from 8 to 32 removed the queue (over 1 h sequentially versus 17 min), and Modal was
  never the limit (a real peak of 14–16 sandboxes out of 60).
- In #2297 the end is set by **one straggling Defender call**: in v2, 90 % of the answers had arrived by 396 s and the
  last one arrived at 1006 s.
- Gemini's latency variation between runs (p90 from 396 to 595 s; from 0 to 6 errors 504) is **larger than the effect
  of each improvement**. With one run per version, the difference cannot be attributed to the code. That is why
  hedging is **off by default** (`LEDGER_HEDGE=1` turns it on): extra spend with no proven benefit.
- Valid objections range from 3 to 12 out of 36. We do not know whether that is LLM noise or rounds lost to 504s.
- To really cut the time, another lever is needed: a faster model for the Defender (at a cost in quality) or more
  provider capacity. Measuring it properly would take ~3 runs per version (15–20 min and ~$0.5 of Modal each).

## Concurrency limit (capacity sweep, 14/11/2026)

`python -m ledger.evidence.capacity <modal|llm pro|llm fiscal> <levels>`: N identical jobs at once per level, each
resource on its own. Sources: `fixtures/capacity_modal.json`, `fixtures/capacity_llm_pro.json`, `fixtures/capacity_llm_fiscal.json`.

| N at once | Modal: real #2283 test (total / max) | Gemini pro `gemini-4-hydrogen`, ping (total / max) | Nemotron, ping (total) |
|---|---|---|---|
| 1 | 18.7 s / 18.4 s | 5.9 s | 4.3 s |
| 8 | 20.0 s / 19.7 s | 5.5 s | 1.3 s |
| 16 | 23.1 s / 22.5 s | 5.9 s | 1.3 s |
| 32 | 32.1 s / 30.9 s | 9.6 s / 9.2 s | 1.7 s |
| 48 | — | 11.8 s / 11.7 s | — |
| 60 / 64 | **27.4 s / 27.0 s (60)**, 0 failures | **29.9 s / 29.2 s (64)**: throughput halves | — |

- **Modal is not the limit:** 60 sandboxes at once (real peak 60) finish in 27 s with no failure: 131 runs/min versus 3.2 with one.
- **Gemini pro is the ceiling:** no 429 across the whole sweep, but from 48 on the requests queue at the provider and at
  64 throughput collapses (243 → 129 per minute). The ping measures concurrency, not tokens: real prompts weigh much
  more, so the default limit is **32** (it was 8, which was the real bottleneck).
- **Nemotron** scales with no problem up to 32; the Prosecutor makes one call per file, so its limit is **16**.
- On top of that, the first Defender call is requested **at the same time** as the mutant (it never sees it): one
  fewer `pro` call on the critical path of each objection.

## Cost

- Modal, total for the whole hackathon (`modal billing report --for "this month"`): **$2.86 in credits**, $0 in charges
  (usage limit set to the free credits).
- LLM: on-disk cache keyed by the prompt's hash; replaying the demo makes no new calls.

## AI failures the funnel stopped (details in `docs/lessons-log.md`)

| Failure | Layer that stopped it |
|---|---|
| Nemotron invents the Roslyn API (#2283) | T2: a test that does not compile does not count |
| A weak assertion that passes on base (#2234) | T2: in a bug fix, the test must fail on base |
| A test that looks at the wrong file (#2129) | T2: a test that does not pass on the PR does not count; the invariance points at the test |
| The #2283 fix moves the crash instead of removing it | T3: the Defender's test fails on the PR |
| A network failure counted as "bug reproduced" (network-less sandbox) | T2: infrastructure failures are marked `INFRA` and are never evidence |
| The scaffold's design used `Sandbox.open`, which does not exist in Modal 1.6.4 | Check the installed API before implementing |

## Known limits (honesty)

- The **invariance** runs locally (the probe does not go through the `TestRunner`); on Modal runs all the AI-written code.
- `InvarianceEvidence.unexpected_diffs` guards every case outside the issue only when the evidence carries its
  `issue_id`; without it (the recorded #2129 fixture), only the `general` cases count. In the four PRs no other
  issue's case changes (checked).
- The local runner executes the AI's tests and mutants **without isolation** (which is why mutants are limited to
  `src/**/*.cs`); for untrusted code, use `LEDGER_RUNNER=modal`.
- A badly written issue produces weak evidence (#2234): the system says so, it does not fix it.
