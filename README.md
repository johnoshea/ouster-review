# ouster-review

> [!IMPORTANT]
> This repo is deprecated, as of 2026-09-07
>
> The skill has been migrated to the <https://github.com/johnoshea/johns-way> plugin

A Claude Code skill that runs a deep, opinionated code review in the style of
John Ousterhout's *[A Philosophy of Software Design][aposd]* (APoSD). It has a
single obsession: **reducing complexity** — how hard a system is to understand
and change.

The goal is to make `/ouster-review` feel as close as possible to having
Ousterhout pair with you: unvarnished opinions on the code, no hesitation about
calling for a significant reshape, and the willingness to rethink a design from
several layers up the stack rather than patch it locally.

## What it does

Pointed at a subsystem, file, diff, or PR, the skill:

- **Maps the real modules and their interfaces first**, top-down, before judging
  anything — it does not read line 1 downward.
- **Measures one thing: complexity.** Every finding must trace to a symptom
  (change amplification, cognitive load, or unknown unknowns) and a cause
  (dependencies or obscurity). If a point can't, it's labelled taste, not a
  defect — or cut.
- **Is strategic, not tactical.** It is licensed to say "this decomposition is
  wrong, here is the reshape" and to show the better interface, not just smooth
  over a local nit.
- **Prioritises ruthlessly.** A review that flags forty equal-weight nits is a
  failed review. It finds the few things that matter and makes the case hard.
- **Designs it twice** for the single highest-leverage problem — sketching the
  alternative interface side by side so you can see why it's simpler.

It deliberately does **not** do linting, formatting, or style-guide conformance.
A linter does that.

## When to use it

This is a deep, periodic audit — meant to be run rarely and deliberately on a
whole subsystem, not a per-PR gate. Reach for it when you want a structural
opinion on the *shape* of code: deep-vs-shallow modules, information
hiding/leakage, abstraction quality, or whether a layer should exist at all.

You don't have to name the book. Asking for a "design review", "complexity
review", "architecture review", or help making code "easier to understand and
modify" all trigger it.

## Install

Personal Claude Code skills live in `~/.claude/skills/<name>/SKILL.md`. Clone
this repo straight into that location (the directory name must match the skill
name):

```sh
git clone <this-repo> ~/.claude/skills/ouster-review
```

Or symlink a working copy:

```sh
ln -s "$(pwd)" ~/.claude/skills/ouster-review
```

Claude Code discovers the skill on its next session. For a project-scoped
install instead, use `.claude/skills/ouster-review/` inside the target repo.

## Use

In a Claude Code session, either invoke it directly:

```text
/ouster-review the src/ingest pipeline
```

or just ask for the kind of review it covers ("give me an Ousterhout-style review
of this module", "where's the unnecessary complexity in this subsystem?") and the
skill's description triggers it.

The review always comes back in a fixed shape: a one-paragraph **verdict**, the
**2–5 structural issues that matter** (each with the refactor and resulting
interface shown), a short ranked list of **lesser findings**, an honest note on
**what's already good**, and the **single highest-leverage next step**.

## Repo layout

The skill is structured the way it tells you to structure code — deep, not
shallow. `SKILL.md` is the interface (small, loaded every time); the references
are the implementation (loaded on demand).

| File | Role |
| --- | --- |
| `SKILL.md` | The reviewer's stance, reading method, priority list, and output format. Always in context. |
| `references/playbook.md` | Detection catalog: 17 red flags, each with the symptom, the principle, the refactor, and how *not* to overcorrect into the opposite failure. |
| `references/examples.md` | Worked before/after reviews showing the voice and the structural-reshape move. |

## A note on language

The principles are language-independent, but the skill is currently **calibrated
to Python** (gradual typing, EAFP, convention-based hiding). It will still review
other languages, but the Python-specific tells in the playbook won't all apply.

## Further reading

- *A Philosophy of Software Design*, John Ousterhout — [book site][aposd]

[aposd]: https://web.stanford.edu/~ouster/cgi-bin/aposd.php
