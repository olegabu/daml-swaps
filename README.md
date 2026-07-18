# IRS on Canton/DAML — Dev Bootstrap

## What this is

A learning/prototyping project: an Interest Rate Swap (IRS) modeled as DAML
smart contracts, intended to run on Canton, plus supporting client code.
The code is intentionally close to real derivatives lifecycle mechanics: execution,
periodic settlement, variation margin, novation, and termination — with
settlement modeled as **atomic delivery-versus-payment (DvP)**: cash moves
in the *same transaction* as the trade's state update, never as a separate
off-ledger step.

## Roles

- **`fixedRatePayer` / `floatingRatePayer`** — the two trade counterparties,
  named for which leg each one pays (an ISDA-confirmation-style defined
  term, not an invented label). Signatories of the trade; they cannot move
  cash — their own or each other's — outside a settlement (moving a
  `CashHolding` requires the oracle to co-authorize), and they cannot
  trigger settlement themselves.
- **Oracle** — the calculation/settlement agent. The *sole* controller of
  `SettlePeriod` and `SettleMargin`. It submits only objective external
  numbers (a rate fixing, a mark-to-market); it never names a party or a
  contract ID, so it stays oblivious to counterparty identities. The
  contract derives who owes whom. Observer on the trade and on both cash
  accounts (so it can see them to move them — see `ARCHITECTURE.md`).
- **Bank** — issuer of the tokenized cash accounts (`CashHolding`).

## Modules

`daml/` contains four modules:

- **`Types.daml`** — supporting data types (FixedLeg, FloatingLeg,
  PeriodRecord, MarginRecord) plus closed enums `DayCountConvention`,
  `Frequency`, `Currency` (USD/EUR), and `ReferenceRate`
  (SOFR/LIBOR/EURIBOR). `LifecycleEvent` is a sum type (`PeriodEvent
  PeriodRecord | MarginEvent MarginRecord`) letting `IRSTrade.history` hold
  both differently-shaped audit entries in one chronological list.
  `dayCountFraction` implements `Act360`/`Act365`/`Thirty360` as real date
  arithmetic (no holiday/business-day adjustment). Shaped like ISDA CDM's
  auto-generated data classes in spirit, simplified. Not a copy of
  ISDA/Digital Asset's actual CDM-DAML source.
- **`Cash.daml`** — a minimal tokenized `CashHolding` (one account per
  party, `label` for a human-readable alias, `observers` so the oracle can
  see it). Its `AdjustBalance` choice debits/credits the balance in place;
  this is the primitive the trade's atomic DvP is built on. It's controlled
  by **owner + oracle together**, so a counterparty can't move (or mint into)
  its own account unilaterally — the balance only ever changes inside a
  settlement, where both authorities co-occur.
