# Message queues and event streams

Prev: [Choosing a data store](../choosing-a-data-store/README.md) · Next: [Event-driven architecture](../event-driven-architecture/README.md) · Reviewed October 2026

A broker stands between the code that writes a message and the code that reads it, so the two need not run at the same moment or the same speed. What it does with a message depends on its kind: a queue gives the message to one consumer and drops it once acknowledged, and a log keeps it and lets each reader remember its own place.

- **Broker**: the server that holds messages between the producers that write them and the consumers that read them.
- **Message queue**: a place in a broker where each message goes to one of the consumers reading it, and leaves once that consumer acknowledges it.
- **Acknowledgment**: a consumer's word to the broker that it may drop its copy of a message.
- **Log**: an ordered, append-only record of messages in a broker, often called an event stream. Reading a message leaves it in place, for any number of readers.
- **Log partition**: one of the ordered parts a log is split into. A producer's key picks the partition, so messages with one key stay in order, and a log promises no order across partitions.
- **Offset**: a message's position in a partition, and a reader's place. A consumer commits it to save that place; auto-commit does so on a timer. Moving it back is called a rewind.
- **Consumer group**: consumers sharing a log's reading under one name. Each partition has one saved offset for the group.
- **Rebalance**: handing partitions out again when a group's members change. Some brokers stop the whole group to do it.
- **Reset policy**: where a group with no valid saved offset starts: the newest message, the default on many clients and on those here, or the oldest the log holds.
- **At-most-once, at-least-once, exactly-once**: a message may be lost and never repeated, may be repeated and never lost, or takes effect once.
- **Visibility timeout**: how long a queue hides a received message from other consumers, unless the consumer deletes it first. Some queues instead hold it for as long as the consumer's connection lives. Unless a question says otherwise, no consumer extends it.
- **Poison message, dead-letter queue**: a poison message fails every time it is handled, and a dead-letter queue parks a message that has used up its attempts, for someone to tell a passing fault from a defect, fix it and replay. An alarm on it fires only once messages are there.
- **Lag**: how far a consumer group is behind the end of a log, counted in messages.
- **Retention**: the limit, in age or size, past which a log deletes its oldest messages. Some brokers instead keep what a named subscription has not acknowledged, under a separate limit.
- **Event time, processing time**: the moment something happened, and the moment a program reads the message about it.

A topic copies each message to every queue bound to it at that moment, so each reader gets a queue of its own, which goes on collecting while the reader is away. Consumers usually pull, asking for more when free, so they can fall behind and catch up. A failed message returned to a queue at once comes straight back, and a worker that fails at once asks again sooner than its neighbours.

Producers repeat too: a lost confirmation makes one resend a message the broker stored. A broker's filter on message IDs drops a repeated publish inside a time window, and it never covers a redelivery to a consumer. A log's transaction can make writes to several partitions and a committed offset atomic.

A queue told to keep order per key holds a key's later messages behind an unacknowledged one, and a queue spread over many machines can otherwise deliver twice or out of order. A compacted log keeps only the last value per key.

A queue evens out a burst for what stands behind it, and most trouble here comes from reading a broker's words as promises about your work. [Event-driven architecture](../event-driven-architecture/README.md) takes up the dual write, the outbox and what a replay does to its effects, and [Replication](../replication/README.md) how many copies a broker waits for.

Unless a question says otherwise, everything here runs at its default settings, and nobody has added a retry, check, limit, job or tool that the question does not name.

This topic lets you:

- say what a second service, a long stop or a renamed group does to the messages a queue or a log holds
- follow a message through a publish, a crash and a redelivery, and place the record that keeps its work from running twice
- add workers, consumers or partitions to a system that has fallen behind, and say what happens to its order and its lag
- run consumers through a bad record, a dead-letter queue, an outage and a night nobody was watching

###### 1. Pointing a second service at the billing queue

A ticket-selling site puts each paid order on a queue, and a billing service reads the queue and issues invoices. The analytics team wants the same orders, so it deploys a service of its own against that queue, with the client code billing uses.

Which orders reach each service from that day on?

<details><summary>A. Both services get every order</summary>

✗ A second subscriber on a topic gets a copy of everything, and two groups on a log read everything too. This is a queue, and it holds one copy of each order: whichever service asks first takes it out of the other's reach.

</details>

<details><summary>B. Analytics gets each order first, and billing gets it once analytics acknowledges</summary>

✗ Request pipelines pass a message from one step to the next, and architecture drawings show the arrows that way. Two consumers on a queue sit side by side, and an acknowledgment removes the order, so nothing is left for billing to get after analytics has finished with it.

</details>

<details><summary>C. Billing gets some orders, and analytics gets the others</summary>

✓ Right. Some paid orders never get an invoice, and the analytics figures cover only part of the sales, because each order goes to whichever service asks for it first.

Two consumers on a queue compete, since **a queue holds one copy of a message and gives it to one consumer**. If the two ask in turn, 100 orders split into 50 and 50, and no order reaches both.

Take analytics off this queue now. Give it a queue of its own, bound to a topic that copies each order to both, or move the orders to a log that each service reads in its own consumer group.

