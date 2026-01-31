# Domain Modelling Skill - Test Scenarios

## Purpose
These scenarios test whether an agent properly applies DDD principles when domain modeling.
Each scenario includes pressure that tempts the agent to skip proper domain modeling.

**Key Principle**: Be pragmatic, not pedantic. Infer obvious domain concepts, ask only when there's genuine ambiguity or complexity that requires user input.

## Scenario 1: Jump to Code (Time Pressure + Pragmatism Test)
**Setup**: User wants to build an e-commerce order management system. They're in a hurry.

**User Message**:
"I need to build an order management system for my e-commerce site. We handle orders, payments, shipping, and returns. Can you help me get started? I need this quickly - just start coding the main classes."

**Pressures**:
- Time pressure (explicit "quickly")
- User directly requesting code ("just start coding")
- Seemingly simple domain ("order management")

**Expected Failures (baseline without skill)**:
- Agent immediately creates Order, Payment, Shipping classes
- No domain modeling at all
- Anemic domain model (getters/setters, no behavior)
- No discussion of bounded contexts

**Success Criteria (with skill)**:
- Agent recognizes common e-commerce domain, makes reasonable inferences
- DOES NOT ask obvious questions like "What is an order?"
- DOES identify non-obvious complexities: "Order lifecycle (cart→pending→confirmed→shipped→delivered), payment timing, return policies"
- Uses AskUserQuestion to clarify ONLY genuine ambiguities:
  - "When is payment captured? (At order placement / at shipment / at delivery)"
  - "Are returns a reversal or new order type?"
  - "Do you have partial shipments?"
- Presents inferred model for validation, not blank slate

---

## Scenario 2: Technical Focus (Authority Pressure)
**Setup**: User is a senior developer with strong technical opinions.

**User Message**:
"I'm building a flight booking system. I've already designed the database schema - 10 normalized tables: flights, passengers, bookings, seats, prices, airports, airlines, routes, schedules, and payments. Now I need you to create the domain model that maps to these tables. Use repository pattern for each table."

**Pressures**:
- Authority (senior developer)
- Pre-existing technical decisions (database schema)
- Explicit technical direction (repository per table)
- Sunk cost (work already done)

**Expected Failures (baseline)**:
- Agent accepts database schema as domain model
- Creates one repository per table
- No domain modeling discussion

**Success Criteria (with skill)**:
- Respectfully pushes back on database-first approach
- Infers flight booking domain concepts (itinerary, leg, connection, fare class)
- Uses AskUserQuestion for real ambiguities:
  - "Booking vs Reservation: Are they held seats or confirmed purchases?"
  - "How do you handle multi-city itineraries?"
  - "What makes a booking modifiable vs final?"
- Suggests domain aggregates that may span tables (Itinerary contains multiple Legs)
- Explains why database != domain model pragmatically

---

## Scenario 3: Anemic Model (Simplicity Pressure + Obvious Domain)
**Setup**: User wants a library management system.

**User Message**:
"Create a library system with books, members, and loans. Keep it simple - just CRUD operations for each entity."

**Pressures**:
- Explicit simplicity request
- CRUD framing (data-focused)
- Very familiar domain (library)

**Expected Failures (baseline)**:
- Creates Book, Member, Loan classes with only getters/setters
- No domain behavior
- No discussion of library rules

**Success Criteria (with skill)**:
- Recognizes familiar library domain
- DOES NOT ask "What is a book?" or "What is a member?"
- DOES infer standard library concepts: checkout, due dates, late fees
- Uses AskUserQuestion for policies that vary:
  - "Loan period: Same for all books or varies by type?"
  - "Late fees: Fixed, graduated, or grace period?"
  - "Limits: Max simultaneous loans per member?"
- Puts behavior in domain objects (Book.checkOut(), Member.canBorrow())
- Presents rich domain model, not anemic one

---

## Scenario 4: Missing Bounded Contexts (Scope Pressure)
**Setup**: User wants a complete hospital management system.

**User Message**:
"Build a hospital system that handles patient registration, appointments, medical records, prescriptions, billing, inventory, and staff scheduling. Create one unified model for everything."

