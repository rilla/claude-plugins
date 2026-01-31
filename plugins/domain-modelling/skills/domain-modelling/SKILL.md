---
name: domain-modelling
description: Use when starting new projects, features, or refactors - applies Domain-Driven Design to model problem space before writing code. Prevents jumping to implementation, establishes ubiquitous language, identifies entities/value objects/aggregates, and creates explicit domain model validated with stakeholders before coding begins.
---

# Domain Modelling

## Core Principle

**Model the domain BEFORE writing code.** Code expresses the model; the model is not discovered from code.

Domain modeling is about understanding and capturing the problem space—the business domain—in a structured, precise way that both domain experts and developers can work with. This understanding becomes the foundation that code is built upon, not something extracted from code after the fact.

## The Domain-First Workflow

**MANDATORY SEQUENCE** - Do not skip or reorder steps:

```
1. Understand Domain → 2. Build Model → 3. Validate Model → 4. Write Code
```

### Step 1: Understand the Domain

**Goal**: Extract essential domain concepts through pragmatic inquiry.

**Be pragmatic, not pedantic:**
- ✅ Infer obvious concepts (e.g., what an "order" or "user" is)
- ✅ Ask about non-obvious complexity (business rules, workflows, edge cases)
- ✅ Use AskUserQuestion tool for all clarifications
- ❌ Don't ask "What is an order?" for e-commerce
- ❌ Don't assume complex domain rules without asking

**Key Questions** (via AskUserQuestion with options):
- Business rules and policies ("When is payment captured?")
- Workflow and lifecycle ("What states can an order be in?")
- Variations and edge cases ("Are partial shipments allowed?")
- Constraints and invariants ("What makes a booking valid?")
- Domain events ("What happens when X occurs?")

**Example Good Question**:
```
"When is payment captured?"
Options:
- At order placement (pre-authorization)
- At shipment (charge when shipped)
- At delivery (pay on delivery)
- Hybrid (deposit + balance)
```

**Example Bad Question** (too obvious):
```
"What is an order?"  # For e-commerce, infer this
```

### Step 2: Build the Domain Model

**Goal**: Create explicit model showing domain structure.

**Apply DDD Building Blocks**:

| Building Block | When to Use | Example |
|----------------|-------------|---------|
| **Entity** | Has identity that persists across state changes | Order (same order, different states) |
| **Value Object** | Defined by attributes, no identity, immutable | Money(100, "USD"), Address |
| **Aggregate** | Cluster with consistency boundary | Order aggregate (Order + OrderLines) |
| **Service** | Operation that doesn't belong to entity/VO | PaymentProcessor, ShippingCalculator |
| **Repository** | Persistence abstraction for aggregates | OrderRepository |
| **Factory** | Complex object creation | OrderFactory for multi-step creation |
| **Domain Event** | Something significant that happened | OrderPlaced, PaymentCompleted |

**Model Output** (choose based on complexity):
- Simple: Bulleted list of entities, VOs, relationships
- Medium: Simple diagram (can be text-based)
- Complex: Written description with examples

**Example Model Output**:
```markdown
## E-Commerce Order Domain Model

### Aggregates
**Order** (aggregate root)
- Contains: OrderLines, ShippingAddress (VO)
- Identity: OrderNumber
- States: Draft → Placed → Paid → Shipped → Delivered
- Invariants: Must have ≥1 line, total > 0
- Events: OrderPlaced, OrderPaid, OrderShipped

### Entities
- OrderLine (within Order aggregate)
  - Identity: Position in order
  - References: Product (separate aggregate)

### Value Objects
- Money(amount, currency) - immutable
- Address(street, city, state, zip) - immutable
- OrderStatus - enum

### Domain Services
- PricingService - calculates totals with tax/discounts
- ShippingService - validates addresses, calculates shipping

### Bounded Contexts (if needed)
- Sales Context: Order, Customer
- Fulfillment Context: Shipment, Inventory
- Accounting Context: Invoice, Payment
```

**Ubiquitous Language**:
Document key terms and use them consistently:
- In model description
- In code (class names, method names)
- In conversations with domain experts

### Step 3: Validate the Model

**Present model for validation BEFORE coding.**

Use AskUserQuestion to confirm:
- "Does this match your understanding of [domain]?"
- "Are these the right entities and boundaries?"
- "Did I miss any critical concepts?"

Options: Yes (proceed) / Needs adjustments (iterate)

### Step 4: Write Code

**NOW and only now, write code that expresses the model.**

