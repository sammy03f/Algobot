# Simulated Exchange + Algobot Tournament - Project Specification

## Overview

**What You're Building**: A simulated exchange where multiple algorithmic trading bots compete against each other

**Core Philosophy**: 
- Test-Driven Development (TDD): Write tests first, verify they fail, implement, verify they pass
- Modular Code: Separate concerns so you can swap components without breaking everything
- Reproducibility: Same seed = same results (critical for debugging)

**Why This Matters for Future Focus**:
- Friday = 1 hour to adapt bot to new constraints under pressure
- Success requires: clean code structure, understanding bot behavior, exploiting opponents
- "Don't need stats to make money off other bots" - understand flows and patterns

---

## Recommended Project Structure

```
sim/
├── config.py          # All configuration constants
├── run.py             # Main simulation loop
├── exchange.py        # Market simulation + order matching
├── bot_engine.py      # Bot execution (state tracking, order management)
├── strategy.py        # Decision-making logic (separated from execution)
├── toxic_defense.py   # Flow detection and defensive actions
└── logger.py          # Event logging for replay/debugging

tests/
├── test_exchange.py
├── test_bot.py
└── test_lifecycle.py
```

---

## Stage 1: Minimum Viable End-to-End System
**Goal**: Get a working simulation running that you can reproduce and modify

### Task 1: Create the project files + run loop

**Required Functionality**:
- Create a runnable script that executes a simulation for N_TICKS
- Simulation must be reproducible using a random seed
- All configuration constants (N_TICKS, SEED, SPREAD, etc.) must be stored in a separate config file, not hardcoded

**Learning Goal**: 
Build comfort with project structure and reproducibility. The seed discipline is critical - you'll need this for debugging under time pressure.

**Test Requirements**:
- Write a test that runs the simulation twice with the same seed and verifies identical results

---

### Task 2: Define core data objects (Snapshot, Order, Fill)

**Required Functionality**:

Create data containers for three core types:

**Snapshot** - represents market state at a given tick:
- Current tick number
- Mid price
- Best bid price
- Best ask price
- Optional event flag (for future use)

**Order** - represents a limit order placed by a bot:
- Unique identifier
- Which bot placed it
- Side (BUY or SELL)
- Limit price
- Quantity (shares)
- Status (OPEN, PARTIAL, FILLED, CANCELED)

**Fill** - represents an executed trade:
- Tick when fill occurred
- Which order was filled
- Which bot got filled
- Side (BUY or SELL)
- Execution price
- Quantity filled

**Learning Goal**: 
Practice clean interfaces. Your objects should be easy to print and inspect, which helps massively during debugging.

**Hint**: Python's `@dataclass` decorator can help make objects more readable.

---

### Task 3: Exchange.step() with a simple market model

**Required Functionality**:
- Exchange must maintain a "market mid price" that changes each tick
- Each tick, exchange must output a Snapshot containing:
  - Current tick number
  - Mid price
  - Bid = mid - spread/2
  - Ask = mid + spread/2
- Mid price should change via random walk: add a random normal value each tick
- Use a seeded random number generator so results are reproducible

**Critical Requirement**:
- Add an assertion that bid < ask every single tick (this catches many bugs early)

**Learning Goal**: 
Learn state updates over time and how to produce clean "market data" for bots to consume.

**Test Requirements**:
- Verify bid < ask for at least 100 consecutive ticks
- Verify mid price actually changes over time (not static)
- Verify same seed produces same sequence of mid prices

---

### Task 4: Minimal order placement (marketable fills only)

**Required Functionality**:

The exchange must support:
- Placing a limit order (specify owner, side, price, quantity) and receive back an order ID
- Canceling an order by order ID
- Storing non-marketable orders somewhere for future processing

