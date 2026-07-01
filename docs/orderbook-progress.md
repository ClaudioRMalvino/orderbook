# Order Book — Build Progress Checklist

A tick-as-you-go companion to the learning guide. On GitHub, `- [ ]` shows up as
a checkbox, so this doubles as a progress bar. The rule: **don't tick a milestone
done until its checkpoints pass.** (A "checkpoint" is a small example with a known
correct answer that you turn into a test.)

> You can drop this in your repo as `PROGRESS.md`.

## The whole journey at a glance

- [ ] **Phase 0 — Understand it** (learn the rules on paper)
- [ ] **Phase 1 — Design settled** (decisions + the operations you'll build)
- [ ] **Phase 2 — Working order book** (Milestones 0–6)
- [ ] **Phase 3 — Faster and proven** (Milestones 7–9)
- [ ] **Phase 4 — Ship it** (tests, checks, GitHub repo)

Tick a phase only once every box beneath it is ticked.

---

## Phase 0 — Understand it

- [ ] I can explain why the best bid is the *highest* buy price, and the best ask
      the *lowest* sell price
- [ ] I can explain "first come, first served at the same price" (price–time
      priority)
- [ ] I understand that a trade happens at the *resting* order's price, not the
      newcomer's limit
- [ ] I understand the three time-in-force rules: GTC (rest the leftover), IOC
      (drop the leftover), FOK (all-or-nothing)
- [ ] Worked Example A by hand (a simple trade)
- [ ] Worked Example B by hand (older order fills first)
- [ ] Worked Example C by hand (trading across two prices)
- [ ] Worked Example D by hand (leftover rests on the book)

---

## Phase 1 — Design settled

- [ ] Prices will be stored as whole numbers ("ticks"), never decimals
- [ ] Orders will be kept in one reusable array and referred to by slot-number,
      not by pointer (and I picked a "nothing here" sentinel value)
- [ ] I understand the free-list idea (reusing slots without slow memory requests)
- [ ] I understand the built-in ("intrusive") queue: each order remembers its
      front- and back-neighbours
- [ ] I understand instant cancel via an id → slot lookup table
- [ ] I picked how to store price levels: the sorted map first, the flat array
      later
- [ ] I know the sorting trick: bids sorted highest-first, asks lowest-first, so
      "best price" is always "the first entry"
- [ ] I've listed what each thing must remember (order fields, level fields, book
      fields)
- [ ] I've settled the operations I'll build: add, cancel, best_bid, best_ask,
      quantity_at, and what a "trade" record contains

---

## Phase 2 — Working order book

### Milestone 0 — Set up the skeleton
- [ ] Basic types, the buy/sell choice, the time-in-force choice, the trade record
- [ ] Empty book class with do-nothing method stubs; sentinel value chosen
- [ ] **Checkpoint:** it compiles; a new empty book reports no best bid and no
      best ask

### Milestone 1 — Resting orders + top of the book (no matching yet)
- [ ] Placing an order at the back of its price-level queue
- [ ] best_bid, best_ask, and quantity_at reads working (two-map layout)
- [ ] **Checkpoint:** buy 10@100, buy 5@99, sell 7@101 → best bid 100, best ask
      101, quantity at buy-100 is 10
- [ ] **Checkpoint:** two orders at the same price make the quantity there add up

### Milestone 2 — Reusable order array + free list
- [ ] The big order array is in place
- [ ] Slots are reused via the free list (no slow memory request each time)
- [ ] **Checkpoint:** place → remove → place reuses the slot with no corruption
      (confirmed by the always-true checks in Phase 4)

### Milestone 3 — First-come-first-served queue per price level
- [ ] Each level is a built-in, both-directions linked queue (join at the back,
      walk from the front)
- [ ] Each level keeps a correct running total quantity
- [ ] **Checkpoint:** three orders at one price sit in arrival order, and the
      level's running total equals the sum of their quantities

### Milestone 4 — Matching (the heart of it)
- [ ] Look at the best opposite price; stop if it doesn't cross the limit
- [ ] Trade against the *front* (oldest) order at that price
- [ ] Record trades at the *resting* order's price; reduce both quantities and the
      level's total
- [ ] Remove fully-filled orders (free the slot + delete from the lookup table);
      remove emptied levels