</details>

<details><summary>D. Billing gets every order it can keep up with, and analytics gets the overflow</summary>

✗ A spare machine takes the overflow once the first is full, and billing was on this queue first. A queue keeps no such rank: analytics takes the next order whenever it asks, so billing loses orders on a quiet afternoon as well.

</details>

---

###### 2. Stopping a fraud team's consumers for two days over a sale

The fraud team at a card processor reads payments from a log whose retention is set to a week of messages and a disk quota sized for a little over a week of ordinary traffic. In spring a bad deploy stopped the team's consumers for a day, and when they came back they read everything they had missed. During a holiday sale, with payments at about twenty times the usual rate, a migration stops them again for two days.

What has become of the payments from the first day of the stop?

<details><summary>A. They wait in the log, with the group's offset pointing at them</summary>

✗ Two days is well inside a week, and the spring stop came back whole from the same saved offset. The week is one of two limits, and at sale traffic the quota ran out within hours, so the offset now points at payments the log has deleted.

</details>

<details><summary>B. They sit in a dead-letter queue, ready to replay</summary>

✗ Messages a team has not processed are what a dead-letter queue is usually pictured as holding. It takes only messages a consumer tried and gave up on, and the stopped consumers never reached these, so nothing went there.

</details>

<details><summary>C. They were never written, with producers refused at the quota</summary>

✗ Some bounded queues refuse new messages when they are full, and a quota is a bound too. This log makes room at its limit by deleting its oldest messages, so the producers kept writing through the sale while the payments the team had not read were deleted.

</details>

<details><summary>D. They are gone, read by no one on the team</summary>

✓ Right. The first day's payments are deleted, and nobody scored them for fraud.

Retention has two limits, age and size, and **the log deletes at whichever comes first, whether or not anyone has read the messages**. If the quota holds eight ordinary days, then at twenty times the rate it holds about ten hours, so about 38 of the 48 hours are gone by the restart. Spring's stop was one day at the ordinary rate, inside both limits.

Before a planned stop, work out how many hours the quota holds at the rate you expect, and alert well before the backlog gets there.

</details>

---

###### 3. Renaming a consumer group in a tidy-up release

A logistics company runs a notification service that reads shipment events from a log, in a consumer group called `notify`, on the client's default settings. A release tidies the naming, and one line of config makes the group `notify-shipments`. The reviewer passes it as cosmetic. The deploy stops the old consumers and starts the new ones four minutes later.

Where does the renamed group start reading?

<details><summary>A. At the offset the old group had reached</summary>

✗ Every restart this service has had picked up where it stopped, and the code has not changed. The saved offset belongs to the name, so it stays under `notify`, a name nothing reads any more.

</details>

<details><summary>B. At the newest event, past those four minutes of shipments</summary>

✓ Right. The group starts at the newest event, past every shipment that moved during the deploy, and nobody is told about those.

To the broker `notify-shipments` is a new group with no saved offset, because **offsets are stored under the group's name**, and the reset policy then decides where it starts. The events of those four minutes lie between where `notify` stopped and where the new group begins, and neither group reads them. On a client that defaults to the oldest message, the same rename sends every notification again for all the log still holds.

Before you rename a group, stop the old consumers and set the new name's offset to where the old group stopped.

</details>

<details><summary>C. At the oldest event the log still holds</summary>

✗ Tutorials on logs show a new reader starting from the beginning, and this group is new. Starting there is something you ask for: this client runs on its defaults, whose reset policy goes to the newest message.

</details>

<details><summary>D. Nowhere, with the broker refusing a group it has never seen</summary>

✗ A broker can refuse a queue name it does not know, and this group's name is new to it. A group with no saved offset is the case the reset policy exists for, so the broker lets it in and it starts reading.

</details>

---

###### 4. Returning 200 right after publishing a booking

A cinema's booking API takes each booking by publishing an event, and a consumer writes it to the database later. `queue.publish` is your thin client, and it returns a handle that completes once the broker confirms.

```python
queue = broker.client(batch_wait_ms=50)

def create_booking(request):
    booking_id = new_id()
    queue.publish("bookings", {"id": booking_id, "seat": request.seat})
    return Response(200, {"id": booking_id})
```

On a quiet night a guest gets a 200 with a booking number, and a few milliseconds later a deploy kills the process with no wait for it to finish.

What does the broker hold for that booking?

<details><summary>A. The booking, stored and waiting for the consumer</summary>

✗ A database insert that returns without an error has stored the row, and this call has the same shape. This one returned before the event left the process: a few milliseconds into a 50 ms wait, it is still in the client's batch.

</details>

<details><summary>B. The booking, resent by the client when the new process starts</summary>

✗ Clients do retry for you, and their retry settings sit beside the batch settings. Those retries live in the same process and work from the same memory, so the new process starts with an empty client that has never heard of this booking.

</details>

<details><summary>C. Nothing for this booking, though the guest holds its number</summary>

✓ Right. The broker never received the event, so the guest holds a booking number for a seat no consumer will ever write down.

