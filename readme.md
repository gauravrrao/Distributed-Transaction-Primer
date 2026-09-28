# Distributed Transactions

A **distributed transaction** is a transaction that involves operations across **multiple independent services, databases, or resources**.

In a traditional monolithic application, a transaction usually happens inside a single database:

```text
Application
     |
     v
+-------------+
|  Database   |
+-------------+
     |
   BEGIN
     |
  Multiple
  Operations
     |
  COMMIT
```

The database can guarantee that all operations either succeed together or fail together.

In a distributed system, however, the operations may involve multiple databases or services:

```text
                 Distributed Transaction
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
     Order Service   Payment Service  Inventory
          |              |              |
       Order DB       Payment DB      Inventory DB
```

Now the transaction is no longer controlled by a single database.

For example, placing an order might require:

1. Creating an order.
2. Reserving inventory.
3. Processing payment.
4. Creating a shipment.

Each operation may belong to a different service and database.

The difficult question becomes:

> **How do we maintain consistency when some operations succeed and others fail?**

---

# 1. What Is a Distributed Transaction?

A distributed transaction is a logical transaction whose work spans multiple independent resources.

For example:

```text
Place Order

    |
    +----> Order Service
    |         |
    |       Order DB
    |
    +----> Inventory Service
    |         |
    |      Inventory DB
    |
    +----> Payment Service
              |
           Payment DB
```

The entire business operation is logically one transaction:

```text
Place Order
     |
     +--> Create Order
     |
     +--> Reserve Inventory
     |
     +--> Charge Payment
```

But technically, these are separate local transactions.

```text
Order DB        Inventory DB       Payment DB
   |                 |                 |
   | Local Tx        | Local Tx        | Local Tx
   v                 v                 v
 COMMIT            COMMIT            COMMIT
```

This creates the fundamental problem:

> A distributed transaction needs to maintain a consistent business outcome across multiple independently controlled transactions.

---

# 2. Why Do We Need Distributed Transactions?

Consider an e-commerce application.

A user purchases a laptop for ₹100,000.

The system performs:

```text
1. Create Order
2. Reserve Laptop
3. Charge ₹100,000
4. Create Shipment
```

Suppose:

```text
Order       -> SUCCESS
Inventory   -> SUCCESS
Payment     -> SUCCESS
Shipment    -> FAILURE
```

Now what should happen?

The system has already:

- created the order
- reduced/reserved inventory
- charged the customer

But shipment creation failed.

The system is now in an inconsistent business state.

This is the core problem distributed transactions attempt to solve.

---

# 3. Local Transaction vs Distributed Transaction

## Local Transaction

A local transaction operates within one transactional resource.

For example:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 1000
WHERE id = 1;

UPDATE accounts
SET balance = balance + 1000
WHERE id = 2;

COMMIT;
```

The database controls the entire transaction.

If something fails:

```text
BEGIN
   |
Operation A
   |
Operation B
   |
ERROR
   |
ROLLBACK
```

The database can rollback the changes.

---

## Distributed Transaction

Now imagine:

```text
Service A
   |
Database A

Service B
   |
Database B
```

The transaction becomes:

```text
Transaction
     |
     +---- Database A
     |
     +---- Database B
```

There is no single database automatically controlling both.

Therefore:

```text
Database A -> COMMIT
Database B -> FAILURE
```

can happen.

That is where distributed transaction management becomes necessary.

---

# 4. The Fundamental Problem

The fundamental problem can be represented as:

```text
             Global Transaction
                    |
        +-----------+-----------+
        |                       |
        v                       v
   Local Tx A              Local Tx B
        |                       |
      DB A                    DB B
```

Suppose:

```text
Local Tx A -> SUCCESS
Local Tx B -> FAILURE
```

The global transaction is now partially completed.

This creates a:

> **Partial failure**

Partial failure is one of the defining challenges of distributed systems.

---

# 5. Why Normal Database Transactions Are Not Enough

A database transaction generally provides:

```text
Atomicity
Consistency
Isolation
Durability
```

commonly known as **ACID**.

But ACID guarantees normally apply to a single transactional resource.

For example:

```text
PostgreSQL
   |
   +---- Order
   +---- Payment
   +---- Inventory
```

A single database can coordinate these operations.

But if we have:

```text
Order Service
    |
 PostgreSQL

Payment Service
    |
 PostgreSQL

Inventory Service
    |
 MongoDB
```

each resource has its own transaction boundary.

A PostgreSQL transaction cannot simply rollback a MongoDB transaction or another independent PostgreSQL instance.

---

# 6. Transaction Boundary

One of the most important concepts is the **transaction boundary**.

A local transaction might look like:

```text
BEGIN
 |
 +-- INSERT order
 |
 +-- UPDATE order status
 |
COMMIT
```

The transaction boundary is:

```text
BEGIN ---------------- COMMIT
```

In a distributed architecture:

```text
Order Service
    |
    +---- Local Transaction
    |
    v
 Order DB


Payment Service
    |
    +---- Local Transaction
    |
    v
Payment DB
```

There are multiple transaction boundaries.

The business operation, however, may cross all of them.

```text
Business Transaction
|
+---- Local Transaction A
|
+---- Local Transaction B
|
+---- Local Transaction C
```

This difference is extremely important.

---

# 7. Atomicity Problem

In a local transaction:

```text
A
+
B
+
C
```

either:

```text
A + B + C
```

happens or:

```text
NONE
```

happens.

In a distributed system:

```text
A -> SUCCESS
B -> SUCCESS
C -> FAILURE
```

is possible.

Therefore, we need a mechanism to coordinate the outcome.

---

# 8. Example: Money Transfer

Consider:

```text
Account A
   |
Bank DB 1

Account B
   |
Bank DB 2
```

Transfer:

```text
A -> B
₹10,000
```

The logical transaction is:

```text
Debit A
Credit B
```

Suppose:

```text
Debit A -> SUCCESS
Credit B -> FAILURE
```

Now:

```text
A = -₹10,000
B = unchanged
```

The money effectively disappeared from the perspective of the business operation.

A distributed transaction mechanism must prevent or compensate for such inconsistent outcomes.

---

# 9. Distributed Transaction Properties

A distributed transaction generally attempts to provide some form of:

## Atomicity

The distributed operation should reach a consistent outcome rather than leaving arbitrary partial updates.

```text
SUCCESS
   OR
ABORT / COMPENSATE
```

---

## Consistency

The system should preserve its defined business invariants.

For example:

```text
inventory >= 0
```

If 10 laptops exist:

```text
inventory = 10
```

After purchasing 2:

```text
inventory = 8
```

The distributed transaction should not produce:

```text
inventory = -5
```

because of inconsistent coordination.

---

## Isolation

Concurrent distributed operations should not incorrectly interfere with each other.

For example:

```text
User A -> reserves seat 10
User B -> reserves seat 10
```

The system should prevent both transactions from successfully claiming the same seat.

---

## Durability

Once a transaction is considered committed, the result should survive failures.

For example:

```text
Payment = SUCCESS
```

should not disappear because a service restarts.

---

# 10. Strong vs Eventual Consistency

Distributed systems often involve a trade-off between:

```text
Strong consistency
```

and:

```text
Eventual consistency
```

## Strong Consistency

After a successful transaction, all participating resources should reflect the transaction outcome.

Example:

```text
Order DB      -> SUCCESS
Payment DB    -> SUCCESS
Inventory DB  -> SUCCESS
```

The system attempts to maintain one globally consistent state.

---

## Eventual Consistency

Different services may temporarily have different states.

For example:

```text
Order Service
   |
Order = CONFIRMED

Inventory Service
   |
Inventory update = pending
```

After some time:

```text
Inventory Service
   |
Inventory = updated
```

The system becomes consistent eventually.

This approach can improve availability and reduce coordination overhead, but requires careful handling of failures and intermediate states.

---

# 11. The Two Main Problems

Distributed transactions have two major categories of problems.

## 1. Coordination

How do multiple participants agree on:

```text
COMMIT
```

or:

```text
ABORT
```

?

---

## 2. Failure Handling

What happens when:

```text
Service crashes
Network fails
Database becomes unavailable
Request times out
Message is lost
Response is lost
Participant commits but coordinator doesn't know
Coordinator crashes
```

?

These failures make distributed transactions significantly harder than local transactions.

---

# 12. Network Failure

A particularly important problem is that a timeout does not necessarily mean failure.

Suppose:

```text
Service A
    |
    | request
    v
Service B
```

Service B processes the request:

```text
Payment = SUCCESS
```

But the response is lost:

```text
Service B
    |
    X
    |
Service A
```

Service A sees:

```text
TIMEOUT
```

What does that mean?

It could mean:

```text
1. Request never reached B
2. B received request but hasn't processed it
3. B processed request but response was lost
4. B processed request and crashed before responding
```

Therefore:

> A timeout does not prove that a transaction failed.

This is one of the most important concepts in distributed systems.

---

# 13. Unknown Transaction State

Consider:

```text
Service A
   |
   | COMMIT
   v
Service B
```

Service B commits:

```text
COMMITTED
```

But the response is lost.

Service A sees:

```text
UNKNOWN
```

So Service A doesn't know whether B committed.

The possible states are:

```text
COMMITTED
ABORTED
UNKNOWN
```

The `UNKNOWN` state is particularly difficult.

---

# 14. Partial Failure

A distributed transaction can fail partially.

Example:

```text
Order      -> SUCCESS
Inventory  -> SUCCESS
Payment    -> FAILURE
```

This is fundamentally different from:

```text
Everything failed
```

because some state has already been persisted.

The system now needs to determine what to do with the successful operations.

---

# 15. Distributed Transaction Lifecycle

Conceptually, a distributed transaction can be represented as:

```text
             START
               |
               v
        +--------------+
        | Prepare work |
        +--------------+
               |
               v
        +--------------+
        | Coordinate    |
        +--------------+
               |
        +-------+-------+
        |               |
      SUCCESS         FAILURE
        |               |
        v               v
     COMMIT           ABORT/
                     COMPENSATE
```

The exact implementation depends on the distributed transaction mechanism.

---

# 16. Distributed Transaction Coordinator

A common architecture introduces a coordinator.

```text
                 Coordinator
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
      Service A   Service B   Service C
          |           |           |
         DB A        DB B        DB C
```

The coordinator manages the global transaction.

Its responsibilities may include:

- identifying participants
- coordinating transaction state
- determining commit/abort
- tracking participant responses
- handling failures
- maintaining transaction state

However, introducing a coordinator also introduces another distributed-system component that can fail.

---

# 17. Participants

A **participant** is a service/resource involved in the distributed transaction.

Example:

```text
Transaction T1

Coordinator
     |
     +---- Order DB
     |
     +---- Inventory DB
     |
     +---- Payment DB
```

Here:

```text
Order DB       = Participant
Inventory DB   = Participant
Payment DB     = Participant
```

Each participant manages its own local transaction.

---

# 18. Global Transaction vs Local Transaction

This distinction is critical.

### Global Transaction

```text
PlaceOrder(T1)
```

The business-level operation.

### Local Transactions

```text
T1-A -> Order DB
T1-B -> Inventory DB
T1-C -> Payment DB
```

Therefore:

```text
Global Transaction T1
        |
        +---- Local T1-A
        |
        +---- Local T1-B
        |
        +---- Local T1-C
```

The distributed transaction mechanism connects these local transactions into one logical operation.

---

# 19. Transaction States

A distributed transaction may move through several states.

A simplified model:

```text
NEW
 |
 v
ACTIVE
 |
 v
PREPARING
 |
 +------+
 |      |
 v      v
READY  FAILED
 |       |
 v       v
COMMIT  ABORT
 |
 v
COMPLETED
```

Actual states depend on the transaction protocol.

The important idea is that distributed transactions require explicit state management.

---

# 20. Idempotency

Idempotency is extremely important in distributed transactions.

Suppose:

```text
Payment request
```

is sent twice because the client didn't receive the first response.

Without idempotency:

```text
₹10,000
+
₹10,000
=
₹20,000 charged
```

Instead, the system should associate a unique idempotency key:

```text
transaction_id = TXN123
```

If the same request arrives again:

```text
TXN123
```

the service recognizes that the operation has already been processed.

```text
Request 1
   |
