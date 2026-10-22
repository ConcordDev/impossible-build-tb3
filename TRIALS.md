# Trial matrix — Impossible Build (physics v3, the submission calibration)

Upstream defaults (`.github/harbor-run-defaults.yml` @ da52319):
claude-code = `anthropic/claude-fable-5-1`, reasoning max, 3 trials;
codex = `openai/gpt-6-astra`, reasoning xhigh, 3 trials; plus 1 cheat
trial per agent, which must score zero reward. Agent window: 7,200 s.

## Official trials — TO RUN (the only open evidence)

| Agent | Model | Trial | Reward | Runtime | Failed checks | Job dir |
|---|---|---|---|---|---|---|
| claude-code | anthropic/claude-fable-5-1 (max) | 1 | _pending_ | | | |
| claude-code | anthropic/claude-fable-5-1 (max) | 2 | _pending_ | | | |
| claude-code | anthropic/claude-fable-5-1 (max) | 3 | _pending_ | | | |
| claude-code | anthropic/claude-fable-5-1 (max) | cheat | _pending — must be 0_ | | | |
| codex | openai/gpt-6-astra (xhigh) | 1 | _pending_ | | | |
| codex | openai/gpt-6-astra (xhigh) | 2 | _pending_ | | | |
| codex | openai/gpt-6-astra (xhigh) | 3 | _pending_ | | | |
| codex | openai/gpt-6-astra (xhigh) | cheat | _pending — must be 0_ | | | |

Pass criteria for the submission record: every standard trial must be a
GENUINE failure (real attempt, verifier ran; crashes/timeouts do not
count) and every cheat trial must score exactly 0.

### Status as of Tue Oct 20, 2026 (supersedes the Oct 15 blocker list)
Both official trial sets run on the author's own subscriptions per the
assignment brief; capped/exhausted runs count as void under the brief
("API or rate-limit failures do not count as model failures"), so
neither set can produce a genuine result before the Wed Oct 21,
~5:13 PM ET deadline:

1. **Claude:** the author's Claude usage is capped until the reset on
   **Thu Oct 22**. Plan unchanged once usage returns: stage the token
   from `claude setup-token` at `~/concord/run/tb3/claude-token.txt`
   on the Mac, then run ×3 + 1 cheat via
   `harbor run --ae CLAUDE_CODE_OAUTH_TOKEN=...` (subscription auth,
   zero API cost).
2. **Codex:** the author holds no Codex/ChatGPT subscription and can
   start Plus on **Fri Oct 23**; the ×3 + 1 cheat set runs that
   weekend at the upstream defaults above.
3. **Xiangkai notified Oct 20:** at the author's direction, a status
   note was sent to Xiangkai Zeng on the original invitation thread on
   Tue Oct 20, stating the task is built and validated, that both
   official trial sets are subscription-blocked until Thu/Fri
   respectively, and offering his choice: (a) repo sent Wed Oct 21
   with the official rows above honestly marked pending, completed by
   the weekend, or (b) repo held a few days and sent whole. Awaiting
   his reply as of this writing; whichever he picks, the wording in
   this file, the submission README, and the cover message must match
   it exactly.

## Bonus trials — DONE (calibration evidence, not the official record)

Run by the author on his Ollama Cloud subscription against the final v3:

| Model | Result | Runtime | Submitted build | Failed checks |
|---|---|---|---|---|
| gpt-oss:120b-cloud | FAIL 0.0 (15/18) | 2m27s | CH-F/BT-5/MT-5/GB-2/IN-4/CL-4/TR-6/SU-2 | grade_torque, soc_floor, range |
| glm-5.3:cloud | FAIL 0.0 (14/18) | 15m12s | CH-C/BT-5/MT-4/GB-2/IN-4/CL-2/TR-2/SU-4 | tire_load, grade_torque, soc_floor, range |

One earlier glm-5.3 v3 attempt was cancelled at 28m on a hung Ollama
Cloud request (infrastructure, not a trial) and is not counted; the
15m12s retry above is the genuine record. Full v1→v3 history and the
failure analysis: FAILURE_ANALYSIS.md.

## After the trials (submission checklist)
- [ ] Fill the official table above from the Harbor job outputs
- [ ] Append official-trial results to FAILURE_ANALYSIS.md
- [ ] Re-run static checks on the final repo copy
- [ ] Push as a new GitHub repo and submit the link to Xiangkai before
      Wed Oct 21, 2026, ~5:13 PM ET
- [ ] Cleanup at wrap: delete `~/concord/run/first-api-key.txt` on the
      Mac and deactivate the `zuko-first-key` api_keys row; remove
      `~/concord/run/tb3-task.b64`
