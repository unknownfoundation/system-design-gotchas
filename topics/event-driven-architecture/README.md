# Event-driven architecture

Prev: [Message queues and event streams](../message-queues-and-event-streams/README.md) · Next: [WebSockets and real-time delivery](../websockets-and-real-time-delivery/README.md) · Reviewed October 2026

In an event-driven design a service records that something happened and publishes it as an event, and other services react in their own time, with a broker carrying the event between them. It decides what a sender knows about the work that follows, how a change in one store reaches the others, and what stands in for a transaction that spans services.

- **Event**: a record that something has already happened, named in the past tense, with any number of readers, or none.
- **Command**: a request to one service to do something, which that service can refuse.
- **Thin event**: an event that holds identifiers and little else, so a consumer gets the rest from the service that owns it.
- **Event-carried state**: an event that holds the changed data, so a consumer can keep its own copy.
- **Saga**: a business transaction across services, run as a series of local transactions, one per service.
- **Compensation**: a new action that makes up for a saga step already committed, such as a refund for a charge.
- **Orchestration**: a saga run by one service, the orchestrator, which keeps each saga's state in a store and sends each step's command.
- **Choreography**: a saga run with no such service, where each one reacts to events from the others and publishes its own.
- **Schema**: the agreed shape of an event. A change is backward compatible when new readers can read old events, and forward compatible when old readers can read new events.
- **Event sourcing**: keeping the events as the record of truth, and building current state by folding them: applying each event in order to the result so far.
- **Snapshot**: the saved result of folding a stream up to some event, so that a load can start there.
- **Read model**: a table built from events for queries; separating it from the write side is CQRS, and CQRS and event sourcing each exist without the other.

How a broker delivers, orders and retries events, and what its outage does, are on [Message queues and event streams](../message-queues-and-event-streams/README.md). In short, a consumer is a service that reads events, a log keeps them after they are read, and an offset is a consumer's place in the log. A sender's publish ends at the broker, so a consumer that is down holds up its own work for as long as the broker has room, and its readers see the old state in the meantime. A step whose answer the caller must show now stays a call, or a request over a queue with a reply queue and an identifier that ties the two together.

A change in a store usually reaches other services in one of these ways:

| Way | What it does |
| --- | --- |
| Dual write | writes the store, then publishes the event, as two separate steps |
| Outbox | writes the event as a row in the same transaction as the change, and a relay process publishes the rows later |
| Change data capture | reads the store's change log, its own ordered record of writes, and publishes the changes |
| Polling a table | runs a query at intervals for rows changed since the last run |

Two-phase commit, a protocol that makes two systems commit together or not at all, can tie a store and a broker where both support it, at a cost [Consistency, consensus and clocks](../consistency-consensus-and-clocks/README.md) takes up. A relay that polls by the highest id it has sent can skip a row that commits late, and two relays on one outbox can publish one order's events out of order. Some stores keep their change log for a set time, and others until a registered reader has read it. An event can record the new value or the intent behind it, and one meant for other services is often called an integration event, as against a domain event that stays inside a service. Several stores can also stay consistent by all deriving from one append-only log, and a service can update its own store only from the events it publishes.

A saga has a pivot step, after which it can no longer be undone and later steps can only be retried. Two services that react to each other's events can keep each other going with no error anywhere. A flow that stops with no failure event is found by reconciliation, a job that compares what services hold. An event read late can still set off an action that no longer makes sense. An event store can refuse an append when the stream has moved past the version the writer read, and a broker's log usually has no such check. A read model's rebuild walks through old states, so it is built beside the live table and swapped in.

Most of the trouble sits in the gaps between one service's commit and the next service's reaction, where no transaction reaches.

Unless a question says otherwise, everything here runs at its default settings, and nobody has added a retry, check, limit, job or tool that the question does not name.

This topic lets you:

- work out what a sender, a consumer and a customer each know after an order event, and what a copy built from two streams ends up holding
- get a change out of a store and into events through a crash, a late restart, a migration or a delete
- run a business transaction across services, and say what a timeout, a lost orchestrator or one more subscriber does to it
- change an event's shape, fix a bug in an event-sourced system, erase a customer's address, rebuild a table or decide from a read model, and say what each does to the history and to the customers

###### 1. A warehouse with no stock for a placed order

A furniture shop runs an order service that saves each order, shows the customer a confirmation and publishes `OrderPlaced`. The warehouse service subscribes to that event and reserves the stock. Tonight the shop floor sells the last sofa a moment before an online customer's `OrderPlaced` for the same sofa reaches the warehouse.

What happens to the online customer's order?

<details><summary>A. It is cancelled when the warehouse turns the event down</summary>

✗ In a chain of service calls, a refusal further down fails the whole request. The warehouse is in no such chain: the order was saved and confirmed before it read the event, and a cancel would take a message of its own, such as a `StockUnavailable` event that the order service reads.

</details>

<details><summary>B. It goes back to pending while the order service waits to hear from the warehouse</summary>

✗ Order screens often show a pending stage, and a shop would want one here. The confirmation came before the publish, so this order has already gone past that stage, and the order service has no reason to expect word from anyone.

</details>

<details><summary>C. It stays confirmed, with no sofa reserved</summary>

✓ Right. The order stays confirmed and nothing is reserved for it. The customer is expecting a sofa the warehouse cannot send, and the order service does not know.

