# IRS, CDS, and TRS on Canton/DAML

## What this is

A learning/prototyping project: an Interest Rate Swap (IRS), a Credit
Default Swap (CDS), and an equity index Total Return Swap (TRS), all
modeled as DAML smart contracts, intended to run on Canton. The code is
intentionally close to real derivatives lifecycle mechanics: execution,
periodic settlement, variation margin, novation, and termination — with
settlement modeled as **atomic delivery-versus-payment (DvP)**: cash moves
in the *same transaction* as the trade's state update, never as a separate
off-ledger step.

All three products share their novation and termination mechanics through
a single Daml **interface** implementation (`Lifecycle.daml`) rather than
three copies of the same logic — see "Modules" below and
[ARCHITECTURE.md](ARCHITECTURE.md) for why and how.

## Roles

### Common to all three products

- **Oracle** — the calculation/settlement agent. The *sole* controller of
  every settlement/margin/credit-event choice. It submits only objective
  external numbers (a rate fixing, a mark-to-market, a recovery rate); it
  never names a party or a contract ID, so it stays oblivious to
  counterparty identities. The contract itself derives who owes whom.
  Observer on the trade and on every cash account involved (so it can see
  them to move them — see [ARCHITECTURE.md](ARCHITECTURE.md)).
- **Bank** — issuer of the tokenized cash accounts (`CashHolding`).

### IRS

- **`fixedRatePayer` / `floatingRatePayer`** — the two trade counterparties,
  named for which leg each one pays. Signatories of the trade.

### CDS

- **`protectionBuyer` / `protectionSeller`** — the two trade
  counterparties. The buyer pays a running premium (the spread) for
  protection against a credit event on a reference entity; the seller
  collects the premium and is on the hook for the contingent payout if a
  credit event occurs. Signatories of the trade.

### TRS

- **`totalReturnReceiver` / `totalReturnPayer`** — the two trade
  counterparties. The receiver gets the index's total return (price
  appreciation plus reinvested dividends — see `TRS.daml` below) and pays
  financing; the payer — the dealer side, already capitalized against the
  swing — pays the index return and collects financing. Signatories of
  the trade.

## Modules

`daml/` contains ten modules:

- **`Types.daml`** — supporting data types shared by all three products:
  `FixedLeg`, `FloatingLeg`, `CashflowRecord`, `MarginRecord` (IRS's own
  `LifecycleEvent` sum type wraps the last two), plus closed enums
  `DayCountConvention`, `Frequency`, `Currency` (USD/EUR), and
  `ReferenceRate` (SOFR/LIBOR/EURIBOR). `dayCountFraction` implements
  `Act360`/`Act365`/`Thirty360` as real date arithmetic (no
  holiday/business-day adjustment). `FixedLeg` is reused as-is for CDS's
  premium leg (`rate` = the running spread, e.g. `0.01` = 100bps) — same
  `rate`/`dayCount`/`frequency` shape as an IRS fixed coupon, so there's no
  parallel `PremiumLeg` type; `FloatingLeg` is reused as-is for TRS's
  financing leg the same way (`spread` on top of the oracle-submitted
  overnight rate). Shaped like ISDA CDM's auto-generated data classes in
  spirit, simplified. Not a copy of ISDA/Digital Asset's actual CDM-DAML
  source.
- **`Cash.daml`** — a minimal tokenized `CashHolding`, identified by a
  **contract key** (`CashAccountKey` = issuer + owner + currency, not a
  contract id — see "What `CashHolding` maps to" below), plus the shared
  **`tryMoveCash`** DvP helper every product's settlement choices call
  into, which resolves both parties' CURRENT accounts by key. Two
  choices, split by what each actually needs to be true: `CreditAccount`
  (`controller issuer` alone) mints new balance — models the custodian
  recording a genuine incoming transfer, so an owner can't fabricate
  balance just by asking; `Transfer` (`controller owner` alone) debits
  this account and credits a destination account by key, atomically — a
  real, value-conserving move a party can make unilaterally, no oracle
  needed, the same as a real bank transfer. `tryMoveCash` is just one
  `Transfer` call (payer → receiver) exercised as a nested action inside
  a settlement choice.
- **`Lifecycle.daml`** — Daml **interfaces** hosting trade origination,
  novation, and termination, written once and shared by `IRSTrade`,
  `CDSTrade`, and `TRSTrade`: `IProposable`, `INovatable`/
  `INovationProposal`/`INovationConfirmation`, and `ITerminable`/
  `ITerminationProposal`.
  Modeled directly on the Daml SDK's own `daml-intro-13` tutorial
  (`IAsset`/`Cash`/`NFT`) — see [ARCHITECTURE.md](ARCHITECTURE.md),
  "Shared lifecycle via interfaces", for the full design and why it's
  safe. `IProposable` is the simplest of the three: one per-product hook
  (`toLiveTrade`) builds the first live trade, and `RejectTrade`'s body
  is identical across products (`pure ()`) — nothing product-specific in
  it at all.
