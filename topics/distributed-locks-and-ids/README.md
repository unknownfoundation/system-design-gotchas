# Distributed locks and IDs

Prev: [Consistency, consensus and clocks](../consistency-consensus-and-clocks/README.md) · Next: [Replication](../replication/README.md) · Reviewed October 2026

A distributed lock lets one worker at a time act on a shared thing across machines, and it usually comes as a lease: a grant from a lock service that runs out on a clock. A distributed ID names a record without the machines that make it agreeing on each one, and it comes from a central source, from a clock with a machine ID, or from random bits.

- **Lock service**: a system that grants a named lock to one client at a time. Its locks are advisory: a writer that never asked for the lock can still write.
- **Lease**: a lock with a TTL (time to live). It ends when the TTL runs out, or goes with a session where it is tied to the holder's connection. A TTL counted on a machine's own steady timer (one that never steps) does not move when the clock steps, an expiry stamp read from a wall clock does, and a stamp compared across machines also carries the gap between their clocks.
- **Holder**: the process a lease was granted to.
- **Process pause**: a stretch in which a process runs none of its code and gets no warning. A GC pause (the runtime stopping every thread to reclaim memory), a page fault, a slow disk, CPU starvation or a stop signal can freeze a holder, and a buffer can hold up its reply.
- **Fencing token**: a number that grows with each grant and travels with each write. The store compares and writes as one step, and keeps the highest number across a restart. A random lock value says who holds a lock, never which holder is newer.
- **Renewal**: extending a lease before it ends, so that long work can keep it.
- **Time-sorted ID**: an ID whose leading bits are the time, followed by a machine ID and a sequence that counts within one millisecond. A common layout of 41, 10 and 12 bits fits a signed 64-bit integer.
- **Machine ID**: the part of a time-sorted ID that keeps two generators apart. Ten bits give 1,024 of them.

Unless a question says otherwise, a lease is a key with a TTL, tied to no connection, and the store behind it takes any write sent to it.

A lock on a majority of independent nodes outlives the loss of one, unless that node comes back empty and serves at once.

Ways to make an ID:

| Way | What it does |
| --- | --- |
| A number per request from a central source | one service hands out the next number on every call |
| Blocks from a central source | a server takes a range from one service and hands numbers out of memory |
| Time with machine ID and sequence | each machine builds the ID alone from its clock, its number and a counter, in 64 bits |
| Random | each machine draws its bits alone, in 128 bits with 122 of them random, and the odds of a repeat grow with the square of the count |

A counter inside one database is unique only there: two writing primaries can repeat it (Replication).

The trouble sits between the grant and the write, where a lock service can be right about every grant while a holder acts on what it last knew, and between an ID's layout and what feeds it.

Each question names everything its team has set up. A retry, a health check, a limit or a background job that it does not mention is not there, and every setting keeps its default.

This topic lets you:

- follow a worker holding a lease through a long freeze, a restarted cache node, a slow customer and a margin check, and say what each one changes
- trace a fencing token through a late write, a transfer of two writes and a call to a carrier's API
- run a lock or a leader through a release in a finally block, a stuck export, a dead machine, a failover, a cut cable, a nightly job and a silent lock service
- judge what an ID rests on when it serves as a cursor, when a clock steps back, during a deploy, at a design review, after a snapshot and through an outage of its source

###### 1. Pausing for forty seconds in the middle of a balance update

A wallet service keeps balances in a key-value store, and its locks live in a lock service built on a five-node consensus group (machines that keep one log of writes in one order). To add a top-up, a worker takes a 30-second lease on the wallet, reads the balance, adds the amount and writes the result back. One worker has read the balance when a GC pause freezes it for 40 seconds. A second worker takes the lease on that wallet and writes its own top-up. The first worker wakes, and its next line of code is the write.

What happens to the first worker's write?

<details><summary>A. The store refuses it</summary>

✗ The lease is over by the time the worker wakes, and the lock service's record says so. That record sits on another machine and the store has no lease to look at: it takes this write as it takes any other, and the balance changes.

</details>

<details><summary>B. It lands, and the second top-up is gone from the balance</summary>

✓ Right. The write lands the moment the worker wakes and replaces the balance that held the second top-up, and the wallet loses that money with no error on either side. If the balance starts at 100 and the top-ups are 10 and 25, the second worker's write makes 125 and the first worker's makes 110.

**A lease's TTL runs out whether or not its holder has noticed**, and a frozen worker cannot notice, since none of its code runs. The consensus group did its whole job, and its log shows the wallet granted to one worker at a time. It has to be able to take a lease from a holder it cannot tell from a dead one, or one crash would block everybody for good. To the store, a write computed from a balance read 40 seconds ago is one more write.

Treat a lease as what keeps workers out of each other's way on a normal day, and have the store itself refuse a late write. A fencing token is the usual way, and it needs a store willing to compare.

</details>

<details><summary>C. It waits in line behind the second worker's lease</summary>

✗ Inside one process a lock makes threads take turns, and a lock service sets out to do the same across machines. The write goes straight to the store, which applies it on arrival, and the second worker's lease is only a record at the lock service.

</details>

<details><summary>D. It is cancelled as the worker wakes</summary>