The warehouse read `OrderPlaced` after the order was confirmed, and **an event cannot be refused, because it states what has already happened**. Whatever the warehouse's handler does next, whether it logs a warning or fails and gets the event again, stays on the warehouse's side of the broker. The order service does not know who reads `OrderPlaced`, or whether anyone does.

Decide which you need. If nobody may be told yes before stock is held, reserve the stock first with a command, `ReserveStock`, whose reply can be a no. If a late no is acceptable, have the warehouse publish `StockUnavailable`, and have the order service react with an apology and a refund.

</details>

<details><summary>D. It is rolled back by an error from the publish call</summary>

✗ A remote call hands the callee's failure back to the caller as an error. A publish only hands the event towards the broker, and the warehouse reads it after that, so by the time the sofa is missed no call is open to carry an error back.

</details>

---

###### 2. Writing invoices from a thin order event

A firm that hires out tools has an order service that confirms each order to the customer with its price and publishes `OrderPlaced`, a thin event holding the order id and the product id. The invoice service reads these events, looks up each product in the catalogue service and writes an invoice for each one. On Tuesday evening the invoice service is down for maintenance. At midnight the catalogue service puts up the price of a drill, and at 6 a.m. the invoice service comes back and picks up the evening's events.

What price do Tuesday evening's drill invoices show?

<details><summary>A. Tuesday's price, matching the customer's confirmation</summary>

✗ The event was published on Tuesday, and an event is taught as the record of a moment. `OrderPlaced` has no price in it, only two ids, so the price comes from the catalogue when the invoice service asks, and at 6 a.m. the catalogue holds the new one.

</details>

<details><summary>B. Wednesday's price, above the customer's confirmation</summary>

✓ Right. Each of those invoices shows the raised price, so customers who were told Tuesday's price are billed more than their confirmation says.

A thin event holds identifiers, so **everything else is read when the consumer handles the event**, whenever that turns out to be.

Put what has to stay as it was into the event, which makes it event-carried state: the price the customer agreed belongs in `OrderPlaced`. Keep thin events for what a consumer should read fresh, such as a customer's current address.

</details>

<details><summary>C. No invoices, with the evening's events expired overnight</summary>

✗ Some brokers let a message carry an expiry time, and these sat unread all night. An expiry is something a team sets on a message that goes stale within moments, such as a live quote, and an order is still wanted the next morning. The events are there at 6 a.m., and each one gets its invoice.

</details>

---

###### 3. Keeping an order's status from two event streams

A marketplace's support app keeps its own copy of each order's status, so agents need not call other services. The payment service publishes `OrderPaid` to one log and the shipping service publishes `OrderShipped` to another, both keyed by order id, and the broker keeps order within each key. One consumer per log runs this update.

```sql
UPDATE order_view SET status = :status WHERE order_id = :order_id;
```

This morning the payment log's consumer restarts after a bad release, an hour behind, while shipping's stays current. A customer's order was paid at 09:00 and shipped at 09:05.

What status does the agent see once both consumers have handled this order?

<details><summary>A. <code>Paid</code>, for a parcel already on its way</summary>

✓ Right. The agent sees `Paid` and tells the customer the parcel has not left, though it left at 09:05.

The shipping consumer writes `Shipped`, and then the payment consumer reaches `OrderPaid` in its backlog and writes `Paid` over it. The update writes whatever arrives last. Order within a key holds inside one log, and **between two logs a broker promises nothing**.

Make the order of arrival stop mattering. Keep `paid_at` and `shipped_at` as separate columns, so that each event writes its own, or keep a rank column beside the status and update only when the rank goes up: `WHERE status_rank < :rank`. Timestamps from two services' clocks are a weaker guard than either.

</details>

<details><summary>B. <code>Shipped</code>, as the later of the two facts</summary>

✗ Both events carry the order id as their key, and a key is how you get order from a broker. That order holds inside one log, and these are two, each with its own consumer. The one behind does its update last, so `Paid` replaces `Shipped`.

</details>

<details><summary>C. <code>Shipped</code>, with the late update logged as a conflict</summary>

✗ Order status diagrams only draw forward moves, and a store can enforce them with a check on the transition. The `UPDATE` here sets the column to what the event says, and the store has no idea that paid comes before shipped.

</details>

<details><summary>D. <code>Paid</code> for a moment, then <code>Shipped</code> again</summary>

✗ Eventual consistency is taught as every copy reaching the same data in the end. A copy ends up where its last update leaves it, and no further event about this order is on its way to correct it, so `Paid` stays.

</details>

---

###### 4. Saving the order, then publishing with retries

You build the checkout of a bicycle shop: it saves the order and then tells the warehouse. `db` and `queue` are thin clients around the store and the broker. You wrapped the publish in retries because the broker sometimes times out, and in months of production no timed-out publish has lost an event.

```python
def checkout(order):
    with db.transaction():
        db.insert("orders", order)
    for attempt in range(3):
        try:
            queue.publish({"type": "OrderPlaced", "id": order.id})
            return
        except TimeoutError:
            continue
    raise PublishFailed(order.id)
```

Today's deploy kills copies of the service that are in the middle of a checkout.

What can a killed checkout leave behind?

<details><summary>A. An order row, with <code>PublishFailed</code> in the logs for someone to act on</summary>

