# Build an Order Book in C++ — A Beginner-Friendly Learning Guide

This guide walks you through building an **order book** — the core piece of
software that stock exchanges and trading firms use to match buyers with sellers
— in C++. It's written so you can build it *yourself*, step by step, and
understand *why* each piece works the way it does.

It deliberately does **not** give you finished code to copy. You get the ideas,
the reasoning, the algorithms explained in plain steps, and small worked
examples you can turn into tests. Writing the actual C++ is your job — that's
where the learning happens.

**Jargon promise:** every specialised term is explained the first time it
appears. If you hit one that isn't, that's a bug in this guide.

**What you should already know:** enough C++ to write a struct, a class, and a
loop, and to use the standard containers `std::vector`, `std::map`, and
`std::unordered_map`. You do *not* need any finance background.

---

## Table of contents

1. [What is an order book?](#1-what-is-an-order-book)
2. [The rules of matching (learn these on paper first)](#2-the-rules-of-matching-learn-these-on-paper-first)
3. [The key design decisions, explained](#3-the-key-design-decisions-explained)
4. [The interface you'll build](#4-the-interface-youll-build)
5. [The step-by-step build plan](#5-the-step-by-step-build-plan)
6. [Checking your work: invariants and tests](#6-checking-your-work-invariants-and-tests)
7. [Measuring speed (benchmarking)](#7-measuring-speed-benchmarking)
8. [Making it faster (stretch goals)](#8-making-it-faster-stretch-goals)
9. [Turning it into a GitHub repo](#9-turning-it-into-a-github-repo)
10. [Where to read more](#10-where-to-read-more)

---

## 1. What is an order book?

Imagine a marketplace for one thing — say, shares of a company. At any moment,
some people want to **buy** and some want to **sell**, and each person names a
price. An **order book** is simply the organised list of all those offers.

Some vocabulary, all of it plain once you see it:

- An **order** is one person's request: "buy 10 at 101" or "sell 5 at 102."
- A **limit order** is an order with a price limit: "buy 10, but pay *no more
  than* 101." (Almost every order you'll handle is a limit order.)
- The **buy side** is called the **bids**. The **sell side** is called the
  **asks** (also "offers").
- The **best bid** is the *highest* price any buyer is currently willing to pay
  — because if you're selling, the highest buyer is the best one for you. The
  **best ask** is the *lowest* price any seller will accept — because if you're
  buying, the cheapest seller is the best one for you. This "highest buy /
  lowest sell" flip trips people up, so hold onto it.
- A **price level** is the group of all orders sitting at the exact same price.
  For example, everyone who wants to buy at 100 forms one price level.
- An order that is sitting on the book waiting is called a **resting** order.

Now the important part: **matching.** When a new order arrives that *can* trade
with the other side, the order book pairs them up and a **trade** happens. For
example, if someone is resting a sell order at 101 and you send a buy order
willing to pay 101, those two match and trade. An order that can trade
immediately like this is called **marketable** or **aggressive** (it's
"crossing" the gap between buyers and sellers). The unlucky-sounding but standard
names for the two sides of a trade:

- The **maker** is the order that was already resting on the book (it "made"
  liquidity available for others).
- The **taker** is the incoming order that trades against it (it "takes" that
  liquidity).

The whole program you're building is often called a **matching engine**,
because matching incoming orders against resting ones is its main job.

**What your order book must be able to do:**

- **Add** a new order (which may trade immediately, rest, or both).
- **Cancel** an order that's resting (someone changed their mind).
- Report the **best bid** and **best ask** at any time.
- Report how much quantity is resting at a given price.

**A note on speed.** Trading firms care enormously about doing these operations
*fast*, even when the book holds millions of orders. So throughout this guide,
when we make a design choice, the goal is that adding, cancelling, and reading
the top of the book stay quick no matter how large the book grows. We'll make
that concrete as we go — you don't need to worry about it yet.

**Your task for this section:** be able to explain, in your own words, why the
best bid is the *highest* buy price but the best ask is the *lowest* sell price.

---

## 2. The rules of matching (learn these on paper first)

Before writing any code, get comfortable running the matching rules by hand.
These same examples become your first tests later.

**The rules, stated plainly:**

1. **Better price goes first.** A buyer offering more, or a seller asking less,
   gets served before others.
2. **Same price? First come, first served.** If two orders sit at the same
   price, the one that arrived *earlier* trades first. Think of a queue at a
   bakery counter: same counter, earlier arrival gets served first. The formal
   name for "better price first, then earlier arrival" is **price–time
   priority**. When you place a *new* resting order, it joins the *back* of the
   line at its price.
3. **The trade happens at the resting order's price**, not the newcomer's limit.
   This one surprises people. If a seller is resting at 101 and a buyer arrives
   willing to pay *up to* 105, they trade at **101** — the buyer got a good deal.
   The resting order's price is the one that counts.
4. **An order can be filled in pieces.** "Fill" just means "trade." If you want
   to buy 10 but only 4 are available right now, you get a **partial fill** of 4,
   and the leftover 6 is called the **remainder**. One incoming order can be
   filled by several resting orders, possibly across several price levels, all in
   one go.

**What happens to a remainder** depends on an instruction attached to the order
called its **time-in-force** — basically "how long should this order stay
alive?" The three you'll support:

- **GTC** — "Good 'Til Cancelled." Keep any leftover resting on the book until
  it trades or the owner cancels it. So a "GTC remainder" is just: the part that
  didn't trade, parked on the book to wait.
- **IOC** — "Immediate Or Cancel." Trade whatever you can *right now*, and throw
  the leftover away (don't rest it).
- **FOK** — "Fill Or Kill." Only trade if the *whole* order can be filled
  immediately; otherwise do nothing at all.
- (A **market order** — "buy/sell right now at any price" — is just a marketable
  order with no price limit. You can treat it as a limit order priced at the
  worst possible value, behaving like IOC.)

**Four worked examples — trace each by hand until it's obvious:**

- **Example A (a simple trade).** The book has one resting sell: sell 10 at
  price 101. A buy order for 4 at 101 arrives. → They trade 4 at price 101. The
  sell order now has 6 left. The buyer is fully done and rests nothing.

- **Example B (first come, first served).** Two sells rest at 101: first 5
  (older), then another 5 (newer). A buy for 6 at 101 arrives. → The buyer trades
  5 against the *older* sell (it empties), then 1 against the newer sell. The
  newer sell has 4 left.

- **Example C (sweeping several prices).** Resting sells: 3 at price 101, and 3
  at price 102. A buy for 5 arrives willing to pay up to 102. → It trades 3 at
  101 (the better price, taken first), then 2 at 102. The best ask is now 102
  with 1 remaining.

- **Example D (leftover rests).** Resting sell: 3 at 101. A buy for 10 at 101
  arrives, marked GTC. → It trades 3 at 101, then the remaining 7 rests on the
  book as a *buy* order at 101. The sell side is now empty, so there's a best bid
  of 101 and no best ask at all.

**Your task:** work Examples A–D on paper. When their outcomes feel automatic,
you're ready to design the data structures.

---

## 3. The key design decisions, explained

These are the choices that make an order book correct *and* fast. Understanding
*why* matters more than any clever trick later. Read this section slowly.

### 3.1 Store prices as whole numbers, never decimals

Computers store decimals (like `101.10`) approximately, which quietly breaks
equality checks and adds up tiny errors — a disaster when you're matching money.
The fix everyone uses: represent every price as a **whole number of "ticks."** A
tick is just the smallest price step allowed (for example, cents). So `$101.10`
becomes the integer `10110`. You'll define short type names for these — an order
id, a price (a signed 64-bit integer), a quantity (an unsigned 64-bit integer),
and a two-value "side" (buy or sell) — but the key rule is simply: **prices are
integers, never floating-point.**

### 3.2 Keep all orders in one big reusable array, and refer to them by number

You'll create many orders and destroy many orders. If you ask the operating
system for fresh memory (`new`/`malloc`) every single time, that's slow and
occasionally *very* slow — an unpredictable delay you can't afford.

Instead, allocate one large array of order slots up front, like a tray of
numbered pigeonholes, and reuse the slots. When you need to refer to a specific
order elsewhere in your program, refer to it by its **slot number** (its index in
the array), *not* by a memory pointer to it.

Why a number and not a pointer? Because when that array runs out of room and has
to grow, the computer may move the whole array to a new location — and any raw
pointers into the old location would then point at garbage. Slot *numbers* stay
correct across a move. (As a bonus, a 32-bit number is smaller than a 64-bit
pointer, so your data packs more tightly, which the CPU likes.)

You'll also want one special "this points to nothing" number — a **sentinel**.
People usually pick the largest possible 32-bit value and give it a name like
`NIL`.

### 3.3 Never do slow work while matching — reuse slots with a "free list"

Building on 3.2: to reuse slots cheaply, keep a simple to-do list of which slots
are currently free. This is called a **free list**. When you need a slot, take
one off the free list; when you delete an order, put its slot back on the free
list. A neat trick: you can store the free list *inside the unused order slots
themselves* (each free slot just remembers the next free slot), so it costs no
extra memory. Grabbing and returning a slot then becomes near-instant.

### 3.4 Give each price level a simple queue built into the orders

Remember rule 2: orders at the same price are served first-come-first-served. So
each price level needs a **queue** (a line). The classic, fast way to build this
is a **doubly-linked list**: each order remembers the order in front of it and
the order behind it. "Doubly-linked" just means the links go both directions,
which lets you remove any order from the middle instantly.

Here's the efficient twist: store those two links (front-neighbour and
back-neighbour) *inside the order itself*, as two more slot-numbers. A linked
list whose links live inside the items themselves is called **intrusive**. The
payoff: adding an order to the back of the line, and removing one from the front,
take a fixed tiny amount of work — and you never allocate memory to do it,
because the orders already exist in your array from 3.2.

### 3.5 Make cancelling instant with a lookup table

When someone cancels order #500, you must find it fast — you can't go searching
through the whole book. So keep a **lookup table** that maps each order's id
directly to its slot number. In C++ that's a `std::unordered_map` (a hash map —
think of a coat-check: you hand over your ticket and get your exact coat back
immediately, no searching). To cancel: look up the id, unhook that order from its
queue (easy, thanks to the doubly-linked list in 3.4), and return its slot to the
free list.

### 3.6 The one real decision: how to store the price levels

You have many price levels, and you constantly need two things: *find the best
price quickly*, and *find any specific price quickly*. There are two sensible
ways to organise them, and you should start with the simpler one.

**Option 1 — a sorted map (start here).** Use C++'s `std::map`, which keeps its
entries sorted by key automatically. Keep one map for bids and one for asks. Here
is a small but genuinely useful trick: `std::map` lets you choose the sort
direction. Sort the **bids** map so the *highest* price comes first, and leave
the **asks** map sorted so the *lowest* price comes first. Then, for *both* maps,
"the best price" is simply "the first entry" — no special cases. (In C++ terms:
give the bids map the `std::greater<>` comparator; leave asks default.) This
option handles any price range and is easy to get right. Finding a level is fast
but gets very slightly slower as the number of *distinct prices* grows.

**Option 2 — a flat array indexed by price (do this later, once it works).**
Because a price is now a whole number (3.1), you can use it directly as an index
into a plain array of levels: level for price *p* lives at array position
*p − base*, where *base* is the lowest price you're tracking. Looking up any
level is then instant, no matter how many there are. The catch: you must decide a
fixed price *range* up front, and you must separately keep track of where the
best price currently is (and move that marker along when a level empties). In
real markets almost all activity clusters in a narrow band of prices, so this
works beautifully — but it's more bookkeeping, so save it for a later milestone.

**Recommendation:** build Option 1 first and get everything working and correct.
Only switch to Option 2 after you've *measured* that finding levels is actually
your slow point.

### 3.7 The information each thing needs to remember (a sketch, not code)

Deciding *what to store* is half the design. Here's the state you'll want. You
still write the actual structs and all the logic — this is just the shopping
list:

- **An order** remembers: its id, its price, how much quantity is *still*
  unfilled, its side (buy or sell), and the two link slot-numbers for the queue
  it's in (front-neighbour and back-neighbour). Those same two link fields can
  double as the free-list link when the slot is unused.
- **A price level** remembers: the total quantity resting there (kept as a
  running sum so you never have to add it up on demand), plus the slot-numbers of
  the first and last orders in its queue.
- **The book as a whole** remembers: the bids container and the asks container
  (from 3.6), the big order array and its free-list marker (3.2–3.3), and the
  id-to-slot lookup table (3.5). A small scratch list to collect the trades
  produced by one incoming order is handy too.

---

## 4. The interface you'll build

Aim for a small, clear set of public operations. You'll implement these; deciding
exactly how is part of the exercise:

- **add(id, side, price, quantity, time-in-force)** — try to match the new order
  against the other side, then (if it's GTC) rest whatever is left. It should
  hand back the list of trades it produced.
- **cancel(id)** — remove a resting order; report back whether the id was found.
- **best_bid()** and **best_ask()** — the current best prices, or "nothing" if
  that side is empty.
- **quantity_at(side, price)** — how much is resting at a given price.

A **trade** you report should include: the taker's id, the maker's id, the price
it happened at (the maker's price — rule 3), and the quantity traded.

For time-in-force, support GTC, IOC, and FOK as described in section 2.

---

## 5. The step-by-step build plan

Build in the order below. Each **milestone** has a goal, hints, and
**checkpoints** — small examples with a known correct outcome that you should
turn into automated tests. Rule: don't start the next milestone until the current
one's checkpoints pass.

### Milestone 0 — Set up the skeleton
Define your basic types (from 3.1), the "side" and "time-in-force" choices, the
"trade" record, and an empty book class with do-nothing method stubs. Pick your
`NIL` sentinel value.
**Checkpoint:** it compiles, and a brand-new empty book reports no best bid and
no best ask.

### Milestone 1 — Resting orders and the top of the book (no matching yet)
For now, pretend every order just rests (skip matching entirely). Implement:
placing an order at the back of its price-level queue, and the `best_bid`,
`best_ask`, and `quantity_at` reads, using the two-map layout from 3.6.
**Checkpoints:**
- Add buy 10 at 100, buy 5 at 99, sell 7 at 101 → best bid is 100, best ask is
  101, and the quantity resting at buy-price 100 is 10.
- Adding two orders at the same price makes the quantity there add up.

### Milestone 2 — The reusable order array and free list
Introduce the big order array and the free-list reuse of slots (3.2–3.3). Route
resting through it. After a warm-up, placing and removing orders shouldn't be
asking the operating system for memory each time.
**Checkpoint:** place an order, remove it, place another — the freed slot gets
reused and nothing gets corrupted (you'll verify this properly with the checks in
section 6).

### Milestone 3 — The first-come-first-served queue per price level
Turn each price level's orders into the doubly-linked, intrusive queue from 3.4:
new orders join the back, and you can walk from the front. Keep each level's
running total quantity correct as orders come and go.
**Checkpoint:** three orders placed at one price form a front-to-back line in
arrival order, and the level's running total equals the sum of their quantities.

### Milestone 4 — Matching (the heart of it)
Now make an incoming order actually trade. Here is the logic in plain steps —
work out the C++ yourself:

1. Look at the best price level on the *opposite* side (asks if the incoming
   order is a buy; bids if it's a sell).
2. Does it "cross"? For an incoming buy, it crosses only if that best sell price
   is *at or below* the buyer's limit. For an incoming sell, only if that best
   buy price is *at or above* the seller's limit. If it doesn't cross, stop —
   there's nothing to trade.
3. If it crosses, trade against the order at the *front* of that level's queue
   (the oldest one). The amount traded is the smaller of "what the newcomer still
   wants" and "what that resting order has left."
4. Record a trade at the *resting* order's price (rule 3). Reduce both the
   newcomer's remaining quantity and the resting order's remaining quantity by the
   amount traded, and reduce the level's running total.
5. If that resting order is now completely filled, remove it from the front of
   the queue, delete it from the id lookup table, and return its slot to the free
   list.
6. Keep going through that level's queue until either the newcomer is fully
   filled or the level is empty. If the level empties, remove it and move to the
   next-best price. Stop when the newcomer is filled or nothing crosses anymore.

Then, back in `add`: after matching, if any quantity remains *and* the order is
GTC, rest it (Milestone 1's placing logic).
**Checkpoints:** reproduce Examples **A, B, C, and D** from section 2 exactly —
including that trades happen at the maker's price and that older orders at a price
fill before newer ones.

### Milestone 5 — Instant cancel
Implement `cancel` using the lookup table (3.5): find the slot, unhook the order
from its queue, fix the level's running total, remove the level if it just became
empty, return the slot to the free list, and remove it from the lookup table.
**Hint:** carefully handle every position the order might be in its queue — the
only one, the front, the back, or somewhere in the middle. Getting the neighbour
links right here is the classic place for bugs.
**Checkpoints:**
- Two orders resting at 100: cancel the first → only the second's quantity
  remains; cancel the second → the best bid disappears.
- Cancelling an id that doesn't exist reports "not found."

### Milestone 6 — The IOC and FOK rules
IOC: after matching, simply don't rest the leftover. FOK: *before* you trade
anything, check whether the whole order could be filled right now at its price or
better; if not, do nothing and report no trades; if yes, proceed normally.
**Hint for FOK:** walk down the opposite side from the best price, adding up the
quantities available at prices that cross the limit, and stop as soon as the
running sum reaches what the order needs.
**Checkpoints:**
- IOC buy 10 at 101 against a resting sell of 4 at 101 → trades 4, rests nothing.
- FOK buy 10 at 101 when only 4 are available → *no* trades and the book is
  untouched; FOK buy 4 at 101 when exactly 4 are available → fully trades.

### Milestone 7 — The faster level storage (optional but worthwhile)
Swap the sorted-map level storage for the flat array indexed by price (Option 2
in 3.6). Everything else — the reusable array, the intrusive queues, cancelling —
stays the same; only how you find a level changes. You'll add a marker for where
the best price currently is, and move it to the next non-empty level whenever the
current best empties.
**Checkpoint:** re-run *all* the earlier checkpoints against this new version and
get identical results.

### Milestone 8 — Measure it
See section 7. Build a little speed test, measure honestly, and write the numbers
into your project's README.

### Milestone 9 — Prove it with random testing
Build a deliberately simple, obviously-correct second version of the book (speed
doesn't matter — just clarity). Then generate long streams of random adds and
cancels, feed them to *both* versions, and check that they always agree on the
trades and on the best bid/ask. This is called **differential testing**, and it
catches subtle bugs that hand-written examples miss. It's also the thing that
makes your project look genuinely rigorous.

---

## 6. Checking your work: invariants and tests

Turn every example above into an automated test. Beyond examples, write one
helper — call it something like `verify()` — that checks a handful of "things
that must *always* be true," and call it after every operation while you're
developing. An "always true" statement like this is called an **invariant**.
Yours:

- If both sides have orders, the best bid is *below* the best ask. (After
  matching, the book should never be left in a state where a buyer and seller
  who could trade are both still waiting.)
- Each price level's stored running total equals the actual sum of its orders'
  remaining quantities.
- Every id in your lookup table points to a real order whose id matches, and each
  resting order appears in exactly one level's queue, exactly once.
- The queue links are consistent: a level has "no front" exactly when it has "no
  back," which is exactly when its total is zero.

While developing, compile with your compiler's **sanitizers** turned on
(`-fsanitize=address,undefined`). These are free bug-detectors built into the
compiler that catch things like using a slot you already freed — exactly the kind
of mistake this style of code invites.

---

## 7. Measuring speed (benchmarking)

Measuring speed is called **benchmarking**. "Fast" isn't one number, so be
clear about what you're measuring:

- **Latency** — how long *one* operation takes. Here the *worst* cases matter as
  much as the typical case: a single slow operation can be very bad. People report
  latency as **percentiles** — for example, "p50" is the median (half of
  operations were faster), and "p99" means 99% were faster and 1% were slower.
  That slow 1% is called the **tail**, and taming it is much of the art.
- **Throughput** — how many operations per second you can sustain.
- **Consistency** — low variation. A steady, predictable speed is often worth
  more than a faster-on-average one that occasionally stalls.

A trustworthy speed test:

1. Fill the book with a realistic number of resting orders first.
2. Then run a realistic *mix* of operations. In real markets, **cancels are the
   most common operation by far** — most messages are orders being placed and
   quickly cancelled, not trades. Include some immediately-trading orders too.
3. Time each operation and report several percentiles (p50, p99, p99.9, and the
   worst), never just the average.
4. Do a **warm-up** run first and throw its numbers away. Set aside all your
   memory up front so the measurement itself isn't doing slow allocation.
5. Note the machine, compiler, and settings you used, so others can reproduce it.

If you ever see a huge worst-case sitting next to a small typical case, that's
almost always your containers quietly resizing themselves mid-run — the classic
cause of tail spikes, and a good motivation for the faster storage in Milestone 7.

---

## 8. Making it faster (stretch goals)

Once it's correct and measured, here are worthwhile improvements, roughly
best-value-first. Do them one at a time and re-measure after each — some help
only in certain situations.

- **Use a faster lookup table.** The standard `std::unordered_map` is convenient
  but not the fastest; specialised "flat" hash maps are friendlier to the CPU's
  memory cache and often shrink the slow tail.
- **Adopt the flat-array level storage** from Milestone 7, once your price range
  is settled.
- **Keep the top of the book cached** in plain variables you update on every
  change, so reading the best prices is instant.
- **Remove decisions ("branches") from the busiest code** so the CPU can run it
  more predictably.
- **Threading and memory-layout tricks** (keeping data the CPU accesses together
  physically close, avoiding two CPU cores fighting over the same memory line,
  and so on) — advanced, and only worth it after you've measured.
- **Real-exchange features** you could add for depth: preventing someone from
  trading with themselves, hidden ("iceberg") quantity, sharing fills
  proportionally instead of first-come-first-served (some markets do this), and
  fees.

---

## 9. Turning it into a GitHub repo

A tidy layout that reads well:

```
orderbook/
├── include/       your order book code (a single header file is fine)
├── tests/         your correctness tests and your speed test
├── CMakeLists.txt the build instructions
├── README.md      your write-up
└── LICENSE
```

What makes a project look thoughtful rather than thrown-together:

- **Automatic testing on every change.** GitHub Actions can build your project
  and run your tests each time you push. Add a second run with the sanitizers
  (section 6) turned on.
- **A short "Design" section in your README** explaining the *why* behind your
  choices (whole-number prices, slot-numbers instead of pointers, the reusable
  array, the built-in queues, and which level storage you picked). Reviewers read
  that first.
- **Honest speed numbers**, with the machine and settings stated.
- **A clear license** (MIT or Apache-2.0 are common, permissive choices).

Write the build file (`CMakeLists.txt`) yourself as part of the exercise — it's
short: describe your code, make one runnable program per test file, and register
those tests so a single command runs them all.

---

## 10. Where to read more

- **"How to Build a Fast Limit Order Book"** by WK Selph — the well-known article
  behind the design in this guide (reusable array + built-in queues + id lookup
  table). Read it *after* you've attempted Milestones 1–5, so you're comparing it
  to your own solution rather than copying.
- **The QuantCup order-book challenge** and its published solutions — good for
  seeing the flat-array approach pushed to its limits.
- **Real exchange documentation** (search for a stock exchange's "matching engine"
  or "market data" specification) — for the fuller set of real-world rules beyond
  the basics here.

---

*Build the milestones in order, write the tests as you go, and resist looking up
finished solutions — including the articles above — until you've made your own
attempt. The reasoning in section 3 and the always-true checks in section 6 are
the parts genuinely worth internalising. Good luck.*
