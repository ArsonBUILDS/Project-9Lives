<p align="center">
  <img src="9livestp.png" width="260" />
</p>

# Project 9Lives

Project 9Lives is a self-funded execution system with exactly nine operational lives. The system sustains itself by recycling fees generated from on-chain activity back into its treasury. Each life represents a finite budget window. When a life's budget depletes, the system transitions to the next life if sufficient funds exist, or degrades and eventually halts if they do not. After all nine lives are exhausted, the system shuts down permanently. This is a deterministic, inspectable mechanism designed to demonstrate uptime resilience through fee recycling, not perpetual motion.

## What This Repo Contains

- Core mechanism specification for the 9Lives lifecycle model
- Pseudocode implementations of budget tracking and state transitions
- ASCII diagrams illustrating state machines and fee flows
- Parameter definitions for tuning system behavior
- Security notes on attack vectors and mitigation strategies
- Glossary of system-specific terminology
- FAQ addressing common questions about operation and limits
- Contribution guidelines for developers and auditors
- Reference documentation for transparency and auditability

## Core Idea

The system has nine lives. Each life is a funded operational window backed by a budget denominated in the system's native treasury asset. Activity within the system generates fees. These fees are routed back into the treasury to replenish the current life's budget or fund future lives.

When the current life's budget falls below a threshold, the system enters a degraded state. If incoming fees restore the budget above the threshold, the system continues in the current life. If the budget fully depletes and insufficient funds exist to start a new life, the system moves to the next life. If all nine lives are consumed and no further funding is available, the system halts permanently.

The number nine is fixed by design. There is no resurrection after the ninth life. The system's longevity depends entirely on sustained fee generation and treasury health.

## How Fees Keep It Alive

Fees are generated from activity within the system. Examples include transaction fees, execution costs, or protocol-level charges. The fee collection mechanism routes a configurable percentage of fees directly back into the treasury.

The treasury maintains a running balance. This balance funds the current life's operational budget. The budget decreases over time due to execution costs, decay, and withdrawals. Incoming fees offset this decrease.

If fees exceed costs, the budget grows and the system remains stable. If costs exceed fees, the budget shrinks. The system tracks this delta continuously and triggers state transitions based on budget thresholds.
```
┌─────────────┐
│   Activity  │
└──────┬──────┘
       │ generates
       ▼
   ┌───────┐
   │  Fees │
   └───┬───┘
       │ recycled
       ▼
┌──────────────┐
│   Treasury   │
└──────┬───────┘
       │ funds
       ▼
┌──────────────┐
│ Life Budget  │
└──────┬───────┘
       │ enables
       ▼
┌──────────────┐
│  Execution   │
└──────┬───────┘
       │ drives
       └────────> Activity (loop)
```

## Lifecycle Model

The system operates in one of four states at any time:

**Alive**: The current life's budget is above the minimum uptime threshold. Execution proceeds normally. Fees are collected and routed to the treasury. The system is fully operational.

**Degrading**: The budget has fallen below the minimum threshold but remains above zero. Execution may continue at reduced capacity or frequency. The system is vulnerable. If fees do not recover the budget soon, the life will end.

**Revived**: The budget has recovered from a degraded state back above the threshold within the same life. The system returns to the Alive state without consuming a new life. This state is transient and immediately transitions to Alive.

**Exhausted**: All nine lives have been consumed and the treasury cannot fund another life. The system halts all execution permanently. No further state transitions are possible.

Between lives, the system performs a life transition. The life counter increments. If lives remain and the treasury holds sufficient funds to meet the revive threshold, a new life begins with a fresh budget. If not, the system enters the Exhausted state.