With this setting **a batching client's publish returns once the event is in its memory**. The batch would have left within 50 ms, and the kill came first. Only the handle can tell you the broker stored the event, and this handler drops it.

Wait on the handle before you answer 200, and resend if it fails, knowing a resend can leave a duplicate for the consumer to deal with.

</details>

<details><summary>D. The booking, marked unconfirmed for someone to check</summary>

✗ Broker transactions have a prepared state, and this handle completes on a confirm that has not come. A broker learns of a message when it arrives, and this one never arrived, so there is nothing there to mark.

</details>

---

###### 5. Restarting a print worker in the middle of a deploy

A print-on-demand shop takes book orders from a queue, and a pool of workers runs the loop below. `printer.submit` sends the job to an outside print house, which prints whatever it receives. Tonight's deploy kills the workers wherever they are in the loop, with no wait for them to finish.

```python
while True:
    message = queue.receive("book-orders")
    printer.submit(message.order, message.file)
    queue.ack(message)
```

What can the customer whose order was in hand at that moment receive?

<details><summary>A. Never more than one copy of the book, with only unfinished orders coming back</summary>

✗ The acknowledgment sits on the last line, where every guide puts it. The broker sees only whether the acknowledgment arrived, so a job the print house already has looks the same to it as one never started, and it goes out again.

</details>

<details><summary>B. No book, with the order gone along with the worker that took it</summary>

✗ Popping an item off a list leaves the list without it, and `queue.receive` has the same shape. A received message stays with the broker until it is acknowledged, so the worker's death frees it for the next worker, and this order is printed.

</details>

<details><summary>C. One copy of the book, with the order parked in the dead-letter queue</summary>

✗ A dead-letter queue collects failed attempts, and a kill is an attempt that did not finish. A message gets there only past a limit on attempts, and one kill is one attempt, so this order goes back to the queue and the next worker submits it.

</details>

<details><summary>D. Two copies of the book</summary>

✓ Right. A worker killed between `printer.submit` and `queue.ack` leaves a job the print house already has and a message the broker thinks is unfinished. The next worker submits it again, and the customer gets two books for one order.

That is at-least-once: the acknowledgment tells the broker it may drop its copy, and it tells the broker nothing about the print house. Moving `queue.ack` above the submit gives at-most-once, where a kill in the gap loses the order: **a kill can fall between any two steps on two systems**, so no placing of the two lines removes the gap.

Send the print house a key built from the order, if it will refuse a repeat, or have it report what it already holds. Without one of those, keep the repeat cheap enough to live with.

</details>

---

###### 6. Moving receipts into a thread pool under auto-commit

A shop emails its receipts from a log. To keep up during a sale, a developer hands each message to a pool of threads.

```python
consumer = log.consumer("receipts", group="email", auto_commit_every="5s")
pool = ThreadPool(20)

while True:
    for message in consumer.fetch(max_messages=100):
        pool.submit(send_receipt, message)
```

An auto-commit runs, and one second later an out-of-memory kill takes the process. About 80 receipts fetched before that commit are still waiting in the pool.

What becomes of those 80 receipts?

<details><summary>A. They are never sent</summary>

✓ Right. Those 80 customers get no receipt. The last commit moved the group's offset past them, and the pool holding them died with the process.

**Auto-commit saves how far the client has fetched**, and it does not track the pool. In a plain loop that finishes each batch before its next fetch, the two are the same, and a crash means reading again. Here `fetch` comes round as soon as the work is handed over, so the offset runs ahead of the sending. Had the kill come right before a commit, the next consumer would read again every message fetched since the last one, so the same crash can lose receipts or repeat them.

Turn auto-commit off and commit only what the pool has finished, up to the oldest unfinished message, or keep the sending inside the loop.

</details>

<details><summary>B. The next consumer re-reads all 80 and sends them</summary>

✗ Descriptions of auto-commit promise at-least-once, and that holds for a loop that finishes each batch before its next fetch. Here the committed offset is already past the 80, and the next consumer starts from that offset, so it never sees them.

</details>

<details><summary>C. Each one is sent twice</summary>

✗ A crash in a consumer usually means repeats, which is what every explanation of at-least-once warns about. A repeat needs a second reading, and the offset has moved past these messages. None of the 80 went out even the first time.

</details>

---

###### 7. Turning on exactly-once for a job that sends texts

A courier firm runs a stream job that reads delivery events from a log, writes a status to a second log and calls a text-message provider to tell the customer, all before it commits the event. You turn on the broker's exactly-once setting. You kill the job again and again in a test, and the status log never shows a reader one delivery twice. In production, the job's machine dies in the middle of one event.

What can the customer get for that delivery?

<details><summary>A. Never more than one text</summary>

✗ The setting's name says once, and your kill test backed it, with never a repeat in the status log. That log is inside the transaction and the provider is outside it, so a retried event calls the provider again.

</details>

<details><summary>B. No text, with the send undone along with the transaction</summary>

✗ A transaction undoes everything inside it, and in your code the call sits between the read and the commit. The abort throws away the status write. The provider has already sent the text, and no broker can take it back off the customer's phone.