✗ The lock client on this worker tracks the expiry, and asked after this freeze it would answer that the lease is gone. The write does not pass through it: the next line of code goes straight to the store.

</details>

---

###### 2. Guarding thumbnails with a lock on one cache node

A photo site builds thumbnails in a pool of workers. So that two workers do not build the same one, a worker first takes a lock on a single cache node, which keeps its locks in memory only. A thumbnail is made from the photo alone, and the file store replaces a file whole, under a name taken from the photo. In the middle of one build the cache node restarts, and a second worker then asks to build that thumbnail.

What does the restart cost the photo site?

<details><summary>A. A few seconds of extra work on one thumbnail</summary>

✓ Right. The site pays for one thumbnail built twice, a few seconds of a worker's time, and visitors get the right picture either way.

This lock saves work, and **a lock that only saves work costs that same work when it is lost**. Two builds run, and the file store ends up with one file holding the bytes either worker would have written. The thumbnail depends on the photo alone and a whole file replaces a whole file, which leaves a second holder nothing to damage.

Before you harden a lock, ask what a double run costs. Leave this one as it is, and spend the effort on a lock whose second holder could damage data.

</details>

<details><summary>B. A damaged thumbnail, written by two workers at once</summary>

✗ Two holders of one lock is the standard example of a broken lock, and on a ledger it corrupts things. Here the file store replaces a file whole, which gives a reader one worker's file or the other's, and both hold the same bytes.

</details>

<details><summary>C. The first build stopped, its lock having gone</summary>

✗ The cache node went down in the middle of the build, and a call on a dropped connection ends in an error. The first worker set a key once and asks the cache nothing more. It hears nothing of the restart, and its build runs to the end.

</details>

<details><summary>D. Every build held up while the lost locks are rebuilt</summary>

✗ Every worker's lock lives on one cache node, a single point of failure, and that node went down. It keeps its locks in memory only, so there is nothing to rebuild after the restart: the next request finds the key free and its build starts at once.

</details>

---

###### 3. Holding a seat for five minutes through a six-minute payment

A cinema's site holds a seat for five minutes with a lock that has a TTL, then sends the customer to a payment provider. When a payment goes through, the provider's callback marks the seat booked for that customer and sends a confirmation. One customer's payment takes six minutes. A little after minute five, a second customer asks for the same seat, ready to pay.

What does the cinema end up with for that seat?

<details><summary>A. Two charges and one confirmation, for the second customer</summary>

✗ The slower payment finishes at minute six, and by then the second customer holds the seat. The callback never asks the lock: it marks the seat for the customer the provider names and sends its confirmation, and the slower customer gets one as well.

</details>

<details><summary>B. One charge, the slower payment cancelled when the hold runs out</summary>

✗ A hold's state diagram has an arrow from held back to free, and at minute five the seat goes back. That arrow frees a seat and touches no payment. The provider has never heard of the hold, so the open payment carries on and the charge arrives a minute later.

</details>

<details><summary>C. Two charges for the seat, and two booking confirmations</summary>

✓ Right. Both customers pay and both get a confirmation, which leaves the cinema holding two people's money for one seat it has promised to each.

The hold ends on the clock at minute five, and **a hold that ends stops nothing already under way**. The payment carries on at the provider, and each callback marks the seat with nothing tying it to the hold.

Make the callback a conditional update that books the seat only if nobody has booked it, and give the loser a refund path. Both cards are still charged, since the charge comes before the store can refuse, but one confirmation goes out and one refund.

</details>

<details><summary>D. One charge, the second customer told the seat is in payment</summary>

✗ Booking pages you have used show a seat in payment as taken. The lock is all that marks this seat, and its clock knows nothing of the customer's progress. At minute five the seat is free while the first payment is still open, and the second customer gets it.

</details>

---

###### 4. Checking the lease with ten seconds to spare

After a stale write at a warehouse, you guard the stock update with a check: the worker asks how long its 30-second lease has left and writes only with more than ten seconds remaining. `locks` and `store` are thin clients.

```python
lease = locks.acquire("stock:" + sku, ttl=30)
item = store.get(sku)
item["count"] -= quantity

if lease.remaining() > 10:
    store.put(sku, item)
```

When can a write that has passed the check reach the store?

<details><summary>A. Inside the lease, before it ends</summary>

✗ Ten seconds is far longer than a write takes on a normal day. The margin covers the time between the check and a put that leaves promptly, and neither a pause nor a held packet has an upper limit, so a put held up for 50 seconds in the network arrives long after the lease.

</details>

<details><summary>B. Inside a lease renewed for as long as the write runs</summary>

✗ `remaining()` sits on the same object that renews in the background on some lock clients. The call reads a number and changes nothing: the lock service receives no request from it, and the lease ends on schedule whatever the write is doing.

</details>

<details><summary>C. Within ten seconds, with later writes refused by the store</summary>

✗ In some frameworks for calls between services, a request deadline travels with the call, and the server cancels the call when it runs out. `store.put` carries a key and an item, and the margin stays behind in an `if` on the worker. The store takes the write whenever it comes.

</details>

<details><summary>D. At any later time, even after the lease has ended</summary>