### State Machine Diagram
```
         ┌───────────────────────────────────────┐
         │                                       │
         │    ┌───────┐   budget falls  ┌───────────┐
         │    │ Alive │ ───────────────> │ Degrading │
         │    └───┬───┘                  └─────┬─────┘
         │        │                            │
         │        │ budget recovers            │ life depleted,
         │        │                            │ lives remain,
         │        ▼                            │ treasury sufficient
         │    ┌────────┐                       │
         └────│Revived │◄──────────────────────┘
              └────────┘                       
                  │                            
                  │ transition to next life    
                  ▼                            
              ┌───────┐   all lives consumed   ┌───────────┐
              │ Alive │ ───────────────────────>│ Exhausted │
              └───────┘                         └───────────┘
                                                (terminal state)
```

## Determinism and Transparency

Every state transition is triggered by measurable on-chain data: budget levels, fee inflows, treasury balances, and elapsed time. The system does not rely on off-chain oracles or subjective inputs for core lifecycle logic.

All state changes are logged with timestamps, life counts, budget deltas, and fee amounts. These logs can be replayed to verify system behavior. Auditors can reconstruct the full history of lives consumed, budgets funded, and transitions executed.

The mechanism is deterministic given the same sequence of fee events and execution costs. This allows for simulation, testing, and formal verification of edge cases.

## System Architecture

The system is composed of five primary components:

**Fee Collector**: Intercepts fees from activity and calculates the recyclable portion based on the configured fee recycle rate. Routes funds to the treasury.

**Treasury**: Holds the system's balance. Tracks total funds, allocates budgets to lives, and enforces spending limits. The treasury is the single source of truth for financial state.

**Life Counter**: Maintains the current life index (1 through 9) and total lives consumed. Triggers life transitions when budgets deplete. Prevents transitions beyond the ninth life.

**Execution Engine**: Performs the system's operational tasks. Draws from the current life's budget for each execution cycle. Halts if the budget is insufficient or the system is exhausted.

**Logger**: Records all state transitions, fee collections, budget updates, and execution events. Provides an immutable audit trail for transparency.

Components communicate through well-defined interfaces. The Life Counter queries the Treasury for budget status. The Execution Engine requests budget allocations. The Fee Collector writes directly to the Treasury. The Logger observes all component interactions.
```
┌────────────────┐      ┌──────────────┐      ┌────────────────┐
│ Fee Collector  │─────>│   Treasury   │<─────│ Life Counter   │
└────────────────┘      └──────┬───────┘      └────────┬───────┘
                               │                       │
                               │ budget requests       │ life transitions
                               ▼                       ▼
                        ┌──────────────┐       ┌──────────────┐
                        │  Execution   │       │    Logger    │
                        │    Engine    │──────>│              │
                        └──────────────┘       └──────────────┘
                                               (audit trail)
```

## Parameters

The system is controlled by a set of tunable parameters. These values determine budget thresholds, fee routing rates, and lifecycle behavior.

- **LIVES_TOTAL**: Total number of lives available to the system. Fixed at 9 by design.

- **LIFE_BUDGET_TARGET**: Target budget allocated to each new life when it begins. Denominated in treasury asset units.

- **FEE_RECYCLE_RATE**: Percentage of collected fees routed back to the treasury. Range: 0.0 to 1.0. Example: 0.8 means 80% recycled.

- **MIN_UPTIME_BUDGET**: Minimum budget required for the system to remain in the Alive state. Falling below this triggers Degrading.

- **DECAY_RATE**: Rate at which the budget decreases per time unit due to operational costs, even without execution. Simulates passive drain.

- **REVIVE_THRESHOLD**: Minimum treasury balance required to start a new life after the current life depletes. Prevents starting a life with insufficient runway.

- **EXECUTION_INTERVAL**: Time between execution cycles. Shorter intervals increase responsiveness but drain the budget faster.

- **MAX_DRAWDOWN**: Maximum allowed budget decrease in a single interval before triggering safety checks or pauses.

- **SAFETY_PAUSE**: Boolean flag. If true, the system halts execution when anomalous budget behavior is detected (e.g., rapid depletion, fee manipulation).

## Reference Pseudocode

### update_budget_and_life()