- **`IRS.daml`** — the main contract:
  - `IRSProposal` / `AcceptIRS` / `RejectIRS` — propose-accept execution;
    each party names its own settlement account.
  - `IRSTrade` — the live, dual-signatory contract, carrying both parties'
    cash account IDs (kept fresh on every settle).
  - `SettlePeriod` — oracle submits a `periodEnd` and the floating fixing;
    the contract computes the real day-count fraction (`Types.daml`'s
    `dayCountFraction`, from the trade's own tracked `lastSettledDate` to
    `periodEnd`, under the fixed leg's `DayCountConvention`), nets the
    coupon, moves cash atomically (DvP), advances `lastSettledDate`, and
    appends a `PeriodEvent` to `history`. See the demo's "Periodic
    settlement" section for the math.
  - `SettleMargin` — oracle submits an `asOf` date and the mark-to-market;
    the contract derives direction, moves variation margin atomically, and
    appends a `MarginEvent` to `history` (same list `SettlePeriod` appends
    to — see `Types.daml`'s `LifecycleEvent`). Also bumps
    `postedMarginByFixedRatePayer`/`postedMarginByFloatingRatePayer`, a
    separate *cumulative, non-netted* running total per party — not to be
    confused with the per-call `history` entries.
  - `ProposeNovation` / `AcceptNovation` / `RejectNovation` — propose-accept
    novation, named for the ISDA Novation Agreement roles: the
    **transferor** proposes (controller = transferor alone); the
    **transferee** then accepts, bringing its own account, or declines
    (controller = transferee alone).
  - `ProposeTermination` / `AcceptTermination` / `RejectTermination` —
    propose-accept early termination: either party proposes (controller =
    proposer alone); the other party then ends the trade for good, or
    declines and the original trade is recreated unchanged (controller =
    the other party alone). See [ARCHITECTURE.md](ARCHITECTURE.md),
    "Propose-accept: carrying authority across transactions", for why both
    of these replaced a direct dual-controller choice.
  - `SettlementFailed` — terminal state created when a payer's account can't
    cover what's owed; the trade is archived with no successor, so no
    further lifecycle choices are structurally possible.
- **`Setup.daml`** — the init-script (wired via `daml.yaml`'s
  `init-script`, runs automatically on every `daml start`). Allocates
  Alice, Bob, Charlie, Bank, Oracle; registers them as Navigator/JSON API
  users; funds Alice's, Bob's, and Charlie's accounts (1,000,000 USD each,
  observed by the oracle — Charlie's is unattached to any trade, so he can
  accept a novation later with no separate funding step); and strikes a
  live `IRSTrade` via the real propose-accept flow (Alice/Bob only).
- **`Test.daml`** — Daml Script tests (`daml test`), two kinds: direct,
  ledger-free unit tests of `dayCountFraction` (`Act360`, `Act365`, and
  `Thirty360`'s day-31 capping logic specifically — the lifecycle tests
  below only ever hit a round `90/360 = 0.25` span, which wouldn't catch a
  scaling bug that happened to still land on `0.25`); and full lifecycle
  tests: settlement moves cash, margin moves cash, insufficient funds →
  default, a party can't self-submit, out-of-band rate / oversized MTM are
  rejected, a stale `periodEnd` is rejected, a second period correctly
  nets from the advanced `lastSettledDate`, and novation accept/reject/
  authorization guards.

All state changes follow DAML's archive-and-recreate pattern (no in-place
mutation). See `ARCHITECTURE.md` for why, and for the broader Canton
concepts this code assumes.

## Getting started

The only hard dependency is the **Daml SDK**; it bundles everything else
(the sandbox ledger, Navigator, `daml` assistant). The SDK needs a **JDK**
on the machine.

**1. Install a JDK (11+, 17 recommended).** The Daml SDK's sandbox ledger,
Navigator, and the `daml` assistant itself are all JVM-based, so they need
a JDK on `PATH` even though the code you write is DAML, not Java.

```sh
# macOS
brew install openjdk@17

# Debian/Ubuntu
sudo apt-get install -y openjdk-17-jdk

# verify
java -version
```

**2. Install the Daml assistant.**

```sh
curl -sSL https://get.daml.com/ | sh
```

This drops the `daml` binary under `~/.daml/bin`. Add it to your `PATH` if
the installer didn't (it prints the exact line):

```sh
export PATH="$HOME/.daml/bin:$PATH"   # add to ~/.bashrc or ~/.zshrc
```

**3. Get the pinned SDK version.** This project pins **SDK 2.10.4** in
[daml.yaml](daml.yaml). The first `daml build`/`daml start` inside the
project auto-installs that version; to pull it ahead of time:

```sh
daml install 2.10.4
daml version        # 2.10.4 should be listed
```

## Running it

From the project root:

```sh
daml build              # compile the four modules to a DAR
daml test               # run the Daml Script lifecycle tests
daml start              # sandbox + JSON API (7575) + Navigator (7500),
                        # runs Setup automatically
```

`daml start` runs [Setup.daml](daml/Setup.daml) for you (via `daml.yaml`'s
`init-script`), which:

- allocates **Alice, Bob, Charlie, Bank, Oracle** and registers a Navigator
  user for each (`alice`, `bob`, `charlie`, `bank`, `oracle`);
- has the Bank fund **Alice**, **Bob**, and **Charlie** with **1,000,000
  USD** each (oracle added as observer, so it can move them) — Charlie's
  account sits ready for the novation step below, unattached to any trade;
- strikes one live **`IRSTrade`** between **Alice and Bob** via the real
  propose→accept flow — 10,000,000 USD notional, Alice pays **4% fixed**,
  Bob pays **SOFR** floating, quarterly, maturing 2027-07-20.

Leave `daml start` running; the demo below drives this seeded trade. The
ledger/API ports it prints are: **6865** (gRPC Ledger API), **7575** (JSON
API), **7500** (Navigator).

## Demo

