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