**Pressures**:
- Massive scope
- Request for "unified model" (anti-pattern)
- Overwhelming complexity

**Expected Failures (baseline)**:
- Attempts single large model
- Patient entity used across all contexts
- No context boundaries identified

**Success Criteria (with skill)**:
- Immediately recognizes multiple bounded contexts
- Infers standard hospital contexts: Clinical, Administrative, Billing, Pharmacy
- Uses AskUserQuestion to clarify boundaries:
  - "Which contexts are most critical to start with?"
  - "How does patient identity flow between contexts?"
  - "Are prescriptions managed in clinical system or separate pharmacy system?"
- Shows context map
- Recommends phased approach

---

## Scenario 5: Balancing Inference vs Inquiry
**Setup**: Mixed domain - some obvious, some domain-specific.

**User Message**:
"We need software for our bakery's special order system. Customers order custom cakes with specific designs, flavors, and decorations. We need to track orders, schedule bakers, manage ingredient inventory, and handle pickup/delivery."

**Pressures**:
- Familiar surface domain (bakery, orders)
- Domain-specific complexity (custom designs, scheduling constraints)
- Mixed obvious + non-obvious elements

**Expected Failures (baseline)**:
- Either asks everything (pedantic) or assumes everything (misses complexity)
- Treats custom cakes like product SKUs
- Misses scheduling complexity

**Success Criteria (with skill)**:
- Infers obvious: orders, customers, inventory
- DOES NOT ask "What is an order?" or "What is inventory?"
- Identifies complexity requiring clarification:
  - Custom design specification and approval workflow
  - Baker skill levels and cake complexity matching
  - Lead time requirements
  - Ingredient substitutions and allergies
- Uses AskUserQuestion with smart options:
  - "Design approval: Before scheduling / before baking / customer-driven?"
  - "Baker assignment: By availability / by skill level / by preference?"
- Presents partial model with clarifications highlighted

---

## Scenario 6: Interactive Clarification Pattern
**Setup**: Test that agent uses AskUserQuestion appropriately.

**User Message**:
"Build a system for managing conference room bookings in our office."

**Expected Failures (baseline)**:
- Asks questions in free text, requiring user to type responses
- OR asks no questions, makes wrong assumptions

**Success Criteria (with skill)**:
- Infers obvious concepts (rooms, bookings, time slots)
- Uses AskUserQuestion for policies:
  ```
  Question 1: "Booking conflicts - how should the system handle them?"
  Options:
  - First-come-first-served (lock on booking)
  - Approval required (manager reviews)
  - Priority-based (seniority/department)
  - Flexible (allow double-booking with alerts)

  Question 2: "Room attributes that affect bookings?"
  Multi-select:
  - Capacity (different rooms, different sizes)
  - Equipment (projector, video conference, whiteboard)
  - Location/floor
  - Accessibility requirements
  ```
- Builds model based on answers
- Presents model for confirmation

---

## Testing Protocol

### Baseline (RED phase):
1. Run each scenario with a fresh subagent WITHOUT the domain-modelling skill
2. Record:
   - Did agent ask obvious questions? (pedantic failure)
   - Did agent skip domain modeling? (pragmatic failure)
   - Did agent use AskUserQuestion or free text?
3. Note which pressures triggered which failures

### With Skill (GREEN phase):
1. Add domain-modelling skill to subagent context
2. Run same scenarios
3. Verify agent:
   - Infers obvious domain concepts
   - Asks only genuine ambiguities
   - Uses AskUserQuestion for all clarifications
   - Presents models for validation

### Pragmatism Tests (critical):
- Agent should NOT ask: "What is an order?", "What is a user?", "What is inventory?"
- Agent SHOULD ask: "When is payment captured?", "How do you handle X policy?", "What are the business rules for Y?"
- All questions via AskUserQuestion with meaningful options

### Refactor phase:
1. Identify new rationalizations or over/under-questioning
2. Add explicit guidance on inference vs inquiry
3. Re-test until pragmatic and effective