✗ `raise PublishFailed` is the last line of the function, and you wrote it for the day a publish goes wrong, after three timeouts. A killed process runs no more lines, so a copy killed after the commit leaves no error and no log line behind.

</details>

<details><summary>B. Nothing out of step, with an interrupted checkout rolled back whole</summary>

✗ Web frameworks often wrap a whole request in one transaction, and there a kill does undo the request's writes. The transaction block closes before the loop, so by the time the publish runs the row is committed, and a kill cannot take it back.

</details>

<details><summary>C. Nothing out of step, with the restarted copy finishing the publish</summary>

✗ The deploy starts a new copy at once, and job runners do rerun what a dead worker left. The loop and its `order` lived in the memory of the killed process. The new copy has never heard of them and waits for new requests.

</details>

<details><summary>D. An order row that the warehouse never hears about</summary>

✓ Right. A copy killed after the commit and before the publish leaves the order in the store and no event on the broker. The customer's bike is never packed.

The row and the event form a dual write: **two writes to two systems, with no transaction across them**. The retries cover a slow broker. They cannot cover the process dying, and the store holds no note that an event is owed, so nothing ever sends it. Only a deploy or a crash lands in that short window, which is why months of production showed no loss.

Write the event as an outbox row in the same transaction as the order, and let a relay publish it. Where that is not possible, run a reconciliation job that compares orders with what the warehouse has.

</details>

---

###### 5. Publishing inside the transaction block

On a site that sells concert tickets, the checkout publishes from inside its transaction, so that the row and the event go together. The email service subscribes and sends a confirmation to the address in each `OrderPlaced`. `db` and `queue` are thin clients around the store and the broker.

```python
def checkout(order):
    with db.transaction():
        db.insert("orders", order)
        queue.publish({"type": "OrderPlaced", "id": order.id, "email": order.email})
        db.update("tickets", order.show, -order.quantity)
```

Tonight two fans go for the last ticket to a show, and the update for the second one breaks a constraint on the `tickets` table, so that transaction rolls back.

What does the second fan find in their inbox?

<details><summary>A. No email, with the event gone along with the order row</summary>

✗ Inside the transaction block every write goes or stays together, and the publish sits among them in the code. The broker is another system, and the store's transaction has no hold on it: a rollback cannot call back an event the publish has handed over, and the email service sends from the address the event carries.

</details>

<details><summary>B. A confirmation, followed by a cancellation</summary>

✗ Saga recipes send out an undo when a step fails, and a rollback is a failure of a kind. A rollback is the store's business: it undoes the store's writes and tells nobody. No line of this code sends a cancellation, so the confirmation stands alone.

</details>

<details><summary>C. A confirmation for an order the store does not hold</summary>

✓ Right. The second fan gets a confirmation for a ticket they do not have, since the order row is gone and the event is not.

The publish has already handed the event to the broker when the update breaks the constraint, and **a store's own transaction covers the store and nothing else**. Moving the publish inside the block changes the order of the two writes and leaves them as two. Some brokers can join a two-phase commit with some stores, which takes setup that a plain publish call does not have.

Inside the transaction, write an outbox row. Publish from outside it, through a relay that reads committed rows.

</details>

---

###### 6. Restarting the outbox relay after a publish

At a pharmacy chain, the order service writes each order and an outbox row in one transaction, after losing events to crashes the year before. A relay process reads unsent rows from the outbox, publishes each one to the broker and marks it sent, logging each step. The prescription service subscribes to these events. Last night the relay's machine lost power, and its log ended like this.

```text
02:14:07  relay  read outbox row 8841 (OrderPaid)
02:14:07  relay  publish row 8841: ok
02:14:52  relay  started
```

What does the relay do with row 8841 when it starts again?

<details><summary>A. It publishes the row again</summary>

✓ Right. The relay publishes row 8841 a second time, and the prescription service gets `OrderPaid` twice for one payment.

The power went between the publish and the mark, and the store still shows the row as unsent, which is all the relay has to go on. **Publishing a row and marking it are themselves two writes to two systems**, and the outbox leaves you to choose the failure: mark after publishing and a crash repeats an event, mark before publishing and a crash loses one.

Treat the outbox as delivering at least once: an event may arrive more than once. Send the row's id as the event's id, and make the prescription service's handler recognise an id it has already handled.

</details>

<details><summary>B. It skips the row, which the broker has already taken</summary>

✗ The log says `ok`, and that is what you read the next morning. The relay's knowledge of that `ok` was in memory, and the power cut took it. What survives is row 8841 still unsent, and unsent rows are what the relay reads when it starts.

</details>

<details><summary>C. It asks the broker whether it holds the row, and carries on from the answer</summary>

✗ Recovery in a two-phase commit does work by asking the other side what it decided. The relay's whole job is read, publish and mark. Asking would take a look-up by row id on the broker and code to use it, and the loop here publishes what it finds unsent.

</details>

<details><summary>D. It parks the row for someone to check by hand</summary>

✗ A message in doubt often ends up in a dead-letter queue for a person to check. Row 8841 is not in doubt as far as the relay can tell: an unsent row looks the same whether it never went out or went out a moment before the power cut.

</details>

---

###### 7. Migrating a table that change data capture reads

A payments team keeps its store private: no other team connects to it. To avoid writing events by hand it turned on change data capture for the `invoices` table, which sends each changed row with its current columns as JSON, and the ledger team's service consumes that stream. This sprint the payments team replaces the column `amount` with `amount_minor` and `currency`: its migration adds the columns, fills them in on all four million rows in batches and drops the old one. All its own tests pass, and it tells nobody, since the table is private and no invoice has changed.

