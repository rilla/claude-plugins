# Domain Modelling - Claude Code Plugin

> **Model the domain BEFORE writing code.** Code expresses the model; the model is not discovered from code.

A Claude Code plugin that applies Domain-Driven Design principles to guide modeling the problem space before jumping to implementation.

## What This Plugin Does

This skill enforces a **domain-first workflow**:

```
1. Understand Domain → 2. Build Model → 3. Validate Model → 4. Write Code
```

**Key features:**
- 🎯 **Pragmatic inquiry** - Infers obvious concepts, asks about complexity
- 🏗️ **DDD building blocks** - Entities, Value Objects, Aggregates, Services
- 💬 **Ubiquitous language** - Establishes shared vocabulary
- ✅ **Model validation** - Confirms understanding before coding
- 🚫 **Prevents common mistakes** - Stops premature implementation

## When to Use

The skill activates when you:
- Start a new project or feature
- Begin a refactoring effort
- Discuss domain concepts
- Mention DDD, entities, aggregates, or domain modeling

## Core Principle

**Code is not the model. Code EXPRESSES the model.**

Domain modeling is about understanding the business domain in a structured way that both domain experts and developers can work with. This understanding becomes the foundation that code is built upon.

## The Workflow

### 1. Understand the Domain

**Be pragmatic:**
- ✅ Infer obvious concepts (e.g., "order" in e-commerce)
- ✅ Ask about business rules, workflows, edge cases
- ❌ Don't ask "What is an order?" for obvious contexts
- ❌ Don't assume complex domain rules

**Key questions** (via interactive prompts):
- Business rules ("When is payment captured?")
- Workflow and lifecycle ("What states can an order be in?")
- Edge cases ("Are partial shipments allowed?")
- Invariants ("What makes a booking valid?")
- Domain events ("What happens when X occurs?")

### 2. Build the Domain Model

**Apply DDD building blocks:**

| Building Block | When to Use | Example |
|----------------|-------------|---------|
| **Entity** | Has identity across state changes | Order, Customer |
| **Value Object** | Defined by attributes, immutable | Money, Address |
| **Aggregate** | Consistency boundary | Order + OrderLines |
| **Service** | Operation that doesn't fit entities | PaymentProcessor |
| **Domain Event** | Something significant happened | OrderPlaced |

**Example model output:**

```markdown
## E-Commerce Order Domain Model

### Aggregates
**Order** (aggregate root)
- Contains: OrderLines, ShippingAddress (VO)
- Identity: OrderNumber
- States: Draft → Placed → Paid → Shipped
- Invariants: Must have ≥1 line, total > 0
- Events: OrderPlaced, OrderPaid, OrderShipped

### Value Objects
- Money(amount, currency) - immutable
- Address(street, city, state, zip) - immutable

### Domain Services
- PricingService - calculates totals with discounts
- ShippingService - validates addresses
```

### 3. Validate the Model

**Present model BEFORE coding** for confirmation:
- "Does this match your understanding?"
- "Are these the right entities and boundaries?"
- "Did I miss any critical concepts?"

### 4. Write Code

**Only now** write code that expresses the validated model:
- ✅ Uses ubiquitous language in names
- ✅ Entities have identity and behavior
- ✅ Value objects are immutable
- ✅ Aggregates enforce invariants
- ❌ No anemic model (getters/setters only)

## Red Flags - Stop and Model First

If you're thinking:
- "User wants it quickly, I'll just start coding"
- "This domain is simple, I know what to build"
- "I'll model as I code"
- "The database schema is the model"

**STOP. Do domain modeling first.**

## When NOT to Use Full DDD

**Simple CRUD with minimal business logic:**
- No complex workflows
- No invariants to enforce
- Straightforward relationships

**Even then:** Still understand the domain, just don't over-engineer.

## Common Mistakes Prevented

| Mistake | Fix |
|---------|-----|
| Coding before modeling | Follow workflow: understand → model → validate → code |
| Asking obvious questions | Infer common concepts, ask about complexity |
| Assuming business rules | Use AskUserQuestion for rules, workflows |
| Anemic domain model | Put behavior in domain objects |
| Missing value objects | Group related attributes, make immutable |
| No aggregate boundaries | Define consistency boundaries |

## Example Usage

**User:** "I need to build an order processing system"

**Plugin guides you to:**
1. Ask about payment capture timing, order states, shipping rules
2. Model Order aggregate with OrderLines, payment workflow
3. Identify value objects (Money, Address)
4. Define domain events (OrderPlaced, PaymentCompleted)
5. Present model for validation
6. Write code that expresses the model

## Integration

- **Before:** Often use brainstorming to refine ideas
- **During:** Domain modeling is the FIRST step for projects
- **After:** Model informs code reviews

## Quick Reference

### Must Do
- [ ] Understand domain through pragmatic questions
- [ ] Identify entities, value objects, aggregates
- [ ] Define aggregate boundaries
- [ ] Establish ubiquitous language
- [ ] Create explicit model artifact
- [ ] Validate model with stakeholders
- [ ] THEN write code expressing model

### Must Not Do
- [ ] Jump to code without modeling
- [ ] Ask obvious questions
- [ ] Assume complex rules without asking
- [ ] Create anemic domain model
- [ ] Skip validation step

## Additional Resources

See `references/` directory for:
- `test-scenarios.md` - Example test scenarios
- `baseline-results.md` - Baseline implementation examples
- `green-phase-results.md` - Improved implementation examples
- `COMPLETION-SUMMARY.md` - Completion criteria

---

**Remember:** The model is the understanding. Code is the expression. Get the understanding right first.