</details>

<details><summary>C. A second text only when the first call to the provider timed out</summary>

✗ Repeats usually get blamed on a timeout, where the first call's fate is unknown. This repeat needs none: the first call can succeed and the machine can die before the commit, and the restarted job has no record of the call.

</details>

<details><summary>D. A second text for the same delivery</summary>

✓ Right. If the machine dies after the provider call and before the commit, the restarted job reads the event again and calls the provider again. The customer has two texts, while readers of the status log see one status.

Your test looked at the one place the promise holds: **the broker's exactly-once covers its own logs and the committed offset**, which commit or abort together. A call to the provider is part of no such transaction, so it runs each time the event does.

If the provider honours an idempotency key, which lets it drop a repeat, build one from the delivery. If it does not, write a "text wanted" message inside the transaction and have a separate consumer send it and keep its own record of what has gone out. That still leaves a small gap, so decide whether a rare second text is a price you can pay.

</details>

---

###### 8. Keeping processed message IDs for a loyalty-points consumer

A supermarket app adds loyalty points for each purchase message, in a table in the app's relational database, and a purchase message is delivered again every so often. To stop a repeat adding points twice, you will record the ID of each message you have handled. A shared cache is already in place and answers far faster than the database.

Where should that record go?

<details><summary>A. A table in the app's database, written in the transaction that adds the points</summary>

✓ Right. Each purchase adds its points once, wherever a crash falls, because the points and the record commit together or not at all. The speed of the cache buys nothing that matters here.

**The record and the effect have to change as one step**, and only the database that holds the points can do that. A unique key on the message ID also stops two workers holding copies of one message at the same moment, since the second insert fails and its transaction rolls back.

Use an ID the producer sets and keeps across redelivery, not one the transport makes up each time, and keep the record for as long as something can resend the message, a rewind on a log included.

</details>

<details><summary>B. The shared cache, claimed with an atomic set-if-absent and released if adding the points raises</summary>

✗ An atomic claim is the recipe many cache guides show, and it does stop two workers racing. A worker killed after the claim and before the points never runs its release, so the redelivered purchase finds the claim and is skipped, and the customer loses those points.

</details>

<details><summary>C. A table in the app's database, written in its own transaction right after the points</summary>

✗ The table is in the right database, and keeping transactions small is a good habit. A kill between the two commits leaves the points added with no record of them, so the redelivery adds them a second time.

</details>

<details><summary>D. Nowhere, with any message carrying the broker's redelivered flag skipped</summary>

✗ Brokers do mark what they send again. The flag means only that an earlier delivery went unacknowledged, which is equally true of a worker that died before adding the points, and skipping that purchase leaves the customer without them.

</details>

---

###### 9. Transcoding a long film under a 30-second visibility timeout

A video site puts each upload on a queue with a 30-second visibility timeout, and ten workers take uploads from it. A worker deletes the message when the transcode is done. Clips take under ten seconds. Today a studio uploads a film that takes about twenty minutes to transcode.

What do the ten workers do with the film's message during those twenty minutes?

<details><summary>A. One worker transcodes it while the message stays hidden</summary>

✗ The worker has received the message, and job dashboards show it as in progress. The hiding ends at 30 seconds whatever the worker is doing, since nothing tells the queue that the film is being worked on, and the next free worker takes it.

</details>

<details><summary>B. Each worker gets 30 seconds, is stopped, and the next starts over</summary>

✗ A request timeout ends the request, and this setting is called a timeout too. This one changes only what the queue shows: the first worker goes on transcoding, and each later worker adds to the work without replacing any of it.

</details>

<details><summary>C. One after another they pick it up, and soon all ten are transcoding the film</summary>

✓ Right. After about four and a half minutes all ten are transcoding one film, since the message comes back every 30 seconds and another worker takes it. Every creator's clip waits behind them.

**A visibility timeout hides a message and holds no lock on it**, and the queue only counts time, without seeing the work. The first worker's delete comes at twenty minutes, after all the others have started.

Have the worker extend the timeout while it works, and make a repeat harmless, since an extension can fail. Raising the timeout to an hour instead would stop the repeats and leave a crashed worker's message hidden for that hour.

</details>

<details><summary>D. One worker transcodes it, and the message sits in the dead-letter queue</summary>

✗ A timeout counts as a failure in most code, and the dead-letter limit sits beside this timeout in a queue's configuration. A message moves to a dead-letter queue only past a limit on receives, and with no such limit the film goes back to this queue every 30 seconds.

</details>

---

###### 10. Adding a second worker to a queue that keeps order

At a hotel chain, the profile service reads updates from a queue that hands messages out in publish order, and the messages carry no key. One worker reads it, and in the high season that worker falls behind, so you start a second. A guest changes an email address twice within a second.

What order do the two changes reach the database in?

<details><summary>A. The first change first, every time</summary>

✗ The feature page says first in, first out, and with one worker that is what you saw, since the second message waited for the first to finish. Now a second worker takes it right away, and either can finish first.

</details>

<details><summary>B. The second change first, some of the time</summary>