What does the ledger service get on the stream during the migration?

<details><summary>A. Nothing, with new invoices carrying <code>amount</code> as before</summary>

✗ A published event is a contract, and `amount` is the field the ledger team has always had. The stream was never written as a contract: it carries what is in the log, and the migration's updates are in the log like any other writes.

</details>

<details><summary>B. Nothing, with capture halted until the ledger team upgrades</summary>

✗ Schema registries exist to refuse breaking changes, and a capture tool can be set up behind one. JSON with no check refuses nothing, and a store's change stream carries on through a table change. So nothing stops, and nobody on the ledger team is asked.

</details>

<details><summary>C. One notice of the table change, and then new invoices in the new shape</summary>

✗ A schema change is one statement, and capture tools can emit a record for it. Filling the new columns is no schema change: it is an update to each row, each one committed and each one in the log.

</details>

<details><summary>D. A change event for every existing invoice, in the new shape</summary>

✓ Right. The ledger service gets about four million change events for invoices nobody has touched, with fields it has never seen, and no `amount` once the column goes.

Change data capture publishes each committed row change as the row now stands, and **a store's log records a migration's updates like any other write**. The table's columns are the event's fields, and the private table became a public interface on the day capture was turned on. One store's own capture is built differently: it keeps the columns it started with until someone creates a new capture, so new columns stay off its stream and a dropped one arrives empty. Some stores let a session keep its writes off that stream, and that takes someone asking for it.

Publish events shaped for readers: write them to an outbox table and point capture at that table, so `invoices` can change freely. Where capture stays on the table itself, treat each migration as a release that its consumers hear about first.

</details>

---

###### 8. Restarting a change data capture reader after a week

At a logistics firm, the orders store keeps three days of its change log, and a change data capture reader feeds an analytics warehouse from that log, saving its position as it goes. The orders team's load tests showed no extra load from the reader, so no alert watches it. On a Friday the reader's credential expires and it stops. The next Friday a data analyst asks why the warehouse looks a week old, and an engineer renews the credential and starts the reader.

What happens when the reader starts again?

<details><summary>A. It carries on from its saved position and reads the whole week</summary>

✗ A consumer on a broker resumes from its saved offset after any restart, and this reader saved its position too. The position only points into the log, and the store kept three days: the first four days of that week have been removed, so it points at nothing.

</details>

<details><summary>B. It stops with an error, with most of the week out of its reach</summary>

✓ Right. The reader stops and reports that the position it saved is no longer in the log. The analyst's warehouse stays a week old, and the log can no longer give four of those days.

The store keeps its log for its own needs, and **retention by age does not wait for a reader that has stopped**. Capture tools stop here by default, and a store's own change feed fails the same way, since jumping ahead would leave a hole in the data.

Copy the whole table again and read the log from the point the copy was taken, since a log alone typically holds too little to rebuild it. Alert on a stopped reader, and keep the log for longer than a stopped reader could go unnoticed.

</details>

<details><summary>C. It starts from the newest entry, and the warehouse looks current</summary>

✗ Some broker clients jump to the newest message when their offset has gone, and a tool can be set up to recover by itself. A capture tool's default is to stop and report it, since a silent jump would leave the warehouse missing four days.

</details>

<details><summary>D. It finds the store's disk full, with new orders being refused</summary>

✗ A log held for a registered reader does grow while that reader is away, and on a store that works that way a full disk is the failure to fear. The orders store goes by age and keeps three days whoever is reading, so the log stays its usual size and orders carry on.

</details>

---

###### 9. Feeding a search index from `updated_at`

A classifieds site keeps its search index in step with its `listings` table by a job that runs every minute. A trigger sets `updated_at` on every write, and the job has kept up with every new listing and every price edit for a year.

```sql
SELECT id, title, price
FROM listings
WHERE updated_at > :last_run;
```

A seller finds a buyer for their bike and deletes the listing, and the site removes the row.

What does a search for that bike return an hour later?

<details><summary>A. No result, with the entry dropped when a run does not return it</summary>

✗ A year of edits have reached the index within a minute, and a mirror drops whatever has gone from its source. A run returns only rows changed since the last one, so a missing id tells the job nothing: if absence removed entries, each run would empty the index of every listing nobody edited that minute.

</details>

<details><summary>B. The listing, flagged as removed</summary>

✗ A change log reports a delete as an entry of its own, and a table that marks rows as deleted keeps the row with a flag and a new `updated_at`. The site removes the row, the trigger has no row left to stamp and the query has nothing to return, so the index never hears of it.

</details>

<details><summary>C. The listing, as it was before</summary>

✓ Right. The search still returns the bike with its price, a listing that no longer exists.

**Only rows that exist can come back from a query**, so a job built on one sees inserts and updates and never a delete. The edits of the past year all had a row to find.

Have the site mark a deleted listing and stamp it, so that the job copies the mark, and remove the row later. Or read the store's change log, where a delete is an entry of its own, and compare the index with the table from time to time to catch what has already been missed.

</details>

<details><summary>D. The listing, with an alert from the job about a missing row</summary>

✗ Sync tools compare two sides and report what has gone missing, and this job is your sync tool. It reads one result set from the table and never looks at what the index holds, so the missing row gives it nothing to complain about.