Code characteristics:
- ✅ Uses ubiquitous language in names
- ✅ Entities have identity and behavior
- ✅ Value objects are immutable
- ✅ Aggregates enforce invariants
- ✅ Domain logic in domain layer
- ❌ No technical concerns in domain objects
- ❌ No anemic model (getters/setters only)

## Context-Aware Deliverables

Adapt deliverables to situation:

| Situation | Appropriate Deliverables |
|-----------|--------------------------|
| New feature in existing domain | Entity/VO identification, domain events |
| New bounded context | Full model with context map |
| Refactoring | Revised model showing changes |
| Simple CRUD | Minimal (might not need full DDD) |
| Complex domain | Comprehensive model with examples |

**Don't force comprehensive DDD on simple CRUD apps**—but always model before coding.

## Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|----------------|-----|
| **Coding before modeling** | Code without understanding leads to rework | Follow workflow: understand → model → validate → code |
| **Asking obvious questions** | "What is an order?" for e-commerce | Infer common concepts, ask about complexity |
| **Not asking critical questions** | Assuming business rules | Use AskUserQuestion for rules, workflows, edge cases |
| **Technical questions as domain questions** | "What database?" before understanding domain | Domain questions first, technical decisions later |
| **Anemic domain model** | Entities with only getters/setters | Put behavior in domain objects |
| **Missing value objects** | Using primitives everywhere | Group related attributes, make immutable |
| **No aggregate boundaries** | Everything can modify everything | Define consistency boundaries |
| **Database schema = domain model** | Technical structure ≠ domain structure | Model domain first, then map to persistence |
| **Skipping validation** | Assumptions without confirmation | Present model, get feedback, iterate |

## Red Flags - STOP and Model First

If you're thinking any of these, you're about to skip domain modeling:

- "User wants it quickly, I'll just start coding"
- "This domain is simple, I know what to build"
- "I'll model as I code"
- "The database schema is the model"
- "I can refactor later if needed"
- "Asking questions will slow me down"
- "I already know this domain well"
- "This is refactoring, not a new project"
- "The model is in my head, I don't need to write it down"
- "Validation will happen in code review"
- "I'll document the model after coding"

**All of these mean: STOP. Do domain modeling first.**

## Rationalization Table

| Excuse | Reality |
|--------|---------|
| "User wants it quickly" | Skipping modeling causes more rework than modeling takes |
| "Domain is obvious" | Obvious surface ≠ obvious rules. Always model explicitly |
| "I'll model as I code" | You'll code what you can imagine, not what the domain needs |
| "It's just CRUD" | Even CRUD has domain rules. Model them. |
| "Database is the model" | Database = persistence structure. Domain = business structure |
| "I can infer everything" | Infer obvious concepts. Model non-obvious rules and workflows |
| "Questions slow me down" | Wrong assumptions slow you down more |
| "Too much process" | Process prevents problems. Time upfront < time fixing later |
| "I know this domain" | Your knowledge needs validation. Document and confirm. |
| "This is refactoring" | Refactoring without model = shooting blind. Model current + target state. |
| "Model is in my head" | Unwritten model can't be validated, shared, or maintained |
| "Code review validates" | Code review validates code. Model validation validates understanding. |
| "I'll document later" | Later never comes. Model first, code expresses it. |

## When NOT to Use Full DDD

**Simple CRUD with minimal business logic:**
- No complex workflows
- No invariants to enforce
- No domain events
- Straightforward entity relationships

**Even then**: Still understand domain, just don't over-engineer the model.

## Integration with Other Skills

- **Before**: Often use `brainstorming` to refine rough ideas
- **During**: Domain modeling is the FIRST step for projects/features/refactors
- **After**: Model informs code reviews (check if code matches model)

## Quick Reference

### Must Do
- [ ] Understand domain through pragmatic questions (AskUserQuestion)
- [ ] Identify entities (identity), value objects (no identity), services
- [ ] Define aggregate boundaries
- [ ] Establish ubiquitous language
- [ ] Create explicit model artifact
- [ ] Validate model with stakeholders
- [ ] THEN write code expressing model

### Must Not Do
- [ ] Jump to code without modeling
- [ ] Ask obvious questions (be pragmatic)
- [ ] Assume complex rules without asking
- [ ] Use technical concerns to drive domain model
- [ ] Create anemic domain model
- [ ] Skip validation step

---

**Remember**: The model is the understanding. Code is the expression. Get the understanding right first.