**Immediate Fill Rules** (you don't need a full order book yet):
- If a BUY order has price >= current ask, it fills immediately at the ask price
- If a SELL order has price <= current bid, it fills immediately at the bid price
- Non-marketable orders (price doesn't cross the spread) should be stored but not filled yet

**Output Requirement**:
- The exchange must produce Fill objects for all executed trades

**Learning Goal**: 
Learn order lifecycle basics and careful bookkeeping. Every fill must be tracked.

**Test Requirements**:
- Verify a marketable BUY order fills at ask price (not the order's price)
- Verify a marketable SELL order fills at bid price (not the order's price)
- Verify a non-marketable order stays in the system (doesn't disappear)
- Verify canceled orders produce no fills

---

### Task 5: Minimal bot state + accounting

**Required Functionality**:

A bot must track:
- Position (number of shares held, can be negative for short positions)
- Cash (cash balance)
- Open orders (which orders are currently active)

A bot must implement two callback methods:
- `on_snapshot(snapshot)`: Called each tick with current market data, bot decides what orders to place
- `on_fills(fills)`: Called when fills occur, bot must update its cash and position accordingly

**Accounting Rules**:
- Buying qty shares at price means: position increases by qty, cash decreases by price * qty
- Selling qty shares at price means: position decreases by qty, cash increases by price * qty

**PnL Calculation**:
- Mark-to-market PnL = cash + position * current_mid_price

**Learning Goal**: 
Build basic Python proficiency with arithmetic updates and state tracking. Getting fills right is critical.

**Test Requirements**:
- Verify a BUY fill correctly updates both position and cash
- Verify a SELL fill correctly updates both position and cash
- Verify PnL calculation matches expected value given known position, cash, and mid price
- Verify multiple fills compound correctly (position and cash track accurately over many trades)

---

## Stage 2: Testing Infrastructure (TDD Skills)
**Goal**: Build confidence to refactor without breaking everything

### Task 6: Add pytest + three "core correctness" tests

**Required Tests**:

1. **Seed reproducibility test**:
   - Run simulation twice with identical seed and configuration
   - Verify the mid-price path is identical across both runs
   - This proves your simulation is deterministic

2. **Marketable fill price test**:
   - Create an exchange with known bid/ask
   - Place a marketable order
   - Verify it fills at the correct price (bid or ask, not the order price)

3. **Bot accounting test**:
   - Create a bot with known starting state
   - Give it a fill
   - Verify position and cash update correctly

**Learning Goal**: 
Learn TDD discipline. These tests are your safety net for Friday-style rapid changes.

**Why Small and Deterministic**:
- Fixed seed = no random failures
- Small tick count = fast test execution  
- Simple scenarios = clear pass/fail criteria

---

## Stage 3: Real Order Lifecycle (Where You Level Up)
**Goal**: Handle the complexity of asynchronous fills and partial fills

### Task 7: Passive fills via external order flow

**Required Functionality**:

Each tick, "external flow" should arrive with some probability:
- With probability p_buy, an external BUY market order of random size appears
- With probability p_sell, an external SELL market order of random size appears
- External flow must fill against the best available price among all open bot orders

**Why This Matters**:
In real markets, you get filled when other participants hit your quotes. This is asynchronous - you don't control when it happens.

**Simplification for Now**:
You don't need a full order book yet. You can start by just tracking:
- The best (highest) bid price among all open bot orders
- The best (lowest) ask price among all open bot orders

**Learning Goal**: 
Learn to design systems where fills happen asynchronously (core algobot reality).

**Test Requirements**:
- Place a bot bid order
- Trigger external sell flow
- Verify the bot gets filled
- Verify no fill occurs if external flow is on the wrong side

---

### Task 8: Partial fills + order statuses

**Required Functionality**:

Orders can now fill partially:
- When an order fills, it may fill for less than its full quantity
- A partial fill should reduce the order's remaining quantity
- If quantity remains > 0, the order should stay open (possibly with status PARTIAL)
- If quantity becomes 0, the order should be marked FILLED and removed from open orders

**Order Status Lifecycle**:
- OPEN: Just placed, waiting for fill
- PARTIAL: Some quantity filled, some remains
- FILLED: Completely filled, no longer in open orders
- CANCELED: Canceled by bot before filling, no longer in open orders

**Critical Requirement**:
- Order quantity must NEVER go negative
- Fill quantity cannot exceed available order quantity

**Learning Goal**: 
Build robustness and edge-case handling. This is where most bots break.

**Test Requirements**:
- Verify partial fill leaves correct quantity remaining
- Verify partial fill updates order status appropriately
- Verify full fill removes order from open orders
- Verify order quantity never goes negative even with large external flow
- Verify multiple partial fills on the same order work correctly

---

### Task 9: Quote maintenance (cancel/replace instead of spamming)

**Required Functionality**:

Bots should maintain at most:
- One live bid quote
- One live ask quote

Each tick, a bot should:
- Check if existing quotes are still reasonable given current market conditions
- If quotes are stale (market moved significantly), cancel old quotes and place new ones
- If quotes are still good, keep them

**Why This Matters**:
- Prevents order spam (will be penalized in Friday scenarios)
- Makes state manageable (only track 2 orders instead of 100)
- Mirrors real market making behavior

**Learning Goal**: 
Learn execution hygiene and modular coding habits. Clean quote management prevents bugs.

**Test Requirements**:
- Verify bot never has more than 2 open orders at any time
- Verify bot cancels stale quotes when market moves
- Verify bot doesn't spam new orders every tick when quotes are still valid

---

### Task 10: Lifecycle tests

**Required Tests**:

1. **Cancel prevents fills**:
   - Place an order
   - Cancel it
   - Trigger external flow that would have matched the order
   - Verify no fill occurs

2. **Partial fill correctness**:
   - Place order for quantity 15
   - Trigger external flow that fills 7
   - Verify order quantity becomes 8 and status is PARTIAL
   - Trigger external flow that fills the remaining 8
   - Verify order is now FILLED and removed from open orders

3. **Open orders consistency**:
   - Run simulation for 1000 ticks with random external flow
   - After each tick, verify all open orders have quantity > 0
   - Verify no "zombie" orders (orders that should be filled/canceled but are still open)

4. **Bot state consistency over time**:
   - Run bot for 1000 ticks
   - Track position and cash changes from all fills
   - Verify bot's internal state matches your independent calculation

**Learning Goal**: 
Learn to test the tricky cases that will appear under Friday pressure.

---

## Stage 4: Logging + Replay (Debug Skills)
**Goal**: Debug fast via structured logs and reproducibility

### Task 11: Structured event logging

**Required Functionality**:

Implement two types of logging:

**Tick-level state** (logged every tick):
- Tick number
- Mid, bid, ask prices
- Bot position, cash, PnL

**Event-level actions** (logged when they occur):
- Order placements: which bot, side, price, quantity, timestamp
- Order cancellations: which order, timestamp
- Fills: which bot, side, price, quantity, timestamp

**Output Format**:
- Use CSV files (easy to inspect in Excel or text editor)
- Keep logs human-readable
- Save to disk so you can review after a run

**Learning Goal**: 
Learn fast debugging and post-mortem analysis. "What happened at tick 427?" must be answerable in 30 seconds.

---

### Task 12: Replay / reproducibility guarantee

**Required Functionality**:

Running the same simulation twice with identical seed and config must produce **byte-for-byte identical** log files.

**Verification Method**:
- Run simulation with seed X, save logs to file A
- Run simulation again with seed X, save logs to file B
- Use file comparison (diff, hash comparison, or line-by-line check)
- Verify A and B are identical

**Learning Goal**: 
Learn systematic debugging. Reproducibility is critical for debugging under pressure (Friday reality).

**Common Reproducibility Killers to Avoid**:
- Using unseeded randomness
- Depending on dict iteration order (use ordered collections if needed)
- System time dependencies
- Any external state that isn't controlled by your seed

**Test Requirements**:
- Write a test that runs simulation twice and verifies log file identity
- Test should fail if you introduce any non-determinism

---

## Stage 5: Modularity (Swap Strategies Quickly)
**Goal**: Separate "what to do" from "how to do it"

### Task 13: Create a Strategy interface

**Required Functionality**:

Separate decision-making from execution by creating two distinct components:

**Strategy Component** (decides WHAT to do):
- Takes current market snapshot and bot state as input
- Returns desired bid and ask prices (or None if no quote desired)
- Contains NO execution logic (no order placement, no tracking)
- Pure decision-making only

**Bot Engine Component** (decides HOW to execute):
- Manages position, cash, open orders
- Receives desired quotes from strategy
- Handles actual order placement and cancellation
- Tracks order IDs and manages quote lifecycle
- Updates state when fills occur

**Why This Separation Matters**:
- Swapping strategies = change one class, not rewrite entire bot
- Easy to A/B test different strategies
- Matches real trading systems architecture
- Critical for Friday speed (change strategy params in 10 min, not rewrite bot)

**Learning Goal**: 
Learn system design and clean separation of concerns.

**Test Requirements**:
- Verify you can swap strategy classes without modifying bot engine
- Verify strategy receives correct inputs (snapshot and state)
- Verify bot engine correctly executes strategy's desired quotes

---

### Task 14: Config-driven parameters

**Required Functionality**:

All strategy behavior must be controlled by configuration parameters, not hardcoded values:
- Spread size
- Order size
- Position limits
- Skew factors
- Defense thresholds
- Any other tunable parameters

**Requirements**:
- Store all params in a config file (or config section)
- Strategy should read params from config, never use hardcoded values
- Changing behavior should require editing config only, not code

**Why This Matters**:
Friday scenarios require rapid parameter changes. If you hardcoded spread=0.02 throughout your code, changing it to 0.05 requires finding and editing dozens of lines. With config, you edit one line.

**Learning Goal**: 
Learn experimentation workflow. Change behavior without touching code = Friday speed.

**Test Requirements**:
- Verify strategy behavior changes when config parameters change
- Verify same config produces same behavior (reproducibility with config)

---

## Stage 6: Risk + Inventory (Core Practical Behavior)
**Goal**: Don't blow up, adapt to position

### Task 15: Hard position limits

**Required Functionality**:

Implement position limit enforcement:
- Define a max_position parameter in config
- If bot position >= max_position, stop placing BUY orders (would increase position)
- If bot position <= -max_position, stop placing SELL orders (would decrease position)
- Optionally: reduce order sizes as you approach the limit

**Why This Matters**:
Friday constraints often include position limits. Your bot must respect them.

**Learning Goal**: 
Learn defensive coding and constraint handling.

**Test Requirements**:
- Run simulation where bot gets repeatedly filled on one side
- Verify position never exceeds max_position
- Verify bot stops quoting on the appropriate side when at limit
- Verify bot resumes quoting when position comes back within limit

---

### Task 16: Inventory skew

**Required Functionality**:

Bot should shift quotes based on current inventory:

**Concept**:
- When bot has large long position (many shares), it wants to sell → shift quotes down to encourage selling
- When bot has large short position, it wants to buy → shift quotes up to encourage buying
- When neutral, quotes are centered around fair value

**Behavior**:
- Calculate a "skew" based on current position (e.g., skew = position * skew_factor)
- Add skew to both bid and ask prices
- Result: quotes shift in the direction that helps you get back to neutral

**Learning Goal**: 
Learn state-based policies. Simple but powerful adaptation mechanism.

**Test Requirements**:
- Verify skew direction is correct: long position → lower quotes, short position → higher quotes
- Verify skew magnitude scales with position size
- Verify skew returns to zero when position is neutral
- Verify bot remains stable when holding large positions (doesn't oscillate)

---

### Task 17: Risk/inventory tests

**Required Tests**:

1. **Position limit enforcement**:
   - Set max_position to 50
   - Force bot to receive 100 consecutive BUY fills
   - Verify position never exceeds 50
   - Verify bot stops placing BUY orders when at limit

2. **Skew adapts correctly**:
   - Start bot at neutral position
   - Force bot to accumulate large long position
   - Verify quotes shift downward
   - Force bot to accumulate large short position  
   - Verify quotes shift upward

3. **Stability under one-sided flow**:
   - Force bot to get filled 50 times on BUY side
   - Verify bot doesn't crash
   - Verify position stays within limits
   - Verify bot still quotes on SELL side (trying to revert to neutral)

**Learning Goal**: 
Learn to test behavior over time, not just single outputs.

---

## Stage 7: Toxic Flow Detection + Defense
**Goal**: Detect when you're being picked off and defend yourself

**Context (from Leon's advice)**:
"Don't need to look at stats to make money off other bots. Toxic flows; understand the flows of the bots."

### Task 18: Flow signals (no heavy stats)

**Required Functionality**:

Implement simple rolling signals to detect toxic flow:

1. **Fill rate**: 
   - Track: How often are you getting filled over the last N ticks?
   - Compute: (number of ticks with fills) / (total ticks in window)
   - Toxic signal: Fill rate suddenly spikes above normal

2. **One-sided fill streak**: 
   - Track: Have you been repeatedly filled on the same side?
   - Detect: 5+ consecutive fills all on BUY or all on SELL
   - Toxic signal: Someone is consistently picking off one side

3. **Post-fill adverse move**: 
   - Track: After you get filled, does the mid price move against you?
   - Detect: Within K ticks of a fill, mid moves in the wrong direction
   - Example: You bought at 100, mid drops to 99.5 within 5 ticks
   - Toxic signal: You're buying tops and selling bottoms

**Implementation Requirement**:
- Use simple rolling windows (last N ticks or last N events)
- No complex statistics or machine learning needed
- Should be fast to compute (happens every tick)

**Learning Goal**: 
Learn to reason about adversarial environments using simple rolling counters, not complex statistics.

**Test Requirements**:
- Verify fill rate calculation is correct over rolling window
- Verify one-sided streak detection triggers after N consecutive same-side fills
- Verify adverse move detection works (buy → mid drops, sell → mid rises)
- Verify signals reset correctly after toxic flow stops

---

### Task 19: Toxic defense actions

**Required Functionality**:

When toxic flow is detected, take defensive actions:

**Defense Action 1: Widen spread**
- When fill rate exceeds threshold, multiply spread by some factor (e.g., 2x)
- Makes it more expensive for others to hit your quotes
- Reduces adverse selection

**Defense Action 2: Reduce size**
- When one-sided streak detected, reduce order quantity
- Limits exposure to continued toxic flow
- Caps losses from being picked off

**Defense Action 3: Cooldown**
- When severe adverse moves detected, stop quoting entirely for N ticks
- Gives time to reassess market conditions
- Prevents compounding losses

**Requirements**:
- Defensive logic should be in a separate module (not embedded in strategy)
- Easy to tune thresholds without rewriting logic
- Should automatically revert to normal when toxic flow stops

**Learning Goal**: 
Learn adaptation and feedback incorporation. Defense module is separate, so it's easy to tune.

**Test Requirements**:
- Verify spread widens when fill rate exceeds threshold
- Verify size reduces when one-sided streak detected
- Verify bot stops quoting during cooldown period
- Verify bot resumes normal behavior after cooldown expires

---

### Task 20: Toxic-flow regression tests

**Required Tests**:

Create deterministic scenarios that force toxic flow and verify defense triggers:

1. **High fill rate scenario**:
   - Force bot to get filled 10 times in 20 ticks (50% fill rate)
   - Verify spread widens appropriately
   - Verify spread returns to normal when fill rate drops

2. **One-sided streak scenario**:
   - Force bot to get filled 5 consecutive times on BUY side
   - Verify cooldown triggers or size reduces
   - Verify bot recovers after streak ends

3. **Adverse move scenario**:
   - Force bot to buy at price 100
   - Force mid price to drop to 99.5 within 5 ticks
   - Verify adverse move is detected
   - Verify appropriate defense action occurs

**Learning Goal**: 
Learn to test dynamic behavior over deterministic scenarios.

**Hint**: 
You can "stub" the exchange to force specific mid price movements for testing purposes.

---

## Stage 8: Opponent Bots + Exploitation Practice
**Goal**: Learn to exploit predictable bot patterns

**Context (from Leon's advice)**:
"Learning how to exploit those bots and what other people are doing."

### Task 21: Implement 3 opponent bot archetypes

**Required Functionality**:

Create three simple bots with intentionally exploitable weaknesses:

**Bot 1: Naive Market Maker**
- Behavior: Quotes fixed spread, no risk management, slow to update quotes
- Weakness: Quotes go stale after mid price moves
- Your exploitation opportunity: Hit stale quotes when mid jumps

**Bot 2: Simple Aggressor**
- Behavior: Takes quotes (crosses spread) based on simple momentum signal
- Weakness: Predictable entry timing and direction
- Your exploitation opportunity: Fade their trades (quote on same side they just hit)

**Bot 3: Event Sniper**
- Behavior: Only activates when event_flag appears, tries to hit stale quotes
- Weakness: Only active during events, predictable activation pattern
- Your exploitation opportunity: Cancel quotes quickly when events occur, re-quote after they're done

**Requirements**:
- Keep opponent bots simple (they're not trying to win, they're trying to be exploitable)
- Each should have a clear, predictable pattern
- Should be easy to identify their behavior from logs

**Learning Goal**: 
Learn adversarial thinking. Understanding opponent patterns = free PnL.

**Challenge**: 
Write a strategy that exploits all three opponent types.

---

### Task 22: Tournament harness (multi-seed evaluation)

**Required Functionality**:

Create a tournament system that:
- Runs multiple simulations across different random seeds (e.g., 20 seeds)
- Tracks performance metrics for each bot:
  - Average PnL
  - Worst PnL (robustness measure)
  - Best PnL
  - Standard deviation of PnL
  - Number of violations (position limit, order spam, etc.)
- Ranks bots by performance

**Critical Requirement**:
- All bots must see the same seeds (fair comparison)
- Seeds should be different from each other (test robustness across scenarios)

**Output Format**:
Produce a ranked table showing bot performance:
```
Rank | Bot Name          | Avg PnL | Worst | Best  | Violations
-----|-------------------|---------|-------|-------|------------
1    | YourBot           | +$125   | -$20  | +$380 | 0
2    | OpponentBot1      | +$89    | -$45  | +$290 | 0
3    | OpponentBot2      | -$15    | -$150 | +$80  | 3
```

**Learning Goal**: 
Learn proper evaluation and A/B testing discipline.

**Why Multiple Seeds**:
- Single seed can be lucky or unlucky
- Robust strategies perform consistently across seeds
- Identifies strategies that actually work vs. got lucky

---

### Task 23: "Explain Results" output

**Required Functionality**:

After each tournament run, automatically generate a summary report with:

**Section 1: Configuration**
- Strategy name
- Key parameters used
- Any changes from previous run

**Section 2: Results**
- Average PnL
- Worst PnL
- Best PnL  
- Number of violations

**Section 3: Three-bullet summary**
- What changed compared to previous run
- Why it helped or hurt (mechanism, not just "PnL went up")
- What to try next (hypothesis-driven)

**Example Output**:
```
Strategy: DefensiveMarketMaker
Parameters: spread=0.02, toxic_defense=enabled, threshold=0.5

Results:
  Avg PnL: $125.34
  Worst: -$20.15
  Best: $380.22

Summary:
  - Changed: Added toxic flow defense (spread widening on high fill rate)
  - Impact: Avg PnL improved $35 vs baseline, worst case improved 30%
  - Next: Test lower threshold (0.3) to trigger defense earlier
```

**Learning Goal**: 
Learn structured iteration and communication. Treat this like a trading desk debrief.

**Why This Matters**:
- Forces you to think about WHY results changed
- Builds hypothesis-driven experimentation muscle
- Critical for holistic evaluation (Future Focus cares about explanation, not just best PnL)

---

## Stage 9: Friday Drill Mode (Time Pressure)
**Goal**: Practice implementing changes in <60 minutes under pressure

### Task 24: Weekly timed drill constraints

**Required Practice**:

Complete each of these drills in **under 60 minutes**:

**Drill 1: Transaction Fee**
- Scenario: "All trades now have $0.01 fee per share"
- Requirements: Update fill accounting, adjust spread to remain profitable
- Success: Bot still makes money, tests pass, completed in <60 min

**Drill 2: Position Limit Change**
- Scenario: "Max position is now 50 (was 100)"
- Requirements: Update config, verify bot respects new limit
- Success: Position never exceeds 50, completed in <60 min

**Drill 3: Volatility Regime**
- Scenario: "When recent volatility > threshold, widen spread"
- Requirements: Detect high volatility (rolling std of mid changes), adjust spread
- Success: Bot widens during high vol, reverts during low vol, <60 min

**Drill 4: Rate Limiting**
- Scenario: "Max 10 order placements per tick"
- Requirements: Count orders per tick, enforce limit
- Success: Never exceeds 10 orders/tick, <60 min

**Drill 5: Event-Driven Defense**
- Scenario: "When event_flag=True, widen spread for next 5 ticks"
- Requirements: Detect flag, apply multiplier, track cooldown
- Success: Spread widens during events, reverts after, <60 min

**How to Practice**:
1. Set timer for 60 minutes
2. Read scenario (timer starts when you start reading)
3. Implement the change
4. Run tests to verify correctness
5. Success = all tests pass before 60 minutes

**Learning Goal**: 
Learn speed, calm debugging, and modular architecture under pressure.

**Why This Matters**:
Friday at Future Focus is exactly this - you get a constraint, 1 hour, and your PnL depends on implementing it correctly.

---

## Optional Advanced Tasks
*(Only attempt if you finish all core tasks early)*

### Task 25: Multi-asset support

**Required Functionality**:
- Support multiple markets (symbols) running simultaneously
- Separate positions, quotes, and state per symbol
- Handle cross-market scenarios (e.g., spreads between correlated assets)

**Learning Goal**: 
Learn scalable system design.

---

### Task 26: Basic visualization

**Required Functionality**:
- Plot PnL over time for a simulation run
- Overlay fills on the plot (mark buys and sells)
- Save plot to file

**Learning Goal**: 
Learn debugging via visualization.

---

## Development Workflow

**Before coding each task**:
- [ ] Read the learning goal - understand WHY you're building this
- [ ] Write tests first (TDD)
- [ ] Verify tests fail (confirms you're testing the right thing)

**After implementing each task**:
- [ ] Verify all tests pass
- [ ] Run with fixed seed and inspect logs (does output make sense?)
- [ ] Try to break it (edge cases, large N, extreme parameters)

**Weekly self-check**:
- [ ] Can I run simulation from scratch in <5 minutes?
- [ ] Can I explain what my bot does to a non-technical person?
- [ ] Can I reproduce any bug from logs in <2 minutes?
- [ ] Can I implement a constraint change in <60 minutes?

---

## Getting Started

**Step 1: Set up project**
```bash
mkdir algobot-tournament
cd algobot-tournament
mkdir sim tests
```

**Step 2: Install dependencies**
```bash
pip install pytest pandas numpy matplotlib
```

**Step 3: Start with Stage 1, Task 1**
- Create a simple loop that runs for N_TICKS
- Make it reproducible with a seed
- Write a test that verifies reproducibility

**Step 4: Don't skip tests**
- Tests are your Friday safety net
- Small deterministic tests catch bugs before they compound

---

## Success Criteria

**You're ready for Future Focus when:**

✅ **Modularity**: Can swap strategies in <10 minutes without breaking anything  
✅ **Speed**: Can implement 5 different constraints in <60 min each  
✅ **Understanding**: Can explain PnL differences between runs mechanistically  
✅ **Robustness**: Tests catch bugs before you notice them  
✅ **Exploitation**: Can spot and exploit opponent bot patterns from logs  
✅ **Calmness**: Stay systematic when things break under pressure

**You're NOT ready if:**

❌ Changing one thing breaks three other things  
❌ Can't reproduce bugs from logs  
❌ Don't know why PnL changed between runs  
❌ Panic when the timer starts  

---

## Remember

This isn't about perfect code. It's about code you can:
- Change quickly under pressure
- Debug systematically when it breaks
- Explain clearly to others

Start simple, test everything, and build confidence as you go.
