# Next Steps

Rough priority order for building this out. Earlier items unblock later
ones.

## 1. Get the DAML side compiling and tested — DONE

- [x] `daml build` on the modules; fixed compile errors.
- [x] `Test.daml` using **Daml Script** covering the full happy path
      (propose → accept → `SettlePeriod` moving cash → `SettleMargin`
      moving cash) end-to-end in a `Script`.
- [x] Negative-path script tests: a party can't self-submit a settlement
      (only the oracle can); insufficient funds → `IRSDefaulted`;
      out-of-band rate and oversized mark-to-market are rejected.
- [x] `Setup.daml` init-script seeds parties, Navigator/JSON API users,
      funded accounts, and a live trade automatically on `daml start`.

## 2. Stand up a real two-participant local topology

- [ ] Move from `daml start`/sandbox (single participant) to a proper
      local Canton setup: two participant configs + one embedded domain
      config, per `ARCHITECTURE.md`'s deployment notes.
- [ ] Bootstrap the domain, connect both participants, allocate PartyA
      and PartyB on separate participants (not both on one — the point is
      to exercise real cross-participant view routing).
- [ ] Decide where the Oracle and Bank parties live, and confirm the
      oracle-as-observer visibility on `CashHolding` still resolves when
      accounts and oracle are on different participants (may need explicit
      disclosure, or co-hosting — see "Authority vs. visibility").
- [ ] Upload the DAR to all participants.
- [ ] Re-run the propose/accept + settlement flow against this topology.

## 3. Auth for local dev — PARTIALLY DONE

- [x] Unsafe HS256 shared-secret tokens work against the sandbox's JSON
      API (see the `curl` demos). Oracle submits with `actAs: [oracle]`
      and needs no `readAs` (it's an observer on the cash accounts).
- [ ] Configure the participants to actually *enforce* HS256 auth (the
      sandbox currently runs with `authConfig=None`).
- [ ] (Later) swap in a local Keycloak instance for something closer to
      production OIDC.

## 4. Minimal TypeScript client

- [ ] `daml codegen js` to generate typed bindings from the DAR.
- [ ] Small Node/TS script using `@daml/ledger`:
      - Query the live `IRSTrade` and both `CashHolding` balances.
      - Stream the trade via WebSocket and log settlement/margin events.
- [ ] An oracle-role script: submit `SettlePeriod` / `SettleMargin` and
      observe the atomic cash movement (and the `IRSDefaulted` path).

## 5. Off-ledger automation stub (the oracle service)

- [ ] A small scheduled service acting as the oracle that:
      - Queries live `IRSTrade`s.
      - For dev, computes a placeholder rate fixing / mark-to-market
        instead of a real Strata/pricing-engine call.
      - Submits `SettlePeriod` on schedule and `SettleMargin` on a daily
        (or intraday) mark — the concrete illustration of "nothing fires
        on its own, something has to notice and submit."
- [ ] Handle the `IRSDefaulted` outcome (alerting / downstream routing).

## 6. Stretch goals (only after 1–5 are solid)

- [ ] Replace the placeholder `dayFraction` in `SettlePeriod` with real
      day-count-fraction logic (using the `DayCountConvention`/`Frequency`
      already on the legs).
- [ ] Model `CashHolding` closer to a real token standard shape (Canton
      Token Standard / CIP-56) instead of the bespoke local template.
- [ ] Route `IRSDefaulted` into a close-out / auction workflow instead of
      just recording the facts. Add a cure period before default triggers.
- [ ] Add a bucketed-DV01-style tolerance check as a guard before allowing
      `Novate`, as a toy stand-in for a compression/risk-limit check.
- [ ] Sequence diagram or short doc showing the actual view/confirmation
      traffic captured from a real two-participant run.
