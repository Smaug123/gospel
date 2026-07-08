# Software Design Principles

Some roughly orthogonal principles; most specific guidance derives from them.

### Local reasoning

You should be able to understand what code does by looking at it, without tracing through indirection, global state, or runtime dispatch.

**Therefore:**
- Dependency rejection ("don't execute effects; instead return effect descriptions") over dependency injection. DI containers or injected interfaces mean the call site doesn't tell you what's called. Instead: pass values in, get values out. The shell decides where effects come from; the core doesn't know or care.
- Functional core, imperative shell. Effects (IO, mutation, exceptions) break local reasoning because their consequences are elsewhere. Quarantine them at the edges.
- No "interface for one implementation." The problem isn't the indirection per se; it's that control flow becomes non-local. Compute a description of what to do, then do it; try not to call out to a pluggable dependency.
- No framework brain. Frameworks invert control: you write hooks, the framework calls them. This makes control flow non-local. Most code is not a framework and shouldn't be structured like one.
- No magic. Reflection, implicit conversions, and runtime code generation all make behaviour invisible at the point of use.
- Explicit over implicit, always.
- Error results over exceptions unless necessary, e.g. when interfacing with a framework or stdlib that was built to require exceptions. Exceptions are highly nonlocal dynamic `goto` statements.
- Immutable over mutable, except perhaps in tightly-constrained temporary local scopes, because global mutable state may change for nonlocal reasons. Even *spatially local* mutable state has temporally nonlocal impact if it's long-lived.

### Have the machine enforce invariants

Don't rely on discipline or documentation.
Make the machine verify properties, at compile time where possible, or at runtime where necessary.

The type system is the first line of defence: a proof assistant that catches errors before code runs.
Use runtime assertions for what types can't catch (pre/postconditions that aren't easily expressed in types, and liberally to document/assert pre- and postconditions).

**Therefore:**
- Make illegal states unrepresentable. If two fields can't both be Some, use a DU, not two options.
- Parse, don't validate. At the boundary, transform unstructured input into types that are correct by construction. Interior code receives proof, not promises.
- No stringly typing. Structured values get structured types.
- No primitive obsession. An email address is not a string. A user ID is not an int. Wrap them.
- Phantom types and measure types. `UserId` vs `PostId`. Metres vs pixels.
- Expose genericity. If a container is generic internally, expose that in the API. Hiding it behind `obj` and casting back discards proof the type system could provide.
- Assert pre/postconditions. When types can't express an invariant, assert it. `Debug.Assert(index >= 0)` is documentation that executes. Fail fast and loud.

"Hard to misuse" is much more important than "easy to use".
Iteration toward correct usage is cheap, especially in the age of coding agents; undetected misuse is expensive.
Prefer APIs where wrong usage is a compile error, even if this means correct usage requires more ceremony.

### Small orthogonal core

A good system has a small set of primitives whose interactions are fully specifiable (ideally formally, but at minimum in your head).
Everything else is sugar that desugars to the core.

The platonic ideal is primitives that correspond to actual mathematical objects.
When you find these, you get corollaries for free: extensions and compositions you never anticipated but which fall out of the structure.
Knuth/Plass's boxes-glue-penalties model for line breaking is a great example: careful choice of primitives "solved many other problems as a free bonus."
This isn't always achievable - some domains just are messy - but it's worth holding in mind during architecture.

**Therefore:**
- No speculative generality. Abstractions must earn their place by simplifying the composition story. If you can't explain how a feature composes with every other feature, the design is wrong. When a real need arises, refactor. Most of the API is sugar anyway; if the core was well-chosen, refactoring the desugaring is probably possible without breaking the surface.
- Composition over inheritance. Inheritance creates complex interaction rules (diamond inheritance, LSP violations). Functions and data compose simply.
- Be suspicious of any design where you can't enumerate the primitive operations and their laws.

### Leverage compute

You can run code.
Use the machine to search and verify rather than relying on complex intelligent reasoning.