This function updates the current life's budget based on fees and costs, and handles state transitions.
```python
def update_budget_and_life():
    # Collect fees from recent activity
    fees_collected = fee_collector.collect()
    recycled_fees = fees_collected * FEE_RECYCLE_RATE
    treasury.deposit(recycled_fees)
    
    # Deduct execution and decay costs
    costs = execution_engine.get_costs() + DECAY_RATE
    current_budget = treasury.get_life_budget(current_life)
    new_budget = current_budget + recycled_fees - costs
    
    # Update budget
    treasury.set_life_budget(current_life, new_budget)
    
    # Check state transitions
    if new_budget <= 0:
        if current_life < LIVES_TOTAL and treasury.balance >= REVIVE_THRESHOLD:
            # Transition to next life
            current_life += 1
            treasury.set_life_budget(current_life, LIFE_BUDGET_TARGET)
            logger.log("Life transition", current_life)
            state = "Alive"
        else:
            # No lives left or insufficient funds
            state = "Exhausted"
            logger.log("System exhausted")
            execution_engine.halt()
    elif new_budget < MIN_UPTIME_BUDGET:
        state = "Degrading"
        logger.log("Budget degrading", new_budget)
    else:
        if state == "Degrading":
            state = "Revived"
            logger.log("Budget revived", new_budget)
        state = "Alive"
    
    return state
```

### tick_execution()

This function executes one operational cycle and charges the budget.
```python
def tick_execution():
    if state == "Exhausted":
        return  # No execution allowed
    
    if state == "Degrading":
        # Optionally reduce execution frequency or capacity
        pass
    
    # Perform execution task
    task_cost = execution_engine.execute_task()
    
    # Deduct cost from current budget
    current_budget = treasury.get_life_budget(current_life)
    treasury.set_life_budget(current_life, current_budget - task_cost)
    
    # Log execution
    logger.log("Execution tick", task_cost, current_budget - task_cost)
    
    # Check if safety pause is triggered
    if SAFETY_PAUSE and (current_budget - task_cost) < 0:
        execution_engine.pause()
        logger.log("Safety pause triggered")
```

## Security and Safety Notes

**Fee Spoofing**: Malicious actors may attempt to artificially inflate fee generation to manipulate the system's budget. Mitigation: Validate fee sources cryptographically. Use on-chain transaction signatures to verify authenticity.

**Griefing Attacks**: Attackers may spam low-value activity to drain the budget faster than fees replenish it. Mitigation: Implement rate limiting on execution requests. Set minimum fee thresholds.

**Treasury Drain**: If the fee recycle rate is misconfigured or execution costs exceed fees for extended periods, the treasury will deplete prematurely. Mitigation: Monitor treasury health off-chain. Adjust parameters dynamically if governance permits.

**Wallet Security**: The treasury wallet is a single point of failure. Compromise of private keys results in total fund loss. Mitigation: Use multi-signature wallets, hardware security modules, or threshold cryptography.

**Reentrancy and State Consistency**: Concurrent calls to budget update functions may cause race conditions or double-spending. Mitigation: Use locking mechanisms or atomic state transitions. Ensure idempotency.

**Parameter Manipulation**: If parameters are mutable and controlled by a centralized admin, abuse is possible. Mitigation: Use time-locked governance or immutable parameters post-deployment.

**Pause Abuse**: The safety pause mechanism can be exploited to halt the system indefinitely. Mitigation: Require multi-signature or DAO approval to trigger pauses. Log all pause events.

**Budget Caps**: Enforce maximum budget sizes per life to prevent runaway growth from fee spikes. Cap total treasury balance to limit attack incentives.

## FAQ

**Q: Can the system run forever?**  
A: No. The system has exactly nine lives. Once all nine are consumed and the treasury cannot fund another life, the system halts permanently.

**Q: What happens if fees stop completely?**  
A: The budget will decay over time due to execution costs and passive drain. The system will degrade and eventually exhaust its current life. If no fees resume, subsequent lives will deplete faster until all nine are consumed.

