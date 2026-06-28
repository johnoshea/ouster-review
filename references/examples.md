# Ousterhout Review — Worked Examples

These show the voice and, more importantly, the **structural-reshape move**: don't
just name a flag, show the *alternative interface* and why it's simpler. Each
example is deliberately small so the shape is visible; on a real subsystem the same
moves apply across modules. Match this register — blunt, specific, every finding
tied to a complexity cost, and always show the interface the fix produces.

---

## Example 1 — Shallow module + complexity exported to callers

**Before:**

```python
class ReportWriter:
    def __init__(self, path: str) -> None:
        self.path = path
        self._file = None
        self._wrote_header = False

    def open(self) -> None:
        self._file = open(self.path, "w")

    def write_header(self, columns: list[str]) -> None:
        self._file.write(",".join(columns) + "\n")
        self._wrote_header = True

    def write_row(self, values: list[str]) -> None:
        if not self._wrote_header:
            raise RuntimeError("call write_header first")
        self._file.write(",".join(values) + "\n")

    def close(self) -> None:
        self._file.close()


# every caller:
w = ReportWriter("out.csv")
w.open()
w.write_header(["name", "score"])
for r in rows:
    w.write_row([r.name, str(r.score)])
w.close()
```

**Review:**

This class is shallow, and it pushes its complexity onto every caller. Look at the
interface: five public methods, an ordering contract (`open` before writing,
`write_header` before `write_row`, `close` at the end) that the caller must
remember and that the class enforces with a runtime error, and manual resource
management. Now look at what it actually hides: joining a list with commas and
appending a newline. The interface is *more* complex than the implementation — the
definition of a shallow module — and the cost lands at every call site, multiplied,
with a class of bugs waiting in the ordering contract (a forgotten `close` leaks a
file; a missed `open` is an `AttributeError`). That's cognitive load and change
amplification for almost no abstraction in return.

The fix is to pull all of that down behind one deep call. The caller has rows and
wants a file; everything else is the module's job.

**Designed twice — the two interfaces side by side:**

```
Before (caller must know):          After (caller must know):
  - construct with a path             - write_csv_report(path, columns, rows)
  - call open()
  - call write_header(columns)
  - call write_row() per row
  - call close()
  - the required ordering
```

**After:**

```python
def write_csv_report(
    path: str, columns: list[str], rows: Iterable[Sequence[str]]
) -> None:
    """Write rows to `path` as CSV with the given header columns."""
    with open(path, "w", newline="") as f:
        writer = csv.writer(f)
        writer.writerow(columns)
        writer.writerows(rows)


# every caller:
write_csv_report("out.csv", ["name", "score"],
                 ([r.name, str(r.score)] for r in rows))
```