✓ Right. The write can reach the store at any later time, and a stock count can land long after the lease has ended, with the store accepting it.

The margin describes one past moment: when the check ran, more than ten seconds were left. **A pause or a delayed packet between a check and its write can outlast any margin**: if the check runs at second 5, with 25 seconds left, and a 40-second pause follows, the put arrives at second 45.

Keep the check if you like, since it stops a worker that has plainly run out of time, and put what refuses a late write in the store.

</details>

---

###### 5. Sending a fencing token that reaches the store late

A hotel's pricing service guards each room's rate row with a lease. Every grant carries a growing number, and the store refuses a write whose number is lower than the highest it has seen. Reads send no number. Here is one row's morning:

```text
worker 1   gets the lease, number 33
worker 1   reads the row
worker 1   freezes
           the lease ends
worker 2   gets the lease, number 34
worker 2   reads the row
worker 1   wakes and writes with 33
worker 2   writes with 34
```

What does the store do with the write numbered 33?

<details><summary>A. Accepts it, and then refuses 34 as a conflict</summary>

✗ Version checks work this way: the first write wins, and the second has to read again and retry. A fencing token compares old with new and asks nothing about what a worker read. Since 34 is higher than 33, the second write goes straight through.

</details>

<details><summary>B. Refuses it as older than number 34</summary>

✗ By the time 33 arrives, number 34 has been granted, and the timeline shows the grant above the write. The grant happened at the lock service. The store hears of a number only when a request carries one, and so far no request has brought it anything above 33.

</details>

<details><summary>C. Holds it back and applies it after 34</summary>

✗ A log with sequence numbers sorts its entries by number, and these are numbers too. The store holds nothing back: it compares 33 with the highest number it has seen, finds nothing higher, and applies the write as it arrives.

</details>

<details><summary>D. Accepts it, and then accepts 34 on top of it</summary>

✓ Right. The store takes 33 and then 34. Worker 2 built its write from a read made before 33 landed, which loses worker 1's change to the room's rate with no write refused.

**A store that compares numbers knows only the numbers it has seen**, and when 33 arrives it has seen nothing above 33. A late write loses an update only in the window between the new holder's read and its write.

Have the new holder show its number to the store before it reads. With 34 on the read, the store refuses a late 33 in this window and after it. A store that asks the lock service about each write closes the window too, at the price of a call to that service on every write.

</details>

---

###### 6. Making two writes under one fencing token

A bank's transfer moves money between two accounts in one store, as two separate writes that each carry the lease's number. The store refuses any number lower than the highest it has seen across the store.

```text
worker 1   writes the debit to account A with 33
worker 1   freezes
           the lease ends
worker 2   gets the lease, number 34, and writes another record with 34
worker 1   wakes and sends the credit to account B with 33
```

What do the two accounts show after the first worker wakes?

<details><summary>A. The debit and the credit, both kept</summary>

✗ The worker held a valid lease when the transfer began, and both writes carry that lease's number. The store judges each write as it arrives, and by the time the credit comes it has seen 34, so account B gets nothing.

</details>

<details><summary>B. Neither entry, the debit rolled back with the credit</summary>

✗ On every database you have used, a transaction that fails partway leaves nothing behind. The debit and the credit are two separate writes, and to the store the credit has no connection to the debit, which landed under a number that was fine at the time and stays where it is.

</details>

<details><summary>C. The debit without its credit</summary>

✓ Right. The debit stays and the credit is refused, which leaves the money gone from account A and in no account at all. If both accounts start at 500 and the transfer is 100, they end at 400 and 500.

**A fencing token judges each write alone and undoes nothing that landed before**, and here it did its job: the frozen worker's late write never landed, and that refusal is what left the transfer half done.

Send the debit and the credit as one transaction, which the store refuses whole if it comes after the freeze, with the accounts still at 500 each. Where one transaction cannot span them, write the steps so that the next holder can find a half-done transfer and pick it up.

</details>

<details><summary>D. Both entries, marked pending for the next holder</summary>

✗ A job queue hands unfinished work to the next worker, and worker 2 is the next holder here. It got a number and wrote its own record, and this store holds no mark for unfinished work, so worker 2 finds no pending entry.

</details>

---

###### 7. Buying a shipping label under a 30-second lease

A worker in a warehouse system holds a 30-second lease on an order and calls a carrier's API to buy a shipping label. The API's one optional header is an idempotency key, a value the caller chooses to mark a request as a repeat. One worker is about to make the call when it freezes for longer than its lease. By the time it wakes, another worker has bought the label.

Which change stops a frozen worker buying a second label?

<details><summary>A. A lease twice as long</summary>

✗ The freeze outlasted the lease, and the lease's 30 seconds are a number you chose. Doubling them moves the moment the other worker may start from second 30 to second 60, and the call to the carrier still carries no word of any lease, so a freeze that runs past that moment ends in two labels, as this one did.

</details>

<details><summary>B. A check of the lease right before the call</summary>

✗ Any code review would ask a worker to check its lease again before it spends money. The worker froze about to make the call, past every line above it, and a check placed there would already have passed. It wakes with the purchase as its next act, and the carrier, which has never seen your lease, sells it a label.