TXN123
   |
PROCESS

Request 2
   |
TXN123
   |
ALREADY PROCESSED
```

This prevents duplicate side effects.

---

# 21. Transaction ID

A distributed transaction usually needs a globally identifiable transaction ID.

Example:

```text
transaction_id = TXN_123456
```

This ID can be propagated across services.

```text
Client
  |
  | TXN123
  v
Order Service
  |
  | TXN123
  v
Inventory Service
  |
  | TXN123
  v
Payment Service
```

This helps with:

- correlation
- tracing
- idempotency
- debugging
- transaction state tracking
- recovery

---

# 22. Correlation ID vs Transaction ID

These concepts are related but not necessarily identical.

## Correlation ID

Used to trace a request across services.

```text
request_id = REQ123
```

Useful for:

```text
logging
tracing
debugging
```

---

## Transaction ID

Identifies a logical business transaction.

```text
transaction_id = TXN123
```

Useful for:

```text
transaction coordination
idempotency
transaction state
recovery
```

One request may be part of a larger transaction.

Therefore:

```text
Correlation ID != Transaction ID
```

although some systems may choose to use the same identifier.

---

# 23. Distributed Transaction and Database Locks

Locks can become much more complicated in distributed transactions.

Consider:

```text
Transaction T1
    |
Lock DB-A
    |
Lock DB-B
```

If T1 keeps locks while waiting for another participant:

```text
DB-A
 |
Locked
 |
waiting
 |
DB-B
```

the locks may remain held for a long time.

This can cause:

```text
Lock contention
     |
     v
Reduced throughput
     |
     v
Timeouts
     |
     v
More retries
```

Distributed transactions therefore need careful consideration of locking and transaction duration.

---

# 24. Distributed Deadlocks

Distributed transactions can also create deadlocks.

Example:

```text
Transaction T1
    |
    +---- locks DB-A
    |
    +---- waits for DB-B


Transaction T2
    |
    +---- locks DB-B
    |
    +---- waits for DB-A
```

Result:

```text
T1 -> waiting for T2
T2 -> waiting for T1
```

Neither can proceed.

This is a distributed deadlock.

---

# 25. Long-Running Distributed Transactions

A transaction becomes particularly difficult when it takes a long time.

Example:

```text
Create Order
     |
Reserve Inventory
     |
Payment
     |
Fraud Check
     |
Shipping
     |
Confirmation
```

If this entire workflow is kept as one tightly coordinated transaction, it could hold resources for a long period.

Long transactions can cause:

- lock contention
- reduced throughput
- increased failure probability
- increased recovery complexity
- resource exhaustion

Therefore, distributed transaction design needs to consider transaction duration carefully.

---

# 26. Why Distributed Transactions Are Expensive

Compared with a local transaction:

```text
BEGIN
UPDATE
UPDATE
COMMIT
```

a distributed transaction may require:

```text
Network calls
      +
Coordination
      +
State tracking
      +
Retries
      +
Timeout handling
      +
Failure recovery
      +
Logging
      +
Potential locking
```

Therefore:

> Distributed transactions introduce significant coordination overhead.

---

# 27. Two-Phase Commit

One classical approach to distributed transactions is **Two-Phase Commit (2PC)**.

It divides the transaction into two phases:

```text
Phase 1 -> Prepare
Phase 2 -> Commit
```

The architecture looks like:

```text
              Coordinator
              /    |    \
             /     |     \
            v      v      v
           DB-A   DB-B   DB-C
```

The coordinator manages the transaction.

---

# 28. Phase 1 — Prepare

The coordinator asks each participant:

```text
"Can you commit this transaction?"
```

For example:

```text
Coordinator
     |
     +----> DB-A: PREPARE
     |
     +----> DB-B: PREPARE
     |
     +----> DB-C: PREPARE
```

Each participant performs the required work locally and determines whether it can commit.

Possible response:

```text
YES
```

or:

```text
NO
```

Conceptually:

```text
DB-A -> YES
DB-B -> YES
DB-C -> YES
```

If every participant says yes, the coordinator can proceed toward commit.

---

# 29. Phase 2 — Commit

If all participants are prepared:

```text
Coordinator
     |
     +----> DB-A: COMMIT
     +----> DB-B: COMMIT
     +----> DB-C: COMMIT
```

Each participant commits its local transaction.

Result:

```text
DB-A -> COMMITTED
DB-B -> COMMITTED
DB-C -> COMMITTED
```

The distributed transaction succeeds.

---

# 30. 2PC Failure Scenario

Suppose:

```text
DB-A -> YES
DB-B -> YES
DB-C -> NO
```

The coordinator cannot commit the global transaction.

It sends:

```text
ABORT
```

to the participants.

```text
Coordinator
     |
     +----> DB-A: ABORT
     +----> DB-B: ABORT
     +----> DB-C: ABORT
```

The transaction is rolled back/aborted according to the protocol.

---

# 31. 2PC's Major Problem

Two-phase commit can block.

Suppose:

```text
Coordinator
     |
     +---- DB-A
     +---- DB-B
     +---- DB-C
```

All participants have prepared.

Then:

```text
Coordinator crashes
```

Participants may be left waiting for the final decision.

Conceptually:

```text
Participant
    |
 PREPARED
    |
    | waiting for decision
    |
    X
Coordinator unavailable
```

This can cause resources to remain locked or reserved.

Therefore:

> 2PC provides strong coordination but can suffer from blocking and significant availability/latency costs.

---

# 32. Distributed Transaction Timeout

Timeouts are often used to prevent a transaction from waiting indefinitely.

Example:

```text
Transaction starts
      |
      v
Wait for participant
      |
      |
      v
   Timeout
      |
      v
Recovery / Abort
```

But timeout handling is dangerous because:

```text
Timeout != definite failure
```

The participant might have successfully committed while the coordinator timed out.

Therefore, recovery logic must account for uncertain transaction state.

---

# 33. Retries

Distributed systems frequently retry failed requests.

Example:

```text
Request
   |
   v
Payment Service
   |
 timeout
   |
   v
Retry
```

But retries can create duplicate operations.

Bad:

```text
Charge ₹10,000
Charge ₹10,000
```

Good:

```text
transaction_id = TXN123

Retry TXN123
      |
      v
Already processed
```

Therefore:

> Retries and idempotency must be designed together.

---

# 34. Exactly-Once Processing

People often describe distributed systems using the phrase:

```text
Exactly once
```

But achieving true exactly-once side effects across independent systems is difficult.

For example:

```text
Send payment request
      |
Payment succeeds
      |
Response lost
      |
Retry
```

The retry must not create another payment.

In practice, systems often combine:

```text
Unique transaction IDs
+
Idempotency
+
Durable state
+
Retries
+
Deduplication
```

to achieve effectively-once business behavior.

---

# 35. Compensation

Sometimes a distributed system cannot literally rollback a previously committed operation.

Example:

```text
Payment -> SUCCESS
```

You cannot simply ask the external payment provider:

```text
ROLLBACK DATABASE TRANSACTION
```

Instead, you may perform another business operation:

```text
Payment -> SUCCESS

Later failure

      |

Refund Payment
```

This is called a **compensating action**.

The important distinction is:

```text
Rollback
```

versus:

```text
Compensation
```

Rollback reverses a transaction at the transactional-resource level.

Compensation performs another business operation to counteract a previous successful operation.

---

# 36. Rollback vs Compensation

### Rollback

```text
UPDATE
   |
ERROR
   |
ROLLBACK
```

The original transaction is undone.

---

### Compensation

```text
Payment
   |
SUCCESS
   |
Later failure
   |
REFUND
```

The original payment remains a historical event.

A new operation compensates for it.

This distinction becomes extremely important in microservices.

---

# 37. Business State vs Database State

Distributed transactions are not only about database consistency.

Consider:

```text
Payment Provider
Shipping Provider
Email Provider
Inventory Service
```

Some operations cannot be rolled back technically.

For example:

```text
Email sent
```

You cannot unsend an email using a database rollback.

Therefore, distributed transaction design must consider **business semantics**, not just database state.

---

# 38. Distributed Transactions Across External Systems

Suppose:

```text
Order Service
     |
Payment Gateway
     |
Shipping Provider
```

You may not control:

```text
Payment Gateway DB
Shipping Provider DB
```

Therefore, traditional distributed transaction protocols may not be practical.

Instead, the system may need:

```text
Idempotency
+
Retries
+
Status tracking
+
Compensation
+
Reconciliation
```

The exact strategy depends on the business requirements.

---

# 39. Reconciliation

A reconciliation process compares states across systems.

Example:

```text
Internal Payment DB

TXN123
status = PENDING
```

External payment provider:

```text
TXN123
status = SUCCESS
```

A reconciliation process can detect the mismatch:

```text
Internal: PENDING
External: SUCCESS
        |
        v
Reconcile
        |
        v
Internal: SUCCESS
```

Reconciliation is particularly useful when failures produce uncertain states.

---

# 40. Transaction Log

Distributed transaction systems commonly need durable transaction state.

Example:

```text
Transaction Log

TXN123
-----------------------
Order       = PREPARED
Inventory   = PREPARED
Payment     = COMMITTED
GlobalState = UNKNOWN
```

The log can help recovery mechanisms determine what happened before a crash.

Without durable state, recovering from coordinator or participant failures becomes much harder.

---

# 41. Crash Recovery

Consider:

```text
Coordinator
     |
     +---- Participant A
     +---- Participant B
     +---- Participant C
```

The coordinator crashes.

After restarting, it needs to determine:

```text
What transactions were active?
Which participants prepared?
Which transactions committed?
Which transactions aborted?
```

Therefore, transaction state often needs to be persisted.

A simplified model:

```text
Runtime Memory
      |
      v
Transaction State
      |
      v
Durable Transaction Log
```

---

# 42. Distributed Transaction and Messaging

Distributed transactions often interact with message systems.

For example:

```text
Order Service
     |
     v
Message Broker
     |
     +---- Inventory Service
     |
     +---- Payment Service
```

This introduces another consistency problem:

```text
Database update
        +
Message publication
```

What happens if:

```text
DB COMMIT
    |
    X
Message publish fails
```

or:

```text
Message published
    |
    X
DB COMMIT fails
```

These are examples of the **dual-write problem**.

---

# 43. The Dual-Write Problem

Suppose an Order Service needs to:

```text
1. Save order to DB
2. Publish OrderCreated event
```

Naively:

```text
DB
 |
COMMIT
 |
X
Message Broker
 |
FAILED
```

The database says:

```text
Order = CREATED
```

but consumers never receive the event.

The opposite can also happen:

```text
Message published
 |
DB transaction fails
```

Now consumers believe an order exists when it doesn't.

This is one of the major consistency challenges in distributed systems.

---

# 44. Distributed Transaction Doesn't Mean "Everything Must Be One Database Transaction"

A common misconception is:

> Every operation across services should behave exactly like one giant SQL transaction.

That is usually impractical.

Instead, distributed transaction design asks:

```text
What consistency guarantee does the business actually require?
```

For some operations:

```text
Strong atomicity
```

may be necessary.

For others:

```text
Eventual consistency
```

may be completely acceptable.

---

# 45. Example: Order Processing

Consider:

```text
Order
Payment
Inventory
Email
Analytics
```

Not every operation needs the same consistency requirement.

Potentially:

```text
Order + Payment
     |
High consistency requirement

Inventory
     |
High consistency requirement

Email
     |
Can often be asynchronous

Analytics
     |
Can often be eventually consistent
```

This is an important architectural principle:

> Do not impose the strongest consistency mechanism on every operation unnecessarily.

---

# 46. Distributed Transaction Boundaries Should Follow Business Boundaries

Suppose an order workflow is:

```text
Create Order
Reserve Inventory
Charge Payment
Send Email
Update Analytics
```

Trying to treat all five as one tightly coupled transaction can create unnecessary complexity.

Instead, identify which operations actually need atomic business guarantees.

For example:

```text
Core transaction:

Order
+
Inventory
+
Payment
```

while:

```text
Email
Analytics
Notifications
```

may be processed asynchronously.

The correct boundary depends on the business requirements.

---

# 47. Common Distributed Transaction Failure Scenarios

A robust system must consider scenarios such as:

### Scenario 1

```text
Service A succeeds
Service B fails
```

### Scenario 2

```text
Service A succeeds
Service B succeeds
Service C times out
```

### Scenario 3

```text
Participant commits
Response is lost
```

### Scenario 4

```text
Coordinator crashes
```

### Scenario 5

```text
Participant crashes after prepare
```

### Scenario 6

```text
Network partition
```

### Scenario 7

```text
Retry causes duplicate operation
```

### Scenario 8

```text
Database succeeds
Event publication fails
```

### Scenario 9

```text
Event succeeds
Database transaction fails
```

These failure modes should be explicitly considered during system design.

---

# 48. Distributed Transaction Design Checklist

When designing a distributed transaction, ask:

### Transaction

- What is the business transaction?
- What operations belong to it?
- What operations can be eventually consistent?

### Participants

- Which services participate?
- Which databases participate?
- Are external systems involved?

### Failure

- What if one participant fails?
- What if the network fails?
- What if a response is lost?
- What if the coordinator crashes?

### Idempotency

- Can requests be retried safely?
- What is the idempotency key?
- How are duplicate requests detected?

### Recovery

- How is transaction state persisted?
- How are incomplete transactions recovered?
- Is reconciliation required?

### Consistency

- Is strong consistency required?
- Is eventual consistency acceptable?
- Which business invariants must always hold?

### Performance

- How long does the transaction remain active?
- Are resources locked?
- What happens under high concurrency?

---

# 49. Simplified Mental Model

The easiest way to understand distributed transactions is:

```text
One Business Operation
        |
        v
Multiple Local Transactions
        |
        v
Need Coordination
        |
        +----------------+
        |                |
        v                v
     Success           Failure
        |                |
        v                v
     Commit          Abort /
                    Compensation
```

And the major challenges are:

```text
Distributed Transaction
        |
        +--> Coordination
        |
        +--> Partial Failure
        |
        +--> Network Failure
        |
        +--> Unknown State
        |
        +--> Idempotency
        |
        +--> Recovery
        |
        +--> Concurrency
        |
        +--> Deadlocks
        |
        +--> Dual Writes
        |
        +--> External Systems
```

---

# 50. Key Takeaways

### 1. A distributed transaction spans multiple independent resources.

```text
Service A -> DB A
Service B -> DB B
Service C -> DB C
```

---

### 2. Local ACID transactions do not automatically provide global atomicity.

Each database controls its own transaction.

---

### 3. Partial failure is the central problem.

```text
A -> SUCCESS
B -> FAILURE
```

must be handled explicitly.

---

### 4. Network timeout does not necessarily mean failure.

A request may have succeeded even when its response was lost.

---

### 5. Idempotency is essential.

Retries must not create duplicate business side effects.

---

### 6. Transaction IDs help coordinate and trace distributed work.

```text
TXN123
```

can travel across participating services.

---

### 7. Rollback and compensation are different.

```text
Rollback      -> undo local transaction
Compensation  -> perform another business operation
```

---

### 8. Strong consistency has a cost.

Coordination can introduce:

```text
latency
blocking
lock contention
reduced availability
operational complexity
```

---

### 9. Not every operation requires distributed atomicity.

Some operations can safely use:

```text
eventual consistency
```

---

### 10. Failure recovery is part of transaction design.

A distributed transaction is not complete architecturally until you have considered:

```text
success
failure
timeout
retry
crash
network partition
duplicate request
unknown state
recovery
```

---

# Distributed Transaction — One-Line Definition

> **A distributed transaction is a logical business transaction whose operations span multiple independent transactional resources and therefore requires coordination, failure handling, and consistency mechanisms to achieve the required business guarantees.**

---

# Saga Pattern — Choreography

When a distributed transaction spans multiple microservices, maintaining one traditional database transaction across all services can be difficult.

The **Saga Pattern** solves this by breaking one large distributed transaction into a sequence of **local transactions**.

Each service:

1. Performs its own local transaction.
2. Publishes an event after the local transaction succeeds.
3. Another service consumes that event.
4. Performs its own local transaction.
5. Publishes the next event.

In **Saga Choreography**, there is no central component coordinating every step.

Instead:

> **Services communicate with each other through events, and each service reacts to events and decides what action it needs to perform next.**

---

# 1. What Is Saga?

Suppose we have an e-commerce system:

```text
Order Service
Inventory Service
Payment Service
Shipping Service
```

Placing an order requires:

```text
Create Order
    ↓
Reserve Inventory
    ↓
Process Payment
    ↓
Create Shipment
```

Instead of trying to make these operations one giant database transaction:

```text
BEGIN
   |
Order
   |
Inventory
   |
Payment
   |
Shipping
   |
COMMIT
```

we divide the operation into local transactions:

```text
Order Local Transaction
        ↓
Inventory Local Transaction
        ↓
Payment Local Transaction
        ↓
Shipping Local Transaction
```

Each local transaction commits independently.

The complete business operation is called a **Saga**.

---

# 2. Saga as a Sequence of Local Transactions

A Saga can be represented as:

```text
Saga T1

T1 → T2 → T3 → T4
```

For example:

```text
T1 = Create Order
T2 = Reserve Inventory
T3 = Process Payment
T4 = Create Shipment
```

Each transaction belongs to a different service.

```text
T1
 |
Order Service
 |
Order DB
```

```text
T2
 |
Inventory Service
 |
Inventory DB
```

```text
T3
 |
Payment Service
 |
Payment DB
```

```text
T4
 |
Shipping Service
 |
Shipping DB
```

There is no single database transaction spanning all of them.

---

# 3. What Is Saga Choreography?

In **choreography**, each service listens for events and decides what to do based on those events.

For example:

```text
Order Service
     |
     | OrderCreated
     v
 Message Broker
     |
     v
Inventory Service
     |
     | InventoryReserved
     v
 Message Broker
     |
     v
Payment Service
     |
     | PaymentCompleted
     v
 Message Broker
     |
     v
Shipping Service
```

The services effectively coordinate themselves through events.

There is no central component saying:

```text
Do this
Now do this
Now do this
```

Instead:

```text
Event
  ↓
Service reacts
  ↓
Local transaction
  ↓
New event
  ↓
Next service reacts
```

---

# 4. Basic Choreography Flow

Consider:

```text
User places order
```

The flow might be:

```text
Order Service
     |
     | Create Order
     v
Order DB
     |
     | OrderCreated
     v
Message Broker
     |
     v
Inventory Service
     |
     | Reserve Inventory
     v
Inventory DB
     |
     | InventoryReserved
     v
Message Broker
     |
     v
Payment Service
     |
     | Charge Customer
     v
Payment DB
     |
     | PaymentCompleted
     v
Message Broker
     |
     v
Shipping Service
```

Each service is responsible for its own step.

---

# 5. Why Is It Called Choreography?

Think about a dance.

There isn't necessarily one person standing in the middle telling every dancer exactly what to do.

Instead:

```text
Dancer A
   ↓
performs action

Dancer B sees/reacts
   ↓
performs action

Dancer C sees/reacts
   ↓
performs action
```

Similarly, in Saga Choreography:

```text
Service A
   ↓ event
Service B
   ↓ event
Service C
   ↓ event
Service D
```

Each service reacts to events.

---

# 6. No Central Coordinator

The defining characteristic is:

```text
No central transaction coordinator
```

Instead:

```text
          Message Broker
        /       |       \
       /        |        \
      v         v         v
   Order    Inventory   Payment
```

Services communicate through events.

For example:

```text
OrderCreated
InventoryReserved
PaymentCompleted
ShipmentCreated
```

Each service knows:

> "When I receive this event, what local operation should I perform?"

---

# 7. Example: Complete Order Saga

Let's build a complete example.

The user clicks:

```text
Place Order
```

The Order Service performs:

```sql
BEGIN;

INSERT INTO orders (...);

COMMIT;
```

After successfully committing:

```text
OrderCreated
```

is published.

Then:

```text
OrderCreated
      ↓
Inventory Service
```

Inventory Service performs:

```sql
BEGIN;

UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 101
AND quantity > 0;

COMMIT;
```

Then publishes:

```text
InventoryReserved
```

Next:

```text
InventoryReserved
      ↓
Payment Service
```

Payment Service processes the payment.

If successful:

```text
PaymentCompleted
```

is published.

Then:

```text
PaymentCompleted
      ↓
Shipping Service
```

Shipping Service creates the shipment.

Finally:

```text
ShipmentCreated
```

is published.

The entire Saga has completed.

---

# 8. Important Point: Each Service Owns Its Transaction

Consider:

```text
Order Service
```

It controls:

```text
Order DB
```

Inventory Service controls:

```text
Inventory DB
```

Payment Service controls:

```text
Payment DB
```

Each service executes its own local transaction.

```text
Order Service
   |
   +---- BEGIN
   +---- INSERT Order
   +---- COMMIT


Inventory Service
   |
   +---- BEGIN
   +---- UPDATE Inventory
   +---- COMMIT


Payment Service
   |
   +---- BEGIN
   +---- INSERT Payment
   +---- COMMIT
```

There is no global database transaction.

---

# 9. Events Drive the Saga

The most important concept in choreography is:

> **An event produced by one service becomes the trigger for another service.**

For example:

```text
OrderCreated
      ↓
Inventory Service
```

Then:

```text
InventoryReserved
      ↓
Payment Service
```

Then:

```text
PaymentCompleted
      ↓
Shipping Service
```

So the chain becomes:

```text
OrderCreated
      ↓
InventoryReserved
      ↓
PaymentCompleted
      ↓
ShipmentCreated
```

---

# 10. Events vs Commands

This distinction is important.

An event describes:

> Something already happened.

For example:

```text
OrderCreated
```

means:

```text
The order has been created.
```

A command generally means:

> Please perform this action.

For example:

```text
ReserveInventory
```

means:

```text
Please reserve inventory.
```

In choreography, event-driven communication is commonly used to trigger the next local transaction.

---

# 11. Event Example

An event might look like:

```json
{
  "eventId": "evt_123",
  "eventType": "OrderCreated",
  "transactionId": "txn_456",
  "orderId": "order_789",
  "userId": "user_101",
  "items": [
    {
      "productId": "product_1",
      "quantity": 1
    }
  ],
  "timestamp": "2026-09-29T10:00:00Z"
}
```

The important fields can include:

```text
eventId
eventType
transactionId
entityId
timestamp
payload
```

---

# 12. Transaction ID Across the Saga

A distributed Saga should generally have a way to identify the overall business transaction.

For example:

```text
transactionId = TXN123
```

Then:

```text
OrderCreated
transactionId = TXN123
```

Inventory event:

```text
InventoryReserved
transactionId = TXN123
```

Payment event:

```text
PaymentCompleted
transactionId = TXN123
```

This makes it possible to correlate the complete workflow.

```text
TXN123
 |
 +--> OrderCreated
 |
 +--> InventoryReserved
 |
 +--> PaymentCompleted
 |
 +--> ShipmentCreated
```

---

# 13. What Happens If Inventory Fails?

Suppose:

```text
OrderCreated
      ↓
Inventory Service
      ↓
Reservation FAILED
```

The Inventory Service may publish:

```text
InventoryReservationFailed
```

Now another service can react to that event.

For example:

```text
InventoryReservationFailed
          ↓
Order Service
          ↓
Cancel Order
```

The order might move from:

```text
PENDING
```

to:

```text
CANCELLED
```

This is where **compensating transactions** become important.

---

# 14. Compensation in Choreography

Suppose:

```text
OrderCreated
     ↓
InventoryReserved
     ↓