A guided walkthrough of the full lifecycle on the seeded trade: **daily
margin in both directions**, a **periodic settlement**, then **novation**
and **termination** — watching Alice's and Bob's cash balances move in
real time. Every step is drivable from Navigator alone; nothing here needs
`daml repl` or any other tool.

### Open three windows, one identity each

Navigator keeps you logged in as a single user per browser session, so give
each identity its **own Chrome profile** (separate `--user-data-dir`) and
watch all three side by side. With `daml start` running:

```sh
# Linux
google-chrome --user-data-dir=/tmp/chrome-alice  http://localhost:7500 & \
google-chrome --user-data-dir=/tmp/chrome-bob    http://localhost:7500 & \
google-chrome --user-data-dir=/tmp/chrome-oracle http://localhost:7500 &

# macOS
open -na "Google Chrome" --args --user-data-dir=/tmp/chrome-alice  http://localhost:7500
open -na "Google Chrome" --args --user-data-dir=/tmp/chrome-bob    http://localhost:7500
open -na "Google Chrome" --args --user-data-dir=/tmp/chrome-oracle http://localhost:7500
```

In each window pick the matching user on Navigator's sign-in screen:

| Window | Log in as | Watches |
|--------|-----------|---------|
| 1 | `alice`  | her `CashHolding` **Alice-USD** + the `IRSTrade` |
| 2 | `bob`    | his `CashHolding` **Bob-USD** + the `IRSTrade` |
| 3 | `oracle` | drives every settlement; sees both accounts + the trade |

Starting balances: **Alice-USD = 1,000,000**, **Bob-USD = 1,000,000**.
Keep the two contracts visible in Alice's and Bob's windows — every step
below archives-and-recreates them with a new balance, so you'll see the
numbers change live. (Each move creates a *fresh* `IRSTrade` and
`CashHolding`; the old ones drop out of the active set.)

> **Try to cheat first (optional):** in Alice's window, open her
> **Alice-USD** `CashHolding` and exercise **`AdjustBalance`** directly with
> `delta = 500000` — it **fails** authorization ("missing Oracle"). A
> counterparty can't mint into or move its own account; a balance change
> needs the owner *and* the oracle together, which only happens inside a
> settlement below.

### 1. Daily variation margin — direction A (Bob → Alice)

Margin is driven by the **oracle**, which submits a single mark-to-market
from **Alice's (`fixedRatePayer`) perspective**. Positive MTM favors Alice,
so Bob posts to her.

In the **oracle** window, open the live `IRSTrade` and exercise
**`SettleMargin`**:

- `asOf` = `2026-07-20`  (the date this mark was struck)
- `markToMarket` = `50000`

Result — cash moves atomically in the same transaction:

| | Before | After |
|---|--:|--:|
| Alice-USD | 1,000,000 | **1,050,000** |
| Bob-USD   | 1,000,000 | **950,000** |

Watch Alice's and Bob's windows update. The recreated trade also carries
`postedMarginByFloatingRatePayer = 50000`, and appends a `MarginEvent` to
`history`.

### 2. Daily variation margin — direction B (Alice → Bob)

Next day the mark flips. On the current `IRSTrade`, exercise
**`SettleMargin`** again in the **oracle** window with a **negative** mark
(now favoring Bob, so Alice posts):

- `asOf` = `2026-07-21`
- `markToMarket` = `-30000`

| | Before | After |
|---|--:|--:|
| Alice-USD | 1,050,000 | **1,020,000** |
| Bob-USD   | 950,000 | **980,000** |

You've now seen margin flow **both directions** off one oracle-submitted
number, with the contract deriving who owes whom. `history` now holds two
`MarginEvent`s in the order they happened.

> **Fat-finger / default guards:** a `markToMarket` of `0`, or one whose
> magnitude exceeds the 10,000,000 notional, is rejected outright. A move
> larger than the payer's balance instead archives the trade into
> **`SettlementFailed`** (no successor trade) — try `markToMarket = 9000000`
> on a fresh run to see the default path.

### 3. Periodic settlement (the coupon / net payment)

Now the quarterly reset. In the **oracle** window, on the current
`IRSTrade`, exercise **`SettlePeriod`**:

- `periodEnd`     = `2026-10-20`  (a quarter after the effective date)
- `observedRate`  = `0.03`  (the SOFR fixing — a **decimal fraction**,
  `0.03` = 3%, **not** `3`)

**How the payment is calculated.** The contract nets the legs — fixed 4%
vs. floating 3% on 10,000,000 — over the elapsed period, using a *real* day
count fraction, not a hardcoded constant:

```
dayFraction = dayCountFraction fixedLeg.dayCount lastSettledDate periodEnd
(0.04 − 0.03) × 10,000,000 × dayFraction = netPayment  → Alice (fixed payer) owes Bob
```

`lastSettledDate` is a field the trade tracks **itself** — `effectiveDate`
until the first `SettlePeriod`, then that call's `periodEnd` — so the
elapsed period is derived from the contract's own state, not from a second
date the oracle would otherwise have to submit (and could get wrong). The
seeded trade's fixed leg uses `Thirty360` (30/360: every month counted as
30 days, year as 360), so the elapsed time from `2026-07-20` to `2026-10-20`
— exactly three calendar months — comes out to exactly `90/360 = 0.25`:

```
(0.04 − 0.03) × 10,000,000 × 0.25 = 25,000  → Alice (fixed payer) owes Bob
```

| | Before | After |
|---|--:|--:|
| Alice-USD | 1,020,000 | **995,000** |
| Bob-USD   | 980,000 | **1,005,000** |

The recreated trade advances `lastSettledDate` to `2026-10-20` and appends
a `PeriodEvent` (wrapping a `PeriodRecord`: rate, net payment, who paid
whom) to `history` — the same list the two margin calls above appended
`MarginEvent`s to, now three entries long, in order. Inspect it in any
window. Above the 4% fixed rate the flow reverses: try `observedRate =
0.05` and Bob pays Alice 25,000 instead.

> **Try a second period:** exercising `SettlePeriod` again with `periodEnd
> = 2026-10-20` (the same date) **fails** — `lastSettledDate` is now
> `2026-10-20`, and the contract requires `periodEnd` to be strictly after
> it, so a stale or repeated fixing can't silently corrupt the schedule.
> Submit `periodEnd = 2027-01-20` instead (the next quarter): the day count
> runs from the *new* `lastSettledDate` (`2026-10-20`), not the original
> `effectiveDate`, and — since `Thirty360` treats every 3-calendar-month
> gap as exactly 90/360 — nets the same 25,000 again.
>
> **Why `Thirty360` and not `Act360`/`Act365`?** Those count *actual*
> calendar days instead of treating every month as 30: `2026-07-20` to
> `2026-10-20` is actually **92** days, so `Act360` would net
> `0.01 × 10,000,000 × (92/360) = 25,555.56` for this same period — not the
> round `25,000` above. (This particular trade's four quarters run 92, 92,
> 90, and 91 actual days respectively, so `Act360`/`Act365` give a slightly
> different fraction each period even at a constant rate — only
> `Thirty360`'s every-month-is-30-days rule makes every calendar quarter
> exactly `90/360 = 0.25`.) Try switching the seeded trade's
> `fixedLeg.dayCount` to `Act360` in `Setup.daml` to see this in action.

### 4. Novation (propose → accept, entirely in Navigator)

Moving a party in or out of the trade needs two parties' consent, but as
**propose-accept** rather than one dual-controller choice — so, unlike the
old `Novate`, it's fully drivable from Navigator, no `daml repl` needed.
See [ARCHITECTURE.md](ARCHITECTURE.md), "Propose-accept: carrying authority
across transactions", for why this split was necessary in the first place
(and why it's the only thing that works once Bob and Charlie are on
genuinely separate participant nodes).

Open a **fourth** Chrome profile for **Charlie** — `Setup.daml` already
funded his **Charlie-USD** account (1,000,000 USD), unattached to any
trade, exactly so he can accept a novation with no separate funding step:

```sh
# Linux
google-chrome --user-data-dir=/tmp/chrome-charlie http://localhost:7500 &

# macOS
open -na "Google Chrome" --args --user-data-dir=/tmp/chrome-charlie http://localhost:7500
```

Log in as `charlie`.

**Propose** — in **Bob's** window, on the current `IRSTrade`, exercise
**`ProposeNovation`**:

- `transferor` = Bob (Navigator offers a dropdown of known parties)
- `transferee` = Charlie

This archives the `IRSTrade` and creates a `NovationProposal`. Bob's part
is done: watch it disappear from **Bob's** window (he's no longer a
stakeholder of anything trade-related) and appear as a proposal in
**Charlie's** window (he's an observer of it).

**Accept** — in **Charlie's** window, open the `NovationProposal` and
exercise **`AcceptNovation`**:

- `transfereeCashCid` = Charlie's **Charlie-USD** `CashHolding` contract ID
  (open that contract in his window to copy it)

A new `IRSTrade` appears with **Charlie in Bob's slot** — check **Alice's**
window: her counterparty is now Charlie, and her balance/history are
unchanged (novation transfers the *position*, not a cash payment).
Charlie-USD still reads 1,000,000.

> **Decline instead:** exercising **`RejectNovation`** on the proposal
> (also a Charlie-only choice) recreates the ORIGINAL trade — Alice and Bob,
> unchanged — rather than stranding it. Try this on a fresh run: the trade
> survives a declined novation and keeps settling normally afterward.

### 5. Termination (propose → accept, entirely in Navigator)

Ending the trade early needs both current counterparties' consent too, and
it's propose-accept for the same reason novation is: `ProposeTermination`
(controller = proposer alone) / `AcceptTermination` or `RejectTermination`
(controller = the other party alone). See
[ARCHITECTURE.md](ARCHITECTURE.md), "Propose-accept: carrying authority
across transactions" — termination gets the identical treatment novation
does, and the identical reasoning for why a direct `controller
fixedRatePayer, floatingRatePayer` choice can't be submitted from a single
Navigator window (or, in a real deployment, from a single participant at
all).

Continuing from the novated trade above, the current counterparties are
**Alice** and **Charlie** — both already have open windows.

**Propose** — in **Alice's** window, on the current `IRSTrade`, exercise
**`ProposeTermination`**:

- `proposer` = Alice

This archives the `IRSTrade` and creates a `TerminationProposal`. Alice's
part is done: watch it disappear from her window and appear as a proposal
in **Charlie's** window (he's the `otherParty`, and thus its controller).

**Accept** — in **Charlie's** window, open the `TerminationProposal` and
exercise **`AcceptTermination`** (no arguments). The `IRSTrade` is gone
from every window with no successor — the trade is over.

> **Decline instead:** exercising **`RejectTermination`** on the proposal
> (also Charlie-only) recreates the ORIGINAL trade unchanged, same as
> declining a novation. Either party can be the one to `ProposeTermination`
> — it isn't limited to whoever proposed the most recent novation.

> The oracle always submits as itself (`actAs: [oracle]`) with no `readAs` —
> it's a declared observer on both cash accounts, which is what lets it
> *see* them in order to move them. See [ARCHITECTURE.md](ARCHITECTURE.md),
> "Authority vs. visibility".

## What's intentionally simplified / left as TODO

- `SettlePeriod`'s day-count fraction is now real date arithmetic
  (`Types.daml`'s `dayCountFraction`: `Act360`/`Act365`/`Thirty360`), but
  with **no holiday calendars or business-day adjustment** — period
  boundaries are used exactly as submitted rather than rolled to the
  nearest business day. It also nets a single rate difference under one
  convention (the fixed leg's), rather than accruing each leg separately
  under its own convention and netting the two gross amounts, which is
  what a real IRS does. Real holiday-calendar-aware scheduling and
  rate-reset logic belong in an off-ledger pricing engine (e.g. something
  Strata-shaped) whose output the oracle brings on-ledger as a verified
  input — not computed inside the DAML choice itself.
- No grace period: a payer who can't cover a settlement/margin move fails
  immediately. Real CSAs allow a cure period (typically T+1) before
  failure-to-pay escalates to a formal ISDA Master Agreement Event of
  Default — what's modeled here (`SettlementFailed`) is closer to a single
  settlement fail than that broader, more consequential concept.
- `SettlementFailed` records the facts but doesn't route into any
  default-management / close-out-auction logic.
- Novation here is **bilateral** (transferor + transferee consent only).
  Real ISDA novations are typically **tri-party** — they also require the
  *remaining* party's consent, since novating changes who they face for
  counterparty credit risk. This codebase currently assumes that consent
  is implicit in having agreed to the trade documentation up front.
- Mark-to-market and rate fixings are trusted oracle inputs; there's no
  on-ledger cross-check of the number itself (a real deployment would use
  a signed/attested feed).
- `CashHolding` is a bespoke local template for dev purposes — a real
  deployment would use an external, standards-based token (Canton Token
  Standard / CIP-56) representing a tokenized deposit or similar. No
  formal verification, no ISDA SIMM/margin-methodology integration yet.

## Goals

See `NEXT_STEPS.md` for the prioritized build list.
