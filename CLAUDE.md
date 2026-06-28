# Working on the `ouster-review` skill

This repo *is* a Claude Code skill. The deliverable is the prompt that steers a
reviewer — prose, not code. Judge every change by whether it makes the review
sharper and more disciplined, not by code metrics. (Read `README.md` for what the
skill does and how it's laid out.)

## Invariants — do not let these drift

These are the spine of the skill. Weakening any one is a regression even if the
prose reads well.

- **Every finding traces to complexity.** A symptom (change amplification /
  cognitive load / unknown unknowns) *and* a cause (dependencies / obscurity). If
  a guideline can't be tied to one, it's taste — cut it or label it as taste.
- **"Define errors out of existence" must never become "swallow errors."** This
  is the most dangerous misread of the book. The guardrails in `SKILL.md` and
  playbook §11 are load-bearing: redesign the *contract* so a case is normal;
  never silence a genuine fault. Don't soften that warning.
- **Leverage over coverage.** The skill's failure mode is a flat list of forty
  nits. Any edit that nudges toward exhaustiveness over prioritisation works
  against the whole point.
- **Depth is not bigness.** "Make it deeper" / "pull complexity down" must keep
  warning against the god-module overcorrection. Depth is
  interface-simpler-than-implementation, a ratio — not "merge everything."
- **Stay calibrated to Python.** Gradual typing and EAFP are not defects. Keep
  the language calibration honest; don't import another language's habits.

## Structure mirrors the principles it teaches

Eat the dog food. `SKILL.md` is the interface: loaded every session, so keep it
lean — push detail *down* into `references/`, don't bloat it. The references are
the implementation: loaded on demand.

- **Playbook entries follow a fixed shape:** *Symptom you'll see* / *Why it's
  complexity* / *The refactor* / *Do NOT overcorrect* / *Related flags*. New
  entries match it exactly. The "Do NOT overcorrect" half is not optional —
  every principle has an opposite failure mode, and naming it is what keeps the
  skill from doing harm.
- **Examples show the reshape move:** before → review → *designed twice, two
  interfaces side by side* → after. Match that register: blunt, specific, every
  claim tied to a cost.
- **Keep the SKILL.md priority list and the playbook order in sync.** They are
  meant to agree.

## Voice

Blunt and specific, in service of the code, never to perform rigor. Say the hard
thing plainly; don't be cold to be credible. Don't bikeshed names or
formatting — the skill explicitly disowns that, so its own prose must not model
it.

## Toolchain

- Markdown is linted with **rumdl** (`rumdl check`). Keep it clean; the cache
  lives in `.rumdl_cache/` (gitignored).
- The frontmatter `name:` in `SKILL.md` must match the install directory
  (`ouster-review`).

## Changing the trigger

The `description:` in `SKILL.md` frontmatter is what decides *when* the skill
fires. If you edit it, sanity-check that the intended phrasings still trigger and
that pure-linting requests still don't.

## Testing

There's no unit harness — the real test is dogfooding. Run `/ouster-review` on an
actual subsystem and check the output holds the bar: does it map structure before
judging, tie every finding to a complexity cost, prioritise, and design-it-twice
for the top issue? If it slips into a nit list, the prompt has drifted.