</details>

---

###### 10. Booking the last cabin during a saga

A ferry company books a crossing as a saga: the cabin service reserves a cabin, the payment service charges the card, and the deck service books space for the car. One customer has taken the last cabin on Friday's crossing, and the deck step is still running. Right then a second customer tries to book that cabin. A few seconds later the deck turns out to be full, and the saga gives the first customer's cabin back.

What happens to the second customer?

<details><summary>A. They are told it is sold out</summary>

✓ Right. The second customer is told Friday's crossing has no cabin, and seconds later that cabin is back on sale.

A saga has no isolation, because **each step is a local transaction that commits on its own**. The cabin's reservation counts for every other booking from the moment the cabin service commits it, and the compensation that gives the cabin back is a later, separate write. The second customer has read a booking that never finished, which is often called a dirty read.

Have the first step mark the cabin as held, so the page can say to try again in a minute. Put the steps most likely to fail first, so that fewer bookings hold something they will give back.

</details>

<details><summary>B. They wait a few seconds, and then they get the cabin</summary>

✗ In one database, a row in the middle of a transaction can make the next writer wait for the commit, and a saga is often called a distributed transaction. The cabin service's transaction has already finished, and no lock lasts across the steps, so there is nothing to wait on and the answer comes at once.

</details>

<details><summary>C. They book it ahead of the first customer, who has not finished</summary>

✗ A single store shows a write to others only after its commit, and the first customer's booking has not finished. A saga has one commit per step, and the cabin service's has happened, so that cabin is taken as far as any other booking can tell.

</details>

<details><summary>D. They book it, and they get an error at the end over the clash</summary>

✗ Under optimistic concurrency two writers on one row get found out at commit. A saga has no last commit that could check them both: the cabin service answers from its row right away, and the row says taken.

</details>

---

###### 11. Compensating a hotel booking that timed out

On a travel site, an orchestrator books a flight and then a hotel room, and a step with no reply within ten seconds counts as failed. The hotel service takes commands from a queue with a pool of workers: `ReserveRoom` adds a reservation with no expiry, and `CancelRoom` deletes the trip's reservation. This morning those workers are slow, and `ReserveRoom` for one trip gets no reply in time. The orchestrator sends `CancelFlight` and `CancelRoom`, each is answered as done, and it tells the customer the trip could not be booked.

What can the hotel still hold for that trip an hour later?

<details><summary>A. No room, whichever order the two commands ran in</summary>

✗ `CancelRoom` was answered as done, and the orchestrator has closed the trip. Done can mean that it found nothing to cancel, because `ReserveRoom` had not yet run. A worker then runs `ReserveRoom`, and nothing on record tells it that the trip is off.

</details>

<details><summary>B. A room that nobody will pay for</summary>

✓ Right. If `ReserveRoom` runs after `CancelRoom`, the hotel holds a room for a trip that no longer exists, and it sits unsold with nobody paying.

**All a timeout tells you is that no reply came**, and the command can still be waiting in the queue or running on a worker. A compensation that finds nothing to undo leaves no trace, so a late `ReserveRoom` looks like any other. In the other order, `CancelRoom` finds the reservation and removes it, and no room is left.

Have `CancelRoom` leave a record for the trip even when it finds nothing, and have `ReserveRoom` refuse a trip with such a record. Give an unconfirmed reservation an expiry as well, and run a reconciliation job between the orchestrator's trips and the hotel's rooms.

</details>

<details><summary>C. A room, with the flight booked again around it</summary>

✗ An orchestrator follows the replies it gets, and a late yes from the hotel does come back. The saga has already compensated and closed, so the reply finds no step waiting for it, and the cancelled flight stays cancelled.

</details>

<details><summary>D. A room, with the customer told the hotel is booked after all</summary>

✗ A slow step that succeeds is still a success, and customers do get good news late. The saga has ended and has given this customer its answer, and no code is left running to send a second one, so the room is held and nobody hears of it.

</details>

---

###### 12. Losing the orchestrator for an hour

A car-rental firm runs its bookings as sagas through an orchestrator that sends commands to the vehicle, payment and insurance services and keeps its database on another machine. The architecture review flagged it as a single point of failure and the firm accepted that. A bad disk takes the orchestrator's machine down at 14:00, with about 300 sagas between steps. At 15:00 the on-call engineer starts the orchestrator on a new machine.

What happens to those 300 sagas after 15:00?

<details><summary>A. They all compensate, and the bookings are undone</summary>

✗ An hour is far past any step's timeout, and timeouts are how an orchestrator decides that a step has failed. The hour was an outage of the orchestrator, and no service failed a step in it. A saga between steps has no command out to time out, so the new process reads its step and sends the next command.

</details>

<details><summary>B. They start again from their first step</summary>

✗ A restarted script runs from its top line. The orchestrator's work is not in its process: it keeps each saga's step in its database, so the new process reads where each one stands instead of starting over.

</details>

<details><summary>C. They carry on from the step each one had reached</summary>

✓ Right. Each saga continues from its own step, and those customers' bookings finish about an hour late.

A new process picks up what the old one left, because **saga state lives in the orchestrator's store and not in its process**. A command sent right before the crash, with no reply on record, can go out again. A saga whose step timeout was already running when the machine died can find it expired, and then compensates as it would on any day.

