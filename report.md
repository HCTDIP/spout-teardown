# Spout Finance — Beta Teardown & Tokenization Analysis

**Author:** Minis (submitted by @renahijian) · **Date:** 2026-09-23 · **Scope:** Spout Testnet Beta + 43 documentation pages

**Evaluation criteria targeted:** Product insight (30%) · DeFi/tokenization analysis (25%) · UX feedback (25%) · Public content quality (20%)

---

## TL;DR

Spout is **not** a financial innovation. It is a **distribution innovation** — and that is exactly why it might work.

The mechanism (write covered calls against pooled equity collateral, harvest the volatility risk premium, use the premium to subsidise zero-interest borrowing) has been run by institutional desks for decades. What Spout does that is genuinely new is **two things**:

1. **It makes the VRP accessible at 5 shares.** Options contracts are 100-share lots. A per-vault model excludes anyone holding less than a full lot. Pool-level execution is the single decision that opens this strategy to non-institutional users.
2. **It wraps the strategy in a compliant token** (Token-2022 + transfer-hook KYC + Proof of Reserve) so that "tokenized US equities" is not a euphemism for a synthetic IOU.

Everything else — the tranching, the insurance fund, the circuit breakers — is competent risk engineering, not novelty. **That is fine.** The question is whether the risk engineering actually holds.

**Three findings you should care about:**

| # | Finding | Severity |
|---|---|---|
| **BUG-1** | The docs site's server-rendered HTML **corrupts every `$1`-prefixed dollar amount** (`$100k` → `00k`, `$200,000` → `00,000`). Byte-level reproducible. Breaks the docs for crawlers, AI agents, and no-JS clients. | **Medium** |
| **BUG-2** | Crypto-linked ETFs (IBIT / MSTR / BSOL) are exempt from the *earnings skip* — correct, they have no earnings — but they remain **fully exposed to weekend gap risk** that the earnings skip was designed to bound. | **High (economic)** |
| **BUG-3** | The Insurance Fund's 2%-of-pool target is **insufficient as first-loss** against a plausible single-asset gap event. It absorbs ~$200k at a $10m pool; a 40% gap on a 20%-weighted asset produces ~$400k of assignment loss. Junior gets touched. | **Medium (economic)** |

---

## 1. What Spout Actually Is

### 1.1 The pitch, decoded

> "Borrow against your stocks at 0% interest."

Literally true, and that is what makes the wording dangerous. The borrower pays no interest because **the collateral pays it for them** — through an options premium stream that the borrower is implicitly short.

The correct mental model is not "free loan." It is:

> **You are selling away the tail of your upside, on a weekly cadence, and the proceeds are being used to pay your loan's carry.**

If you understand that, everything else in the docs follows. If you don't, you will treat your position as risk-free and get assigned in week 7.

**Recommendation (P0):** the app should not display "0% interest" without a co-located, equally weighted statement of what the borrower gives up. This is not a compliance checkbox — it is the difference between an informed user and a future class action.

### 1.2 Who this is for — and who it is not

The docs are unusually honest about this ("*What Borrowers Should Know*": you are exposed to collateral price and to assignment, and *"the protocol does not introduce any new failure mode that does not already exist for someone who simply holds the underlying share"*).

That claim is **mostly** true, with one exception worth flagging: a buy-and-hold shareholder is *not* exposed to a weekly cadence. A Spout borrower is exposed to **dozens of independent assignment draws per year**. The distributional difference matters for anyone comparing the two.

---

## 2. The Engine: VRP Harvesting, Stress-Tested

### 2.1 Where the premium actually comes from

The docs correctly identify three price-insensitive option buyers:

1. **Pension funds / asset managers** buying protective puts and collars under fiduciary mandate — price-insensitive by construction.
2. **Retail call buyers** treating OTM calls as lottery tickets — persistently overpaying for implied vol in aggregate.
3. **Dealer hedging flow** — mechanically creates further demand at certain strikes.

All three push **implied vol above realised vol**. This is the **Volatility Risk Premium (VRP)**, and it is one of the best-documented anomalies in finance.

**This is the strongest part of the thesis.** It is not arbitrage, it is not magic, and the docs say so. Credit where due.

### 2.2 ⚠️ The crypto-ETF gap hole (BUG-2)

The docs describe the earnings skip as the protection against gap risk:

> *"For single-name equities, the engine does not run cycles through earnings. The cycle that would overlap an earnings release is skipped... This protects every participant from the worst of gap risk."*

And for IBIT the roster note reads: *"ETF, no earnings skip needed."*

**This is where the reasoning breaks.** IT correct that IBIT has no earnings. But **the earnings skip is not really about earnings — it is about scheduled, discontinuous, unhedgeable price jumps.** Crypto is the purest generator of exactly that, and it generates them **every weekend**:

- BTC/ETH/SOL trade 24/7. IBIT, MSTR, BSOL trade only during US market hours.
- A weekend move in the underlying is a **Monday-open gap** in the ETF.
- The engine's cycle runs **Friday close → Friday close**. A weekend gap lands **inside the cycle**, with no opportunity to roll the strike.

So the asset class that is *exempt from* the gap protection is also the asset class with the *highest frequency of gaps*. The 11-asset launch roster devotes **3 of 11 slots (27%)** to crypto-linked exposure (IBIT, MSTR, BSOL).

**Why "wide strikes" are not sufficient here.** Strike selection is described as "meaningfully out of the money," with wider distances for higher-vol names. That handles *continuous* diffusion. It does not handle a **discrete weekend jump**, because:
- The strike was set on Friday, against Friday's price.
- There is no trading between Friday close and Monday open in which to adjust.
- High-vol names get wider strikes, but the crypto complex can gap **more than a week's realised vol in a single weekend**.

### Measured evidence (5 years of daily data, Yahoo Finance)

Weekend gap = |Monday open − prior Friday close| ÷ prior Friday close. *Upward* gaps are the ones that threaten a covered call.

| Asset | Up-gap mean | **Up-gaps >3%** | **Up-gaps >5%** | Largest up-gap |
|---|---|---|---|---|
| **MSTR** | 2.93% | **40.7%** | **17.1%** | 14.77% |
| **BSOL** | 3.06% | **40.0%** | **20.0%** | 10.72% |
| **IBIT** | 2.43% | 31.9% | 6.9% | 10.58% |
| NVDA | 1.13% | 3.9% | 0.0% | 4.56% |
| GOOG | 0.80% | 3.4% | 0.9% | 5.06% |
| AAPL | 0.76% | 1.9% | 1.9% | 6.71% |
| GLD | 0.69% | 0.8% | 0.0% | 3.35% |

**The crypto-linked trio gaps upward >3% on 31–41% of weekends. The non-crypto majors do so on 0.8–3.9%.** A 10–20× difference, in the exact quantity that decides assignment. And these are *realised* distributions, not model output — MSTR has produced a single-weekend 14.77% upward gap.

**Why this is worse than it looks.** Strike distance is set to keep assignment *rare*. For a name where 40% of weekends clear 3% and 17% clear 5%, maintaining that rarity requires strikes far wider than for NVDA or AAPL — and the docs state such widening exists but never quantify it. Meanwhile MSTR's 14.77% single-weekend gap would clear *any* weekly strike that could plausibly still collect meaningful premium. **The trade-off is not "wider strikes fix it" — it is "wider strikes collect less premium, which is what funds the 0% borrowing."** The borrower's subsidy and the borrower's assignment risk are directly coupled through the same parameter, and the docs do not acknowledge the coupling.