- **`IRS.daml`** — the IRS-specific contract:
  - `TradeProposal` — propose-accept execution; `AcceptTrade`/
    `RejectTrade` are inherited from `Lifecycle.daml`'s `IProposable`,
    not hand-written here. The proposal names the `cashIssuer`
    (custodian) and `settlementCurrency` both parties will settle
    through — not a specific account, which is resolved by key at every
    settlement (see Cash.daml's `CashAccountKey`).
  - `IRSTrade` — the live, dual-signatory contract. Implements
    `INovatable`/`ITerminable` (novation/termination are inherited too).
  - `SettleCashflow` — oracle submits a `periodEnd` and the floating
    fixing; the contract computes the real day-count fraction
    (`Types.daml`'s `dayCountFraction`, from the trade's own tracked
    `lastSettledDate` to `periodEnd`, under the fixed leg's
    `DayCountConvention`), nets the two legs against each other, moves
    cash atomically (DvP), advances `lastSettledDate`, and appends a
    `CashflowEvent` to `history`.
  - `SettleMargin` — oracle submits an `asOf` date and the mark-to-market;
    the contract derives direction, moves variation margin atomically, and
    appends a `MarginEvent` to `history`. Also bumps
    `postedMarginByFixedRatePayer`/`postedMarginByFloatingRatePayer`, a
    separate *cumulative, non-netted* running total per party.
  - `NovationProposal` / `NovationConfirmation` / `TerminationProposal` —
    slim templates whose `interface instance` blocks supply IRS's own
    per-product hooks (the view, and the field-substitution logic); the
    generic consent-flow choices themselves live in `Lifecycle.daml`.
  - `SettlementFailure` — terminal state created when a payer's account
    can't cover what's owed; the trade is archived with no successor, so
    no further lifecycle choices are structurally possible.
- **`CDS.daml`** — the CDS-specific contract, same shape as `IRS.daml`:
  - `TradeProposal` — same propose-accept shape (same `cashIssuer`/
    `settlementCurrency` naming, no account to pin; `AcceptTrade`/
    `RejectTrade` inherited from `IProposable`, same as IRS's); the
    proposer is always the protection buyer.
  - `CDSTrade` — the live, dual-signatory contract. Implements
    `INovatable`/`ITerminable` identically to `IRSTrade`.
  - `SettlePremium` — oracle submits a `periodEnd`; the contract computes
    the elapsed day-count fraction under the premium leg's convention and
    moves the premium **one-directionally** (protection buyer always pays
    protection seller — contrast IRS's two-leg netting) atomically,
    appending a `PremiumEvent`.
  - `SettleMargin` — identical mechanics to IRS's, against
    `protectionBuyer`/`protectionSeller`.
  - `SettleCreditEvent` — oracle submits a `CreditEvent` trigger
    (`Bankruptcy`/`FailureToPay`/`Restructuring`), the date, and a
    `recoveryRate`; the contract computes the cash-settled payout
    (`notional * (1 - recoveryRate)`) and moves it atomically from seller
    to buyer. **Consuming either way** — a credit event is itself a
    terminal lifecycle event (unlike periodic settlement, which
    continues): success creates `CreditEventSettlement` (the payout
    receipt), failure to cover it creates `SettlementFailure` — no
    successor `CDSTrade` on either branch.
  - `CreditEventSettlement` — terminal receipt carrying the payout details
    and the trade's prior `history`.
  - CDS-specific types (`CreditEvent`, `PremiumRecord`,
    `CreditEventRecord`, and CDS's own `LifecycleEvent` sum type —
    `PremiumEvent` / `MarginEvent` / `CreditEventEntry`) live in
    `CDS.daml` itself, not `Types.daml` — see
    [ARCHITECTURE.md](ARCHITECTURE.md), "Qualified imports", for why.
- **`TRS.daml`** — the TRS-specific contract, same propose-accept shape
  as `IRS.daml`/`CDS.daml`, modeling an equity index total return swap:
  - `TradeProposal` — names an `indexName : Text`, an
    `initialIndexLevel`, and a `financingLeg : FloatingLeg` (reused
    as-is — the funding leg's `spread` sits on top of whatever overnight
    rate the oracle fixes at each settlement, same shape as IRS's
    floating leg). No dividend field anywhere: the underlying is a
    *total return* index, meaning dividends are already reinvested in
    the published level, so accounting for them separately would
    double-count — a deliberate simplification, not an oversight.
  - `TRSTrade` — the live, dual-signatory contract. Implements
    `INovatable`/`ITerminable` identically to `IRSTrade`/`CDSTrade`.
    Tracks four fields no other product needs: `lastIndexLevel` (prior
    observation), `lastResetIndexLevel` (the reset anchor),
    `lastSettledDate` (day-count basis for funding, exactly like IRS's
    field of the same name), and `vmPaidSinceReset` (running sum of
    daily performance already paid out since the last reset).
  - `SettleMargin` — the daily variation-margin choice: the oracle
    submits `asOf`, `indexLevel`, and `observedRate` (the funding
    fixing); the contract computes the day's index move against the
    **prior day's level**, nets it against accrued financing
    (`observedRate + financingLeg.spread`, over the real elapsed days),
    and moves cash atomically.
  - `SettleReset` — the periodic true-up: same inputs, but measures the
    **cumulative** return since the last reset (against
    `lastResetIndexLevel`, not the prior day) and pays only the
    **residual** not already exchanged via daily `SettleMargin` calls —
    see [ARCHITECTURE.md](ARCHITECTURE.md)'s TRS section for why this
    never double-pays performance, even though daily percentage moves
    compound rather than telescope.
  - `AdjustForCorporateAction` — a state-mutating oracle event with
    **no cash leg at all** (a shape none of IRS/CDS's oracle choices
    have): the oracle submits a multiplicative `adjustmentFactor`
    rebasing both `lastIndexLevel` and `lastResetIndexLevel` together,
    modeling a stock split, special dividend, or index reconstitution
    that rescales the published level without changing anyone's actual
    economic exposure.
  - `NovationProposal` / `NovationConfirmation` / `TerminationProposal`
    — same slim shape as IRS's/CDS's; `substitute` swaps only the
    party, so the index anchors and `vmPaidSinceReset` ride along
    intact through a novation.
  - `SettlementFailure` — per-product, same terminal shape as IRS's/
    CDS's.
- **`Setup.daml`** — the init-script (wired via `daml.yaml`'s
  `init-script`, runs automatically on every `daml start`). Staged into
  separate functions rather than one long script: `setupUsersAndAccounts`
  allocates Alice, Bob, Charlie, Bank, Oracle, registers them as
  Navigator/JSON API users, and funds **one** `CashHolding` per party
  (Alice, Bob, Charlie — see "What `CashHolding` maps to" below for why
  one account safely serves all three seeded trades, and any number
  more); `proposeIrsTrade`/`proposeCdsTrade`/`proposeTrsTrade` each
  create a `TradeProposal`; `acceptIrsTrade`/`acceptCdsTrade`/
  `acceptTrsTrade` each exercise `AcceptTrade` to make it live. `setup`
  calls all of these in order — **comment out any `acceptXxxTrade` call
  to demo `AcceptTrade` yourself** from Navigator instead of having it
  already done: the corresponding `TradeProposal` is left sitting there,
  live, for Bob to open and accept.
- **`IRSTest.daml`** — IRS's Daml Script tests (31 tests): direct,
  ledger-free unit tests of `dayCountFraction`, plus full lifecycle tests
  covering every explicit choice at least once. Unchanged by the
  interface retrofit — it never inspected `NovationProposal`/
  `NovationConfirmation`'s own fields, only choice names/args and the
  resulting trade, so it serves as a regression net.
- **`CDSTest.daml`** — CDS's Daml Script tests (16 tests), mirroring
  `IRSTest.daml`'s fixture pattern: trade proposal/accept/reject, premium
  settlement, margin (happy path and default), novation and termination
  (via the same shared interfaces), and the credit-event-specific cases
  (payout success, default, out-of-band recovery rate rejected).
- **`TRSTest.daml`** — TRS's Daml Script tests (21 tests), same fixture
  pattern again: proposal/accept/reject, daily VM both directions,
  funding-accrual-per-elapsed-day, the **compounding-residual
  centerpiece** (two offsetting daily moves that net to zero while the
  period's true return is nonzero, trued up exactly by `SettleReset`'s
  residual), a zero-residual control case, reset re-anchoring, corporate
  action rebases (with and without an intervening reset), the full guard
  set, an insufficient-funds default, and novation/termination via the
  same shared interfaces (including that accrued state survives a
  novation).

All state changes follow DAML's archive-and-recreate pattern (no in-place
mutation). See `ARCHITECTURE.md` for why, and for the broader Canton
concepts this code assumes.

### What `CashHolding` maps to

An earlier version of this codebase stored a specific `ContractId
CashHolding` on every trade, refreshing it on each settle. That breaks
the moment two trades share an account: DAML's archive-and-recreate means
every balance change swaps in a brand-new contract id, so whichever trade
settles first silently strands every OTHER trade's stored id, and their
next settlement fails `CONTRACT_NOT_FOUND`. The stopgap was giving every
trade its **own segregated account** — workable for the handful of trades
this demo seeds, but obviously wrong in general: a real trader can have
hundreds of live positions, and nobody opens a hundred settlement
accounts to hold them.

The actual fix is a Daml **contract key** (`Cash.daml`'s
`CashAccountKey` = issuer + owner + currency — see the SDK's own
`daml-intro-3` tutorial for the pattern). `CashHolding` is now identified
by that key, not by a contract id, so `Setup.daml` funds exactly **one**
account per party, and all three seeded trades — indeed, any number of
trades — settle through it safely: `tryMoveCash` resolves each party's
CURRENT account via `fetchByKey`/`exerciseByKey` at the moment of each
settlement (see `IRS.daml`/`CDS.daml`/`TRS.daml`'s `TradeProposal`, which names a
`cashIssuer` and `settlementCurrency` rather than a specific account).
Nothing is ever stored and later found stale.

So what does `CashHolding` actually model? Two honest answers:

- **As built here, a real bank/margin ACCOUNT** — one running cash
  balance per (custodian, holder, currency) that many trades settle
  against, the way a real CSA nets exposure across a whole counterparty
  relationship (or a whole book) rather than being sharded per trade.
  That's a fair simplification for demonstrating atomic DvP, and contract
  keys are exactly the Daml mechanism that makes "many trades, one
  account" correct.
- **Not how a production token/holding actually works.** Real Canton
  Token Standard (CIP-56) and Daml Finance `Holding` implementations are
  closer to UTXOs than bank accounts: a party typically holds many small,
  individually fungible holding contracts — one per lot — that get split,
  merged, and locked as needed, rather than one mutable balance
  archived-and-recreated on every move. `CashHolding`'s single
  running-balance shape is a deliberate simplification for this
  teaching/dev-bootstrap project; it demonstrates atomic DvP without
  needing a full splitting/merging fungible-asset implementation on top.

**A trade-off worth naming, not glossing over:** resolving a shared
account by key means every trade settling through the SAME (issuer,
owner, currency) contends for that one contract — Daml serializes
concurrent transactions touching the same keyed contract, so a party with
many trades settling in the same instant sees contention/retries there,
not silent corruption. At this project's scale (one or two trades per
party) that's a non-issue; a real high-throughput deployment would shard
a party's flow across several accounts (still resolved by key — just
*more* keys), not fall back to per-trade segregation.