Keep a standby so the hour shrinks, and back up the store. Alert on sagas that have not moved, and make each service safe to receive a command twice.

</details>

<details><summary>D. They stay stuck, waiting for someone to release them</summary>

✗ A review that writes single point of failure is thinking of stranded work, and work does strand when its state goes down with the process. The firm's database came through, and each saga's next step can be read from it, so a person has nothing to release.

</details>

---

###### 13. Adding a fraud check to a choreography

An electronics shop runs its order flow as choreography. You have added a fraud service, which takes about a minute over an order, and support staff review what it flags. These are the subscriptions in production today.

| Event | Published by | Read by |
| --- | --- | --- |
| `PaymentCaptured` | payment | shipping, fraud |
| `FraudSuspected` | fraud | support |
| `OrderShipped` | shipping | notifications |

A customer's order looks doubtful to the fraud service.

What happens to that customer's order?

<details><summary>A. Shipping holds the parcel while support reviews the flag</summary>

✗ Checkout flow charts draw the fraud check in front of shipping. In choreography the subscriptions are the flow, and shipping reads `PaymentCaptured`, never `FraudSuspected`, so nothing reaches it that could hold a parcel.

</details>

<details><summary>B. The parcel ships, whatever support decides</summary>

✓ Right. The parcel goes out on `PaymentCaptured`, however long the fraud check takes, and support reviews an order that shipping has already acted on.

The fraud service joined as a subscriber, and **new subscribers run beside the others and not between them**. Adding it touched no other service, and shipping reads what it read before. The order of steps in a choreography is written down in no one place, only in who reads what, so to put a step between two others, the later one has to change what it subscribes to.

Have shipping read `FraudCleared`, published by the fraud service for each order it passes, knowing that a fraud outage then stops shipping. If steps keep moving like this, an orchestrator keeps the flow in one place.

</details>

<details><summary>C. The payment service refunds it, and the parcel stays in the warehouse</summary>

✗ Refunds belong to the payment service, and a suspected order is one you would want to refund. The table shows the payment service reading nothing at all, so it never hears of the flag, and shipping has taken the order as usual.

</details>

<details><summary>D. <code>PaymentCaptured</code> reaches shipping a minute late</summary>

✗ In a chain of calls each step adds its time to the ones after it, and this new step takes a minute. A broker gives each subscriber its own copy, so shipping has its `PaymentCaptured` as early as it ever did, and the minute is spent only inside the fraud service.

</details>

---

###### 14. Renaming a field with producer and consumer deployed together

The billing team at a gym chain renames `amount` to `amount_cents` in its `InvoicePaid` event. Free-trial invoices have always left the field out, so the handler has a default. The team owns the producer and the one consumer, so at 09:00 both go live in a single deploy that takes under a minute, and the tests pass against the new shape. At that hour the consumer runs a few minutes behind the log. The new handler is below.

```python
def on_invoice_paid(event):
    cents = event.get("amount_cents", 0)
    ledger.post(event["member_id"], cents)
```

The next day finance looks at the postings from around 09:00.

What does finance find for invoices paid in the minutes before the deploy?

<details><summary>A. Their real amounts, in cents</summary>

✗ The deploy takes under a minute, and across an API that would end the old shape. Whatever the consumer had not reached on its log at 09:00 is read by the new handler, and its look-up asks for `amount_cents` in events that hold `amount`.

</details>

<details><summary>B. No postings, with each event raising an error</summary>

✗ Strict schema validation fails a read when a field is missing, and that failure would have told you at once. The handler asks with a default, written for free trials, so the look-up returns 0 and no error appears.

</details>

<details><summary>C. Postings of zero</summary>

✓ Right. Each invoice still unread at 09:00 posts as zero with no error at all, and members who paid show as having paid nothing.

The consumer is a few minutes behind at 09:00, and **a deploy changes the code but not the events waiting to be read**, so those few minutes of `InvoicePaid` reach the new handler. Its default meant a free trial, and a paid invoice with the old field now looks exactly like one. The tests could not show this, since their events all have the new field.

Rename in steps: the producer writes both fields, the consumer reads the new one and falls back to the old, and the old goes only when no event still holds it, including any history you may replay. Treat a missing amount on a paid invoice as an error, and give free trials an explicit zero.

</details>

<details><summary>D. Their real amounts, posted by the old consumer on its way out</summary>

✗ Graceful shutdown finishes what is in flight, and the old consumer was working right up to 09:00. What it finishes is the batch it holds. The rest of the backlog stays on the log for whoever reads next, and at 09:00 that is the new handler.

</details>

---

###### 15. Adding a new payment status to an event

At a language school, the payment service publishes `PaymentUpdated`, whose `status` has only ever been `paid` or `declined`. The payment team adds `paid_in_instalments` for a new plan. No field changes, and the schema's forward compatibility check in the release pipeline passes. In the team thread, the on-call engineer for the enrolment service, which has not changed since last year, asks whether they need to do anything.

What can the payment team tell them from the passing check?

<details><summary>A. That a reader can still parse the event</summary>

✓ Right. The passing check shows that the event still has a shape a reader can parse. What enrolment does with the new status is outside the check, and the honest answer is that they need to read their handler.

A compatibility check compares a new schema with the previous one, and **what a value means lives in each consumer's code**. The field is still there with the same type, so the check has nothing to object to. Take a handler that sets `active = status == "paid"`: it reads the new event without trouble and marks a student inactive after they have paid their first instalment.

