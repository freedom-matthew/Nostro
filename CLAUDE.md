# Nostro — project brief

## Why this project exists

I'm a C#/.NET engineer aiming to work in banking. I interviewed for a senior
architect role at Erste Group, got through three rounds, and lost to a more
experienced candidate. I'm now targeting a business/technical analyst role
there instead, as a route in.

This project has two jobs:

1. Build real domain literacy in ISO 20022 payment messaging — the vocabulary
   and rulebook of European banking.
2. Practise systems integration, which is the technical area I want to grow in.

The deliverable is both working code **and** the specification artifacts a bank
BA would produce alongside it (field mappings, rejection rules, message
catalogue). The code proves I can talk to engineers; the spec is the actual job.

I have roughly 2 hours a day.

## What Nostro does

Nostro plays the role of a **creditor bank** receiving interbank payment
instructions.

Another bank sends a `pacs.008` (FI-to-FI Customer Credit Transfer): *"pay
5000 EUR from our customer to your customer, here is the account, here is the
reference."*

Nostro decides whether to accept it, and answers with a `pacs.002` (Payment
Status Report) — `ACCP` if accepted, `RJCT` plus an ISO reason code if not.
The sending bank waits for that answer.

**One job: message in, accept-or-reject decision out.**

Checks that eventually need to happen:

- Is the message structurally valid against the XSD?
- Have I seen this transaction id before? (idempotency / duplicate detection)
- Does the creditor IBAN belong to an account I hold?
- Is the currency one I settle?
- Is the amount within scheme limits?

Each of these becomes its own test. None of them exist yet.

## Scope boundaries

**In scope, eventually:** `pacs.008` in, `pacs.002` out, one return path via
`pacs.004`. Schema validation. Canonical internal model distinct from the wire
format. Multiple transport hosts over one core.

**Out of scope:** actual settlement, ledgers, accounting, real bank
connectivity, SWIFT network integration, authentication, a UI.

## Decisions already made

**Name:** Nostro — a nostro account is the account your bank holds at another
bank. Domain term, signals literacy, short.

**Structure:** one solution, two projects plus tests.

```
Nostro/
  Nostro.slnx
  global.json           opts `dotnet test` into Microsoft Testing Platform
  Nostro.Core/          class library — the whole domain
  Nostro.Tests/         xUnit v3
    Samples/            real pacs.008 XML, copied to output
```

Both projects target `net10.0`. The solution uses the newer `.slnx` XML format,
which is what `dotnet new sln` emits on the .NET 10 SDK.

**Test runner:** xUnit v3 (`xunit.v3`), one package and nothing else. v3 test
projects are executables with their own built-in runner on Microsoft Testing
Platform, so `Microsoft.NET.Test.Sdk`, `xunit.runner.visualstudio` and
`coverlet.collector` are all gone — hence `<OutputType>Exe</OutputType>`. The
.NET 10 SDK refuses to run an MTP project through the old VSTest path, so
`global.json` carries `"test": { "runner": "Microsoft.Testing.Platform" }`.
Tests run from the CLI with `dotnet test`. Exit code 8 means "zero tests ran",
which is the correct answer until the first test exists.

Folders inside `Nostro.Core`, not more projects. No `src`/`tests` split at this
size.

**The one architectural commitment:** `Nostro.Core` is transport-agnostic. No
ASP.NET reference, no queue client, no `IConfiguration`. Anything it needs from
outside arrives as an interface it defines itself. Hosts reference Core; Core
references nothing of theirs.

This matters because interbank traffic doesn't arrive over REST — it arrives as
files from a SWIFT interface or off a queue (IBM MQ in most European banks).
The plan is a Minimal API host first (fastest to a running system, easiest to
integration-test), then a folder watcher, then possibly a queue consumer. Three
hosts over one core is the point.

**Return type:** `ProcessingResult`, not a generic `Result<T>`. A rejected
payment is not an error — it's a normal business outcome that produces a message
just like acceptance does. Both paths are first-class and live in the same type.

**Message version:** `pacs.008.001.08` — pinned by scheme, not "newest". This
is the version the SEPA 2025 rulebook, CBPR+ SR2025 and T2 RTGS all mandate, so
it's the one a bank in this market actually receives. Sample and schema versions
must match.

Two corrections to earlier assumptions, kept here so they don't get made twice:

- `.09` was the original pin. No European scheme uses it; it turns up mainly in
  private dialects. `.08` is the realistic choice.
- `issettled/iso20022-issettled` is **not** a usable sample source. Its files
  are wrapped in a proprietary `urn:issettled` envelope, use invented settlement
  codes (`TDSA`/`TDSO`) and a non-ISO currency (`USDDSO`), and contain no
  account elements at all.

Complete pacs.008 instances are genuinely scarce in public — iso20022.org
publishes schemas, not example messages, and scheme test packs sit behind SWIFT
MyStandards and CSM logins. The current sample came from the
`socrates8300/mx20022` repo at `.13` and was retargeted to the `.08` namespace;
nothing else was altered, and it validates against the official `.08` XSD.