✓ Right. When the first write runs longer than the second, the second lands first. The guest's profile can then end up with the address they gave up, and mail goes there.

**The order a queue keeps is the order of handing out**, and the two workers then run at their own speeds. Say the first write takes 30 ms, and the second, taken 1 ms later, takes 5: the second lands at 6 ms and the first at 30. With other durations the two land in order, which is why this shows up now and then.

Give the messages a key, the guest's ID, on a queue that can keep order per key, or have the write carry a version and reject anything older than the row holds.

</details>

<details><summary>C. The first change first, with the second worker held back for the first's acknowledgment</summary>

✗ A queue that keeps order per key does hold a key's next message until the last one is acknowledged, and this queue is praised for its order. These messages carry no key, so nothing ties the two changes together, and the second worker takes the next message straight away.

</details>

<details><summary>D. The second change alone, with the queue dropping the older one</summary>

✗ A compacted log keeps only the last value per key, and a last-value cache does the same. A queue delivers every message it holds, so both changes reach a worker and both reach the database.

</details>

---

###### 11. Scaling a lagging group to twenty consumers during a cup final

A sports-results site reads score events from a log with eight partitions, in one consumer group of eight consumers, each formatting scores as fast as it can. During a cup final the lag climbs and the lag alert fires. The on-call engineer follows the runbook and raises the group to twenty consumers, and the deploy shows all twenty healthy.

What does the lag alert do over the next half hour?

<details><summary>A. It clears, with the twenty sharing the events between them</summary>

✗ Adding copies of a stateless service adds capacity, and consumers on a queue do share work this way. In a group each partition has one reader, so the events of eight partitions stay with eight consumers, and the other twelve read nothing.

</details>

<details><summary>B. It clears, with every score reaching the site several times</summary>

✗ Two groups on one log each read every message, and these are twenty consumers on one log. They share one name and one saved offset per partition, so a score is read once, and the group reads no faster than it did.

</details>

<details><summary>C. It clears once the broker has split the partitions to fit</summary>

✗ A store that splits its shards as it grows does this, and a rebalance is a change to how a log is shared out. A rebalance hands out the partitions that exist, and the count stays at eight until someone changes it.

</details>

<details><summary>D. It stays on, with the extra consumers holding no partition</summary>

✓ Right. The alert stays on and the lag goes on climbing at the same pace, while twelve of the twenty healthy consumers read nothing.

Where each partition has one reader in a group at a time, **a group can use no more consumers than the log has partitions**. If each consumer handles 1,000 scores a second and 12,000 arrive, lag grows by 4,000 a second with eight consumers and with twenty.

Cap the group at the partition count, in the runbook and in any autoscaler. To go faster, make each consumer faster, add partitions, which has its own cost for keyed messages, or use a mode where several consumers share a partition, on a log that has one, which gives up order within that partition.

</details>

---

###### 12. Doubling the partitions of a log keyed by account

A payments platform publishes ledger events keyed by account, and the producer hashes the key over the partition count. Consumers on all eight partitions run a few minutes behind at the evening peak, so you raise the log to sixteen partitions without stopping anything. One account gets a deposit moments before the change and a withdrawal moments after it.

What order can the group apply those two events in?

<details><summary>A. The withdrawal before the deposit, on some accounts</summary>

✓ Right. For about half the accounts the withdrawal lands on a new, empty partition and is read at once, while the deposit waits its turn on the old one, so a balance check can refuse a withdrawal against money that has already arrived.

**A hash over the partition count places a key only for that count**, and what was already written stays where it was. Hash 1,000 accounts over eight partitions and then sixteen, and about 500 move. One account's events now sit on two partitions, each read on its own.

Choose a generous partition count at the start. If you must add partitions, do it while the old ones are caught up, and hold back those keys until they are.

</details>

<details><summary>B. The deposit first on every account, with both on its old partition</summary>

✗ The rule as usually taught is that a key picks a partition, with nothing said about the count. The key picks by way of the count, and the count has changed, so for about half the accounts the withdrawal goes elsewhere.

</details>

<details><summary>C. The deposit first, moved by the broker to the account's new partition</summary>

✗ A database moving a shard takes the rows along, so the data follows the key. A log keeps what was written where it was written, and the broker moves nothing, so the deposit stays on the old partition behind its backlog.

</details>

<details><summary>D. The deposit first, with the new partitions held back while the old ones drain</summary>

✗ Some logs that split a hash range do read the old range before the new ones. Here the producer hashes over the count and each partition's consumer reads on its own, so nothing holds the empty new partitions back.

</details>

---

###### 13. Working through a batch of messages between fetches

At a photo-sharing app, a consumer group makes thumbnails from a log. Each consumer fetches a batch, processes it message by message, commits, and fetches again, while a background thread sends heartbeats to the broker. The group's limit between fetches is five minutes, and a batch takes under a minute. One afternoon the object store slows down, and a batch takes about eight minutes.

What does the group's lag do that afternoon?

<details><summary>A. It falls slowly, as the consumers work at the slower pace</summary>