Treat a new value as a change to the contract. List who reads the field, ask them before the release, and have consumers stop for a person on a value they do not know, where falling into a default is costly.

</details>

<details><summary>B. That enrolment treats the new status as it treats <code>paid</code></summary>

✗ The check is called compatibility, and the new value starts with the word paid. It reads two schemas, and neither says how a consumer branches on a value, so enrolment may treat this status as paid, as unpaid or as neither.

</details>

<details><summary>C. That enrolment's tests pass against the new status</summary>

✗ In most pipelines a green check means somebody's tests ran. The check runs in the payment team's release and its inputs are schemas, and enrolment's code and tests are no part of it.

</details>

<details><summary>D. That the new status is among the ones enrolment accepts</summary>

✗ Schemas can list a field's allowed values, and a consumer can hold such a list. The check compares the payment team's new schema with its previous one, so whatever enrolment holds, in a schema or in its code, it has never seen.

</details>

---

###### 16. Replaying after fixing a discount bug

A subscription service is event-sourced. A command handler works out each discount and appends `DiscountApplied` with the amount, and balances are built by folding the events. For a week a bug in the handler gave 50% where 5% was due. You fix the handler, deploy it and rebuild the balances from the start of the history, as the team's runbook says to do after a bug.

What do the affected customers' balances show after the rebuild?

<details><summary>A. The right discounts, worked out again by the fixed handler</summary>

✗ Fix the code and replay is how event sourcing is usually sold, and your runbook says the same. A rebuild runs the fold and never the command handler. The fold reads the amount each event holds, and those amounts were written during the bad week.

</details>

<details><summary>B. The wrong and the right discounts, both applied</summary>

✗ Replays are known for applying things twice. A rebuild starts from nothing and folds each event once. The stream holds one `DiscountApplied` per discount, the wrong one, and the fixed handler has written no second one.

</details>

<details><summary>C. The same wrong discounts as before</summary>

✓ Right. The rebuild comes out the same as before, with 50% discounts for that week, and those customers keep money they were not due.

The amount was worked out once and stored in the event, and a replay reads what is stored. Replay does fix a bug in the fold or in a read model, where the mistake is in how events are read. **Code that writes events puts its bugs in the history**, and the fix to that code reaches only events yet to come.

Append a correcting event for each bad `DiscountApplied`, with the amount and the reason, and let the fold apply it. The history then shows both the mistake and the correction.

</details>

<details><summary>D. No new balance, with the rebuild stopped at the first bad event</summary>

✗ A batch job with validation stops at a bad row, and these amounts are ten times the amount due. The fold has no rule about that: a 50% discount is a well-formed event with a number in it, so the rebuild folds it and moves on.

</details>

---

###### 17. Fixing the balance calculation with snapshots on

A wallet app builds each balance by folding an account's events, and it saves a snapshot of the balance every 1,000 events. Loading an account starts from its latest snapshot and folds the events after it. Since launch the fold has ignored `FeeRefunded` events. You fix the fold and deploy it, leaving the snapshots where they are. A long-time customer has 4,230 events, with one refund at event 900 and another at event 4,100.

What balance does the app load for this customer?

<details><summary>A. A balance with both refunds in it</summary>

✗ Event sourcing guides call a snapshot an optimisation that cannot change the result, and that holds for as long as the fold stays the same. The snapshot at event 4,000 holds a number from the old fold, and the load never reads event 900 again.

</details>

<details><summary>B. A balance with the later refund and without the earlier one</summary>

✓ Right. The load starts from the snapshot at event 4,000 and folds the 230 events after it. The refund at 4,100 counts, the one at 900 does not, and this customer's balance is still short by one refund.

A snapshot stores the result of the fold that wrote it, so **changing the fold reaches only the events after the latest snapshot**.

Tag each snapshot with the version of the fold, and throw away the old ones when the fold changes: the events are still the record, so any snapshot can be built again from them.

</details>

<details><summary>C. A balance with neither refund in it</summary>

✗ A saved copy is used whole, as a backup is, and the snapshot predates your fix. A load only starts from it, and the events after it pass through the new fold, the refund at event 4,100 among them.

</details>

<details><summary>D. No balance, with an error about a snapshot from older code</summary>

✗ Stored data from older code does get refused when a serialisation format changes. The snapshot at event 4,000 is still a balance in the same format, and refusing it would take a version number on the snapshot and a check on load, so it loads with no complaint.

</details>

---

###### 18. Choosing how an event-sourced shop holds addresses

An online pharmacy is moving its customer records to event sourcing before launch, and `AddressChanged` is one of the first events. Events are kept for seven years and backed up. The privacy officer's one requirement for launch is that when a customer asks, nobody can read that customer's address any more. The design meeting is this afternoon, and you are to bring the plan.

Which plan meets the privacy officer's requirement?

<details><summary>A. <code>AddressErased</code> appended to the customer's stream on request</summary>

✗ Appending a new event is how event sourcing puts everything else right, from a wrong discount to a cancelled seat. An appended event changes what the fold produces and leaves the old one where it was, so the address is still in `AddressChanged`, readable by anyone who reads the stream.

</details>

<details><summary>B. The customer's rows cleared from the read models on request</summary>

✗ Erasing a customer usually means deleting their rows, and pages read the read models. The events are still the record: the address stays in them, and the next rebuild of that read model writes it straight back.