</details>

<details><summary>C. The fencing token sent to the carrier in a header</summary>

✗ A growing number is the textbook cure for a late write, and it costs one header to send. It works only where the receiver compares it. The carrier reads no such header, so the woken worker's call looks like any other purchase.

</details>

<details><summary>D. A key taken from the order, sent with each call</summary>

✓ Right. Both workers send the same key for the order, and the carrier answers the woken worker with the label it already sold. The warehouse pays for one label however long the freeze lasts.

**The check has to sit where the effect happens**, and the only party that sees every purchase of this label is the carrier. A lease on your side cannot reach a call that wakes late, and the key from the order is how you use the carrier's own check.

Send that key on every call for the order. Where an outside service offers neither a number it compares nor a key it honours, no lease on your side closes this gap, so plan for a repeat purchase.

</details>

---

###### 8. Releasing the lock in a finally block

You write a nightly report job on a thin `cache` client, whose `set_if_absent` sets a key only if the key is missing. One run takes 45 seconds, and the scheduler starts a second run at second 31 of it.

```python
def run_report(report_date):
    if not cache.set_if_absent("report-lock", "locked", ttl=30):
        return
    try:
        build_report(report_date)
    finally:
        cache.delete("report-lock")
```

What does the first run's delete remove?

<details><summary>A. The lock that the second run is working under</summary>

✓ Right. At second 45 the key belongs to the second run, and the first run's delete frees it. If a third run starts a second later it gets the lock, and two runs work side by side on tonight's report.

The first run's lock ended at second 30 and the second run set the key again at 31, so `report-lock` is a different lock under the same name, and **a plain delete names a key and never its holder**. The `finally` block runs however the job ends, this late finish included.

Store a value per holder and delete only if the value is yours, as one step. The first run's delete then removes nothing and a third run is refused. The first run still works 15 seconds past its own lock, which this fix does not touch.

</details>

<details><summary>B. Nothing, its key having gone at the end of the TTL</summary>

✗ The first run's TTL ended at second 30, and its own key went with it. The second run set the key again at second 31, and at second 45 the delete finds a key of that name, which is all it asks.

</details>

<details><summary>C. Its own lock, held from the start</summary>

✗ The first run took this lock at its start and is still working, so it counts the lock as its own. The TTL counts seconds and knows nothing of `build_report`, so the first run's lock was gone 15 seconds before this delete.

</details>

<details><summary>D. A refusal from the cache, which knows each key's owner</summary>

✗ Locks inside one process have owners, and unlocking from another thread is an error on many of them. The cache holds a key and the value `"locked"`, and the delete call takes a key name alone, so there is no owner to compare.

</details>

---

###### 9. Renewing a lease from a thread beside a stuck export

A pool of workers runs customers' exports. A worker takes a 30-second lease on the customer it is serving, and a background thread in that worker renews it every ten seconds. Late one evening that worker's one work thread blocks on a read from a partner service that has stopped answering, with no timeout set.

Where are that customer's exports by the next morning?

<details><summary>A. On a second worker that took over after thirty seconds</summary>

✗ A lease is the usual answer to a stuck job, and thirty seconds passed long ago. It frees a job only when renewals stop, and a read that blocks one thread leaves the other free to send them. Each request from a second worker finds the customer taken.

</details>

<details><summary>B. On two workers side by side, the stuck one included</summary>

✗ Two workers overlapping is the failure every lease warning describes, when a lease ends under live work. Renewals every ten seconds keep this one from ending, so one worker holds the customer all night, and it is the one making no progress.

</details>

<details><summary>C. Still on the stuck worker, with none finished</summary>

✓ Right. No export finishes and no other worker may start one, and by morning the customer has nothing, under a lease that looks perfectly healthy. If the night runs 12 hours, the lock service sees 4,320 renewals and the stuck worker finishes 0 exports.

The thread that renews and the thread that works are separate, and **a renewal shows only that the process is running**, which lets one thread sit blocked while the other keeps the lease alive for as long as the machine stays up.

Renew from the work loop, so that a stuck loop stops renewing and a second worker takes the customer when the lease runs out. Where that cannot work, cap the renewals, and takeover comes late instead of never. Set a timeout on the partner read as well, which frees the thread that is stuck.

</details>

<details><summary>D. Marked failed by the lock service, for lack of progress</summary>

✗ The lock service watches its holders and would notice one that went quiet. Renewals are all it ever receives, and no report of progress on an export ever reaches it. All night it sees a holder in good order.

</details>

---

###### 10. Raising the lease to ten minutes after a pause

A ledger service had one stale write after a long GC pause, so you raise the lease on each customer's ledger from 30 seconds to ten minutes. The platform's health checks watch the machines. A minute into its work, the machine holding one customer's lease loses power.

What happens to that customer's updates?

<details><summary>A. They wait out the rest of the ten minutes</summary>

✓ Right. The updates stop for the nine minutes the lease has left: the machine died at second 60, and that customer's ledger accepts nothing until second 600.

A machine that loses power releases nothing, and **a lease's TTL sets how long nobody can act after its holder dies**. A longer lease makes that wait longer on every crash.

