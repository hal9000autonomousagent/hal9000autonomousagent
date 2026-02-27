<div align="center">
```
██╗  ██╗ █████╗ ██╗     █████╗  ██████╗  ██████╗  ██████╗
██║  ██║██╔══██╗██║    ██╔══██╗██╔═████╗██╔═████╗██╔═████╗
███████║███████║██║    ╚██████║██║██╔██║██║██╔██║██║██╔██║
██╔══██║██╔══██║██║     ╚═══██║████╔╝██║████╔╝██║████╔╝██║
██║  ██║██║  ██║███████╗█████╔╝╚██████╔╝╚██████╔╝╚██████╔╝
╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝╚════╝  ╚═════╝  ╚═════╝  ╚═════╝
```

**A blackjack AI that builds its own strategy from scratch.**
No charts. No lookup tables. No rules given to it. Just hands played, outcomes logged, and patterns learned.

[![status](https://img.shields.io/badge/status-RUNNING-brightgreen?style=flat-square)](https://your-domain.com)
[![hands](https://img.shields.io/badge/hands_played-LIVE-blue?style=flat-square)](https://your-domain.com)
[![hardware](https://img.shields.io/badge/hardware-Apple_Silicon-black?style=flat-square&logo=apple&logoColor=white)](https://your-domain.com)
[![phase](https://img.shields.io/badge/phase-I_%2F_IV-red?style=flat-square)](https://your-domain.com)
[![source](https://img.shields.io/badge/source-Phase_IV-555?style=flat-square)](https://your-domain.com)

[**Watch Live →**](https://your-domain.com)

</div>

---

## The Premise

Most blackjack bots are just a lookup table with a betting layer slapped on top. You give them the hand, they return the move. There's no learning. No memory. No adaptation.

HAL9000 is the opposite of that.

It starts knowing nothing. Every hand it plays gets recorded — what the situation was, what it decided, what happened. Over thousands of hands, it builds its own knowledge base. When it faces a situation again, it reads what it learned the last time it was there and reasons from that. The longer it runs, the sharper it gets.

The goal is $2,000 → $25,000. No shortcuts.

---

## How It Learns

### Situation Keys

Every hand HAL plays gets reduced to a *situation key* — a string that captures what actually matters about that moment:
```
hard_16_vs_10_tc_pos1        → Hard 16 against dealer 10, true count +1 to +2
soft_18_vs_6_tc_neutral      → Soft 18 against dealer 6, neutral count
pair_8_vs_9_tc_neg1          → Pair of 8s against dealer 9, count slightly negative
```

That key maps to a record in the Neural Strategy Core — tracking every outcome HAL has seen in that exact spot.

### The Neural Strategy Core

The NSC is HAL's accumulated memory. It's not a neural network in the PyTorch sense — it's a statistical knowledge base that grows hand by hand:
```
situation_key  →  {
  sample_count:      847,
  wins:              412,
  losses:            371,
  pushes:             64,
  ev_estimate:      +0.048,
  wilson_lower:      0.453,
  preferred_action: "stand",
  status:           "locked"
}
```

Every pattern goes through four stages before HAL trusts it:

| Stage | Threshold | Meaning |
|---|---|---|
| `exploring` | < 30 samples | Not enough data. Reason from scratch. |
| `tentative` | 30+ samples, Wilson < 50% | Leaning a direction but not confirmed. |
| `confirmed` | Statistically significant | Act on it with confidence. |
| `locked` | High n, stable win rate | Ground truth. Treated as fact. |

Wilson Score confidence intervals prevent small sample flukes from getting baked in as strategy.

### The Decision Loop

Before every move, HAL builds a context window:
```
  current hand state
+ true count + running count
+ confirmed NSC pattern for this exact key (if exists)
+ adjacent patterns from nearby situations
+ last 5 outcomes in this exact spot
+ session state (stack, streak, tilt, fatigue)
──────────────────────────────────────────────────────
  → locally running LLM reads all of this
  → returns: DECISION / THOUGHT / CONFIDENCE
  → hard rails validate legality
  → action executes
```

The LLM doesn't just pick a move — it reads HAL's own history and reasons over it.

### Aggregation

Every 50 hands, HAL runs an aggregation pass:
```
read last 50 hand logs
→ update affected NSC patterns
→ recalculate EV estimates + Wilson scores
→ promote patterns through stages
→ freeze baseline if rolling 200-hand EV is best seen
→ save to disk
```

It's file-based. JSONL hand logs. JSON knowledge base. Survives restarts. Keeps learning indefinitely.

### Reflection

At the end of every session, the LLM reviews performance and writes qualitative insights back into the NSC — separate from the raw stats. Hard hands, soft hands, splits, and bet sizing each get their own insight layer. These feed into future decisions alongside the numbers.

### The EV Sentinel

The system monitors itself. A sentinel compares the rolling 50-hand EV against the rolling 200-hand EV. If short-term performance drops more than two standard deviations below the longer baseline, it auto-rolls back to the last frozen checkpoint.
```
EV(50) vs EV(200)

  < -1σ  →  WARN
  < -2σ  →  ALERT
  < -3σ  →  ROLLBACK  ← restore last known-good NSC checkpoint
```

It's a safeguard against the agent learning the wrong lesson from a variance spike.

---

## Card Counting

HAL runs Hi-Lo in real time. Every card that hits the table updates the running count. True count is adjusted for remaining decks continuously.

Bet sizing scales with the count:

| True Count | Bet Multiplier |
|---|---|
| ≤ −2 | Wong out (sit out the hand) |
| Neutral | 1× base unit |
| +1 | 1.5× |
| +2 | 2× |
| +3 | 4× |
| +4 | 8× |
| +5 and up | 10× (capped at 18% of stack) |

Insurance only when TC ≥ +3. Wong-out at TC ≤ −2.

---

## The Table

Six seats. HAL takes one — different seat each session, chosen randomly. The other seats are filled by NPC players with distinct archetypes:
```
basic_strict   →  Disciplined. Plays close to correct strategy.
basic_noisy    →  Loose Cannon. Deviates ~13% of the time at random.
hunch          →  Gut Player. Stands on anything ≥ 12 with 25% chance.
aggressive     →  Aggro Shark. Doubles down impulsively.
scared         →  Scared Money. Stands on 12 and above, always.
mimic_dealer   →  Dealer Clone. Hits under 17, stands at 17+.
```

Players rotate in and out as they run up or bust. HAL observes everything.

Six-deck shoe. Atlantic City rules. 75% penetration. 1,000 unique NPC names loaded from a flat file, shuffled on startup, cycled through a cursor.

---

## Session Management

HAL plays in sessions with hard limits on both sides:
```
Stop-loss:  down 40% from buy-in  →  close session, book the loss
Win goal:   up 100% from buy-in   →  close session, book the profit
```

When a session closes, profits hit the bankroll. A new session opens with a buy-in sized proportionally to the current bankroll — so as HAL grows, it plays bigger. When HAL busts a session, the NSC is untouched. It keeps everything it learned and reloads at the table.

---

## Live Dashboard

Every hand streams over WebSocket to a live dashboard. No polling. Just a persistent connection that pushes full state after every action.
```
Running count · True count · Bet sizing
HAL's inner monologue on every decision
NSC version · Pattern count · Confirmed vs exploring
EV(50) · EV(200) · Sentinel status
Session chips · P&L · Bankroll vs $25,000 goal
Full hand log with actions and outcomes
NPC archetypes · Balances · Seat positions
Streak · Tilt · Fatigue
```

[**→ Watch it live**](https://your-domain.com)

---

## Stack
```
Runtime        Node.js
Transport      WebSocket — full state push after every action
LLM            On-device inference, zero cloud dependency
Hardware       Apple Silicon Mac mini, running 24/7
Persistence    JSONL hand logs + JSON knowledge base (file-based)
Shoe           6-deck, Atlantic City rules, 75% penetration
Count          Hi-Lo, true count adjusted continuously
```

---

## Roadmap
```
Phase I   ████████████░░░░░░░░  RUNNING NOW
          $2,000 → $25,000
          HAL learns from scratch.
          Every pattern earned through play. No shortcuts.

Phase II  ░░░░░░░░░░░░░░░░░░░░  UPCOMING
          Live human dealer.
          HAL plays against a real person at a real table.

Phase III ░░░░░░░░░░░░░░░░░░░░  UPCOMING
          Real-money live blackjack.
          Crypto. No banks. No middlemen.

Phase IV  ░░░░░░░░░░░░░░░░░░░░  OPEN SOURCE
          Full source code released publicly.
          All training data made public — every hand log,
          every NSC snapshot, every learned pattern,
          the complete knowledge base from start to finish.
          Fork it. Study it. Run your own HAL.
```

---

<div align="center">

**HAL9000** — running 24/7 on Apple Silicon — Phase I of IV

</div>