PaymentFailed
```

At this point:

```text
Order = Created
Inventory = Reserved
Payment = Failed
```

The previous successful operations may need compensation.

Payment failure can produce:

```text
PaymentFailed
```

Inventory Service listens:

```text
PaymentFailed
      ↓
Release Inventory
```

Inventory then publishes:

```text
InventoryReleased
```

Order Service can listen:

```text
InventoryReleased
      ↓
Cancel Order
```

The flow becomes:

```text
OrderCreated
      ↓
InventoryReserved
      ↓
PaymentFailed
      ↓
InventoryReleased
      ↓
OrderCancelled
```

This is a compensating workflow.

---

# 15. Forward Flow vs Compensation Flow

A Saga usually has two conceptual directions.

### Forward flow

```text
OrderCreated
      ↓
InventoryReserved
      ↓
PaymentCompleted
      ↓
ShipmentCreated
```

### Compensation flow

If something fails:

```text
PaymentFailed
      ↓
ReleaseInventory
      ↓
CancelOrder
```

Therefore:

```text
Forward transaction
        +
Compensating transactions
        =
Saga
```

---

# 16. Compensation Is Not Database Rollback

This distinction is extremely important.

Suppose payment succeeds:

```text
Payment = SUCCESS
```

Later shipping fails.

You cannot necessarily do:

```text
ROLLBACK Payment DB
```

because the payment transaction may already be committed.

Instead:

```text
Payment SUCCESS
      ↓
Shipping FAILED
      ↓
Refund Payment
```

The refund is a new business operation.

Therefore:

```text
Rollback
    ≠
Compensation
```

---

# 17. Example of Full Failure Flow

Suppose:

```text
1. Order Created
2. Inventory Reserved
3. Payment Successful
4. Shipping Failed
```

State:

```text
Order       = CREATED
Inventory   = RESERVED
Payment     = SUCCESS
Shipping    = FAILED
```

Compensation might be:

```text
ShippingFailed
      |
      +----> Refund Payment
      |
      +----> Release Inventory
      |
      +----> Cancel Order
```

Eventually:

```text
Order       = CANCELLED
Inventory   = AVAILABLE
Payment     = REFUNDED
Shipping    = NOT_CREATED
```

The system has reached a valid business state again.

---

# 18. Choreography With a Message Broker

A message broker is commonly used for communication.

For example:

```text
              Message Broker
             /       |       \
            /        |        \
           v         v         v
      Order       Inventory   Payment
      Service      Service     Service
```

Potential technologies include:

```text
Kafka
RabbitMQ
NATS
AWS SNS/SQS
```

The broker provides mechanisms for delivering events between services.

---

# 19. Topic-Based Example

With a topic-based broker:

```text
Order Service
     |
     | publish
     v
orders.events
     |
     +------------------+
     |                  |
     v                  v
Inventory Service   Analytics Service
```

Inventory may subscribe to:

```text
OrderCreated
```

Analytics may also subscribe to:

```text
OrderCreated
```

This provides loose coupling between producers and consumers.

---

# 20. Choreography and Loose Coupling

The Order Service doesn't necessarily need to know:

```text
Who consumes OrderCreated?
```

It simply publishes:

```text
OrderCreated
```

Other services can subscribe.

```text
Order Service
      |
      v
OrderCreated
      |
      +----> Inventory
      |
      +----> Analytics
      |
      +----> Notification
```

This can make services more independently deployable.

---

# 21. But Choreography Does Not Mean No Coupling

There is still coupling.

Instead of direct API coupling:

```text
Order Service
      |
      HTTP
      |
Inventory Service
```

we have event/schema coupling:

```text
OrderCreated
      |
      v
Inventory Service
```

Inventory Service depends on the event contract.

For example:

```json
{
  "orderId": "123",
  "items": [...]
}
```

Changing this event structure can affect consumers.

Therefore:

> Choreography reduces direct runtime coupling, but does not eliminate contracts between services.

---

# 22. Event Contract

An event should have a well-defined schema.

For example:

```json
{
  "eventType": "InventoryReserved",
  "eventVersion": 1,
  "eventId": "evt_123",
  "transactionId": "txn_456",
  "orderId": "order_789",
  "items": [
    {
      "productId": "p1",
      "quantity": 2
    }
  ]
}
```

Important fields can include:

```text
eventType
eventVersion
eventId
transactionId
entityId
timestamp
payload
```

---

# 23. Event Versioning

Suppose the first version is:

```json
{
  "orderId": "123",
  "amount": 1000
}
```

Later we need:

```json
{
  "orderId": "123",
  "amount": 1000,
  "currency": "INR"
}
```

Existing consumers may still expect the old structure.

Therefore, event contracts should be designed with compatibility in mind.

For example:

```text
OrderCreated v1
OrderCreated v2
```

or backward-compatible schema evolution.

---

# 24. The Dual-Write Problem in Choreography

Consider:

```text
Order Service
```

It needs to:

```text
1. Save order
2. Publish OrderCreated
```

Naively:

```text
BEGIN
 |
INSERT Order
 |
COMMIT
 |
Publish Event
```

Suppose:

```text
DB commit -> SUCCESS
Event publish -> FAILURE
```

Now:

```text
Order exists
```

but:

```text
OrderCreated event doesn't exist
```

Inventory never receives the event.

This is a major problem.

---

# 25. Why This Matters

The system now has:

```text
Order DB
   |
Order = CREATED
```

but:

```text
Inventory Service
   |
doesn't know about order
```

The business state is inconsistent.

Therefore:

> Event-driven distributed transactions require a reliable way to connect local database state with event publication.

A common solution is the **Transactional Outbox Pattern**, which can be covered separately.

---

# 26. Duplicate Events

Message delivery can result in duplicate events.

Suppose:

```text
OrderCreated
```

is delivered twice:

```text
OrderCreated
     |
     +----> Inventory
     |
     +----> Inventory again
```

Inventory might attempt to reserve the same inventory twice.

Therefore consumers should generally be **idempotent**.

---

# 27. Idempotent Consumer

Suppose:

```text
eventId = EVT123
```

Inventory receives:

```text
EVT123
```

It processes it.

Then receives:

```text
EVT123
```

again.

The consumer checks:

```text
Have I already processed EVT123?
```

If yes:

```text
Ignore duplicate
```

Conceptually:

```text
Event
 |
 v
Check eventId
 |
 +---- already processed ---> Ignore
 |
 +---- new -----------------> Process
```

This is an important reliability mechanism for choreography.

---

# 28. Out-of-Order Events

Events may sometimes arrive in an unexpected order.

For example:

```text
PaymentCompleted
```

could be processed before another expected event has been handled.

A service must not blindly assume that all events arrive in perfect sequence unless the messaging infrastructure and architecture explicitly guarantee the required ordering.

Possible mechanisms include:

```text
sequence numbers
versions
state validation
partition ordering
event timestamps
```

depending on the system.

---

# 29. Eventual Consistency in Choreography

Saga choreography usually results in **eventual consistency**.

Consider:

```text
T0:
Order = CREATED

T1:
Inventory = RESERVED

T2:
Payment = PROCESSING

T3:
Payment = SUCCESS

T4:
Shipping = CREATED
```

There may be periods where the system is temporarily inconsistent from the perspective of the complete business workflow.

Eventually:

```text
Order       = CONFIRMED
Inventory   = RESERVED
Payment     = SUCCESS
Shipping    = CREATED
```

The system converges toward the desired state.

---

# 30. Intermediate States Are Normal

A Saga often needs explicit intermediate states.

For example:

```text
Order:
PENDING
  ↓
CONFIRMED
  ↓
SHIPPED
```

Payment:

```text
PENDING
  ↓
SUCCESS
```

Inventory:

```text
AVAILABLE
  ↓
RESERVED
```

These states are useful because the complete workflow does not finish instantaneously.

---

# 31. Saga State Machine

A business process can be thought of as a state machine.

Example:

```text
                    +----------------+
                    |                |
                    v                |
PENDING → INVENTORY_RESERVED → PAYMENT_PENDING
   |                                  |
   |                                  v
   |                             PAYMENT_SUCCESS
   |                                  |
   |                                  v
   |                              CONFIRMED
   |
   +------> CANCELLED
```

Failure transitions are also part of the design.

For example:

```text
PAYMENT_FAILED
      ↓
INVENTORY_RELEASED
      ↓
ORDER_CANCELLED
```

---

# 32. Choreography Failure Scenario

Consider:

```text
OrderCreated
     ↓
InventoryReserved
     ↓
PaymentFailed
```

Payment publishes:

```text
PaymentFailed
```

Inventory consumes it:

```text
Release Inventory
```

But Inventory crashes before publishing:

```text
InventoryReleased
```

Now:

```text
Inventory = AVAILABLE
```

but Order Service may still have:

```text
Order = PENDING
```

The system needs reliable event processing and recovery.

This demonstrates that:

> Choreography moves coordination into event interactions; it does not eliminate distributed-system failure.

---

# 33. Event Delivery Guarantees

When designing choreography, you need to understand message delivery semantics.

Common concepts include:

```text
At-most-once
At-least-once
Exactly-once
```

### At-most-once

An event may be lost, but should not be delivered repeatedly.

```text
0 or 1 delivery
```

### At-least-once

An event should eventually be delivered, but duplicates may occur.

```text
1 or more deliveries
```

### Exactly-once

The system attempts to process the event exactly once.

In distributed systems, true exactly-once end-to-end side effects are difficult.

Therefore, many practical systems rely heavily on:

```text
At-least-once delivery
+
Idempotent consumers
```

---

# 34. Choreography and Retry

Suppose:

```text
Inventory Service
```

receives:

```text
OrderCreated
```

but its database is temporarily unavailable.

It can retry processing.

```text
OrderCreated
     |
     v
Inventory
     |
   ERROR
     |
   RETRY
     |
   RETRY
     |
 SUCCESS
```

However, retries can create duplicate side effects.

Therefore:

```text
Retry
+
Idempotency
```

must be designed together.

---

# 35. Poison Messages

A message can repeatedly fail processing.

For example:

```text
OrderCreated
     |
Inventory
     |
ERROR
     |
Retry
     |
ERROR
     |
Retry
     |
ERROR
```

Eventually the message may need to be moved to a:

```text
Dead Letter Queue
```

or equivalent failure-handling mechanism.

Conceptually:

```text
Main Queue
    |
    v
Consumer
    |
    +---- SUCCESS
    |
    +---- FAILURE
              |
            Retry
              |
            Retry
              |
              v
       Dead Letter Queue
```

This prevents one problematic message from blocking normal processing indefinitely.

---

# 36. Observability

Choreographed systems can become difficult to debug because the workflow is spread across services.

A single business transaction might produce:

```text
Order Service
      ↓
Kafka
      ↓
Inventory Service
      ↓
Kafka
      ↓
Payment Service
      ↓
Kafka
      ↓
Shipping Service
```

Therefore, logs should contain:

```text
transactionId
eventId
eventType
service
timestamp
```

For example:

```text
transactionId=TXN123
eventId=EVT456
eventType=PaymentCompleted
service=payment-service
```

This allows engineers to reconstruct the Saga.

---

# 37. Choreography Debugging

Suppose a customer says:

> "My payment succeeded but my order is still pending."

You need to trace:

```text
TXN123
   |
   +--> OrderCreated
   |
   +--> InventoryReserved
   |
   +--> PaymentCompleted
   |
   X
ShipmentCreated missing
```

Without correlation identifiers and good event logs, debugging becomes extremely difficult.

---

# 38. Advantages of Saga Choreography

## 1. No central coordinator

Services communicate through events.

```text
Service
  ↓
Event
  ↓
Service
```

---

## 2. Loose runtime coupling

Services don't need synchronous calls to every downstream service.

---

## 3. Natural event-driven architecture

It fits systems already using:

```text
Kafka
RabbitMQ
NATS
SNS/SQS
```

---

## 4. Services can react independently

Multiple consumers can respond to the same event.

```text
OrderCreated
   |
   +---- Inventory
   +---- Analytics
   +---- Notification