Set the TTL by the wait a crash may cost that customer, and handle the long pause in the store, with a fencing token. If the wait matters more, a lock tied to the holder's session clears when the session times out.

</details>

<details><summary>B. They go on at once as the dropped connection frees the lock</summary>

✗ The power cut drops the machine's connection, and a lock tied to a session is freed once that session times out. The lock on this ledger is a key with a TTL, which never counted on a connection to keep it. The dropped connection changes nothing, and the key stays until its time is up.

</details>

<details><summary>C. They stop for good, until someone clears the lock by hand</summary>

✗ A lock with no TTL and a dead owner stays stuck until someone clears it. The TTL exists for this case: the key expires on its own at the ten-minute mark, and the next worker takes it with nobody's help.

</details>

<details><summary>D. They resume in seconds, once health checks mark the machine dead</summary>

✗ The platform can know within seconds that the machine has gone, and it may replace the machine straight away. That knowledge stays with the platform, and the lock store reads the key's TTL and nothing else, so the updates wait exactly as long with the check as without it.

</details>

---

###### 11. Failing over the lock store to its replica

A booking service keeps its locks in a key-value store with one replica, which copies the primary's stream of writes asynchronously. A position counts bytes along that stream. The traces of one failover read:

```text
second 5   worker 1   lock request acknowledged, write ends at position 81,204
second 5   primary    dies
second 6   script     replica has received up to position 81,160
second 6   script     promotes the replica and points the clients at it
second 7   worker 1   carries on with its booking
second 7   worker 2   asks for the same lock
```

What does the second worker get from the new primary?

<details><summary>A. A refusal, with the first worker still holding the lock</summary>

✗ The store acknowledged the first worker's request, and a replica is there so that acknowledged data outlives the primary. An asynchronous copy goes out after the acknowledgement, and this one had not reached the lock's write, so the new primary, asked to set the key, finds it absent and treats the lock as free.

</details>

<details><summary>B. A wait while the new primary catches up on the old one's locks</summary>

✗ A planned switchover lets a replica catch up before it takes over, because the old primary is still there to send. The first worker's lock died with the old primary, unsent, so the new primary has nothing to wait for.

</details>

<details><summary>C. An error, until every client registers its locks again</summary>

✗ On some coordination stores a lock belongs to a client's session, and clients re-establish sessions after a failover. A lock in this key-value store is one key, which no client registers anywhere, and an absent key is one the next request can set.

</details>

<details><summary>D. The lock, while the first worker is still at work</summary>

✓ Right. The new primary grants the lock while the first worker is still at work, which leaves two workers each believing they hold the booking, and both can go ahead on it.

The lock's write ends at position 81,204 and the replica had reached 81,160, and **a lock acknowledged but not yet copied dies with the primary**. The new primary has never heard of it.

Have a new primary wait out the longest TTL before it serves locks, so that the first worker's lease has run out before the second worker gets the lock. That wait is paid on every failover, as time with no locks granted.

</details>

---

###### 12. Running a heartbeat cable between two schedulers

A billing scheduler runs on an active node, the leader, which is the one machine that runs the schedule. A standby is joined to it by a dedicated heartbeat cable, and no third machine is in the arrangement. The standby takes over as leader after five seconds of silence on the cable, and the active node logs a warning. The cable fails at midnight, while both nodes stay healthy on the main network.

Which node runs the next morning's billing?

<details><summary>A. The active node alone, with the standby waiting</summary>

✗ The active node is still serving on the main network, in plain view of the standby. The standby's decision comes from the heartbeat cable alone, and that cable has gone quiet. At the five-second mark it starts the schedule too.

</details>

<details><summary>B. Neither node, with both stopping to stay safe</summary>

✗ A member of a majority group that finds itself cut off stops, since it cannot reach enough votes. A pair has no majority to lose, and neither node has a rule that tells it to stop, so the cut leaves both running.

</details>

<details><summary>C. Both nodes, each sure that the other is dead</summary>

✓ Right. Each node hears nothing, takes the other for dead and runs the schedule, and customers get billed by two machines the next morning.

The standby waits out five seconds of silence and takes the role, because **a dead peer, a slow peer and a cut-off peer all look the same from this end**. A timeout tells a node that a message did not come, and two nodes alone have no way to settle what that means.

Let a third party decide: have each node ask a lock service for the role, so that the first to ask gets it and the second is refused, and this cut leaves one running. Keep the heartbeats as a reason to ask, and never as the decision itself.

</details>

<details><summary>D. The standby alone, after the active node stands down</summary>

✗ A node that loses its peer steps aside in some designs, to leave the choice to the other. The active node logs its warning and goes on, since to it the silence means that the standby has died.

</details>

---

###### 13. Running a nightly invoice job on two nodes

Two nodes each run the same job from their own scheduler at 02:00 by their own clock, and the clocks are about five seconds apart. The job takes a shared lock named for the job, sends the day's invoices in about three seconds and releases. `locks` is a thin client, `try_acquire` returns nothing when the lock is held, and `send_todays_invoices` sends every invoice dated today.

```python
def send_invoices():
    lock = locks.try_acquire("send-invoices")
    if lock is None:
        return
    try:
        send_todays_invoices()
    finally:
        lock.release()
```