**Q: Can I add more lives after deployment?**  
A: This depends on the implementation. If the LIVES_TOTAL parameter is immutable, no. If it is governed by a DAO or admin, it may be changed, but this would alter the core design principle.

**Q: How long does one life last?**  
A: Variable. It depends on fee inflow, execution costs, decay rate, and the initial budget. High fee activity extends life duration. High costs shorten it.

**Q: What triggers a life transition?**  
A: A life transition occurs when the current life's budget depletes to zero or below, and the treasury has sufficient funds (above REVIVE_THRESHOLD) to start a new life.

**Q: Can the system recover from a degraded state without consuming a life?**  
A: Yes. If fees increase enough to raise the budget back above MIN_UPTIME_BUDGET, the system revives within the same life.

**Q: What happens in the Exhausted state?**  
A: All execution halts. No new lives can begin. The system becomes inert. Any remaining treasury funds are locked unless a withdrawal mechanism is implemented.

**Q: Is the fee recycle rate adjustable?**  
A: This depends on governance. If the parameter is mutable, it can be adjusted. If immutable, it is fixed at deployment.

**Q: Who controls the treasury?**  
A: Implementation-dependent. Common patterns include smart contract custody, multi-sig wallets, or DAO-controlled treasuries.

**Q: Can I simulate the system's lifespan before deploying?**  
A: Yes. The mechanism is deterministic. Given a fee schedule and cost model, you can simulate budget evolution and predict life durations.

**Q: What prevents someone from draining the treasury directly?**  
A: Access control. The treasury should only accept writes from the Fee Collector and budget allocations from the Life Counter. Use role-based permissions and require signatures for withdrawals.

**Q: Does the system generate revenue?**  
A: No. Fees are recycled back into the system to sustain operations. There is no profit extraction unless explicitly designed with a fee split for external parties.

## Glossary

**Life**: A discrete operational window funded by a budget. The system has nine lives total.

**Budget**: The amount of funds allocated to the current life. Decreases due to costs and increases due to fees.

**Treasury**: The central balance pool from which life budgets are drawn and to which fees are deposited.

**Fee Recycle Rate**: The proportion of collected fees returned to the treasury. Expressed as a decimal between 0 and 1.

**Degrading State**: A vulnerable state where the budget has fallen below the minimum threshold but is not yet depleted.

**Revived State**: A transient state indicating recovery from degradation without consuming a new life.

**Exhausted State**: A terminal state where all lives are consumed and the system has halted.

**Life Transition**: The event where one life ends and another begins, incrementing the life counter.

**Decay Rate**: The passive rate at which the budget decreases per time unit, independent of execution.

**Execution Interval**: The time delay between consecutive execution cycles.

**Revive Threshold**: The minimum treasury balance required to fund a new life.

**Safety Pause**: A mechanism to halt execution when anomalous behavior is detected.

## Disclaimer

This project is experimental and provided for educational and research purposes. It is not financial advice. The code may contain bugs, vulnerabilities, or design flaws. Use at your own risk. The mechanism is designed to fail gracefully after nine lives. Do not deploy with funds you cannot afford to lose. Always audit code independently before production use. The authors assume no liability for losses incurred.

## Contributing

Contributions are welcome. Open issues for bugs, feature requests, or design discussions. Submit pull requests for code improvements, documentation fixes, or test coverage.

**Style Rules**:
- Use clear, descriptive variable names.
- Comment all non-obvious logic.
- Write unit tests for state transitions and budget calculations.
- Follow the existing code structure and naming conventions.

**Testing**:
- All PRs must include tests for new functionality.
- Run the full test suite before submitting.
- Include simulation tests for edge cases (e.g., budget depletion, fee spikes, concurrent transitions).

**Documentation**:
- Update this README if you change core behavior or add parameters.
- Maintain the ASCII diagrams if architecture changes.
- Keep the FAQ current with common questions from users and auditors.

## License

MIT License

Copyright (c) 2026 Project 9Lives Contributors

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.


THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
