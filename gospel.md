# Software Design Principles

## The Core

Four principles. Most specific guidance derives from these.

### 1. Local Reasoning

You should be able to understand what code does by looking at it, without tracing through indirection, global state, or runtime dispatch.

**Therefore:**
- Dependency rejection over dependency injection. DI containers mean the call site doesn't tell you what's called. Instead: pass values in, get values out. The shell decides where values come from; the core doesn't know or care.
- Functional core, imperative shell. Effects (IO, mutation, exceptions) break local reasoning because their consequences are elsewhere. Quarantine them at the edges.
- No "interface for one implementation." The problem isn't the indirection per se; it's that control flow becomes non-local. Compute a description of what to do, then do it—don't call out to a pluggable dependency.
- No framework brain. Frameworks invert control: you write hooks, the framework calls them. This makes control flow non-local. Most code is not a framework and shouldn't be structured like one.
- No magic. Reflection, implicit conversions, runtime code generation—these make behaviour invisible at the point of use.
- Explicit over implicit, always.

### 2. Have the Machine Enforce Invariants

The type system is a proof assistant. Make the compiler verify properties instead of relying on discipline or runtime checks.

**Therefore:**
- Make illegal states unrepresentable. If two fields can't both be Some, use a DU, not two options. The compiler enforces the invariant.
- Parse, don't validate. At the boundary, transform unstructured input into types that are correct by construction. Interior code receives proof, not promises.
- No stringly typing. Structured values get structured types.
- No primitive obsession. An email address is not a string. A user ID is not an int. Wrap them; the cost is near-zero and the compiler catches misuse.
- Phantom types and measure types. `UserId` vs `PostId`. Metres vs pixels. The compiler distinguishes them; humans don't have to.
- Expose genericity. If a container is generic internally, expose that in the API. Hiding it behind `obj` and casting back discards proof the type system could provide.

"Hard to misuse" is much more important than "easy to use". Iteration toward correct usage is cheap, especially in the age of coding agents. Undetected misuse is expensive. Prefer APIs where wrong usage is a compile error, even if correct usage requires more ceremony.

### 3. Small Orthogonal Core

A good system has a small set of primitives whose interactions are fully specifiable—ideally formally, at minimum in your head. Everything else is sugar that desugars to the core.

The platonic ideal: primitives that correspond to actual mathematical objects. When you find these, you get corollaries for free—extensions and compositions you never anticipated but which fall out of the structure. Knuth and Plass's boxes-glue-penalties model for line breaking is a famous example: careful choice of primitives "solved many other problems as a free bonus." This isn't always achievable—some domains are genuinely messy—but it's worth holding in mind during architecture.

**Therefore:**
- No speculative generality. Abstractions must earn their place by simplifying the composition story. If you can't explain how a feature composes with every other feature, the design is wrong. When a real need arises, refactor. Most of the API is sugar anyway; if the core was well-chosen, refactoring the desugaring is probably possible without breaking the surface.
- Composition over inheritance. Inheritance creates complex interaction rules (fragile base class, diamond problems, LSP violations). Functions and data compose simply.
- Be suspicious of any design where you can't enumerate the primitive operations and their laws.

### 4. Leverage Compute

You can run code. Use the machine to search and verify rather than relying on complex human reasoning.

**Therefore:**
- Property-based testing over example-based. Individual cases prove little. Find the invariant: "for all valid inputs, P holds." Generate thousands of cases.
- Reference implementations. For complex algorithms, write a naive correct version. Property: fast implementation ≡ slow implementation.
- Search for edge cases. Don't hand-craft inputs triggering branches unless it's really obvious or you already have the motivating example. Write a predicate, generate until you find matches. You have compute.
- Compute over cognition. "What input triggers condition X?" Reason, but also search. "What parameters make this converge?" Derive, but also systematically sweep. Automated search saves effort.
- Use tracing liberally. Knowing for certain what the program did is better than guessing. You can generate lots of telemetry in debug mode that's entirely compiled out of release mode.

---

## Derived Practices

These don't introduce new principles; they're instantiations of the above.

**Performance consciousness** (from 1 + 3): Not premature optimisation, but not closing doors. Ask: "could this be made fast in principle?" If the API forces allocation in a hot path, the design may be wrong. O(n²) when O(n) is obvious is just sloppy. *Measure, don't guess*—you can run benchmarks (principle 4).

**Debug asserts for pre/postconditions** (from 2 + 4): Types can't encode everything. Asserts fill the gap: executable specifications, cheap runtime proof. Often these can be entirely compiled out in release mode.

---

## Advice for coding agents

These are about how you should work, given the above:

- **You can run code.** Test hypotheses. Search parameter spaces. Generate examples. Don't just reason. This is your comparative advantage.
- **Propose the minimal thing...** Resist building abstractions before they're needed. Wait for the pattern to clarify.
- **... but don't be afraid of large mechanical changes.** A small change to the core propagates widely to consumers. The compiler will help you get it right.
- **If the type system fights you, the design may be wrong.** Step back before reaching for casts, reflection, or `obj`.
- **When uncertain, ask.** Don't produce 200 lines in the wrong direction. A quick question is cheaper.
- **Derive from principles, don't pattern-match on practices.** If you encounter a novel situation, reason from (1)–(4). Don't cargo-cult. Time spent getting the primitive design right is never wasted.