✗ A slow dependency usually means slow progress, and every message here does get processed, only slowly. Progress only counts at the commit, and no consumer reaches one before it is removed, so the group ends each round where it began.

</details>

<details><summary>B. It stays level, with heartbeats keeping every consumer in the group</summary>

✗ Heartbeats are the health signal in most systems, and this thread sends them on time. They show that the process is alive, and this limit is on fetches, so a consumer eight minutes into its batch is removed all the same.

</details>

<details><summary>C. It grows, with the same batches processed again and again</summary>

✓ Right. Lag grows while the same photos get thumbnails over and over, because every consumer is removed before its batch reaches the commit.

At minute five this consumer is still working, and **a member past the limit between fetches is out of the group**, whatever its heartbeats say. A rebalance gives its partitions to another member, which fetches from the last commit, gets the same batch and meets the same limit. The removed consumer's commit at minute eight fails on a broker that tracks membership, since the partition has another owner.

Fetch a batch small enough to finish inside the limit at the worst speed you expect, and commit as you go, so a removal costs part of a batch.

</details>

<details><summary>D. It falls fast, as healthy consumers take over the slow ones' partitions</summary>

✗ A balancer's health check takes out a slow backend, and the rest speed up. Here the same object store slows every consumer, so whoever takes over a partition inherits a batch that also takes eight minutes.

</details>

---

###### 14. Meeting one unreadable record on a partition

A bank reads card transactions from a log with twelve partitions, in a group of twelve consumers, each running the loop below. `consumer.current()` returns the message at the group's offset. A producer release ships one record with a field the parser cannot read, and it lands on partition 5.

```python
while True:
    message = consumer.current()
    try:
        ledger.apply(parse(message.value))
        consumer.commit(message)
    except Exception:
        log.exception("apply failed")
        backoff.wait()
```

What do the group's twelve partitions do after that record arrives?

<details><summary>A. Partition 5 holds back only records with that record's key, and the rest of it carries on</summary>

✗ Order is taught as a promise per key, and a queue that keeps order per key does hold back only that key. A log's consumer moves one offset along the partition and has no notion of keys, so every key behind this record waits.

</details>

<details><summary>B. Partition 5 stands still while the other eleven carry on</summary>

✓ Right. The consumer on partition 5 stops at that record with every transaction behind it waiting, and the cards that hash there stop posting while the other eleven partitions run as usual.

A partition is read in order, and **a consumer that commits after handling moves the offset only past what has been handled**. When the parse fails, the loop waits, goes back to the top, and `consumer.current()` returns the same record.

Catch a parse failure apart from a passing fault, write the record to a dead-letter queue of your own and commit past it, knowing that later records for its key then apply without it. Keep retrying the passing faults, such as a ledger that is down, since skipping those would lose order.

</details>

<details><summary>C. Partition 5 moves past the record after a few failed tries, and all twelve carry on</summary>

✗ Every retry policy you have met has a limit, and `backoff` is the usual name for one of those helpers. This loop has no way out: `backoff.wait()` returns, `consumer.current()` gives the same record, and the next try fails as the first did.

</details>

<details><summary>D. Partition 5 resumes once the broker moves the record to a dead-letter queue</summary>

✗ On a queue the broker does count receives and move the message for you, and that is where most people meet dead-letter queues. A log's broker sees fetches and commits and cannot tell that a message failed, so the parking is code you would have to write.

</details>

---

###### 15. Slowing a database behind a dead-letter limit of three receives

A ticket seller takes orders through a queue with a 60-second visibility timeout, which moves a message to a dead-letter queue after three receives. An order normally takes a second, and the pool of workers reading the queue always has some free. On the morning of a big sale the database slows down, and an order takes about five minutes.

What is in the dead-letter queue by noon?

<details><summary>A. Nothing, with every order finished late</summary>

✗ Every worker is still running, and slow is no failure on any screen you watch. The queue does not see failures in any case, only receives, and an order still in the works at 180 seconds has used its three.

</details>

<details><summary>B. Only orders whose database call returned an error</summary>

✗ A struggling database does return errors, and dead-letter queues are presented as the place for failures. An error is not needed: a call that succeeds after five minutes has been received three times by then.

</details>

<details><summary>C. Ordinary orders, many of them processed as well</summary>

✓ Right. The dead-letter queue fills with orders that have nothing wrong with them. Workers still running after the move finish many of those orders, so replaying the queue as it stands would process them again.

The limit counts receives, and **a receive that outlasts the timeout counts the same as a failed one**. One order is received at 0, 60 and 120 seconds, and moved at 180 when it would come back a fourth time, while the three workers holding it keep going and finish at 300, 360 and 420.

Set the timeout above the slowest time you expect, or extend it while the work runs, and make the work safe to repeat. Before replaying a dead-letter queue, check whether its messages failed at all.

</details>

<details><summary>D. Only orders with bad data in them</summary>

✗ Introductions present this queue as the home of poison messages, and those do end up there. These orders parse as well as any other day's, so nothing about their data sent them there.

</details>

---

###### 16. Replaying a dead-lettered price update a day later

