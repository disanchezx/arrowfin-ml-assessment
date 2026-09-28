# ArrowFin ML Engineer Assessment: Unusually Bad Sessions

**Time spent:** _TODO: fill in the real time._

**LLM use:** I used Claude (Claude Code) as a pair programmer throughout. It profiled the dataset,
wrote the first version of the FIFO PnL code and the notebook structure, and drafted parts of this README.
I chose the definition of "unusually bad", the thresholds, and what to cut, and I checked every number quoted here
against the notebook output. _TODO: add what you would change if you had written it yourself
(for example, "the notebook is more verbose than I would write it", "I would have started with a simpler PnL
(round trips only) and only moved to FIFO if needed")._

---

## Background

_TODO: complete it honestly. This section must be yours, not the LLM's._

**Self-ratings** (1 never used, 2 tutorial-level, 3 shipped work with it, 4 comfortable in production, 5 could teach it)

- Python: _
- pandas / polars: _
- SQL: _
- Feature engineering: _
- Model evaluation and validation: _
- Time-series / sequential data: _
- Taking a model into production: _
- Experiment tracking: _

**Five short answers**

- What kind of modelling problem are you strongest with? _
- What would be the steepest learning curve for you here? _
- A core strength you haven't asked about: _
- What kind of work do you most want to do day-to-day? _
- What do you most want to avoid? _

---

## How to run

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/jupyter notebook analysis.ipynb   # Kernel → Restart & Run All
```

The notebook reads `dataset/` and writes `output/pnl_sessions.csv` (one row per trader and session) and `output/top10.csv`.

| Notebook section | What it does |
|---|---|
| 1–2 | Load the data and run integrity checks (asserts) |
| 3 | Add trader, point value, signed quantity and the CME session date |
| 4–5 | Realized PnL with FIFO matching, and unrealized PnL at the marks |
| 6 | PnL table per trader and session, reconciled against a cash-flow total |
| 7 | Suspicious prices |
| 8 | **Task 1:** top 10, plus robustness checks |
| 9 | **Task 5:** T-020 deep dive |

---

## What is in the data (and what I did about it)

| Observation | What I did |
|---|---|
| **Point values differ by up to 62,500×** (MNQ $2/pt, 6E $125,000/pt). NQ is 10× MNQ, ES is 10× MES, CL is 10× MCL. | PnL always uses the instrument's `point_value_usd`. Size is compared as contracts × point value, not raw contracts. |
| **A session is not a UTC day.** It runs 17:00 to 16:00 Chicago time. 10 fills (T-020 and T-021, on the evening of 24 Aug) have a UTC date that differs from their session. | `session_date = (Chicago time + 7h).date()`. |
| **Today's session is incomplete.** The cut is 09:30 CT. Earlier sessions have fills until about 15:20 CT. | Today's number is "so far". It is compared against full past sessions, which makes today look *less* bad than it may end up. |
| **Orders fill in pieces.** 5,146 fills belong to 4,886 orders, and 260 orders have more than one fill. | Activity is counted in orders, not fills. |
| **Commissions** are a flat fee per contract per side ($0.70 on micros, up to $2.48 on 6E), about $1.6k–2.1k per session across all traders. | Always subtracted. For high-frequency traders they can be the whole story: T-013 made +$26 before commissions today and paid $126. |
| **8 positions are open at the cut.** The largest is T-023's 3-lot CL short, at −$2,340 unrealized. | Marked to `market_prices.csv` and added to today's session. No position carries over between earlier sessions, so not having settlement prices does not matter here. |
| **Accounts in odd states:** ACC-2028 (T-025) is active and funded with a **$0 balance** and an open position. ACC-2053 is *closed* and has 20 fills. ACC-2054 is *restricted* and has fills and an open position. | Kept in the analysis and flagged. I'd ask the client about them, since they may be exactly the accounts that already blew up. |
| **Uneven history.** T-024 and T-025 have only today's session and T-026 has two. | Excluded from the z-score and listed separately. **T-024 lost $1,580 in its first session**, and a reviewer should know that. |
| **9 MCL fills** are 2–7% away from the session median (for example, 79.29 against a median of 74.14 on 24 Aug). | Not removed, since I can't tell a bad print from a bad fill. None are from today and none belong to top-10 traders' today sessions. |

The PnL engine is checked twice. A hand-worked FIFO example (including a position flip) is asserted, and the total
FIFO PnL (−$45,253.00) equals the cash-flow PnL (sales − purchases + open positions at the mark) to the cent.

---

## Task 1: The ten traders in trouble

**Definition.** A session is *unusually bad* when the trader is losing money **and** the loss is far below what
*that trader* normally does:

```
z = (today's net PnL − trader's mean net PnL in prior sessions) / trader's std of net PnL in prior sessions
```

Net PnL = realized (FIFO) + unrealized at the mark − commissions. Only sessions **before** today go into the baseline.
A trader needs at least 5 prior sessions to be ranked.

**Why this definition.** A trader who usually trades 10 contracts has daily swings that are ten times bigger in dollars.
Dividing by the trader's own variability makes a $2,000 loss small for that trader and huge for a one-lot trader,
which is exactly the distinction the brief asks for. It is also easy to explain: "this is X times worse than their usual bad day".

**The list** (session of 25 Aug, as of 09:30 CT, worst first)

| # | Trader | Net PnL today | Usual (mean) | z | Size vs usual | Note |
|---|---|---|---|---|---|---|
| 1 | T-018 | −$1,517 | −$110 | −8.9 | 0.9× | Normal size, loss 4× their worst past session |
| 2 | T-023 | −$2,486 | −$378 | −5.5 | 2.3× | Open 3-lot CL short at −$2,340 unrealized |
| 3 | T-019 | −$1,559 | −$9 | −5.1 | 1.1× | Normal size |
| 4 | T-009 | −$670 | −$67 | −5.0 | 3.8× | Sized up and moved from MES to ES |
| 5 | T-011 | −$759 | +$8 | −4.6 | 2.3× | Sized up |
| 6 | T-012 | −$724 | +$22 | −4.3 | 1.0× | Normal size |
| 7 | T-020 | −$2,398 | −$100 | −4.2 | 1.9× | Traded outside their usual hours (see Task 5) |
| 8 | T-010 | −$664 | −$61 | −3.4 | 3.8× | Sized up |
| 9 | T-008 | −$459 | −$29 | −2.8 | 0.4× | Smaller than usual |
| 10 | T-013 | −$100 | −$8 | −1.9 | 1.1× | Borderline: +$26 before commissions, 180 contracts cost $126 in fees |

All ten have their worst session of the whole history today.

**Sanity checks**
- **Ranking by raw dollar loss instead would share only 3 of the 10 names.** That list is filled by T-001 to T-007,
  large traders for whom a $2k day is ordinary. This is the check that the normalization is doing real work.
- **With median and MAD instead of mean and std, 8 of 10 names stay.** T-010 and T-013 drop out and T-044 and T-029 come in.
  With only about 10 prior sessions per trader the std is noisy, so positions 8–10 are the least reliable part of the list.

**What I cut.** Account rules (daily loss limit, trailing drawdown) are not in the data beyond `max_position_size`,
so "distance to liquidation" is not part of the score. I'd want it first (see Task 6).

---

## Task 2: Labels and evaluation

### (a) Labels

I would label a session as **bad** if either (1) it broke an account rule, meaning a daily loss limit, trailing drawdown
or max position hit, which is objective and is what actually costs the firm money, or (2) its net PnL was more than
3 standard deviations below the trader's own trailing baseline, computed only from earlier sessions, with at least 10
of them. Rule (1) is the part that is easy to defend: it is a fact, not an opinion. Rule (2) catches the sessions that
*nearly* breached, or that breached on an account with loose rules. It is also where the ambiguity lives: the threshold is
arbitrary, new traders have no baseline, a session that is bad at 09:30 may recover by the close (so the label must be
taken at the close), and a trader who changes style (bigger size, new instrument) has a baseline that no longer describes them.
To check agreement, I would have two risk reviewers label the same ~200 sessions independently, with the rule written down
and no access to each other's labels, and compute Cohen's kappa. Below ~0.6 means the definition is not operational yet,
and the disagreements show which part of the wording to fix.

**First model.** Logistic regression (or a small gradient-boosted tree) predicting "this session ends bad" from
information available *during* the session: PnL so far relative to the trader's baseline, size versus usual, number of
round trips versus usual, trading outside usual hours, instruments new to the trader, distance to the loss limit, and
days since account start. Split by **time**: train on the earliest sessions, validate on the following weeks, test on
the most recent, never shuffling sessions from the same week across sets. The per-trader baselines must be recomputed
inside each split from past data only.

### (b) Evaluation

The ranking is good if the traders on it are the ones who **then** get into trouble. I would measure **precision@10**
(how many of the ten go on to breach a rule or get liquidated within, say, the next 3 sessions), **recall of breaches**
(how many of the actual breaches were on yesterday's list), and **lead time** (how many hours before the breach the
trader first appeared). The top 10 is fixed by reviewer capacity, so precision@10 is the metric that matters.
This dataset gives me none of the outcomes. I would need liquidation and rule-breach events, the account rule settings,
and ideally what the reviewer did with each flag and whether it was useful.

**One way this could mislead.** The easy mistake is to evaluate the ranking against labels defined by the same z-score
(rule 2 above). The top 10 would score perfectly because it is measuring itself. A second, subtler one is in this dataset:
today's session is **partial** (up to 09:30 CT), while the baseline is built from full sessions. If you backtest on complete
historic sessions instead of replaying them up to the same time of day, the scores look sharper than what the reviewer
actually gets at 09:30. Features computed with future information, such as a baseline that includes the scored session, as
in the Task 4 code, have the same effect.

---

## Task 3: Slides

Two slides for the risk desk: [`slides/risk-desk-briefing.pdf`](slides/risk-desk-briefing.pdf).

1. **How the list is built.** Two real traders side by side: T-001 lost $2,171 and is *not* flagged (about one normal swing for them),
   while T-018 lost $1,517 and is #1 (8.9 of their normal swings). Then the three steps in plain words.
2. **Today's list.** The ten traders with loss so far, usual day, swings below normal and one reason to look, plus a
   "how much to trust it" box: the session is still open, history is short, a flag is not a rule breach, and T-024 is new and unranked.

Speaker notes are included in the deck.

---

## Task 4: Code review of `session_features`

**Verdict: Request changes. This must not ship.** Blocking issues first, the one to fix first on top.

**Blocking**
1. **Fix this first: the feature leaks the label.** `hist = sessions[:i + 1]` includes the current session, so
   `avg_session_pnl` contains today's `net_pnl`, and `label_bad` is also computed from today's `net_pnl`. The model will
   learn "a low average predicts a bad session" and look great offline for the wrong reason. Use `sessions[:i]` and decide
   what to return when there is no history (NaN, not 0).
2. **History is not per trader.** The docstring says one dict per (trader, session) ordered by `session_date`. If the list
   holds several traders, `hist` mixes all of them. Group by `trader_id` first, or assert that there is only one trader.
3. **`POINT_VALUE = 20` is only right for NQ.** It is 10× too big for MNQ, 2.5× too small for ES, and 6,250× too small for 6E.
   `avg_price` across instruments in a mixed session means nothing either. Notional has to be computed per instrument with
   its own point value, then summed.
4. **`pace` crashes when `minutes_active == 0`** (a single-fill session). It also counts fills, and one order can fill in
   several pieces, so use orders.

**Comments (non-blocking, but I'd want them in this PR or the next)**
- `label_bad` does not belong in a feature function, and a fixed −$1,000 contradicts the per-trader point of the model:
  it is noise for a large trader and a disaster for a one-lot trader. Move labelling to its own step with a relative definition.
- `qty_total × avg_price` adds buys and sells together. Say whether this is traded notional or exposure, since they answer different questions.
- Is `net_pnl` after commissions? It should be, and the docstring should say so.
- The ordering is assumed, not checked. Sort by `session_date` or assert it.
- Recomputing the average from scratch each step is O(n²). A running sum is simpler. This one is a preference at current sizes.
- No tests. A 3-session fixture would have caught issues 1 and 4.

---

## Task 5: Reply to the client

> Hi R., I rechecked T-020 against their own history before replying.
> They are partly right: their order size was normal (at most 6 MES per order, the same as in their previous 11 sessions) and they stayed in MES.
> What changed is *when* and *how often* they traded: in all 11 previous sessions they traded between about 05:30 and 15:20 Chicago time, but today 8 of their 11 fills came outside those hours, six at the evening Globex open and two around 02:00 CT.
> Those off-hours trades account for about $2,076 of the $2,361 realized loss, and overall they placed 10 orders and 52 contracts, against a usual 6 orders and 20–34 contracts.
> Net of commissions they are at −$2,398 so far, almost double their worst previous session (−$1,267).
> Two caveats: the session is not over (our data stops at 09:30 CT), and the flag means the day is unusual for them, not that they broke a rule or did anything wrong.
> I'd suggest the account manager tell them the flag is about the pattern (new hours, more round trips), not their size, and ask whether the evening trading was intentional.
> I'll keep them on tomorrow's list and send you an update after today's close.

---

## Task 6: Reflection

_TODO: write in your own words. Ideas to react to:_

- _Proudest decision: normalizing by the trader's own history, and backing it with a check (the dollar ranking shares
  only 3 of 10 names). The cost: traders with little history, like T-024 who lost $1,580 on day one, cannot be ranked, and
  with about 10 sessions the std is noisy._
- _Second day: add distance to the account's loss limit or drawdown (the thing that actually liquidates), replay history up
  to 09:30 CT so today is compared like for like, and a peer baseline for new traders. From ArrowFin: account rule settings,
  liquidation and breach history to evaluate against, and whether the reviewer acts once per day or intraday._
