### What is 2PC?

2PC stands for **Two-Phase Commit**, a protocol used in distributed systems to ensure atomicity across multiple participants (e.g., databases or services) in a transaction. It's a way to coordinate commits so that either all changes succeed or none do, maintaining consistency (part of ACID properties).

#### How 2PC Works

1. **Prepare Phase (Voting)**:
    - A coordinator (e.g., transaction manager) asks all participants (e.g., databases) if they can commit the transaction.
    - Each participant performs the work locally (e.g., locks resources, writes to logs) but doesn't commit yet.
    - They vote "Yes" (ready to commit) or "No" (abort).
2. **Commit Phase (Decision)**:
    - If all vote "Yes", the coordinator sends "Commit" to all → everyone commits.
    - If any votes "No" (or times out), the coordinator sends "Abort" → everyone rolls back.

#### Example

In a banking app transferring money between two accounts on different databases:

- Coordinator: "Prepare to deduct $100 from Account A" (DB1) and "Prepare to add $100 to Account B" (DB2).
- Both say "Yes" → Commit both.
- If DB2 fails (e.g., insufficient funds check), it says "No" → Abort both, no partial transfer.

#### Pros

- Strong consistency: All or nothing.
- Atomicity in distributed setups.

#### Cons

- Blocking: Participants hold locks during prepare, reducing availability if coordinator fails.
- Single point of failure: Coordinator downtime can stall the system.
- Scalability issues: Doesn't work well in microservices with high latency or failures (e.g., networks).
- Not fault-tolerant: If a participant crashes after voting "Yes" but before commit, recovery is complex.

