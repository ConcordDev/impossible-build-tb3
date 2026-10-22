# Impossible Build — Terminal-Bench 3 task submission

Author: Ramaj Duncan · Task: `terminal-bench/impossible-build`
Assignment: Klavis AI take-home (Xiangkai Zeng), due **Wed Oct 21, 2026, ~5:13 PM ET**.

> STATUS (2026-10-22): Task is final (physics v3) and Mac-validated.
> Submitted to Klavis AI (Xiangkai Zeng) on Thu Oct 22, 2026 with
> official agent trials honestly pending — see TRIALS.md: both trial
> sets are subscription-gated (Claude resets Thu Oct 22, Codex starts
> Fri Oct 23), and trial results will be pushed to this repo and the
> submission thread by the weekend.

## The task

Configure a compact electric utility vehicle from a catalog so the build
satisfies a coupled set of engineering requirements — gross mass, range over
a 210-substep route simulation with evolving coolant temperature and battery
state of charge, top speed, gradeability, a towing load case, launch,
thermal limits, and budget. Of 4,741,632 configurations, exactly 8 pass.
Full task documentation: `tasks/impossible-build/README.md`.
Specification given to agents: `tasks/impossible-build/environment/SPEC.md`.

## Validation record (v3, on the author's Mac)

- Static checks: **27 pass, 1 environment-limited** (check-image-digests;
  helper exists only in TB3 CI — both Dockerfiles pin base images by
  sha256 digest, verified manually).
- Oracle validation: **reward 1.0**, 0 exceptions (job 2026-10-15__02-50-09).
- Nop validation: **reward 0.0**, 0 exceptions (job 2026-10-15__02-51-08).
- Transfer integrity: task tarball sha256 `53b5c739…` verified on the Mac
  before install.
- Verifier/oracle agreement: 8/8 valid builds, 376/376 single-part swaps,
  4,000/4,000 random builds, 220/220 stateless survivors.
- Full receipts: `tasks/impossible-build/validation-receipt.json` and
  `tasks/impossible-build/calibration-receipt.json`.

## Trial record

See **TRIALS.md** for the full matrix. Summary: both bonus frontier-class
models reachable on the author's Ollama Cloud subscription (GLM-5.3 and
gpt-oss:120b) genuinely **fail** v3 after passing earlier calibrations —
the failure analysis is in **FAILURE_ANALYSIS.md**. Official trials
(Claude Code ×3 + cheat, Codex ×3 + cheat) run at the upstream TB3
defaults and are pending the two author steps listed in TRIALS.md.

## Model-ID note (flagged per the assignment's docs-win rule)

The Klavis assignment doc names a different model pair ("gpt 6.1 sol /
opus 5.5"). Those IDs appear in no upstream TB3 defaults file. The pinned
upstream `.github/harbor-run-defaults.yml`
(harbor-framework/terminal-bench @ da52319) specifies codex =
`openai/gpt-6-astra` (xhigh) and claude-code =
`anthropic/claude-fable-5-1` (max), 3 trials each. Official trials here
use/document exactly those defaults.