Bob's single account is funded far more heavily than Alice's
(10,000,000 vs. 1,000,000): as the CDS **protection seller** he's on the
hook for a credit event payout that can run up to the full notional, on
top of his IRS margin/settlement flows through that same account —
real protection sellers are capitalized against exactly this tail risk.

### `tryMoveCash`: how the oracle moves money it doesn't own

`CashHolding` has two choices, split by what each one actually needs to
be true, not lumped into one loosely-guarded "adjust the balance" choice:

- **`CreditAccount`** (`controller issuer` alone) — mints new balance.
  Models the custodian recording a real incoming transfer; the owner
  can't fabricate balance just by asking, so the owner isn't a controller
  here at all.
- **`Transfer`** (`controller owner` alone) — debits this account and
  credits a `destination` account (found by key) atomically, so nothing
  is fabricated: whatever leaves one account is exactly what appears in
  the other. Moving money you already own needs no one else's consent,
  same as a real bank transfer — not the issuer, and (deliberately) not
  the oracle either.

`tryMoveCash` — the helper every settlement choice in every product calls
— is just **one `Transfer` call**, payer's account to receiver's,
resolved by key. The interesting part: this works from inside a
derivatives settlement with **zero special-casing for the oracle** — the
oracle submits `SettleMargin`/`SettlePremium`/etc., but it is never a
*controller* of anything on `CashHolding`, only a declared *observer* (so
it can see the accounts to name them — see "What `CashHolding` maps to"
above). Authority for the actual money movement comes entirely from the
paying counterparty already being a signatory of the trade the oracle is
exercising a choice on — the same "authority carried forward" trick
`ProposeNovation`/`ProposeTermination` use (see
[ARCHITECTURE.md](ARCHITECTURE.md), "Propose-accept: carrying authority
across transactions") — plus one more step: the `issuer` party, being the
signatory of *both* the payer's and receiver's accounts (every account in
this project shares one Bank), becomes available to authorize the
destination-side `CreditAccount` the moment `Transfer` touches the
payer's account, with no separate consent needed. See
[ARCHITECTURE.md](ARCHITECTURE.md), "Mint vs. transfer," for that
authorization chain traced precisely, step by step.

One consequence worth knowing: a party CAN unilaterally `Transfer` their
own real money to any account they can see — no settlement, no oracle
involved (see the demo's "Try to cheat first" callout below for exactly
what that does and doesn't let you do from Navigator).

**Why a plain Alice → Bob transfer fails outside this sandbox, and how to
actually model one.** `IRSTest.daml`'s `testOwnerCanTransferWithReadAs`
only works because this whole demo (and its test suite) runs on a
**single participant** hosting Alice, Bob, the Bank, and the Oracle
together — `readAs: [bob]` just grants Alice's submission permission to
use data that ONE participant already has locally. In a **real
deployment**, Alice and Bob are normally on *separate* participants
(their own organizations' nodes), and neither visibility mechanism Daml
actually offers closes that gap for `Transfer` as designed:

- `readAs` doesn't reach across participants at all — it scopes which of
  a participant's own already-locally-hosted parties' authority a
  submission can use, not a way to pull data from a different node.
- Daml's real cross-participant visibility mechanism — **explicit
  contract disclosure**, where a stakeholder hands the submitter a signed
  contract payload out-of-band (HTTPS, email, whatever), attached to the
  command — doesn't help either, because it only works for plain
  `fetch`/`exercise` **by contract id**. `Transfer`'s `destination :
  CashAccountKey` resolves via `exerciseByKey`, and key-based lookups are
  explicitly excluded from what disclosure can satisfy: Alice's
  participant genuinely cannot resolve Bob's key, no matter what Bob
  discloses to her.

So `Transfer`, as designed, is a real party-to-party payment primitive
only when both accounts already happen to be visible to the submitter —
in a genuinely multi-participant deployment, "Alice submits `Transfer`
straight to Bob" just doesn't work, and it's not a bug to fix so much as
a structural consequence of contract keys being participant-local. Two
ways to actually model a party-to-party payment correctly:

- **Route it through the issuer (Bank)**, not the customer. Since
  `issuer` is a signatory of every `CashHolding` in this project, the
  Bank's own participant already has both accounts visible, by
  construction — no disclosure or readAs needed. This is also the
  realistic shape: a "send money" feature is normally the *custodian's*
  own service executing the transfer (authorized by the customer's
  `actAs`), not peer-to-peer visibility between two customers' own
  nodes — exactly how a real bank wire, within or between banks, actually
  works.
- **Take an explicit `ContractId`, not a key, for the destination** — a
  variant choice accepting `destinationCid : ContractId CashHolding` fed
  by a disclosed contract Bob supplies fresh, instead of
  `CashAccountKey`. This trades back some of the staleness-safety the key
  design exists for (see "What `CashHolding` maps to" above), so it only
  makes sense for a genuinely interactive, one-off flow — not a repeated
  settlement primitive like `tryMoveCash`, which must always resolve
  whatever the CURRENT account is without any hand-off dance.

`tryMoveCash` itself never hits this problem: the oracle is already a
declared observer on every account (visibility, not authority — see
above), so the party actually submitting always has full visibility
already, the same shape as "route it through a party that can see both
sides."

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

## Running it

From the project root:

```sh
daml build              # compile the ten modules to a DAR
daml test               # run all three products' Daml Script lifecycle tests (70 total)
daml start              # sandbox + JSON API (7575) + Navigator (7500),
                        # runs Setup automatically
```

`daml start` runs [Setup.daml](daml/Setup.daml) for you (via `daml.yaml`'s
`init-script`), which:

- allocates **Alice, Bob, Charlie, Bank, Oracle** and registers a Navigator
  user for each (`alice`, `bob`, `charlie`, `bank`, `oracle`);
- has the Bank fund **one account per party**: `Alice-USD` and `Charlie-USD`
  (1,000,000 USD each), `Bob-USD` (10,000,000 USD — see "What
  `CashHolding` maps to" above); Charlie's is unattached to any trade,
  ready for the novation demo below;
- strikes a live **`IRSTrade`** between **Alice and Bob** via the real
  propose→accept flow — 10,000,000 USD notional, Alice pays **4% fixed**,
  Bob pays **SOFR** floating, quarterly, maturing 2027-07-20, settling
  through `Alice-USD`/`Bob-USD`;
- strikes a live **`CDSTrade`** between **Alice and Bob** the same way,
  through the SAME two accounts — 10,000,000 USD notional, Alice
  (protection buyer) pays Bob (protection seller) a **100bps (1%)**
  running premium on reference entity "Acme Corp", quarterly, maturing
  2031-07-20;
- strikes a live **`TRSTrade`** between **Alice and Bob** the same way,
  through the SAME two accounts again — 10,000,000 USD notional, Alice
  (total return receiver) receives the total return of the "SPX-TR"
  index (starting level 16650.0) from Bob (total return payer), funded at
  **SOFR flat** (Act360), maturing 2027-07-20.

Leave `daml start` running; the demo below drives all three seeded
trades. The ledger/API ports it prints are: **6865** (gRPC Ledger API),
**7575** (JSON API), **7500** (Navigator).

## Demo

A guided walkthrough of all three products' full lifecycles, one section
per product: IRS's **daily margin in both directions**, a **periodic
settlement**, then **novation** and **termination**; CDS's **premium
settlement** and **credit event**; TRS's **daily variation margin**,
**periodic reset**, and **corporate-action rebase**. CDS and TRS both
reuse the exact same novation/termination mechanics the IRS section
walks, under the hood, but don't re-walk them a second and third time in
Navigator, since they're identical to IRS's. Every step is drivable from
Navigator alone; nothing here needs `daml repl` or any other tool.

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
| 1 | `alice`  | her `CashHolding` (shared by all three trades) + all three live trades |
| 2 | `bob`    | his `CashHolding` (shared by all three trades) + all three live trades |
| 3 | `oracle` | drives every settlement; sees every account + all three trades |

Starting balances: **Alice-USD = 1,000,000**, **Bob-USD = 10,000,000** —
the SAME one account each, settling the `IRSTrade`, the `CDSTrade`, and
the `TRSTrade` all together (see "What `CashHolding` maps to" above). The
three sections below (IRS, CDS, TRS) are each written independently, and
for clarity **assume a fresh `daml start` restart before each one**, so
every section's tables start from this same clean 1,000,000/10,000,000
baseline rather than wherever the previous section left the shared
account. Keep the account and the live trade visible in Alice's and
Bob's windows — every step below archives-and-recreates them with a new
balance, so you'll see the numbers change live. (Each move creates a
*fresh* trade and `CashHolding`; the old ones drop out of the active
set.)

> **Try to cheat first (optional):** in Alice's window, open her
> **Alice-USD** `CashHolding` and exercise **`CreditAccount`** directly
> with `delta = 500000` — it **fails** authorization ("missing Bank"). A
> counterparty can never fabricate balance out of thin air; only the
> issuer can mint that way.
>
> **`Transfer`, by contrast, genuinely doesn't need the oracle** — moving
> REAL balance a party already has needs no one else's consent, the same
> as a real bank transfer. It's just not demoable from Navigator here:
> Navigator submits `actAs` only, and referencing a *destination* account
> by key needs the submitter to already be able to see it — Alice can't
> see Bob's account any more than the oracle could see hers before it was
> made an observer (same rule, see "What `CashHolding` maps to" above and
> [ARCHITECTURE.md](ARCHITECTURE.md)'s "Authority vs. visibility"). See
> `IRSTest.daml`'s `testOwnerCanTransferWithReadAs` for this exercised
> directly with `readAs` — though note that's a **sandbox-only** fix:
> `readAs` only reaches contracts your own participant already has, so it
> works there purely because Alice and Bob happen to be co-hosted on one
> participant. It's not a general fix for a real, separately-hosted
> deployment — see "Why a plain Alice → Bob transfer fails outside this
> sandbox" above for what actually would be.

## IRS lifecycle

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
| Bob-USD   | 10,000,000 | **9,950,000** |

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
| Bob-USD   | 9,950,000 | **9,980,000** |

You've now seen margin flow **both directions** off one oracle-submitted
number, with the contract deriving who owes whom. `history` now holds two
`MarginEvent`s in the order they happened.

> **Fat-finger / default guards:** a `markToMarket` of `0`, or one whose
> magnitude exceeds the 10,000,000 notional, is rejected outright. A move
> larger than the payer's balance instead archives the trade into
> **`SettlementFailure`** (no successor trade) — try `markToMarket =
> -2000000` on a fresh run (favors Bob, so Alice pays) to see the default
> path: Alice's fresh 1,000,000 can't cover it. (Bob's own balance is
> deliberately hard to default this way in the guided demo — see "What
> `CashHolding` maps to" above for why he's funded so much more heavily.)

### 3. Periodic settlement (the coupon / net payment)

Now the quarterly reset. In the **oracle** window, on the current
`IRSTrade`, exercise **`SettleCashflow`**:

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
until the first `SettleCashflow`, then that call's `periodEnd` — so the
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
| Bob-USD   | 9,980,000 | **10,005,000** |

The recreated trade advances `lastSettledDate` to `2026-10-20` and appends
a `CashflowEvent` (wrapping a `CashflowRecord`: rate, net payment, who paid
whom) to `history` — the same list the two margin calls above appended
`MarginEvent`s to, now three entries long, in order. Inspect it in any
window. Above the 4% fixed rate the flow reverses: try `observedRate =
0.05` and Bob pays Alice 25,000 instead.

> **Try a second period:** exercising `SettleCashflow` again with `periodEnd
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

### 4. Novation (propose → accept → confirm, tri-party)

Moving a party in or out of the trade needs **three** consents, real ISDA
novation-style — the outgoing party, the incoming party, and the
*remaining* party, whose counterparty credit risk changes as a result and
who gets the final say. Each is still a single-party step via
**propose-accept**, chained twice, and — as of this version — the choices
themselves (`ProposeNovation`/`AcceptNovation`/`ConfirmNovation`/etc.) are
inherited from `Lifecycle.daml`'s shared interfaces rather than hand-written
per product. See [ARCHITECTURE.md](ARCHITECTURE.md), "Shared lifecycle via
interfaces" and "Propose-accept: carrying authority across transactions".

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
exercise **`AcceptNovation`** (no arguments — Charlie doesn't name an
account; his `Charlie-USD` is resolved automatically by key at the next
settlement, same as everyone else's — see "What `CashHolding` maps to"
above).

This does **not** create the novated trade yet — check **Alice's** window:
there is no live `IRSTrade` anywhere right now, for anyone. It creates a
`NovationConfirmation` instead, appearing in **Alice's** window (she's a
signatory of it — the remaining party's decision is what's still pending).

**Confirm** — in **Alice's** window, open the `NovationConfirmation` and
exercise **`ConfirmNovation`** (no arguments). *Now* a new `IRSTrade`
appears with **Charlie in Bob's slot** — Alice's counterparty is Charlie,
and her balance/history are unchanged (novation transfers the *position*,
not a cash payment). Charlie-USD still reads 1,000,000.

> **Decline instead, two different points:** exercising **`RejectNovation`**
> on the `NovationProposal` (a Charlie-only choice) has him decline outright
> before Alice is ever involved. Exercising **`DeclineNovation`** on the
> `NovationConfirmation` (an Alice-only choice) is her veto **after**
> Charlie has already accepted — she still gets the final word, because
> novating changes whose credit risk she's carrying. Either path recreates
> the ORIGINAL trade — Alice and Bob, unchanged — rather than stranding it,
> and it keeps settling normally afterward. Try both on a fresh run.

### 5. Termination (propose → accept)

Ending the trade early needs both current counterparties' consent, and
it's propose-accept for the same reason novation is: `ProposeTermination`
(controller = proposer alone) / `AcceptTermination` or `RejectTermination`
(controller = the other party alone) — also inherited from
`Lifecycle.daml`. See [ARCHITECTURE.md](ARCHITECTURE.md), "Propose-accept:
carrying authority across transactions" — termination gets the identical
treatment novation does, and the identical reasoning for why a direct
`controller fixedRatePayer, floatingRatePayer` choice can't be submitted
from a single Navigator window (or, in a real deployment, from a single
participant at all).

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

## CDS lifecycle

CDS's novation and termination work **exactly as walked above** — same
choice names, same tri-party consent flow, same authority-carrying trick
— because `CDSTrade` implements the identical `Lifecycle.daml` interfaces
`IRSTrade` does. Nothing new to demo there; this section covers only
what's actually different about CDS: **one-directional premium
settlement** (no netting, contrast the IRS section's periodic settlement)
and the **credit event** trigger.

`Alice-USD`/`Bob-USD` are the SAME accounts the IRS section above just
moved — the CDS trade was seeded from the start alongside the IRS one,
settling through the identical `Bank`/`USD` (see "What `CashHolding` maps
to" above). As noted above, this section **assumes a fresh `daml start`
restart**, so the tables below start from the clean seeded baseline —
**Alice-USD = 1,000,000, Bob-USD = 10,000,000** — rather than wherever
the IRS section happened to leave the shared account.

### 1. Premium settlement (the running spread)

In the **oracle** window, on the current `CDSTrade`, exercise
**`SettlePremium`**:

- `periodEnd` = `2026-10-20`

Unlike `SettleCashflow`, there's no `observedRate` to submit and no
netting — the protection buyer always pays the protection seller:

```
dayFraction = dayCountFraction premiumLeg.dayCount lastSettledDate periodEnd
premiumLeg.rate × notional × dayFraction = netPayment  → Alice (buyer) pays Bob (seller)
```

The seeded trade's premium leg is `Thirty360` at `1%` (100bps), so the
same clean `90/360 = 0.25` fraction as the IRS section applies:

```
0.01 × 10,000,000 × 0.25 = 25,000  → Alice pays Bob
```

| | Before | After |
|---|--:|--:|
| Alice-USD | 1,000,000 | **975,000** |
| Bob-USD   | 10,000,000 | **10,025,000** |

The recreated trade advances `lastSettledDate` and appends a
`PremiumEvent` to `history` — same rolling-schedule guard as
`SettleCashflow`: re-submitting `periodEnd = 2026-10-20` a second time
fails; the next quarter is `2027-01-20`.

### 2. Variation margin

Identical mechanics to the IRS section's §1–2, against `protectionBuyer`/
`protectionSeller` instead of `fixedRatePayer`/`floatingRatePayer` — not
walked again here.

### 3. Credit event (the contingent payout, terminal)

The reason this product exists: a credit event on the reference entity.
In the **oracle** window, on the current `CDSTrade`, exercise
**`SettleCreditEvent`**:

- `eventType` = `Bankruptcy` (or `FailureToPay` / `Restructuring`)
- `asOf` = `2027-01-21`  (any date on/before maturity)
- `recoveryRate` = `0.4`  (40% recovery — a **decimal fraction**, like
  `observedRate`/`markToMarket` elsewhere)

**Unlike every other choice in this codebase, this one is terminal no
matter what happens** — a credit event ends the trade, it doesn't
continue it. The payout is cash-settled (no physical delivery of the
defaulted obligation):

```
payout = notional × (1 − recoveryRate) = 10,000,000 × (1 − 0.4) = 6,000,000  → Bob (seller) pays Alice (buyer)
```

| | Before | After |
|---|--:|--:|
| Alice-USD | 975,000 | **6,975,000** |
| Bob-USD   | 10,025,000 | **4,025,000** |

The `CDSTrade` is archived with **no successor** — check every window,
it's simply gone — and a `CreditEventSettlement` receipt appears instead,
carrying the `CreditEventRecord` (event type, date, recovery rate, payout
amount) and the trade's prior `history` for the audit trail. This is why
Bob's account was seeded so much more heavily than Alice's: as protection
seller he's carrying contingent risk up to the full notional (at a 0%
recovery, the whole 10,000,000), on top of everything else settling
through that same account.

> **Fat-finger guard:** a `recoveryRate` outside `[0, 1]` (e.g. `3.0`,
> meant as "300%") is rejected outright, same reasoning as
> `observedRate`/`markToMarket` elsewhere.
>
> **Default instead of payout:** if the protection seller's account can't
> cover the payout, the same transaction archives the trade into
> `SettlementFailure` instead of `CreditEventSettlement` — same pattern as
> every other settlement default in this codebase, and still terminal
> either way (no successor `CDSTrade`). Bob is deliberately well-funded
> enough that this isn't reachable in the guided walkthrough itself
> (10,000,000 covers even a 0%-recovery payout); see
> `CDSTest.daml`'s `testCreditEventInsufficientDefaults` for this path
> exercised directly, or lower Bob's starting balance in `Setup.daml` to
> reach it here too.

> The oracle always submits as itself (`actAs: [oracle]`) with no `readAs` —
> it's a declared observer on every cash account, which is what lets it
> *see* them in order to move them. See [ARCHITECTURE.md](ARCHITECTURE.md),
> "Authority vs. visibility".

## TRS lifecycle

TRS's novation and termination work **exactly as walked in the IRS
section** — same choice names, same tri-party consent flow — because
`TRSTrade` implements the identical `Lifecycle.daml` interfaces
`IRSTrade`/`CDSTrade` do. Not re-walked here either; this section covers
only what's new: **daily variation margin against the prior day's
level**, a **periodic reset that trues up only the residual**, and a
**corporate action** that rebases the index scale with no cash leg at
all.

`Alice-USD`/`Bob-USD` are again the SAME accounts the IRS and CDS
sections above just moved — the `TRSTrade` was seeded from the start
alongside the IRS and CDS trades, settling through the identical
`Bank`/`USD` (see "What `CashHolding` maps to" above). As noted above,
this section too **assumes a fresh `daml start` restart**, so the tables
below start from the clean seeded baseline again — **Alice-USD =
1,000,000, Bob-USD = 10,000,000** — rather than wherever the CDS section
happened to leave the shared account.

Funding throughout uses `observedRate = 0.036` (3.6%) against the SPX-TR
index — every step below advances the date by exactly **9 days**, the
smallest gap for which Act360's day fraction (`elapsedDays / 360`)
divides evenly at `Numeric 10` (`9 / 360 = 0.025` exactly); a plain 1-day
gap leaves a `~0.000008` rounding remainder in the funding amount — see
`TRSTest.daml`'s comment on this for the full explanation. On
10,000,000 notional that's `0.036 × 10,000,000 × 0.025 = 9,000` funding
per 9-day step. The seeded `initialIndexLevel` is `16650.0` — a round
number in the ballpark of where SPX-TR actually trades, picked for
demo-arithmetic convenience rather than a live quote (see "What's
intentionally simplified" below); every dollar amount that follows is a
pure ratio of notional, so it's identical regardless of the index's
absolute level — only the *percentage* moves matter.

### 1. Daily variation margin — index up

In the **oracle** window, open the live `TRSTrade` and exercise
**`SettleMargin`**:

- `asOf` = `2026-07-29` (9 days after the 2026-07-20 effective date)
- `indexLevel` = `18315.0` (+10% from the seeded 16650.0)
- `observedRate` = `0.036`

The contract measures the move against the **prior observation**
(`lastIndexLevel`, seeded from `initialIndexLevel`) and nets it against
accrued funding:

```
dailyPerf = (18315 − 16650) / 16650 × 10,000,000 = 1,000,000
funding   = (0.036 + 0) × 10,000,000 × (9/360) = 9,000
net       = 1,000,000 − 9,000 = 991,000  → Bob (payer) pays Alice (receiver)
```

| | Before | After |
|---|--:|--:|
| Alice-USD | 1,000,000 | **1,991,000** |
| Bob-USD   | 10,000,000 | **9,009,000** |

The recreated trade advances `lastIndexLevel` to `18315.0`,
`lastSettledDate` to `2026-07-29`, and `vmPaidSinceReset` to
`1,000,000` — `lastResetIndexLevel` stays at `16650.0`, untouched by a
daily VM.

### 2. Daily variation margin — index down (the compounding case)

On the current `TRSTrade`, exercise **`SettleMargin`** again:

- `asOf` = `2026-08-07` (9 days later)
- `indexLevel` = `16483.5` (exactly −10% of 18315.0)
- `observedRate` = `0.036`

```
dailyPerf = (16483.5 − 18315) / 18315 × 10,000,000 = −1,000,000
funding   = 9,000
net       = −1,000,000 − 9,000 = −1,009,000  → Alice pays Bob
```

| | Before | After |
|---|--:|--:|
| Alice-USD | 1,991,000 | **982,000** |
| Bob-USD   | 9,009,000 | **10,018,000** |

`vmPaidSinceReset` returns to **`0`** (`1,000,000 + (−1,000,000)`) — the
two daily moves paid equal and opposite amounts. But the index itself
is **not** back where it started: 16650 → 18315 → 16483.5 is a genuine
**−1%** over the two days, because each daily percentage move applies to
a *different* base (16650, then 18315) — the well-known daily-reset
compounding effect (see [ARCHITECTURE.md](ARCHITECTURE.md)'s TRS
section). The next step's reset is exactly what catches this gap.

### 3. Periodic reset (the residual true-up)

On the current `TRSTrade`, exercise **`SettleReset`**:

- `asOf` = `2026-08-16` (9 days later)
- `indexLevel` = `16483.5` (unchanged since the last VM)
- `observedRate` = `0.036`

Unlike `SettleMargin`, this measures the **cumulative** return since the
last reset (against `lastResetIndexLevel`, still `16650.0`) and pays
only what daily VM hasn't already exchanged:

```
cumulative = (16483.5 − 16650) / 16650 × 10,000,000 = −100,000
residual   = cumulative − vmPaidSinceReset = −100,000 − 0 = −100,000
funding    = 9,000
net        = −100,000 − 9,000 = −109,000  → Alice pays Bob
```

| | Before | After |
|---|--:|--:|
| Alice-USD | 982,000 | **873,000** |
| Bob-USD   | 10,018,000 | **10,127,000** |

The residual is exactly the compounding gap from step 2 — the two daily
VMs summed to zero, yet the index is genuinely down 1%, and the reset
pays precisely that difference, **never** the two days' performance a
second time. `lastResetIndexLevel` re-anchors to `16483.5`,
`vmPaidSinceReset` returns to `0`, ready for the next period.

> **Zero-residual check:** if a reset's `indexLevel` exactly matches
> what the daily VMs already trued up to, `residual` comes out to `0`
> and the reset moves only funding — see `TRSTest.daml`'s
> `testZeroResidualReset` for this exercised directly.

### 4. Corporate action (rebase, no cash)

The index provider announces a corporate action rescaling the published
level — e.g. an index reconstitution. In the **oracle** window, on the
current `TRSTrade`, exercise **`AdjustForCorporateAction`**:

- `actionType` = `IndexReconstitution`
- `asOf` = `2026-08-16` (same day is fine — no date-ordering guard here)
- `adjustmentFactor` = `0.01`

**No cash moves at all** — check Alice's and Bob's balances, unchanged.
Both `lastIndexLevel` and `lastResetIndexLevel` are rebased together
(`16483.5 → 164.835`), so the next settlement measures against the new
scale rather than booking a phantom ~99% loss. `lastSettledDate` is
**not** touched by a corporate action — only `SettleMargin`/
`SettleReset` advance it, so the next VM's funding still counts elapsed
days from `2026-08-16` (the reset three steps up), not from this rebase.

### 5. Daily variation margin — after the rebase

On the current `TRSTrade`, exercise **`SettleMargin`**:

- `asOf` = `2026-08-25` (9 days after the reset in step 3 — the
  corporate action didn't move `lastSettledDate`, so this is still a
  clean 9-day funding gap)
- `indexLevel` = `168.1317` (+2% of the rebased 164.835)
- `observedRate` = `0.036`

```
dailyPerf = (168.1317 − 164.835) / 164.835 × 10,000,000 = 200,000
funding   = 9,000
net       = 200,000 − 9,000 = 191,000  → Bob pays Alice
```

| | Before | After |
|---|--:|--:|
| Alice-USD | 873,000 | **1,064,000** |
| Bob-USD   | 10,127,000 | **9,936,000** |

A genuine +2% move against the rebased scale nets a clean 191,000 —
proof the rebase in step 4 didn't distort the next settlement, because
both anchors moved together. See `TRSTest.daml`'s
`testCorporateActionRebaseNoPhantomLoss` for this guarded directly, and
`testCorporateActionRebaseThenResetResidualCorrect` for confirming a
reset's residual still comes out right after a mid-period rebase too.

> **Guards:** `indexLevel ≤ 0`, `observedRate` outside `[0, 1]`, and — on
> `SettleMargin` only — a daily move greater than 50% (a fat-fingered
> `indexLevel`, mirroring IRS's MTM-vs-notional guard) are all rejected
> outright; `SettleReset` has no such daily-move bound, since a
> legitimate reset can span a large gap. `adjustmentFactor ≤ 0` is
> rejected on `AdjustForCorporateAction`, but there's deliberately no
> upper bound — a real rebase can be `×0.01` (the demo just used one).
>
> **Insufficient funds:** same as every other settlement in this
> codebase, a payer who can't cover a `SettleMargin`/`SettleReset` move
> archives the trade into `SettlementFailure` instead — see
> `TRSTest.daml`'s `testInsufficientFundsDefaultsFromSettleMargin`.

## What's intentionally simplified / left as TODO

### All three products

- Mark-to-market and rate fixings (including CDS's `recoveryRate` and
  TRS's `indexLevel`/`observedRate`) are trusted oracle inputs; there's
  no on-ledger cross-check of the number itself (a real deployment would
  use a signed/attested feed).
- No grace period: a payer who can't cover a settlement/margin/credit-event
  move fails immediately. Real CSAs allow a cure period (typically T+1)
  before failure-to-pay escalates to a formal ISDA Master Agreement Event
  of Default — what's modeled here (`SettlementFailure`) is closer to a
  single settlement fail than that broader, more consequential concept.
- `SettlementFailure` records the facts but doesn't route into any
  default-management / close-out-auction logic.
- `CashHolding` is a bespoke local template for dev purposes — a real
  deployment would use an external, standards-based token (Canton Token
  Standard / CIP-56) representing a tokenized deposit or similar. No
  formal verification, no ISDA SIMM/margin-methodology integration yet.
- Inital Margin is not posted

### IRS

- `SettleCashflow`'s day-count fraction is real date arithmetic
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

### CDS

- **Cash settlement only** — no physical delivery of the defaulted
  obligation, and no auction-based ISDA-standard recovery-rate discovery
  (a real CDS auction determines the recovery price via a dealer
  auction process; here the oracle just submits a number).
- **Credit event enum is a 3-item closed set** (`Bankruptcy` /
  `FailureToPay` / `Restructuring`) — real ISDA definitions distinguish
  further subtypes (e.g. Restructuring has multiple clause variants with
  different triggering conditions across jurisdictions).
- **No accrued-premium proration.** A real CDS settles accrued premium up
  to the credit event date as part of the close-out; this model's
  `SettleCreditEvent` moves only the contingent payout, with no
  reconciliation of premium accrued since the last `SettlePremium`.
- **No upfront-payment convention.** Standardized CDS (since the 2009 "Big
  Bang" protocol) trade with a fixed coupon (e.g. 100bps or 500bps) plus
  an upfront cash settlement to true up to the actual market spread; this
  model only has the running premium.
- **`referenceEntity` is a plain `Text`**, not a structured
  legal-entity/RED-code reference.

### TRS

- **Fixed (non-resetting) notional.** A real equity swap's notional can
  reset with the underlying at each period (following the number of
  index units, not a fixed dollar amount); this model keeps `notional`
  constant for the trade's life.
- **Total-return index by construction, so no separate dividend or
  withholding-tax modeling.** `indexName`'s published level already
  reflects reinvested dividends — a real deployment settling against a
  *price*-return index (not total-return) would need a genuine
  `AccrueDividend`-style choice to true up the difference; that's
  explicitly out of scope here since "SPX-TR"-style total-return indices
  don't need it.
- **Simple, non-compounded daily funding accrual** — each `SettleMargin`/
  `SettleReset` call accrues `(observedRate + spread) × notional ×
  dayCountFraction` once per call, rather than the market-standard daily
  compounding of an overnight rate (e.g. compounded SOFR) within the
  accrual period.
- **Corporate actions as a single multiplicative factor** — real ISDA
  2002 Equity Derivatives Definitions machinery distinguishes many
  corporate-action types (extraordinary dividend, merger, tender offer,
  nationalization, delisting, ...) each with its own adjustment
  methodology; this model reduces all of them to one
  `adjustmentFactor : Decimal` rebasing both index anchors. Heavier
  ISDA-adjacent outcomes — substituting the reference index entirely
  (`SubstituteReference`), reducing exposure and settling the cancelled
  portion (`PartialCancelAndPay`), or a calculation-agent-determined
  early close-out (`CancelAndPay`) — aren't modeled at all, not even as
  stubs; `AdjustForCorporateAction`'s multiplicative rebase is the only
  corporate-action response this codebase has.
- **`AdjustForCorporateAction` is controlled by `oracle`, not a separate
  Calculation Agent party.** Real ISDA practice distinguishes a party
  that reports objective market data from the Calculation Agent, who
  makes discretionary determinations like how to rebase for a corporate
  action — a real distinction this codebase collapses into one party for
  simplicity. Relatedly, the choice only prevents "phantom VM" (a stale
  pre-event anchor compared against a post-event level producing a
  catastrophic false margin call) if the oracle actually calls it
  *before* the next `SettleMargin`/`SettleReset` observes a post-event
  level — there's no on-ledger way to detect a forgotten rebase step; see
  [ARCHITECTURE.md](ARCHITECTURE.md)'s TRS section for this traced
  through in full.
- **`indexName` is a plain `Text`**, not a structured index/ticker
  reference.
- **The seeded `initialIndexLevel` (16650.0) is a round approximation**
  in the ballpark of where SPX-TR actually trades, chosen so every demo/
  test number stays an exact, terminating decimal (see the day-count
  note above) — not a live quote, and not meant to track the real index
  day to day.
- **Early termination pays no close-out amount.** `ProposeTermination`/
  `AcceptTermination` end the trade with no final mark-to-market
  settlement — in practice, a trade would settle a final `SettleReset`
  first (parallels CDS's no-accrued-premium-on-termination note above).

## Goals

See `NEXT_STEPS.md` for the prioritized build list.
