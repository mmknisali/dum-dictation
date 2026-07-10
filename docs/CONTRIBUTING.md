# Contributing

Most contributions are vocabulary fixes: teaching the tool a technical term it
mis-transcribes. Read the rule below first - a wrong call degrades the tool for everyone.

## The one rule: General vs Personal (the "JITHUB rule")

- **General** - the recognizer's fault. A word said normally, mis-transcribed by the model
  ("ten stack query" => `TanStack Query`, "postgress" => `PostgreSQL`). Belongs in the
  shipped vocab packs; any user speaking standard English hits the same error.
- **Personal** - your accent or idiolect. The model heard you correctly (you say "JITHUB"
  but mean GitHub). Does NOT belong in the shipped tool; a global "fix" breaks it for
  users who don't talk that way.

Litmus test: **would a general user, speaking standard English, produce this same error?**
Yes => General, add it. No => Personal, leave it out.

The trap: if you dictate to test the tool, you're both a tester and a person with an
accent. Two edits can look identical to the machine but have opposite verdicts:

- `Ugres => PostGres` - a recognizer mishear of "postgres" => General, accept.
- `the => this` - you changed your wording, not a mishear => Personal / neither, never add.

Only a careful human read tells them apart. When in doubt, leave it out.

## Adding a General term

1. Add a phrase alias to the relevant file in `packs/` (`spoken form => Canonical Form`,
   see existing entries). Use misheard jargon, never a common English word.
2. Run the gate; it must stay green:
   ```
   scripts/test
   ```
3. Open a PR describing the mishear and why it's General, not Personal.

## Code changes

Run `scripts/test` before opening a PR; correctness (term recall, WER, zero
over-correction) must not regress. See `DEV-NOTES.md` for the local dev loop and
`ARCHITECTURE.md` for how the pipeline fits together.