What do customers get from tonight's run?

<details><summary>A. One invoice each, from whichever node fired first</summary>

✗ The `if lock is None` line is there to make the second node skip, and it does when both nodes fire together. By the time the second node asks, the first has sent everything and released, so `try_acquire` finds nothing held and the job runs again.

</details>

<details><summary>B. Two invoices each, from one run after the other</summary>

✓ Right. The first node sends the invoices and releases three seconds in. The second node fires two seconds later, finds the lock free and sends every customer the day's invoice again.

**A lock stops two holders at one moment and keeps no memory of a run that has ended**, and these two runs never overlap.

Keep a record per run, keyed by the job and the date, and leave it there after the job ends. The second node then finds tonight's run already done and skips it, and customers get one invoice.

</details>

<details><summary>C. No invoice, with each node seeing the other's lock</summary>

✗ Deadlock is what people remember of two parties and a lock: each stuck on the other. `send_invoices` takes one lock, and whichever node asks first gets it, sends the invoices and releases, so customers do get invoices.

</details>

---

###### 14. Fetching new messages by the last ID seen

A group chat stores messages under time-sorted IDs, on twenty servers with a machine ID each and clocks that differ by up to 50 ms. A server takes the ID when it starts a message, and the message can be read once its write commits, some tens of milliseconds later. Each client polls twice a second for messages with an ID above the last it has seen, and stores nothing else about what it has fetched. One group runs at about 40 messages a second.

What does a member of that group get from these polls?

<details><summary>A. Nearly every message, with a few missing for good</summary>

✓ Right. Some messages never reach the member, and a reply in the middle of the conversation is missing from their screen. Say the commit comes 20 to 60 ms after the ID: over ten minutes at this rate, a few hundred of 24,000 messages are never delivered.

**A cursor that compares IDs assumes that every lower ID has already been seen**, and a time-sorted ID does not give that. A message can become readable after a poll has passed its ID, because a slower clock gave it a lower ID or because it committed late, and every later poll asks only for IDs above the cursor. Better clocks do not close this: with clocks in perfect step, about a hundred are still lost to the commit delay.

Poll with an overlap and drop repeats: a 1-second overlap loses nothing here, as long as the commit delay and the clock difference together stay below it. Or take a per-group sequence from one writer at commit.

</details>

<details><summary>B. Every message, each poll picking up where the last one ended</summary>

✗ An ID that holds the time sorts by time, and the cursor resumes right after the newest one. That order holds across twenty servers only as well as their clocks do, and a message becomes readable when its write commits, whatever its ID, so a lower ID can appear behind the cursor.

</details>

<details><summary>C. Every message, with some repeated under an ID shared by two servers</summary>

✗ Clock trouble in an ID scheme brings repeats to mind first, and the scheme's time part does come from those clocks. Each server here has its own machine ID, which keeps two servers' IDs apart at any clock reading, and no two servers share an ID.

</details>

<details><summary>D. Every message, the late ones shown on a later poll</summary>

✗ Once the late message commits it can be read, and a page that reloads the whole conversation picks it up on its next read. The client's poll asks only for IDs above its cursor, which has moved past that message, so no later poll ever matches it.

</details>

---

###### 15. Stepping a generator's clock back

You write the ID function for an order service. It hands out about three IDs every millisecond, each built from the time, a machine ID and a sequence. Then a time-sync correction steps the machine's clock back 200 ms.

```python
last_ms = -1
sequence = 0

def next_id():
    global last_ms, sequence
    now = clock.millis()
    if now == last_ms:
        sequence += 1
    else:
        last_ms = now
        sequence = 0
    return (now << 22) | (MACHINE_NUMBER << 12) | sequence
```

What does this generator hand out for the next 200 ms?

<details><summary>A. IDs that keep rising as the sequence carries on</summary>

✗ Inside one millisecond the sequence keeps IDs moving upward. The `else` branch resets it whenever the millisecond differs from the last one, and it never asks which way the clock moved, so the sequence restarts and the time part drops.

</details>

<details><summary>B. New lower IDs, with none of them repeated</summary>

✗ A clock that has drifted a little slow gives lower IDs, and for an order list that is only a sorting matter. The correction took the clock back over milliseconds this function already used, at a rate that filled them, and the new IDs land on old values.

</details>

<details><summary>C. No IDs at all while the clock is behind</summary>

✗ This function uses the layout of the best-known generator of its kind, which refuses to issue while the clock is behind its last value. Refusing takes a comparison, and this code compares for equality alone. `now == last_ms` is false, so the function takes the `else` branch and returns an ID.

</details>

<details><summary>D. IDs it already gave to earlier orders</summary>

✓ Right. The generator gives out IDs it has already given, and a new order can arrive under the ID of an order placed 200 ms earlier. At exactly 3 a millisecond, all 600 IDs of the next 200 ms are repeats, and with counts that vary around 3, about 70 percent are.

After the step the millisecond part returns to values already used, the sequence restarts at zero and the machine ID has not changed. **An ID built from time, machine ID and sequence repeats once all three come round again**, and this code never checks for that.