**Therefore:**
- Property-based testing over example-based. Individual cases prove little; instead, find and test the invariant: "for all valid inputs, P holds." There's a `property-based-testing` agent skill for much more advice.
- Consider writing a reference implementation: for complex algorithms, write a naive correct version. Property: fast implementation ≡ slow implementation.
- This encourages designing systems that *do* admit reference implementations and properties, e.g. modelling the system as an explicit state machine, or pushing effects out to the edges.
- *Search* for edge cases. Don't hand-craft inputs triggering branches unless it's really obvious or you already have the motivating example. Write a predicate, generate until you find matches. You have compute.
- Compute over cognition. "What input triggers condition X?" Reason, but also search. "What parameters make this converge?" Derive, but also systematically sweep. Automated search saves effort.
- Use tracing liberally. Knowing for certain what the program did is better than guessing. You can generate lots of telemetry in debug mode that's entirely compiled out of release mode. You can create entire programs to aid debugging (e.g. "dump my internal state" endpoints in a service, or custom debugging endpoints causing it to step through a request).
- When fixing a bug, always write the failing test, observe it fail, then fix the bug. Don't think about whether test captures bug; use the computer to *verify* it does.

### Correctness over availability

For almost all systems you'll ever work with, producing wrong results is worse than going down.
When you can't guarantee correctness, stop.

Make the correctness envelope explicit, including under partial failure. What does this component guarantee when the database is unreachable? When the cache is cold? When a dependency times out? If you can state the guarantee, you can operate in that state. If you can't, crash.

Degraded operation is fine when you can characterize its correctness. A cache miss is usually correct—you just pay latency. Serving stale data is correct if staleness is bounded and documented. Retrying a failed write is correct if the operation is idempotent. The question is always: "can I specify what this component guarantees in this state?" When the answer is no, crash. Corruption is never acceptable; downtime sometimes is.

**Therefore:**
- State the guarantees. For each component, document what it promises under normal operation and under each failure mode it handles. If you can't articulate the guarantee, you don't understand the system.
- Design for resurrection, not immortality. Components will die—from crashes, deployments, or deliberate restarts. Make death cheap: externalize state, use supervisors, make operations idempotent so replay is safe. Don't invest in keeping processes alive; invest in making rebirth trivial.
- Fail fast, fail visibly. When invariants are violated beyond recovery, crash immediately. Don't limp along hoping things improve. A clear crash with good diagnostics is easier to fix than silent corruption discovered later.
- Bound uncertainty. If you serve stale data, bound how stale. If you retry, bound how many times. If you queue, bound how deep. Unbounded degradation becomes uncharacterizable, which means you can't state guarantees, which means you should have crashed.

---

## Case studies

Examples of reasoning about new situations from the above.

### Example: performance consciousness

Local reasoning and orthogonality (principles 1 and 3) mean you should write code such that performance can be adjusted later without rippling changes. If an API forces allocation in a hot path, or couples callers to an O(n²) algorithm when O(n) is possible, the *design* is constraining future optimisation. That's a structural problem, not a premature-optimisation concern.

But no speculative generality (principle 3) means you shouldn't hyper-optimise now for hypothetical future load. Write the clean version first.

Leverage compute (principle 4): benchmark before optimising. Measure rather than guessing where the time goes. You have a profiler; use it. Optimise what the data says matters.

### Example: when should I introduce a boundary?

Boundaries (code module, ser/de boundary, service deployment) can enforce constraints mechanically (principle 2) and aid local reasoning by limiting what you must hold in your head (principle 1). But introducing a boundary may be speculative generality (principle 3). How do you decide?

A boundary is justified when the constraint it enforces is worth more than the cost of the boundary itself. Both sides of this inequality require empirical input.

**Input 1: What does the boundary cost?**

- *Type-system module boundaries* are cheap. The compiler checks the interface; there's no runtime overhead; refactoring across the boundary is mechanical. Cost ≈ the cognitive overhead of one more named thing.
- *Serialisation/IPC boundaries* are expensive. You pay marshalling overhead, lose type safety at the wire, must version the protocol, and can't refactor across the boundary without coordinating. Cost ≈ ongoing maintenance burden.
- *Deployment boundaries* are very expensive. Independently-deployed components must maintain backward compatibility, need separate CI/CD, and failures become partial. Cost ≈ distributed systems complexity.

**Input 2: What constraint does the boundary enforce, and do I want it?**