In microservices, 2PC is often avoided due to these issues. Instead, patterns like **Saga** (which we've discussed) use eventual consistency and compensations for better resilience and scalability. Saga trades strong consistency for availability, making it suitable for cloud/distributed environments where failures are common.

If this isn't what you meant by "2PC," feel free to clarify!

---

### Detailed Documentation: Resilient Backend Patterns for Distributed Systems

This document summarizes and expands on the patterns we've discussed in our conversation, focusing on building fault-resilient backend systems, especially in microservices and event-driven architectures. We've covered the **Outbox Pattern**, **Saga Pattern**, **Event Sourcing**, **CQRS (Command Query Responsibility Segregation)**, and the related **Inbox Pattern**. These are commonly used to handle reliability, consistency, and scalability in systems where traditional ACID transactions (like 2PC) fall short.

The document is structured with definitions, purposes, mechanisms, examples, pros/cons, implementation tips, and relations to other patterns. It's based on industry best practices (e.g., from Domain-Driven Design, microservices patterns by Chris Richardson, and tools like Kafka, Debezium).

#### 1. Transactional Outbox Pattern

**Definition**: A reliability pattern that ensures atomicity when performing a local database write and publishing an event/message to an external system (e.g., message broker). It solves the "dual-write problem" where one operation might succeed while the other fails, leading to inconsistencies.

**Purpose**:

- Guarantee at-least-once delivery of events without risking partial failures.
- Decouple the business transaction from unreliable network calls to brokers.

**Mechanism**:

- During a business operation, write the event payload to an "outbox" table in the same database transaction as the main entity.
- A separate process (poller/relay) scans the outbox, publishes events to the broker (e.g., Kafka, RabbitMQ), and marks them as processed (or deletes them).
- Alternatives: Use Change Data Capture (CDC) tools like Debezium to detect outbox inserts via the DB's transaction log, avoiding polling.

**Detailed Example** (E-commerce Order Service in Java/Spring Boot):

- Business flow: Create an order and notify downstream services.

Java

```
import org.springframework.transaction.annotation.Transactional;
import com.fasterxml.jackson.databind.ObjectMapper;

@Entity
public class OutboxEvent {
    @Id private UUID id;
    private String eventType;
    private String payload;
    private boolean sent;
    // Timestamps, etc.
}

@Service
public class OrderService {
    @Autowired private OrderRepository orderRepo;
    @Autowired private OutboxRepository outboxRepo;
    @Autowired private ObjectMapper jsonMapper;

    @Transactional
    public void createOrder(OrderRequest req) {
        Order order = new Order(req);
        orderRepo.save(order);  // Main DB write

        // Outbox write in same TX
        String payload = jsonMapper.writeValueAsString(new OrderCreatedEvent(order.getId(), order.getDetails()));
        OutboxEvent event = new OutboxEvent(UUID.randomUUID(), "OrderCreated", payload, false);
        outboxRepo.save(event);
    }
}
```

- Poller (e.g., scheduled job):

Java

```
@Scheduled(fixedRate = 5000)  // Every 5s
public void processOutbox() {
    List<OutboxEvent> pending = outboxRepo.findBySentFalse();
    for (OutboxEvent e : pending) {
        try {
            kafkaTemplate.send("orders-topic", e.getPayload());  // Publish
            e.setSent(true);
            outboxRepo.save(e);
        } catch (Exception ex) {
            // Log and retry later (exponential backoff)
        }
    }
}
```

- If the service crashes after DB commit but before publish, the poller retries on restart.

**Pros**:

- Atomicity via DB transaction.
- Handles broker downtime gracefully.
- Easy to implement with existing DBs.

**Cons**:

- Polling adds latency/overhead (mitigated by CDC).
- Duplicate events possible (handle with Inbox Pattern).
- DB bloat if outbox isn't pruned.

**Implementation Tips**:

- Use idempotency keys in events.
- Scale poller horizontally.
- Tools: Debezium for CDC, Spring Boot for transactions.

**Relations**: Often used with Saga (to publish saga events) and Inbox (to consume reliably).

#### 2. Saga Pattern

**Definition**: A coordination pattern for managing long-running, distributed transactions across microservices without locking or 2PC. It uses a series of local transactions, with compensations for failures.

**Purpose**:

- Achieve eventual consistency in distributed systems.
- Avoid blocking and single points of failure in 2PC.

**Mechanism**:

- Break a transaction into steps (saga steps), each a local TX in a service.
- If a step fails, invoke compensating actions (undo) in reverse order.
- Two variants:
    - **Choreography**: Decentralized—services react to events from others.
    - **Orchestration**: Centralized—a coordinator directs steps.

**Detailed Example** (Travel Booking: Book Flight + Hotel + Car; choreography style in Node.js with Kafka):

- Services: Booking, Flight, Hotel, Car.
- Flow:
    1. Booking Service: Create booking → publish BookingStarted event.
    2. Flight Service: Listen to BookingStarted → reserve seat (local TX) → publish FlightReserved or FlightFailed.
    3. Hotel Service: Listen to FlightReserved → book room → publish HotelBooked or HotelFailed.
    4. Car Service: Listen to HotelBooked → rent car → publish CarRented or CarFailed.
    5. If any failure (e.g., HotelFailed): Previous services listen and compensate (e.g., Flight cancels reservation).
- Compensation example (Flight Service):

JavaScript

```
kafka.on('HotelFailed', async (event) => {
  const bookingId = event.bookingId;
  // Compensate: Release seat
  await db.transaction(async (tx) => {
    await tx.query('UPDATE seats SET reserved = false WHERE booking_id = ?', [bookingId]);
  });
  kafka.publish('FlightCompensationDone', { bookingId });
});
```

- For orchestration: Use a tool like Temporal.io to define workflow:

JavaScript

```
async function bookingSaga(bookingId) {
  try {
    await flightWorkflow.reserve(bookingId);
    await hotelWorkflow.book(bookingId);
    await carWorkflow.rent(bookingId);
  } catch (err) {
    // Compensate in reverse
    await carWorkflow.cancel(bookingId);
    await hotelWorkflow.cancel(bookingId);
    await flightWorkflow.cancel(bookingId);
    throw err;
  }
}
```

**Pros**:

- High availability (no blocking).
- Scalable for microservices.
- Flexible for complex flows.

**Cons**:

- Eventual consistency (not immediate).
- Complex error handling/debugging (use tracing like Zipkin).
- Requires idempotent operations.

**Implementation Tips**:

- Make steps idempotent (e.g., check if already done).
- Use correlation IDs for tracing.
- Tools: Apache Camel, Axon Framework, or serverless like AWS Step Functions.

**Relations**: Relies on Outbox for reliable event publishing; often combined with Event Sourcing for state management.

#### 3. Event Sourcing

**Definition**: A persistence pattern where state is stored as an immutable sequence of events rather than mutable records. Current state is derived by replaying events.

**Purpose**:

- Provide a full audit trail and history.
- Enable temporal queries (e.g., "What was the state at time X?").

**Mechanism**:

- Append events to an event store (e.g., EventStoreDB, Kafka).
- To get state: Replay events from start (or use snapshots for optimization).
- Project events into read models for queries.

**Detailed Example** (Inventory Management in Python with SQLAlchemy):

- Events: ItemAdded, ItemRemoved, StockUpdated.
- Event store table: events (id, aggregate_id, type, data, timestamp).
- Adding stock:

Python

```
class InventoryAggregate:
    def __init__(self):
        self.stock = 0

    def apply(self, event):
        if event.type == 'ItemAdded':
            self.stock += event.data['quantity']

# Persist
def add_item(item_id, quantity):
    event = Event(aggregate_id=item_id, type='ItemAdded', data={'quantity': quantity})
    db.session.add(event)
    db.session.commit()

# Rebuild state
def get_stock(item_id):
    events = db.query(Event).filter_by(aggregate_id=item_id).order_by('timestamp').all()
    aggregate = InventoryAggregate()
    for e in events:
        aggregate.apply(e)
    return aggregate.stock
```

- Optimization: Snapshot every 100 events to avoid full replay.

**Pros**:

- Immutable history for auditing/compliance.
- Easy undo/redo.
- Integrates with event-driven systems.

**Cons**:

- Query complexity (needs projections).
- Storage growth (events are append-only).
- Performance on large histories (use snapshots).

**Implementation Tips**:

- Use GUIDs for event IDs.
- Tools: EventStoreDB, Kafka Streams for projections.

**Relations**: Pairs with CQRS (events for writes, projections for reads); used in Sagas for step tracking.

#### 4. CQRS (Command Query Responsibility Segregation)

**Definition**: Separates the write (command) model from the read (query) model, allowing independent scaling and optimization.

**Purpose**:

- Optimize for different workloads (writes: consistency; reads: speed/denormalization).
- Simplify complex domains.

**Mechanism**:

- Commands: Mutate state (e.g., via events in Event Sourcing).
- Queries: Read from a separate, optimized store (e.g., denormalized views).
- Sync via event projections.

**Detailed Example** (User Profile Service in .NET):

- Write side: Command UpdateProfile → append event ProfileUpdated.
- Projector listens: Updates read DB (e.g., MongoDB for fast queries).

C#

```
// Command Handler
public class UpdateProfileHandler {
    public async Task Handle(UpdateProfileCommand cmd) {
        var events = await eventStore.Load(cmd.UserId);
        var aggregate = ProfileAggregate.FromEvents(events);
        aggregate.Update(cmd.NewName, cmd.NewEmail);
        await eventStore.Append(aggregate.PendingEvents);
    }
}

// Projector
public class ProfileProjector {
    public void On(ProfileUpdatedEvent evt) {
        var doc = new ProfileDocument { Id = evt.UserId, Name = evt.NewName, Email = evt.NewEmail };
        mongoCollection.ReplaceOne(d => d.Id == evt.UserId, doc, new ReplaceOptions { IsUpsert = true });
    }
}

// Query
public ProfileDto GetProfile(Guid userId) {
    return mongoCollection.Find(d => d.Id == userId).FirstOrDefault();
}
```

**Pros**:

- Scalable (scale reads independently).
- Flexible models (e.g., relational for writes, NoSQL for reads).

**Cons**:

- Added complexity (two models to maintain).
- Eventual consistency for reads.

**Implementation Tips**:

- Use message buses for syncing.
- Tools: MediatR (.NET), Axon (Java).

**Relations**: Often with Event Sourcing (commands append events; queries use projections); complements Saga for read-after-write needs.

#### 5. Inbox Pattern (Bonus: Closely Related as Consumer-Side Counterpart)

**Definition**: Ensures idempotent processing of incoming messages/events, handling duplicates from at-least-once delivery.

**Purpose**:

- Prevent side effects from retries/redeliveries.

**Mechanism**:

- On receive: Check if message ID in "inbox" table.
- If exists, skip; else, insert and process.

**Detailed Example** (Go with PostgreSQL):

Go

```
func handleEvent(event Event) error {
	tx, err := db.Begin()
	if err != nil { return err }
	defer tx.Rollback()

	var exists bool
	err = tx.QueryRow("SELECT EXISTS(SELECT 1 FROM inbox WHERE msg_id = $1)", event.MsgID).Scan(&exists)
	if err != nil { return err }
	if exists { return nil } // Idempotent skip

	_, err = tx.Exec("INSERT INTO inbox (msg_id, processed_at) VALUES ($1, NOW())", event.MsgID)
	if err != nil { return err }

	// Business logic
	processBusinessLogic(event)

	return tx.Commit()
}
```

**Pros/Cons**: Simple reliability; but adds DB overhead.

**Relations**: Pairs with Outbox for end-to-end reliability.