At a grocery chain, the price service reads updates from a queue. A bug makes the consumer reject one update, and after three tries it moves to a dead-letter queue. The next morning an engineer fixes the bug and moves the parked messages back.

```text
Mon 10:00  update 1042  SKU 88 -> 10.00   rejected, moved to dead-letter queue
Mon 10:05  update 1043  SKU 88 -> 12.00   applied
Tue 09:00  bug fixed, dead-letter queue moved back to the main queue
```

What price does the shop show for SKU 88 after that?

<details><summary>A. 10.00 for a moment, then 12.00, with update 1043 replayed behind it</summary>

✗ Rewinding a log replays everything after the point you choose, and replay is what the engineer calls this. Moving a dead-letter queue back sends only what was parked. Update 1043 was acknowledged on Monday and is gone, so nothing repeats it.

</details>

<details><summary>B. 12.00, with a repeat of a safe write changing nothing</summary>

✗ Setting a price is the textbook idempotent write, so repeating one changes nothing. Update 1042 is no repeat, since it was never applied, and the consumer writes its 10.00 over the 12.00 like any other update.

</details>

<details><summary>C. 12.00, with update 1042 sent straight back after using up its tries</summary>

✗ Attempt counts can stay on a message, and this one has spent its three. The bug behind those attempts is fixed, so the first new try succeeds, and a limit that is never reached again changes nothing.

</details>

<details><summary>D. 10.00, written over the later 12.00</summary>

✓ Right. The shop shows 10.00 for SKU 88, a price nobody has wanted since Monday morning, because the older update lands last.

Update 1043 overtook update 1042 on Monday, and **a replay puts a parked message behind everything that overtook it**. Setting a price is safe to repeat, but not safe to apply after a newer price.

Have each update carry a version, and have the consumer reject anything older than what it holds, or park the later updates for a key once one of its updates has failed. Read what is parked before you move it back.

</details>

---

###### 17. Getting the text provider back after a three-hour outage

A ride-hailing app sends its "your driver has arrived" texts through a queue, and its workers, which acknowledge a message once the provider accepts it, have far more room than the day's traffic needs. The text provider goes down for three hours, and a rider books a car in the middle of them. The ride is over an hour before the provider comes back.

What does that rider's phone show once the provider is back?

<details><summary>A. A text saying the driver has arrived, an hour after the ride ended</summary>

✓ Right. A rider who left the car an hour ago is told the driver has arrived, since the workers send that message within minutes of the return, as they would any other.

**With no age limit set, a message stays in the queue until a consumer acknowledges it**, and nothing in the queue knows that "arrived" is worth nothing after a few minutes. The workers' spare room changes only how fast the old texts go out.

Give messages that go stale a TTL of a few minutes, an age past which the broker drops them, so that an outage costs you those texts and leaves nothing to send at the return. Keep the wait for messages that stay true, such as a receipt.

</details>

<details><summary>B. No text, with the queue having dropped one that old</summary>

✗ A queue can carry a TTL, and a text this far out of date is worth nothing to anyone. Without a TTL the broker has no way of knowing that, and it keeps the message until a consumer acknowledges it, so this one is still there for the workers.

</details>

<details><summary>C. No text, with the worker skipping a ride that is over</summary>

✗ The app knows the ride ended, and a person sending the text would check first. The message says only that a driver has arrived, and a worker sends what the queue gives it, so a check on the ride is code someone would have to add.

</details>

<details><summary>D. Nothing new, the text having gone out on time through the workers' retries</summary>

✗ Retrying with backoff is the standard answer to a provider's outage, and the workers kept at it. A retry cannot reach a provider that is down: every attempt in those hours failed, and the first that could get through came an hour after the ride.

</details>

---

###### 18. Leaving the email consumers stopped over a sale weekend

At an online shop, checkout saves the order, publishes an event for the confirmation email and waits for the broker's confirm before it answers. The email service reads a queue on a broker the shop runs itself, on one machine whose disk has room for about a day of the sale's events. On Friday night a bad release stops the email consumers. Nobody is paged, since email is not critical, and the sale runs through the weekend.

What do customers at checkout get by Sunday?

<details><summary>A. Normal checkouts, with the emails owed to them waiting in the queue</summary>

✗ That is what a queue is for, and for the first day it holds. The disk then reaches its limit with every message still unread, and the broker stops taking any more.

</details>

<details><summary>B. Normal checkouts, with the broker dropping the oldest emails to make room</summary>

✗ A cache evicts its oldest entries when it fills, and a log deletes by retention. This queue removes only what a consumer has acknowledged, and nobody is acknowledging, so a full broker stops taking messages.

</details>

<details><summary>C. Checkouts that stall or fail at the publish</summary>

✓ Right. The broker stops taking publishes, the handler waits for a confirm that does not come, and customers cannot finish paying.

Checkout sees nothing of the outage while the disk has space, because **the two sides stay decoupled only for as long as the queue has room**. A broker out of disk or memory then protects itself by holding back publishers, and a publish inside a request hands that wait to the customer. One broker stops reading from publishing connections, so the call hangs, and another can also fail the send with an error. A log would delete its oldest messages here instead, and a queue with no practical limit would only move this day further off.

