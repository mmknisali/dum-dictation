# Dogfood logging

`dogfood_log.py` measures the User Correction Rate (how much you edit dictated text after commit), so we can tell whether the vocab/repo/fuzzy-symbol features actually reduce corrections.

## Flag a bad dictation

Double-tap the LEFT Option key (same gesture style as the double-tap left Cmd that starts/stops dictation). Marks the last committed dictation as a problem (`user.verdict` event), plays a cue, logs `[FLAG]`. Flag only when dissatisfied; there is no "good" button. Audio + correction context are already saved, so flagged items replay offline. List them:

```
.venv/bin/python scripts/analyze_user_corrections.py dogfood/sessions/*.jsonl   # flagged problems at top
```

## On / off

Master switch: `DUM_DOGFOOD_FULL`. Turns on the whole capture stack (dogfood logging, audio retention, keystroke proxy, correction pairs, fuzzy-symbol recovery). Every piece defaults OFF at the code level, so shipped builds are privacy-first. The `./dum` launcher sets `DUM_DOGFOOD_FULL=1` for dogfood sessions.

Per-piece overrides:

```
DUM_KEEP_AUDIO=0 ./dum          # drop audio clips
DUM_KEYSTROKE_PROXY=0 ./dum     # drop keystroke counts
DUM_KEEP_CORRECTIONS=0 ./dum    # drop verbatim correction pairs
DUM_FUZZY_SYMBOLS=0 ./dum       # disable fuzzy-symbol recovery
DUM_DOGFOOD_FULL=0 ./dum        # disable everything
```

## Storage (local only)

- `dogfood/sessions/dictation-<session>.jsonl` - one file per run
- `dogfood/audio/<session>/<commit_id>.wav` - utterance clips
- The whole `dogfood/` tree is gitignored: never committed, never uploaded, no network anywhere
- Delete everything: `rm -rf dogfood/sessions dogfood/audio`

## What is logged

Per commit (`type: "commit"`):
- timestamp, session id, cwd, repo root (if a git repo)
- focused app name, surface, insertion mode
- raw recognizer transcript, final committed text, whether it changed
- model name, feature flags (`global_vocab`/`repo_vocab`/`fuzzy_symbols`/`llm`)
- latency (ms), committed length, word count

Post-commit, best-effort (`type: "user.refix"`):
- `edit_distance`, `normalized` rate, `accepted_unchanged`
- truncated snippets of committed vs final text (max 200 chars)
- if Accessibility can't read the focused field (most non-native apps): just `edit_capture: "unavailable"`, no field content captured

## Audio retention (dogfood default: ON)

Each committed utterance is saved as a small WAV (16 kHz, ~32 KB/s) so recognition failures can be replayed offline against pipeline/recognizer changes, in every app - including where edit capture is blind (VS Code, terminal). The commit record stores an `audio_ref` (path + sha256 + seconds).

- Written after the text is on screen; adds no dictation latency
- Turn off per run: `DUM_KEEP_AUDIO=0 ./dum`
- Auto-pruned at session start by both caps, whichever bites first: older than 30 days OR total over 2 GB (oldest first). Override: `DUM_AUDIO_MAX_DAYS`, `DUM_AUDIO_MAX_GB`. Pruning logs an `audio.prune` event so dropped coverage is never silent
- Shipped builds flip this OFF / opt-in; only the dogfood profile defaults it on

## Post-commit signals (Step 4)

Did you fix the dictation, or move on? AX text-capture is blind in the main apps and gives false signals in others, so each `user.refix` also records these signals for ~20s after a commit. They work in every app:

- App-switch timeline: `commit_app`, `app_switches[{t_rel, app}]`, `final_app`, `switched_away_s`. Polls the frontmost app ~1x/s. App names only, no content.
- Keystroke proxy: `keystroke_summary{backspaces, deletes, nav_keys, other_keys}`. Counts only, never which characters; gated to the commit's app. Reuses Input Monitoring. Disable per run: `DUM_KEYSTROKE_PROXY=0 ./dum` (app-switch timeline stays).
- `capture_method` (`ax` | `keystroke` | `unavailable`): which edit signal was available.
- `correction_pair` (when AX-readable): the minimal changed-token diff `committed -> corrected` (e.g. `postgress -> PostgreSQL`), NOT the whole field. Core learning signal and vocab/alias-candidate source. Default ON in dogfood (your own text, local-only); `DUM_KEEP_CORRECTIONS=0` disables it (keeps the distance, drops the verbatim pair).
- Shipped builds flip the keystroke proxy AND correction_pair OFF / opt-in.

## Privacy guarantees

- Never logs the surrounding document: edit capture stores only the dictated text and a truncated window of the edited region (`REDACT_MAX = 200` chars), not the whole file/field
- The raw/final transcripts are your dictated speech (the thing being measured); they stay local
- No cloud, no telemetry, no network calls

## Analyze

```
.venv/bin/python scripts/analyze_user_corrections.py dogfood/sessions/*.jsonl
```

Reports the edit-capture breakdown first (total commits = observable + unobservable, where unobservable = AX-unavailable + no-signal, plus coverage %), then on the observable subset only: accepted-unchanged %, avg edit distance, User Correction Rate, corrections per 100 words. Always available: top repeated mishears, rate by app / repo / feature flags, and a `DUM_FUZZY_SYMBOLS` on-vs-off comparison. Every correction-rate number is labelled observable-only, so a "10% correction rate" at 20% coverage can't be mistaken for one at 90%.

## Known limitation

Many apps (terminals, VS Code, browsers) don't expose the focused text field to macOS Accessibility, so `edit_capture` is often `unavailable`. The analyzer reports coverage; commit-level stats (volume, mishears, by app/repo/flag) are always available regardless. Next step: per-app AX handlers, or an inserted-range diff.
