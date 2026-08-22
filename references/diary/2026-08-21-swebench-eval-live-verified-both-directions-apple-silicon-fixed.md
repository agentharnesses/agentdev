---
description: swebench_eval.py live-verified against a real Docker run in both directions (gold patch resolves, empty patch doesn't) after fixing three real gaps the live attempt surfaced -- wrong dataset org, no arm64 pre-pull, and a misleading error on an empty patch. Supersedes the "still pending" entry from earlier today.
date: 2026-08-21 18:36 CDT
git:
  agentdev: 2a407d3
---

Direct continuation of `2026-08-21-swebench-eval-built-patch-verified-docker-run-still-pending.md`
from earlier today. User started Docker Desktop and asked to actually verify. Also asked to manage
Python dependencies with `uv` rather than the global `pip install` used first — switched to a
`uv`-managed `.venv` (`uv sync --extra swebench`, `.venv/` added to `.gitignore`) for everything
from here on.

Three real gaps found only by actually running it, none guessable from reading the harness's
source alone:

1. **Wrong dataset org.** `princeton-nlp/SWE-bench_Lite` (what `dataset.yaml` and
   `swebench_dataset.py` both used) has no `image` column at all — the installed harness
   (`swebench==5.0.2`) needs one (a pre-built Docker Hub tag per instance) and crashes with a bare
   `KeyError: 'image'` before evaluating anything. The dataset moved to a new canonical org,
   `SWE-bench/SWE-bench_Lite`, which does carry it. Fixed in `dataset.yaml`; `SweBenchInstance`
   gained an `image` field (empty string when absent, so the older org still loads without
   crashing) and `_parse_test_list`'s docstring now notes both orgs' real FAIL_TO_PASS encoding
   (JSON-string vs. native list — genuinely different between them, live-confirmed for both).
2. **Apple Silicon pull gap.** `swebench`'s own `create_container` calls
   `docker.Client.images.pull(image)` with no `platform` argument. Every published SWE-bench
   evaluation image is `linux/amd64`-only, so on this arm64 host that pull always 404s
   ("no matching manifest for linux/arm64/v8") before the harness ever gets to running anything —
   confirmed by reading `run_instance.log` directly, not guessed. Fixed by having `evaluate_patch`
   pre-pull the instance's image itself with `docker pull --platform linux/amd64` before invoking
   the harness; `client.images.get()` (which the harness calls first) then finds it already present
   and the broken pull path never runs. No-op on x86_64 hosts. Confirmed the image itself pulls
   fine under `--platform linux/amd64` (QEMU emulation, since this is Apple Silicon) before writing
   any code, to make sure the fix was pointed at a real, fixable gap and not a dead end.
3. **Empty-patch misreporting.** The harness's own `get_dataset` filters instances with an
   empty/`None` patch out of its run entirely and never writes a `report.json` for one — confirmed
   live: a run against an empty patch left no per-instance report, only an aggregate
   `empty_patch_instances` count. `evaluate_patch` previously waited on the full subprocess and
   then raised `SweBenchEvalError` claiming Docker probably wasn't running — actively misleading
   for a completely ordinary outcome (a `baseline` variant that made no changes). Fixed with a
   short-circuit before touching Docker at all, returning a result shaped like the harness's own
   empty-patch report fields would be.

All three had unit tests written for them (including one asserting `subprocess.run` is never
called for the empty-patch short-circuit) before the real live re-runs. Final live results, both
against `psf__requests-2317` on the corrected `SWE-bench/SWE-bench_Lite` dataset: the real gold
patch → `resolved: True`, all 8 `FAIL_TO_PASS` tests and all 118 `PASS_TO_PASS` tests genuinely
passing under actual execution; an empty patch → `resolved: False` via the short-circuit, no Docker
touched. `build_model_patch`'s own diff was what actually got applied and graded in the gold run —
not a separate check, the real patch this module produced round-tripped through the real harness
end to end.

Still true from the earlier entry, unchanged: not wired into `cli.py`'s live `run`/`batch`/`elo`
path. That remains a separate, deliberate next step (sandboxes are ephemeral, so it has to run
inline in the hot path) — this session closes the "is the module itself correct" question, not the
"is it integrated" one. `SUITES.md` also got a pass here — it was still describing the deleted
single-`question-set-1.yaml` layout and the old dataset org from before the crash, now corrected to
describe the dataset-native `dataset.yaml` shape and both grading signals accurately.