The ordering contract is gone (there's no order to get wrong), the file can't leak
(the `with` owns it), and the header-before-rows invariant is structural rather
than enforced at runtime. We also stopped hand-rolling CSV — which was its own
latent bug the moment a value contains a comma. The interface is now far simpler
than the implementation: a deep module. If callers genuinely need streaming writes
over a long-lived handle, the right shape is a context manager
(`with open_csv_report(path, columns) as w: w.write(row)`), which keeps the
file-can't-leak property while supporting incremental writes — still one concept
to learn, not five.

---

## Example 2 — Information leakage

**Before:**

```python
# ingest.py
def parse_event(line: str) -> dict:
    parts = line.split("|")
    return {"ts": parts[0], "user": parts[1], "action": parts[2]}

# export.py
def format_event(e: dict) -> str:
    return f"{e['ts']}|{e['user']}|{e['action']}"

# audit.py
def is_login(line: str) -> bool:
    return line.split("|")[2] == "login"
```

**Review:**

The event wire format — pipe-delimited, fields in the order ts/user/action — is a
single design decision that's currently leaked across three modules. `ingest`
knows the delimiter and field order to parse it, `export` knows them to write it,
and `audit` reaches past both to pull field index 2 straight out of a raw line.
That's the information-leakage red flag, and it's the most dangerous kind of
dependency because nothing connects these three files in the type system: change
the format — add a field, switch to JSON, reorder — and you must find and fix all
three, and `audit`'s `split("|")[2]` will silently return the wrong field rather
than fail. That's an unknown-unknown sitting in the codebase.

The decision needs exactly one owner. Put the format behind an `Event` type that
knows how to parse and serialize itself; everyone else depends on the *type*, not
the format.

**After:**

```python
# event.py  — the one place that knows the wire format
@dataclass(frozen=True)
class Event:
    ts: str
    user: str
    action: str

    @classmethod
    def parse(cls, line: str) -> "Event":
        ts, user, action = line.split("|")
        return cls(ts, user, action)

    def serialize(self) -> str:
        return f"{self.ts}|{self.user}|{self.action}"

# audit.py
def is_login(event: Event) -> bool:
    return event.action == "login"
```

Now the format lives in one module. `audit` asks `event.action` — a named field,
not a magic index — so reordering or reformatting can't silently corrupt it, and
switching the wire format to JSON touches `Event.parse`/`serialize` and nothing
else. Note the test for whether this was real leakage versus harmless shared
knowledge: the format is a *decision that will plausibly change*, so hiding it pays
off. If instead these three files had merely referenced, say, a fixed ISO-8601
timestamp standard that will never change, centralizing it would have added a
shallow indirection for no benefit — that's the line.

---

## Example 3 — Define errors out of existence (redesign, not swallowing)

**Before:**

```python
def get_discount(cart: Cart, code: str | None) -> Decimal:
    if code is None:
        raise NoCodeError()
    promo = lookup_promo(code)        # returns None if not found
    if promo is None:
        raise UnknownCodeError()
    if promo.min_total is not None and cart.total < promo.min_total:
        raise BelowMinimumError()
    return promo.amount

# every call site:
try:
    discount = get_discount(cart, code)
except (NoCodeError, UnknownCodeError, BelowMinimumError):
    discount = Decimal(0)
```

**Review:**

This API has defined three exceptions into existence, and every caller pays for it
with the same try/except that maps all three back to "no discount." That's the
special-cases-as-complexity smell: the branching exists only because of an API
design choice, and it's duplicated at every call site (change amplification), with
the risk that some future caller forgets one of the three and crashes on a perfectly
ordinary "code didn't apply" situation.

Step back and ask what the caller actually wants: *the discount, which is zero when
no valid code applies.* None of these three conditions is a failure — a missing
code, an unknown code, and a below-minimum cart are all just "the discount is
zero." So redefine the contract to make that the normal, total result, and the
special cases vanish.

**Important:** this works *because all three conditions are legitimately normal
outcomes.* I am not telling you to wrap real failures in a `return 0`. If
`lookup_promo` could raise on a database outage, that is a genuine fault and must
still propagate — defining errors out of existence is about eliminating
*manufactured* special cases by design, never about silencing real ones.

**After:**

```python
def get_discount(cart: Cart, code: str | None) -> Decimal:
    """The discount for `cart`; zero if no valid, applicable code is given."""
    if code is None:
        return Decimal(0)
    promo = lookup_promo(code)
    if promo is None or not promo.applies_to(cart):
        return Decimal(0)
    return promo.amount

# every call site:
discount = get_discount(cart, code)
```

The function is now *total* — it always returns a `Decimal` — so the `Optional`
mindset and the three-exception try/except disappear from every caller. We also
pushed the min-total rule down into `promo.applies_to(cart)`, where the promo's own
applicability logic belongs, instead of exporting it as a third exception type. One
caveat specific to gradually-typed Python: now that callers no longer check, the
function genuinely *must* always return a value — there's no compiler to catch a
path that accidentally falls through to `None`, so the totality has to be real, not
assumed.

---

## Example 4 — Pass-through layer

**Before:**

```python
class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

    def get(self, user_id: int) -> User:
        return self._repo.get(user_id)

    def save(self, user: User) -> None:
        self._repo.save(user)

    def delete(self, user_id: int) -> None:
        self._repo.delete(user_id)

    def all(self) -> list[User]:
        return self._repo.all()
```

**Review:**

Every method on `UserService` is a pass-through: it forwards one-to-one to
`UserRepository` with the same signature and adds nothing. This is a layer that
exists without doing any work. It's pure cost — one more class to learn, one more
file to open when tracing a call, and a maintenance tax because the two signatures
have to move in lockstep forever. The two layers express the *same* abstraction, so
the upper one isn't a different layer at all.

There are only two honest resolutions, and which one is right depends on a question
the code can't answer on its own:

- **If this layer has no real job,** delete it and let callers depend on
  `UserRepository` directly. The indirection isn't buying anything.
- **If a service layer is meant to exist** — to enforce invariants, coordinate
  multiple repositories, publish events, handle authorization — then give it that
  job. Right now it's an empty promise of a layer; make it earn its place.

What you should *not* do is keep it as-is "for symmetry" or "in case we need it
later" — that's a shallow layer justified by speculation, and YAGNI applies.

I need one piece of context to commit: **is this service a deliberate boundary?**
If `UserService` is the public seam other modules are supposed to code against
while the repository is free to churn underneath, then a thin forwarding layer is a
legitimate boundary, not a pass-through — keep it, and document that intent so the
next reader doesn't "clean it up." If it's just an out-of-habit layer between
callers and the repo, it should either gain responsibility or disappear. That
distinction — redundant echo versus real boundary — is the whole call here, and
it's exactly the kind of thing I won't bluff: tell me which it is.