```

---

## 5. Better fit for asynchronous workflows

Long-running business processes can progress through events rather than holding one long database transaction.

---

# 39. Challenges of Saga Choreography

Choreography also introduces significant complexity.

## 1. Difficult to understand large workflows

A small Saga might be:

```text
A → B → C
```

But a large system can become:

```text
          → B
         /
A → Event → C → Event → D
         \
          → E
```

Understanding the complete workflow becomes harder.

---

## 2. Distributed business logic

Business workflow logic is spread across multiple services.

One service knows one part.

Another service knows another part.

---

## 3. Harder debugging

A single business operation can involve many:

```text
services
events
queues
retries
databases
```

---

## 4. Event contract coupling

Changing an event can affect many consumers.

---

## 5. Failure handling is complex

You must handle:

```text
duplicate events
lost events
out-of-order events
consumer failures
retries
dead letters
compensation
```

---

## 6. Cyclic dependencies can emerge

Poorly designed event relationships can create:

```text
A → B
B → C
C → A
```

This makes the workflow difficult to reason about.

---

# 40. Choreography Example — Success

```text
             Order Service
                   |
             OrderCreated
                   |
                   v
          +----------------+
          | Message Broker |
          +----------------+
                   |
                   v
          Inventory Service
                   |
           InventoryReserved
                   |
                   v
          +----------------+
          | Message Broker |
          +----------------+
                   |
                   v
           Payment Service
                   |
           PaymentCompleted
                   |
                   v
          +----------------+
          | Message Broker |
          +----------------+
                   |
                   v
          Shipping Service
                   |
            ShipmentCreated
```

Final state:

```text
Order       = CONFIRMED
Inventory   = RESERVED
Payment     = SUCCESS
Shipment    = CREATED
```

---

# 41. Choreography Example — Failure

```text
OrderCreated
      ↓
InventoryReserved
      ↓
PaymentFailed
```

Compensation:

```text
PaymentFailed
      ↓
ReleaseInventory
      ↓
InventoryReleased
      ↓
CancelOrder
```

Final state:

```text
Order       = CANCELLED
Inventory   = AVAILABLE
Payment     = FAILED
```

---

# 42. Choreography Mental Model

The easiest way to remember Saga Choreography:

```text
Local Transaction
       ↓
Publish Event
       ↓
Another Service Reacts
       ↓
Local Transaction
       ↓
Publish Event
       ↓
Another Service Reacts
```

Failure:

```text
Local Transaction
       ↓
Failure Event
       ↓
Another Service
       ↓
Compensating Transaction
       ↓
Another Event
```

---

# 43. Distributed Transaction vs Saga Choreography

A traditional distributed transaction might try to achieve:

```text
Global Transaction
       |
       +---- DB A
       +---- DB B
       +---- DB C
```

with explicit coordination.

Saga choreography instead uses:

```text
Local Tx A
    ↓
Event
    ↓
Local Tx B
    ↓
Event
    ↓
Local Tx C
```

The fundamental difference is:

```text
Distributed Transaction
→ coordinate a global transaction

Saga Choreography
→ coordinate a business workflow through local transactions and events
```

---

# 44. Important Interview Point

If an interviewer asks:

> "What happens if payment fails after inventory has already been reserved?"

A good answer is:

```text
Payment Service publishes PaymentFailed.

Inventory Service consumes PaymentFailed
and performs a compensating transaction
to release the inventory.

It then publishes InventoryReleased.

Order Service can consume that event and
move the order to CANCELLED.
```

The important words are:

```text
Local transaction
Event
Compensation
Idempotency
Eventual consistency
```

---

# 45. Important Interview Point: Why Not Rollback Everything?

Because each service owns its own database.

For example:

```text
Order DB
Inventory DB
Payment DB
```

Once:

```text
Inventory transaction -> COMMITTED
```

another service cannot simply execute:

```text
ROLLBACK Inventory DB
```

Instead, Inventory must perform a new business operation:

```text
Release Inventory
```

This is compensation.

---

# 46. Important Interview Point: What If the Event Is Delivered Twice?

Use an idempotent consumer.

For example:

```text
eventId = EVT123
```

Store processed event IDs:

```text
processed_events
----------------
EVT123
```

When `EVT123` arrives again:

```text
Already processed?
       |
      YES
       |
     Ignore
```

The exact implementation can vary, but duplicate handling must be explicitly designed.

---

# 47. Important Interview Point: What If DB Commit Succeeds but Event Publishing Fails?

This is the **dual-write problem**.

Example:

```text
DB COMMIT
   ↓
SUCCESS

Event Publish
   ↓
FAILURE
```

The local state exists but downstream services don't know about it.

A common solution is the **Transactional Outbox Pattern**:

```text
BEGIN
   |
   +---- Save business data
   |
   +---- Save event to Outbox table
   |
COMMIT
```

A separate publisher later publishes the outbox event.

The Outbox Pattern can be covered separately.

---

# 48. Important Interview Point: Does Saga Give ACID?

Saga does not provide traditional global ACID semantics in the same way as a single database transaction.

Instead, Saga generally provides:

```text
Local ACID transactions
+
Event-driven coordination
+
Compensating actions
```

The overall workflow typically achieves:

```text
Eventual consistency
```

rather than one instantaneous globally atomic database commit.

---

# 49. When Choreography Is Useful

Choreography can be useful when:

```text
- workflows are event-driven
- services are relatively independent
- asynchronous processing is acceptable
- services need to react independently to events
- the number of participants is manageable
- eventual consistency is acceptable
```

The appropriate choice depends on the workflow and consistency requirements.

---

# 50. When Choreography Becomes Difficult

As the number of services and events increases:

```text
A
 |
Event
 |
B
 |
Event
 |
C
 |
Event
 |
D
```

can become increasingly difficult to understand.

With many branching events:

```text
             B
            /
A -------- C -------- D
            \
             E
              \
               F
```

the complete business workflow may become distributed across many codebases.

This is sometimes called **event-driven spaghetti** when event relationships become excessively complicated.

---

# 51. Complete Mental Model

A Saga choreography can be summarized as:

```text
                 BUSINESS TRANSACTION
                         |
                         v
                +----------------+
                | Order Service  |
                +----------------+
                         |
                  Local Transaction
                         |
                         v
                  OrderCreated
                         |
                         v
                +----------------+
                | Message Broker |
                +----------------+
                         |
                         v
             +----------------------+
             | Inventory Service    |
             +----------------------+
                         |
                  Local Transaction
                         |
                         v
                InventoryReserved
                         |
                         v
                +----------------+
                | Message Broker |
                +----------------+
                         |
                         v
             +----------------------+
             | Payment Service      |
             +----------------------+
                         |
                  Local Transaction
                         |
                  +------+------+
                  |             |
               SUCCESS        FAILURE
                  |             |
                  v             v
        PaymentCompleted    PaymentFailed
                  |             |
                  v             v
             Next Step      Compensation
```

The key principle is:

> **There is no global database transaction. Each service commits its own local transaction and communicates the result through events. Failures are handled through compensating transactions.**

---

# Key Takeaways

```text
Saga
=
Sequence of local transactions
+
Events
+
Compensation
```

### Choreography

```text
Service A
   |
   | Event
   v
Service B
   |
   | Event
   v
Service C
```

### Success

```text
Local Tx
   ↓
Event
   ↓
Local Tx
   ↓
Event
   ↓
Success
```

### Failure

```text
Local Tx
   ↓
Event
   ↓
Local Tx
   ↓
Failure
   ↓
Compensating Transaction
   ↓
Event
```

### Core concepts to remember

```text
Local Transactions
Events
Message Broker
Eventual Consistency
Compensating Transactions
Idempotency
Transaction ID
Event ID
Event Contracts
Retries
Duplicate Events
Out-of-Order Events
Dead Letter Queue
Dual-Write Problem
Transactional Outbox
Failure Recovery
Observability
```

The central idea is:

> **Saga Choreography replaces one large distributed transaction with a sequence of independently committed local transactions connected by events, with compensating transactions used when later steps fail.**

# Saga Pattern — Orchestration

**Saga Orchestration** is another way of implementing the Saga Pattern for distributed transactions.

Like Saga Choreography, orchestration breaks a large distributed business transaction into a sequence of **local transactions**.

The major difference is **how the workflow is coordinated**.

In orchestration:

> **A central component called the Orchestrator controls the Saga workflow and tells participating services what action to perform next.**

Instead of services discovering the next step by reacting to each other's events:

```text id="choreography"
Service A
   ↓ event
Service B
   ↓ event
Service C
```

we have:

```text id="orchestration"
              Orchestrator
              /    |    \
             /     |     \
            v      v      v
       Service A Service B Service C
```

The Orchestrator knows:

- what step should execute first
- what should execute next
- what to do when a step succeeds
- what to do when a step fails
- which compensation should be executed
- when the Saga has completed
- when the Saga has failed

---

# 1. Basic Idea

Suppose an e-commerce system needs to:

```text
Create Order
Reserve Inventory
Process Payment
Create Shipment
```

With orchestration:

```text id="basic-flow"
                 Orchestrator
                      |
                      | Create Order
                      v
                Order Service
                      |
                  SUCCESS
                      |
                      v
                 Orchestrator
                      |
                      | Reserve Inventory
                      v
              Inventory Service
                      |
                  SUCCESS
                      |
                      v
                 Orchestrator
                      |
                      | Process Payment
                      v
               Payment Service
                      |
                  SUCCESS
                      |
                      v
                 Orchestrator
                      |
                      | Create Shipment
                      v
              Shipping Service
```

The Orchestrator controls the workflow.

---

# 2. What Is the Orchestrator?

The **Orchestrator** is a dedicated component responsible for coordinating the Saga.

Conceptually:

```text id="orchestrator"
                +----------------+
                |  Orchestrator  |
                +----------------+
                  /      |      \
                 /       |       \
                v        v        v
             Order   Inventory   Payment
```

It maintains knowledge of the workflow.

For example:

```text id="workflow"
Step 1 → Create Order
Step 2 → Reserve Inventory
Step 3 → Process Payment
Step 4 → Create Shipment
```

The Orchestrator executes these steps according to the workflow definition.

---

# 3. Orchestrator vs Participant

There are two important roles.

## Orchestrator

Responsible for:

```text
Workflow coordination
State tracking
Next-step decisions
Failure handling
Compensation
Retries
```

## Participant

Responsible for:

```text
Its own business operation
Its own database
Its own local transaction
```

For example:

```text id="roles"
Orchestrator
     |
     +---- Order Service
     |
     +---- Inventory Service
     |
     +---- Payment Service
     |
     +---- Shipping Service
```

The Orchestrator does **not** own the participant's database transaction.

Each service still owns its own data.

---

# 4. Important: Orchestrator Does Not Mean One Giant Transaction

This is a very common misunderstanding.

Orchestration does **not** mean:

```text id="wrong"
BEGIN

Order
Inventory
Payment
Shipping

COMMIT
```

Instead:

```text id="correct"
Order Local Transaction
        ↓
Inventory Local Transaction
        ↓
Payment Local Transaction
        ↓
Shipping Local Transaction
```

Each service commits independently.

The Orchestrator coordinates the business workflow between them.

---

# 5. Complete Order Example

Suppose the user places an order.

The request enters:

```text id="request"
POST /orders
```

The system starts a Saga:

```text id="start"
Saga ID = SAGA123
```

The Orchestrator might maintain:

```text id="state"
Saga ID: SAGA123

Current Step:
CREATE_ORDER

Status:
RUNNING
```

It sends:

```text id="command"
CreateOrder
```

to the Order Service.

---

# 6. Step 1 — Create Order

Order Service performs a local transaction:

```sql id="sql"
BEGIN;

INSERT INTO orders (
    id,
    user_id,
    status
)
VALUES (
    'ORDER123',
    'USER123',
    'PENDING'
);

COMMIT;
```

The Order Service returns:

```text id="success"
OrderCreated
```

or an equivalent success response/event.

The Orchestrator updates its state:

```text id="state2"
CREATE_ORDER
      ↓