</details>

<details><summary>C. The customer's stream rewritten without the address on request</summary>

✗ It is your own store, and a script can rewrite it, as some teams do for a migration. Events are also copied to the logs of whoever consumes them and into your backups, and each of those keeps the address after the rewrite has taken it out of the store.

</details>

<details><summary>D. The customer's own encryption key destroyed on request</summary>

✓ Right. Destroying the key under which that customer's addresses were encrypted leaves them unreadable in every copy of the events. The officer's requirement is met, and the history stays untouched.

This is often called crypto-shredding: you make a field in an append-only history unreadable by **destroying the only key that reads it**. A fold still runs over the events, and the address field is noise once the key has gone. Holding the address outside the events, with a reference in them and a row you can delete, meets the requirement the same way. Both have to be chosen before the first event is written, since events already in plain text stay as they are.

Keep keys in a store of their own, with its backups kept short. Encryption adds work to every read and write, and a consumer that has decrypted an address into its own table has to delete it when told.

</details>

---

###### 19. Rewinding a consumer to rebuild its table

An insurer's reporting consumer reads claim events from a log. You have fixed a bug that damaged a month of `claims_report`, and this is the handler now.

```python
def on_claim_event(event):
    claims_report.upsert(event.claim_id, event.fields)
    if event.type == "ClaimApproved":
        notifier.send_approval(event.claim_id)
```

Late on Saturday you reset the consumer's offset on the log to 30 days back and let it run, since the log still holds those events.

What has the rewind changed by Sunday morning?

<details><summary>A. The table, repaired, with no notice sent again</summary>

✗ Rewinding is sold as the way to rebuild a view, and you started this for a table. The rewind runs the handler as it stands, and the handler has no idea what you started it for: `ClaimApproved` reaches the notifier on Saturday night as it did the first time.

</details>

<details><summary>B. The table, repaired, and every approval notice sent again</summary>

✓ Right. The table comes back right, and each claimant approved in those 30 days gets the approval notice a second time.

The consumer cannot tell a rewind from the first run, because **a replay runs the same handler on the same events**. Writing the rows is safe to repeat, since the upsert lands on the same row. The line that calls the notifier reaches outside the table, and a notice goes out each time it runs.

Split the handler before you need a rewind: one consumer builds the table, and another, with its own offset, sends the notices. Failing that, give a rewind a mode that turns the outside calls off, or have the notifier recognise the claim id.

</details>

<details><summary>C. The table, repaired, and a notice only where a row was damaged</summary>

✗ A replay repairs what is broken, and that is why you started it. The handler knows nothing about which rows were damaged: it sees events in order, and it sends a notice for every `ClaimApproved` among them.

</details>

---

###### 20. Cancelling an order against the read model

A kitchenware shop separates its write side from its read side. Every change to an order is appended to the event store, and a consumer feeds those events into an `orders` read model that every page queries. The cancel handler looks the order up in that read model and appends `OrderCancelled` unless the status there says shipped. On a busy afternoon the consumer is a few seconds behind. The warehouse scans a parcel, which appends `OrderShipped`, and a moment later a support agent with the customer on the phone presses Cancel.

What does the event store hold for that order?

<details><summary>A. <code>OrderShipped</code> alone, with the agent told it is too late</summary>

✗ The check is there to catch this, and in testing, with time between the two clicks, it does. The look-up reads the read model, and the consumer has not yet handled `OrderShipped`. It finds the old status, and the handler appends the cancel.

</details>

<details><summary>B. <code>OrderShipped</code> alone, with the cancel waiting on the read model</summary>

✗ A read model is known to trail the store, and a wait for it to catch up would close the gap. The handler looks once and acts on what it finds, so it never learns that an event is still on its way.

</details>

<details><summary>C. <code>OrderCancelled</code> alone, in place of the scan</summary>

✗ A later write replaces an earlier one in a table row, and the cancel is the later write. An event store only appends, so the scan's `OrderShipped` stays where it is and the cancel goes in after it.

</details>

<details><summary>D. <code>OrderShipped</code> followed by <code>OrderCancelled</code></summary>

✓ Right. The store holds both events, and the agent tells the customer the order is cancelled while the parcel is on its way.

The look-up ran before the consumer had handled `OrderShipped`, so the read model still showed the order as not shipped. The check and the append are two steps on two stores, and **a read model is fed after the write, however short the gap usually is**. A faster consumer makes this rarer and leaves it possible.

Make the decision on the write side: load the order's stream, check the status there, and append with the version you read, so the store refuses the cancel if `OrderShipped` got in first.

</details>

---

## Further reading

- [Online Event Processing](https://doi.org/10.1145/3312527), 2019: how applications built on an append-only event log can keep different storage systems consistent where distributed transactions have not worked.
- [DBLog](https://arxiv.org/abs/2010.12597), 2020: why a transaction log alone typically cannot give a full copy of a table, and how to take one while the log keeps flowing.
- [An Empirical Characterization of Event Sourced Systems and Their Schema Evolution](https://arxiv.org/abs/2104.01146), 2021: 25 engineers on 19 event-sourced systems, the challenges they met and the tactics they used to change event schemas.

Prev: [Message queues and event streams](../message-queues-and-event-streams/README.md) · Next: [WebSockets and real-time delivery](../websockets-and-real-time-delivery/README.md)