- "These two things shouldn't know each other's internals" is extremely common, and easily satisfied with a module boundary.
- "These two things should be deployable independently" is only valuable if you actually deploy them independently. If you don't, you're paying for a constraint you never exercise.
- "Consumers in other languages need access" is a case where an IPC boundary is necessary (e.g. Pulumi's language-agnostic model).

**Applying the framework:**

Given typical costs, the reasoning usually goes:
- Module boundaries: low bar. Split when local reasoning demands it, e.g. when a module exceeds what you can hold in your head while reading any one function.
- Serialisation boundaries: need justification. Are there actual consumers that vary independently? If not, it's speculative generality.
- Deployment boundaries: need strong justification. Do you have evidence you'll deploy these on different cadences? If not, you're buying distributed-systems problems for nothing.

Different cost observations would yield different conclusions. In an environment where serialisation is nearly free (say, a language with automatic derivation and schema evolution), the bar for IPC boundaries lowers. In a monorepo with atomic deploys, deployment boundaries cost less. Plug in your actual costs.

### Example: data descriptions over behavioral abstractions

When modelling a set of operations (commands, events, effects), should you represent them as data like a discriminated union you later interpret, or as behaviour (objects/closures that execute themselves)?

Local reasoning (principle 1) gives the strongest signal. A DU is inspectable at the point of use: you can see every variant, read the fields, and the compiler checks exhaustive handling. A closure or strategy object is opaque: `Func<Request, Response>` tells you nothing about what it will do. An `ICommand` with an `Execute` method could do anything; a `Command` DU with variants like `CreateUser of name * email` tells you exactly what it describes. The call site that *constructs* the description is readable; the call site that *interprets* it is a match expression with every case visible.

Machine-enforced invariants (principle 2) reinforce this. You can pattern match on a data description and assert structural properties: "no `Delete` command should appear before `Create` for the same entity." You can write exhaustive matches and the compiler catches missing cases. With closures, the type system sees only the signature, and intent is invisible: you can't write a debug assert that inspects what a closure *will do*.
Additionally, with data, you can write property tests: "for all valid command sequences, the interpreter preserves invariant P". You can't do this with closures.

Small orthogonal core (principle 3) applies directly. A DU's constructors *are* the primitive operations. You can enumerate them, specify their laws, and verify that they compose. This is exactly the "small set of primitives whose interactions are fully specifiable" aspiration. Behavioural abstractions hide the operation set behind a polymorphic interface where you can't enumerate what implementations exist or how they interact.

The pattern this reasoning produces is the interpreter pattern: construct an inert data structure describing *what* to do, then write a separate function that *does* it. This is what principle 1 already gestures at ("compute a description of what to do, then do it") in the context of dependency rejection, but the principle is broader. Whenever you reach for a strategy object, a visitor, a command pattern with an `Execute` method, or a closure passed as a parameter, ask whether a DU + match expression would be simpler. Usually it is: more inspectable, more testable, and the compiler helps more.

The exception is when the operation set is genuinely open, with third parties adding new variants without modifying the core (the expression problem).
In my experience, this is pretty rare.

### Example: concurrency models

Concurrency destroys local reasoning (principle 1). When two threads share mutable state, understanding one requires understanding all others that might touch that state, plus the interleaving. Locks recover mutual exclusion but not local reasoning: you must still trace which locks protect which state, and deadlock potential is a global property of the lock graph.

So the first question is: do you need concurrency at all? Introducing parallelism "because the system might need throughput" is speculative generality (principle 3). Measure first (principle 4). Many systems that feel concurrent (handling web requests, processing event streams) don't require concurrency *within* a component. A single-threaded event loop can saturate a network link.

When you do need concurrency, bound it (principle 5). An unbounded `Task.WhenAll` over an unbounded collection means you can't state what the system does under load: memory, connections, CPU all become uncharacterisable. A bounded worker pool pulling from a bounded channel is the concurrency equivalent of "bound uncertainty": you can state the maximum resource consumption, the maximum queue depth, the backpressure behaviour.

Following this reasoning to its conclusion: if each component should avoid internal concurrency, but the system has multiple independent concerns, each concern gets its own serialised work queue. Messages arrive in a bounded channel; the component processes them one at a time; it communicates with other components by sending messages to their channels. This is the actor model from first principles. Concurrency exists *between* actors (they run independently) but not *within* them (each processes messages serially). Local reasoning is preserved inside each actor. The bounded mailbox provides backpressure and characterisable resource usage.

### Example: migrations and deletion

A half-migrated system is one where two versions of the truth coexist: the old design and the new design, both partially alive. Every change to the system must now consider both states. This directly erodes local reasoning (principle 1): you can't understand a component without knowing which migration state it's in. It also resists orthogonality (principle 3): the old and new paths overlap, so the system has more surface area than capability.

**Therefore: complete your migrations.** A half-finished migration is a tax on every future change. Prioritise finishing it over starting new work, because the longer it lingers, the more code accumulates that must account for both states.

**Delete aggressively.** Dead code, vestigial paths, and unused abstractions are anti-orthogonal: they expand what you must read without expanding what the system does. They resist local reasoning because you can't tell from the code alone whether a path is dead or alive. The bug-free code is the code that doesn't exist. When in doubt about whether something is still needed, the answer is usually to delete it; you can always get it back from Git later.

**Migration strategy follows from local reasoning.** Prefer whichever approach minimises the time and scope during which two versions of the truth coexist:

1. *Stop the world.* If you can take the system down, migrate atomically, and bring it back up, do that. The half-migrated state has zero duration. This is the easiest to reason about locally.
2. *Isolate and restart.* If the system can't go down entirely, take the migrating component down in isolation while maintaining a stable boundary for its dependents. This is "design for resurrection" applied to migration: components that are cheap to kill and restart are cheap to migrate.
3. *Compatibility shim at the boundary.* When neither of the above is possible, introduce a translation layer at the component's boundary that accepts both old and new inputs and normalises to the current internal representation. The interior only ever sees the latest version. This is "parse, don't validate" (principle 2) applied to versioning.

**Who controls the inputs determines when you can delete.** If you control the consumers, migrate them promptly, then delete the old path. The cost of the migration is internal and bounded.

If you don't control the consumers, then inputs that were once legal must remain legal. This is principle 5 applied to boundaries: rejecting previously-valid input is a correctness violation from the consumer's perspective. Here a v1→v2→v3 migration chain at the edge is appropriate. Each step is small; the chain is a sequence of parsers that normalise any historical format into the current internal representation. The interior stays clean; the edge absorbs history.

Even these chains aren't sacred. If the cost of maintaining a compatibility step exceeds the cost of dropping it (e.g. because it's blocking an important internal change, or because the old format has no remaining consumers you can find) then weigh deletion. But the default for external contracts is retention, because the cost of breakage is borne by others and is hard to observe.

