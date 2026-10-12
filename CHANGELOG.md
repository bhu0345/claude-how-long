# Changelog

## 1.1.0 — 2026-10

Reliability
- Claude saves the page address in `progress-state.json` and recovers the page and its state after context compaction, instead of silently stopping updates.
- If the first progress write is refused, Claude turns live progress off for that task and uses its task list, instead of retrying.
- Time spent waiting for the user (`waitingSince`, `pausedSec`) no longer counts as work, so it doesn't stretch the ETA.
- After a long silence the page shows "may have stopped" instead of "running" forever. The silence threshold now uses the step's own estimate, not the pace-adjusted one.

Cost and setup
- The page template moved out of `SKILL.md` into `template.html`: the skill loads about 9 KB instead of 35 KB. Reinstall from the new zip.
- `SKILL.md` is now in English; the page stays bilingual (中文 / English).
- Triggers on tasks of about 10+ minutes or 6+ steps (was 5 / 5), and respects "no progress page" / "不要进度页".
- Sends a push notification when the task finishes, where the environment has one.
- README: where it works, how to tell it's working, troubleshooting, privacy and sharing.

## 1.0.0 — 2026-09

First release.