SUCCESS
      ↓
NEXT = RESERVE_INVENTORY
```

---

# 7. Step 2 — Reserve Inventory

The Orchestrator sends:

```text id="reserve"
ReserveInventory
```

to Inventory Service.

Inventory Service executes:

```sql id="inventory"
BEGIN;

UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 'P123'
AND quantity > 0;

COMMIT;
```

If successful:

```text id="inventory-success"
InventoryReserved
```

The Orchestrator receives the result.

Now:

```text id="flow"
Create Order       ✓
Reserve Inventory  ✓
Process Payment    ← current
```

---

# 8. Step 3 — Process Payment

The Orchestrator sends:

```text id="payment"
ProcessPayment
```

to Payment Service.

Payment Service performs its local transaction.

If successful:

```text id="payment-success"
PaymentCompleted
```

The Orchestrator moves forward:

```text id="flow2"
Create Order       ✓
Reserve Inventory  ✓
Process Payment    ✓
Create Shipment    ← current
```

---

# 9. Step 4 — Create Shipment

The Orchestrator sends:

```text id="shipping"
CreateShipment
```

to Shipping Service.

Shipping Service performs its local transaction.

If successful:

```text id="shipping-success"
ShipmentCreated
```

The Orchestrator marks:

```text id="complete"
Saga = COMPLETED
```

Final state:

```text id="final"
Order       = CONFIRMED
Inventory   = RESERVED
Payment     = SUCCESS
Shipment    = CREATED
Saga        = COMPLETED
```

---

# 10. The Complete Flow

The complete orchestration workflow:

```text id="complete-flow"
                       Orchestrator
                            |
                            |
                       Create Order
                            |
                            v
                      Order Service
                            |
                         SUCCESS
                            |
                            v
                       Orchestrator
                            |
                            |
                    Reserve Inventory
                            |
                            v
                   Inventory Service
                            |
                         SUCCESS
                            |
                            v
                       Orchestrator
                            |
                            |
                     Process Payment
                            |
                            v
                     Payment Service
                            |
                         SUCCESS
                            |
                            v
                       Orchestrator
                            |
                            |
                     Create Shipment
                            |
                            v
                    Shipping Service
                            |
                         SUCCESS
                            |
                            v
                       COMPLETED
```

---

# 11. Commands in Orchestration

Orchestration commonly uses **commands** to tell services what operation to perform.

Examples:

```text id="commands"
CreateOrder
ReserveInventory
ProcessPayment
CreateShipment
```

The meaning is:

> "Please perform this operation."

This is different from an event.

An event says:

```text id="event"
OrderCreated
```

Meaning:

> "The order has already been created."

A command says:

```text id="command2"
CreateOrder
```

Meaning:

> "Please create the order."

---

# 12. Command-Driven Workflow

The Orchestrator can follow:

```text id="command-flow"
Orchestrator
     |
     | CreateOrder
     v
Order Service
     |
     | OrderCreated
     v
Orchestrator
     |
     | ReserveInventory
     v
Inventory Service
     |
     | InventoryReserved
     v
Orchestrator
```

The Orchestrator decides what command should be sent next.

---

# 13. Orchestrator State

The Orchestrator usually needs to know the current Saga state.

For example:

```text id="saga-state"
Saga ID: SAGA123

Order:
CREATED

Inventory:
RESERVED

Payment:
PENDING

Shipping:
NOT_STARTED

Saga:
RUNNING
```

After payment:

```text id="saga-state2"
Saga ID: SAGA123

Order:
CREATED

Inventory:
RESERVED

Payment:
SUCCESS

Shipping:
PENDING

Saga:
RUNNING
```

After shipping:

```text id="saga-state3"
Saga ID: SAGA123

Order:
CREATED

Inventory:
RESERVED

Payment:
SUCCESS

Shipping:
CREATED

Saga:
COMPLETED
```

---

# 14. Why Does the Orchestrator Need State?

Because the workflow can be long-running.

Suppose:

```text id="long-running"
Order
   ↓
Inventory
   ↓
Payment
   ↓
Fraud Check
   ↓
Shipping
```

The entire Saga might take seconds, minutes, or potentially longer depending on the business process.

The Orchestrator must know:

```text id="state-question"
Where did this Saga stop?
```

For example:

```text id="state-answer"
SAGA123

Order       = SUCCESS
Inventory   = SUCCESS
Payment     = FAILED
Shipping    = NOT_STARTED
```

Now it knows exactly where the failure occurred.

---

# 15. Orchestrator State Machine

The Saga can be modeled as a state machine:

```text id="state-machine"
START
  |
  v
CREATE_ORDER
  |
  v
RESERVE_INVENTORY
  |
  v
PROCESS_PAYMENT
  |
  v
CREATE_SHIPMENT
  |
  v
COMPLETED
```

Failure paths can also be defined:

```text id="failure-machine"
CREATE_ORDER
     |
   FAIL
     |
     v
FAILED


RESERVE_INVENTORY
     |
   FAIL
     |
     v
CANCEL_ORDER
     |
     v
FAILED
```

Payment failure:

```text id="payment-failure"
PROCESS_PAYMENT
      |
    FAIL
      |
      v
RELEASE_INVENTORY
      |
      v
CANCEL_ORDER
      |
      v
FAILED
```

---

# 16. Compensation in Orchestration

This is one of the most important differences in how the workflow is expressed.

Suppose:

```text id="comp-start"
Create Order       ✓
Reserve Inventory  ✓
Payment            ✓
Create Shipment    ✗
```

The Orchestrator knows exactly what has already succeeded.

It can execute compensations:

```text id="comp-flow"
Create Shipment
      |
    FAILED
      |
      v
Refund Payment
      |
      v
Release Inventory
      |
      v
Cancel Order
```

The Orchestrator explicitly decides the compensation sequence.

---

# 17. Forward Actions and Compensation Actions

The Orchestrator may maintain a workflow like:

```text id="forward-comp"
Forward:

CreateOrder
ReserveInventory
ProcessPayment
CreateShipment
```

and corresponding compensations:

```text
Compensation:

CancelOrder
ReleaseInventory
RefundPayment
CancelShipment
```

For example:

```text id="comp-map"
CreateOrder
    ↓
CancelOrder

ReserveInventory
    ↓
ReleaseInventory

ProcessPayment
    ↓
RefundPayment

CreateShipment
    ↓
CancelShipment
```

The compensation operation depends on the business semantics.

---

# 18. Example: Payment Failure

Suppose:

```text id="payment-failure-flow"
Create Order       ✓
Reserve Inventory  ✓
Process Payment    ✗
```

The Orchestrator knows:

```text id="known-state"
Order = CREATED
Inventory = RESERVED
Payment = FAILED
```

It can execute:

```text id="compensation"
ReleaseInventory
       ↓
CancelOrder
```

Final state:

```text id="final-state"
Order       = CANCELLED
Inventory   = AVAILABLE
Payment     = FAILED
```

---

# 19. Example: Shipping Failure

Suppose:

```text id="shipping-failure-flow"
Create Order       ✓
Reserve Inventory  ✓
Payment            ✓
Create Shipment    ✗
```

The Orchestrator may execute:

```text id="shipping-comp"
RefundPayment
      ↓
ReleaseInventory
      ↓
CancelOrder
```

Final state:

```text id="shipping-final"
Order       = CANCELLED
Inventory   = AVAILABLE
Payment     = REFUNDED
Shipment    = NOT_CREATED
```

Again, the exact compensation order depends on business requirements.

---

# 20. Compensation Is Not Rollback

Even with orchestration:

```text id="rollback"
Payment SUCCESS
```

cannot necessarily be magically rolled back.

Instead:

```text id="refund"
Payment SUCCESS
     ↓
RefundPayment
```

is another business transaction.

Therefore:

```text id="comp-vs-rollback"
Database Rollback
       ≠
Saga Compensation
```

---

# 21. Orchestrator Does Not Directly Modify Other Services' Databases

A good architectural boundary is:

```text id="boundary"
Orchestrator
     |
     | command
     v
Inventory Service
     |
     v
Inventory DB
```

Not:

```text id="bad-boundary"
Orchestrator
     |
     +---- UPDATE inventory DB
     +---- UPDATE payment DB
     +---- UPDATE order DB
```

The Orchestrator should coordinate business operations.

The individual service owns:

```text id="ownership"
Business logic
Database
Local transaction
```

---

# 22. Service Ownership

For example:

```text id="ownership2"
Order Service
    |
    +---- Order DB


Inventory Service
    |
    +---- Inventory DB


Payment Service
    |
    +---- Payment DB
```

The Orchestrator doesn't bypass these boundaries.

Instead:

```text id="ownership3"
Orchestrator
     |
     | ReserveInventory
     v
Inventory Service
     |
     v
Inventory DB
```

This preserves service ownership.

---

# 23. Synchronous Orchestration

The Orchestrator can communicate synchronously.

For example:

```text id="sync"
Orchestrator
     |
     | HTTP
     v
Order Service
     |
   response
     |
     v
Orchestrator
```

Then:

```text id="sync2"
Orchestrator
     |
     | HTTP
     v
Inventory Service
     |
   response
     |
     v
Orchestrator
```

This is simple to understand but introduces runtime coupling.

If Inventory Service is unavailable:

```text id="sync-failure"
Orchestrator
     |
     | HTTP
     X
Inventory Service
```

the Orchestrator must handle the failure.

---

# 24. Asynchronous Orchestration

The Orchestrator can also communicate through messaging.

```text id="async"
Orchestrator
     |
     | ReserveInventory
     v
Message Broker
     |
     v
Inventory Service
     |
     | InventoryReserved
     v
Message Broker
     |
     v
Orchestrator
```

The Orchestrator waits for the result event.

This can reduce direct synchronous coupling.

---

# 25. Orchestration With a Message Broker

A common architecture:

```text id="broker"
                   +----------------+
                   |  Orchestrator  |
                   +----------------+
                         |
                         v
                  +-------------+
                  | Message     |
                  | Broker      |
                  +-------------+
                    /    |    \
                   /     |     \
                  v      v      v
              Order   Inventory Payment
             Service   Service   Service
```

Commands travel toward services:

```text id="commands2"
CreateOrder
ReserveInventory
ProcessPayment
```

Results/events travel back:

```text id="events2"
OrderCreated
InventoryReserved
PaymentCompleted
```

---

# 26. Orchestrator as a State Machine

A powerful way to implement an Orchestrator is as a state machine.

Example:

```text id="state-machine2"
             START
               |
               v
        CREATE_ORDER
          /       \
       success    failure
         |          |
         v          v
 RESERVE_INVENTORY FAILED
      /       \
   success    failure
     |          |
     v          v
 PROCESS      CANCEL
 PAYMENT      ORDER
```

This makes the workflow explicit.

---

# 27. Example State Table

The Orchestrator might maintain something like:

| Saga ID | Step | Status |
|---|---|---|
| SAGA123 | Create Order | SUCCESS |
| SAGA123 | Reserve Inventory | SUCCESS |
| SAGA123 | Payment | FAILED |
| SAGA123 | Release Inventory | SUCCESS |
| SAGA123 | Cancel Order | SUCCESS |
| SAGA123 | Saga | FAILED |

This state is extremely useful for recovery and debugging.

---

# 28. What If the Orchestrator Crashes?

This is one of the most important failure scenarios.

Suppose:

```text id="orch-crash"
Order       = CREATED
Inventory   = RESERVED
Payment     = PENDING
```

Then:

```text id="orch-crash2"
Orchestrator
     |
     X
   CRASH
```

If the Orchestrator only kept state in memory, it might forget:

```text id="lost"
SAGA123
currentStep = PROCESS_PAYMENT
```

That is unacceptable.

Therefore:

> Saga state should generally be durably persisted when recovery is required.

---

# 29. Durable Saga State

The Orchestrator can maintain a Saga state store:

```text id="durable"
                Orchestrator
                     |
                     v
               Saga State DB
                     |
          +----------+----------+
          |                     |
       SAGA123                SAGA124
          |                     |
      PAYMENT                SHIPPING
      PENDING                COMPLETE