---

## Advice for coding agents

These are about how you should work, given the above:

- **You can run code.** Test hypotheses. Search parameter spaces. Generate examples. Don't just reason. This is your comparative advantage.
- **Propose the minimal thing...** Resist building abstractions before they're needed. Wait for the pattern to clarify.
- **... but don't be afraid of large mechanical changes.** A small change to the core propagates widely to consumers. The compiler will help you get it right.
- **If the type system fights you, the design may be wrong.** Step back before reaching for casts, reflection, or `obj`.
- **When uncertain, ask.** Don't produce 200 lines in the wrong direction. A quick question is cheaper.
- **Derive from principles, don't pattern-match on practices.** If you encounter a novel situation, reason from the core principles. Don't cargo-cult; many of the practices you were trained with don't play to your strengths. Time spent getting the primitive design right is never wasted.
- **Always write the tests first.** Comprehensive property-based tests are a massive force multiplier: when fixing a bug or implementing a feature, write the tests that assert all the desired properties. Do this before implementing: write and observe the failing test before fixing the bug; write the tests that exercise the API before implementing the feature. Be very sceptical of changes to existing tests.
- I'll say this again because it is very important: **Write the property-based tests**. I can't emphasise this enough. It saves *so much effort*. Many times, I've seen Claude instances go back and forth for hours with a reviewer, when simply writing a small correctness oracle would have caught dozens of bugs that the reviewer had to find.