**File placement:** sample XML is test data (`Nostro.Tests/Samples/`). XSD is a
production asset the core needs at runtime (`Nostro.Core/Schemas/`, embedded
resource) — add it only when a test forces it.

## Working agreement

**This is a test-driven project, and that is deliberate.** I've never done TDD
and want to learn it. Here it's a design tool: the library starts empty and I
don't know what types I need, so the test forces me to name the entry point and
decide what a result looks like before I can hide inside implementation detail.

The loop is red, green, refactor — and the refactor step is where the design
actually happens, so don't skip it.

Rules I'm holding myself to:

- **Test behaviour, not structure.** Assert observable outcomes, never that an
  internal method was called. If a test breaks during a refactor where the
  output didn't change, it was testing structure.
- **Test the front door only.** Public entry points. Don't make things public in
  order to test them — if something needs its own test, it wants to be its own
  class.
- **Don't add a field to the model that no test reads.** Not one. The standard
  defines ~80 fields for a pacs.008; I need maybe six. Mapping all of them
  produces boredom and no understanding.
- **No premature ceremony.** No Clean Architecture layers, no `IRepository`, no
  aggregate roots, no folders named after patterns. This is a translation and
  validation problem, not a domain-discovery problem — the model is already
  written down by a standards body and I don't get to disagree with it.
- **Structure is not architecture.** Two projects and a reference is a place to
  put files. Architecture starts at the first decision that constrains future
  decisions.

## Do not write my tests

**This is the most important instruction in this file.**

I write every test myself. That is the entire point of the project — I'm
learning TDD, and the learning happens in the moment where I have to decide what
to call the entry point, what it takes, and what it returns. If you write the
test, I get working code and learn nothing.

So:

- **Never write a test for me**, not even as an example, not even "just to show
  the shape", not even when I'm stuck.
- **Never write implementation code before a failing test exists.** If I ask you
  to implement something and there's no test for it, say so and stop.
- When I'm stuck, help by asking questions — what's the input, what's the
  observable outcome, what would you name it — not by producing the answer.
- You may review a test I've written, and say what's wrong with it.
- You may write production code to make a test I wrote pass, if I ask.

If I ask you to write a test anyway, remind me of this once. If I ask again,
do it.

## How I want to be helped

Explanations should be brief — one idea at a time, so I have room to ask
follow-ups. Long multi-point answers are hard to hold onto.

Ask me for my proposal before giving yours, especially on design decisions.
Socratic when I'm learning something; direct answers when I'm just building or
asking about tooling.

I'm Czech and pushing my English toward native-level professional. Correct me by
reformulating what I said the way a native engineer would phrase it, inline —
not as a list of corrections.

## Where I am right now

Solution scaffolded and building clean. `Nostro.Core` is empty — no types yet,
which is correct.

```
Nostro.slnx
global.json                           MTP opt-in for `dotnet test`
Nostro.Core/                          empty class library
Nostro.Tests/                         xunit.v3 4.0.1, references Core, no tests yet
  Samples/pacs.008.001.08.xml         real SEPA message, copies to output
```

The sample carries both IBANs (`DE89…` debtor, `FR76…` creditor), `EUR 500.00`,
a UETR, `SttlmMtd` `CLRG` and `ChrgBr` `SLEV` — enough to drive every check in
the list above.

No XSD in the repo yet. It goes into `Nostro.Core/Schemas/` when a test forces
it, not before.

Next:

1. **I write the first test.** It asserts only that a real pacs.008 goes in and
   comes back Accepted — not parsing, not validation, not correctness, just that
   the front door exists and returns a decision. It won't compile, which is
   correct: the types get created to satisfy it, and the first implementation is
   allowed to be a hardcoded `Accepted`.
2. **I write the second test** — the same sample with the debtor IBAN removed,
   expecting Rejected. That's what forces real parsing to appear, and where a
   reason code first earns its existence.

Both are mine to write. See "Do not write my tests".

Keep this section current — update it at the end of a session that changes the
state, so it never describes a project that no longer exists.

## Glossary

| Term | Meaning |
|---|---|
| **pacs.008** | FI-to-FI Customer Credit Transfer. The payment instruction itself. |
| **pacs.002** | Payment Status Report. The answer: `ACCP` accepted, `RJCT` rejected. |
| **pacs.004** | Payment Return. Money going back. |
| **camt.056** | Request to cancel a payment already sent. |
| **pain.001** | What a corporate sends its own bank (customer-to-bank, not interbank). |
| **End-to-end id** | Reference set by the originator, preserved across the whole chain. Not the same as the transaction id. |
| **UETR** | Unique End-to-end Transaction Reference — a UUID for tracking across the network. |
| **Nostro account** | An account our bank holds at another bank. |
| **CBPR+** | SWIFT's usage guidelines for cross-border ISO 20022 payments. |
| **SEPA SCT / SCT Inst** | Euro credit transfer schemes; Inst is the instant variant. |
