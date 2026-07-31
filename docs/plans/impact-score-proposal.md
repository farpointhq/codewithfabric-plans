# Engineering Impact Score — proposal for the Enterprise Reporting dashboard

**Author:** research agent · **Date:** 2026-07-30 · **Status:** proposal, no code changed

**Scope.** Replace the current `productivityScore` in `/api/reporting/productivity` with (a) an **Impact Index** built only from GitHub-obtainable signals, and (b) a **separate** efficiency view that relates that output to token spend. Covers formula, credit split for creators/mergers/reviewers, anti-gaming provisions, API cost, and a v1/v2 split.

**Evidence convention.** Every number below is either measured live in this session against `github.com` with Ryan's PAT (marked **[MEASURED]**), read out of the repo (**[CODE]**), or cited to a URL that was fetched (**[CITED]**). Anything I could not confirm is marked **NOT VERIFIED** and is never used as load-bearing.

---

## 0. Executive summary

1. **The current score is not miscalibrated, it is inverted.** It pays developers for *not* using Fabric. A developer shipping 3 merged PRs on 1M tokens scores **75/100 ("excellent")**; a developer shipping 196 merged PRs on 3.76B tokens scores **9/100 ("low")**. Worked below in §1.
2. **The root cause is putting tokens in the denominator of the output metric.** Output and cost are different questions and must be different columns. Every framework in the literature that survived scrutiny — SPACE, DORA, DX Core 4 — separates them.
3. **Proposed Impact Index**: a 4-term weighted composite (throughput, substance, breadth, collaboration), each term normalised against a *published industry median* so that **50 = median engineer, 100 = 2× median or better**. No tokens. Full formula in §3.
4. **Creator credit is already correct and squash-merging does not steal it** — GitHub search `author:` matches the PR author and a squash merge never changes it. The gap is that assists are invisible, not that credit is misassigned. Assists are **additive, never subtractive** (§4).
5. **Anti-gaming is structural, not exhortative** (§5): generated-file exclusion, per-file `max(add,del)`, per-PR churn cap, and — most importantly — **period-level churn aggregation**, which makes PR-splitting exactly neutral. Splitting diffs is the single documented real-world gaming response to this class of metric (Uber, via Orosz).
6. **Efficiency view becomes `cost per merged PR` in real dollars.** `DailyUsageSummary.costCents` already exists in the schema and the route currently ignores it **[CODE]**. DORA explicitly endorses "cost per accepted change" and explicitly condemns token-volume leaderboards **[CITED]**.
7. **The v2 data fetch is cheaper than what ships today, not more expensive.** [MEASURED] One GraphQL request returns **100 PRs with full diff stats, files, commits, reviews and `mergedBy` for 3 rate-limit points** out of a 5,000-point/hour budget. The current REST Search design burns a 30-request/minute budget and takes ~60s cold. A whole team-quarter costs ~**390 points worst case ≈ 8% of one hour's GraphQL budget**.

---

## 1. The current formula's failure mode, numerically

### 1.1 The formula as shipped [CODE]

`src/lib/reporting/productivity.ts`:

```
efficiency = (10·prsMerged + 5·issuesClosed + 3·commits + 2·reviews) / (tokens / 1e6)
score      = clamp(round(log10(efficiency + 1) · 50), 0, 100)
```

`commits` and `reviews` are hard-coded to `0` at the call site in `src/app/api/reporting/productivity/route.ts:159-160` **[CODE]**, so in production the numerator is `10·mergedPRs + 5·closedIssues`.

### 1.2 Solving the band boundaries for tokens-per-PR

Set `issuesClosed = 0`. Let `t` = tokens per merged PR. Then `efficiency = 10·10⁶/t`, and the four rating bands invert to:

| Rating | Score threshold | Required `efficiency` | **Required tokens per merged PR** |
|---|---|---|---|
| excellent | ≥ 75 | ≥ 30.62 | **< 326,600** |
| good | ≥ 50 | ≥ 9.00 | **< 1,111,111** |
| average | ≥ 25 | ≥ 2.162 | **< 4,624,700** |
| low | < 25 | < 2.162 | **≥ 4,624,700** |

*(Derivation: score ≥ S ⟺ log₁₀(e+1) ≥ S/50 ⟺ e ≥ 10^(S/50) − 1. Then t = 10⁷/e.)*

### 1.3 Why that is fatal

The only large-N published dataset of tokens-per-PR is Jellyfish's, over 7,548 engineers: **median developer ≈ 7.5M tokens per PR, top decile ≈ 69M** ([jellyfish.co/library/ai-token-usage-monitoring](https://jellyfish.co/library/ai-token-usage-monitoring/), 20 Jul 2026) **[CITED — vendor-published, correlational]**.

**The "average" band boundary (4.62M tokens/PR) sits below the industry median (7.5M).** A perfectly ordinary AI-assisted developer is classified `low` by construction. The top decile scores:

```
e = 10e6 / 69e6 = 0.1449   →  log10(1.1449)·50 = 2.94  →  3/100
```

### 1.4 The headline exhibit

| Developer | Merged PRs / quarter | Tokens | `efficiency` | **Score** | Rating |
|---|---|---|---|---|---|
| Light user | 3 | 1.0M | 30.00 | **75** | excellent |
| Light user | 1 | 0.1M | 100.00 | **100** (clamped) | excellent |
| Heavy user | 77 | 3.76B | 0.2048 | **4** | low |
| Heavy user | 196 | 3.76B | 0.5213 | **9** | low |

Two consequences follow directly:

- **A Fabric-sold dashboard currently tells enterprise buyers that the engineers using Fabric most are their worst performers.** That is a commercial problem before it is an engineering one.
- The `lowProductivityRule` and `zeroOutputRule` alerts in `productivity.ts:112-150` **[CODE]** emit the sentence *"token usage doesn't reflect productivity"* by name, at the person, keyed off this number. Those alerts are firing on an artefact.

### 1.5 The structural error, stated once

Dividing output by cost produces a **ratio**, and a ratio is maximised by driving the denominator to zero. The current score's global maximum is *"ship one small thing and never open the product."* You cannot patch the weights out of that; the operator has to change.