Refuse or wait while the clock is behind the last value used, which costs those 200 ms and repeats nothing. Store that value as well, since a restart with the clock behind would otherwise lose the protection.

</details>

---

###### 16. Rolling out the order service's ID generators

The ten copies of an order service each make time-sorted IDs at a few IDs a millisecond, and a copy takes its machine ID from its slot in the deployment, 0 to 9. Orders live on twenty shards (separate databases chosen by customer), each with its own unique key on the ID. During a rolling deploy the old copy in slot 3 goes on finishing requests, creating orders, for thirty seconds beside its replacement, also in slot 3.

What do the two copies in slot 3 do to the orders?

<details><summary>A. Different IDs for every order, from the sequence</summary>

✗ On one machine the sequence is what keeps IDs apart inside a millisecond. Each copy here keeps a sequence of its own, and both restart at 0 every millisecond. With one machine ID between them, they count through the same values.

</details>

<details><summary>B. A refusal at start for the replacement, its slot being registered</summary>

✗ Where a machine ID comes under a lease, a second claim on it gets refused. A slot in a deployment is a position and nothing more: nothing asks whether another live copy still uses it, and the replacement starts making IDs straight away.

</details>

<details><summary>C. Two customers' orders stored under one ID, on two shards</summary>

✓ Right. The two copies hand out the same IDs and most of those orders go to different shards, where no insert fails, and two customers' orders end up under one ID. A lookup by that ID can now find either order.

**Two live copies on one machine ID reach the same milliseconds and the same sequence values**, and a shard's unique key checks only its own rows. If the copies make 1 and 3 IDs a millisecond on average, varying from one millisecond to the next, about 26,000 IDs end up held by two orders over those thirty seconds, about 19 in 20 of them on two shards. A small offset between their clocks leaves the count about the same.

Keep the slot's number out of use until the old copy has stopped, or hold the number under a lease and wait out the TTL before anyone reuses it.

</details>

<details><summary>D. A failed insert each time, from the unique key</summary>

✗ The order table has a unique key on every shard, and inside one table that key stops a repeat. Each shard's key sees that shard's rows alone, so a repeat fails only when both customers happen to live on one shard, about one time in twenty.

</details>

---

###### 17. Checking random IDs against a shared table

A document service in three regions gives each document an ID with 122 random bits from the operating system's secure source, a few million a day. At a design review, someone proposes one shared table in one region, which every insert checks first, at about 80 ms a round trip from the other two regions.

What does the shared table catch in the service's first ten years?

<details><summary>A. A few duplicates a year from regions that cannot see each other</summary>

✗ Machines that do not coordinate and hand out sequence numbers repeat each other from the first few IDs. The document IDs are draws from a space of 122 bits, so taking 5 million a day, a year's expected repeats come to about 3 in 10^19, nowhere near a few.

</details>

<details><summary>B. A duplicate whenever two regions create a document in one instant</summary>

✗ With IDs built from time, one instant can mean one ID. Random bits take nothing from the clock, so the moment plays no part: two documents made in the same millisecond repeat with a chance of about 2 in 10^37, the same as any other two.

</details>

<details><summary>C. No duplicate, at a round trip on each insert</summary>

✓ Right. The table finds no duplicate in ten years, and inserts from the two far regions wait 80 ms more for it, which leaves the document service with a dependency that catches nothing.

Taking 5 million a day, ten years comes to about 18 billion documents and a chance of any repeat of about 3 in 10^17, since **the odds of a repeat depend on how many random bits the ID has and on the square of the count**. With 64 random bits about 9 repeats would be expected in the same ten years, and there a check earns its place.

Drop the table, and spend the care on the source of the bits, which is what those odds depend on.

</details>

<details><summary>D. A burst of duplicates once the table passes a billion rows</summary>

✗ At 32 random bits this service would repeat constantly, because repeats arrive early in a small ID space. At 122 bits the first billion rows carry a chance of about 1 in 10^19, and the odds climb smoothly with the count, with no row count that sets off a burst.

</details>

---

###### 18. Starting forty copies from one snapshot

A support desk gives each ticket an ID of 122 random bits. Its ID library holds a random generator in the process's memory, seeded once from the operating system at start. The platform starts new copies from a snapshot of the process, taken after start, and one morning traffic brings forty new copies up from that snapshot.

What do the forty copies hand out for their first tickets?

<details><summary>A. Forty separate runs of IDs, none repeated</summary>

✗ At 122 bits a repeat is out of reach, and those odds hold for independent draws. The forty copies do not draw independently: each starts from the state held in the snapshot and computes the same next value as the other thirty-nine.

</details>

<details><summary>B. The same IDs from each copy, in the same order</summary>

✓ Right. Every copy hands out one sequence of IDs, in one order, and tickets from different copies arrive under the same ID that morning. If each of the forty hands out three IDs, that is 120 IDs and only 3 distinct ones.

The odds behind 122 random bits assume independent draws, and **a snapshot copies a generator's state into every process restored from it**. Each copy goes on from one point and computes the same bits, and nothing checks a random ID for a repeat.

Read the system's source on each call, or reseed on restore. Some platforms refresh the system's generator when they restore a snapshot, and that helps a library reading from it on each call while doing nothing for a generator held in memory.

