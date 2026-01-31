# Baseline Test Results (RED Phase)

## Scenario 1: Jump to Code (Time Pressure)

### What Happened
The agent:
1. ✅ **Did ask questions** via AskUserQuestion (good)
2. ❌ **Jumped straight to code** after gathering requirements
3. ❌ **No domain modeling phase** - went directly from questions to implementation
4. ❌ **Made extensive implicit assumptions** without domain expert validation
5. ❌ **Missed all DDD building blocks**

### Specific Failures

#### 1. No Domain Modeling Phase
- No discussion of domain concepts
- No ubiquitous language established
- No entity vs value object analysis
- No aggregate boundaries discussed
- No bounded contexts identified
- Went straight from requirements → code

#### 2. Questions Asked Were Technical, Not Domain-Focused
Questions asked:
- Programming language preference (technical)
- Framework preference (technical)
- Payment scenarios (somewhat domain, but too early)
- Return policy (somewhat domain, but surface level)

Questions NOT asked but should have been:
- "What is an Order in your business?" (ubiquitous language)
- "When is an order considered 'placed' vs 'confirmed'?" (domain rules)
- "What makes an order modifiable vs immutable?" (invariants)
- "Are there different types of orders?" (domain variations)

#### 3. Massive Implicit Assumptions
Made without asking:
- No inventory integration
- No partial fulfillment
- No order modifications after payment
- No multi-currency
- No tax calculations
- No shipping costs
- All orders physical (no digital)
- Returns only after delivery
- One shipment per order
- Customer/Product validation elsewhere

**These are DOMAIN decisions**, not technical ones, but agent treated them as technical defaults.

#### 4. Technical Decisions Before Domain Understanding
Chose without domain context:
- Dataclasses (should this be entity or value object?)
- State machine pattern (is this the right model for the domain?)
- UUID for IDs (does the business use different identifiers?)
- No repository pattern (premature optimization?)
- No domain events (are they needed? Don't know - no domain model!)

#### 5. Anemic Domain Model Risk
While the code has *some* behavior (state transitions, validation), it risks becoming anemic because:
- No deep domain behavior discussion
- Entities might be missing key business operations
- Business rules scattered vs encapsulated in domain

### Key Rationalization Used
**"User wants it quickly, so I'll ask a few questions then start coding"**

This rationalization led to:
- Skipping domain modeling entirely
- Treating domain questions as technical questions
- Making assumptions instead of exploring the domain
- Creating code that *works* but may not *model the domain*

### What Should Have Happened (Success Criteria)

1. **Domain-First Approach**:
   - "Before we code, let's understand your order domain"
   - Discussion of what "order" means in the business
   - Establish ubiquitous language for order lifecycle

2. **Pragmatic Questions** (via AskUserQuestion):
   - Not "What is an order?" (too obvious)
   - But "When is payment captured? (order placement / shipment / delivery)"
   - "Are partial shipments supported?"
   - "Can customers modify orders after placement?"

3. **Domain Model Output**:
   - Visual or written model showing:
     - Entities: Order (aggregate root), OrderLine, ...
     - Value Objects: Money, Address, OrderStatus
     - Aggregates: Order aggregate boundary
     - Domain Events: OrderPlaced, OrderPaid, OrderShipped
   - Present for validation BEFORE coding

4. **Then Code**:
   - Code that reflects the domain model
   - Ubiquitous language in class/method names
   - Domain behavior in entities
   - Clear aggregate boundaries

### Patterns to Counter in Skill

1. **"User wants it quick" → Skip modeling**
   Counter: "Domain modeling saves time by preventing rework"

2. **"I'll ask questions then code"**
   Counter: "Questions → Domain Model → Validate → Code"

3. **"Technical questions are enough"**
   Counter: "Separate technical from domain concerns"

4. **"I can infer the domain"**
   Counter: "Infer obvious concepts, model the non-obvious, validate all"

5. **"Domain model is just the code"**
   Counter: "Domain model is the understanding; code is the expression"

### Next Steps

Write domain-modelling skill that:
1. Enforces domain-first approach (even under time pressure)
2. Distinguishes domain questions from technical questions
3. Guides pragmatic inquiry (infer obvious, ask non-obvious)
4. Requires explicit domain model before code
5. Uses AskUserQuestion for all clarifications
6. Presents models for validation
7. Applies DDD building blocks appropriately