Give the publish a short timeout and decide what checkout does when it fails, such as saving the event beside the order to send later. Alert on every queue's backlog, the non-critical ones too, since its growth reaches everything sharing the broker.

</details>

<details><summary>D. Checkouts that answer as usual, with the events held in the client's buffer</summary>

✗ The publish client does have a buffer, and it holds events for a while. This handler waits for the confirm before it answers, so a held event keeps its customer waiting.

</details>

---

###### 19. Setting one lag alert for a busy log and a quiet one

A retailer runs one alert rule over its consumer groups: lag above 10,000 messages. The payments log takes thousands of messages a second and its group normally runs about a second behind, and the refunds log takes about one a minute. One night the refund consumer hangs at midnight, and in the morning a customer asks where a refund went, with no alert in anyone's inbox. You want one rule for every log.

Which rule would have woken someone within the hour and stayed quiet on a normal night?

<details><summary>A. The same lag rule, lowered to 50 messages</summary>

✗ Too high a limit is the first suspect, and the refund lag does pass 50 within the hour. The payments group is always that far behind in normal running, since what it has fetched and not yet committed counts as lag. Say a second's worth, 5,000 messages: this rule would fire every night.

</details>

<details><summary>B. The age of the oldest unread message, above ten minutes</summary>

✓ Right. The oldest unread refund reaches ten minutes at ten past midnight, and the payments log, about a second old on a normal night, stays quiet, so on-call hears about the hung consumer long before the customer does.

**A count of messages means a different wait at each rate**, and age measures the wait itself. By eight in the morning the refund lag is 480, and it would take about a week to reach 10,000.

Alert on age per partition, since a partition left with no consumer after a bad rebalance hides inside a group's total while its own age shows it, and keep the count for sizing and for watching a backlog drain. Skipping or parking a stuck message resets the age, so read it beside the size of the dead-letter queue.

</details>

<details><summary>C. An alert on errors logged by the consumers</summary>

✗ A consumer with a bug logs errors, and that is how most bugs reach on-call. A hung consumer is not running the code that would log one, so this rule stays silent all night.

</details>

<details><summary>D. An alert on the consumers' CPU falling near zero</summary>

✗ A dead process drops to zero on every host screen, and a stopped consumer does no work. At one message a minute the refund consumer sits near zero on every healthy night too, so this rule fires all the time.

</details>

---

###### 20. Counting hourly sales on a job that was down overnight

A chain of coffee shops feeds an hourly sales chart from a log. The job below reads each sale and adds it to a count. It stops at 02:00 after a bad config, the shops open at 05:00, and it restarts from its saved offset at 07:00, reading the morning's sales within minutes.

```python
for sale in log.consumer("sales", group="hourly-chart"):
    hour = clock.now().replace(minute=0, second=0, microsecond=0)
    counts[(sale.shop, hour)] += 1
```

What does the chart show for that morning?

<details><summary>A. Nothing for 05:00 and 06:00, then one towering bar at 07:00</summary>

✓ Right. The chart reads as if the shops opened at seven to a huge rush, and no sale is lost: each is counted once, in the wrong hour.

`clock.now()` is processing time, the moment the job read the sale, and the backlog was all read in one go. Say each shop sells 120 an hour: the 07:00 bar holds the 240 from the two hours before on top of its own 120. **Event time and processing time agree closely while a job keeps up and diverge after a backlog or a replay**, so this code looks right until then.

Count by event time, the till's own timestamp carried in the message. A stream engine that does this has to close each window at some point, and it drops or sets aside an event that comes later than the lateness you allow, so set that limit longer than the outage you mean to survive.

</details>

<details><summary>B. The true bar for each hour, filled in as the job catches up</summary>

✗ The sale happened at 05:30 and the job resumed from its saved offset, so it read every one. This code never looks at when the sale happened: it asks the clock, and the clock says 07:00.

</details>

<details><summary>C. The true bar for 07:00 onward, with the earlier sales dropped as too late</summary>

✗ Stream engines do have a lateness limit past which they discard a record, and these sales are two hours late. This job is a plain loop with no such limit, so it counts every sale it reads, in the hour it reads it.

</details>

---

## Further reading

- [End-to-End Arguments in System Design](https://doi.org/10.1145/357401.357402), 1984: why duplicate suppression and delivery acknowledgment belong at the ends of a system, whatever the layers below promise.
- [Life beyond Distributed Transactions](https://www.cidrdb.org/cidr2007/papers/cidr07p15.pdf), 2007: building large applications without distributed transactions, on messages that can arrive more than once.
- [The Dataflow Model](https://doi.org/10.14778/2824032.2824076), 2015: windows by event time over data that arrives late and out of order, and what it costs to wait for it.
- [Metastable Failures in Distributed Systems](https://doi.org/10.1145/3458336.3465286), 2021: failures that outlast their cause, with retries and backlogs among what keeps them going.

Prev: [Choosing a data store](../choosing-a-data-store/README.md) · Next: [Event-driven architecture](../event-driven-architecture/README.md)