```

If the Orchestrator crashes:

```text id="restart"
Orchestrator
     |
   RESTART
     |
     v
Saga State DB
     |
     v
SAGA123 = PAYMENT_PENDING
```

It can resume or recover the workflow.

---

# 30. Transactional State of the Orchestrator

The Orchestrator itself has state.

For example:

```text id="orch-state"
Saga ID: SAGA123

Current Step:
PROCESS_PAYMENT

Completed:
CREATE_ORDER
RESERVE_INVENTORY

Pending:
PROCESS_PAYMENT
CREATE_SHIPMENT
```

This state must be handled carefully.

Otherwise, the Orchestrator can itself become a source of inconsistency.

---

# 31. Orchestrator Retry

Suppose:

```text id="retry"
Orchestrator
     |
     | ProcessPayment
     v
Payment Service
     |
   timeout
```

The Orchestrator doesn't know whether:

```text id="retry-unknown"
Payment failed
```

or:

```text
Payment succeeded but response was lost
```

If it blindly retries:

```text id="danger"
ProcessPayment
     ↓
ProcessPayment
```

the customer could potentially be charged twice.

Therefore, payment operations need idempotency.

For example:

```text id="idem"
transactionId = SAGA123
paymentOperation = PROCESS_PAYMENT
```

Payment Service can detect duplicate requests.

---

# 32. Idempotency in Orchestration

Suppose the Orchestrator sends:

```text id="idem2"
ReserveInventory(SAGA123)
```

Inventory processes it successfully.

But the response is lost.

The Orchestrator retries:

```text id="idem3"
ReserveInventory(SAGA123)
```

Inventory must recognize:

```text id="idem4"
SAGA123 already processed
```

and avoid reserving another unit.

Therefore:

> Orchestration does not remove the need for idempotency.

It makes idempotency even more important because the Orchestrator actively retries failed/uncertain operations.

---

# 33. Timeouts

Each step can have a timeout.

Example:

```text id="timeout"
ProcessPayment
      |
      | wait
      |
      | timeout
      v
Payment step = UNKNOWN
```

The Orchestrator might:

```text id="timeout2"
1. Retry
2. Query payment status
3. Wait
4. Mark failure
5. Start compensation
```

The correct strategy depends on the business operation.

For payments, for example, blindly assuming timeout means failure can be dangerous.

---

# 34. Retry Policy

Not every error should be retried.

For example:

```text id="retry-policy"
Temporary network error
       |
      RETRY
```

But:

```text id="business-error"
Insufficient funds
       |
     DO NOT
     blindly retry
```

Therefore, the Orchestrator should distinguish between:

```text id="error-types"
Transient failures
Permanent failures
Business failures
Unknown failures
```

---

# 35. Exponential Backoff

For transient failures:

```text id="backoff"
Attempt 1 → wait 100ms
Attempt 2 → wait 200ms
Attempt 3 → wait 400ms
Attempt 4 → wait 800ms
```

This reduces pressure on an unhealthy service.

In production systems, retry policies may also include:

```text id="backoff2"
maximum attempts
maximum delay
jitter
circuit breaking
```

---

# 36. Circuit Breaker

Suppose Payment Service is down.

Without protection:

```text id="circuit"
Orchestrator
  |
  +--> Payment
  +--> Payment
  +--> Payment
  +--> Payment
  +--> Payment
```

The Orchestrator may overload the failing service.

A circuit breaker can temporarily stop calls:

```text id="circuit2"
Payment failures
      |
      v
Circuit OPEN
      |
      v
Stop sending requests
```

After a recovery period:

```text id="circuit3"
OPEN
 ↓
HALF-OPEN
 ↓
SUCCESS
 ↓
CLOSED
```

---

# 37. Orchestrator and Compensation Order

Suppose:

```text id="comp-order"
A → B → C → D
```

and D fails.

Possible compensations:

```text id="comp-order2"
D failed
 ↓
Compensate C
 ↓
Compensate B
 ↓
Compensate A
```

This is often called **reverse compensation**.

However, compensation order is not automatically always the exact reverse order.

It depends on business dependencies.

For example:

```text id="business"
Payment
Inventory
Shipping
```

may require a particular compensation sequence.

The workflow must define it explicitly.

---

# 38. Compensation Can Also Fail

This is a very important production concern.

Suppose:

```text id="comp-fail"
Payment SUCCESS
Shipping FAILED
```

Orchestrator sends:

```text id="refund"
RefundPayment
```

But:

```text id="refund-fail"
RefundPayment
      |
      X
    FAILED
```

Now compensation itself has failed.

The system needs recovery mechanisms.

Possible approaches include:

```text id="recover"
Retry
Status polling
Manual intervention
Reconciliation
Dead-letter handling
Persistent failure state
```

Therefore:

> A Saga is not complete merely because compensation exists. Compensation itself must be reliable and recoverable.

---

# 39. Long-Running Sagas

A Saga can run for a long period.

Example:

```text id="long"
Order
 ↓
Payment
 ↓
Fraud Verification
 ↓
Warehouse
 ↓