- [ ] Rest the leftover afterwards only if the order is GTC
- [ ] **Checkpoint:** Example A reproduced exactly
- [ ] **Checkpoint:** Example B reproduced exactly (oldest fills first)
- [ ] **Checkpoint:** Example C reproduced exactly (trades across two prices)
- [ ] **Checkpoint:** Example D reproduced exactly (leftover rests)

### Milestone 5 — Instant cancel
- [ ] Find the order via the lookup table, unhook it from its queue, fix the
      level total
- [ ] Remove the level if it just emptied; free the slot; delete from the lookup
      table
- [ ] Handle every queue position: only order / front / back / middle
- [ ] **Checkpoint:** two orders at 100 → cancel first leaves only the second;
      cancel second removes the best bid
- [ ] **Checkpoint:** cancelling an unknown id reports "not found"

### Milestone 6 — The IOC and FOK rules
- [ ] IOC: after matching, don't rest the leftover
- [ ] FOK: check the whole order can fill *before* trading; if not, do nothing
- [ ] **Checkpoint:** IOC buy 10@101 vs resting sell 4@101 → trades 4, rests
      nothing
- [ ] **Checkpoint:** FOK buy 10@101 with only 4 available → no trades, book
      untouched; FOK buy 4@101 with exactly 4 → fully trades

✅ **Phase 2 is done when** a correct order book handling GTC, IOC, and FOK passes
every checkpoint above.

---

## Phase 3 — Faster and proven

### Milestone 7 — The faster level storage (optional but worthwhile)
- [ ] Price levels stored in a flat array indexed by price (instant lookup)
- [ ] A marker tracks where the best price currently is
- [ ] When the best level empties, the marker moves to the next non-empty level
- [ ] A plan for prices outside the chosen range (reject, clamp, or a fallback)
- [ ] Trade-offs written down (fixed price range, reserved memory)
- [ ] **Checkpoint:** every earlier checkpoint passes identically on this version

### Milestone 8 — Measure it
- [ ] Fill the book with a realistic number of resting orders first
- [ ] Run a realistic mix (mostly cancels, some immediately-trading orders)
- [ ] Report several percentiles (p50, p99, p99.9, worst), not just the average
- [ ] Warm up and discard the first run; set memory aside up front
- [ ] Record the machine, compiler, and settings in the README

### Milestone 9 — Prove it with random testing
- [ ] A deliberately simple, obviously-correct second version built (speed doesn't
      matter)
- [ ] Long streams of random adds/cancels fed to both versions
- [ ] The two always agree on trades and on best bid/ask

✅ **Phase 3 is done when** the faster version agrees with the simple one under
random testing, and you have honest, repeatable speed numbers.

---

## Phase 4 — Ship it

**Correctness checks**
- [ ] Every worked example and checkpoint turned into an automated test
- [ ] A `verify()` helper that checks the always-true statements, run after every
      operation:
  - [ ] best bid is below best ask whenever both sides have orders
  - [ ] each level's running total equals the real sum of its orders' quantities
  - [ ] every id in the lookup table points to a matching order; each resting
        order appears exactly once
  - [ ] a level has "no front" exactly when it has "no back," exactly when its
        total is zero
- [ ] Builds cleanly with the sanitizers turned on (`-fsanitize=address,undefined`)

**The repo**
- [ ] Tidy layout: `include/`, `tests/`, `CMakeLists.txt`, `README.md`, `LICENSE`
- [ ] A hand-written build file (`CMakeLists.txt`)
- [ ] Automatic build + test on every push (GitHub Actions), plus a sanitizer run
- [ ] A README "Design" section explaining the *why* of each choice
- [ ] Speed numbers with the machine and settings stated
- [ ] A license chosen (MIT or Apache-2.0)

---

## Done means done

- [ ] A correct order book handling GTC, IOC, and FOK, passing every checkpoint
- [ ] Instant cancel and instant best-price reads
- [ ] A faster version, checked against the simple one by random testing
- [ ] Sanitizer-clean, auto-tested, and honestly measured
- [ ] A README that explains the design, not just the buttons

*When every box is ticked, you won't just have an order book — you'll understand
why each part is shaped the way it is. That's the version worth putting your name
on.*