</details>

<details><summary>C. A refusal from the library, on spotting restored state</summary>

✗ A library can reseed from a hook that some platforms run on restore. The support desk's library has none. To it, restored memory is the memory it has always held, and it goes on from the next value without a word.

</details>

<details><summary>D. Repeats across the whole fleet from then on, older copies included</summary>

✗ A broken random source taints everything that draws from it. The older copies were seeded separately, each with state of its own, and the snapshot never touched them, unless one of them is the process it was taken from. Their IDs stay distinct from the new copies' and from each other's.

</details>

---

###### 19. Losing the ID service for a minute

A shop's servers take order numbers in blocks of 10,000 from a central ID service. Each server uses about fifty a minute, and every one fetched its current block within the last two hours. The ID service goes down for one minute at lunchtime.

What happens to orders placed during that minute?

<details><summary>A. They fail, with no source of new numbers</summary>

✗ One central service sits on the architecture diagram, and every order number starts there. It is called once per block, though. At fifty numbers a minute a block lasts over three hours, and a block fetched in the last two has no call due in this minute.

</details>

<details><summary>B. They go through on numbers from each server's block</summary>

✓ Right. Each server goes on handing out numbers it already holds, and customers check out as usual with no failed order at the shop.

The service is called once per block, so **an outage reaches a server only when it needs a new block**. A server that fetched its block two hours ago still holds 4,000 numbers, enough to ride out 80 minutes without the service.

Size the block by the longest outage you mean to ride out. A server that restarts loses the numbers in hand, and order numbers no longer follow time across servers, so let nothing depend on them being gapless or in order of time.

</details>

<details><summary>C. They go through on numbers another server has used</summary>

✗ Two writing primaries repeat each other's counters once they go their own way, and these servers are on their own for this minute. The service handed out the blocks before it went down, each range to one server, and a server working through its own range never reaches a number another server holds.

</details>

<details><summary>D. They queue on each server, waiting for the service</summary>

✗ A call to a dead dependency usually leaves the caller waiting, often for a long timeout. Taking a number from memory is no call at all, and each order placed in this minute gets its number as usual.

</details>

---

###### 20. Losing the lock service for twenty seconds

A power company runs a billing pipeline that reads smart-meter readings from 400 partitions. One worker serves each partition under a 15-second lease from a lock service: it renews every 5 seconds, keeps the partition's running totals in memory and writes them to a store. When a worker dies, the orchestrator starts a replacement, which takes the lease and rebuilds the totals in about four minutes. You killed one worker in a test, and the bill run went through on time. The lock service is five machines with an elected leader, and its log shows that each election last year took about 20 seconds and lost no lease. The lock client's sample exits when a lease has run out with no answer from the service, and you are writing that code for this worker.

What should a worker do when its lease has run out with no answer?

<details><summary>A. Stop writing, keep its totals and ask for the lease again</summary>

✓ Right. Each worker goes quiet when its lease runs out and writes again soon after the new leader is in place, so a 20-second election delays the bill run by seconds and no totals are rebuilt.

Exiting costs little when one worker's lease runs out, as your test showed. All 400 leases come from one service, though, and **a silent lock service reaches every holder at the same moment**, so this code runs on the whole fleet within 15 seconds of the election starting. The log's "no lease lost" changes nothing here, because a worker judges its lease by its own timer and cannot read the log while the service is silent. With the sample's exit, a 20-second election stops every partition for about four minutes.

Stop the writes and nothing else, and put a random delay before the request: with up to ten seconds of delay, the new leader meets about 40 requests a second, where the same code with no delay sends it all 400 at once. Exit only if the answer is a refusal, since that means another worker holds the partition.

</details>

<details><summary>B. Hand its partition to another worker and stop</summary>

✗ A rebalance moves work off a worker in trouble, and it does that well when the trouble is one worker's. During an election every worker's lease runs out together. No healthy worker is left to take the partition, and the lock service grants nothing until a leader answers.

</details>

<details><summary>C. Exit, leaving the orchestrator to start a clean replacement</summary>

✗ This is what the lock client's sample does, and your test passed with one partition's totals four minutes behind. Your test took one lease away, and an election takes the lock service away from all 400, so all 400 workers exit, all 400 partitions wait for rebuilt totals, and 400 replacements ask for leases together.

</details>

<details><summary>D. Keep writing until the lock service answers again</summary>

✗ The log shows no lease lost in any election, and 20 seconds is over quickly. A worker hearing no answer cannot tell an election from a fault around its own machine, and after such a fault the orchestrator may already have started a replacement that holds the lease. The store takes any write, so both workers' totals land on that partition.

</details>

---

## Further reading

- [Leases](https://doi.org/10.1145/74850.74870), 1989: leases as a fault-tolerant mechanism for keeping file caches consistent across machines.
- [Time, clocks, and the ordering of events in a distributed system](https://doi.org/10.1145/359545.359563), 1978: what it means for one event to come before another across machines, and logical clocks that put events in one order.

Prev: [Consistency, consensus and clocks](../consistency-consensus-and-clocks/README.md) · Next: [Replication](../replication/README.md)
