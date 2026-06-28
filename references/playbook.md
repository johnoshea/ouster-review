# Ousterhout Review — Playbook

A detection-first catalog. You start from a **symptom** you can see in the code,
and each entry tells you what it means, the principle behind it, the refactor,
and — just as important — how *not* to overcorrect into the opposite failure.

Entries are ordered by leverage, roughly matching the priority list in SKILL.md.
You don't need to read all of this; jump to the flag you're looking at.

## Contents

1. [Shallow Module](#1-shallow-module) — *deep modules*
2. [Information Leakage](#2-information-leakage) — *information hiding*
3. [Temporal Decomposition](#3-temporal-decomposition) — *information hiding*
4. [Overexposure](#4-overexposure) — *simple common case*
5. [Pass-Through Method](#5-pass-through-method) — *different layer, different abstraction*
6. [Pass-Through Variable](#6-pass-through-variable) — *different layer, different abstraction*
7. [Conjoined Methods](#7-conjoined-methods) — *dependencies / decomposition*
8. [Complexity Exported to Callers](#8-complexity-exported-to-callers) — *pull complexity downward* (extra)
9. [Special-General Mixture](#9-special-general-mixture) — *separate general & special purpose*
10. [Repetition](#10-repetition) — *missing abstraction*
11. [Special Cases and Errors as Complexity](#11-special-cases-and-errors-as-complexity) — *define errors out of existence* (extra)
12. [Comment Repeats Code](#12-comment-repeats-code) — *comments describe the non-obvious*
13. [Implementation Detail Contaminates Interface](#13-implementation-detail-contaminates-interface) — *interface vs implementation comments*
14. [Hard to Describe](#14-hard-to-describe) — *design smell*
15. [Vague Name](#15-vague-name) — *choosing names*
16. [Hard to Pick a Name](#16-hard-to-pick-a-name) — *design smell*
17. [Nonobvious Code](#17-nonobvious-code) — *code should be obvious* (catch-all)

---

## 1. Shallow Module
*Principle: modules should be deep.*

**Symptom you'll see:** A class or function whose interface is about as complex as
its implementation. Tells in Python: classes that are mostly `@property`
getters/setters over fields; dataclasses that hold state while all the real
behavior lives in free functions elsewhere; "manager"/"helper"/"util" classes
that wrap a couple of calls; a method whose signature is as hard to learn as just
inlining its body; callers that must make several calls in a fixed sequence to get
one outcome.

**Why it's complexity:** A module's *cost* to the system is its interface; its
*benefit* is the functionality it hides. A shallow module pays nearly full
interface cost while hiding little, so it adds cognitive load (more surface to
learn) without buying abstraction. Many shallow modules also create dependencies —
callers must orchestrate them — which drives change amplification.

**The refactor:** Make it deeper — give it more functionality behind the same or a
*simpler* interface, so callers say *what* they want and the module handles *how*.
Often: merge several shallow modules that are always used together; absorb a
caller-side sequence into one call; replace a knob the caller must understand with
a sensible default. Show the resulting interface and count what the caller no
longer has to know.

**Do NOT overcorrect:** "Deeper" is interface-simpler-than-implementation, not
"bigger," and not "merge unrelated things." If combining two modules makes the
*interface* more complex, or fuses two concerns that change for different reasons,
you've built a god-module — worse than what you started with. Depth is a ratio;
don't chase it by inflating the implementation.

**Related flags:** Pass-Through Method, Conjoined Methods, Overexposure.

---

## 2. Information Leakage
*Principle: information hiding.* This is one of the most important flags in the
book — when you find real leakage, it's usually your top finding.

**Symptom you'll see:** A single design decision shows up in two or more modules,
so they must change together. Tells in Python: the same file format, wire format,
or date format parsed/emitted in several places; magic column indices or dict keys
known to multiple modules; one module that builds a structure and another that
must know its internal layout to consume it; two classes that both encode the same
assumption about units, ordering, or protocol. A good detection heuristic: ask
"if this decision changed, how many files would I have to touch, and would I find
them all?"

**Why it's complexity:** The shared decision is a dependency the type system won't
catch and that's easy to miss (obscurity), so it produces both change
amplification and, worse, unknown unknowns — a developer changes one copy and the
others silently drift.

**The refactor:** Encapsulate the decision behind one module that owns it, so
nothing else needs to know it. The format lives in one parser/serializer; the
schema lives behind one accessor; the ordering assumption is enforced in one
place. Other modules depend on that module's *interface*, not on the decision
itself.

**Do NOT overcorrect:** Not all shared knowledge is leakage. Two modules
referencing the same genuinely-stable fact (a mathematical constant, a protocol
that will never change) isn't a defect, and over-abstracting to "centralize" it
can add a shallow indirection module that's worse. The test is whether the shared
thing is a *decision that will plausibly change together* — if yes, hide it; if
it's a fixed fact, leave it.

**Related flags:** Temporal Decomposition, Duplication, Conjoined Methods.

---

## 3. Temporal Decomposition
*Principle: information hiding (decompose by knowledge, not by execution order).*

**Symptom you'll see:** The module structure mirrors the *order operations run in*
rather than the *knowledge they share*. The classic shape: separate
`ReadX` / `ProcessX` / `WriteX` modules (or steps) where the *same* knowledge —
the format of X — has to live in both the reader and the writer. In Python, a
pipeline of stage-functions or stage-classes where each stage re-derives or
re-encodes the same structural assumption.

**Why it's complexity:** Splitting by time forces the shared knowledge to leak
across the stages (see Information Leakage), so a change to the format means
changing every stage that touches it — change amplification, with drift risk.

**The refactor:** Organize around knowledge instead. The thing that *knows the
format of X* handles both reading and writing it, so that knowledge lives in one
module; the pipeline orchestrates calls to it rather than re-implementing pieces
of it. Order-of-execution becomes a thin coordinator over knowledge-owning
modules.

**Do NOT overcorrect:** Some pipelines genuinely have independent stages with no
shared knowledge (each stage transforms a stable, well-defined intermediate). Don't
collapse those just to avoid the *appearance* of temporal decomposition — forcing
unrelated stages together creates coupling. The smell is specifically *shared
knowledge split across time-ordered units*, not "any pipeline."

**Related flags:** Information Leakage, Shallow Module.

---

## 4. Overexposure
*Principle: design the interface to make the most common usage simple.*

**Symptom you'll see:** Using the common case forces the caller to learn or supply
things only the rare case needs. Tells in Python: a function with many required
parameters where most callers want the same defaults; an `__init__` that demands
configuration objects for the 90% case; a single entry point that exposes
advanced/rarely-used options inline with everyday ones; `**kwargs` signatures that
make the caller guess what's actually accepted.

**Why it's complexity:** The common path carries the cognitive load of the
uncommon path. Every caller pays the learning cost of features they'll never use.

**The refactor:** Make the common case trivial — sensible defaults, a minimal
required signature — and move advanced features onto a separate, optional path
(extra keyword args with defaults, a builder, a separate "advanced" entry point).
The 90% caller should need to know almost nothing.

**Do NOT overcorrect:** Don't split one coherent operation into a maze of tiny
"simple" entry points to avoid parameters — that just trades overexposure for a
discovery problem (which function do I even call?). And defaults must be *safe*
defaults; hiding a consequential choice behind a default the caller doesn't notice
is its own trap.

**Related flags:** Shallow Module, Nonobvious Code.

---

## 5. Pass-Through Method
*Principle: different layer, different abstraction.*

**Symptom you'll see:** A method that does almost nothing except call another
method — often with the same or a nearly identical signature — and return its
result. In Python: `def save(self, x): return self._store.save(x)` repeated across
a class; a "service" or "manager" whose methods each forward one-to-one to a
repository or client. A strong tell: many of a class's public methods are
one-liners delegating downward.

**Why it's complexity:** The method adds interface (one more thing to learn, one
more place to look) without adding abstraction — the two adjacent layers express
the *same* abstraction, so the upper one is pure cost. It also creates a
dependency: the signatures must stay in lockstep.

**The refactor:** Remove the redundant layer, or give it a real job. Options:
expose the lower object and let callers use it directly; or let the upper method
actually *add* something (combine several lower calls, enforce an invariant,
translate to a genuinely different abstraction). If after the change the method
still just forwards, it shouldn't exist.

**Do NOT overcorrect:** A thin layer that forwards is legitimate when it's a real
*boundary* doing real work — a public API facade that stabilizes an interface
against internal churn, or an anti-corruption layer translating an external
model into yours. The difference: a boundary adapter changes the abstraction or
isolates change even if the code looks like forwarding; a pass-through repeats the
same abstraction for nothing. Don't delete boundaries; delete redundant echoes.

**Related flags:** Pass-Through Variable, Shallow Module.

---

## 6. Pass-Through Variable
*Principle: different layer, different abstraction.*

**Symptom you'll see:** A variable threaded through a long chain of functions or
methods purely so something deep down can use it — every function in the middle
accepts it and passes it on without touching it. In Python, the parameter that
appears in five signatures along a call path and is only *read* at the bottom.

**Why it's complexity:** It creates a dependency across every intervening function
(they all must know about a thing that isn't theirs), and adding a new
deep-down need means editing the whole chain — change amplification. The middle
functions' interfaces are polluted by knowledge they don't use.

**The refactor:** Get the value to where it's needed without threading it through
everyone. Options, in rough order of preference: store it on a shared *context*
object that the relevant code already has access to; make it available through the
object that owns the deep code; or, if truly cross-cutting and read-only, a
well-scoped module-level configuration. The middle functions should stop mentioning
it.

**Do NOT overcorrect:** A single grab-bag "context" object that accumulates every
loosely-related value becomes its own obscurity — callers can't tell what's in it
or what's safe to touch. Keep context objects coherent and minimal; don't turn a
threading problem into a global-state problem.

**Related flags:** Pass-Through Method, Information Leakage.

---

## 7. Conjoined Methods
*Principle: dependencies / decomposition.*

**Symptom you'll see:** Two methods (or functions) so interdependent that you
can't understand one without reading the other — shared mutable state, an implicit
ordering contract ("you must call `_prepare` before `_run`"), or each making
detailed assumptions about the other's internals. In Python, watch for methods
that communicate through instance attributes set as side effects rather than
through parameters and return values.

**Why it's complexity:** High cognitive load — the unit you actually have to hold
in your head is "both methods plus their hidden contract," not either one alone.
The split gives the *appearance* of decomposition without the benefit.

**The refactor:** Either make each method understandable on its own — communicate
through explicit arguments and return values instead of shared state, remove
hidden ordering contracts — or, if they're genuinely one operation that was split
arbitrarily, merge them into one method that's coherent. The goal is that a reader
can understand each piece in isolation.

**Do NOT overcorrect:** Merging isn't always right — if the two methods are large,
fusing them can create a long, hard-to-follow procedure. Prefer fixing the
*communication* (explicit in/out) over blindly combining. And some coupling is
intrinsic; the target is "understandable independently," not "zero shared
context."

**Related flags:** Shallow Module, Nonobvious Code.

---

## 8. Complexity Exported to Callers
*Principle: pull complexity downward.* (Not one of the book's named flags, but the
detector for a top-tier principle.)

**Symptom you'll see:** A module makes its own implementation simpler by pushing
work, decisions, or cleanup onto every caller. Tells in Python: callers must
remember to call `close()`/`cleanup()` (no context manager provided); callers must
check a returned sentinel or status and handle it; callers must pass configuration
the module could default; the same post-processing appears at every call site
because the module returns something half-finished.

**Why it's complexity:** One simplification inside the module is paid for N times
at the call sites — and each caller can get it wrong (cognitive load + a class of
bugs). It's the wrong trade: it's more important for the *interface* to be simple
than the implementation.

**The refactor:** Pull the complexity down into the module, behind the interface.
Provide a context manager so callers can't forget cleanup; return a finished
result instead of a half-finished one; supply the sensible default; handle the
common condition internally. The caller's job should shrink to stating intent.

**Do NOT overcorrect:** "Pull down" is "pull down when it simplifies the
interface," not "absorb everything." Don't pull a genuinely *caller-specific*
decision into the module — that forces the module to know things it shouldn't and
breeds special-case parameters (see Special-General Mixture). The aim is a simpler
interface, not a module that secretly makes choices on the caller's behalf.

**Related flags:** Overexposure, Special-General Mixture, Special Cases and Errors.

---

## 9. Special-General Mixture
*Principle: separate general-purpose and special-purpose code.*

**Symptom you'll see:** Special-purpose logic baked into a general-purpose
mechanism, creating a dependency between them. Tells in Python: a generic utility,
base class, or framework component with an `if`/branch handling one specific
caller's case; a reusable function that hardcodes one application's keys,
formats, or policy; a "generic" component that imports from a specific feature
module.

**Why it's complexity:** The general mechanism is now coupled to the special case,
so it's harder to reuse and harder to understand (you must mentally subtract the
special bits), and the special case is harder to find. Both sides get more
complex.

**The refactor:** Pull the special-purpose code *up and out* into a higher layer
that *uses* the general mechanism, leaving the general mechanism free of any
specific case. The general code exposes a clean, case-agnostic interface; the
special behavior is composed on top.

**Do NOT overcorrect:** This is in direct tension with YAGNI. Don't generalize a
mechanism for *imagined* future cases — "general-purpose is deeper" means general
enough to serve *today's* needs cleanly, not speculative flexibility. A mechanism
made abstract for users who don't exist yet is often shallower and more obscure,
not deeper.

**Related flags:** Repetition, Complexity Exported to Callers.

---

## 10. Repetition
*Principle: a missing abstraction (general-purpose modules are deeper).*

**Symptom you'll see:** A nontrivial chunk of code appears in multiple places, so a
change must be made in all of them. In Python: copy-pasted blocks with small
edits; the same validation/transform inlined at several call sites; parallel
`if`-ladders that encode the same rule.

**Why it's complexity:** Change amplification, plus drift risk — someone updates
three of the four copies and the fourth silently rots into a bug (unknown
unknowns).

**The refactor:** Factor the repeated logic into one well-named function or method
that everyone calls. If the repetition is *structural* (the same shape with
different details), the abstraction may be a helper that takes the varying parts as
parameters, or a small strategy object.

**Do NOT overcorrect:** Beware *incidental* duplication — two pieces of code that
merely look alike today but represent *different decisions* that will evolve
independently. Coupling them under one abstraction means a future change to one
forces an awkward parameter or branch into the other. A wrong abstraction is worse
than duplication; if you can't name the shared concept cleanly, leave the copies
apart and revisit when the pattern is real.

**Related flags:** Special-General Mixture, Information Leakage.

---

## 11. Special Cases and Errors as Complexity
*Principle: define errors (and special cases) out of existence.* (Not one of the
book's named flags, but the detector for a top-five principle.)

> **READ THIS FIRST — the most dangerous principle to misapply.** "Define errors
> out of existence" means **redesign the contract so the condition is normal and
> needs no special handling.** It does **NOT** mean catch-and-ignore. Never
> recommend swallowing a genuine fault — an I/O failure, a real validation error,
> a programming bug, a violated invariant. When in doubt, surface the error. The
> move below is about *eliminating* special cases by design, not *hiding* failures
> at runtime.

**Symptom you'll see:** A proliferation of special-case branches and exception
handling for conditions that could have been defined as ordinary. Tells in Python:
APIs that raise for benign cases callers must wrap (`KeyError` where a default
would do, an exception when removing something that isn't there, an error on an
empty/zero/edge input); defensive `if x is None` checks scattered across many call
sites because a function *might* return `None`; `Optional[...]` threaded
everywhere because one function won't commit to always returning a value; the same
try/except boilerplate repeated at every call site.

**Why it's complexity:** Every special case is a branch every caller must know
about and handle — cognitive load multiplied across call sites, and each unhandled
case is a latent bug (unknown unknowns). The book's insight: most exceptions are
*defined into existence* by an API design choice, and a different design can make
them simply not arise.

**The refactor (by design, not by silencing):**
- **Make the edge case normal.** Redefine the operation so the "error" is a
  legal outcome: removing a missing item is a no-op; a range query clamps to
  bounds; a lookup returns a documented default. Now there's nothing to branch on.
- **Return a total result instead of `None` + checks.** If a function can always
  return *something* meaningful (an empty collection, a null-object, a defined
  default), callers stop checking and the `Optional` disappears.
- **Mask low, so high layers don't see it.** Handle a condition once, deep in the
  module that's positioned to deal with it, so it never propagates as a special
  case to everyone above.
- **For truly unrecoverable conditions, fail fast and loud** — don't manufacture a
  fake "normal" value. Defining errors out of existence is for cases the design
  can legitimately absorb, not for pretending real failures didn't happen.

**Do NOT overcorrect:**
- Don't delete error *handling* — change the *contract* so handling is
  unnecessary. If you can't legitimately redefine the case as normal, the error
  stays.
- Don't fight Python's EAFP. Idiomatic `try/except` as control flow (e.g., around
  a dict access) is fine and is not the target. The target is an *API* that raises
  for a condition it could have defined as ordinary, forcing every caller to cope.
- In gradually-typed Python you can't lean on the compiler to prove a value is
  always present, so when you remove an `Optional`, make sure the function really
  is total — otherwise you've moved a visible check into a hidden crash.

**Related flags:** Complexity Exported to Callers, Nonobvious Code.

---

## 12. Comment Repeats Code
*Principle: comments should describe what's not obvious from the code.*

**Symptom you'll see:** A comment that restates exactly what the adjacent code
already says. `# increment count` over `count += 1`; a docstring that just respells
the function name; `# loop over users` over `for user in users`.

**Why it's complexity:** It adds reading volume and maintenance burden with zero
information, and it trains readers to ignore comments (so the *useful* ones get
skipped too).

**The refactor:** Delete it, or replace it with the information the code *can't*
express — the *why* (intent, rationale), units and ranges, invariants and
preconditions, what a value means, or a warning about a non-obvious consequence.
A good comment captures design knowledge that has no place to live in the code
itself.

**Do NOT overcorrect:** "Delete redundant comments" is not "delete comments."
Ousterhout argues *for* comments that carry non-obvious information, especially on
interfaces and on what member variables represent — stripping those to chase
"self-documenting code" loses real design intent. Cut redundancy, keep meaning.

**Related flags:** Implementation Detail Contaminates Interface, Nonobvious Code.

---

## 13. Implementation Detail Contaminates Interface
*Principle: separate interface comments from implementation comments.*

**Symptom you'll see:** A docstring meant for *callers* describing *how the thing
works inside*. In Python: a function/method/class docstring that explains the
algorithm, internal data structures, or which private helpers it calls — details a
user of the interface neither needs nor should depend on.

**Why it's complexity:** It leaks implementation into the interface, creating a
dependency (callers may rely on stated internals) and burdening every reader of the
interface with details that aren't theirs. It also rots: the internals change, the
interface doc lies.

**The refactor:** The docstring (interface comment) describes *what* the thing does
and *how to use it* — the abstraction, the contract, parameters, returns, errors,
and any non-obvious usage notes. Implementation comments — the *how* and *why* of
the internals — live *inside* the body, where implementers read them.

**Do NOT overcorrect:** Some "implementation-sounding" facts are genuinely part of
the contract (e.g., "this is O(n²), don't call it on large inputs," or "not
thread-safe"). Those belong in the interface comment because callers must know
them. The line is *need-to-know-as-a-user*, not *mentions-internals*.

**Related flags:** Comment Repeats Code, Overexposure.

---

## 14. Hard to Describe
*Principle: difficulty of description is a design smell.* One of the most
underrated tells in the book — treat it as structural, not cosmetic.

**Symptom you'll see:** To document a variable or method *completely and
precisely*, the comment has to be long, full of caveats, or cover many cases
("returns the user, unless X, in which case the group, unless Y, in which case
`None`, and also mutates Z as a side effect"). The struggle to write a crisp
docstring is the signal.

**Why it's complexity:** If the *description* is complicated, the *thing* is
complicated — it probably has a fuzzy abstraction, too many responsibilities, or
hidden side effects. The hard-to-write comment is your design feedback, surfacing
early.

**The refactor:** Don't paper over it with a longer comment — simplify the thing
until it's easy to describe. Split mixed responsibilities; remove the side effect;
make the return total; narrow the contract. When the description becomes short and
precise, the design has improved.

**Do NOT overcorrect:** Not every long comment means bad design — genuinely
intricate domains (a subtle algorithm, a hairy external protocol) can need real
explanation, and that's essential, not incidental, complexity. The smell is
specifically when the *length comes from the design being muddled*, not from the
problem being hard.

**Related flags:** Shallow Module, Conjoined Methods, Hard to Pick a Name.

---

## 15. Vague Name
*Principle: choosing names.* Keep this on a tight leash — only when a name
genuinely *misleads* or *hides information*, not merely "could be nicer."

**Symptom you'll see:** A name so generic it carries little information: `data`,
`obj`, `info`, `tmp`, `val`, `result`, `do_it`, `handle`. Worse: a name that's
actively misleading (a `count` that's really an index; a `users` that's actually a
single user) or that collides in meaning with another variable so the two get
confused.

**Why it's complexity:** Obscurity — the reader must go read other code to learn
what the thing actually is, and a misleading name plants a wrong assumption that
becomes a bug (unknown unknowns).

**The refactor:** Name it for what it *is*, precisely and consistently: the unit,
the role, the entity. `elapsed_ms` not `time`; `selected_user` not `data`. Use the
same name for the same concept across the codebase so readers can rely on it.

**Do NOT overcorrect:** Don't bikeshed adequate names into "perfect" ones, and
don't make names so long they hurt readability. Short, conventional names for
short-lived locals (`i` in a tight loop) are fine. Flag names that *mislead or
obscure*; leave names that are merely improvable.

**Related flags:** Nonobvious Code, Hard to Pick a Name.

---

## 16. Hard to Pick a Name
*Principle: difficulty of naming is a design smell.*

**Symptom you'll see:** You (or the author) genuinely struggle to find a precise,
intuitive name for a variable, method, or class — every candidate feels partial or
wrong, or the honest name is "and"-shaped (`process_and_save_and_notify`).

**Why it's complexity:** Naming trouble usually means the entity has a fuzzy or
*compound* responsibility — it does more than one thing, or its concept isn't clean
— which is a design problem, not a vocabulary problem.

**The refactor:** Use the difficulty as feedback. If the honest name has "and" in
it, that's a hint to split. If the concept is fuzzy, sharpen the responsibility
until a clean name presents itself. The name getting easy is a sign the design got
better.

**Do NOT overcorrect:** Occasionally a thing is well-formed but just lives in a
domain with no established term; inventing a reasonable name and defining it in a
comment is fine. Don't restructure clean code purely because the *word* is
awkward — restructure when the awkwardness reflects a muddled responsibility.

**Related flags:** Hard to Describe, Shallow Module.

---

## 17. Nonobvious Code
*Principle: code should be obvious.* The catch-all for obscurity.

**Symptom you'll see:** A reader can't quickly understand what a piece of code does
or *means*, and has to work hard or guess. Tells in Python: clever one-liners and
dense comprehensions that pack several steps; reliance on non-obvious truthiness or
implicit conversions; behavior that depends on a side effect or global state the
reader can't see locally; results that depend on subtle ordering. The practical
test: *would a competent developer guess wrong about what this does on first read?*

**Why it's complexity:** Obscurity directly — high cognitive load, and when the
reader guesses wrong, unknown-unknown bugs. "Obvious" code is code where a reader
can make a correct quick guess about behavior and about what a change requires,
without reading everything.

**The refactor:** Make the obvious reading the correct one. Often that's better
names and types; sometimes a short comment stating the non-obvious *why*; sometimes
unpacking a dense expression into named steps; sometimes removing the hidden
dependency (pass it explicitly) so behavior is local. Prefer making the code
clearer over explaining unclear code.

**Do NOT overcorrect:** "Obvious" is relative to a competent reader of this
codebase, not the most junior possible reader — don't unroll idiomatic, widely-
understood constructs into verbose longhand in the name of obviousness. And a
genuinely subtle algorithm may be irreducibly non-obvious; there, a clear comment
explaining *why* is the right tool, not a forced rewrite.

**Related flags:** Vague Name, Conjoined Methods, Hard to Describe.
