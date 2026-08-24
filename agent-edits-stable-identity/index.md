**TL;DR** — Text is a useful projection of a program, but it is a fragile
address space for autonomous edits. In SEMAPRAX, a public declaration has a
persistent ID that is separate from its display name. A semantic patch selects
that ID, names the exact graph revision it expects, and is rejected before
mutation if the source has moved on. The current prototype also offers bounded
impact and independently replayable evidence, while deliberately stopping
short of claiming signatures, general proofs, or repository-wide analysis.

This is an implementation case study from Wavect GmbH, the team building
[SEMAPRAX](https://wavect.io/semaprax/), an experimental Apache-2.0 pre-alpha
language and compiler. The implementation discussed here is public in the
[SEMAPRAX repository](https://github.com/wavect/semaprax).

---

## The Addressing Problem in Agentic Coding

Most editing tools ultimately say: replace these bytes, lines, or syntax-tree
positions. That works well when a human has just inspected the same revision.
It becomes less reliable when an agent plans a change, another actor edits the
file, and the first agent later tries to apply its plan.

Even without concurrency, names are awkward identities. Consider a function
called `add`. A repository can contain many functions with that spelling. The
name may change during a refactor. Moving a declaration to another file can
also invalidate a path-based selector even when its role stays the same.

The compiler therefore needs to answer two separate questions:

1. Which declaration does the edit mean?
2. Which exact program state did the edit inspect?

SEMAPRAX answers them with persistent declaration IDs and revision-bound
semantic patches.

## Separate Identity From Spelling

A small SEMAPRAX program can declare both a human name and an explicit ID:

```semaprax
@id("math.add")
fn add(left: i64, right: i64) -> i64 {
    left + right
}
```

Here, `add` is the source-level name. `math.add` is the persistent identity.
The name can change while callers, graph queries, and target projections keep
referring to the same declaration.

That distinction reaches the compiler's resolved representation. Stable IDs
are validated for uniqueness, and automatic name-derived identities produce a
diagnostic because they cannot survive a rename. SEMAPRAX does admit a narrow
repair that assigns a persistent ID to one eligible function, but classifies
that operation honestly as a breaking identity rebase—not as a harmless
rename.

This is the first useful design lesson: do not hide identity migration inside
a cosmetic refactor. Identity establishment and display-name changes have
different compatibility consequences.

## Bind the Edit to a Canonical Revision

Stable selection is necessary, but it does not solve stale plans. An edit also
needs to state which source state it was derived from.

SEMAPRAX computes its graph revision from canonical source, using a
domain-separated SHA-256 digest. Formatting-only differences do not invalidate
a transaction, but a semantic source change produces a different revision.
The token is content addressing for stale-base detection. It is not presented
as a signature or a message-authentication code.

A minimal rename patch looks like this:

```text
base sha256:<expected-revision>
rename math.add to sum
require no-new-effects
```

The important parts are visible in the patch itself:

- `math.add` selects meaning rather than a line number.
- `base` records the revision the agent inspected.
- `require no-new-effects` adds a checked constraint.

During preflight, the compiler parses the current source, recomputes its graph
revision, and compares it with the patch base. A mismatch returns a stable
diagnostic before candidate construction or source mutation. The caller must
re-read the new graph and regenerate its proposal.

That is much safer than trying to make an old textual diff fit a new file.

## Preflight Is the Semantic Firewall

After the base matches, SEMAPRAX does not immediately rewrite text. Its
single-file patch path first builds a candidate and re-runs the relevant
compiler checks.

For the currently admitted patch operations, preflight verifies facts such as:

- the stable selector resolves exactly;
- the new spelling is a valid identifier;
- the declaration has an explicit persistent ID;
- resolved HIR before and after the edit is valid;
- the semantic delta stays inside the operation's
  admitted shape;
- authored requirements such as no new effects hold.

Only then does the compiler canonically format the candidate and calculate its
candidate revision. This makes the proposed result an inspectable compiler
artifact rather than an assumed consequence of text replacement.

The public surface is deliberately smaller than the ambition. Patch v1 and v2
currently admit stable-ID renames, selected member and case renames, a bounded
generic call type-argument replacement, and requirements such as no new
effects. Patch v3 admits one tightly constrained persistent-ID assignment
repair. This is not a general-purpose semantic transformation language yet.

## Preview Consequences Without Granting Authority

An agent often needs to know more than whether a patch type-checks. It needs to
know what the change touches.

SEMAPRAX's current Semantic Impact path runs the same pure preflight used by
apply, then emits a deterministic report containing the base and candidate
revisions, exact patch digest, operations, changes, source consumers, and an
optional bounded reverse-call closure.

The boundary matters. Today this is single-file analysis over the admitted
Patch v1/v2 domain. Renames report source-projection consumers but do not
pretend that every textual rename is a behavioral change. Reverse traversal is
call-graph impact, not universal type, capability, schema, migration, test, or
repository analysis.

Preview is also read-only. It creates no lock, staging file, rename, or source
write. Before returning, it rechecks the authenticated source snapshot so a
concurrent replacement fails closed instead of producing a report for mixed
states.

This separation is valuable for agent tooling:

```text
inspect -> propose -> impact -> review -> apply
```

Knowledge-producing operations do not silently acquire mutation authority.

## Replay Evidence, Then Commit

SEMAPRAX also has a bounded Semantic Patch Evidence path. It can render a
canonical evidence capsule for an admitted patch, independently rebuild the
supporting review and impact facts, and require exact replay before the
evidence-gated apply route prepares a candidate for commit.

The phrase "proof-carrying" is easy to overstate, so the implementation carries
explicit nonclaims. The capsule is not a signature, authenticated provenance,
human approval, target execution, a reusable authorization token, or a general
proof system. Possessing the JSON does not grant commit authority.

On apply, the implementation acquires its ordinary single-file lock,
authenticates the source, performs replay, and only then creates a staging
file. Before the final atomic rename it rechecks that the original source and
the staged bytes still match the authenticated state. A stale or externally
replaced source is preserved rather than overwritten.

This gives the narrow route a useful failure rule:

> If exact meaning cannot be reconstructed, do not mutate the source.

It does not imply filesystem-wide, Git-wide, or editor-wide atomicity. The
claim is scoped to the compiler's admitted single-file transaction.

## What We Learned Building It in Rust

The hardest part was not parsing the patch grammar. It was keeping every layer
honest about authority and evidence.

Four implementation choices proved especially useful:

1. **One preflight path.** Preview and apply share the same candidate-building
   logic, reducing the risk that an accepted preview differs from commit.
2. **Closed schemas.** Patch and evidence versions admit explicit operations
   and reject unknown structure instead of guessing intent.
3. **Deterministic serialization.** Canonical source, graph JSON, reports, and
   evidence make exact replay and regression fixtures practical.
4. **Nonclaims beside claims.** Evidence artifacts state what they do not
   establish, making narrow experimental guarantees harder to market as broad
   safety.

Rust helps with owned snapshots and explicit result paths, but the language
alone does not create these properties. They come from protocol design:
bounded inputs, closed domains, shared validation, deterministic output, and
rechecks at authority boundaries.

## A Practical Checklist

If you are adding agent-driven edits to an existing compiler or code tool,
start with these questions:

- Does a selector identify meaning or merely a location?
- Can the identity survive a rename or movement?
- Does every proposal bind the state it inspected?
- Is stale input rejected before mutation?
- Do preview and apply share candidate construction?
- Can impact queries be bounded without hiding truncation?
- Is evidence distinct from authorization?
- Are atomicity claims scoped to the actual commit boundary?

You do not need a new programming language to adopt those ideas. A compiler
plugin or repository service can introduce persistent IDs and revision-bound
operations incrementally. The important move is to stop treating a successful
text replacement as sufficient evidence that the intended semantic change
occurred.

SEMAPRAX remains pre-alpha research, and its schemas and language surface will
change. Its current value is as an executable experiment: give agent edits
stable semantic addresses, make their assumptions explicit, and fail closed
when those assumptions no longer hold.