> SPACE, on precisely this: *"Activity metrics alone do not reveal which of these is the case, so they should never be used in isolation either to reward or to penalize developers. Even straightforward metrics such as number of pull requests, commits, or code reviews are prone to errors…"* — Forsgren, Storey, Maddila, Zimmermann, Houck & Butler, *ACM Queue* 19(1), 2021, [queue.acm.org/detail.cfm?id=3454124](https://queue.acm.org/detail.cfm?id=3454124) **[CITED]**

---

## 2. What the literature actually supports

Condensed to the claims this proposal leans on. Fuller citations inline.

### 2.1 Never one number, never called "productivity"

- **SPACE**: *"The most important takeaway from exposing these myths is that productivity cannot be reduced to a single dimension (or metric!)"*; recommends metrics across **at least three** dimensions, and *"at least one of the metrics include perceptual measures such as survey data."* [queue.acm.org/detail.cfm?id=3454124](https://queue.acm.org/detail.cfm?id=3454124) **[CITED]**
- **Google** (Jaspan & Sadowski, *Rethinking Productivity in SE*, 2019): *"there is no metric that adequately captures the full space of developer productivity and that attempting to find one is counterproductive."* And on composites specifically: *"if a variety of factors … are compressed into one number … it will not be immediately clear why one patch scores 24 and another one scores 37."* [link.springer.com/chapter/10.1007/978-1-4842-4221-6_2](https://link.springer.com/chapter/10.1007/978-1-4842-4221-6_2) **[CITED]**

**Design consequence.** Rename the field. It is an **Impact Index** — a bounded measure of *shipped change volume and collaboration*, one tile among several. It is not "productivity", and the UI must not say productivity. Google's composite critique is answered by making every term **individually inspectable in the tooltip** (§3.5), so "why 74" always has an answer.

### 2.2 Individual scoring is the sharp edge

Unanimous across sources, including the vendors who sell the dashboards:

- **DORA**: *"The four key metrics should be used at the application and service levels … The metrics should be used to visualize your efforts in continuous improvement and not to compare teams — and certainly not to compare individuals."* (2024 State of DevOps, p.74) [PDF](https://dora.dev/research/2024/dora-report/2024-dora-accelerate-state-of-devops-report.pdf) **[CITED]**. Blunter, 2026: *"At DORA, we believe company-wide aggregates are meaningless, and individual scores are harmful."* [dora.dev/insights/finding-balance-in-the-era-of-tokenmaxxing](https://dora.dev/insights/finding-balance-in-the-era-of-tokenmaxxing/) **[CITED]**
- **DX** on their own headline metric: *"It is critical that this metric is never used at the individual level or tied to performance evaluations."* [getdx.com/research/measuring-developer-productivity-with-the-dx-core-4](https://getdx.com/research/measuring-developer-productivity-with-the-dx-core-4/) **[CITED]**
- **Google** (*SWE at Google*, ch. 7, fn. 9): *"If productivity metrics are used for performance reviews, engineers will be quick to game the metrics… The only way to make these measurements work is to let go of the idea of measuring individuals."* [abseil.io/resources/swe-book/html/ch07.html](https://abseil.io/resources/swe-book/html/ch07.html) **[CITED]**
- **Meta** on its own official metric: *"We introduce Diff Authoring Time (DAT) as a key metric for assessing development, not developer, productivity."* [arxiv.org/html/2503.10977v1](https://arxiv.org/html/2503.10977v1) **[CITED]**
- **SPACE** adds a legal note: *"(In some countries, reporting on individual productivity isn't legal.)"* **[CITED]**

**Design consequence.** This dashboard *is* per-person — that is the product, and I am not going to pretend otherwise. The mitigation is that the **team roll-up is the default view**, the per-person table is behind a click, and the page carries a standing caption naming what the index does not measure. §5.11 gives the exact provisions.

⚠️ **Flag for Ryan:** SPACE's legality note is real and I did not research which jurisdictions. If any enterprise customer has EU/German works-council exposure, per-person output reporting needs a legal read before it ships to them. **NOT VERIFIED** — flagged, not researched.

### 2.3 The one metric the industry converged on is merged-PR throughput

- **DX Core 4** headline speed metric is **Diffs per Engineer (DPE)**, defined verbatim as *"The number of pull requests merged each week, per engineer."* Published benchmarks over 500+ companies: **P50 ≈ 3.0–3.8/week**, P90 ≈ 3.4–5.5/week depending on segment. [getdx.com/research/benchmarks](https://getdx.com/research/benchmarks/) **[CITED]**
- **Cui, Demirer, Jaffe, Musolff, Peng & Salz** used weekly merged PRs as the outcome variable across 4,867 developers in three field experiments, published in *Management Science*: *"A main outcome of interest is 'pull requests', which can be thought of as a unit of work for software developers. Within an organization, the scope of a pull request is likely to remain relatively stable over time."* [economics.mit.edu draft](https://economics.mit.edu/sites/default/files/inline-files/draft_copilot_experiments.pdf) · [DOI 10.1287/mnsc.2025.00535](https://pubsonline.informs.org/doi/10.1287/mnsc.2025.00535) **[CITED]**
- **Microsoft's 2026 CLI-agent study** (tens of thousands of engineers) used merged PRs per engineer and found **+24.0% [95% CI +14.5%, +33.7%]** — while stating their own caveat: *"merged PRs are an imperfect proxy for throughput and reward small, frequent PRs."* [arxiv.org/html/2607.01418](https://arxiv.org/html/2607.01418) **[CITED]**

**Design consequence.** Merged-PR throughput is the correct primary term and it has a **published median to normalise against**, which is what makes the output score interpretable rather than arbitrary. The Microsoft caveat — that it rewards small frequent PRs — is exactly what the substance term and the caps in §3 exist to counterbalance.

### 2.4 What LOC does and does not measure — be honest about both

The lazy version of this argument ("Dijkstra said LOC is bad") is weaker than the true one.

- **Dijkstra, EWD1036 (1988)**: *"if we wish to count lines of code, we should not regard them as 'lines produced' but as 'lines spent': the current conventional wisdom is so foolish as to book that count on the wrong side of the ledger."* [cs.utexas.edu/~EWD/transcriptions/EWD10xx/EWD1036.html](https://www.cs.utexas.edu/~EWD/transcriptions/EWD10xx/EWD1036.html) **[CITED]**
- **Graves, Karr, Marron & Siy (2000)**, IEEE TSE: *"the number of times code has been changed is a better indication of how many faults it will contain than is its length"*, and *"once deltas have been taken into account, lines are not capable of improving the prediction."* [PDF](https://cs.uwaterloo.ca/~m2nagapp/courses/CS846/1171/papers/graves_tse98.pdf) **[CITED]**
- **Nagappan & Ball (2005)**, ICSE, over 44.97M LOC of Windows Server 2003 — the honest counterpoint: **absolute** churn measures give R² = 0.052; **relative** churn measures give R² = 0.811 and classify fault-prone binaries at 89.0%. [PDF](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/icse05churn.pdf) **[CITED]**

**Synthesis, and it is the justification for every cap in §3:** churn is a well-validated predictor of **defects — i.e. cost**. Nobody has shown it predicts value. So churn may enter the index as *evidence that substantive work occurred*, with **strongly diminishing returns and a hard ceiling**, and it may never enter linearly. That is Dijkstra's "lines spent" restated as a regression result.

⚠️ **Do not use** the Bill Gates "measuring progress by lines of code is like measuring aircraft building progress by weight" quote. No primary source exists for it; the trail dead-ends in quote aggregators. **NOT VERIFIED.** If a rhetorical hook is wanted, Steve Ballmer's on-the-record K-LOC passage in the PBS *Triumph of the Nerds* transcript ([pbs.org/nerds/part2.html](https://www.pbs.org/nerds/part2.html)) makes the same point and is citable.

### 2.5 The gaming evidence base is thinner than people assume — say so

There is **no peer-reviewed quantitative study documenting PR-count or LOC gaming at a named company.** What exists is named-practitioner testimony, and it is consistent:

- **Uber, diff-splitting** (Orosz, who was an Uber EM), after an eng-metrics dashboard launched: *"the lines of code did not change drastically, but the CI systems saw a visible increase in utilization … people started to create far, far more diffs, which put a much larger load on the continuous integration (CI) systems, and increased CI costs."* [newsletter.pragmaticengineer.com/p/measuring-developer-productivity](https://newsletter.pragmaticengineer.com/p/measuring-developer-productivity) **[CITED — practitioner testimony]**
- **Meta, DDM** (James Everingham, ex-Meta DevInfra): *"DDM: diffs per developer per month. It showed up everywhere."* … *"It became tempting to slice work thinner, to favor low-risk changes that count, and to avoid the gnarly, high-leverage problems."* [devinterrupted.substack.com](https://devinterrupted.substack.com/p/what-metas-diffs-per-developer-metric) **[CITED — secondhand-named]**
- **GitPrime's own 2017 help page**, written by its CEO: *"TL;DR The point is to game the system." … "Active Days: This one is easy — just push your work every day."* — [2017 capture](https://web.archive.org/web/20170325145422/http://help.gitprime.com/general/cant-you-just-game-the-metrics). By 2023 Pluralsight had rewritten the same page to *"We don't want you to game metrics like this… Don't start rubber-stamping PRs to increase this metric."* [2025 capture](https://web.archive.org/web/2025id_/https://help.pluralsight.com/help/cant-you-just-game-the-metrics) **[CITED]**

**Design consequence.** The one gaming behaviour with two independent named accounts is **diff-splitting**. It gets the strongest structural countermeasure in this proposal (§5.4). The rubber-stamp risk gets a discount, on the vendor's own published advice (§4.4).

⚠️ **NOT VERIFIED:** I found no GeePaw Hill response to McKinsey. The sub-researcher pulled geepawhill.org's full post sitemap (397 URLs) and found **zero posts published between Feb 2023 and Feb 2024**, bracketing the entire debate. Do not cite it.

⚠️ **Premise correction:** **LinearB has no "Impact" metric.** "Impact" is GitPrime/Pluralsight. LinearB's size metric is *PR Size*. If the brief assumed otherwise, it was wrong.

### 2.6 Token spend as a denominator: use the phrase DORA already blessed

DORA has publicly named the failure mode this dashboard risks being accused of:

> *"measuring software development is more complex than a single metric, especially one like token consumption that prioritizes quantifying activity over examining actual outcomes"* … developers *"running autonomous agents on nonsense projects simply to calibrate their spend and stay above the company average, or climb their way to the top of the charts."*
> — [dora.dev/insights/finding-balance-in-the-era-of-tokenmaxxing](https://dora.dev/insights/finding-balance-in-the-era-of-tokenmaxxing/) **[CITED]**

And in the same piece DORA recommends the alternative: pair delivery metrics with **"cost per accepted change and code rework rates."**

**That is this proposal's efficiency metric, under the name its most credible critic gave it.** Adopting DORA's own phrase pre-empts the "you built tokenmaxxing" objection before it is raised.

Precedent that the metric is shippable: **LinearB** ships an *AI Cost per Pull Request* report ([helpdocs](https://linearb.helpdocs.io/article/w1yl8ejm8p-ai-cost-and-token-consumption)); **DX** ships *"estimated cost per PR"* ([getdx.com/blog/meet-the-new-ai-cost-management-report](https://getdx.com/blog/meet-the-new-ai-cost-management-report/)); **Vantage** publishes the formula outright — `Cost per PR = Monthly AI spend / PRs merged` — with the framing line **"Raw cost without a denominator is meaningless"** ([vantage.sh/blog/agentic-coding-efficiency](https://www.vantage.sh/blog/agentic-coding-efficiency)) **[CITED, all vendor]**.

---

## 3. The Impact Index

### 3.1 Design rules, each traceable to §2

| # | Rule | Source |
|---|---|---|
| R1 | Tokens never appear in the output score | §1.5 |
| R2 | Primary term is merged-PR throughput, normalised to a published median | DX Core 4 §2.3 |
| R3 | Churn enters with diminishing returns and a hard cap, never linearly | Nagappan & Ball §2.4 |
| R4 | Churn is summed **over the period**, so PR-splitting is neutral | Uber/Meta §2.5 |
| R5 | Collaboration is a scored term, not a footnote | SPACE "C" dimension §2.1 |
| R6 | Every term is individually displayed | Google's composite critique §2.1 |
| R7 | 50 = industry median, 100 = 2× median. Bands are meanings, not magic | GSM traceability, §2.1 |

### 3.2 Per-PR quantities

For each merged PR `p` credited to developer `d` in the reporting period:

**Effective churn.**
```
C_p = min( Σ_{f ∈ files(p), f ∉ EXCLUDED}  max(f.additions, f.deletions),  CHURN_CAP )
CHURN_CAP = 3000
```

- `max(additions, deletions)` **per file**, not the sum. A modified line appears in a unified diff as both `+1` and `−1`; summing double-counts every edit. This is Pluralsight's HALOC rule verbatim: *"In each hunk, Flow looks at the number of insertions and the number of deletions, then takes the higher number as the HALOC for the hunk."* ([archived help page](https://web.archive.org/web/2025id_/https://help.pluralsight.com/help/haloc)) **[CITED]**. GitHub's API gives per-file, not per-hunk granularity, so per-file `max` is the available analogue — strictly more conservative than summing, strictly less precise than HALOC. State that limitation in the docs.
- `EXCLUDED` is defined in §5.1.

**Effective breadth.**
```
F_p = min( |{ f ∈ files(p) : f ∉ EXCLUDED ∧ f.additions + f.deletions > 0 }|,  FILE_CAP )
FILE_CAP = 40
```

Files-touched is one of the six inputs GitPrime's Impact metric named — *"The number of files affected"*, *"The surface area of the change"* ([archived](https://web.archive.org/web/2019id_/https://blog.gitprime.com/impact-a-better-way-to-measure-codebase-change/)) **[CITED]**. Note that GitPrime/Pluralsight **never published the weights**, which is exactly the failure this proposal avoids: ours are in this document.

### 3.3 Period-level terms

For developer `d` over `W` weeks (`W = spanDays / 7`, floor 1):

```
N   = Σ credited merged PRs                 (creator credit, §4)
C   = Σ_p C_p    over credited PRs
F   = Σ_p F_p    over credited PRs
A   = Σ assistPoints earned                 (§4)
```

Four normalised terms, each of the identical shape `min(x / anchor, 2) / 2` so each lands in **[0, 1]** with **median = 0.5**:

| Term | Rate | Anchor | Normalised |
|---|---|---|---|
| **T** Throughput | `N / W` | `A_T = 3.5` merged PRs/week | `T̂ = min(N/W / 3.5, 2) / 2` |
| **S** Substance | `C / W` | `A_S = 400` effective lines/week | `Ŝ = min(C/W / 400, 2) / 2` |
| **B** Breadth | `F / W` | `A_B = 25` effective files/week | `B̂ = min(F/W / 25, 2) / 2` |
| **R** Collaboration | `A / W` | `A_R = 2.0` assist points/week | `R̂ = min(A/W / 2.0, 2) / 2` |

**Anchor provenance — this is what makes the score mean something:**

- `A_T = 3.5` is DX Core 4's published **P50 diffs per engineer per week** across 500+ companies (tech <100 eng: 3.8; tech 100–500: 3.5; tech 2500+: 3.35). [getdx.com/research/benchmarks](https://getdx.com/research/benchmarks/) **[CITED]**
- `A_S = 400` derives from `A_T` × LinearB's "Good" PR-size band. LinearB's published bands over **8.1M pull requests from 4,813 teams**: PR Size Elite < 100, Good 100–155, Fair 156–228, Needs focus > 228 lines. [linearb.io/resources/engineering-benchmarks](https://linearb.io/resources/engineering-benchmarks) **[CITED]**. 3.5 PRs/wk × ~115 lines ≈ 400.
- `A_B = 25` ≈ 3.5 PRs/wk × 7 files, consistent with LinearB's operational target of *"< 100 net LOC or ≤ 5 files"* per PR ([helpdocs](https://linearb.helpdocs.io/article/drons87w53-managing-large-pull-requests)) **[CITED]**.
- `A_R = 2.0` is **not** externally anchored — no vendor publishes an assists-per-week benchmark. **NOT VERIFIED**, set by judgement, and see the calibration warning in §4.6.

**All four anchors must be config, not constants**, and the dashboard should offer "benchmark: industry / this team's trailing 4 quarters" — Haystack's rule: *"Each team should be improving and compared to their own history rather than other teams."* ([support.usehaystack.io](https://support.usehaystack.io/en/articles/4540688-software-engineering-metrics-faqs)) **[CITED]**

### 3.4 The composite

```
ImpactIndex = round( 100 × ( 0.40·T̂  +  0.30·Ŝ  +  0.10·B̂  +  0.20·R̂ ) )
```

| Weight | Term | Why this weight |
|---|---|---|
| 0.40 | Throughput | The only term with a large-N validated benchmark and the only one used as an outcome variable in peer-reviewed causal work (§2.3). Highest confidence ⇒ highest weight. |
| 0.30 | Substance | Ryan's explicit requirement (LOC + files). Weighted below throughput because churn is a validated predictor of *defects*, not value (§2.4). It exists to stop 40 one-line PRs outranking a real feature — a counterweight, not a driver. |
| 0.10 | Breadth | Highly collinear with churn; earns a small weight so "touched 40 files" is not free but cannot dominate. |
| 0.20 | Collaboration | SPACE's "C" dimension and Dan North's *"Your highest-value developers are 10x by enabling other developers"* ([dannorth.net/blog/mckinsey-review](https://dannorth.net/blog/mckinsey-review/)) **[CITED]**. Also Ryan's assist requirement. Held to 0.20 because review culture varies enormously between orgs — see the calibration warning in §4.6. |

**Reading the number:** 50 = an engineer at the published industry median on all four axes. 100 = at or above 2× median on all four. 0 = shipped nothing. Bands become `≥75 strong / ≥50 at-benchmark / ≥25 below-benchmark / <25 low` — and the word "productivity" appears nowhere.

### 3.5 Mandatory disclosure (answering Google's composite critique)

Every cell must expose the decomposition on hover, e.g.:

```
Impact Index 74
  Throughput   5.9 PRs/wk vs 3.5 benchmark   → 0.85  × 0.40 = 0.338
  Substance   2488 lines/wk vs 400 benchmark → 1.00  × 0.30 = 0.300
  Breadth       41 files/wk vs 25 benchmark  → 0.83  × 0.10 = 0.083
  Collaboration 0.4 assists/wk vs 2.0        → 0.11  × 0.20 = 0.021
  Excluded from churn: 11.8% (lockfiles, dist/, generated/)
```

This is non-negotiable. Jaspan & Sadowski's objection to composites is that *"it will not be immediately clear why one patch scores 24 and another one scores 37"* **[CITED]**. A visible decomposition is the entire answer.

### 3.6 Worked example — three PRs

Using the three cases from the brief.

---

**PR A — 10-line bugfix.** 1 file, `src/lib/reporting/productivity.ts`, `+6 / −4`.

| Step | Value |
|---|---|
| Files after exclusion | 1 |
| Per-file `max(6, 4)` | 6 |
| `C_p` (cap 3000) | **6** |
| `F_p` (cap 40) | **1** |
| Period contribution | `N += 1`, `C += 6`, `F += 1` |

---

**PR B — 2,000-line feature, 40 files.** Raw: `additions 2000`, `deletions 440`, `changedFiles 40`. Composition: 38 source files (`Σ max = 1820`), plus `package-lock.json` (+240/−60) and `dist/bundle.min.js` (+60/−0).

| Step | Value |
|---|---|
| Excluded files | `package-lock.json`, `dist/bundle.min.js` → 2 files, 300 raw lines |
| `Σ max(add, del)` over survivors | 1,820 |
| `C_p` (cap 3000, not binding) | **1,820** |
| `F_p` (cap 40, not binding) | **38** |
| Period contribution | `N += 1`, `C += 1820`, `F += 38` |

**PR B is worth 303× PR A on churn and 38× on breadth — before the log-free caps and before the period-level normalisation flatten it.** After normalisation (§3.3), that ratio compresses to whatever the developer's *weekly rate* looks like against the benchmark, which is the point: PR B alone is 4.5 weeks of benchmark substance (1820 / 400).

---

**PR C — 50k-line lockfile / generated PR.** 6 files:

| File | additions | deletions | `max` | Excluded? |
|---|---|---|---|---|
| `package-lock.json` | 24,000 | 23,000 | 24,000 | ✅ lockfile |
| `pnpm-lock.yaml` | 1,200 | 1,100 | 1,200 | ✅ lockfile |
| `src/generated/schema.ts` | 600 | 500 | 600 | ✅ `**/generated/**` |
| `dist/app.bundle.js` | 300 | 280 | 300 | ✅ `**/dist/**` |
| `package.json` | 3 | 3 | 3 | — |
| `src/config.ts` | 8 | 2 | 8 | — |
| **Raw totals** | **26,111** | **24,885** | | `additions+deletions = 50,996` |

| Step | Value |
|---|---|
| `Σ max` over survivors | 3 + 8 = **11** |
| `C_p` | **11** |
| `F_p` | **2** |

**The three provisions decompose cleanly, and you need all of them:**

| Provisions applied | Resulting `C_p` |
|---|---|
| None — naive `additions + deletions` | 50,996 |
| + per-file `max()` only | 26,111 |
| + `CHURN_CAP = 3000` only | 3,000 |
| + generated-file exclusion (**proposed**) | **11** |

A 51,000-line dependency bump ends up worth **twice a 10-line bugfix** and roughly **1/165th of the real feature**. That is the correct answer, and it is the single most visible thing the current dashboard gets wrong.

### 3.7 Worked example — a full developer-quarter

The "heavy AI user" from §1.4. Quarter = 13 weeks, 77 credited merged PRs, mean effective churn 420/PR, mean 7 effective files/PR, assist points 5.5 (§4.7 derives these).

```
N = 77            dpe  = 77 / 13      =  5.923 PR/wk
C = 32,340        cpw  = 32340 / 13   =  2487.7 lines/wk
F = 539           fpw  = 539 / 13     =  41.5 files/wk
A = 5.5           apw  = 5.5 / 13     =  0.423 assists/wk

T̂ = min(5.923/3.5, 2)/2 = min(1.692, 2)/2 = 0.846
Ŝ = min(2487.7/400, 2)/2 = min(6.219, 2)/2 = 1.000   ← at cap
B̂ = min(41.5/25,   2)/2 = min(1.660, 2)/2 = 0.830
R̂ = min(0.423/2.0, 2)/2 = min(0.212, 2)/2 = 0.106

ImpactIndex = 100 × (0.40·0.846 + 0.30·1.000 + 0.10·0.830 + 0.20·0.106)
            = 100 × (0.3384 + 0.3000 + 0.0830 + 0.0212)
            = 100 × 0.7426
            = 74
```

**74/100 ("strong"), against 4/100 today.**

Same person at 196 PRs/quarter: `dpe = 15.08` → `T̂ = 1.000`; substance and breadth also at cap; `ImpactIndex ≈ 0.40 + 0.30 + 0.10 + 0.20·R̂` → **80–100**, against **9/100** today.

Note the two capped terms. They are doing exactly their job: this developer is far above benchmark on volume, and the index says "at ceiling" rather than running away to an unbounded number. It also means **the marginal return on shipping a 78th PR this quarter is small** — which is the anti-gaming property, not a bug.

And the low `R̂` is informative rather than punitive: it says this person ships a great deal and reviews almost nothing. That is a real, actionable finding for a team owner, and it is invisible in the current dashboard.

---

## 4. Credit split — creator, merger, reviewer

### 4.1 The most important structural decision: assists are additive

**Credit for a merged PR is `1.0` to the creator, always, and is never divided.** Assists accrue to a *separate* bucket (`assistPoints`) that feeds only the `R̂` term.

Rationale, from first principles and from the literature:

- A zero-sum split makes every request for review a tax on your own score. Beck: *"Red Team any incentives. What would a clever … motivated person do to make the numbers look better."* **[CITED]**. A clever person facing a subtractive split stops asking for review, or splits work to avoid needing one. That is the exact failure mode SPACE warns about when it says *"A developer who optimizes only for their own personal productivity may hurt the productivity of the team."* **[CITED]**
- Additive assists mean the org's total credited work exceeds the number of PRs. That is fine — this is an index, not an accounting identity. It is not a budget being allocated.

### 4.2 Creator credit is already correct — including under squash merge

Worth stating plainly because the brief raises it:

- The dashboard's existing query is `author:LOGIN is:pr is:merged merged:RANGE` **[CODE]**. GitHub's `author:` qualifier matches the **pull request author**, which is set at creation and **never changes**.
- **A squash merge by a reviewer does not transfer PR authorship.** It creates a single commit whose *committer* may be the merger, but the PR object's `author` is untouched.
- **[MEASURED]** In 100 merged PRs from `org:farpointhq` (May–Jul 2026), `author.login` was populated on 100/100 and `author.__typename` was `User` on 50/50 in the sub-sample checked.

**So: no creator-credit leak exists today.** The gap is that assists are invisible. Say this to Ryan explicitly — it means §4 is purely additive work, with no migration risk to existing numbers.

⚠️ One real caveat: the dashboard counts **PRs, not commits**. If your org ships via direct pushes to a trunk, or via a stacked-diff tool that lands changes without a PR, that work is invisible to every metric here. Worth a line in the UI.

### 4.3 Detection logic — exact GraphQL paths

All field names below were **verified live** by GraphQL introspection and by executing the query, this session **[MEASURED]**.

```graphql
search(query: $q, type: ISSUE, first: 100) {
  nodes { ... on PullRequest {
    number
    author        { login __typename }      # Actor; __typename ∈ {User, Bot, Mannequin, Organization}
    mergedBy      { login __typename }      # Actor
    mergedAt
    additions deletions changedFiles
    repository { nameWithOwner }
    files(first: 100) {
      totalCount
      nodes { path additions deletions changeType }   # changeType: PatchStatus
    }
    reviews(first: 20) {
      totalCount
      nodes { state author { login __typename } submittedAt comments { totalCount } }
    }
    commits(first: 20) {
      totalCount
      nodes { commit {
        oid
        author  { user { login } email name }          # GitActor
        authors(first: 5) { totalCount nodes { user { login } email name } }
      } }
    }
  } }
}
```

**Enum values, introspected live [MEASURED]:**
- `PullRequestReviewState` = `PENDING`, `COMMENTED`, `APPROVED`, `CHANGES_REQUESTED`, `DISMISSED`
- `PatchStatus` = `ADDED`, `DELETED`, `RENAMED`, `COPIED`, `MODIFIED`, `CHANGED`

**REST equivalents**, if you stay on REST: `GET /repos/{owner}/{repo}/pulls/{pull_number}` returns `additions`, `deletions`, `changed_files`, `commits`, `merged`, `merged_by`, `merged_at`, `review_comments`, `comments`, `requested_reviewers`. ⚠️ **The LIST endpoint `GET /repos/{owner}/{repo}/pulls` returns "Pull Request Simple" objects which omit `additions`, `deletions`, `changed_files`, `commits`, `review_comments` and `comments`** — verified against [docs.github.com/en/rest/pulls/pulls](https://docs.github.com/en/rest/pulls/pulls?apiVersion=2022-11-28) **[CITED]**. That is why v2 is GraphQL: REST would need one request *per PR*.

### 4.4 Assist table

For each merged PR, for each non-author actor `x`:

| Role | Detection | Points |
|---|---|---|
| **Co-implementer** | ∃ `commits.nodes[].commit.author.user.login == x` where `x ≠ author.login` and `x ∉ BOT_LOGINS` | **0.50** |
| **Changes-requested reviewer** | ∃ `reviews.nodes[]` with `state == "CHANGES_REQUESTED"` and `author.login == x` | **0.40** |
| **Substantive approver** | ∃ review with `state == "APPROVED"` **and** `comments.totalCount > 0`, `author.login == x` | **0.30** |
| **Co-author (trailer)** | ∃ `commits.nodes[].commit.authors.nodes[].user.login == x`, `x ≠` that commit's `author.user.login`, `x ∉ BOT_LOGINS` | **0.25** |
| **Merger-assist** | `mergedBy.login == x`, `x ≠ author.login`, and `x` has no review row on this PR | **0.15** |
| **Rubber-stamp approver** | `state == "APPROVED"` and `comments.totalCount == 0` | **0.10** |
| **Self-merge** | `mergedBy.login == author.login` | **0.00** — no assist created |

**`assist(x, PR) = max(applicable role points)`, capped at 1.0.** Take the maximum, not the sum. A reviewer who requests changes, then merges, then pushes a fix would otherwise triple-count for one act of engagement. Max is both simpler to explain and strictly harder to farm.

**Justification for the rubber-stamp discount (0.10 vs 0.30)** is not my invention — it is Pluralsight's own published guidance for their equivalent metric: *"Don't start rubber-stamping PRs to increase this metric. That's a risk to your organization. To help strike a good balance, Flow shows both Unreviewed PRs and Thoroughly reviewed PRs."* ([archived](https://web.archive.org/web/2025id_/https://help.pluralsight.com/help/cant-you-just-game-the-metrics)) **[CITED]**

**Justification for co-implementer > merger-assist (0.50 vs 0.15).** Ryan's requirement was *"a reviewer might merge the PR and they might make changes before merging."* Those are two different acts with very different effort, and the API distinguishes them cleanly. Pushing a commit to someone else's branch is the strongest available evidence of substantive contribution short of authorship; clicking merge is the weakest signal that still means anything.

### 4.5 Bot and AI-agent exclusion — a live finding

**[MEASURED]** Across 216 commits in 50 merged `org:farpointhq` PRs (15 Jun – 30 Jul 2026):

- **16 commits carried more than one author** (a `Co-Authored-By:` trailer).
- The co-author login on those commits was **`claude`** — Claude Code's default `Co-Authored-By: Claude` trailer, which **GitHub resolves to a real GitHub account with the login `claude`**.
- **60 author entries had no linked GitHub user at all** (unmatched commit email; `commit.author.user` was `null`, `name` was `"Test"`).

Two hard requirements fall out of this:

1. **`BOT_LOGINS` must include `claude`, plus `github-actions[bot]`, `dependabot[bot]`, `renovate[bot]`, `copilot-swe-agent[bot]`, and any `__typename == "Bot"` actor.** Without this, **every Claude-Code-assisted PR in the org silently awards a 0.25 co-author assist to a nonexistent teammate.** On a Fabric-sold dashboard that would be a spectacular own goal.
2. **`commit.author.user` is nullable and is null ~28% of the time in this sample.** Never dereference it. Fall back to matching `commit.author.email` against the team's known emails; if neither resolves, drop the entry rather than guessing.

Swarmia's published rule is the cleanest external precedent for agent attribution: attribute a PR to an agent if *"the PR or its first commit has been authored by GitHub Copilot, Cursor, or Claude Code"* ([help.swarmia.com](https://help.swarmia.com/features/ai-tools/cloud-agents.md)) **[CITED]**.

**Recommendation:** surface AI co-authorship as its own dashboard column (`% of PRs AI-co-authored`) rather than burying it. That is the metric enterprise buyers actually want, and Fabric is uniquely positioned to report it.

### 4.6 ⚠️ Calibration warning — the review terms are inert on this org's data

**[MEASURED]** In 100 merged PRs from `org:farpointhq` (May–Jul 2026):

| Signal | Count |
|---|---|
| PRs with **any** `APPROVED` review | **0 / 100** |
| Self-merged (`mergedBy == author`) | **74 / 100** |
| Merged by someone else | **26 / 100** |
| PRs with ≥1 commit from a non-author human | **17 / 100** |
| PRs with `files.totalCount > 100` (files list truncated) | **3 / 100** |
| Median PR churn (`additions + deletions`) | **579** |
| p90 PR churn | **4,947** |
| Max PR churn | **59,544** |
| Share of file churn matching generated-file patterns | **11.8%** |

**Read this carefully before shipping the 0.20 collaboration weight.** On this org's own data:

- The approval-based assists (0.30 and 0.10) would fire **never**.
- The only live assist signals are **merger-assist (26%)** and **co-implementer (17%)**.
- Consequently `R̂` would read near 0 for nearly everyone, mechanically capping the index at ~80.

Three options, in order of preference:

1. **Validate the weights against a customer org that actually uses reviews** before enabling `R̂` at 0.20. This is a one-query check.
2. **Auto-detect review culture**: if a team's `APPROVED`-review rate is below some threshold (say 10% of merged PRs), **renormalise the remaining three weights to sum to 1.0** and hide the collaboration column. This is better than showing everyone a zero.
3. Ship `R̂` at 0.20 with the merger/co-implementer signals only, and accept that review-heavy orgs will score higher. Weakest option — it makes cross-org comparison meaningless.

I recommend **option 2**, implemented as: `w_R = 0.20 if teamApprovalRate ≥ 0.10 else 0`, with the other three weights scaled by `1/(1 − w_R_dropped)`.

Also note the churn figures: **median 579, p90 4,947, max 59,544.** The `CHURN_CAP = 3000` sits between p90 and max — it binds on roughly the top 5–8% of PRs, which is the intended blast radius. The 59,544-line PR is real and is exactly the case §3.6/PR C describes. And **11.8% of measured churn matched the generated-file list** — that is the size of the correction being proposed, on live data.

### 4.7 Derivation of the assist figure used in §3.7

For the worked developer: merged 20 of others' PRs without reviewing them (`20 × 0.15 = 3.0`) and pushed commits to 5 others' branches (`5 × 0.50 = 2.5`) → **A = 5.5**. Consistent with the 26% merger-assist / 17% co-implementer rates measured above.

---

## 5. Anti-gaming provisions

Each provision states the attack, the countermeasure, and the source.

### 5.1 Generated-file exclusion

**Attack.** Bump a lockfile; book 24,000 lines.

**Countermeasure.** Exclude a file from `C_p` and `F_p` if **either**:

**(a)** its path matches the exclusion list, or
**(b)** it is marked `linguist-generated` in the repo's `.gitattributes`.

Baseline list — lift GitHub's **own published exclusion set for Copilot code review** ([docs.github.com/en/copilot/reference/review-excluded-files](https://docs.github.com/en/copilot/reference/review-excluded-files)) **[CITED]**:

```
**/*.svg  **/*.log  **/*.lock  **/*.min.js  **/*.d.ts  **/*.bundle.js  **/*.map
**/go.sum  **/dist/**/*  **/out/**/*  **/bin/**/*  **/node_modules/**/*
**/coverage/**/*  **/vendor/**/*  **/generated/**/*  **/generated-sources/**/*
package-lock.json  yarn.lock  pnpm-lock.yaml  Pipfile.lock  Gemfile.lock
composer.lock  Cargo.lock  pubspec.lock  mix.lock  Podfile.lock  Package.resolved
```

**Why this list and not a hand-rolled one:** it is published by GitHub, maintained by GitHub, and defensible to a customer who asks "why is my PR not counted" — the answer is "GitHub's own code-review tool ignores these files too."

**Why `linguist-generated` specifically:** of Linguist's three attributes, **only `linguist-generated` also hides files in diffs**; `linguist-vendored` and `linguist-documentation` are stats-only ([linguist docs/overrides.md](https://github.com/github-linguist/linguist/blob/main/docs/overrides.md)) **[CITED]**. And there is direct commercial precedent for using it to gate a *size* judgment: Graphite treats a `linguist-generated=true` file as *"Excluded from AI review when determining if a PR is too large"* ([graphite.com/docs/diamond-customization](https://graphite.com/docs/diamond-customization)) **[CITED]**.

**Vendor practice confirms this is table stakes:** LinearB ships a restorable default regex set including `.*package-lock\.json$` and `.*dist/.*\.js` ([helpdocs](https://linearb.helpdocs.io/article/04kg2oyhhr-how-do-i-exclude-branches-from-my-dashboards)); Swarmia *"automatically cleans up the total change count of some commonly used generated files"* ([help.swarmia.com](https://help.swarmia.com/definitions/batch-size)) **[CITED — neither publishes its full list]**.

**[MEASURED] impact: 11.8% of file churn in the sampled 100 PRs.**

⚠️ **Reading `.gitattributes` costs an extra API call per repo per refresh.** Cache it per `repo@default-branch-sha`, effectively forever. If that is too much for v2.0, ship the path list alone and add `.gitattributes` in v2.1 — the path list captures the overwhelming majority.

### 5.2 Per-file `max(additions, deletions)`

**Attack.** Reformat a file, or delete-and-re-add a block, and book double.

**Countermeasure.** `max(additions, deletions)` per file, never the sum.

**Source.** Pluralsight HALOC, verbatim: *"In each hunk, Flow looks at the number of insertions and the number of deletions, then takes the higher number as the HALOC for the hunk."* **[CITED]**. Note the deliberate contrast with Swarmia, whose batch size *sums* added + removed ([help.swarmia.com](https://help.swarmia.com/definitions/batch-size)) — this proposal follows Pluralsight, not Swarmia, on this specific point, because a modified line genuinely appears twice in a unified diff.

⚠️ **Known limitation:** GitHub's API is per-file, not per-hunk, so a file with 100 additions in one place and 100 deletions in another counts as 100, not 200. This under-counts genuine work. Accept it; it errs conservative.

### 5.3 Log-free hard caps rather than unbounded churn

**Attack.** Land one enormous PR.

**Countermeasure.** `CHURN_CAP = 3000` and `FILE_CAP = 40` per PR; and the period-level terms cap at 2× benchmark.

**Justification.** Three independent lines:
- **Nagappan & Ball**: churn is a strong predictor of **defect density** (R² 0.811, 89.0% classification) **[CITED]** — so beyond a point, more churn is more liability, not more value.
- **LinearB's operational thresholds**: *"> 400 LOC slows reviews; > 800 LOC = very high risk"* **[CITED]**.
- **DORA's own diagnosis of the 2024 AI-throughput regression**: *"since AI allows respondents to produce a much greater amount of code in the same amount of time, it is possible, even likely, that changelists are growing in size. DORA has consistently shown that larger changes are slower and more prone to creating instability."* **[CITED]**

A score that pays linearly for churn would, in an AI-agent org, pay for exactly the behaviour DORA identifies as the mechanism of harm.

**[MEASURED]** `CHURN_CAP = 3000` sits between this org's p90 (4,947) and max (59,544), binding on ~5–8% of PRs.

### 5.4 Period-level churn aggregation — the anti-splitting provision

**Attack.** Split one 2,000-line PR into ten 200-line PRs. This is *the* documented real-world attack (Uber and Meta, §2.5).

**Countermeasure.** Churn, files and assists are **summed over the period, then normalised once**. Splitting a 2,000-line change into ten PRs leaves `C` **exactly unchanged**, so `Ŝ` and `B̂` are exactly invariant. Only `T̂` moves — and `T̂` caps at 2× benchmark (7 PRs/week).

Worked: a developer at 3.5 PRs/wk and 400 lines/wk scores `T̂ = 0.5, Ŝ = 0.5`. Splitting every PR in half → 7 PRs/wk, same lines: `T̂ = 1.0, Ŝ = 0.5`. Index rises by `0.40 × 0.5 = 0.20` → **+20 points**. Splitting again → 14 PRs/wk: `T̂` is already at cap, **+0 points**. So the attack yields one bounded step and then dies.

**And note the first step is behaviour we want.** Going from 400-line PRs to 200-line PRs moves a team from LinearB's "Needs focus" band into "Elite" (< 100–155 lines) and is exactly DORA's small-batch principle. **The only diff-splitting this design rewards is the diff-splitting the entire literature recommends.** That is the design goal, stated plainly.

**Design precedent for a structural cap:** Swarmia's developer-effort model allocates *"one FTE per month"* per developer, so volume can only *redistribute* effort across categories, never inflate the total ([help.swarmia.com](https://help.swarmia.com/definitions/developer-effort-ftes.md)) **[CITED]**. Our 2×-benchmark ceiling is the same idea with a softer edge.

### 5.5 Many-tiny-PRs vs one-big-PR neutrality — the explicit answer

Ryan's requirement, answered directly:

All rows hold the *same total work* (2,000 effective lines in one week) except row 4, which holds PR count constant and strips the substance out.

| Scenario, one week | `N` | `C` | `T̂` | `Ŝ` | Index (T+S only, reweighted) |
|---|---|---|---|---|---|
| 1 × 2,000-line PR | 1 | 2,000 | 0.143 | 1.000 | **51.0** |
| 5 × 400-line PRs | 5 | 2,000 | 0.714 | 1.000 | **83.7** |
| 10 × 200-line PRs | 10 | 2,000 | 1.000 | 1.000 | **100.0** |
| 40 × 5-line PRs | 40 | 200 | 1.000 | 0.250 | **67.9** |
| 40 × 50-line PRs | 40 | 2,000 | 1.000 | 1.000 | **100.0** |

*(computed as `100 × (0.40·T̂ + 0.30·Ŝ) / 0.70`)*

Reading the table:
- **40 trivial PRs (row 4, 67.9) score below 10 substantive PRs (row 3, 100.0).** The substance term is what stops PR-count farming, and it works: quadrupling the PR count while cutting the content by 90% *loses* 32 points.
- **Rows 3 and 5 are identical (100.0).** Splitting 10 PRs into 40 of the same total size buys **nothing** — `T̂` is already at cap and `Ŝ` is unchanged. The attack dies.
- **Splitting is worth up to +49 points (row 1 → row 3), then stops dead.** That is a large single step and it should be named honestly rather than buried. Three reasons it is acceptable:
  1. It rewards the transition from 2,000-line PRs to 200-line PRs, which LinearB bands as "Needs focus → Elite" and DORA identifies as the mechanism of delivery stability. **The metric is paying for the behaviour the literature says to buy.**
  2. Row 1 is not really "one big PR" — it is *a whole week in which the developer merged exactly once*. `dpe = 1` against a benchmark of 3.5 is a genuinely low-throughput week, and the index is right to say so.
  3. It is bounded and non-repeatable. There is no second +49.
- **One giant PR is penalised** (51.0) relative to the same work split sanely. Intentional, and consistent with both DORA and LinearB.

### 5.6 Self-merge handling

**Attack.** Merge your own PRs to farm merger-assists.

**Countermeasure.** `mergedBy.login == author.login` → **zero assist**. Self-merge is not an assist under any circumstance.

⚠️ Also do **not** penalise self-merge in the creator's throughput. **[MEASURED] 74% of this org's PRs are self-merged** — that is a normal working style for small teams and agent-heavy workflows, not a deficiency. A dashboard that docked 74% of PRs would be worthless here.

### 5.7 Assist = max, not sum

Covered in §4.4. Prevents one act of engagement counting three times.

### 5.8 Revert detection (v2.1, recommended)

**Attack.** Land it, get credit, revert it next week.

**Countermeasure.** Detect merged PRs whose `title` matches `^Revert "` (GitHub's auto-generated revert title) and subtract the reverted PR's credit from the original author's period, if the original falls in the same window.

**Why it matters more than it sounds.** Four independent sources say "merged" is not a quality gate in the AI era:
- Cui et al. measured Accenture's **build success rate at −17.4% (significant)** post-Copilot **[CITED]**;
- DORA's 2026 ROI model books an AI-induced change-failure-rate rise as a **−$344,000** cost line ([dora.dev/ai/roi/report](https://dora.dev/ai/roi/report/)) **[CITED]**;
- GitClear measured commits containing a duplicated 5+-line block rising **0.45% (2022) → 6.66% (2024)**, ~10× in two years, over 211M changed lines ([PDF](https://gitclear-public.s3.us-west-2.amazonaws.com/GitClear-AI-Copilot-Code-Quality-2025.pdf)) **[CITED]**;
- a single-author controlled experiment merged agent implementations scoring **16/38 to 30/38** on deterministic tests, all in one review cycle ([blog.insight-services-apac.dev](https://blog.insight-services-apac.dev/2026/07/06/cost-to-a-merged-feature)) **[CITED — n=1, illustrative only]**.

**Ship the index without revert detection if you must, but do not market it as measuring quality.** It measures *shipped change volume and collaboration*. Say so in the caption.

### 5.9 Suppression floor for comparison views

**Countermeasure.** Do not render any cross-person ranking view for a team with **fewer than 5 mapped, scored members**.

**Source.** Graphite: *"We won't attempt to draw comparisons between developers using Graphite or GitHub if there are fewer than 5 active Graphite users"* ([graphite.com/docs/insights-stats-definitions](https://graphite.com/docs/insights-stats-definitions)) **[CITED]**. Same page also uses **median-of-medians** rather than means for weekly aggregates — worth adopting for the team roll-up, since the current code uses a plain mean (`route.ts:200-204` **[CODE]**) which one outlier distorts.

### 5.10 Compare to own history by default

**Countermeasure.** Default the benchmark toggle to **"this team's trailing 4 quarters"**, with "industry median" as the opt-in.

**Source.** Haystack: *"When we compare teams/members against one another we are establishing an unhealthy pattern in our team culture… Each team should be improving and compared to their own history rather than other teams."* **[CITED]**. Also SPACE's normalisation warning: *"looking at metrics over a year would bias against those taking parental leave."* **[CITED]** — which argues for the per-week rates this design already uses, and for excluding zero-activity weeks from `W` where leave is known.

### 5.11 Framing provisions

These are cheap and they are what stops the metric being weaponised:

1. **Rename.** `productivityScore` → `impactIndex`. UI label: **"Engineering Impact"** or **"Shipped Work Index"**. Never "productivity." Keep `productivityScore` as a deprecated alias for one release.
2. **Team roll-up is the default view**; per-person table behind a click.
3. **Standing caption**, verbatim suggestion:
   > *Impact Index measures shipped change volume and collaboration visible on GitHub. It does not measure code quality, design work, mentoring, incident response, or work done outside pull requests. It is a conversation starter, not a performance rating.*
   Justified by SPACE's list of intractable activities: *"attending team meetings, participating in brainstorming, helping other team members… providing architectural guidance."* **[CITED]**
4. **Publish the formula in-product.** Developers will reverse-engineer it regardless (the brief assumes this correctly). A published formula with visible caps is *harder* to game than a secret one, because the caps are visibly binding. This is the specific failure of GitPrime's Impact metric: it named six inputs and **never published a single weight** **[CITED]** — the canonical weighted-LOC composite is a black box, and that is why it is the one everyone cites as a cautionary tale.
5. **Never set per-individual targets on it.** DX's own longitudinal study *excluded* such teams from their data as contaminated: *"We've filtered out potential gamification effects by excluding teams that set PR throughput targets for individual engineers"* ([getdx.com/report/ai-and-engineering-velocity-a-longitudinal-analysis](https://getdx.com/report/ai-and-engineering-velocity-a-longitudinal-analysis/)) **[CITED]**.

---

## 6. The efficiency view

### 6.1 Replace the ratio-score with dollars per accepted change

**Delete `computeScore` from the efficiency column entirely.** Replace with:

```
costPerMergedPR      = spendUSD / N                    ← PRIMARY (DORA's phrase)
tokensPerMergedPR    = tokens / N
costPer1kEffLines    = spendUSD / (C / 1000)           ← COUNTER-METRIC
spendPerWeek         = spendUSD / W
```

**`spendUSD` is already available.** `DailyUsageSummary.costCents` exists in `prisma/schema.prisma:492` **[CODE]** and the productivity route currently selects only `inputTokens`, `outputTokens`, `requests` (`route.ts:89-94` **[CODE]**). Adding `costCents: true` to that select is a one-line change and upgrades the whole efficiency view from tokens to dollars.

**Why `costPer1kEffLines` is required as a counter-metric.** `costPerMergedPR` is minimised by splitting PRs — the same attack as §5.4, pointed at the denominator. `costPer1kEffLines` is **exactly invariant to splitting** because `C` is. Reporting both means the two can only be improved together by genuinely doing more with less.

This is SPACE's design principle applied deliberately: *"Including metrics from multiple dimensions and types of measurements often creates metrics in tension; this is by design, because a balanced view provides a truer picture."* **[CITED]**

### 6.2 Banding: percentile, not absolute

Do **not** re-introduce fixed thresholds. Band `costPerMergedPR` against **the team's own distribution** over the trailing 4 quarters, and label:

| Band | Definition |
|---|---|
| Efficient | ≤ team P25 |
| Typical | P25 – P75 |
| Heavy | P75 – P90 |
| Outlier | > P90 |

Rationale: the published spread is enormous and no absolute threshold survives it. Jellyfish, over 12,000 developers / 200 companies: **cost per merged PR ranges $0.28 (lightest decile) to $89.32 (heaviest) — a ~300× spread** ([jellyfish.co/library/ai-token-usage-monitoring](https://jellyfish.co/library/ai-token-usage-monitoring/)) **[CITED — vendor, correlational]**. And a single controlled experiment moved one identical feature from **$3 to $33 to merge** purely by changing harness and model ([blog.insight-services-apac.dev](https://blog.insight-services-apac.dev/2026/07/06/cost-to-a-merged-feature)) **[CITED — n=1]**.

**Surface the distribution, not the mean.** Absolute reference points, for context only, all **[CITED]**:

| Reference | Figure | Source |
|---|---|---|
| Jellyfish median dev | ~7.5M tokens/PR; ~$52/month | [jellyfish.co](https://jellyfish.co/library/ai-token-usage-monitoring/) |
| Jellyfish P90 dev | ~69M tokens/PR; ~$691/month | same |
| Anthropic published Claude Code spend | *"around $13 per developer per active day and $150–250 per developer per month"* | [code.claude.com/docs/en/costs](https://code.claude.com/docs/en/costs) |
| DX observed all-in | *"$200–$600/month"* seat + token, teams mixing inline and agentic tools | [getdx.com](https://getdx.com/blog/ai-coding-assistant-pricing/) |

### 6.3 The two-axis view is the actual product

The dashboard's real job is the **scatter, not the score**: Impact Index on one axis, cost per merged PR on the other, one dot per engineer, quadrant lines at team medians.

```
        high impact
             │
   efficient │ expensive
   & shipping│ & shipping      ← the interesting quadrant:
             │                    high output, high burn
─────────────┼──────────────      → a model-routing conversation,
             │                      not a performance conversation
   efficient │ expensive
   & quiet   │ & quiet         ← the genuine alert
             │
        low impact
```

This directly answers the stated audience ("enterprise team owners assessing whether AI spend converts to output") without ever collapsing two questions into one number — which is the mistake §1 diagnoses.

DX ships a structurally identical thing: their AI Measurement Framework's Cost dimension includes *"AI spend (total and per developer)"* and *"agent hourly rate (human-equivalent hours / AI spend)"* ([getdx.com/blog/introducing-the-ai-measurement-framework](https://getdx.com/blog/introducing-the-ai-measurement-framework/)) **[CITED]**.

### 6.4 Rewrite the alerts

The three alert rules in `productivity.ts:112-168` **[CODE]** are keyed off the broken score and must be retargeted:

| Current rule | Problem | Replacement |
|---|---|---|
| `zeroOutputRule` — tokens > 1M and 0 PRs and 0 issues | Threshold of 1M tokens is ~1 working day for an agent user. Will fire constantly. | Fire on `tokens > teamP50 AND N == 0` over a period ≥ 2 weeks. |
| `lowProductivityRule` — score < 25 | Fires on the artefact. §1.3 shows the median AI developer trips it by construction. | Fire on `costPerMergedPR > teamP90 AND N > 0`. Reword to *"X's cost per merged PR is in the top decile for this team"* — an observation, not a verdict. |
| `highTokensRule` — no GitHub mapping | Fine. Keep. | Keep; it is about data coverage, not people. |

⚠️ **The current message string — `"token usage doesn't reflect productivity"` — must go.** It is a verdict on a person, generated by a formula that (§1.4) rates the org's most productive engineers at 4/100.

### 6.5 Numeric before/after on the same inputs

| Developer | Merged PRs/qtr | Tokens | **Current score** | Current rating | **Impact Index** | **tokens/PR** | Efficiency band |
|---|---|---|---|---|---|---|---|
| Heavy A | 77 | 3.76B | **4** | low | **74** | 48.8M | Heavy (vs Jellyfish P90 of 69M) |
| Heavy B | 196 | 3.76B | **9** | low | **85** | 19.2M | Typical–Heavy |
| Light C | 3 | 1.0M | **75** | excellent | **5** | 0.33M | Efficient |
| Light D | 1 | 0.1M | **100** | excellent | **2** | 0.10M | Efficient |

*(Impact Index for B/C/D computed on the §3.7 assumptions of 420 effective lines and 7 effective files per PR; `A = 14` for B, `A = 0` for C and D. Worked: C → `T̂ = 0.033, Ŝ = 0.121, B̂ = 0.032, R̂ = 0` → `100 × (0.0132 + 0.0363 + 0.0032) = 5.3`.)*

**The two columns now disagree, and that disagreement is the product.** Developer A ships a great deal and spends a great deal; Developer D spends nothing and ships nothing. The current dashboard rates D at 100 and A at 4. The proposed dashboard rates A at 74/Heavy and D at 3/Efficient — and a team owner can immediately see which of those is a person and which is a licence to reclaim.

---

## 7. API cost analysis

### 7.1 Measured facts

All **[MEASURED]** live this session against `github.com`.

**Rate limit budgets** (`gh api rate_limit`):

| Resource | Limit |
|---|---|
| `core` (REST) | 5,000 / hour |
| `search` (REST Search) | **30 / minute** |
| `graphql` | **5,000 points / hour** |

Documented secondary limits ([docs.github.com GraphQL rate limits](https://docs.github.com/en/graphql/overview/rate-limits-and-node-limits-for-the-graphql-api)) **[CITED]**: *"no more than 2,000 points per minute"*, *"No more than 100 concurrent requests"*, *"Individual calls cannot request more than 500,000 total nodes."*

**Measured cost of candidate queries**, each one request against `repo:farpointhq/Fabric` / `org:farpointhq`:

| Query shape | PRs/page | **Points** | Nodes |
|---|---|---|---|
| Scalars only (`additions deletions changedFiles author mergedBy`) | 100 | **1** | 100 |
| + `files(first:100)` | 100 | **1** | 10,100 |
| + `reviews(10)` + `commits(20)` | 100 | **2** | 3,100 |
| **+ `reviews(10)` + `commits(20)` + `files(100)`** ← proposed v2 | **100** | **3** | **13,100** |
| + `commits(20){authors(5)}` | 50 | 11 | 6,050 |
| + `commits(100)` + `reviews(20)` + `files(100)` | 50 | 52 | 36,050 |

**Two design conclusions fall straight out:**
1. **`files(first:100)` is effectively free** (cost 1 → 1). Generated-file exclusion costs nothing.
2. **Nested `authors` inside `commits` is expensive** (11 points / 50 PRs = 22 per 100). Fetch co-author trailers only when needed, or accept ~22 points per 100 PRs — still trivial against 5,000/hr.
3. **`commits(first: 100)` is a cliff** (52 points / 50 PRs). Use `commits(first: 20)`; **[MEASURED]** the sampled PRs averaged 4.3 commits each, so 20 covers the overwhelming majority. Flag PRs where `commits.totalCount > 20` rather than paginating.

**Search result cap — same 1,000 limit as REST.** [MEASURED] `org:farpointhq is:pr is:merged merged:2025-01-01..2026-07-30` reported `issueCount: 1474`, but paging stopped after **exactly 10 pages / 1,000 nodes** with `hasNextPage: false`. Total cost for all 10 pages: **10 points.**

**Multi-author OR works identically in GraphQL.** [MEASURED] `author:samit-farpoint author:jensen is:pr is:merged merged:...` → `issueCount 62`, correctly split 52/10 across the two authors, cost 1.

### 7.2 Cost per team refresh — current vs proposed

**Current (v0), REST Search** [CODE, `github-activity.ts`]:
- 5 metrics × chunked `author:` queries (≤256 chars ⇒ ~15 logins/chunk).
- 30-member team → 2 chunks × 5 metrics = **10 requests minimum**, plus up to 10 pagination pages per chunk-metric, plus binary-halving recursion when a chunk exceeds 1,000 matches.
- Realistic cold run: **10–60 requests** against a **30/min** budget, self-throttled to 25/min (`SEARCH_RATE_LIMIT_PER_MINUTE`).
- Wall clock: the code's own comment says *"a cold run over a whole team takes ~60s"*, and `maxDuration = 300` **[CODE]**.

**Proposed (v2), GraphQL Search:**
- Query by **org and time window**, not per author. One query covers every member at once.
- To stay under the 1,000-result cap, partition the period into windows that each yield < 1,000 merged PRs. Weekly windows are safe for almost any team; the fetcher should adaptively halve a window whose `issueCount > 1000`.

| Team profile | PRs/quarter | Windows | Pages | **Points** | % of hourly budget |
|---|---|---|---|---|---|
| 10 devs, ~35 PRs/wk | ~455 | 13 weekly | ~18 | **~54** | 1.1% |
| 30 devs, ~105 PRs/wk | ~1,365 | 13 weekly | ~27 | **~81** | 1.6% |
| 100 devs, ~350 PRs/wk | ~4,550 | 13 weekly | ~59 | **~177** | 3.5% |
| Worst case, 13 windows × 10 pages | 13,000 | 13 | 130 | **390** | **7.8%** |

*(Page counts rounded up for headroom — e.g. 30 devs at 105 PRs/week needs 2 pages/window × 13 = 26 pages = 78 points; the table books 27/81.)*

Add ~22 points/100 PRs if co-author trailers are fetched in the same pass → worst case ~**680 points**, still **13.6%** of one hour's budget.

**Headline: a 100-developer team-quarter refresh costs ~177 GraphQL points out of 5,000/hour — about 28 full refreshes per hour — and returns strictly more data than the current design, which can manage roughly one refresh per minute of wall clock against a 30/min budget.**

The 2,000-points-per-minute secondary limit is never approached (max ~390 points total).

### 7.3 What to cache and precompute

**The key insight: a merged PR's record is immutable.** `additions`, `deletions`, `changedFiles`, `mergedAt`, `mergedBy`, `author` and the file list never change after merge. The current design caches per-`login:start:end` for 30 minutes **[CODE]** — which re-fetches settled history on every window change.

**Recommended architecture:**

1. **New Prisma model, `PrImpactRecord`** — one row per merged PR, written once:
   ```
   repoFullName  String
   prNumber      Int
   authorLogin   String
   mergedAt      DateTime
   mergedByLogin String?
   effectiveChurn      Int      // C_p, post-exclusion, post-cap
   effectiveFiles      Int      // F_p
   rawAdditions/rawDeletions/rawChangedFiles  Int   // for audit + recompute
   filesTruncated      Boolean  // files.totalCount > 100
   commitsTruncated    Boolean  // commits.totalCount > 20
   assists             Json     // [{login, role, points}]
   exclusionListVersion String  // so the list can evolve and rows be recomputed
   @@unique([repoFullName, prNumber])
   ```
   Store the **raw** stats alongside the derived ones so the formula can be re-tuned without re-hitting GitHub. That is the single most valuable property of this design.

2. **Nightly backfill job** ingests `merged:YESTERDAY` for each connected org. Steady-state cost: a handful of points per org per night.

3. **Dashboard reads Postgres only** for any fully-elapsed day. Live GitHub calls are needed **only for the current partial day**, which is at most 1–2 points.

4. **Keep the existing L1/L2 runtime cache** for the assembled summary payload (`SUMMARY_TTL_SECONDS = 300` **[CODE]**) — it is fine and unchanged.

5. **`.gitattributes` cache**: key on `repo@defaultBranchSha`, TTL effectively infinite.

6. **Recompute path**: because raw stats are stored, changing `CHURN_CAP`, the anchors, the weights, or the exclusion list is a **pure SQL/compute job with zero GitHub calls**. Bump `exclusionListVersion` and recompute. This is what makes the calibration work in §4.6 practical.

**Net effect:** the steady-state dashboard makes **near-zero GitHub requests**, and the 25/min throttle in `github-activity.ts` becomes relevant only to backfills.

---

## 8. Minimum viable v1 vs full v2

### 8.1 v1 — shippable in a day, no new data source

**Constraint honoured:** keeps the existing per-login REST Search count architecture in `github-activity.ts` **[CODE]** untouched. Only `productivity.ts` and the route's assembly change.

**v1 formula** — throughput only, since `prsMerged` is already fetched:

```
W    = max(1, spanDays / 7)
dpe  = prsMerged / W
T̂    = min(dpe / 3.5, 2) / 2
Î    = min((issuesClosed / W) / 2.0, 2) / 2        // issuesClosed already fetched
impactIndex = round(100 × (0.80·T̂ + 0.20·Î))
```

Substance and breadth are unavailable without per-PR detail, so their weight redistributes onto throughput. Closed issues get the small residual weight (they are already counted **[CODE]** and are a weak but real signal).

**v1 output on the brief's cases:**

| Case | dpe | T̂ | impactIndex (issues = 0) | vs current |
|---|---|---|---|---|
| 77 PRs / 13 wk | 5.92 | 0.846 | **68** | 4 |
| 196 PRs / 13 wk | 15.08 | 1.000 | **80** | 9 |
| 3 PRs / 13 wk | 0.23 | 0.033 | **3** | 75 |

**v1 checklist:**

1. `productivity.ts`: add `computeImpactIndex(...)`. Keep `computeScore` exported and unused for one release so nothing breaks.
2. `route.ts`: add `costCents: true` to the `dailyUsageSummary.select` (**one line**, field exists **[CODE]**) and emit `spendCents`, `costPerMergedPRCents`, `tokensPerMergedPR`.
3. Replace `efficiencyRating(score)` with a **team-percentile band** on `costPerMergedPR` computed in-route over the already-loaded team array. No extra queries.
4. Rewrite the three alert rules per §6.4.
5. Rename in the payload: `productivityScore` → `impactIndex`; keep `productivityScore` as a duplicated deprecated field for one release. Change the UI label off the word "productivity."
6. Change `avgProductivity` from a mean to a **median** (`route.ts:200-204` **[CODE]**) — Graphite precedent §5.9.
7. Add the standing caption (§5.11.3) and the ≥5-member suppression floor on any ranking view.
8. **Optional, +1–2 chunked searches per team:** add a review-assist proxy using the search qualifier `is:pr is:merged merged:RANGE reviewed-by:LOGIN -author:LOGIN`, feeding a v1 `R̂`.
   ⚠️ **I did not verify this qualifier combination executes correctly in this session — NOT VERIFIED.** It is a documented GitHub search qualifier, but confirm with one call before relying on it, and note it costs one more chunked query per metric against the 30/min budget.

**Explicitly out of scope for v1:** per-file data, generated-file exclusion, churn, merger-assist, co-implementer detection, bot exclusion, revert detection. **v1 fixes the inversion; it does not implement the substance or credit-split requirements.** Be clear with Ryan about that.

### 8.2 v2 — full per-PR detail

1. **New fetcher** `src/lib/reporting/github-pr-details.ts` using the GraphQL query in §4.3, org-scoped with adaptive time-window halving on `issueCount > 1000`.
2. **`PrImpactRecord` model** + nightly backfill (§7.3).
3. **Exclusion engine**: Copilot path list + `.gitattributes` `linguist-generated`, versioned.
4. **Per-PR derivation**: `C_p`, `F_p` with per-file `max()` and caps.
5. **Assist engine** per §4.4, with `BOT_LOGINS` including **`claude`** (§4.5) and null-safe `commit.author.user`.
6. **Full four-term index** with the §4.6 adaptive collaboration weight.
7. **Two-axis scatter** (§6.3) replacing the single efficiency column.
8. **Tooltip decomposition** (§3.5) — mandatory, not optional.
9. **v2.1**: revert detection (§5.8); AI-co-authorship column (§4.5).

### 8.3 Correctness risks to handle in v2

| Risk | Evidence | Mitigation |
|---|---|---|
| `files` truncates at 100 | **[MEASURED] 3/100 PRs** | Set `filesTruncated`; fall back to PR-level `max(additions, deletions)`; show a "size estimated" marker |
| `commits` truncates at 20 | mean 4.3 commits/PR **[MEASURED]** | Set `commitsTruncated`; do not paginate (§7.1 cost cliff) |
| `commit.author.user` is null | **[MEASURED] ~28% of author entries** | Never dereference; fall back to email match; else drop |
| `claude` co-author credited as human | **[MEASURED] 16/216 commits** | `BOT_LOGINS` |
| Search 1,000-result cap | **[MEASURED]** | Adaptive window halving |
| PAT visibility | — | Search returns only what the token can see. Surface "N repos visible to this token" so a missing repo is diagnosable, not silent |
| Review-based assists inert | **[MEASURED] 0/100 approvals** | Adaptive weight, §4.6 |

---

## 9. Open questions for Ryan

1. **Anchors: industry or own-history as the default?** I recommend own-history default with an industry toggle (Haystack). Industry-median-as-default makes the number comparable across customers, which may be worth more commercially.
2. **Collaboration weight.** §4.6 shows it would read ~0 on Fabric's own org. Ship adaptive (option 2), or defer `R̂` to v2.1?
3. **Is the per-person table the product, or the team roll-up?** Every source in §2.2 says team. The brief says the audience is team owners assessing individuals. This is a deliberate product decision, not a research finding — I have flagged it rather than resolved it.
4. **Revert detection in v2.0 or v2.1?** Without it, do not claim the index reflects quality.
5. **Legal review** for per-person output reporting in EU-exposed enterprise accounts (§2.2). **NOT VERIFIED** — flagged only.

---

## 10. Source index

**Peer-reviewed / primary**
- SPACE — Forsgren, Storey, Maddila, Zimmermann, Houck, Butler, *ACM Queue* 19(1) 2021 — https://queue.acm.org/detail.cfm?id=3454124 (open PDF mirror: https://people.uncw.edu/vetterr/classes/csc550-spring2023/The%20SPACE%20of%20Developer%20Productivity.pdf)
- Jaspan & Sadowski, "No Single Metric Captures Productivity", *Rethinking Productivity in SE*, 2019 — https://link.springer.com/chapter/10.1007/978-1-4842-4221-6_2
- *Software Engineering at Google*, ch. 7 (Jaspan) — https://abseil.io/resources/swe-book/html/ch07.html
- Nagappan & Ball, ICSE 2005 — https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/icse05churn.pdf
- Graves, Karr, Marron & Siy, IEEE TSE 26(7) 2000 — https://cs.uwaterloo.ca/~m2nagapp/courses/CS846/1171/papers/graves_tse98.pdf
- Dijkstra EWD1036 — https://www.cs.utexas.edu/~EWD/transcriptions/EWD10xx/EWD1036.html
- Cui, Demirer, Jaffe, Musolff, Peng & Salz, *Management Science* 2026 — https://pubsonline.informs.org/doi/10.1287/mnsc.2025.00535 · draft https://economics.mit.edu/sites/default/files/inline-files/draft_copilot_experiments.pdf
- Peng, Kalliamvakou, Cihon & Demirer, arXiv 2302.06590 — https://arxiv.org/abs/2302.06590
- Murphy-Hill, Butler & Savelieva, arXiv 2607.01418 — https://arxiv.org/html/2607.01418
- Beller et al., "What's DAT?" (Meta), arXiv 2503.10977 — https://arxiv.org/html/2503.10977v1
- METR uplift study — https://arxiv.org/abs/2507.09089 · **update (use this one)** https://metr.org/blog/2026-02-24-uplift-update/
- Strathern, *European Review* 5(3) 1997 — https://gwern.net/doc/statistics/decision/1997-strathern.pdf

**DORA**
- 2024 State of DevOps PDF — https://dora.dev/research/2024/dora-report/2024-dora-accelerate-state-of-devops-report.pdf
- 2025 State of AI-assisted Software Development — https://services.google.com/fh/files/misc/2025_state_of_ai_assisted_software_development.pdf
- "Finding balance in the era of tokenmaxxing" — https://dora.dev/insights/finding-balance-in-the-era-of-tokenmaxxing/
- ROI of AI-assisted software development 2026 — https://dora.dev/ai/roi/report/

**Vendor / industry**
- DX Core 4 — https://getdx.com/research/measuring-developer-productivity-with-the-dx-core-4/ · benchmarks https://getdx.com/research/benchmarks/
- DX AI Measurement Framework — https://getdx.com/blog/introducing-the-ai-measurement-framework/ · cost report https://getdx.com/blog/meet-the-new-ai-cost-management-report/ · longitudinal https://getdx.com/report/ai-and-engineering-velocity-a-longitudinal-analysis/
- LinearB benchmarks — https://linearb.io/resources/engineering-benchmarks · PR size https://linearb.helpdocs.io/article/oxqqb67fbf-pr-size-metric · exclusions https://linearb.helpdocs.io/article/04kg2oyhhr-how-do-i-exclude-branches-from-my-dashboards · AI cost/PR https://linearb.helpdocs.io/article/w1yl8ejm8p-ai-cost-and-token-consumption
- Swarmia — batch size https://help.swarmia.com/definitions/batch-size · effort FTEs https://help.swarmia.com/definitions/developer-effort-ftes.md · agents https://help.swarmia.com/features/ai-tools/cloud-agents.md · anti-stack-rank https://www.swarmia.com/blog/dont-stack-rank-your-developers/
- Jellyfish token/PR data — https://jellyfish.co/library/ai-token-usage-monitoring/ · Claude Code attribution window https://jellyfish.co/library/claude-code-monitoring/
- Pluralsight/GitPrime Impact (archived) — https://web.archive.org/web/2019id_/https://blog.gitprime.com/impact-a-better-way-to-measure-codebase-change/ · HALOC https://web.archive.org/web/2025id_/https://help.pluralsight.com/help/haloc · gaming 2017 https://web.archive.org/web/20170325145422/http://help.gitprime.com/general/cant-you-just-game-the-metrics · gaming 2023 https://web.archive.org/web/2025id_/https://help.pluralsight.com/help/cant-you-just-game-the-metrics
- Graphite — https://graphite.com/docs/insights-stats-definitions · https://graphite.com/docs/diamond-customization
- Haystack — https://support.usehaystack.io/en/articles/4540688-software-engineering-metrics-faqs
- Vantage cost-per-PR — https://www.vantage.sh/blog/agentic-coding-efficiency
- GitClear 2025 code-quality report — https://gitclear-public.s3.us-west-2.amazonaws.com/GitClear-AI-Copilot-Code-Quality-2025.pdf

**Critique**
- Beck & Orosz — https://newsletter.pragmaticengineer.com/p/measuring-developer-productivity · part 2 https://newsletter.pragmaticengineer.com/p/measuring-developer-productivity-part-2 · Beck's part 2 https://tidyfirst.substack.com/p/measuring-developer-productivity-440
- Dan North — https://dannorth.net/blog/mckinsey-review/
- McKinsey (the target) — https://www.mckinsey.com/industries/technology-media-and-telecommunications/our-insights/yes-you-can-measure-software-developer-productivity
- Fowler, CannotMeasureProductivity — https://martinfowler.com/bliki/CannotMeasureProductivity.html

**GitHub API**
- REST pulls — https://docs.github.com/en/rest/pulls/pulls?apiVersion=2022-11-28
- GraphQL rate/node limits — https://docs.github.com/en/graphql/overview/rate-limits-and-node-limits-for-the-graphql-api
- Linguist overrides — https://github.com/github-linguist/linguist/blob/main/docs/overrides.md
- Copilot code review excluded files — https://docs.github.com/en/copilot/reference/review-excluded-files

**Explicitly NOT VERIFIED / do not cite:** Bill Gates "aircraft by weight" quote · a GeePaw Hill McKinsey response · a LinearB "Impact" metric · SWE-bench per-instance costs · DORA 2025 per-outcome percentages (chart-only) · the "2 diffs/week at Meta" figure (single anonymous Blind post) · `reviewed-by:` + `-author:` search combination (documented, untested here).