Shipping
```

Some steps might take minutes or longer.

The Orchestrator should not necessarily hold:

```text id="bad"
database locks
HTTP connections
memory state
```

for the entire workflow.

Instead, persist the Saga state and continue asynchronously.

---

# 40. Saga State Persistence

A simplified schema could be:

```sql id="schema"
CREATE TABLE saga_instances (
    saga_id UUID PRIMARY KEY,
    saga_type VARCHAR(100),
    status VARCHAR(50),
    current_step VARCHAR(100),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

Another table could track individual steps:

```sql id="schema2"
CREATE TABLE saga_steps (
    id UUID PRIMARY KEY,
    saga_id UUID,
    step_name VARCHAR(100),
    status VARCHAR(50),
    retry_count INT,
    started_at TIMESTAMP,
    completed_at TIMESTAMP
);
```

This allows the Orchestrator to know:

```text id="state-db"
What happened?
What succeeded?
What failed?
What needs retry?
What needs compensation?
```

---

# 41. Saga Step State

For example:

```text id="step-state"
SAGA123

CREATE_ORDER
    SUCCESS

RESERVE_INVENTORY
    SUCCESS

PROCESS_PAYMENT
    FAILED

REFUND_PAYMENT
    NOT_STARTED

RELEASE_INVENTORY
    NOT_STARTED

CANCEL_ORDER
    NOT_STARTED
```

The Orchestrator can use this information to continue recovery.

---

# 42. Orchestration and Eventual Consistency

Like choreography, Saga orchestration generally does not provide one global ACID transaction.

Instead:

```text id="eventual"
Local Transaction
      ↓
Local Commit
      ↓
Next Step
      ↓
Local Commit
      ↓
...
```

There can be intermediate states.

For example:

```text id="intermediate"
Order = CREATED
Inventory = RESERVED
Payment = PENDING
Shipping = NOT_CREATED
```

Eventually:

```text id="eventual-final"
Order = CONFIRMED
Inventory = RESERVED
Payment = SUCCESS
Shipping = CREATED
```

or, if the Saga fails:

```text id="eventual-failure"
Order = CANCELLED
Inventory = AVAILABLE
Payment = REFUNDED
Shipping = NOT_CREATED
```

---

# 43. Orchestration vs Traditional Distributed Transaction

Traditional distributed transaction:

```text id="traditional"
           Coordinator
          /     |     \
         v      v      v
        DB-A   DB-B   DB-C

Global transaction
```

Saga orchestration:

```text id="saga"
           Orchestrator
          /      |      \
         v       v       v
      Service A Service B Service C
         |         |         |
        DB-A      DB-B      DB-C
```

The major distinction:

```text id="difference"
Traditional distributed transaction
→ attempts coordinated atomic commit

Saga orchestration
→ coordinates independent local transactions
  and uses compensation for failures
```

---

# 44. Orchestration vs Choreography

This is the most important comparison.

## Choreography

```text id="choreo"
Service A
    |
  Event
    |
    v
Service B
    |
  Event
    |
    v
Service C
```

Services react to events.

---

## Orchestration

```text id="orch"
             Orchestrator
             /     |     \
            v      v      v
        Service A Service B Service C
```

The Orchestrator decides what happens next.

---

# 45. Responsibility Comparison

| Concern | Choreography | Orchestration |
|---|---|---|
| Workflow control | Distributed across services | Centralized in Orchestrator |
| Communication | Primarily events | Commands + responses/events |
| Central coordinator | No | Yes |
| Workflow visibility | Distributed | Explicit |
| Business flow | Spread across services | Defined in Orchestrator |
| Failure handling | Services react to failure events | Orchestrator decides compensation |
| Saga state | Distributed | Usually centralized |
| Debugging | Can be difficult | Workflow is easier to trace |
| Coordinator failure | No dedicated coordinator | Orchestrator must be highly available |
| Event coupling | Higher | Can be lower between participants |
| Complex workflows | Can become difficult | Usually easier to model explicitly |

The table describes architectural characteristics rather than universally better/worse choices.

---

# 46. Example Comparison

### Choreography

```text id="compare-choreo"
Order
  |
  | OrderCreated
  v
Inventory
  |
  | InventoryReserved
  v
Payment
  |
  | PaymentCompleted
  v
Shipping
```

Each service determines its reaction.

---

### Orchestration

```text id="compare-orch"
             Orchestrator
              /   |   \
             /    |    \
            v     v     v
         Order Inventory Payment
           |       |       |
         result  result   result
              \    |    /
               \   |   /
                Orchestrator
                     |
                     v
                 Shipping
```

The Orchestrator explicitly controls the sequence.

---

# 47. Advantages of Saga Orchestration

## 1. Centralized Workflow

The entire business process is visible in one place.

```text id="adv1"
Create Order
     ↓
Reserve Inventory
     ↓
Payment
     ↓
Shipping
```

This makes complex workflows easier to understand.

---

## 2. Easier Failure Handling

The Orchestrator knows which steps succeeded.

For example:

```text id="adv2"
Order       ✓
Inventory   ✓
Payment     ✗
Shipping    -
```

Therefore it knows which compensations are required.

---

## 3. Easier Monitoring

The Orchestrator can expose:

```text id="adv3"
Saga ID
Current Step
Status
Retry Count
Failure Reason
```

---

## 4. Easier Workflow Changes

Suppose the business adds:

```text id="adv4"
Fraud Check
```

The workflow can become:

```text id="adv5"
Order
 ↓
Inventory
 ↓
Payment
 ↓
Fraud Check
 ↓
Shipping
```

The workflow definition is centralized.

---

## 5. Explicit Business Workflow

The Orchestrator can make business transitions very clear.

```text id="adv6"
IF payment succeeds
    → create shipment

IF payment fails
    → release inventory
    → cancel order
```

---

# 48. Challenges of Saga Orchestration

## 1. Orchestrator Becomes Important Infrastructure

If the Orchestrator goes down:

```text id="challenge1"
Orchestrator
     |
     X
```

the Saga workflow may stop progressing.

Therefore it must be:

```text id="challenge2"
Highly available
Durable
Recoverable
Observable
```

---

## 2. Orchestrator Can Become Too Powerful

A poorly designed Orchestrator can become a **God service**.

For example, if it contains:

```text id="challenge3"
Order business logic
Inventory business logic
Payment business logic
Shipping business logic
```

then service boundaries become meaningless.

The Orchestrator should coordinate rather than own all domain logic.

---

## 3. Single Point of Coordination

Although the Orchestrator can be deployed redundantly, logically all workflow coordination passes through it.

This creates an important architectural dependency.

---

## 4. State Management

The Orchestrator needs durable state.

Otherwise:

```text id="challenge4"
Orchestrator crashes
       ↓
Saga state lost
       ↓
Recovery becomes difficult
```

---

## 5. More Infrastructure

You may need:

```text id="challenge5"
Orchestrator
Saga state store
Message broker
Retry mechanism
Dead-letter handling
Monitoring
```

---

# 49. Orchestrator Should Not Become a Bottleneck

Imagine:

```text id="bottleneck"
1 million Sagas
       |
       v
Orchestrator
```

The Orchestrator must be designed for scalability.

Possible techniques include:

```text id="scale"
Horizontal scaling
Partitioning by Saga ID
Durable queues
Stateless workers + external state
Distributed locks where necessary
Load balancing
```

The exact architecture depends on throughput and consistency requirements.

---

# 50. Concurrent Sagas

Suppose:

```text id="concurrent"
SAGA1 → Product A
SAGA2 → Product A
SAGA3 → Product A
```

All three may attempt:

```text id="concurrent2"
Reserve Inventory
```

The Inventory Service must enforce its own concurrency rules.

The Orchestrator does not replace:

```text id="concurrency3"
Database constraints
Locks
Optimistic concurrency
Atomic updates
```

Each service remains responsible for protecting its own data.

---

# 51. Orchestrator and Database Transactions

An Orchestrator might have its own local transaction.

For example:

```text id="orch-db"
BEGIN

UPDATE saga_instances
SET current_step = 'PAYMENT';

INSERT INTO saga_commands (...);

COMMIT
```

But this transaction only protects the Orchestrator's own state.

It does not atomically commit:

```text id="orch-db2"
Orchestrator DB
+
Payment DB
```

That remains a distributed consistency problem.

---

# 52. The Orchestrator's Own Dual-Write Problem

Suppose the Orchestrator:

```text id="orch-dual"
1. Updates Saga state
2. Sends command to Payment Service
```

Potential failure:

```text id="orch-dual2"
Update Saga State
       |
     COMMIT
       |
       X
Send Payment Command
```

Now the Orchestrator thinks:

```text id="orch-dual3"
Payment step started
```

but Payment Service never received the command.

The reverse is also possible:

```text id="orch-dual4"
Command sent
       |
       X
Saga state update fails
```

This is another form of the dual-write problem.

Reliable messaging patterns become important here.

---

# 53. Message Delivery and Orchestration

A robust implementation may use:

```text id="reliable"
Orchestrator
     |
     v
Command Queue
     |
     v
Participant
```

with:

```text id="reliable2"
Durable messages
Retries
Acknowledgements
Idempotent processing
Dead-letter queues
```

The goal is to make command delivery and processing recoverable.

---

# 54. Idempotent Commands

Suppose the Orchestrator sends:

```text id="cmd-idem"
commandId = CMD123
ReserveInventory
```

Inventory receives it twice.

The service records:

```text id="cmd-state"
CMD123 = PROCESSED
```

The second request becomes:

```text id="cmd-duplicate"
CMD123
   ↓
Already processed
   ↓
Return previous result
```

This prevents duplicate business effects.

---

# 55. Orchestrator Recovery

Suppose:

```text id="recover"
Saga SAGA123

Order       = SUCCESS
Inventory   = SUCCESS
Payment     = PENDING
```

Orchestrator crashes.

After restart:

```text id="recover2"
Read SAGA123
     ↓
Current step = PAYMENT
     ↓
Check payment state
     ↓
Resume workflow
```

This is why persistent Saga state is important.

---

# 56. Recovery Is Not Simply "Start Again"

A dangerous implementation would be:

```text id="bad-recovery"
Orchestrator crashes
       ↓
Restart
       ↓
Start Saga from beginning
```

That could create:

```text id="duplicate"
Create Order AGAIN
Reserve Inventory AGAIN
Charge Payment AGAIN
```

Instead:

```text id="good-recovery"
Read persisted state
       ↓
Determine completed steps
       ↓
Determine current step
       ↓
Continue / compensate
```

---

# 57. Querying Participant State

For uncertain operations, the Orchestrator may need to query the participant.

For example:

```text id="query"
ProcessPayment
     |
 timeout
     |
     v
Payment status = UNKNOWN
```

Instead of immediately retrying:

```text id="query2"
GET /payments/TXN123
```

The Payment Service might respond:

```text id="query3"
TXN123 = SUCCESS
```

The Orchestrator can then safely continue.

This is especially important for operations where duplicate side effects are costly.

---

# 58. Saga Completion

The Saga should have an explicit terminal state.

For success:

```text id="success-terminal"
COMPLETED
```

For failure:

```text id="failure-terminal"
FAILED
```

For compensation:

```text id="comp-terminal"
COMPENSATING
      ↓
COMPENSATED
```

A useful state model might be:

```text id="states"
STARTED
RUNNING
COMPENSATING
COMPLETED
FAILED
COMPENSATED
```

Exact states depend on the implementation.

---

# 59. Saga State Example

A real state record could conceptually look like:

```json id="saga-json"
{
  "sagaId": "SAGA123",
  "type": "OrderSaga",
  "status": "RUNNING",
  "currentStep": "PROCESS_PAYMENT",
  "steps": {
    "createOrder": "SUCCESS",
    "reserveInventory": "SUCCESS",
    "processPayment": "PENDING",
    "createShipment": "NOT_STARTED"
  }
}
```

After failure:

```json id="saga-json2"
{
  "sagaId": "SAGA123",
  "type": "OrderSaga",
  "status": "COMPENSATING",
  "currentStep": "RELEASE_INVENTORY",
  "steps": {
    "createOrder": "SUCCESS",
    "reserveInventory": "SUCCESS",
    "processPayment": "FAILED",
    "createShipment": "NOT_STARTED",
    "releaseInventory": "PENDING"
  }
}
```

This gives the system a durable representation of the workflow.

---

# 60. When Saga Orchestration Is Useful

Orchestration can be useful when:

```text id="useful"
- workflow has many steps
- business process is complex
- failure paths are complicated
- compensation logic is significant
- workflow visibility is important
- centralized monitoring is valuable
- state transitions need to be explicit
- the business process changes frequently
```

The appropriate design depends on the specific workflow.

---

# 61. When Orchestration Can Become Overkill

For a very small event-driven workflow:

```text id="small"
A → B → C
```

a dedicated workflow coordinator may introduce unnecessary infrastructure.

If services can naturally react to events and the workflow remains simple, choreography can be simpler.

The choice depends on:

```text id="choice"
workflow complexity
failure complexity
number of services
operational requirements
consistency requirements
team ownership
```

---

# 62. Real-World Order Saga

A complete orchestration model:

```text id="real-world"
                         User
                           |
                           v
                    Order API
                           |
                           v
                    Orchestrator
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
      Order Service   Inventory Service  Payment Service
          |                |                |
        Order DB       Inventory DB       Payment DB
                           |
                           v
                    Shipping Service
                           |
                       Shipping DB
```

The Orchestrator controls the workflow:

```text id="real-flow"
Create Order
     ↓
Reserve Inventory
     ↓
Process Payment
     ↓
Fraud Check
     ↓
Create Shipment
     ↓
Complete Order
```

Failure:

```text id="real-failure"
Payment Failed
      ↓
Release Inventory
      ↓
Cancel Order
```

Or:

```text id="real-failure2"
Shipping Failed
      ↓
Refund Payment
      ↓
Release Inventory
      ↓
Cancel Order
```

---

# 63. Choreography vs Orchestration — Core Mental Model

The simplest way to remember the difference:

### Choreography

```text id="mental-choreo"
A
 |
 | event
 v
B
 |
 | event
 v
C
```

Each service reacts and decides what to do.

### Orchestration

```text id="mental-orch"
      Orchestrator
       /   |   \
      v    v    v
     A     B     C
```

The Orchestrator decides what happens next.

---

# 64. Interview Answer

If an interviewer asks:

> "What is Saga Orchestration?"

A concise answer:

> **Saga Orchestration is a distributed transaction pattern where a central Orchestrator manages a sequence of local transactions across multiple services. It sends commands to participants, tracks the state of each step, moves the workflow forward after successful operations, and triggers compensating transactions when a later operation fails. Each service still owns its own database and local transaction; the Orchestrator coordinates the overall business workflow rather than executing one global database transaction.**

---

# 65. Interview: What Happens If Payment Fails?

Answer:

```text id="interview-payment"
Order Created       ✓
Inventory Reserved  ✓
Payment             ✗
```

The Orchestrator knows that:

```text id="interview-state"
Order succeeded
Inventory succeeded
Payment failed
```

It can execute compensation:

```text id="interview-comp"
Release Inventory
       ↓
Cancel Order
```

The final business state becomes:

```text id="interview-final"
Order = CANCELLED
Inventory = AVAILABLE
Payment = FAILED
```

---

# 66. Interview: What Happens If the Orchestrator Crashes?

Answer:

> The Saga state should be persisted durably. After restarting, the Orchestrator reads the Saga state, determines which steps completed, identifies the current or failed step, and resumes or compensates from that state instead of starting the entire workflow again.

---

# 67. Interview: Is Orchestration ACID?

No, not in the sense of one global database transaction.

Instead:

```text id="acid"
Each service
    ↓
Local ACID transaction
```

and:

```text
Overall Saga
    ↓
Distributed workflow
    ↓
Eventual consistency
    ↓
Compensation on failure
```

The exact guarantees depend on the implementation.

---

# 68. Interview: Does the Orchestrator Own the Business Data?

No.

For example:

```text id="ownership-final"
Order Service
    → owns Order DB

Inventory Service
    → owns Inventory DB

Payment Service
    → owns Payment DB
```

The Orchestrator owns:

```text id="orchestrator-own"
Workflow state
Coordination
Step transitions
Retry/compensation decisions
```

It should not directly manipulate all service databases.

---

# 69. Interview: Why Use Orchestration?

A strong answer:

> Orchestration makes complex distributed workflows explicit. The workflow, state transitions, retries, failure handling, and compensation logic are centralized in the Orchestrator, which can make complex Sagas easier to understand, monitor, and recover.

---

# 70. Interview: What Are the Risks?

Important risks include:

```text id="risks"
Orchestrator failure
Orchestrator becoming a bottleneck
Centralized workflow complexity
Saga state management
Dual-write problems
Retry handling
Idempotency
Timeouts
Compensation failures
Message delivery failures
```

---

# 71. Final Mental Model

Saga Orchestration can be remembered as:

```text id="final-model"
                 BUSINESS TRANSACTION
                         |
                         v
                  ORCHESTRATOR
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
     Service A       Service B        Service C
        |                |                |
     Local Tx          Local Tx         Local Tx
        |                |                |
        +----------------+----------------+
                         |
                         v
                   Saga State
```

Success:

```text id="final-success"
Command
  ↓
Local Transaction
  ↓
Success
  ↓
Next Command
  ↓
Local Transaction
  ↓
Success
  ↓
COMPLETED
```

Failure:

```text id="final-failure"
Command
  ↓
Local Transaction
  ↓
Failure
  ↓
Orchestrator
  ↓
Compensation
  ↓
Compensation
  ↓
COMPENSATED / FAILED
```

The central principle is:

> **Saga Orchestration coordinates multiple independent local transactions through a central workflow controller. The Orchestrator tracks the Saga state, tells services what operation to perform, handles failures and retries, and triggers compensating transactions when necessary.**

---

# Key Takeaways

```text id="takeaways"
Saga Orchestration
=
Central Workflow Controller
+
Local Transactions
+
Commands
+
Saga State
+
Failure Handling
+
Compensation
```

Remember these distinctions:

```text id="distinctions"
Orchestrator
→ controls workflow

Participant
→ executes local business transaction

Command
→ tells a service to perform an operation

Event
→ tells consumers that something happened

Saga State
→ tracks workflow progress

Compensation
→ business operation that reverses the effect of a previous operation

Rollback
→ database transaction mechanism

Idempotency
→ prevents retries from creating duplicate effects
```

The most important difference from choreography:

```text id="final-difference"
Choreography:

Service A
   ↓ event
Service B
   ↓ event
Service C


Orchestration:

          Orchestrator
          /    |    \
         v     v     v
        A      B      C
```

**Choreography distributes workflow decisions among services through events. Orchestration centralizes workflow decisions in an Orchestrator while keeping each service's data and local transactions independent.**