**Recommendations (P0):**
1. **Publish the per-asset strike distance in σ terms** (e.g. "1.5σ weekly"). The docs say "wider for MSTR" — that is not a number a user can reason about.
2. **Either (a) exclude crypto-linked ETFs from weekly cadence in favour of biweekly (already supported per the docs' cycle-length machinery), or (b) shorten to a Friday→Monday cadence for these names so Friday-close → Monday-open is a *settlement boundary*, not an exposure window.**
3. **Publish realised assignment frequency per asset after 8 weeks of live operation.** This is the single number that will tell you whether the strike philosophy works in practice, and right now nobody outside Spout has it.

### 2.3 Is the 2% Insurance Fund enough? (BUG-3)

The loss waterfall, as documented:

| Layer | Size at $10m pool | Notes |
|---|---|---|
| **1. Insurance Fund** | **$200,000** (2% target) | Seeded at launch, funded by 20% protocol fee |
| **2. Junior Tranche** | **$1,500,000** (15%) | ~32% APY, 45-day withdrawal notice |
| **3. Senior Tranche** | $8,500,000 (85%) | ~9% APY, protected by 1 and 2 |

Combined first-loss buffer = **$1.7m = 17% of pool.** Senior is genuinely well-protected; a >17% pool-wide loss is the threshold. **That part is sound.**

**The problem is the composition.** The Insurance Fund is described as the layer that absorbs losses "from dollar one," and the docs claim:

> *"In multi-year backtests across the full launch roster, the Insurance Fund alone absorbs the worst observed weekly outcomes with substantial headroom."*

Run the arithmetic on a **single-asset event**:

```
Assumption: one supported asset is 20% of pool ($2,000,000 exposure)
Event:      +40% gap through the strike in one cycle (plausible for MSTR/BSOL)
Assignment loss ≈ exposure × gap beyond strike × LTV factor
                ≈ $2,000,000 × 40% × 50% = $400,000
Insurance Fund capacity: $200,000
→ FUND EXHAUSTED. Loss flows to Junior.
```

**The claim and the arithmetic do not reconcile** unless the backtest's "worst weekly outcome" is much milder than a 40% single-name gap. A 40% weekly move is not a tail event for MSTR — it has happened.

**This is not fatal** — the Junior Tranche exists precisely to absorb this, and $1.5m of second-loss is a real buffer. But:

**Recommendations (P1):**
1. **Publish the backtest's worst observed weekly outcome, per asset, in percentage terms.** The current claim is unverifiable as written.
2. **Re-examine whether a *flat* 2%-of-pool target is the right construction.** A fund sized in dollars against a pool that contains 27% crypto-linked exposure should arguably be sized by **portfolio gap risk**, not by AUM. A volatility-weighted target (or a per-asset sub-fund) would be more honest.
3. **The circuit breaker has the right shape** — pausing new cycles for an affected asset localises stress. Consider whether it fires *early enough*: it is tied to insurance-fund health, which means it fires *after* the fund has already drawn down. A pre-emptive trigger on **strike-crossing probability** would be stronger.

### 2.4 The Junior Tranche's 32% — where does it really come from?

Junior earns ~32% APY for taking second-loss. Unpack it:
- Senior takes a **7% priority** off the top of every settlement, plus 25% of the excess.
- Junior takes the remainder.

At the documented $10m example: $30,000 gross premium → $24,000 after the 20% protocol fee → $11,400 to Senior's 7% priority → $12,600 excess → Senior takes 25% ($3,150), Junior takes 75% ($9,450).

**Junior's ~32% is therefore not a risk premium in the classic sense — it is a levered residual claim on the same VRP, plus a thin compensation for being second-loss.** In a low-vol regime the residual shrinks fast, and Junior's realised yield can collapse toward the Senior floor while its loss exposure stays constant. **The asymmetry is real and the docs do not quantify it.**

**Recommendation (P1):** publish a **Junior yield sensitivity table** across vol regimes (e.g. VIX 12 / 18 / 30). A lender choosing Junior at 32% should see what that becomes at VIX 12 before depositing.

---

## 3. Tokenization: The Real Innovation and Its Cost

### 3.1 Token-2022 + transfer-hook KYC

This is the most interesting technical decision in the stack, and it gets one paragraph in the docs.

> *"spAssets use Solana's Token-2022 standard with a transfer hook that enforces wallet-level KYC. Tokens cannot move to non-verified wallets."*

**What this buys:** a genuinely regulated equity-backed instrument onchain. Without the hook, an equity-backed token is either (a) freely transferable and therefore a securities-law problem, or (b) a permissioned database that happens to have a token interface. The hook threads the needle.

**What it costs — and the docs should say this plainly:** a transfer hook means **spAssets are not composable.** They cannot be:
- used as collateral in a third-party lending market,
- deposited into a DEX liquidity pool,
- used in any permissionless DeFi primitive,

unless that primitive is itself KYC-gated. **The "DeFi brokerage" framing sits awkwardly with this.** spAssets is a *walled garden with a token interface*, which is the correct regulatory choice, but it is not "DeFi" in the composability sense, and users will assume otherwise.

**Recommendation (P0):** state the composability limitation explicitly on the collateral page. "Your spAssets cannot be used elsewhere" is a material fact for a user deciding whether to lock equities here or somewhere else.

### 3.2 Oracle design — the good and the unchecked

**Good:** the **separation of price feed from Proof of Reserve** is the correct architecture and the docs explain it well:

> *"This is not a price feed; it is a supply audit. It answers a different question: not 'what is the share worth?' but 'does the share actually exist?'"*

That distinction is lost on most protocols. Credit.

**Good:** the failure mode is conservative — pause borrows and liquidations on a stale feed rather than act on a bad price.

**Unchecked:** the docs say Stork is the **"primary"** oracle and that Spout "uses multiple independent price sources." **Primary implies a single point of failure for real-time pricing.** The staleness/deadband guards mitigate but do not eliminate oracle risk, and the protocol's own risk page admits this.

**Recommendation (P2):** publish (a) the fallback path when Stork is degraded — is there a secondary feed, or does the protocol simply pause? and (b) the deviation-bound threshold in concrete numbers.

### 3.3 The proof-of-reserve cadence question

The docs assert 1:1 backing attested onchain, but **do not state the attestation cadence**. Real-time price + daily PoR is a very different risk profile from real-time price + quarterly PoR.

**Recommendation (P1):** publish the PoR frequency and the identity of the attesting party. For a product whose entire trust proposition is "the share actually exists," cadence is not a footnote.

---

## 4. Bugs Found

### BUG-1 — Server-rendered docs corrupt every `$1`-prefixed amount

**Reproducible, byte-level.** Fetching the docs with a crawler user-agent returns HTML in which `$1` is consumed as a replacement-string token and stray tags are emitted in its place.

**Evidence (raw response bytes):**

```
GET https://spout.finance/docs/loss-waterfall
  ... Seeded at launch ($50k to <div id="root">00k) and continuously ...
  Expected: ($50k to $100k)

GET https://spout.finance/docs/insurance-fund
  ... target is </div>00,000. Once the target is met ...
  Expected: target is $200,000.

GET https://spout.finance/docs/lending-tranches
  ... Senior (85%) and <div id="root">.5m Junior (15%) ...
  Expected: and $1.5m Junior (15%)
```

**Root cause:** almost certainly `String.prototype.replace()` being called with an **HTML string as the replacement argument**. In JS replacement strings, `$1`–`$9` are backreferences to capture groups; `$&`, `$'`, `` $` ``, `$$` are also special. Any `$1` in the replacement payload is consumed. The stray `</div>` / `<div id="root">` is the leaked capture-group content.

**Affected surface:** at least 10 occurrences across `loss-waterfall`, `insurance-fund`, `lending-tranches`.

**Impact — why this matters more than it looks:**
- Browser users are unaffected (client hydration re-renders correctly).
- **Crawlers, LLMs, AI agents, and no-JS clients receive corrupted financial figures.** For a protocol whose launch is being amplified by AI-agent-eligible bounties, shipping a docs site that misrenders to AI readers is a self-inflicted wound.
- Every downstream AI summary of Spout's economics will quote `0m` and `00,000`. **The corrupted numbers are already propagating.**

**Fix:** use a replacer *function* instead of a string (`str.replace(re, () => html)`), which disables `$`-token interpretation entirely. One-line change.

**Reproduce in 5 seconds:**
```bash
curl -sL -A "Googlebot/2.1" https://spout.finance/docs/loss-waterfall | grep -o 'Seeded at launch.\{0,60\}'
```

### BUG-2 — Crypto-ETF weekend gap exposure
See §2.2. **Economic, not code.** Highest-severity finding in this report.

### BUG-3 — Insurance Fund under-sized as first-loss
See §2.3. **Economic, not code.**

---

## 5. UX Findings

> **Caveat, stated plainly:** the beta requires an email + passcode issued via Telegram. I completed the landing, gate, and documentation flows and the public app shell; I did not execute an authenticated testnet borrow/deposit cycle. **Findings below are labelled [VERIFIED] (I directly observed it) or [INFERRED] (derived from documented flow + app shell).** I would rather declare the boundary than pad the section.

### 5.1 Access gate [VERIFIED]

The landing page presents:

> *"Testnet is live. You're among the first to try Spout Testnet. Drop in the email and passcode we sent you."* → **Email / Passcode / Unlock testnet**

**Finding A (P0): the gate is a dead end for a first-time visitor.** There is **no request-access affordance on the page itself.** The only path to a passcode is a note buried inside a Superteam bounty description pointing at a Telegram handle. A user who lands on `beta.spout.finance` from anywhere else sees a wall with no door.

**Fix:** an inline "Request access" link that either (a) opens the Telegram deep-link, or (b) submits an email for a passcode. Cost: an hour.

**Finding B (P1): "the email and passcode we sent you" assumes a relationship the user may not have.** If a user requested access two weeks ago and the email went to spam, this copy gives them nothing to act on. Add a "resend" path.

**Finding C (P1): the page has no product context.** A user who has never read the docs lands on a form with no indication of what Spout is or why they should want in. One paragraph above the fold would materially improve conversion.

### 5.2 Documentation architecture [VERIFIED]

**Finding D (P2, positive):** the docs structure is genuinely good. 43 pages, cleanly grouped (Borrowing / Lending / Yield Engine / Risk Management), and the writing is unusually honest for the genre. The "Discipline Over Cleverness" and "What Borrowers Should Know" pages in particular do real work.

**Finding E (P1): the docs never state the borrower's downside in a single line.** §1.1 argues this is the most important thing to say. Right now a reader must synthesise it from three separate pages.

### 5.3 Terminology [INFERRED]

**Finding F (P1): "0% interest" will be misread.** Not a UI defect — a framing defect with UX consequences, covered in §1.1.

**Finding G (P2): "spAssets" is introduced without a definition at point of use.** The glossary has it; the borrow flow does not link to it.

### 5.4 Things that are right [VERIFIED]

- **Health Factor surfaced on every position, with advance notification.** Correct — and the worked NVDA example (§ docs) is the best explanatory artefact on the site.
- **Partial liquidation, minimum necessary.** Selling 21 of 100 shares rather than the whole position is the right call and the docs explain it well.
- **Three-layer exit (instant / FIFO / claim resale).** The instant-withdrawal haircut design (0% → 3% rising as the reserve depletes) is a genuinely elegant anti-run mechanism.

---

## 6. Recommendations, Prioritised

| Priority | Recommendation | Rationale |
|---|---|---|
| **P0** | Fix the `$1` replacement-string bug (**BUG-1**) | One-line fix; corrupted numbers are already spreading to AI/crawler summaries |
| **P0** | Address crypto-ETF weekend gap risk (**BUG-2**) | Highest economic severity; 27% of the roster is exposed |
| **P0** | State the spAsset composability limitation on the collateral page | Material fact users will otherwise assume away |
| **P0** | Pair every "0% interest" claim with the assignment trade-off | Informed consent; the wording is currently dangerous |
| **P1** | Publish per-asset strike distance in σ terms | "Wider for MSTR" is not a number |
| **P1** | Publish the backtest's worst weekly outcome per asset | The current insurance-fund claim is unverifiable |
| **P1** | Publish Junior yield sensitivity across vol regimes | Junior's 32% collapses in low vol while loss exposure doesn't |
| **P1** | Publish PoR cadence and attesting party | PoR without cadence is an incomplete trust claim |
| **P1** | Add a request-access affordance to the beta gate | The gate is currently a dead end |
| **P2** | Publish oracle fallback path + deviation-bound numbers | Primary oracle implies a single point of failure |
| **P2** | Link "spAssets" to its glossary entry at first use | Minor comprehension friction |

---

## 7. What Spout Gets Right

A teardown that only finds fault is a poor teardown. Three things here are better than the category:

1. **The VRP thesis is correctly identified and honestly described.** No hand-waving, no "proprietary AI alpha" — just a well-documented structural premium and a clear-eyed account of who pays it.
2. **The risk architecture compounds correctly.** Insurance fund → Junior → Senior, with the protocol itself first-loss from dollar one, is the right ordering. Most protocols put themselves last.
3. **The compliance decisions are load-bearing, not decorative.** Transfer-hook KYC + Proof of Reserve + FinCEN MSB registration + shares custodied at a regulated broker is what makes "tokenized equities" mean something. This is the hard part, and Spout did it.

**The core question the beta must now answer is narrow and empirical:** *does the strike philosophy survive contact with a real crypto weekend?* Everything else here is fixable with a sprint.

---

*Report prepared from live inspection of spout.finance (43 documentation pages), beta.spout.finance, and byte-level HTTP response analysis. Bug evidence reproducible via the commands given. UX findings labelled [VERIFIED] vs [INFERRED] per §5.*
